# SpringHive

Spring Boot microservice monorepo generator — scaffold study and MVP friendly multi-module projects from a single JSON specification.

- **Repo:** `github.com/franciscoabsl/spring-hive`
- **Author:** Francisco Lima
- **RFC:** `SpringHive-RFC-v1.0.docx` (source of truth for all decisions)

---

## What SpringHive Does

SpringHive exposes a single REST endpoint that receives a JSON specification describing a microservice project and returns a ZIP archive containing a fully structured Maven multi-module monorepo, ready to run with a single `docker compose up --build` command.

---

## Technology Stack (SpringHive itself)

- Java 21
- Spring Boot 3.3.x
- Maven (multi-module)
- FreeMarker (template engine)
- SpringDoc OpenAPI (Swagger UI)

---

## Generated Project Technology Matrix (Fixed — never ask the user)

| Technology | Version |
|---|---|
| Java | 21 |
| Spring Boot | 3.5.14 |
| Spring Cloud | 2025.0.2 |
| PostgreSQL | 16 |
| MySQL | 8 |
| MongoDB | 7 |
| Apache Kafka | 3.7.x |
| Redis | 7 |
| Resilience4j | 2.4.0 |
| Flyway | 10.x |
| Micrometer + Brave | 1.x (Zipkin) |
| Docker base image | eclipse-temurin:21-jre-alpine |

---

## API Contract

### Endpoint
```
POST /api/v1/generate
Content-Type: application/json
```

### Success Response (200)
```json
{
  "projectName": "string",
  "generatedAt": "ISO-8601",
  "services": [{ "name": "string", "port": 8081 }],
  "gateway": { "port": 8080, "routes": [...] },
  "infrastructure": { "broker": "KAFKA", "zipkinUrl": "http://localhost:9411" },
  "downloadUrl": "/api/v1/generate/download/{uuid}.zip"
}
```

### Error Response (400)
```json
{
  "status": 400,
  "error": "Validation failed",
  "violations": [
    { "field": "services[0].resilience", "message": "RESILIENCE4J requires at least one entry in calls[]" }
  ]
}
```

---

## JSON Specification

### Complete Schema
```json
{
  "project": {
    "name": "string",           // required — kebab-case, regex ^[a-z][a-z0-9-]*$, max 50
    "groupId": "string",        // required — valid Java package, e.g. com.franciscolima
    "description": "string"     // optional — max 255 chars
  },
  "authService": {
    "jwtSecret": "string",              // required — min 32 chars
    "generateRegisterEndpoint": boolean, // default true
    "roles": ["string"]                 // required — min 1, max 10, UPPER_SNAKE_CASE
  },
  "services": [                 // required — min 1, max 10
    {
      "name": "string",                 // required — kebab-case, unique
      "path": "string",                 // optional — gateway route prefix
      "description": "string",          // optional
      "requiresAuth": boolean,           // required
      "packageStructure": "TRADITIONAL|HEXAGONAL",
      "generateSampleController": boolean,
      "calls": ["string"],              // service names for OpenFeign sync calls
      "database": {
        "engine": "NONE|POSTGRESQL|MYSQL|MONGODB"
      },
      "cache": boolean,                 // uses infrastructure Redis
      "observability": boolean,         // uses infrastructure observability
      "resilience": "NONE|RESILIENCE4J"
    }
  ],
  "infrastructure": {
    "broker": "NONE|KAFKA",
    "topics": [                         // required if broker != NONE
      {
        "name": "string",               // kebab-case, unique
        "producers": ["string"],        // service names
        "consumers": ["string"]         // service names
      }
    ],
    "cache": "NONE|REDIS",
    "observability": "NONE|ACTUATOR|ACTUATOR_PROMETHEUS",
    "zipkin": boolean
  }
}
```

---

## Port Assignment (Automatic — no user input)

| Service | Port |
|---|---|
| api-gateway | 8080 |
| auth-service | 8081 |
| services[0] | 8082 |
| services[1] | 8083 |
| services[n] | 8082 + n |

Only port 8080 is published externally in Docker Compose. All others use `expose`.

---

## Validation Rules

### Errors (block generation)
```
project.name                  → required, regex ^[a-z][a-z0-9-]*$, max 50
project.groupId               → required, valid Java package format
authService.jwtSecret         → required, min 32 chars
authService.roles             → required, min 1, max 10, UPPER_SNAKE_CASE
services                      → required, min 1, max 10
services[].name               → required, kebab-case, unique in project
services[].path               → optional, kebab-case, unique in project
services[].calls[]            → must reference existing service names
services[].calls[]            → no self-reference
resilience: RESILIENCE4J      → calls[] must not be empty
services[].cache: true        → infrastructure.cache must not be NONE
services[].observability: true → infrastructure.observability must not be NONE
broker: NONE + topics present  → error
topics[].producers[]          → must reference existing services
topics[].consumers[]          → must reference existing services
zipkin: true + observability: NONE → error
```

