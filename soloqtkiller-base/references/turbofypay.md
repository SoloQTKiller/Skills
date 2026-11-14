# TurbofyPay v2

Read this reference completely before implementing, reviewing, migrating, or testing any TurbofyPay integration.

## Sources of truth

Use the current public documentation and OpenAPI as the only authoritative provider contract:

- Documentation index: <https://docs.turbofypay.com/llms.txt>
- Create PIX charge: <https://docs.turbofypay.com/pages/reference/cobranca-pix/criar-cobranca-pix>
- Authentication: <https://docs.turbofypay.com/pages/integracao/autenticacao>
- Webhooks: <https://docs.turbofypay.com/pages/integracao/webhooks>
- Public OpenAPI: <https://docs.turbofypay.com/openapi/turbofy.openapi.json>
- Production base URL: `https://api.turbofypay.com`

Fetch the documentation index and current OpenAPI again when starting a new integration or provider migration. Do not assume this file overrides a newer public contract.

## Do not use the published SDK

Do not install or use `@turbofy/sdk` for new or migrated projects. Its published package is materially outdated relative to the public v2 documentation: it advertises an obsolete default host and uses the legacy `/rifeiro/pix` route. Implement a small direct-HTTP adapter against the current public contract instead.

Do not copy routes, schemas, validation limits, response assumptions, or examples from the SDK. Do not use the SDK as a fallback source when it conflicts with the documentation or OpenAPI. If the provider later publishes a replacement SDK, adopt it only after comparing its host, paths, headers, payloads, webhook signature, status values, idempotency behavior, and error envelopes with the then-current public OpenAPI and passing contract tests.

## Adapter boundary

Keep TurbofyPay code in a provider adapter such as `src/services/turbofyPay.js`. The adapter owns:

- base URL and fixed canonical paths;
- authentication and HMAC headers;
- exact request serialization;
- bounded timeout and `Retry-After` handling;
- idempotency headers/keys;
- response and status normalization;
- safe provider error codes without secrets or raw sensitive bodies.

Handlers and domain services must consume normalized results and must not call TurbofyPay directly.

## Authentication

Every public API request uses backend-only headers:

- `x-client-id`
- `x-client-secret`

Never expose either credential to Discord, browsers, clients, query strings, logs, exceptions, fixtures, or documentation examples containing real values. Store credentials in environment variables and validate only their presence at startup.

Sensitive financial routes additionally require:

- `x-turbofy-timestamp`: current Unix time in milliseconds;
- `x-turbofy-signature`: lowercase hexadecimal HMAC-SHA256.

Build the exact string:

```text
{timestamp}.{METHOD}.{path}.{rawBody}
```

Use the client secret as the HMAC key. `METHOD` is uppercase, `path` excludes domain and query string, and `rawBody` is the exact JSON bytes sent; use an empty body for GET. Generate a fresh timestamp and signature for every request. Never sign one serialization and send another.

## Canonical route inventory

Use canonical routes, not legacy aliases:

| Method | Path | Purpose | Additional security |
| --- | --- | --- | --- |
| POST | `/sellers/pix` | Create PIX charge | `x-idempotency-key` recommended |
| GET | `/sellers/pix/{id}` | Read PIX charge | credentials |
| GET | `/v1/balance` | Read merchant balance | credentials only |
| POST | `/sellers/webhooks/` | Register global webhook | credentials |
| GET | `/sellers/webhooks/` | List global webhooks | credentials |
| GET | `/sellers/saques/` | List withdrawals | credentials |
| POST | `/sellers/saques/` | Create withdrawal | body idempotency |
| GET | `/sellers/saques/{id}` | Read withdrawal | credentials |
| GET | `/sellers/saques/payout-items` | List withdrawal payout items | credentials and filters |
| GET | `/sellers/saques/payout-items/export.xlsx` | Export withdrawal payout items | credentials and filters |
| GET | `/sellers/saques/payout-items/{itemId}/receipt` | Download payout receipt | credentials |
| POST | `/v1/payouts/batches` | Create payout batch | HMAC |
| GET | `/v1/payouts/batches` | List payout batches | HMAC |
| GET | `/v1/payouts/batches/{batchId}` | Read payout batch | HMAC |
| GET | `/v1/payouts/batches/{batchId}/items` | List batch items | HMAC |
| GET | `/v1/payouts/batches/{batchId}/items/{itemId}/receipt` | Download item receipt | HMAC |
| POST | `/v1/payouts/batches/{batchId}/cancel` | Cancel a cancellable batch | HMAC |
| POST | `/v1/payouts/pix-qr/preview` | Preview QR PIX payment | HMAC |
| POST | `/v1/payouts/pix-qr` | Confirm QR PIX payment | HMAC |
| GET | `/v1/payouts/pix-qr/{paymentId}` | Read QR PIX payment | credentials |
| GET | `/v1/receipts/transactions/{id}` | Download transaction PDF | HMAC |
| GET | `/v1/receipts/charges/{id}` | Download receipt by charge ID | HMAC |

`/rifeiro/pix` exists only as a legacy compatibility alias. Never select it for a new adapter and migrate existing adapters to `/sellers/pix` and `/sellers/pix/{id}`.

## Creating a PIX charge

Send `POST /sellers/pix` with `Content-Type: application/json`, credentials, and a stable `x-idempotency-key` persisted before the first attempt.

The body supports:

