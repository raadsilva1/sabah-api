# AGENT_DOTNET.md

## 1. Executive summary

`sabah-api` is a monolithic SBCL REST backend that exposes health, authentication, media-presign, and full CRUD APIs for a therapist-centered domain with 12 DynamoDB-backed entities. The implementation follows a layered structure (`Controller -> Service -> Repository + DTOs + Builders`) and runs with environment-aware configuration (`dev`/`prd`), Cognito JWT validation, and AWS integrations for DynamoDB and S3.

What it does, evidenced by repository code and contracts:
- Serves HTTP endpoints under `/api/v1` via Hunchentoot (`src/http/server.lisp`, `src/http/router.lisp`).
- Implements CRUD for 12 resources (`src/controllers/*`, `src/services/*`, `src/repositories/*`).
- Validates and verifies JWTs (RS256 via Cognito JWKS, HS256 for challenge-issued session tokens) (`src/auth/jwt.lisp`, `src/auth/cognito.lisp`, `src/auth/guards.lisp`).
- Issues S3 presigned upload URLs for therapist photo/video media (`src/services/media-service.lisp`, `src/repositories/s3-client.lisp`, `src/bootstrap/aws.lisp`).
- Persists to DynamoDB through a custom SigV4 + JSON API client and supports `memory` mode fallback (`src/repositories/dynamodb-client.lisp`, `src/bootstrap/aws.lisp`).
- Provides shell-driven build/deploy/stress workflows (`build.zsh`, `dynamo-schema-deploy.zsh`, `stress-test.zsh`).

High-level architecture:
- Runtime bootstrap: config -> logging -> AWS init -> repository mode -> HTTP server (`src/bootstrap/runtime.lisp`).
- Request path: router -> auth guard -> controller -> service -> repository -> result envelope (`src/http/router.lisp`, `src/http/errors.lisp`, `src/http/responses.lisp`).
- Data shape: entity definitions and GSI metadata in code (`src/domain/constants.lisp`) and expanded schema/constraints in docs (`dynamo-schema.md`, `docs/internal/entity-inventory.md`).

Major technical characteristics:
- Language/runtime: Common Lisp (SBCL), ASDF system (`sabah-api.asd`).
- Persistence target: DynamoDB with soft-delete semantics (`removal_date`) in repository logic.
- Auth: Cognito issuer/client checks + JWKS cache; challenge endpoint generates temporary session tokens.
- Ops/tooling: shell-centric with heavy dependency on OS binaries and CLI tools.

Migration target mandated by assignment:
- .NET 10 + F# + NuGet.
- Official Microsoft .NET libraries/patterns.
- Official AWS SDK for .NET for AWS services in use.
- PowerShell to replace current shell automation.
- EF Core only where applicable. For DynamoDB runtime persistence in this repository, EF Core is **not applicable**.

---

## 2. Repository inventory

### Root-level files
- `sabah-api.asd`: ASDF manifest and load order for all source modules.
- `qlfile`: Quicklisp dependency list (`cl-ppcre`, `yason`, `hunchentoot`, `log4cl`, `dexador`, `ironclad`).
- `README.md`: operational overview, build and stress commands, policy notes.
- `conf.yaml`: active runtime configuration example.
- `config/sabah-api.example.conf.yaml`: configuration template.
- `build.zsh`: native SBCL build script + forbidden language embedding scan.
- `stress-test.zsh`: end-to-end stress/consumption script, reporting, media upload validation.
- `dynamo-schema-deploy.zsh`: AWS CLI DynamoDB provisioning script for 12 tables.
- `dynamo-schema.md`: canonical DynamoDB schema translation and access pattern rules.
- `agent.md`: master implementation specification (SBCL/OpenBSD/AWS constraints).
- `execute.md`: current migration assignment requirements.
- `LICENSE`, `.gitignore`: legal and VCS metadata.

### Major directories
- `src/`: application source.
- `docs/`: master spec, CRUD contract, and internal design/spec-map docs.
- `tests/`: unit/integration/contract test plans (non-executable plans).
- `dist/`: built binary plus stress report artifacts.

### Source modules under `src/`
- `src/main.lisp`, `src/package.lisp`: entrypoint and package definitions.
- `src/bootstrap/*`: runtime startup, config, logging, cache, AWS setup.
- `src/http/*`: server start/stop, routing, response envelope, error handling, middleware principal holder.
- `src/auth/*`: JWT parsing/verification, Cognito JWKS retrieval, auth guards, role/group helpers.
- `src/controllers/*`: auth/health handlers and CRUD controller wrappers.
- `src/services/*`: business validation/orchestration for auth/media and all entities.
- `src/repositories/*`: generic DynamoDB/memory repo client, S3 helper, per-entity repository wrappers.
- `src/dtos/*`: DTO shaping functions.
- `src/builders/*`: builder mapping functions.
- `src/domain/*`: entity/GSI metadata, enums, validators.
- `src/support/*`: ids, time, strings, result envelopes, pagination, error type.

### Docs inventory
- `docs/sabah-api-master-spec.md`: expanded project spec and architecture expectations.
- `docs/sabah-api-crud-contracts.md`: endpoint contract and validation matrix.
- `docs/internal/entity-inventory.md`: entity-level inventory extracted from schema/deploy files.
- `docs/internal/dynamodb-access-patterns.md`: expected access patterns and transaction templates.
- `docs/internal/auth-design.md`: auth design intent.
- `docs/internal/auth-definitive-spec-map.md`: Cognito-focused definitive auth spec map.

### Tests inventory
- `tests/unit/validators-test.lisp`: placeholder comments only.
- `tests/integration/api-integration-test-plan.md`: plan-only checklist.
- `tests/contract/contract-test-plan.md`: plan-only checklist.

### CI/CD and deployment definitions
- CI pipeline config files: `Not evidenced in repository`.
- Container definitions (`Dockerfile`, `docker-compose`, k8s manifests): `Not evidenced in repository`.

---

## 3. Architectural overview

### Layering and boundaries (implemented)
- Controller layer: thin wrappers from HTTP route to service (`src/controllers/*.lisp`, `src/http/router.lisp`).
- Service layer: validation, derived fields, auth-challenge orchestration, media request validation (`src/services/*.lisp`).
- Repository layer: memory and DynamoDB modes, generic CRUD, filter/list logic (`src/repositories/dynamodb-client.lisp`).
- DTO/Builder layers: mostly pass-through mapping boundaries, plus auth/media DTO shaping (`src/dtos/*.lisp`, `src/builders/*.lisp`).
- Cross-cutting: config, cache, auth, response/error envelope, logging, ids/time utilities.

### Runtime composition model
Startup sequence (`src/bootstrap/runtime.lisp`):
1. `load-config`
2. `configure-logging`
3. `initialize-aws`
4. `configure-repository-mode`
5. `start-http-server`

Entrypoint (`src/main.lisp`):
- Parses CLI flags (`--config`, `--check-config`, `--print-effective-config`).
- Starts runtime.
- Optionally prints config/check result.
- Sleeps in loop.