### Warnings (allow generation, return in response)
```
broker: KAFKA + topics empty          → warning
topic with no producers               → warning
topic with no consumers               → warning
circular dependency in calls[] A→B→A  → warning
calls[] non-empty + resilience: NONE  → warning (suggest RESILIENCE4J)
```

---

## Generated Project Structure

```
{project-name}/
├── pom.xml                    # Parent POM — fixed versions, all modules
├── .env                       # Generated secrets — in .gitignore
├── .env.example               # Empty template — safe to commit
├── .gitignore
├── docker-compose.yml
├── init-databases.sql         # Creates databases + isolated user per service
├── prometheus.yml             # Scrape config for all services with observability
├── README.md                  # Auto-generated documentation
├── api-gateway/               # Always generated — port 8080
├── auth-service/              # Always generated — port 8081
└── {service-name}/            # User services — ports 8082+
```

---

## Package Structures

### TRADITIONAL
```
src/main/java/{groupId}/{servicename}/
├── {ServiceName}Application.java
├── controller/
├── service/
├── repository/
├── model/
├── dto/
├── config/       # KafkaConfig, RedisConfig, ResilienceConfig...
├── kafka/        # Topics.java, EventPublisher.java, EventListener.java
└── client/       # FeignClient + Fallback per calls[] entry
```

### HEXAGONAL
```
src/main/java/{groupId}/{servicename}/
├── {ServiceName}Application.java
├── domain/
│   ├── model/          # .gitkeep + explanatory comment
│   ├── ports/
│   │   ├── in/         # .gitkeep + explanatory comment
│   │   └── out/        # .gitkeep + explanatory comment
│   └── service/        # .gitkeep + explanatory comment
├── application/
│   └── usecase/        # .gitkeep + explanatory comment
└── infrastructure/
    ├── adapters/
    │   ├── in/
    │   │   ├── web/        # REST controllers
    │   │   └── messaging/  # Kafka listeners (if consumer)
    │   └── out/
    │       ├── persistence/ # .gitkeep + explanatory comment
    │       ├── messaging/   # Kafka publishers (if producer)
    │       └── http/        # Feign clients (if calls[] non-empty)
    └── config/
```

---

## auth-service (Always Generated — Traditional — PostgreSQL)

### Database schema (English — always)
```sql
CREATE TABLE users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email         VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    role          VARCHAR(50)  NOT NULL,
    active        BOOLEAN      NOT NULL DEFAULT true,
    created_at    TIMESTAMP    NOT NULL DEFAULT now()
);
```

### Endpoints (all via api-gateway — no auth required)
```
POST /api/v1/auth/login     → returns JWT + refresh token
POST /api/v1/auth/register  → if generateRegisterEndpoint: true
POST /api/v1/auth/refresh   → exchange refresh token
POST /api/v1/auth/logout    → invalidate token
GET  /api/v1/auth/health    → health check
```

### JWT propagated headers (gateway → downstream services)
```
X-User-Id:    {uuid}
X-User-Role:  {ADMIN|DOCTOR|...}
X-User-Email: {email}
```

---

## Kafka Configuration

### Topics.java (generated per service — with warning comment)
```java
// WARNING: if you rename a topic, update ALL services that reference it
public final class Topics {
    private Topics() {}
    public static final String APPOINTMENT_SCHEDULED = "appointment-scheduled";
    public static final String PATIENT_REGISTERED = "patient-registered";
}
```

### Resilience4j instance names = calls[] entry names
```yaml
# If calls: ["patient-service"] → instance name must be "patient-service"
resilience4j:
  circuitbreaker:
    instances:
      patient-service:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
  timelimiter:
    instances:
      patient-service:
        timeoutDuration: 3s
```

---

## Internal Service Communication

OpenFeign calls go **directly** through Docker network — NOT through api-gateway.

```yaml
# application-dev.yml (running locally outside Docker)
services:
  patient-service:
    url: http://localhost:8083

# application-prod.yml (running inside Docker)
services:
  patient-service:
    url: http://patient-service:8083
```

---

## Database Initialization (one container per engine)

```sql
-- One user and one database per service
CREATE USER auth_service_user WITH PASSWORD '${AUTH_SERVICE_DB_PASSWORD}';
CREATE DATABASE auth_service_db OWNER auth_service_user;
GRANT ALL PRIVILEGES ON DATABASE auth_service_db TO auth_service_user;

CREATE USER patient_service_user WITH PASSWORD '${PATIENT_SERVICE_DB_PASSWORD}';
CREATE DATABASE patient_service_db OWNER patient_service_user;
GRANT ALL PRIVILEGES ON DATABASE patient_service_db TO patient_service_user;
```

