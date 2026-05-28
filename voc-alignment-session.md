# VOC Alignment Session — Technical Analysis

**Service:** `internal-service` (Spring Boot 3 / Java 17)
**Related:** `investments-soap-proxy`
**Date:** 2026-05-28

> **Note:** There are currently zero MCI or VOC references in `internal-service`. All four tickets represent net-new work.

---

## Existing Data Surface (Reference)

| Domain | Key Fields |
|---|---|
| Investor / Bio | `partyId`, `name`, `firstName`, `lastName`, `title`, `gender`, `birthDate`, `language`, address fields |
| Holding | `contractNumber`, `productCode`, `investmentValue`, `inceptionDate`, fund details, withdrawal pots |
| Broker | `brokerCode`, `brokerHouseCode` |

---

## Ticket 1 — MCI API Fields Requirement: Technical Analysis

**Current state:** No MCI integration exists. Existing field coverage is limited to what Oracle and Data Platform currently return.

**Technical questions to clarify:**
- Which MCI API version/spec is being targeted? Is there a Swagger/WSDL available?
- Which existing fields map directly to MCI field names, and what is the gap (net-new fields)?
- Does MCI require a new datasource — Oracle proc, Data Platform endpoint, or a new external API?
- Is the integration push (MCI calls us) or pull (we call MCI)?

**Expected output of this ticket:** A field mapping matrix showing existing fields, MCI field names, source, and any gaps requiring new data sourcing.

---

## Ticket 2 — MCI Bio Data API Integration

**Current state:** Bio data exists in `Investor`/`Party` domain objects sourced from Oracle (`AstuteOracleRepository`) and Data Platform (`DataPlatformRepositoryBean`). No outbound MCI call exists.

**Expected work:**
- New `MciClient` / `WebClient` bean — pattern already exists in `OAuth2ClientConfig` for Data Platform
- New service layer (e.g. `MciService`) following the `controller -> service -> repository` convention
- New REST endpoint or enrichment of the existing Astute response
- Auth scheme to confirm: OAuth2 client credentials (same as Data Platform) or different?

**Risk:** If MCI bio data overlaps with existing Oracle bio data, a comparison/reconciliation logging pattern may be needed — the same pattern is already used for Oracle-vs-DataPlatform in `ComparisonLoggingService`.

---

## Ticket 3 — VOC Contract Routing

**Current state:** Contract routing is handled in two places:

- **REST:** `ContractController` → `ContractService`, keyed on `brokerCode` / `brokerHouseCode`
- **SOAP:** `InternalService.performWithArgList` dispatches on the `perform` string (e.g. `getPortfolioForAstuteByIdNumber:`)

**Expected work:**
- New routing rule(s) in `InternalService` SOAP dispatch or a new `@RequestMapping` in `ContractController`
- Routing criteria must be defined: by product type, contract prefix, or client segment?
- If routing targets `investments-soap-proxy`, the proxy URL and auth config must be added to `application.yml`

**Key question:** Is VOC contract routing about directing certain contract types to a different downstream (e.g. `investments-soap-proxy` vs. current Oracle path), or about exposing a new contract lookup endpoint?

---

## Ticket 4 — VOC Enrichment and Publish

**Current state:** Enrichment exists — `DataPlatformServiceBean` aggregates holdings with withdrawal pots, sub-accounts, and retirement components via parallel `CompletableFuture`. However, **there is no event publishing** — no Kafka, no messaging bus, no async outbound events.

**Expected work:**
- **If publish = event-driven (Kafka/RabbitMQ):** Net-new infrastructure — broker config, producer beans, schema/topic design, and potentially a Flyway migration for an outbox table if transactional guarantees are needed
- **If publish = synchronous API push:** New `WebClient` call to a downstream, following the existing Data Platform pattern
- Enrichment scope: which additional fields must be merged before publishing, and from which source?

**Dependency:** MCI bio data (Ticket 2) likely feeds the enrichment step before publish.

---

## Effort and Risk Summary

| Ticket | Effort | Key Blocker / Unknown |
|---|---|---|
| MCI API fields analysis | Low–Medium | Need MCI API spec / contract |
| MCI bio data integration | Medium | Auth scheme, field mapping gap |
| VOC contract routing | Medium | Routing criteria definition |
| VOC enrichment and publish | High | Publish mechanism (event vs. sync), topic/schema design |

---

## Cross-Cutting Risk

The `investments-soap-proxy` likely holds the SOAP contracts and field definitions that drive MCI. Confirming what that proxy exposes will unblock the field analysis (Ticket 1) and directly inform all downstream tickets.