### HTTP runtime flow
1. Hunchentoot prefix dispatcher `/api/v1` invokes `api-v1-handler` (`src/http/router.lisp`).
2. Route match:
   - health/auth/media fixed paths, or
   - generic entity route (`/api/v1/<resource>[/id]`).
3. Route-level auth enforcement via `require-authenticated` or `require-role`.
4. Controller/service call.
5. Result normalized to envelope (`api-success`/`api-error`) with request id (`src/http/responses.lisp`).
6. Global error handling maps known `sabah-error` or returns generic 500 envelope (`src/http/errors.lisp`).

### Dependency direction
- HTTP depends on controllers/services/auth support.
- Services depend on validators, dto/builder functions, repositories.
- Repositories depend on domain constants/config/aws utils.
- Support/domain are foundational.
- No dependency injection container; dependency graph is mostly static via function calls.

### Cross-cutting concerns
- Error model: typed `sabah-error` with code/details/http status (`src/support/errors.lisp`).
- Response envelope standardization (`src/http/responses.lisp`).
- In-memory cache with TTL and mutex (`src/bootstrap/cache.lisp`).
- Logging: minimal `log4cl` info/error logs (`src/bootstrap/logging.lisp`, `src/http/errors.lisp`, `src/bootstrap/aws.lisp`).
- Resilience:
  - In runtime code: limited retry logic; mostly fail-fast.
  - In deployment script: explicit retry with backoff (`dynamo-schema-deploy.zsh`).
  - In stress script: retry loops for challenge and uploads (`stress-test.zsh`).

### Background processing / messaging
- Queues, workers, schedulers, event bus: `Not evidenced in repository`.
- DynamoDB Streams consumers/Lambda implementation: referenced in docs, implementation `Not evidenced in repository`.

---

## 4. Functional inventory

### Public/system endpoints
- `GET /api/v1/health/live`
- `GET /api/v1/health/ready`
- `GET /api/v1/auth/health`

### Authenticated endpoints
- `GET /api/v1/auth/me`
- `POST /api/v1/auth/challenge/request`
- `POST /api/v1/auth/challenge/solve`
- `POST /api/v1/media/presign-upload`
- All mutating CRUD methods (`POST`, `PUT`, `PATCH`, `DELETE`) for entity resources.

### Admin-protected reads
- `GET /api/v1/therapist-logins[...]`
- `GET /api/v1/admin-logins[...]`
- `GET /api/v1/access-websites[...]`

### CRUD resources (12)
- `therapists`
- `specialty-therapists`
- `therapist-characteristics`
- `therapist-reviews`
- `therapist-photos`
- `therapist-videos`
- `therapist-logins`
- `admin-logins`
- `therapist-schedules`
- `access-websites`
- `temporary-client-sessions`
- `temporary-session-client-requests`

Implemented operations per resource via generic router dispatch:
- `GET /api/v1/<resource>`
- `POST /api/v1/<resource>`
- `GET /api/v1/<resource>/{id}`
- `PUT|PATCH /api/v1/<resource>/{id}`
- `DELETE /api/v1/<resource>/{id}`

List-query features:
- `filter_field`, `filter_value`, `include_removed`, optional `limit`.

### Domain behaviors implemented in services
- Therapist: requires `full_name`; validates `age` range.
- Specialty therapist: requires parent id; validates `experience_years` range.
- Therapist characteristic: requires parent id; validates `foot_size` range.
- Therapist review: requires parent id; validates `stars` 1..5.
- Therapist photo/video: requires parent id and path field.
- Therapist/admin login: requires phone + password hash.
- Therapist schedule: requires parent id; validates date/time parts; derives `slot_ts`.
- Access website: auto-fills `access_date` if absent; derives `access_day`.
- Temporary client session: requires `identification_key`; defaults `expires_at` and `expires_at_epoch`.
- Temporary session client request: requires session/schedule ids; validates enum or defaults status `pending`.

### Auth challenge behavior
- Generates challenge with nonce, difficulty, expiry, max attempts.
- Validates PoW answer by SHA-256 prefix rule.
- On success, issues short-lived HS256 session token derived from private source token and Cognito config.

### Media behavior
- Validates entity/media kind/content type/size.
- Returns S3 presigned PUT URL + bucket/key metadata.

### Operational features
- Build script: SBCL native executable generation + forbidden cross-language embedding scan.
- Deploy script: creates 12 DynamoDB tables with GSIs, PITR, SSE, streams, TTL, retries.
- Stress script: preflight, challenge auth acquisition, full CRUD and media upload paths, retention policy, endpoint coverage and latency ranking report.

---

## 5. Source model inventory

### Canonical entity catalog (from code + schema docs)

1. `sabah_therapist`
- PK: `id`
- Core fields: `full_name`, `age`, `sexual_orientation`, `description`, `creation_date`, `last_updated_date`, `removal_date`, `general_lock_date`
- GSI: none

2. `sabah_specialty_therapist`
- PK: `id`
- Fields: `sabah_therapist_id`, `specialty_name`, `experience_years`, `creation_date`, `last_updated_date`, `removal_date`
- GSI: `gsi_therapist_id (sabah_therapist_id, id)`

3. `sabah_therapist_characteristic`
- PK: `id`
- Fields: `sabah_therapist_id`, `skin_tone`, `hair_tint`, `height`, `foot_size`, `creation_date`, `last_updated_date`, `removal_date`
- GSI: `gsi_therapist_id (sabah_therapist_id, id)`

4. `sabah_therapist_review`
- PK: `id`
- Fields: `sabah_therapist_id`, `description`, `stars`, `creation_date`, `removal_date`
- GSI: `gsi_therapist_id (sabah_therapist_id, creation_date)`

5. `sabah_therapist_photo`
- PK: `id`
- Fields: `sabah_therapist_id`, `image_path`, `creation_date`, `removal_date`
- GSI: `gsi_therapist_id (sabah_therapist_id, id)`

6. `sabah_therapist_video`
- PK: `id`
- Fields: `sabah_therapist_id`, `video_path`, `creation_date`, `removal_date`
- GSI: `gsi_therapist_id (sabah_therapist_id, id)`

7. `sabah_therapist_login`
- PK: `id`
- Fields: `sabah_therapist_id`, `phone_number`, `password_hash`, `creation_date`, `removal_date`
- GSIs: `gsi_therapist_id`, `gsi_phone_number`

8. `sabah_admin_login`
- PK: `id`
- Fields: `phone_number`, `password_hash`, `creation_date`, `removal_date`
- GSI: `gsi_phone_number`

9. `sabah_therapist_schedule`
- PK: `id`
- Fields: `sabah_therapist_id`, `day`, `month`, `year`, `hour`, `minute`, `slot_ts`, `creation_date`, `acceptance_date`, `refusal_date`, `payment_date`, `stipulated_value`
- GSI: `gsi_therapist_id (sabah_therapist_id, slot_ts)`