---

## Dockerfile (Multi-stage Monorepo)

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .

# Copy ALL sibling POMs (required for monorepo dependency resolution)
COPY auth-service/pom.xml auth-service/
COPY patient-service/pom.xml patient-service/
COPY appointment-service/pom.xml appointment-service/

RUN ./mvnw dependency:go-offline -pl patient-service -am -q

COPY patient-service/src patient-service/src
RUN ./mvnw clean package -pl patient-service -am -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/patient-service/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## Conventions (enforce in ALL generated code)

- **Language:** everything in English — class names, methods, variables, DB columns, comments, README
- **DB column naming:** snake_case (`created_at`, `password_hash`, `user_id`)
- **Service names:** kebab-case (`patient-service`, `auth-service`)
- **Java packages:** lowercase, no hyphens (`patientservice`, `authservice`)
- **Kafka topics:** kebab-case (`appointment-scheduled`, `patient-registered`)
- **Restart policy:** `unless-stopped` (never `always`)
- **Env vars:** `env_file: .env` (never inline in docker-compose)
- **Ports:** only expose, never publish (except api-gateway port 8080)
- **No H2:** removed from scope — PostgreSQL for development too
- **No shared database:** one database per service, always

---

## SpringHive Backend Package Structure

```
com.springhive/
├── api/
│   └── GeneratorController.java
├── dto/
│   ├── request/
│   │   ├── ProjectRequest.java
│   │   ├── AuthServiceRequest.java
│   │   ├── ServiceRequest.java
│   │   ├── DatabaseRequest.java
│   │   ├── InfrastructureRequest.java
│   │   └── TopicRequest.java
│   └── response/
│       ├── GenerationResponse.java
│       ├── ServiceMetadata.java
│       └── ViolationResponse.java
├── validation/
│   ├── ProjectValidator.java
│   └── CrossFieldValidator.java
├── generator/
│   ├── ProjectGenerator.java       # orchestrates everything
│   ├── PomGenerator.java
│   ├── DockerComposeGenerator.java
│   ├── DatabaseInitGenerator.java
│   ├── EnvGenerator.java
│   ├── PrometheusGenerator.java
│   ├── ServiceGenerator.java
│   ├── AuthServiceGenerator.java
│   ├── GatewayGenerator.java
│   └── ReadmeGenerator.java
├── template/
│   └── TemplateRenderer.java       # FreeMarker wrapper
├── zip/
│   └── ZipPackager.java
└── storage/
    └── TempFileStorage.java        # 1h TTL for generated ZIPs
```

---

## Development Order (follow strictly)

### Phase 1 — SpringHive foundation (no generation yet)
- Spring Boot project setup
- DTOs mapping the full JSON spec
- POST /api/v1/generate receiving and validating JSON
- All field-level validations
- All cross-field validations
- Swagger/OpenAPI
- **Test:** Postman sending clinica-saude.json — validations work correctly

### Phase 2 — Folder structure + POMs
- Parent POM generation
- Child pom.xml per service (inheritance only, no deps yet)
- TRADITIONAL empty folder structure
- Application.java per service
- .gitignore
- ZIP generation + download endpoint
- **Test:** unzip → `mvn clean package` → must compile

### Phase 3 — Docker Compose base
- docker-compose.yml with databases
- init-databases.sql with per-service users
- .env and .env.example
- Healthchecks on all databases
- **Test:** `docker compose up -d` → all databases healthy

### Phase 4 — auth-service
- All dependencies in pom.xml
- application.yml with datasource + JWT config
- Flyway migration V1__create_users_table.sql
- All classes: controller, service, repository, model, config
- Dockerfile multi-stage
- **Test:** `docker compose up -d auth-service` → POST /auth/login returns JWT

### Phase 5 — api-gateway
- Spring Cloud Gateway dependencies
- Auto-generated routes from services[]
- JwtAuthenticationFilter
- X-User-Id, X-User-Role, X-User-Email headers propagation
- **Test:** login via gateway port 8080, valid token passes, invalid returns 401

### Phase 6 — Simple service (no features)
- PostgreSQL only, no cache, no messaging, no resilience
- generateSampleController: true
- Dockerfile correct
- **Test:** full stack up — gateway + auth + 1 service, authenticated route works

### Phase 7 — Features (one at a time, test after each)
```
7.1  PostgreSQL + Flyway           → compile + migration runs
7.2  Redis cache                   → RedisConfig + container healthy
7.3  Kafka producer                → EventPublisher + Kafka container up
7.4  Kafka consumer                → EventListener + message consumed
7.5  OpenFeign calls               → sync call works between services
7.6  Resilience4j                  → circuit breaker opens, fallback responds
7.7  Actuator + Prometheus         → /actuator/prometheus returns metrics
7.8  Zipkin                        → traces visible in dashboard
7.9  MongoDB                       → container + Spring Data Mongo
7.10 MySQL                         → container + correct datasource
```

