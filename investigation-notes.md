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
