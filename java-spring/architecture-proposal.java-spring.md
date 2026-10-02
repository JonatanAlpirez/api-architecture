# Architecture Proposal — Java + Spring Boot 3

> **Estado:** referencia de implementación cerrada — cubre los 7 puntos del plan (stack, estructura, OpenAPI, DB, sync, endpoints, setup) + tradeoffs vs Node + TS y Python (FastAPI). Comparada con las otras 2 references ([`node-ts`](../node-ts/architecture-proposal.node-ts.md), [`python`](../python/architecture-proposal.python.md)).
>
> **Diferencia clave con las otras 2:** este stack usa **PostgreSQL** (no SQLite) por las fricciones conocidas de Hibernate con SQLite (dialect, type system, ID generation). Ver Q4 en `architecture-decisions.md`.
>
> Todas las decisiones referenciadas viven en [`architecture-decisions.md`](../architecture-decisions.md). Esta propuesta **asume** que esas decisiones están cerradas y solo aterriza nombres concretos, paths y código.

---

## 1. Stack justificado

| Capa | Decisión | Versión target | Por qué |
| --- | --- | --- | --- |
| Lenguaje | **Java** | 21 (LTS) | Records, pattern matching, virtual threads (preview), LTS hasta 2031 |
| Framework HTTP | **Spring Boot 3** | 3.2+ | De facto del mercado Java; batteries-included (DI, security, data, web); springdoc-openapi integration |
| ORM | **Spring Data JPA (Hibernate)** | 6.x (via Spring Boot 3) | JPA estándar; repositorios derivados sin escribir SQL |
| DB | **PostgreSQL** | 16+ | Partner nativo de JPA/Hibernate; JSONB, sequences, full-text out-of-the-box |
| Validación | **Jakarta Validation** | 3.0+ (via Spring Boot 3) | Estándar Java EE/Jakarta EE; anotaciones como `@NotNull`, `@Size`, `@Email` |
| OpenAPI integration | **springdoc-openapi** | 2.x | Genera spec 3.0 desde controllers + anotaciones; Swagger UI built-in |
| Tests | **JUnit 5** + **Mockito** | (latest) | Estándar Java; JUnit 5 moderno vs JUnit 4 legacy |
| HTTP testing | **MockMvc** + **@SpringBootTest** | (latest) | Spring Boot Test; testing de capa HTTP sin levantar server real |
| Logging | **Logback + SLF4J** | (Spring Boot default) | Estándar Java; JSON via `logstash-logback-encoder` si queremos parseo centralizado |
| Lint/format | **Spotless** + **SpotBugs** | (latest) | Formateo (Google Java Format / Palantir) + análisis estático. Alternativa: Checkstyle + PMD |
| Build | **Maven** | 3.9+ | Estándar Java; alternativa: Gradle (más flexible pero menos mainstream para Spring Boot enterprise) |
| Migraciones | **Flyway** | 9.x (via Spring Boot 3) | Spring Boot auto-detecta Flyway en el classpath; SQL plano versionado |
| CORS | `@CrossOrigin` o `WebMvcConfigurer` global | (built-in Spring) | Built-in; `CorsConfigurationSource` bean configurable por `application.yml` |
| Package manager | Maven (`mvn`) | 3.9+ | — |

**Por qué este stack sobre las alternativas evaluadas** (ver Q2/Q4/Q5/Q17/Q21 en [`architecture-decisions.md`](../architecture-decisions.md)):