10. `sabah_access_website`
- PK: `id`
- Fields: `user_agent`, `access_date`, `access_day`, `full_url`
- GSI: `gsi_access_day (access_day, access_date)`

11. `sabah_temporary_client_session`
- PK: `id`
- Fields: `identification_key`, `phone_number`, `name`, `expires_at`, `expires_at_epoch`, `creation_date`
- GSI: `gsi_identification_key (identification_key, id)`
- TTL marker: `expires_at_epoch`

12. `sabah_temporary_session_client_request`
- PK: `id`
- Fields: `sabah_temporary_client_session_id`, `sabah_therapist_schedule_id`, `creation_date`, `removal_date`, `request_status`, `comments`
- GSIs: `gsi_session_id`, `gsi_schedule_id`

### DTOs and contracts
- DTO functions are currently thin pass-through wrappers except:
  - `auth-me-response-dto` (claims projection)
  - `media-presign-request-dto` (explicit request command shape)

### Validation rules implemented in code
- Numeric range checks: age, experience years, foot size, stars, schedule parts.
- Enum check: `request_status` values `pending|accepted|refused|cancelled|completed`.
- ID shape check: 32 hex chars for `md6_128_id` contract.

### Contract/documentation mismatches detected
- Documentation expects transactional uniqueness sentinels and FK checks in DynamoDB transactions; runtime repository code currently uses per-item conditional create/update only. This is a migration risk and must be resolved in target implementation.

---

## 6. Authentication and authorization

### Mechanisms implemented
- Bearer extraction from `Authorization` header.
- JWT parse from compact JWS segments.
- Signature verification:
  - RS256 via Cognito JWKS key lookup (`kid`).
  - HS256 for challenge-session tokens.
- Claims checks:
  - `exp` (expiration)
  - `iss` equals configured Cognito issuer
  - `aud` or `client_id` equals configured app client id
  - `token_use` in configured enum (`access`, `id`)
- Role/group checks via `cognito:groups`.

### Auth flows
1. Direct Cognito token flow:
- Client presents Cognito JWT.
- API verifies RS256 signature and claims.

2. Challenge session flow:
- `POST /auth/challenge/request`: server returns challenge.
- Client solves computational challenge.
- `POST /auth/challenge/solve`: server validates solution, issues HS256 temporary access token.
- HS256 signing secret derives from private challenge source token + issuer + client id.

### Route policy (implemented)
- Public: health endpoints, auth health, challenge request/solve (for acquisition), entity reads except selected resources.
- Auth required: all writes and media presign.
- Admin read requirement: `therapist-logins`, `admin-logins`, `access-websites`.

### Security boundaries and implications
- Positive:
  - Explicit auth errors and fail-on-invalid signature.
  - JWKS URL enforced as HTTPS in config validation.
  - Token not logged directly by runtime code.
- Risks:
  - `allow_unsigned_dev` can bypass signature checks in dev (`src/auth/jwt.lisp`).
  - Mixed acceptance of RS256 and HS256 in same verifier path increases policy complexity.
  - No explicit `nbf`/`iat` skew validation logic evidenced.

### .NET 10 + F# definitive mapping
- Use ASP.NET Core authentication middleware with explicit schemes:
  - `CognitoJwt` scheme for RS256 JWT bearer validation.
  - `ChallengeSessionJwt` scheme for HS256 challenge-session tokens.
- Use `JwtBearerHandler` and custom `TokenValidationParameters` + key resolver.
- Maintain explicit challenge source token dependency from environment/secret store; never return source token in responses.
- Preserve current auth error code vocabulary in problem payload mapping.

---

## 7. Configuration and environment model

### Current settings source and precedence
- First existing config path:
  1. `/etc/apis/sabah/conf.yaml`
  2. `./conf.yaml`
  3. `/tmp/sabah/conf.yaml`
- Current parser: external `yq` process invocation from runtime (`uiop:run-program` in `src/bootstrap/config.lisp`).

### Validated required settings (implemented)
- App: `env`, `host`, `port`
- AWS: `region`, S3 buckets, Cognito issuer/client/jwks
- Logging: `logging.level`
- Cache: `jwks_ttl_seconds`, `reference_ttl_seconds`
- HTTP: timeout, max request body
- Pagination: default/max limits

### Environment model
- Supported values: `dev`, `prd` only.
- Table name resolution: `<base_table>_<env>`.
- S3 bucket resolution:
  - `dev -> aws.s3.bucket_dev`
  - `prd -> aws.s3.bucket_prd`

### Secrets handling evidenced
- AWS credentials loaded from:
  - env vars (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, optional session token)
  - shared credentials/config files under `~/.aws`.
- Challenge source token loaded from config or env var (`SABAH_AUTH_SOURCE_TOKEN` by default).

### Feature flags and toggles
- `repository.mode` (`memory` or `dynamodb`)
- `auth.allow_unsigned_dev`
- `aws.dynamodb.strong_read_get`
- `aws.dynamodb.strong_read_scan`

### .NET 10 + F# mapping
- Official config stack:
  - `Microsoft.Extensions.Configuration`
  - `Microsoft.Extensions.Configuration.Json`
  - `Microsoft.Extensions.Configuration.EnvironmentVariables`
  - `Microsoft.Extensions.Configuration.CommandLine`
