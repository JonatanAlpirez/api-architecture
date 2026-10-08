# Architecture Proposal — Java + Spring Boot 3 (stack-specific spec template)

> **Rol:** stack-specific guidance para usar [`spec-template.md`](../spec-template.md) con **Java + Spring Boot 3** (Spring Data JPA + Jakarta Validation + Flyway). Llenás el spec template con tu dominio, después consultás este archivo para saber qué tools, versiones y patterns usar para cada sección.
>
> **No es un code template.** No hay worked example en código todavía (`api-java-reference` está pendiente de crear). Las decisiones del "por qué" referenciadas viven en [`architecture-decisions.md`](../architecture-decisions.md) (Q1-Q26). Los estándares agnósticos viven en [`playbook.md`](../playbook.md) (S1-S16 + S18).
>
> **Diferencia clave con Node/Python:** este stack usa **PostgreSQL** (no SQLite) por las fricciones conocidas de Hibernate con SQLite (dialect support limitado, type system dynamic, ID generation quirks). Ver Q4 en [`architecture-decisions.md`](../architecture-decisions.md).
>
> Mientras `api-java-reference` no exista, los "Worked example" linkean a [`api-node-reference`](https://github.com/JonatanAlpirez/api-node-reference) como analogía (los patterns son similares, no idénticos).

## Stack baseline

| Capa | Decisión | Versión | Notas |
| --- | --- | --- | --- |
| Lenguaje | Java | 21 (LTS) | Records, pattern matching, LTS hasta 2031 |
| Framework HTTP | Spring Boot 3 | 3.2+ | De facto Java; batteries-included (DI, security, data, web) |
| ORM | Spring Data JPA (Hibernate) | 6.x | JPA estándar; repositorios derivados sin escribir SQL |
| DB | PostgreSQL | 16+ | Partner nativo de JPA/Hibernate; JSONB, sequences, full-text |
| Validación | Jakarta Validation | 3.0+ | Estándar Java EE/Jakarta EE; `@NotNull`, `@Size`, etc. |
| OpenAPI integration | springdoc-openapi | 2.x | Genera spec 3.0 desde controllers; Swagger UI built-in |
| Tests | JUnit 5 + Mockito | latest | Estándar Java |
| HTTP testing | MockMvc + @SpringBootTest | latest | Spring Boot Test; testing de capa HTTP sin levantar server real |
| Logging | Logback + SLF4J | (Spring Boot default) | JSON via `logstash-logback-encoder` si queremos parseo centralizado |
| Lint/format | Spotless + SpotBugs | latest | Formateo (Google Java Format / Palantir) + análisis estático |
| Build | Maven | 3.9+ | Estándar Java; alternativa: Gradle |
| Migraciones | Flyway | 9.x | Spring Boot auto-detecta Flyway; SQL plano versionado |
| CORS | `@CrossOrigin` o `WebMvcConfigurer` global | built-in Spring | Built-in; `CorsConfigurationSource` bean configurable |
| Package manager | Maven (`mvn`) | 3.9+ | — |

---

## Mapping a las secciones del spec-template

### §1-2. Project identity + Dominio

**No hay tooling Java-specific.** Completá el spec con tu dominio. JPA entities usan anotaciones como `@Entity`, `@Id`, `@Column` para mapear a tablas.

---

### §3. Endpoints (S3, S4)

- **Controller pattern:** Spring `@RestController` con `@RequestMapping("/resources")` por recurso.
- **CRUD verbs:** los 5 endpoints estándar vía `@GetMapping`, `@PostMapping`, `@PatchMapping`, `@DeleteMapping`. `POST` devuelve 201 por default en Spring 6+, `DELETE` se configura con `@ResponseStatus(HttpStatus.NO_CONTENT)`.
- **Auth (S4):** Spring Security con `OncePerRequestFilter` custom que valida `X-API-Key` con `MessageDigest.isEqual()` (constant-time). Aplicar globalmente con `SecurityFilterChain` bean.
- **Path param validation:** `@PathVariable int id` o `@PathVariable @Positive int id` (Jakarta Validation).
- **Documentación OpenAPI:** `@Operation`, `@ApiResponse`, `@Tag` (anotaciones de springdoc-openapi).

**Worked example (analogo Node):** [`api-node-reference/src/modules/resource/resource.controller.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/modules/resource/resource.controller.ts). El patrón es similar — 5 endpoints sobre `/resources`, `@UseGuards(ApiKeyGuard)` ≡ Spring Security filter.

---

### §4. Validación (S5)

- **Tool:** Jakarta Validation (anotaciones en los DTOs).
- **Pattern:** decorás los fields del DTO con `@NotNull`, `@Size(min=1, max=100)`, `@Email`, `@Pattern`, etc. Spring Boot lo valida automáticamente cuando el controller tiene `@Valid` en el parámetro.
- **422 strategy (S5):** Spring Boot devuelve 400 por default en validation errors. Para devolver 422 según el playbook, custom `ExceptionHandler` para `MethodArgumentNotValidException` que devuelva `ResponseEntity.status(422)`.
- **Field-level details:** los errors de Jakarta Validation ya vienen con field-level info en `bindingResult.getFieldErrors()`. Mapealos al envelope.
- **Por qué Jakarta Validation vs custom:** estándar Java EE/Jakarta EE, integración nativa con Spring, ecosistema.

**Worked example (analogo Node):** [`api-node-reference/src/common/pipes/zod-validation.pipe.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/common/pipes/zod-validation.pipe.ts) (custom pipe para 422). Mismo concepto en Java: custom exception handler.

**Checklist S5:** ✓ Jakarta Validation, ✓ custom handler para 422, ✓ field-level details en el envelope.

---

### §5. Error envelope (S6)

- **Shape:** `{ error: { code: string, message: string, details?: unknown } }`.
- **Implementación:** `@ControllerAdvice` global con `@ExceptionHandler` para cada tipo de error (NotFoundException, MethodArgumentNotValidException, etc.).
- **Status → code mapping:** ver tabla en [`spec-template.md` §5](../spec-template.md#5-error-envelope).
- **Spring exceptions:** `NoHandlerFoundException`, `HttpRequestMethodNotSupportedException`, etc. — todas capturadas por el `@ControllerAdvice` global.

**Worked example (analogo Node):** [`api-node-reference/src/common/filters/http-exception.filter.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/common/filters/http-exception.filter.ts) — la lógica es análoga.

**Checklist S6:** ✓ envelope consistente, ✓ status code mapping, ✓ Spring exceptions manejados.

---

### §6. Logging (S7)

- **Stack:** Logback + SLF4J (default de Spring Boot).
- **JSON output:** agregar `logstash-logback-encoder` al classpath y configurar `logback-spring.xml` con el encoder JSON.
- **Config:** `application.yml` o `application.properties` con `logging.level.*` y `logging.pattern.console`.
- **Redaction:** Logback no tiene redaction built-in. Solución: custom `PatternLayout` con regex replacement, o usar `logback-redact` library.
- **JSON en prod, pretty en dev:** profile `prod` usa JSON encoder, `dev` usa el pattern por default.

**Worked example (analogo Node):** [`api-node-reference/src/app.module.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/app.module.ts) (LoggerModule.forRoot). Pino tiene redaction built-in, Logback requiere config adicional.

**Checklist S7:** ✓ JSON a stdout, ✓ redaction de secrets (custom), ✓ pretty en dev.

---

### §7. CORS (S8)

- **Built-in:** `WebMvcConfigurer` bean global con `CorsConfigurationSource` configurable via `application.yml`.
- **O simple:** `@CrossOrigin(origins = "${app.frontend-origin}")` a nivel de controller.
- **Origen configurable:** `app.frontend-origin` env var (default `http://localhost:5173`).

**Worked example (analogo Node):** [`api-node-reference/src/main.ts:17-20`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/main.ts#L17).

**Checklist S8:** ✓ configurable via env, ✓ credentials enabled (si el FE necesita cookies).

---

### §8. Testing (S9)

- **Stack:** JUnit 5 + Mockito + Spring Boot Test (`@SpringBootTest`, `@WebMvcTest`).
- **In-memory DB:** tests usan H2 (compatibilidad JPA) o Testcontainers para PostgreSQL. H2 es más rápido, Testcontainers es más fiel al prod.
- **`@WebMvcTest`:** testing de capa HTTP (controller + validation + filters) sin levantar la DB completa.
- **`@DataJpaTest`:** testing de capa JPA (repositorios) con DB in-memory.
- **MockMvc:** `mockMvc.perform(get("/resources"))` para simular requests HTTP.
- **Sin gotcha de decorators:** Java con annotations es nativo del language, sin tooling extra.

**Worked example (analogo Node):** [`api-node-reference/src/modules/resource/resource.controller.spec.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/modules/resource/resource.controller.spec.ts). El patrón es similar (in-memory DB + re-aplicar cross-cutting en beforeEach).

**Checklist S9:** ✓ integration tests con MockMvc, ✓ in-memory DB o Testcontainers, ✓ @WebMvcTest para capa HTTP.

---

### §9. Lint / format (S10)

- **Tool:** Spotless (format) + SpotBugs (análisis estático). Una config en `pom.xml`.
- **Alternativa moderna:** Spotless solo (con Google Java Format o Palantir) alcanza para v1. SpotBugs para code review.
- **Reemplaza:** Checkstyle + PMD (clásicos, más verbose).
- **Configuración:** plugin Spotless en `pom.xml` con `googleJavaFormat()` o `palantirJavaFormat()`. ~10 líneas.

**Worked example (analogo Node):** [`api-node-reference/biome.json`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/biome.json) — mismo concepto (1 tool, 1 config).

**Checklist S10:** ✓ 1-2 tools, 1 config (en `pom.xml`), ✓ format + análisis estático.

---

### §10. Base de datos (S11, S12)

- **ORM:** Spring Data JPA (Hibernate) (decisión Q5).
- **Engine:** **PostgreSQL** desde el inicio (Q4 — fricciones de Hibernate con SQLite, ver [`architecture-decisions.md`](../architecture-decisions.md)).
  - Dev: `postgresql://user:***@localhost:5432/api_dev` (local Postgres via `brew services` o Docker).
  - Prod: `postgresql://user:***@host:5432/api_prod` (RDS, Supabase, etc.).
- **Migrations (S11):** Flyway, forward-only.
  - Generar: `mvn flyway:migrate` después de modificar un entity. **Spring Boot auto-aplica** las migrations al startup si `flyway-core` está en el classpath.
  - Ubicación: `src/main/resources/db/migration/V<VERSION>__<NAME>.sql`.
  - Tabla interna `flyway_schema_history` trackea cuáles corrieron.
- **Sync (S12):**
  - **Greenfield:** skip.
  - **Existing source DB:** script Java custom (`mvn exec:java -Dexec.mainClass="..."`) o SQL dump/restore. Out of scope para v1.
- **Q4 / Q5:** ver [`architecture-decisions.md`](../architecture-decisions.md) — por qué Postgres (no SQLite), por qué JPA (no jOOQ/MyBatis).

**Worked example (analogo Node):** [`api-node-reference/src/database/`](https://github.com/JonatanAlpirez/api-node-reference/tree/main/src/database). Mismo flujo (generar + aplicar migrations), distinto CLI (Flyway vs MikroORM).

**Checklist S11/S12:** ✓ forward-only migrations, ✓ auto-applied al startup, ✓ DB sync strategy definida.

---

### §11. Deployment (S13, S14)

- **Puerto (S13):** default `8787`, configurable via `SERVER_PORT` env (Spring Boot usa `SERVER_PORT` para Tomcat embebido).
- **Dev mode (S14):** `mvn spring-boot:run` con Spring DevTools activado (auto-reload). O `mvn spring-boot:run -Dspring-boot.run.profiles=dev`.
- **Build:** `mvn clean package` → `target/api.jar` (fat JAR con todo incluido).
- **Prod start:** `java -jar target/api.jar --spring.profiles.active=prod`.
- **Docker:** Eclipse Temurin 21 JRE base, COPY fat JAR, `ENTRYPOINT ["java", "-jar", "/app/api.jar"]`. Opcional pero recomendado.
- **CI:** GitHub Actions matrix JDK 21, `mvn verify` (corre tests + lint). Pendiente de crear.

**Worked example (analogo Node):** [`api-node-reference/package.json`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/package.json). Concepto similar (scripts para dev/build/prod), tools distintas.

**Checklist S13/S14:** ✓ puerto 8787, ✓ dev con auto-reload (DevTools), ✓ prod con fat JAR.

---

### §12. List patterns (S15, S16)

- **Pagination (S15):** offset-based. Spring Data tiene `Pageable` built-in, pero el spec pide `page/limit` (no `page/size` + `sort`). Adaptar: `@RequestParam int page, @RequestParam int limit` + construir `PageRequest.of(page - 1, limit)`.
- **Response shape:** `PaginatedResponse<T>` con `data: List<T>` y `pagination: { page, limit, total, hasNext }` (custom, no el `Page<T>` default de Spring).
- **Filtering (S16):** whitelist cerrada. Usar un enum o un set de strings permitidos en el DTO, validar antes de aplicar el filtro. Spring Data Specification API permite queries dinámicas tipadas.
- **Sorting (S16):** whitelist en el controller, mapping a `Sort.Direction` y nombre de property del entity. Previene SQL injection por columnas arbitrarias.
- **`hasNext`:** calculado como `page * limit < total` en el service (no en el controller).

**Worked example (analogo Node):** [`api-node-reference/src/modules/resource/dto/filter-resource.dto.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/modules/resource/dto/filter-resource.dto.ts). Mismo concepto, distinto syntax (Jakarta Validation vs Zod).

**Checklist S15/S16:** ✓ offset-based pagination, ✓ whitelist cerrada para filter/sort, ✓ mapeo snake_case → property.

---

### §13. Secrets (S18)

- **Spring Boot env:** `application.yml` con `${API_KEY}` placeholders, valores vienen de `process.env` o `.env`.
- **Validación al arranque:** custom `@ConfigurationProperties` class con Jakarta Validation (`@NotBlank`, `@Size(min=16)` para API_KEY). Spring Boot falla en el startup si falta un field.
- **`.env`:** Spring Boot no lee `.env` automáticamente. Opciones:
  - `spring.config.import=optional:file:.env[.properties]` (Spring Boot 2.4+) — lee `.env` como properties file.
  - O externalizar con `APP_ENV_FILE` env var que apunta al path.
- **`.env` gitignored, `.env.example` commiteado:** la secret real nunca va al repo.

**Worked example (analogo Node):** [`api-node-reference/src/config/env.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/config/env.ts) — concepto idéntico, syntax Java más verbosa.

**Checklist S18:** ✓ ConfigurationProperties, ✓ fail-loud al arranque, ✓ `.env` gitignored.

---

### §14. Open questions

No hay tooling Java-specific acá. Si el proyecto tiene preguntas abiertas, listalas en el spec y resolvelas antes/durante implementación.

---

## Por qué este stack (vs Node / Python)

### vs Node + TypeScript (NestJS)

**Ganamos con Java:**
- Type-safety máxima en compile-time (Java es estáticamente tipado, más estricto que TS).
- Ecosystem enterprise (Spring Security, Spring Cloud, etc.) si el proyecto crece.
- Spring Boot + Hibernate + Postgres es el camino más "production-tested" del mercado.
- Records, sealed classes, pattern matching — features modernas de Java 21.

**Perdés con Java:**
- Arranque significativamente más lento (Spring Boot en ~5-10s vs NestJS en ~1s).
- Mucho más boilerplate (no hay equivalente a `nest g resource` para scaffolding).
- Sintaxis más verbosa (anotaciones, tipos explícitos, getters/setters en DTOs).
- Compilación step (compile a bytecode, no ejecución directa).
- Ecosystem más pesado (JVM, dependencias, classpath).

### vs Python (FastAPI)

**Ganamos con Java:**
- Compile-time type safety (Python type hints son opcionales, checked por mypy separado).
- Performance en runtime (JVM JIT vs Python interpreter).
- Spring Data JPA más maduro que SQLAlchemy async para enterprise.
- Mejor tooling de análisis estático (SpotBugs, Error Prone, etc.).

**Perdés con Java:**
- Mucho más boilerplate (Python es conciso, Java verboso).
- Sin build step en Python (más rápido para iterar).
- Sintaxis más flexible en Python (dynamic typing, less ceremony).
- Arranque más lento (5-10s vs <1s).

---

## Stack-specific gotchas (a documentar cuando se cree `api-java-reference`)

10 gotchas que probablemente aparecerán cuando se implemente el reference (a documentar en el `.docs/WALKTHROUGH.md` de `api-java-reference`):

1. **Hibernate lazy loading + Jackson serialization** — `LazyInitializationException` al serializar relaciones. Solución: `@JsonIgnore` o `EntityGraph`.
2. **Spring Boot test slice** — `@WebMvcTest` no carga `@Service` ni `@Repository`, hay que mockear con `@MockBean`.
3. **Flyway migrations inmutable** — no se puede modificar una migration ya aplicada, hay que crear una nueva.
4. **JPA entity equals/hashCode** — usar el ID, no los fields, para evitar infinite recursion en relaciones.
5. **Spring Security 6 lambda DSL** — la config de `SecurityFilterChain` cambió significativamente vs Spring Security 5.

---

## Próximos pasos

- Crear `api-java-reference` (mismo nivel de coverage que `api-node-reference` para Java+Spring Boot 3) — **work pendiente**, ~2-3 días de trabajo (más boilerplate que Python/Node).
- CI + Dockerfile para `api-java-reference` cuando exista.
- Documentar los 10 gotchas específicos de Java en su `.docs/WALKTHROUGH.md` (a crear con el reference).

Para el stack Java en sí, este proposal ya está listo para guiar la implementación de un proyecto nuevo que cumpla S1-S16 + S18.