- **Spring Boot 3 sobre Quarkus/Micronaut/Helidon**: de facto del mercado Java, ecosystem enorme (Spring Security, Spring Data, Spring Cloud si crece), springdoc-openapi. Quarkus es cloud-native pero comunidad más chica; Micronaut similar.
- **Spring Data JPA (Hibernate) sobre jOOQ/MyBatis/Jdbi**: estándar Java, repositorios derivados sin escribir SQL. jOOQ es SQL-first (más cerca de Drizzle); MyBatis es SQL mapper manual.
- **PostgreSQL sobre SQLite** (única stack con Postgres, ver Q4): evita fricciones conocidas de Hibernate con SQLite (dialect support limitado, type system dynamic, ID generation quirks). Costo: Postgres corriendo local (Docker / `brew services`).
- **Jakarta Validation sobre Hibernate Validator custom**: estándar Java EE / Jakarta EE; integración nativa con Spring.
- **Flyway sobre Liquibase**: SQL plano versionado, simple. Liquibase es más potente (XML/YAML) pero más verboso.

---

## 2. Estructura de carpetas

Monolito modular — un solo deployable, packages independientes entre sí (bajo acoplamiento, alta cohesión).

```
api-java-spring/
├── pom.xml                                       # Maven config (deps, plugins, build)
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/api/
│   │   │       ├── ApiApplication.java            # @SpringBootApplication; main() entrypoint
│   │   │       │
│   │   │       ├── config/
│   │   │       │   ├── CorsConfig.java            # WebMvcConfigurer + CorsConfigurationSource bean
│   │   │       │   ├── OpenApiConfig.java         # springdoc-openapi setup (Swagger UI + security scheme)
│   │   │       │   └── ApiKeyFilter.java          # OncePerRequestFilter para X-API-Key
│   │   │       │
│   │   │       ├── common/
│   │   │       │   ├── exception/
│   │   │       │   │   ├── ResourceNotFoundException.java
│   │   │       │   │   ├── GlobalExceptionHandler.java   # @ControllerAdvice
│   │   │       │   │   └── ErrorEnvelope.java            # { error: { code, message, details? } }
│   │   │       │   └── pagination/
│   │   │       │       └── PageResponse.java
│   │   │       │
│   │   │       ├── modules/
│   │   │       │   └── resource/                 # ejemplo: módulo "resource" (otros siguen este patrón)
│   │   │       │       ├── Resource.java          # @Entity JPA model
│   │   │       │       ├── ResourceRepository.java # extends JpaRepository<Resource, Long>
│   │   │       │       ├── ResourceService.java    # @Service — business logic
│   │   │       │       ├── ResourceController.java # @RestController — HTTP layer
│   │   │       │       ├── dto/
│   │   │       │       │   ├── CreateResourceRequest.java   # record con @Valid annotations
│   │   │       │       │   ├── UpdateResourceRequest.java
│   │   │       │       │   └── ResourceResponse.java
│   │   │       │       └── ResourceControllerIT.java        # @SpringBootTest + MockMvc integration test
│   │   │       │
│   │   │       └── health/
│   │   │           ├── HealthController.java     # @RestController — GET /health (sin auth)
│   │   │           └── HealthResponse.java
│   │   │
│   │   └── resources/
│   │       ├── application.yml                   # config principal (DB, server port, OpenAPI, etc.)
│   │       ├── application-dev.yml               # overrides para dev
│   │       └── db/migration/                     # Flyway migrations: V1__create_resources.sql, V2__add_tags.sql, ...
│   │
│   └── test/
│       └── java/
│           └── com/example/api/
│               └── modules/resource/
│                   └── ResourceControllerIT.java  # tests de integración
│
├── .env.example                                  # SPRING_DATASOURCE_URL, SPRING_DATASOURCE_USERNAME, etc.
└── README.md
```

### Convenciones de Spring Boot 3 (antes del patrón)

Cuatro cosas que confunden al que viene de NestJS / FastAPI / Express:

**1. `modules/<feature>/` = organización por feature, no por capa.** Spring (al igual que NestJS/Angular) agrupa **todo** lo relativo a un concepto de negocio en un package: entity, repository, service, controller, dto, tests. No hay `controllers/`, `services/`, `repositories/` globales a nivel de application — eso sería por capa técnica. Cada feature es independiente.

