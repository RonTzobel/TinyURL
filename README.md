# TinyURL – Scalable URL Shortening Service

## Overview

TinyURL is a Spring Boot–based URL shortening service that turns long URLs into short, shareable 6-character codes. The system is designed around three separate data stores, each serving a distinct role: Redis for fast, atomic code reservation; MongoDB for persistent user data and aggregated click statistics; and Cassandra for durable, time-ordered redirect event logs.

**User flow:**
1. A client submits a long URL (optionally tied to a registered user).
2. The service generates a short code and atomically reserves it in Redis.
3. The service returns a short URL of the form `{base.url}{code}/`.
4. Any browser or client following that short URL is redirected to the original long URL.
5. If the short URL was created by a registered user, every redirect is recorded — both as an aggregated monthly counter in MongoDB and as a detailed event row in Cassandra.

---

## Features

- Short URL creation with a 6-character alphanumeric code
- HTTP redirect from short URL to original long URL
- Atomic, collision-safe code generation using Redis `SETNX` (up to 4 retries on collision)
- User registration and lookup
- Per-user redirect analytics: total click count and per-short-URL monthly breakdown (MongoDB)
- Detailed per-user click history with timestamps (Cassandra)
- Swagger 2 API documentation (Springfox)

---

## Architecture

```
Client
  │
  ▼
Spring Boot (AppController)
  ├── Redis          ← short code → JSON payload (atomic reservation via SETNX)
  ├── MongoDB        ← user profiles + aggregated click counters
  └── Cassandra      ← per-redirect event log (time-series)
```

### Component Roles

| Component    | Role |
|--------------|------|
| **Spring Boot** | REST API layer; orchestrates all reads, writes, and redirects |
| **Redis**    | Stores the mapping from short code to `{longUrl, userName}` JSON. Uses `setIfAbsent` (Redis SETNX) to atomically claim a code — if the key already exists the write fails and a new code is generated. No TTL is set, so mappings are permanent by default. |
| **MongoDB**  | Stores `User` documents in the `users` collection (database: `tinydb`). Each user document embeds a `shorts` map keyed by short code, containing a per-month click counter (`shorts.<code>.clicks.<yyyy/MM>`). Also maintains a total `allUrlClicks` counter. Both counters are incremented atomically using `MongoTemplate.updateFirst` with `$inc`. |
| **Cassandra** | Stores every redirect event in the `userclick` table (keyspace: `tiny_keyspace`). The primary key is `(user_name, click_time DESC)` — `user_name` is the partition key for efficient per-user queries and `click_time` is a descending clustering key so the most recent clicks come first. |

### Data Flow — URL Shortening

```
POST /tiny
  │
  ├─ Generate random 6-char code
  ├─ redis.set(code, JSON) via SETNX
  │    ├─ success → return baseUrl + code + "/"
  │    └─ collision → retry (up to 4 times)
  └─ throw RuntimeException if all retries fail
```

### Data Flow — Redirect

```
GET /{tiny}/
  │
  ├─ redis.get(tiny) → deserialize NewTinyRequest
  ├─ if userName present:
  │    ├─ MongoDB: $inc allUrlClicks
  │    ├─ MongoDB: $inc shorts.<tiny>.clicks.<yyyy/MM>
  │    └─ Cassandra: INSERT into userclick
  └─ HTTP 302 redirect → longUrl
```

---

## Tech Stack

### Backend
- Java (Spring Boot)
- Spring Data Redis (`RedisTemplate`, `setIfAbsent`)
- Spring Data MongoDB (`MongoRepository`, `MongoTemplate`)
- Spring Data Cassandra (`CassandraRepository`)
- Jackson (JSON serialization)
- Joda-Time (date utilities)

### Databases
- Redis — short code store / atomic reservation
- MongoDB — user documents with embedded click aggregates
- Apache Cassandra — redirect event time-series

### Documentation
- Springfox Swagger 2 (UI at `/swagger-ui.html`)

### Testing
- JUnit 5 / Spring Boot Test (one context-load smoke test only)