- `amountCents`: required positive integer in the provider transport contract;
- `description`: customer/order description;
- `externalRef`: stable internal reference;
- `expiresAt`: ISO-8601 date-time;
- `metadata`: non-sensitive reconciliation metadata;
- `webhookUrl` and optional `webhookSecret` for a charge-scoped webhook;
- `splits`: optional split array.

`merchantId` does not belong in an ordinary charge body. It is required only in each `splits` item. Do not invent or derive a merchant UUID from the client ID.

The successful response is HTTP 201 and contains at least `id`, `status`, `amountCents`, `pix.qrCode`, `pix.copyPaste`, `pix.expiresAt`, and `createdAt`. Persist the provider charge ID, internal idempotency key, external reference, expected amount, expiry, normalized state, and next polling time before delivering the PIX to the user.

Do not persist raw QR images indefinitely. Do not put personal documents or secrets in `metadata` or `description`.

## Polling and payment lifecycle

Use `GET /sellers/pix/{id}` as the authoritative reconciliation source. Normalize provider states into explicit internal states and treat unknown states as safe failures requiring review.

- Persist the next attempt and use a lease so only one worker polls a charge.
- Use a bounded initial delay and interval required by the product.
- Stop on terminal states.
- Honor `Retry-After` exactly for HTTP 429.
- Stop or disable repeated polling after invalid credentials.
- On timeout or uncertain POST result, reconcile the original idempotency key/operation; never create another charge blindly.
- A webhook may wake polling early, but must not credit, deliver, reserve, refund, or complete an order directly.

Keep customer-facing payment deadlines separate from the provider expiry. A product may warn that the customer has less time while continuing authoritative reconciliation until the provider expiry.

## Webhooks

Global webhooks are managed by `POST /sellers/webhooks/` and `GET /sellers/webhooks/`. A charge-scoped webhook may be created during `POST /sellers/pix` with `webhookUrl`; `webhook_url` is only a legacy field alias.

Documented charge events are:

- `charge.created`
- `charge.paid`
- `charge.expired`
- `charge.cancelled`

The webhook secret is distinct from `x-client-secret`. Receive the raw request body, enforce a small body limit and JSON content type, validate `turbofy-signature` according to the current webhook documentation, apply a replay window when the signature carries a timestamp, and persist an idempotent receipt by `eventId` or a stable payload hash before returning 2xx.

Never parse and reserialize the body before signature verification. Never expose the webhook secret. Return 2xx only after safely recording the event. Process business effects asynchronously and reconcile the charge by GET.

Do not assume the signature format forever. Compare the current webhook documentation with the implemented parser whenever creating or migrating an integration, because incoming webhook HMAC is a different contract from the outgoing `x-turbofy-signature` used on sensitive API requests.

## Balance, withdrawals, payouts, and receipts

`GET /v1/balance` uses only client credentials. Do not send a merchant ID, body, HMAC, or idempotency key. Treat service failure as unavailable balance, never as zero.

Do not automatically enable withdrawal or payout routes merely because they exist. Automated money-out is a separate product decision requiring explicit authorization, provider scopes, balance and freeze checks, recipient validation, HMAC, idempotency, approval policy, audit records, reconciliation, receipt handling, recovery, and controlled real-money homologation.

Payout batches use a stable `idempotencyKey` in the body and items with `pixKey`, `pixKeyType`, `amountCents`, `recipientName`, optional `recipientDocument`, and optional `referenceId`. Supported documented PIX key types are `CPF`, `CNPJ`, `EMAIL`, `PHONE`, and `EVP`.

Receipt endpoints return PDF rather than JSON. Validate status and content type, bound the download size, avoid storing unnecessary personal data, send Discord attachment logs as legacy `{ embeds, files }`, and follow the project's retention policy.

## Errors and safety

Handle at least:

- 400 validation or unresolved merchant errors;
- 401 missing/invalid credentials or signature;
- 403 IP policy/access denial;
- 404 provider object not found within the authenticated merchant;
- 409 idempotency, busy, freeze, unavailable receipt, or state conflict;
- 422 invalid/expired QR or amount mismatch;
- 429 rate limit with `Retry-After`;
- 500 internal provider error;
- 502 provider issuance failure;
- 503 temporarily unavailable financial read.

Expose a stable internal error code and status only. Redact response bodies before private logging and never surface provider internals or credentials to users. An unknown financial result must enter `needs_review`, not be treated as success or failure automatically.

## Required tests and homologation

Add contract tests for:

- exact base URL, canonical path, method, headers, and body;
- omission of `merchantId` from an ordinary charge;
- `merchantId` required only for split items;
- real-to-cent conversion at the adapter boundary without floating-point arithmetic;
- 201 response normalization and every known terminal state;
- malformed JSON, missing PIX payload, unknown status, timeout, 400, 401, 403, 404, 409, 422, 429, 5xx, and `Retry-After`;
- stable idempotency and repeated calls;
- uncertain creation timeout without blind retry;
- webhook signature, raw-body mutation, replay, duplicate event, and out-of-order delivery;
- restart recovery, lost lease, polling overlap, payment arriving after the customer-facing deadline, and duplicated paid observation;
- redaction proving that credentials, webhook secrets, PIX keys, personal documents, and raw provider payloads do not leak.

Before a real charge, perform a read-only credential check with a deliberately nonexistent UUID through `GET /sellers/pix/{id}` and require the documented 404 rather than 401/403. Real charge, payment, expiry, payout, withdrawal, and receipt tests must be explicitly authorized because they can create financial resources or move money. Record only sanitized correlation IDs and outcomes in homologation reports.
