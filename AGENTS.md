# AGENTS — How to be productive in this codebase

Checklist (read first)
- [ ] Understand this is a Micronaut (Java 17) service wired with Gradle (see `build.gradle`).
- [ ] Auth is performed by a global server filter that extracts `userId` from a JWT and places it on each request (see `JwtClaimsFilter`).
- [ ] External user data and auth tokens are fetched via Micronaut HTTP clients (`KindeClient`, `KindeAccessTokenService`).
- [ ] MongoDB is the canonical persistence with Micronaut Data repositories and explicit client sessions for transactions (`GameMongoRepository`, `GameServiceImpl`).

Big picture
- This is a Micronaut HTTP API application with a small domain for running/recording game matches.
  - Entry point: `project.snake.backend.Application` (Micronaut run).
  - Controllers live under `game/controller` and `user/controller`. Example: `GameController` exposes `/game` routes and reads `userId` from the request attribute set by the JWT filter.
  - Domain/services:
    - Match execution: `GameExecutionServiceImplementation` decides a winner, updates a user's rank via `UserService`, persists a `GameResult` via `GameService`.
    - Persistence: `GameServiceImpl` uses a `MongoClient` session + `GameMongoRepository` to save games within a transaction.

Auth & cross-service conventions (important)
- JWT verification is performed in `core/auth/inbound/filter/JwtClaimsFilter.java`:
  - Expects Authorization: Bearer <token>
  - Validates issuer matches `micronaut.jwt.jwk.url` from `application.yml` and verifies token signature using JWKS (com.auth0 library).
  - On success sets request attribute `userId` = JWT `sub` claim. Controllers access this with `@RequestAttribute("userId")`.
- Access tokens for outbound service calls come from `KindeAccessTokenService` which posts to an OAuth token endpoint defined under the `oauth-client` HTTP client in `application.yml`.
- HTTP clients are declared as Micronaut `@Client("user-service-client")` or `@Client("oauth-client")` and those base URLs are configured in `src/main/resources/application.yml` (or via environment variables when running with Docker Compose).

Persistence & data flow
- MongoDB is configured in `application.yml` under `mongodb.servers.default.uri` and the repo is annotated `@MongoRepository(databaseName = "project-snake")`.
- Repositories use Micronaut Data (`GameMongoRepository`) and return domain/entity objects.
- `GameServiceImpl` wraps repository operations in `ClientSession` transactions (important pattern to preserve when modifying save semantics).

Mapping & DTOs
- MapStruct is used for mapping DTO/entity/domain objects. Key mappers:
  - `game/mapper/GameMapper` (componentModel = "jakarta")
  - `user/mapper/UserMapper`
- Mappers are relied on by services and controllers (no manual mapping code in service layers).

Notable implementation patterns and conventions
- Concurrency: Controllers often use `@ExecuteOn(TaskExecutors.BLOCKING)` to run blocking operations off the event loop (see `GameController`). When adding blocking I/O, follow this pattern.
- Dependency injection: `jakarta.inject.Inject` / constructor injection is used consistently (no field injection except Lombok-generated accessors); prefer constructor injection when adding components.
- Error handling: Service methods often catch exceptions and return Optional/empty (see `KindeUserService#getUserById`). Keep side effects explicit.
- Configuration via environment: `application.yml` references environment variables (e.g. `${DB_USER}`, `${KINDE_CLIENT_ID}`); when running in Docker Compose the project expects a `.env` file (see `docker-compose.yml` env_file). Always surface new config keys in both `application.yml` and the `.env` used for local Docker runs.

Build / run / test commands
- Local build (Java/Gradle):
  - Build: `./gradlew build`
  - Run locally (dev): `./gradlew run` (Micronaut application plugin; main class `project.snake.backend.Application` is configured in `build.gradle`).
  - Tests: `./gradlew test`
- Docker-compose (recommended for full stack: Mongo + Redis):
  - `docker compose up --build` (project README and `docker-compose.yml` expect a `.env` in repo root with DB/Redis/ports/credentials)
- When adding new HTTP client config, update `src/main/resources/application.yml` and ensure the same env keys are present in the `.env` used by Docker Compose.

Environment variables & files to know
- `application.yml` reads:
  - `micronaut.http.services.user-service-client.url` and `oauth-client.url` (base URLs for remote Kinde/auth)
  - `micronaut.jwt.jwk.url` (JWKS issuer URL)
  - `mongodb.servers.default.uri` (uses `${DB_USER}` and `${DB_USER_PWD}`)
  - `kinde.client-url`, `kinde.client-id`, `kinde.client-secret`, `kinde.audience`
- `docker-compose.yml` uses `.env` and expects keys such as `DB_ADMIN`, `DB_ADMIN_PWD`, `DB_PORT`, `DB_USER`, `DB_USER_PWD`, `REDIS_PASSWORD`, `SERVER_PORT`, `KINDE_*` variants. Inspect the repo root for `.env` or create one to run compose.

Quick navigational examples (use these file samples when making changes)
- To see how user identity is passed to controllers: open `core/auth/inbound/filter/JwtClaimsFilter.java` and `game/controller/GameController.java`.
- To see how external user calls are performed: open `user/client/KindeClient.java` and `user/service/KindeUserService.java`.
- To understand persistence + transactions: open `game/repository/GameMongoRepository.java` and `game/service/GameServiceImpl.java`.
- To add mapping: follow `game/mapper/GameMapper.java` and `user/mapper/UserMapper.java` using MapStruct and `componentModel = "jakarta"`.