---

## Project Structure

```
TinyURL/
└── src/
    ├── main/java/com/handson/tinyurl/
    │   ├── TinyurlApplication.java          # Spring Boot entry point
    │   ├── config/
    │   │   ├── CassandraConfig.java         # CqlSession bean (host: cassandra:9042, keyspace: tiny_keyspace)
    │   │   ├── MongoConfig.java             # MongoDB config (database: tinydb, auto-index: true)
    │   │   └── SwaggerConfig.java           # Springfox Swagger 2 setup
    │   ├── controller/
    │   │   └── AppController.java           # All REST endpoints
    │   ├── model/
    │   │   ├── NewTinyRequest.java          # Request body: longUrl + userName
    │   │   ├── ShortUrl.java                # Embedded in User: clicks by month
    │   │   ├── User.java                    # MongoDB document (@Document "users")
    │   │   ├── UserClick.java               # Cassandra table (@Table)
    │   │   ├── UserClickKey.java            # Cassandra composite PK: userName + clickTime
    │   │   └── UserClickOut.java            # API response DTO for click history
    │   ├── repository/
    │   │   ├── UserRepository.java          # MongoRepository<User>
    │   │   └── UserClickRepository.java     # CassandraRepository<UserClick>
    │   ├── service/
    │   │   └── Redis.java                   # RedisTemplate wrapper (String, Hash, Set, List ops)
    │   └── util/
    │       └── Dates.java                   # Date formatting / timezone helpers
    └── test/java/com/handson/tinyurl/
        └── TinyurlApplicationTests.java     # Single context-load test
```

---

## How It Works

### 1. Short URL creation (`POST /tiny`)

The controller generates a 6-character code by randomly selecting from the 61-character pool `ABCDEFHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789`. It then calls `redis.set(code, jsonPayload)`, which internally calls `RedisTemplate.opsForValue().setIfAbsent()` — Redis's atomic `SETNX`. If the key already exists (collision), a new code is generated and retried up to 4 times. On success, the service returns `{base.url}{code}/`.

> Note: The character pool omits the letter `G` (a minor gap in the uppercase range — likely a typo). This reduces the theoretical code space slightly but has no functional impact.

### 2. Redirect (`GET /{tiny}/`)

The controller fetches the JSON payload stored in Redis for the given code, deserializes it into a `NewTinyRequest` (containing `longUrl` and optionally `userName`), and issues an HTTP redirect. If a `userName` is present, two analytics writes happen synchronously before the redirect:
- Two `$inc` operations on the user's MongoDB document (total clicks + monthly breakdown per code).
- One `INSERT` into Cassandra's `userclick` table.

### 3. User management (`POST /user`, `GET /user`)

Users are stored in MongoDB's `users` collection. The `name` field has a unique index. A `User` document embeds a `shorts` map that grows automatically as the user creates and clicks short URLs.

### 4. Click analytics (`GET /user/{name}/clicks`)

Queries Cassandra for all `userclick` rows matching the given `user_name`, ordered by `click_time DESC`. Returns a list of `UserClickOut` objects containing `userName`, `clickTime`, `tiny`, and `longUrl`.

---

## API Endpoints

All endpoints are served by `AppController` on the default port (`8080`).

### Create Short URL

```
POST /tiny
Content-Type: application/json

{
  "longUrl": "https://www.example.com/some/very/long/path",
  "userName": "alice"   // optional — omit if not tracking analytics
}
```

**Response:** Plain string with the full short URL, e.g. `http://localhost:8080/aB3xYz/`

---

### Redirect

```
GET /{tiny}/
```

**Response:** HTTP 302 redirect to the original long URL. If the code does not exist in Redis, throws a `RuntimeException`.

---

### Create User

```
POST /user?name={name}
```

**Response:** The created `User` JSON object (id, name, allUrlClicks).

---

### Get User

```
GET /user?name={name}
```

**Response:** The `User` JSON object from MongoDB.

---

### Get User Click History

```
GET /user/{name}/clicks?name={name}
```