- Strongly typed options via `IOptions<T>`.
- Preserve current fallback precedence semantics.
- YAML format compatibility with only official Microsoft stack: `Not evidenced in repository` as directly supported; migration should use JSON config files (or implement internal parser in F# if strict YAML compatibility is required).

---

## 8. Persistence and data access

### Current persistence model
- Repository mode switch:
  - `:memory` in-process hash table.
  - `:dynamodb` via custom SigV4 JSON API calls.

### CRUD behavior in repositories
- Create:
  - ensures/validates id format.
  - adds timestamps.
  - Dynamo conditional create uses `attribute_not_exists(id)`.
- Get:
  - single-item lookup by `id`.
- Update:
  - conditional `attribute_exists(id)`.
  - updates `last_updated_date`.
- Delete:
  - soft-delete by default (`removal_date` + `last_updated_date`).
  - optional hard-delete path in generic function.
- List:
  - if filter field maps to known GSI, query by GSI.
  - else table scan.
  - optional include-removed and limit truncation.

### Transactions and integrity
- Runtime code uses conditional writes and updates.
- Multi-item transactions (`TransactWriteItems`) for uniqueness sentinel/FK checks: `Not evidenced in repository implementation`.
- Schema docs specify transaction templates; runtime currently diverges.

### Query behavior
- Consistency flags configurable separately for get vs scan.
- Filtering logic re-applies application-level filter after query/scan.
- Cursor-based pagination: placeholder only (`src/support/pagination.lisp`), no tokenized pagination flow implemented in repository.

### Migrations and ORM
- Database migration framework in code: `Not evidenced in repository`.
- ORM usage in current code: `Not evidenced in repository`.

### EF Core target mapping
- Runtime persistence is DynamoDB; EF Core is not the primary runtime access mechanism here.
- EF Core applicability for current runtime persistence: **Not applicable**.
- If relational projections/reporting stores are introduced later, EF Core can be applied there; such relational store is `Not evidenced in repository`.

### .NET 10 + F# data-access target
- Use official AWS SDK for .NET `AWSSDK.DynamoDBv2` with `IAmazonDynamoDB`.
- Implement repository interfaces in F# using low-level request models (`GetItemRequest`, `PutItemRequest`, `UpdateItemRequest`, `QueryRequest`, `ScanRequest`, `TransactWriteItemsRequest` where required).
- Preserve soft-delete and include-removed semantics.
- Add missing transaction-based uniqueness/FK guarantees mandated by docs.

---

## 9. AWS integration mapping

| AWS service | Current usage purpose | Current implementation evidence | Official .NET package(s) | Recommended .NET 10 + F# integration pattern |
|---|---|---|---|---|
| DynamoDB | Primary persistence for all 12 entities | `src/repositories/dynamodb-client.lisp`, `src/bootstrap/aws.lisp`, `dynamo-schema.md`, `dynamo-schema-deploy.zsh` | `AWSSDK.Core`, `AWSSDK.DynamoDBv2` | Register `IAmazonDynamoDB` via AWS SDK default credential chain; implement typed F# repositories with request objects and conditional/transactional writes. |
| S3 | Presigned PUT URL issuance for photo/video uploads | `src/repositories/s3-client.lisp`, `src/services/media-service.lisp`, `src/bootstrap/aws.lisp`, `stress-test.zsh` | `AWSSDK.Core`, `AWSSDK.S3` | Use `IAmazonS3.GetPreSignedURL` (or equivalent) with explicit content-type and expiration; keep bucket-by-env routing and object-key convention. |
| Cognito User Pools | JWT issuer/audience model and JWKS key source for RS256 verification | `src/auth/jwt.lisp`, `src/auth/cognito.lisp`, `conf.yaml`, docs under `docs/internal/auth-*` | `System.IdentityModel.Tokens.Jwt`, `Microsoft.IdentityModel.Tokens`, `Microsoft.IdentityModel.Protocols.OpenIdConnect`, `AWSSDK.CognitoIdentityProvider` (for administrative ops/workflows) | Use JWT bearer auth with issuer/audience validation and JWKS retrieval/caching; for provisioning/admin flows use CognitoIdentityProvider SDK in dedicated tools/services. |
| AWS shared credentials chain | Runtime credential resolution | `src/bootstrap/aws.lisp` | `AWSSDK.Core` | Use `FallbackCredentialsFactory` / default AWS SDK credential chain; remove custom credentials-file parser. |
| DynamoDB Streams | Documented for cascade patterns | `dynamo-schema.md`, `docs/internal/entity-inventory.md`, deploy script stream settings | `AWSSDK.DynamoDBv2` (stream config mgmt), AWS Lambda runtime packages if consumer is added | Consumer implementation is `Not evidenced in repository`; if required, implement separate .NET Lambda/worker project using official AWS .NET runtime. |
| SSM Parameter Store | Documented in auth spec map for auth config persistence | `docs/internal/auth-definitive-spec-map.md` | `AWSSDK.SimpleSystemsManagement` | Runtime use is `Not evidenced in repository`; if adopted, load secure config via SSM in bootstrap tooling/service layer. |

---

## 10. Build, restore, test, compile, publish, and deployment model

### Current developer/ops workflows (evidenced)
- Dependency model:
  - ASDF system manifest (`sabah-api.asd`) + Quicklisp deps (`qlfile`).
- Build:
  - `build.zsh` checks toolchain and scans forbidden patterns.
  - Builds native executable with SBCL `save-lisp-and-die` into `dist/sabah-api`.
- Runtime:
  - executable starts server and loops indefinitely.
- Stress test:
  - `stress-test.zsh` uses `curl`, `jq`, `sort`, `cksum`, `sha256`, `dd`, `mktemp`; performs endpoint coverage, media uploads, retention checks, and report generation.
- Infrastructure deploy:
  - `dynamo-schema-deploy.zsh` uses AWS CLI for table lifecycle and settings.
- Tests:
  - executable automated test suite: `Not evidenced in repository`.
  - test plans exist as markdown docs.
- CI/CD:
  - `Not evidenced in repository`.

### Direct .NET 10 + NuGet + PowerShell replacement approach

#### Build and publish
- Replace `build.zsh` with `build.ps1`:
  - `dotnet restore`
  - `dotnet build -c Release`
  - `dotnet test -c Release`
  - `dotnet publish src/Sabah.Api/Sabah.Api.fsproj -c Release -o dist`
- Use SDK-style `*.fsproj` and NuGet package references only.

#### Runtime configuration
- Replace runtime YAML+`yq` with official .NET configuration providers (JSON/env/command-line).
- Preserve path precedence behavior.

#### Stress test
- Replace `stress-test.zsh` with a native F# console project (`Sabah.StressTest`) using:
  - `HttpClient`
  - `System.Text.Json`
  - async concurrency and structured markdown report writer.
- No `curl`, `jq`, or OS utility dependencies.

#### Dynamo deploy workflow
- Replace `dynamo-schema-deploy.zsh` with PowerShell using official AWS tools:
  - `AWS.Tools.DynamoDBv2`
  - optional `AWS.Tools.Common`.
- Preserve: idempotency checks, retries, PITR, SSE, streams, TTL, tagging, env suffixing.

#### CI/CD target
- Pipeline definitions are `Not evidenced in repository`; recommended baseline is PowerShell-driven `dotnet` commands and AWS deployment scripts, then integrate into chosen CI platform.

---

## 11. 1:1 migration matrix

Each row includes required fields: source path(s), responsibility, collaborators, role, target F# equivalent, official package(s), and behavior-preservation notes.

### 11.1 Root, config, docs, tests, and operations resources

| Source path(s) | Current responsibility | Dependencies/collaborators | Current role | Target .NET 10 + F# equivalent | Official package(s) required | Migration notes (behavior-preserving) |
|---|---|---|---|---|---|---|
| `sabah-api.asd` | System manifest/load order | SBCL/ASDF, `src/*` | Build composition | `Sabah.sln` + multiple `*.fsproj` projects | .NET SDK, NuGet | Preserve module boundaries as project references; replace load-order coupling with compile-time references. |
| `qlfile` | Dependency declaration | Quicklisp libs | Package manifest | `Directory.Packages.props` + `PackageReference` | NuGet | Replace Lisp deps with official .NET/AWS packages. |
| `.gitignore` | VCS ignore policy | Git tooling | Repository hygiene policy | `.gitignore` retained and updated for .NET artifacts (`bin/`, `obj/`, publish output) | None | Preserve ignore intent while adding .NET build outputs. |
| `LICENSE` | Project license text | Legal/distribution | Compliance artifact | `LICENSE` retained as-is | None | No behavioral migration; keep licensing unchanged. |
| `README.md` | Runtime/build/stress usage docs | Scripts + routes | Operator guide | Updated .NET README | None | Preserve command intent with `dotnet`/PowerShell equivalents. |
| `conf.yaml`, `config/sabah-api.example.conf.yaml` | Runtime configuration | `src/bootstrap/config.lisp` | Environment configuration source | `appsettings.json` + `appsettings.Development.json` + `appsettings.Production.json` + env vars | `Microsoft.Extensions.Configuration.*` | Preserve keys and precedence; YAML format compatibility is a migration decision (see risks). |
| `build.zsh` | Build binary + policy scan | `sbcl`, `curl`, `jq`, regex scanners | Build automation | `build.ps1` + optional `dotnet tool` analyzer step | .NET SDK, PowerShell | Preserve policy check intent (forbidden embeddings) using .NET analyzer/PowerShell regex scan. |
| `stress-test.zsh` | Full consumption test + report + latency ranking + media upload | API endpoints, `curl`, `jq`, `dd` | Performance and integration workload driver | `Sabah.StressTest` F# console app | `Microsoft.Extensions.Http`, `System.Text.Json` | Preserve endpoint coverage matrix, per-item request/response report, retention ratio checks, ranking model. |
| `dynamo-schema-deploy.zsh` | DynamoDB table provisioning and settings | AWS CLI | Infra deploy automation | `deploy-dynamodb.ps1` or F# deploy tool | `AWS.Tools.DynamoDBv2` or `AWSSDK.DynamoDBv2` | Preserve table names, GSIs, PITR, SSE, streams, TTL, retries, tags, env suffix. |
| `dynamo-schema.md` | Canonical schema/access contract | Docs and deploy script | Data model source-of-truth | `docs/AGENT_DOTNET_SCHEMA.md` (ported spec) | None | Preserve constraints and access patterns; implement missing transaction patterns in code. |
| `agent.md` | Master architecture/process spec | Entire repo | Governance spec | `docs/AGENT_DOTNET_SPEC.md` | None | Preserve non-functional rules that remain relevant after stack migration. |
| `execute.md` | Migration assignment rules | N/A | Migration contract | Incorporated into this document | None | Requirement fulfilled by this `AGENT_DOTNET.md`. |
| `docs/sabah-api-master-spec.md` | Architecture and process specification | `dynamo-schema*`, source tree | Contractual specification | .NET-focused master spec | None | Keep source-of-truth precedence model; update stack-specific sections. |
| `docs/sabah-api-crud-contracts.md` | API contract | Router/controllers/services | API behavior contract | OpenAPI + markdown contract | `Microsoft.AspNetCore.OpenApi` | Preserve envelope/error codes/routes and auth rules. |
| `docs/internal/entity-inventory.md` | Detailed entity inventory | schema + deploy docs | Internal modeling guide | `docs/internal/entity-inventory-dotnet.md` | None | Preserve attribute-level inventory and unresolved markers. |
| `docs/internal/dynamodb-access-patterns.md` | Access pattern map and transaction templates | schema/deploy docs | Data access design guide | `docs/internal/dynamodb-access-patterns-dotnet.md` | None | Convert templates to AWS SDK for .NET request examples. |
| `docs/internal/auth-design.md` | Auth design outline | auth modules | Internal security design | `docs/internal/auth-design-dotnet.md` | None | Preserve issuer/audience/token_use requirements and fail-closed behavior. |
| `docs/internal/auth-definitive-spec-map.md` | Cognito provisioning and auth hardening map | AWS CLI and auth runtime | Definitive auth rollout guide | `docs/internal/auth-definitive-spec-map-dotnet.md` + PowerShell commands | `AWS.Tools.CognitoIdentityProvider`, `AWS.Tools.SimpleSystemsManagement` | Preserve dev/prd isolation and validation matrix. |
| `tests/unit/validators-test.lisp` | Placeholder unit tests | validators | Test placeholder | `tests/Sabah.UnitTests` with MSTest | `MSTest.TestFramework`, `MSTest.TestAdapter`, `Microsoft.NET.Test.Sdk` | Implement real validator coverage currently missing. |
| `tests/integration/api-integration-test-plan.md` | Integration test checklist | API routes | Plan-only artifact | Executable integration tests (`Sabah.IntegrationTests`) | `Microsoft.AspNetCore.Mvc.Testing`, MSTest | Convert checklist to automated tests. |
| `tests/contract/contract-test-plan.md` | Contract test checklist | CRUD docs | Plan-only artifact | Contract tests against OpenAPI + response snapshots | MSTest, `Microsoft.AspNetCore.Mvc.Testing` | Preserve envelope and auth contract assertions. |
| `dist/sabah-api`, `dist/stress-report-*.md` | Build artifact and generated reports | build/stress scripts | Runtime artifact output | `dist/Sabah.Api` publish output + generated stress reports | .NET SDK | Generated artifacts are not source; keep equivalent output directory conventions. |

### 11.2 Core runtime and cross-cutting source mapping

| Source path | Current responsibility | Dependencies/collaborators | Current role | Target .NET 10 + F# equivalent | Official package(s) required | Migration notes |
|---|---|---|---|---|---|---|
| `src/package.lisp` | Package imports/exports | Entire codebase | Namespace boundary | F# namespaces + project references | .NET SDK | Replace dynamic package exports with explicit modules/signatures. |
| `src/main.lisp` | CLI arg parse and runtime boot loop | bootstrap runtime/config | Process entrypoint | `Program.fs` (`Host.CreateDefaultBuilder`) | `Microsoft.Extensions.Hosting` | Preserve `--check-config` and effective config print behavior via command options. |
| `src/bootstrap/runtime.lisp` | Startup sequence and lifecycle | config/logging/aws/repository/http | Runtime orchestrator | `StartupComposition.fs` + hosted lifetime hooks | `Microsoft.Extensions.Hosting`, `Microsoft.Extensions.DependencyInjection` | Preserve startup ordering and fail-fast semantics. |
| `src/bootstrap/config.lisp` | Config load + validation + helpers | `yq`, support strings/domain constants | Config subsystem | `Configuration.fs` + options validators | `Microsoft.Extensions.Configuration.*`, `Microsoft.Extensions.Options` | Remove external `yq` process dependency; preserve validation rules and path precedence. |
| `src/bootstrap/logging.lisp` | Log level normalization | log4cl | Logging bootstrap | Built-in logging config in host | `Microsoft.Extensions.Logging` | Preserve level mapping and structured log intent. |
| `src/bootstrap/cache.lisp` | TTL cache with mutex | support/time | Shared cache for JWKS/challenges | `IMemoryCache` wrappers | `Microsoft.Extensions.Caching.Memory` | Preserve TTL behavior and atomic update/get semantics. |
| `src/bootstrap/aws.lisp` | AWS credentials resolution, custom SigV4 for DynamoDB/S3 presign | Dexador, Ironclad, config | AWS integration core | `AwsClients.fs` and service-specific clients | `AWSSDK.Core`, `AWSSDK.DynamoDBv2`, `AWSSDK.S3` | Replace custom signing code with AWS SDK request pipeline. |
| `src/http/server.lisp` | Start/stop Hunchentoot acceptor | router/config | HTTP host adapter | ASP.NET Core Kestrel host | `Microsoft.AspNetCore.App` | Preserve host/port binding from config. |
| `src/http/router.lisp` | Route dispatch, body parse, auth enforcement | controllers/auth/responses | API routing gateway | ASP.NET Core endpoint routing + controllers/minimal APIs | `Microsoft.AspNetCore.App` | Preserve endpoint set and method/resource policy logic. |
| `src/http/responses.lisp` | JSON envelope and request id | yason, hunchentoot | Transport response formatting | `ApiEnvelope<'T>` + global result helpers | `System.Text.Json`, `Microsoft.AspNetCore.Http` | Preserve `data/meta/error` schema and request id behavior. |
| `src/http/errors.lisp` | Error mapping and global 500 handling | sabah-error/logging | Exception translation boundary | Exception middleware + problem/envelope translator | `Microsoft.AspNetCore.Diagnostics` | Preserve explicit domain error codes and generic internal error for unhandled exceptions. |
| `src/http/middleware.lisp` | request principal holder | auth guards | Request context helper | ClaimsPrincipal and scoped context service | `Microsoft.AspNetCore.Authentication` | Replace global dynamic var with per-request DI context. |
| `src/auth/cognito.lisp` | Cognito config/JWKS fetch/cache | cache, dexador, config | Identity provider client | JWKS metadata fetch service + cache | `Microsoft.IdentityModel.Protocols.OpenIdConnect`, `Microsoft.Extensions.Caching.Memory` | Preserve HTTPS enforcement and cache TTL rules. |
| `src/auth/jwt.lisp` | JWT parsing, signature verify, claims checks, challenge-token issuance | cognito, support, ironclad | Token security core | `JwtSecurityTokenHandler` + custom challenge token service | `System.IdentityModel.Tokens.Jwt`, `Microsoft.IdentityModel.Tokens` | Preserve claim checks and challenge-token semantics; split RS256 and HS256 schemes explicitly. |
| `src/auth/guards.lisp` | Require-auth and require-role guards | jwt/roles | Authorization guard | ASP.NET authorization policies/handlers | `Microsoft.AspNetCore.Authorization` | Preserve role and auth error semantics. |
| `src/auth/roles.lisp` | Claims role/group extraction | hash helpers | Authorization helper | Claims extension helpers | `System.Security.Claims` | Preserve `cognito:groups` usage. |
| `src/domain/constants.lisp` | Supported envs/entity metadata/GSIs | repositories/config | Domain metadata registry | `DomainConstants.fs` + typed records | .NET SDK | Preserve entity registry and GSI metadata as authoritative source in code. |
| `src/domain/enums.lisp` | Enum value lists | validators/auth | Domain constraints | F# discriminated unions + parse/validate | .NET SDK | Preserve accepted status/token_use value sets. |
| `src/domain/validators.lisp` | Numeric/enum validations | support errors/enums | Domain validation | Validation module returning typed errors | .NET SDK | Preserve current 422 validation behavior. |
| `src/support/errors.lisp` | Domain error type | all layers | Error contract | Domain exception/Result error DU | .NET SDK | Preserve code/message/details/http-status structure. |
| `src/support/result.lisp` | Success/error result wrappers | controllers/services/repos | Application result contract | `Result<'T, ApiError>` style DU | .NET SDK | Preserve consistent success/error propagation. |
| `src/support/strings.lisp` | trim/blank helpers | parsing/validation | Utility | String utility module | .NET SDK | Direct transliteration. |
| `src/support/time.lisp` | Unix and ISO time helpers | services/repos/cache | Utility | `DateTimeOffset` helper module | .NET SDK | Preserve UTC formatting and day-bucket derivation. |
| `src/support/ids.lisp` | id generation/shape validation | time/crypto | Identity utility | `IdGenerator.fs` | `System.Security.Cryptography` | Preserve 32-hex contract and document algorithm mismatch resolution (MD6 label vs SHA-256 derivation). |
| `src/support/pagination.lisp` | parse limit/cursor placeholders | router/repos | Pagination utility | `Pagination.fs` with real cursor token strategy | .NET SDK | Preserve current limit behavior; implement full cursor flow if required. |
| `src/repositories/dynamodb-client.lisp` | Generic CRUD for memory/DynamoDB, list/filter logic | bootstrap aws/config, domain constants | Persistence core | `IRepository` + `DynamoRepository` + `MemoryRepository` | `AWSSDK.DynamoDBv2` | Preserve soft delete, filters, strong read flags, and add missing transaction invariants from docs. |
| `src/repositories/s3-client.lisp` | object-key generation + presign result | aws/config/support ids | Media persistence helper | `S3MediaService.fs` | `AWSSDK.S3` | Preserve key format and expires/bucket behavior. |
| `src/controllers/auth-controller.lisp` | auth health/me/challenge handlers | auth service/results | HTTP controller | `AuthController.fs` | `Microsoft.AspNetCore.Mvc` | Preserve endpoints and response envelope/error mapping. |
| `src/controllers/health-controller.lisp` | liveness/readiness handlers | config/aws/repository/id strategy | HTTP controller | `HealthController.fs` + health checks | `Microsoft.Extensions.Diagnostics.HealthChecks` | Preserve readiness payload fields. |
| `src/services/auth-service.lisp` | challenge generation/verification and me endpoint | auth/jwt/cache/result | Auth business service | `AuthService.fs` | `System.Security.Cryptography`, `Microsoft.Extensions.Caching.Memory` | Preserve challenge difficulty/ttl/attempt constraints and mutex-equivalent synchronization. |
| `src/services/media-service.lisp` | presign validation and command orchestration | auth guard, s3 client | Media business service | `MediaService.fs` | `AWSSDK.S3` | Preserve content-type/size/media-kind validation and auth requirement. |
| `src/dtos/auth-dtos.lisp`, `src/builders/auth-builders.lisp` | Auth response DTO projection and builder mapping | auth roles/claims | Contract-mapping support for auth slice | `AuthContracts.fs` + auth mapping module | .NET SDK | Preserve exact `/auth/me` payload shape (`subject`, `issuer`, `token_use`, `groups`). |
| `src/dtos/media-dtos.lisp`, `src/builders/media-builders.lisp` | Media presign command DTO and builder | media service | Contract-mapping support for media slice | `MediaContracts.fs` + media mapping module | .NET SDK | Preserve presign request shape and field names expected by clients and tests. |

### 11.3 Entity stack migration mapping (all CRUD resources)

Each row maps the full stack for one entity resource: controller + service + repository wrapper + DTO + builder.

| Source path set | Current responsibility | Dependencies/collaborators | Current role | Target .NET 10 + F# equivalent | Official package(s) required | Migration notes |
|---|---|---|---|---|---|---|
| `src/controllers/therapist-controller.lisp`; `src/services/therapist-service.lisp`; `src/repositories/therapist-repository.lisp`; `src/dtos/therapist-dtos.lisp`; `src/builders/therapist-builders.lisp` | Therapist CRUD and age/name validation | Generic repo client, validators | Core domain CRUD vertical slice | `TherapistsController.fs`, `TherapistService.fs`, `TherapistRepository.fs`, contracts module | `Microsoft.AspNetCore.Mvc`, `AWSSDK.DynamoDBv2` | Preserve validation (`full_name`, `age` range), soft-delete list behavior, id contract. |
| `src/controllers/specialty-therapist-controller.lisp`; `src/services/specialty-therapist-service.lisp`; `src/repositories/specialty-therapist-repository.lisp`; `src/dtos/specialty-therapist-dtos.lisp`; `src/builders/specialty-therapist-builders.lisp` | Specialty CRUD + parent id/experience years validation | Therapist relationship, generic repo | Child entity CRUD slice | `SpecialtyTherapists*` modules | `Microsoft.AspNetCore.Mvc`, `AWSSDK.DynamoDBv2` | Preserve `experience_years` validation and GSI-based filtering by therapist id. |
| `src/controllers/therapist-characteristic-controller.lisp`; `src/services/therapist-characteristic-service.lisp`; `src/repositories/therapist-characteristic-repository.lisp`; `src/dtos/therapist-characteristic-dtos.lisp`; `src/builders/therapist-characteristic-builders.lisp` | Characteristics CRUD + parent id/foot size validation | Generic repo, validators | Child entity CRUD slice | `TherapistCharacteristics*` modules | `Microsoft.AspNetCore.Mvc`, `AWSSDK.DynamoDBv2` | Preserve optional attributes and foot-size constraints. |
| `src/controllers/therapist-review-controller.lisp`; `src/services/therapist-review-service.lisp`; `src/repositories/therapist-review-repository.lisp`; `src/dtos/therapist-review-dtos.lisp`; `src/builders/therapist-review-builders.lisp` | Reviews CRUD + stars validation | Generic repo, validators | Review CRUD slice | `TherapistReviews*` modules | `Microsoft.AspNetCore.Mvc`, `AWSSDK.DynamoDBv2` | Preserve stars 1..5 and chronological query access pattern. |
| `src/controllers/therapist-photo-controller.lisp`; `src/services/therapist-photo-service.lisp`; `src/repositories/therapist-photo-repository.lisp`; `src/dtos/therapist-photo-dtos.lisp`; `src/builders/therapist-photo-builders.lisp` | Photo metadata CRUD + required `image_path` | Generic repo, media flow | Media metadata CRUD slice | `TherapistPhotos*` modules | `Microsoft.AspNetCore.Mvc`, `AWSSDK.DynamoDBv2`, `AWSSDK.S3` | Preserve metadata + separate upload flow pattern. |
| `src/controllers/therapist-video-controller.lisp`; `src/services/therapist-video-service.lisp`; `src/repositories/therapist-video-repository.lisp`; `src/dtos/therapist-video-dtos.lisp`; `src/builders/therapist-video-builders.lisp` | Video metadata CRUD + required `video_path` | Generic repo, media flow | Media metadata CRUD slice | `TherapistVideos*` modules | `Microsoft.AspNetCore.Mvc`, `AWSSDK.DynamoDBv2`, `AWSSDK.S3` | Preserve metadata + separate upload flow pattern. |
| `src/controllers/therapist-login-controller.lisp`; `src/services/therapist-login-service.lisp`; `src/repositories/therapist-login-repository.lisp`; `src/dtos/therapist-login-dtos.lisp`; `src/builders/therapist-login-builders.lisp` | Therapist login CRUD + required phone/password_hash | Generic repo, admin read policy | Sensitive credential CRUD slice | `TherapistLogins*` modules | `Microsoft.AspNetCore.Mvc`, `AWSSDK.DynamoDBv2`, `Microsoft.AspNetCore.Authorization` | Preserve admin-read policy; implement transaction-based phone uniqueness as specified in docs. |
| `src/controllers/admin-login-controller.lisp`; `src/services/admin-login-service.lisp`; `src/repositories/admin-login-repository.lisp`; `src/dtos/admin-login-dtos.lisp`; `src/builders/admin-login-builders.lisp` | Admin login CRUD + required phone/password_hash | Generic repo, admin scope | Sensitive credential CRUD slice | `AdminLogins*` modules | `Microsoft.AspNetCore.Mvc`, `AWSSDK.DynamoDBv2`, `Microsoft.AspNetCore.Authorization` | Preserve admin protection and enforce phone uniqueness transactionally. |
| `src/controllers/therapist-schedule-controller.lisp`; `src/services/therapist-schedule-service.lisp`; `src/repositories/therapist-schedule-repository.lisp`; `src/dtos/therapist-schedule-dtos.lisp`; `src/builders/therapist-schedule-builders.lisp` | Schedule CRUD + slot derivation and date-time validation | Validators, generic repo | Scheduling CRUD slice | `TherapistSchedules*` modules | `Microsoft.AspNetCore.Mvc`, `AWSSDK.DynamoDBv2` | Preserve `slot_ts` derivation and valid ranges for day/month/hour/minute. |
| `src/controllers/access-website-controller.lisp`; `src/services/access-website-service.lisp`; `src/repositories/access-website-repository.lisp`; `src/dtos/access-website-dtos.lisp`; `src/builders/access-website-builders.lisp` | Access log CRUD + `access_day` derivation | Generic repo | Access logging CRUD slice | `AccessWebsites*` modules | `Microsoft.AspNetCore.Mvc`, `AWSSDK.DynamoDBv2` | Preserve derived `access_day` and admin-read protection. |
| `src/controllers/temporary-client-session-controller.lisp`; `src/services/temporary-client-session-service.lisp`; `src/repositories/temporary-client-session-repository.lisp`; `src/dtos/temporary-client-session-dtos.lisp`; `src/builders/temporary-client-session-builders.lisp` | Temporary session CRUD + expiry defaults | Generic repo/time helpers | TTL-backed session CRUD slice | `TemporaryClientSessions*` modules | `Microsoft.AspNetCore.Mvc`, `AWSSDK.DynamoDBv2` | Preserve `expires_at`/`expires_at_epoch` defaulting and TTL semantics. |
| `src/controllers/temporary-session-client-request-controller.lisp`; `src/services/temporary-session-client-request-service.lisp`; `src/repositories/temporary-session-client-request-repository.lisp`; `src/dtos/temporary-session-client-request-dtos.lisp`; `src/builders/temporary-session-client-request-builders.lisp` | Join-entity CRUD + status validation/default | Validators, generic repo | Session-request workflow CRUD slice | `TemporarySessionClientRequests*` modules | `Microsoft.AspNetCore.Mvc`, `AWSSDK.DynamoDBv2` | Preserve status enum/default and parent reference validations. |

---

## 12. Recommended .NET 10 + F# architecture

### Target solution structure

- `src/Sabah.Api` (ASP.NET Core web API, entrypoint, HTTP transport)
- `src/Sabah.Application` (use cases/services, result types, orchestration)
- `src/Sabah.Domain` (entities, value objects, validation rules, id/time policies)
- `src/Sabah.Infrastructure.Aws` (DynamoDB, S3, Cognito/JWKS integrations)
- `src/Sabah.Contracts` (DTOs/envelopes/openapi contracts)
- `tools/Sabah.StressTest` (native F# stress driver and report generator)
- `tools/deploy` (PowerShell AWS deployment scripts)
- `tests/Sabah.UnitTests`
- `tests/Sabah.IntegrationTests`
- `tests/Sabah.ContractTests`

### Project boundaries and dependency direction
- `Sabah.Api` -> `Sabah.Application`, `Sabah.Contracts`
- `Sabah.Application` -> `Sabah.Domain`, abstraction interfaces
- `Sabah.Infrastructure.Aws` implements interfaces from `Sabah.Application`
- `Sabah.Domain` has no infrastructure dependencies

### Configuration model
- Use options classes with validation at startup.
- Providers: JSON + env vars + command line.
- Preserve current required keys and environment semantics.

### DI pattern
- Built-in `IServiceCollection` registrations in composition root.
- Register AWS clients via standard AWS SDK extensions.
- Register per-request auth/context services and singleton caches.

### Data access pattern
- Repository interfaces in application layer.
- DynamoDB repository implementations with explicit request builders.
- Add transactional write paths for uniqueness/FK invariants that docs require.
- Preserve soft delete and filter semantics.

### AWS integration pattern
- `IAmazonDynamoDB` for CRUD/query/transactions.
- `IAmazonS3` for presign and media bucket routing.
- Cognito JWT validation through JWT middleware + JWKS metadata retrieval and cache.
- Use AWS SDK credential chain only; remove custom file parsing and custom SigV4 code.

### Logging, errors, resilience
- Structured logging with built-in logging abstractions.
- Global exception handler returning existing envelope/error code format.
- HTTP client resilience for JWKS retrieval using official .NET resilience components.
- Health checks endpoint preserving current readiness fields.

### EF Core position
- EF Core is not the primary persistence mechanism for DynamoDB-backed runtime behavior in this repository.
- EF Core can be introduced only if a relational component is added; such component is `Not evidenced in repository`.

---

## 13. Risks, incompatibilities, and migration blockers

1. External runtime config parser dependency (`yq`) in application bootstrap
- Evidence: `src/bootstrap/config.lisp` invokes `uiop:run-program` with `yq` commands.
- Why it matters: violates target requirement to avoid shell-dependent runtime utilities.
- Mitigation in allowed stack: replace with official .NET configuration providers; migrate YAML to JSON or implement internal parser without external process invocation.

2. Custom SigV4 implementation for DynamoDB/S3 in app code
- Evidence: `src/bootstrap/aws.lisp` manually builds canonical request/signature.
- Why it matters: high maintenance/security risk versus official SDK.
- Mitigation: use `AWSSDK.DynamoDBv2` and `AWSSDK.S3` clients directly.

3. Documented transactional invariants are not implemented in runtime repository
- Evidence: docs require uniqueness sentinel/FK transaction templates (`dynamo-schema.md`, `docs/internal/dynamodb-access-patterns.md`); runtime repo only uses conditional single-item operations (`src/repositories/dynamodb-client.lisp`).
- Why it matters: potential integrity violations under concurrency.
- Mitigation: implement `TransactWriteItems` paths for uniqueness and FK checks.

4. ID strategy naming mismatch (MD6 contract vs SHA-256 truncation)
- Evidence: `src/support/ids.lisp` comment states MD6 unavailable and uses SHA-256-derived 32 hex.
- Why it matters: semantic mismatch with stated requirement/contract text.
- Mitigation: formalize strategy name and contract in .NET code/spec; keep deterministic 32-hex behavior unless true MD6 implementation is explicitly required.

5. Optional unsigned-token bypass in dev
- Evidence: `allow-unsigned-dev-p` in `src/auth/jwt.lisp` can skip RS256 verification.
- Why it matters: security drift risk and environment bleed if misconfigured.
- Mitigation: remove bypass in final .NET implementation; keep only strict verified-token path.

6. Mixed JWT algorithm handling in one verifier path
- Evidence: `verify-signature-or-fail` supports RS256 and HS256 in `src/auth/jwt.lisp`.
- Why it matters: increases attack surface if scheme boundaries are unclear.
- Mitigation: separate authentication schemes and enforce endpoint policy by scheme.

7. Stress workflow depends on many OS binaries/utilities and shell semantics
- Evidence: `stress-test.zsh` requires `curl`, `jq`, `sort`, `cksum`, `sha256`, `dd`; also embeds generated Lisp solver script.
- Why it matters: portability/maintainability risk; violates target non-shell runtime policy for application components.
- Mitigation: move stress engine to native F# tool using `HttpClient`/`System.Text.Json`; keep PowerShell only for orchestration if needed.

8. Automated tests are mostly non-existent
- Evidence: `tests/unit/validators-test.lisp` is placeholder; integration/contract tests are markdown plans.
- Why it matters: migration safety and regression detection are weak.
- Mitigation: implement unit/integration/contract suites in .NET before and during migration.

9. CI/CD definition absent
- Evidence: no pipeline manifests in repository.
- Why it matters: release repeatability and verification gates are undefined.
- Mitigation: establish PowerShell-based pipeline steps (`restore/build/test/publish/deploy`) in chosen CI platform.

10. Docs indicate additional AWS operational paths not implemented in runtime
- Evidence: auth spec map includes SSM and extensive Cognito CLI provisioning; runtime code does not consume SSM and does not call Cognito admin APIs.
- Why it matters: migration scope ambiguity.
- Mitigation: keep runtime behavior strictly evidence-based; implement ops tooling separately and document boundary explicitly.

---

## 14. Assumptions and non-evidenced areas

- CI/CD platform and release pipeline configuration: `Not evidenced in repository`.
- Containerization strategy (Docker/Kubernetes): `Not evidenced in repository`.
- Background workers/Lambda consumers for DynamoDB Streams: `Not evidenced in repository`.
- Production-grade observability stack (metrics/tracing export destination): `Not evidenced in repository`.
- Relational database usage requiring EF Core in current runtime: `Not evidenced in repository`.
- Formal backward-compatibility requirements for stress report schema beyond current script output: `Not evidenced in repository`.
- Explicit SLA/SLO latency objectives encoded in code or tests: `Not evidenced in repository`.
- Secret manager integration in runtime (e.g., SSM/Secrets Manager retrieval): `Not evidenced in repository`.
- Deployment topology details (load balancer, autoscaling, networking, WAF): `Not evidenced in repository`.
- Any additional endpoints beyond those routed in `src/http/router.lisp`: `Not evidenced in repository`.
