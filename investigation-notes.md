# Investigation Notes — Oracle vs Data Platform Validation

## What I Was Trying To Do

Trigger the Astute REST endpoint with a client ID number so that the service fetches from both Oracle and Data Platform, logs both responses to the `astute_call_logger` Postgres table, and I can query the results.

---

## Attempt 1 — Get Auth Token

**What I tried:**
```bash
POST https://auth.momentuminv-dev.co.za/oauth/token
grant_type=client_credentials
client_id=VzoeQuA4rplbYsR90coDgES5AJxwK5TS
client_secret=usGSvmt-wniQP4hZ-PbIWk6AuqsS8sXuqe_ghDNGXOyUeok7nmLiIpZZdq-W-gGe
scope=read:astute
audience=api://dm-datamesh
```

**Result:** SUCCESS — got a valid `access_token` back.

**Problem discovered later:** This token is issued by **Auth0** and is meant for calling the **Data Platform** directly. The `internal-service` does not accept it — it only accepts tokens from **AWS Cognito**.

---

## Attempt 2 — Hit the Astute Endpoint (wrong URL)

**What I tried:**
```bash
POST https://internal-dev-investments.mmiholdings.com/astute/
Authorization: Bearer <auth0_token>
Body: {"idNumber": "6202135212088"}
```

**Result:** FAILED — `HTTP 404 Not Found` from nginx.

**Root cause:** The URL was missing the nginx routing prefix. Found the nginx config at `configuration/nginx/nginx.yml`:

```yaml
internal-service:
  internal:
    proxy:
      - location: /internal-service
        upstreamPath: /
```

This means nginx only routes requests that start with `/internal-service` to the Spring Boot app.

---

## Attempt 3 — Hit the Astute Endpoint (correct URL, wrong token)

**What I tried:**
```bash
POST https://internal-dev-investments.mmiholdings.com/internal-service/astute/
Authorization: Bearer <auth0_token>
Body: {"idNumber": "6202135212088"}
```

**Result:** FAILED — `HTTP 403 Forbidden`.

**Root cause:** The Auth0 token is rejected by the service. Found in `src/main/resources/application.yml` line 68:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://cognito-idp.eu-west-1.amazonaws.com/eu-west-1_P4Kyb2BGb
```

The service only accepts JWTs issued by **AWS Cognito** (`eu-west-1_P4Kyb2BGb`), not Auth0. The Auth0 token has a different `iss` claim and is rejected immediately.

---

## What Is Confirmed Working

| Thing | Status |
|---|---|
| Auth0 token endpoint | Works — token is returned successfully |
| Correct service URL | `https://internal-dev-investments.mmiholdings.com/internal-service/astute/` |
| Postgres connection details | Provided by senior — not yet tested |
| Request body shape | `{"idNumber": "..."}` — confirmed from `AstuteIDNumberRequest.java` |

---

## What Is the Data Platform and What Data Is Being Compared?

### Data Platform — Not a Database

The Data Platform is an **external REST API**, not a direct database connection. The service calls it via Spring `WebClient` (synchronous `.block()`) authenticating with OAuth2 client credentials (Auth0) plus a custom `audience` parameter configured in `OAuth2ClientConfig`.

The comparison is therefore:
> **Oracle** (direct DB — SQL/stored procedures returning nested XML) vs **Data Platform** (external REST API returning JSON)

Both sources are expected to return the same portfolio data. The goal of the comparison logging is to validate parity before any cutover from Oracle to Data Platform.

### Portfolio Data Fields Being Compared

| Domain | Key Fields |
|---|---|
| **Holdings** | `CONTRACT_STATUS`, `INCEPTION_DATE`, `PP_MAS_INVESTMENT_VALUE`, `P_LIFE_TYPE`, `P_LEGAL_WRAP`, `P_CO_LICENSE`, fund info |
| **Investor / Party** | `PP_FIRSTNAME`, `PP_NAME`, `PP_GENDER`, `PP_BIRTH_DATE`, `PP_TITLE`, address fields, `PP_CMS_NO` |
| **Sub-Accounts / Funds** | Fund name, `AS_UNITS`, `AS_AMOUNT`, `AS_PRICE`, `AS_PERC`, currency |
| **Two-Pot / Retirement** | `vested_amount`, `non_vested_amount`, `retirement_amount`, `savings_amount` |
| **Withdrawals** | Annual withdrawal total, lifetime total, last withdrawal date |
| **Arrangements** | Debit order type, amount, frequency, next execution date |

### Structural Difference Between the Two Sources

- **Oracle** — one stored proc/SQL call returns the entire portfolio as a single nested XML blob, parsed via JAXB
- **Data Platform** — multiple fine-grained REST calls (`/astute/allHoldings`, `/astute/roleplayers`, `/astute/subAccounts2potComponents`, etc.) stitched together in `DataPlatformRepositoryBean`

The Data Platform responses are reshaped back into the same Oracle/JAXB domain models before being returned, so the SOAP wire format to callers remains unchanged regardless of which source is used.

---

## What Is Blocked

Cannot trigger the endpoint without a valid Cognito token. The Cognito client credentials are stored in AWS Secrets Manager at:

```
/dm/dev/internal-service/auth0-client/internal-service/secret
```

These are not in any config file in the repo.

---

## Questions to Ask Your Senior

1. **Can you give me the Cognito `client_id` and `client_secret` for the dev environment?**
   - The token URL is: `https://invdev.auth.eu-west-1.amazoncognito.com/oauth2/token`
   - Without this I cannot call the service and trigger the comparison logging

2. **Is the Auth0 token you gave me meant for calling this service, or just for the Data Platform?**
   - Based on the code it looks like Auth0 is only used for outbound Data Platform calls, not for inbound requests to this service

3. **Is there already data in the `astute_call_logger` table for these client IDs from normal production traffic?**
   - If the service is already handling live traffic, the comparison logs may already exist and I may just need to query the Postgres DB without triggering any new requests

4. **Do you have AWS console access or CLI access I can use to retrieve the secret from Secrets Manager?**
   - Secret path: `/dm/dev/internal-service/auth0-client/internal-service/secret`

5. **Should I be running this against dev, tst, or pre?**
   - You gave me credentials for all four environments — just want to confirm which one to use for this validation exercise
