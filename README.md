# FraudRuleEngineServiceCollection

Postman collection for testing the local `FraudRuleEngineService` APIs.

The collection is intentionally small and reviewer-focused. It exercises the same flow described in the service README:

1. Confirm the service is healthy.
2. Confirm Swagger/OpenAPI is available locally.
3. Submit the high-risk transaction example.
4. Submit the duplicate event example to prove idempotency.
5. Retrieve fraud alerts.
6. Retrieve one fraud alert by `alertId`.
7. Retrieve the stored transaction fraud evaluation.
8. Run a low-risk evaluation sample.
9. Check common validation and security failures.
10. Check missing-resource `404` responses.
11. Check fraud alert pagination and filter edge cases.
12. Check protected actuator endpoints with an analyst token.
13. Check the alternate transaction evaluation path.
14. Check API-level rule and scoring scenarios for each fraud rule.

## Files

- [postman/FraudRuleEngineService.postman_collection.json](postman/FraudRuleEngineService.postman_collection.json)
- [postman/FraudRuleEngineService.local.postman_environment.json](postman/FraudRuleEngineService.local.postman_environment.json)

## Prerequisites

- Docker running locally.
- `FraudRuleEngineService` cloned locally.
- Postman desktop app, Postman web app, or Newman.
- Local reviewer JWTs generated from the service repo.

## Start The Service

From any working directory, clone and start `FraudRuleEngineService`:

```powershell
git clone https://github.com/SethuBS/FraudRuleEngineService.git
cd FraudRuleEngineService
docker compose up --build
```

The collection defaults to:

```text
http://localhost:8080
```

Override `baseUrl` in the Postman environment if the service is exposed elsewhere.

## Generate Local JWT Tokens

From the `FraudRuleEngineService` repository, use the syntax for your shell.

PowerShell:

```powershell
$systemIngestorToken = .\scripts\generate-jwt.ps1 -Profile system-ingestor
$fraudAnalystToken = .\scripts\generate-jwt.ps1 -Profile fraud-analyst
$ruleAdminToken = .\scripts\generate-jwt.ps1 -Profile rule-admin
```

Bash or Git Bash:

```bash
systemIngestorToken="$(./scripts/generate-jwt.sh --profile system-ingestor)"
fraudAnalystToken="$(./scripts/generate-jwt.sh --profile fraud-analyst)"
ruleAdminToken="$(./scripts/generate-jwt.sh --profile rule-admin)"
```

The transaction evaluation request uses `systemIngestorToken`. If `POST /api/v1/transactions/evaluate` returns `403 ACCESS_DENIED` with `Required scope is missing`, the token is valid but it does not contain `transactions:evaluate`. Regenerate the `system-ingestor` token and paste that value into `systemIngestorToken`.

Copy the generated values into the imported Postman environment:

| Environment variable  | Token profile     |
|-----------------------|-------------------|
| `systemIngestorToken` | `system-ingestor` |
| `fraudAnalystToken`   | `fraud-analyst`   |
| `ruleAdminToken`      | `rule-admin`      |

These tokens are local-only. Production authentication is expected to come from an external identity provider.

## Import Into Postman

1. Import `postman/FraudRuleEngineService.postman_collection.json`.
2. Import `postman/FraudRuleEngineService.local.postman_environment.json`.
3. Select the `FraudRuleEngineService Local` environment.
4. Paste the generated token values into the environment.
5. Run `01 - Reviewer Smoke Flow` in order.

The smoke flow captures these variables as it runs:

- `transactionId`
- `customerId`
- `accountId`
- `riskLevel`
- `alertId`

## Newman

If Newman is available, run with PowerShell:

```powershell
newman run .\postman\FraudRuleEngineService.postman_collection.json `
  -e .\postman\FraudRuleEngineService.local.postman_environment.json `
  --env-var "systemIngestorToken=$systemIngestorToken" `
  --env-var "fraudAnalystToken=$fraudAnalystToken" `
  --env-var "ruleAdminToken=$ruleAdminToken"
```

Or run with Bash or Git Bash:

```bash
newman run ./postman/FraudRuleEngineService.postman_collection.json \
  -e ./postman/FraudRuleEngineService.local.postman_environment.json \
  --env-var "systemIngestorToken=$systemIngestorToken" \
  --env-var "fraudAnalystToken=$fraudAnalystToken" \
  --env-var "ruleAdminToken=$ruleAdminToken"
```

## Reset Test Data

For a completely clean service database, run this from your `FraudRuleEngineService` repository root:

```powershell
docker compose down -v
docker compose up --build
```

The high-risk sample is idempotent. Re-running the collection should return the stored evaluation for duplicate event and transaction IDs rather than creating duplicate business records.

## Included API Coverage

- `GET /actuator/health/readiness`
- `GET /actuator/info`
- `GET /actuator/metrics`
- `GET /actuator/prometheus`
- `GET /swagger-ui/index.html`
- `GET /v3/api-docs`
- `POST /api/v1/transactions/evaluate`
- `POST /api/v1/transaction-evaluations`
- `GET /api/v1/fraud-alerts`
- `GET /api/v1/fraud-alerts/{alertId}`
- `GET /api/v1/transactions/{transactionId}/fraud-evaluation`
- Missing token returns `401`.
- Wrong scope returns `403`.
- Invalid request returns `400`.
- Missing fraud alert returns `404`.
- Missing transaction fraud evaluation returns `404`.
- Fraud alert list pagination uses `page` and `size`.
- Fraud alert filters cover `customerId`, `accountId`, `riskLevel`, `fromDate`, and `toDate`.
- High-value boundary checks prove amount equal to the threshold does not match the high-value rule.
- Rule scenarios exercise high value, foreign country, risky merchant category, suspicious merchant, unusual amount, and velocity rule outcomes.

## Verification

The collection was verified locally with Newman against `FraudRuleEngineService` on `http://localhost:8080`:

```text
36 requests
81 assertions
0 failures
```
