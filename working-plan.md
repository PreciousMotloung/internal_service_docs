# 10-Day Working Plan — `internal-service`

## Day 1 — Environment Setup & Local Run
- Configure Docker Compose and bring up local Postgres instance
- Wire up Oracle credentials and Consul config for local profile
- Boot the application and confirm all Spring contexts load without errors
- Hit key endpoints (Astute, Contract, Token) manually to verify baseline behaviour

## Day 2 — SOAP Interceptor (`InternalService.java`)
- Read and understand the full `performWithArgList` routing logic
- Write unit tests for each routing branch: oracle stored proc, Astute intercept, Gemstone proxy fallback
- Refactor any deeply nested conditionals into clearly named private methods
- Add meaningful log statements at each routing decision point

## Day 3 — Gemstone Proxy (`GemstoneService`)
- Write unit tests for the HTTP proxy logic using a mocked `WebClient`
- Implement proper error handling for Gemstone timeouts and 5xx responses (fallback, retry, or circuit breaker)
- Remove any dead code paths or commented-out blocks
- Ensure request/response headers are forwarded correctly

## Day 4 — Oracle Data Layer (`AstuteRepository`)
- Write unit tests for all Oracle queries using an in-memory or embedded test DB
- Fix any N+1 query patterns — batch or join where appropriate
- Remove the commented-out `/show-secret` debug endpoint from `ContractController`
- Add `@Transactional(readOnly = true)` where missing on read-only query methods

## Day 5 — Data Platform (`dp`) Module & Two-Pot Logic
- Write unit tests for `WebClient` calls to `dm-datamesh` with mocked responses
- Add null-safety and fallback handling for missing `SavingsAccount`, `RetirementAccount`, `VestedAccount`, `NonVestedAccount` components
- Implement retry logic on transient failures for Data Mesh calls
- Validate Auth0 token acquisition and caching — fix any token refresh edge cases

## Day 6 — Astute OLifE XML Builder
- Write unit tests for the XML mapping layer covering happy path and missing-field edge cases
- Fix any null-safety issues in the OLifE XML construction
- Validate generated XML output against the Astute schema using a schema validator in tests
- Align any fields that diverge from `astute-oracle-mapping.md`

## Day 7 — REST Controllers & Input Validation
- Add `@Valid` and appropriate constraint annotations to all controller request models
- Implement a global `@ControllerAdvice` exception handler for consistent error response structure
- Add missing `@Operation` / `@ApiResponse` Swagger annotations to all endpoints
- Write controller-layer unit tests (MockMvc) for validation and error response behaviour

## Day 8 — Security Hardening
- Write tests asserting that protected endpoints reject requests without valid JWT tokens
- Fix JWT validation logic for token expiry and issuer/audience mismatches
- Ensure `SecretManagerService` never logs secret values — add tests to assert this
- Verify Cognito and Auth0 OAuth2 flows work end-to-end in the local environment

## Day 9 — Datadog APM Instrumentation
- Add Datadog Java agent dependency and configure it in the `Dockerfile`
- Annotate key service methods (`AstuteService`, `GemstoneService`, `dp` calls) with custom span traces
- Inject trace IDs into all log statements for log correlation
- Add a custom metric for Astute aggregation duration and Gemstone proxy latency

## Day 10 — Integration Tests & EKS Readiness
- Write end-to-end integration tests for the Astute aggregation flow and Gemstone proxy fallback
- Run the full test suite and fix all failing tests
- Update `Dockerfile` `HEALTHCHECK` to use the Spring Actuator `/health` endpoint
- Verify all config keys are externalized and not hardcoded — flag any that would break in EKS