**2. `<Feature>Controller` ≠ "controller de NestJS" — es un `@RestController`.** Es una clase con anotaciones:

```java
@RestController
@RequestMapping("/resources")
@Tag(name = "resources")  // OpenAPI grouping
public class ResourceController {
    private final ResourceService service;

    public ResourceController(ResourceService service) {
        this.service = service;  // constructor injection (Spring auto-wires)
    }

    @GetMapping
    public List<ResourceResponse> list() {
        return service.findAll();
    }
}
```

La DI se hace vía constructor injection (no decorators como NestJS, no `Depends()` como FastAPI). Spring resuelve el grafo de dependencias automáticamente vía component scanning.

**3. Spring Data JPA repositories son interfaces con queries derivadas.** No escribís SQL — el nombre del método se traduce a query:

```java
public interface ResourceRepository extends JpaRepository<Resource, Long> {
    Optional<Resource> findByName(String name);
    List<Resource> findByCreatedAtAfter(Instant date);
}
```

A diferencia de SQLAlchemy (donde escribís queries explícitas) o Drizzle/MikroORM (donde construyes queries tipadas), Spring Data JPA genera la query del nombre del método. Para queries complejas, `@Query` con JPQL o native SQL.

**4. Records (Java 14+) son el equivalente moderno de DTOs/Pydantic schemas.** Inmutables, concisos, con `equals`/`hashCode`/`toString` autogenerados:

```java
public record CreateResourceRequest(
    @NotBlank @Size(min = 1, max = 100) String name,
    @Size(max = 500) String description
) {}
```

Las anotaciones de Jakarta Validation (`@NotBlank`, `@Size`) son las equivalentes de `Field(min_length=...)` en Pydantic o `.min(1).max(100)` en Zod. Spring las aplica automáticamente cuando el controller tiene `@Valid`.

---

## 3. Flujo OpenAPI

```
Spring controllers + Jakarta Validation annotations
        │
        │ (runtime: springdoc-openapi escanea @RestController + @Schema)
        ▼
OpenAPI spec 3.0 (auto-generado)
        │
        │ servido en:
        ├── /swagger-ui.html  → Swagger UI (springdoc default)
        ├── /v3/api-docs       → spec JSON
        └── /v3/api-docs.yaml  → spec YAML
                  │
                  │ FE corre: openapi-typescript http://localhost:8787/v3/api-docs -o src/api/types.ts
                  ▼
            src/api/types.ts (tipos TS)
```

**Setup en `OpenApiConfig.java`**:

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("API de [recurso]")
                .description("Backend local para servir datos del data warehouse")
                .version("1.0"))
            .components(new Components()
                .addSecuritySchemes("X-API-Key",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.APIKEY)
                        .in(SecurityScheme.In.HEADER)
                        .name("X-API-Key")));
    }
}
```

**FE workflow** (referencia, no parte de este repo):

```bash
# Una vez (o en CI cuando cambia el spec):
pnpm dlx openapi-typescript http://localhost:8787/v3/api-docs -o src/api/types.ts
```

El FE importa los tipos generados sin acoplamiento a un cliente HTTP específico (decisión Q9).

---

## 4. Esquema DB

**PostgreSQL** (única stack con Postgres — ver Q4). Migraciones forward-only via Flyway.

> **Escenarios posibles** (la elección se difiere a implementación, ver Q10 en [`architecture-decisions.md`](../architecture-decisions.md)):
> - **A) Greenfield / API-first:** la DB de la API es la **única** fuente de verdad. Datos nacen vía `POST /resources` (R8 CRUD desde v1).
> - **B) Alongside existing DB (caso actual):** DB fuente pre-existente (SQLite, mismo motor que Node/Python); sync poblará la DB de la API (ver §5).
> - **C) Source sigue activa:** sync periódico o incremental.

**JPA entity example** (recursos del dominio siguen este patrón):

```java
// modules/resource/Resource.java
@Entity
@Table(name = "resources")
public class Resource {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    @NotBlank
    private String name;