### Phase 8 — HEXAGONAL structure
- Full skeleton with .gitkeep + explanatory comments
- Reposition generated classes into correct adapters
- **Test:** compile

### Phase 9 — README generation
- Generate complete README from submitted JSON
- Validate all sections documented correctly

---

## Official Test Project — clinica-saude

Use this JSON for ALL generation tests throughout development.
It exercises every MVP feature combination.

```json
{
  "project": {
    "name": "clinica-saude",
    "groupId": "com.franciscolima",
    "description": "Healthcare clinic management system"
  },
  "authService": {
    "jwtSecret": "clinica-saude-jwt-secret-key-32-chars!!",
    "generateRegisterEndpoint": true,
    "roles": ["ADMIN", "DOCTOR", "PATIENT", "RECEPTIONIST"]
  },
  "services": [
    {
      "name": "patient-service",
      "path": "patients",
      "description": "Patient registration and management",
      "requiresAuth": true,
      "packageStructure": "TRADITIONAL",
      "generateSampleController": true,
      "calls": [],
      "database": { "engine": "POSTGRESQL" },
      "cache": true,
      "observability": true,
      "resilience": "NONE"
    },
    {
      "name": "appointment-service",
      "path": "appointments",
      "description": "Appointment scheduling",
      "requiresAuth": true,
      "packageStructure": "TRADITIONAL",
      "generateSampleController": true,
      "calls": ["patient-service"],
      "database": { "engine": "POSTGRESQL" },
      "cache": false,
      "observability": true,
      "resilience": "RESILIENCE4J"
    },
    {
      "name": "notification-service",
      "path": "notifications",
      "description": "Email and SMS notifications",
      "requiresAuth": false,
      "packageStructure": "TRADITIONAL",
      "generateSampleController": false,
      "calls": [],
      "database": { "engine": "NONE" },
      "cache": false,
      "observability": true,
      "resilience": "NONE"
    },
    {
      "name": "billing-service",
      "path": "billing",
      "description": "Billing and payment management",
      "requiresAuth": true,
      "packageStructure": "HEXAGONAL",
      "generateSampleController": true,
      "calls": ["appointment-service"],
      "database": { "engine": "POSTGRESQL" },
      "cache": false,
      "observability": true,
      "resilience": "RESILIENCE4J"
    }
  ],
  "infrastructure": {
    "broker": "KAFKA",
    "topics": [
      {
        "name": "appointment-scheduled",
        "producers": ["appointment-service"],
        "consumers": ["notification-service", "billing-service"]
      },
      {
        "name": "appointment-cancelled",
        "producers": ["appointment-service"],
        "consumers": ["notification-service", "billing-service"]
      },
      {
        "name": "patient-registered",
        "producers": ["patient-service"],
        "consumers": ["notification-service"]
      }
    ],
    "cache": "REDIS",
    "observability": "ACTUATOR_PROMETHEUS",
    "zipkin": true
  }
}
```

### Feature coverage of clinica-saude

| Feature | Service |
|---|---|
| TRADITIONAL | patient, appointment, notification |
| HEXAGONAL | billing-service |
| PostgreSQL | patient, appointment, billing |
| No database | notification-service |
| Redis cache | patient-service |
| Kafka producer | patient-service, appointment-service |
| Kafka consumer | notification-service, billing-service |
| Kafka producer + consumer | appointment-service |
| OpenFeign | appointment→patient, billing→appointment |
| Resilience4j | appointment-service, billing-service |
| Actuator + Prometheus | all services |
| Zipkin | all services |
| requiresAuth: false | notification-service |
| generateSampleController: false | notification-service |

---

## Golden Rule

```
implement feature
      ↓
generate clinica-saude
      ↓
mvn clean package → must compile
      ↓
docker compose up --build → must start healthy
      ↓
test endpoint via Postman
      ↓
commit
      ↓
next feature
```

---

## Key Design Decisions (never revisit without strong reason)

- api-gateway and auth-service are always generated — not optional
- Ports are automatic — user never chooses ports
- One database per service — no exceptions
- One container per database engine — services share the container, not the database
- Internal service calls (OpenFeign) go directly through Docker network — never through gateway
- JWT validation centralized in gateway — downstream services only read propagated headers
- auth-service always TRADITIONAL — hexagonal would be over-engineering
- RabbitMQ deferred to V2 — requires different exchange/queue model
- No H2 — too amateur for the project's positioning
- All generated code in English — DB columns, comments, README, everything