> **Note:** There is a bug in the current code — the endpoint path uses `{name}` as a path variable, but the controller method uses `@RequestParam` to read it. This means the path segment `{name}` is actually ignored, and the username must be supplied as a query parameter `?name=`. Both the path placeholder and the query param must be provided for the route to match and work correctly.

**Response:** JSON array of click events:
```json
[
  {
    "userName": "alice",
    "clickTime": "2025-04-29T10:00:00.000+0000",
    "tiny": "aB3xYz",
    "longUrl": "https://www.example.com/some/very/long/path"
  }
]
```

---

### Swagger UI

Once the app is running, interactive API docs are available at:

```
http://localhost:8080/swagger-ui.html
```

---

## Data Storage Design

### Redis — Short Code Store

- **Key:** the 6-character short code (e.g. `aB3xYz`)
- **Value:** JSON string of `NewTinyRequest` (`{"longUrl":"...","userName":"..."}`)
- **Write semantics:** `SETNX` (set-if-not-exists) — guarantees that no two requests claim the same code simultaneously
- **Expiry:** none set by default — mappings are permanent
- **Read path:** every redirect hits Redis first; no database is read on the hot redirect path

### MongoDB — User Profiles and Aggregated Analytics

- **Database:** `tinydb`
- **Collection:** `users`
- **Document structure:**
  ```json
  {
    "_id": "...",
    "name": "alice",
    "allUrlClicks": 42,
    "shorts": {
      "aB3xYz": { "clicks": { "2025/04": 17, "2025/05": 25 } },
      "Qr9mNp": { "clicks": { "2025/05": 8 } }
    }
  }
  ```
- **Updates:** `MongoTemplate` with `$inc` — each redirect atomically increments `allUrlClicks` and the current month's counter for the specific short code
- **Index:** unique index on `name` (auto-created via `autoIndexCreation = true`)

### Cassandra — Redirect Event Log

- **Keyspace:** `tiny_keyspace`
- **Table:** `userclick`
- **Schema (inferred from model):**
  ```cql
  CREATE TABLE userclick (
    user_name  text,
    click_time timestamp,
    tiny       text,
    long_url   text,
    PRIMARY KEY (user_name, click_time)
  ) WITH CLUSTERING ORDER BY (click_time DESC);
  ```
- **Partition key:** `user_name` — all clicks for a user are co-located for fast range queries
- **Clustering key:** `click_time DESC` — most recent clicks are returned first
- **Query:** `SELECT * FROM userclick WHERE user_name = ?`

---

## Local Development Setup

> **Important:** The repository does **not** include a `pom.xml`, `docker-compose.yml`, `Dockerfile`, or `application.properties`. These files are required to build and run the project but are absent from the git history. You will need to create them yourself. The sections below describe exactly what is needed based on the source code.

### Required Software

| Tool | Version |
|------|---------|
| Java | 11 or 17 (Spring Boot 2.x compatible) |
| Maven | 3.6+ |
| Docker | any recent version |
| Docker Compose | v1 or v2 |

---

### Step 1 — Create `pom.xml`

The source code uses the following dependencies (inferred from imports):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.18</version>
  </parent>

  <groupId>com.handson</groupId>
  <artifactId>tinyurl</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>jar</packaging>

  <properties>
    <java.version>11</java.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-cassandra</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
      <groupId>io.springfox</groupId>
      <artifactId>springfox-swagger2</artifactId>
      <version>2.9.2</version>
    </dependency>
    <dependency>
      <groupId>io.springfox</groupId>
      <artifactId>springfox-swagger-ui</artifactId>
      <version>2.9.2</version>
    </dependency>
    <dependency>
      <groupId>joda-time</groupId>
      <artifactId>joda-time</artifactId>
      <version>2.12.5</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

---

### Step 2 — Create `src/main/resources/application.properties`