    @Column(length = 500)
    private String description;

    @Column(name = "created_at", nullable = false, updatable = false)
    @CreationTimestamp
    private Instant createdAt;

    // getters, setters, equals, hashCode (o usar Lombok @Data)
}
```

**Migraciones** (Flyway, SQL plano):

```sql
-- src/main/resources/db/migration/V1__create_resources.sql
CREATE TABLE resources (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description VARCHAR(500),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_resources_name ON resources(name);
```

- Spring Boot auto-detecta Flyway al startup y aplica migraciones pendientes.
- Archivos commiteados al repo.
- Forward-only (ver Q6 en [`architecture-decisions.md`](../architecture-decisions.md)).

**Tabla `flyway_schema_history`** (auto-manejada por Flyway):
- Lleva registro de qué migraciones se aplicaron.
- Solo aplica las nuevas al startup.

**Driver JDBC**: `org.postgresql:postgresql` (en `pom.xml`).

**Configuración** (`application.yml`):

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/api_db
    username: api_user
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate  # nunca 'update' en prod — usar Flyway para cambios
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
```

**Postgres local** (dev): `docker run -d -p 5432:5432 -e POSTGRES_DB=api_db -e POSTGRES_USER=api_user -e POSTGRES_PASSWORD=*** postgres:16` o `brew services start postgresql`.

---

## 5. Plan de sync (solo si escenario B o C)

> **Si el escenario es A (greenfield / API-first):** esta sección **no aplica**. La DB de la API se crea vacía desde migraciones y los datos nacen vía `POST /resources` (R8). En ese caso, eliminar `SOURCE_DB_URL`, el script `SyncCommand.java`, y el command line runner. Mantener §4 (esquema DB) y §6 (endpoints) tal cual.
>
> **Nota específica de este stack:** la DB fuente es **SQLite** (mismo motor que Node/Python — la fuente no es Postgres), y la DB target es **PostgreSQL**. Esto requiere leer SQLite con un driver y escribir a Postgres con otro.

Script CLI (Spring Boot CommandLineRunner) que copia datos desde la DB fuente (SQLite) hacia la DB de la API (PostgreSQL). **Idempotente** — re-ejecutable sin duplicar.

```java
// scripts/SyncCommand.java
@Component
public class SyncCommand implements CommandLineRunner {

    private static final Logger log = LoggerFactory.getLogger(SyncCommand.class);

    private final DataSource targetDataSource;  // Postgres (Spring auto-configura)

    @Value("${SOURCE_DB_URL:}")
    private String sourceDbUrl;  // path al SQLite fuente

    @Override
    @Transactional
    public void run(String... args) throws Exception {
        if (sourceDbUrl.isBlank()) {
            log.info("SOURCE_DB_URL not set — skipping sync (escenario A o no aplica)");
            return;
        }

        // 1. Conectar a la DB fuente (SQLite, JDBC directo)
        try (Connection source = DriverManager.getConnection(sourceDbUrl)) {
            // 2. Conectar a la DB target via JPA EntityManager
            EntityManager em = targetDataSource.unwrap... // simplificar con @PersistenceContext
            // 3. Sync por entidad (transacción por batch)
            // ...
        }
    }
}
```

**Comando**: `mvn spring-boot:run -Dspring-boot.run.arguments=--sync` (o un profile dedicado).

**Cuándo corre**:
- Dev: manual cuando el FE necesita data fresca.
- Prod: después de las migraciones en cada deploy (cron o webhook — fuera de scope v1).

**Decisiones de sync** (ver Q10 en [`architecture-decisions.md`](../architecture-decisions.md)):
- Sync one-way (fuente SQLite → API Postgres).
- API mantiene su propia DB; la fuente no se toca en runtime.
- Si la DB crece, sync incremental con `WHERE updated_at > last_sync` — pero v1 hace full sync, se optimiza si la performance lo demanda.

---

## 6. Endpoints iniciales

CRUD completo desde v1 (R8). Ejemplo con `resource` (los demás recursos siguen el patrón).

| Método | Path | Auth | Body | Response | Status |
| --- | --- | --- | --- | --- | --- |
| `GET` | `/resources` | `X-API-Key` | — | `List<ResourceResponse>` | 200 |
| `GET` | `/resources/{id}` | `X-API-Key` | — | `ResourceResponse` | 200 / 404 |
| `POST` | `/resources` | `X-API-Key` | `CreateResourceRequest` | `ResourceResponse` | 201 / 422 |
| `PUT` | `/resources/{id}` | `X-API-Key` | `UpdateResourceRequest` | `ResourceResponse` | 200 / 404 / 422 |
| `PATCH` | `/resources/{id}` | `X-API-Key` | `UpdateResourceRequest` (parcial) | `ResourceResponse` | 200 / 404 / 422 |
| `DELETE` | `/resources/{id}` | `X-API-Key` | — | — | 204 / 404 |
| `GET` | `/health` | — | — | `HealthResponse` | 200 |

**Headers siempre presentes**:
- Request: `X-API-Key: ***` (excepto `/health`).
- Request: `Content-Type: application/json` (en POST/PUT/PATCH).
- Response: `Content-Type: application/json` + CORS headers.

**Envelope de error** (ver Q19 en [`architecture-decisions.md`](../architecture-decisions.md)):

```json
// 4xx / 5xx
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Resource with id 123 does not exist",
    "details": { "resourceId": 123 }
  }
}
```

**Validación Jakarta** (Q17): si el body no cumple las constraints, devuelve 422 con `details` listando los campos inválidos (Spring Boot + `@ControllerAdvice`).

**Códigos de error comunes**:
- `VALIDATION_ERROR` → 422
- `UNAUTHORIZED` → 401 (falta `X-API-Key` o inválido)
- `NOT_FOUND` → 404
- `INTERNAL_ERROR` → 500

---

## 7. Setup commands

```bash
# Setup inicial
mvn clean install              # descarga deps + compila

# Migraciones (auto-aplicadas al startup con Flyway; o manual)
mvn flyway:migrate             # opcional — Spring Boot las aplica al arrancar

# Sync inicial desde la DB fuente (si escenario B/C)
mvn spring-boot:run -Dspring-boot.run.arguments=--sync

# Dev (auto-reload con Spring DevTools)
mvn spring-boot:run             # incluye DevTools si está en pom.xml

# Build
mvn clean package              # genera target/api-java-spring-1.0.jar

# Prod
java -jar target/api-java-spring-1.0.jar

# Tests
mvn test                       # JUnit 5 (corre una vez)
mvn test -Dtest=ResourceControllerIT  # un test específico
mvn verify                     # con coverage (si Jacoco configurado)

# Lint / format
mvn spotless:apply             # formatea código
mvn spotless:check             # verifica formato (CI)
mvn spotbugs:check             # análisis estático
```

**`pom.xml` scripts** (Maven goals):

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>com.diffplug.spotless</groupId>
            <artifactId>spotless-maven-plugin</artifactId>
            <version>2.40.0</version>
            <configuration>
                <java>
                    <googleJavaFormat>
                        <style>GOOGLE</style>
                    </googleJavaFormat>
                </java>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**Variables de entorno** (`.env.example` o `application.yml` overrides):

```bash
# Spring Boot auto-detecta SPRING_DATASOURCE_* env vars
SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/api_db
SPRING_DATASOURCE_USERNAME=api_user
SPRING_DATASOURCE_PASSWORD=***
SPRING_PROFILES_ACTIVE=dev

# Custom
PORT=8787
FRONTEND_ORIGIN=http://localhost:5173
API_KEY=***
# SOURCE_DB_URL solo si escenario B/C (ver §4-§5). En escenario A (greenfield), eliminar.
SOURCE_DB_URL=jdbc:sqlite:./path/to/data-warehouse.db
```

**`application.yml`** (config principal):

```yaml
server:
  port: ${PORT:8787}

spring:
  application:
    name: api-java-spring
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
  datasource:
    url: ${SPRING_DATASOURCE_URL}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect

# CORS (Q22)
app:
  cors:
    allowed-origins: ${FRONTEND_ORIGIN}
  api-key: ${API_KEY}

# springdoc-openapi
springdoc:
  swagger-ui:
    path: /swagger-ui.html
  api-docs:
    path: /v3/api-docs
```

---

## Tradeoffs vs las otras 2 propuestas

### vs Node + TypeScript (NestJS)

**Ganamos**:
- **Compile-time type-safety real** con Java 21: generics reificados, records, sealed types. Errores en compile-time, no en runtime.
- **JPA / Hibernate** es el ORM más maduro del mercado (20+ años de evolución). Spring Data JPA genera repos desde interfaces.
- **Ecosystem enterprise más maduro**: Spring Security, Spring Cloud, Spring Batch si crece a microservicios o jobs pesados.
- **Tooling de análisis estático** (SpotBugs, SonarQube, ArchUnit) para enforce de arquitectura.
- **Mejor performance en CPU-bound** (JIT compilation de la JVM).

**Perdés**:
- **Mucho más boilerplate** (~3-5x más líneas que Node para el mismo feature). Records reducen algo, pero constructor injection, anotaciones, etc. suman.
- **Arranque significativamente más lento** (Spring Boot en ~5-10s vs NestJS en ~1s).
- **Iteración más lenta en dev**: DevTools recarga pero tarda varios segundos; en Node el reload es instantáneo.
- **Stack más pesado**: JVM (~200MB), classpath hell, requiere Docker para reproducibilidad.

### vs Python (FastAPI)

**Ganamos**:
- **Compile-time type-safety real** (vs Pydantic en runtime).
- **Performance en CPU-bound** (JIT de la JVM compila a native; Python es interpretado).
- **Spring Data JPA** es más maduro que SQLAlchemy 2.0 async (más joven).
- **Ecosystem enterprise** ya mencionado arriba.
- **Mejor para equipos grandes**: el sistema de tipos de Java y la verbosidad ayudan a mantener consistencia en codebases grandes.

**Perdés**:
- **Mucho más boilerplate** vs Python (mencionado arriba).
- **Arranque más lento** vs FastAPI (~1-2s).
- **Async story** es más maduro en Python (async/await desde 3.5; en Java recién en 21 con virtual threads).
- **Pydantic v2 (Rust core)** es más rápido que las validaciones de Jakarta en benchmarks (específicamente, en validación pura sin DB).

---

## Próximos pasos

1. **Inicializar el proyecto**: `mvn init` o usar [start.spring.io](https://start.spring.io/) con dependencias Web, JPA, PostgreSQL Driver, Flyway, Validation, Spring Boot DevTools.
2. **Levantar Postgres local** (Docker): `docker run -d -p 5432:5432 -e POSTGRES_DB=api_db -e POSTGRES_USER=api_user -e POSTGRES_PASSWORD=*** postgres:16`.
3. **Implementar el esqueleto**: módulo `resource` mínimo (entity + repository + service + controller + DTOs + test).
4. **Validar manualmente**:
   - Swagger UI en `http://localhost:8787/swagger-ui.html`
   - Un POST → GET → PATCH → DELETE con `curl` o Postman
   - El spec en `/v3/api-docs` genera tipos TS correctos via `openapi-typescript`
5. **Implementar auth + CORS** reales y testear con un FE mínimo.
6. **Implementar sync** desde la DB fuente SQLite para una entidad de ejemplo (si escenario B/C aplica).

---

*Propuesta cerrada 2026-10-01 — list para review.*