```properties
# Base URL returned in short URL responses (must end with no trailing slash before the code)
base.url=http://localhost:8080/

# MongoDB
spring.data.mongodb.uri=mongodb://localhost:27017/tinydb

# Redis
spring.redis.host=localhost
spring.redis.port=6379

# Cassandra is configured programmatically in CassandraConfig.java
# (host: cassandra:9042 — change to localhost:9042 for local non-Docker runs)

# Spring MVC compatibility with Springfox Swagger 2
spring.mvc.pathmatch.matching-strategy=ant_path_matcher
```

> If running the app inside Docker Compose (where all services share a network), the Redis host should be `redis` and the Cassandra host is already hardcoded to `cassandra` in `CassandraConfig.java`. For local JVM runs with Docker-only databases, set Redis host to `localhost` and update `CassandraConfig.java` to use `localhost` instead of `cassandra`.

---

### Step 3 — Start Infrastructure with Docker Compose

Create a `docker-compose.yml`:

```yaml
version: '3.8'
services:
  redis:
    image: redis:7
    ports:
      - "6379:6379"

  mongo:
    image: mongo:6
    ports:
      - "27017:27017"

  cassandra:
    image: cassandra:4
    ports:
      - "9042:9042"
    environment:
      - CASSANDRA_DC=datacenter1
      - CASSANDRA_CLUSTER_NAME=tinyurl-cluster
    healthcheck:
      test: ["CMD", "cqlsh", "-e", "describe keyspaces"]
      interval: 15s
      timeout: 10s
      retries: 10
```

Start all three:

```bash
docker compose up -d
```

---

### Step 4 — Initialize Cassandra Keyspace and Table

Cassandra does not auto-create the keyspace. You must create it before starting the app.

Wait for Cassandra to be ready (about 30–60 seconds), then:

```bash
docker exec -it <cassandra-container-name> cqlsh
```

Inside `cqlsh`:

```cql
CREATE KEYSPACE IF NOT EXISTS tiny_keyspace
  WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};

USE tiny_keyspace;

CREATE TABLE IF NOT EXISTS userclick (
  user_name  text,
  click_time timestamp,
  tiny       text,
  long_url   text,
  PRIMARY KEY (user_name, click_time)
) WITH CLUSTERING ORDER BY (click_time DESC);
```

---

### Step 5 — Build and Run the Application

```bash
cd TinyURL
mvn clean package -DskipTests
java -jar target/tinyurl-0.0.1-SNAPSHOT.jar
```

Or run directly via Maven:

```bash
mvn spring-boot:run
```

---

### Step 6 — Verify the App is Running

```bash
curl http://localhost:8080/swagger-ui.html
# Should return the Swagger HTML page

curl -X POST http://localhost:8080/user?name=alice
# Should return a User JSON object
```

---

## Example Run Flow

### 1. Create a user

```bash
curl -X POST "http://localhost:8080/user?name=alice"
```

Response:
```json
{"id":"...","name":"alice","allUrlClicks":0}
```

### 2. Create a short URL

```bash
curl -X POST http://localhost:8080/tiny \
  -H "Content-Type: application/json" \
  -d '{"longUrl":"https://www.github.com","userName":"alice"}'
```

Response:
```
http://localhost:8080/aB3xYz/
```

### 3. Redirect

```bash
curl -L http://localhost:8080/aB3xYz/
# Follows redirect to https://www.github.com
```

On redirect:
- MongoDB `users.alice.allUrlClicks` increments by 1
- MongoDB `users.alice.shorts.aB3xYz.clicks.2025/04` increments by 1
- Cassandra inserts a row into `userclick` with `user_name=alice` and current timestamp

### 4. Fetch click history

```bash
curl "http://localhost:8080/user/alice/clicks?name=alice"
```

Response:
```json
[
  {
    "userName": "alice",
    "clickTime": "2025-04-29T10:00:00.000+0000",
    "tiny": "aB3xYz",
    "longUrl": "https://www.github.com"
  }
]
```

---

## Design Decisions and Tradeoffs

**Redis for atomic code reservation**
Using `setIfAbsent` (Redis SETNX) means that even under concurrent requests, only one caller can claim a given short code. This avoids the read-check-write race condition you would get with a plain database. No distributed lock or transaction is needed.

**Redis as the primary redirect store**
Every redirect resolves the long URL from Redis only — neither MongoDB nor Cassandra is read in the hot path. This keeps redirect latency low regardless of database load.

**MongoDB for flexible user documents**
The per-user click aggregates are stored as a nested map (`shorts.<code>.clicks.<month>`). MongoDB's document model lets this structure grow dynamically without schema migrations. `MongoTemplate` with `$inc` keeps the counters consistent.

**Cassandra for click event history**
Redirect events are append-only and time-ordered. Cassandra's partition-by-user, cluster-by-time model gives efficient range scans per user. It handles high write throughput without contention since each partition (user) is written to independently.

**No URL expiry**
Short codes stored in Redis have no TTL. Mappings are permanent. This simplifies the design but means Redis memory grows without bound over time.

**Synchronous analytics writes on redirect**
The two MongoDB `$inc` operations and the Cassandra insert happen synchronously inside the redirect handler before the HTTP 302 is returned. This keeps the implementation simple but adds latency to each redirect proportional to database write time.

---

## Possible Improvements

- **Add a `pom.xml`, `docker-compose.yml`, and `application.properties`** to the repository so it can actually be cloned and run — currently these are all missing.
- **Fix the `@RequestParam` / `@PathVariable` mismatch** in `GET /user/{name}/clicks` — the path variable `{name}` is silently ignored.
- **TTL / expiry support** — allow short URLs to expire after a configurable duration using Redis key TTLs.
- **Rate limiting** — prevent abuse on `POST /tiny` and redirect endpoints.
- **Custom short codes / aliases** — let users specify their own short code instead of a random one.
- **Async analytics writes** — move MongoDB and Cassandra writes off the redirect hot path (e.g. using a queue or Spring's `@Async`) to reduce redirect latency.
- **Error handling** — the controller throws raw `RuntimeException` on missing codes and exhausted retries; structured error responses with proper HTTP status codes would be more client-friendly.
- **Authentication** — currently any caller can create users and short URLs with any username; there is no auth or ownership check.
- **Real tests** — the only test is a context-load smoke test; unit and integration tests for code generation, collision handling, and analytics writes are missing.
- **Cassandra schema initialization** — the keyspace and table must be created manually before startup; a schema migration tool (e.g. Liquibase for Cassandra, or a startup `CommandLineRunner`) would make onboarding easier.
- **Observability** — add metrics (e.g. Micrometer + Prometheus) for redirect counts, latency, and cache hit/miss rates.

---

## How to Run It Yourself

```bash
# 1. Clone the repo
git clone <repo-url>
cd TinyURL

# 2. Add the missing files (see Local Development Setup above):
#    - pom.xml
#    - src/main/resources/application.properties
#    - docker-compose.yml

# 3. Start infrastructure
docker compose up -d

# 4. Initialize Cassandra (wait ~60s for Cassandra to be ready)
docker exec -it <cassandra-container> cqlsh -e "
  CREATE KEYSPACE IF NOT EXISTS tiny_keyspace
    WITH replication = {'class':'SimpleStrategy','replication_factor':1};
  USE tiny_keyspace;
  CREATE TABLE IF NOT EXISTS userclick (
    user_name text, click_time timestamp, tiny text, long_url text,
    PRIMARY KEY (user_name, click_time)
  ) WITH CLUSTERING ORDER BY (click_time DESC);"

# 5. Build and run
mvn clean package -DskipTests
java -jar target/tinyurl-0.0.1-SNAPSHOT.jar

# 6. Test
curl -X POST "http://localhost:8080/user?name=alice"
curl -X POST http://localhost:8080/tiny \
  -H "Content-Type: application/json" \
  -d '{"longUrl":"https://github.com","userName":"alice"}'
# Copy the returned short URL and open it in a browser or curl -L
```

Swagger UI: [http://localhost:8080/swagger-ui.html](http://localhost:8080/swagger-ui.html)
