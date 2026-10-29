# Security and testing guide

## Threats to cover

Assume all provider callback query parameters and browser redirects can be forged, replayed, reordered, omitted, or copied from another payment attempt. Specifically test that none of these inputs authorize success by themselves:

- `Status=OK` without a successful server Verify
- a correct-looking Authority that belongs to another attempt
- an amount/order/reference value supplied in the callback
- direct navigation to a success/result URL
- a valid old callback after the attempt has failed or a newer attempt exists
- duplicate callback delivery or simultaneous callback requests

A signed callback lookup value reduces payment-number guessing, but it is a capability for identifying the local attempt, not proof that ZarinPal made a payment. Compare signatures with a constant-time primitive such as PHP [`hash_equals`](https://www.php.net/manual/en/function.hash-equals.php).

## Test layers

### Adapter/client tests

- Request success maps to a redirectable Authority; request rejection, missing Authority, malformed response, and thrown transport/SDK exception map to safe failure or uncertain outcomes. Verify transient failures are retryable/reconcilable and are not confused with definitive declines.
- Verify uses the exact persisted Authority and integer amount; only currently documented success codes map to verified. Confirm whether an already-verified response is a success for the selected API version and how it is made idempotent.
- The SDK client sends the expected merchant, mode, amount unit, currency, description, callback URL, and metadata serialization. Confirm sandbox and production host selection from the installed SDK.
- Verify response reference mapping, unknown response fields, and exception handling without logging secrets or the full response body.
- Test the SDK through a mocked HTTP client or SDK boundary. Do not make a live charge in a normal feature test.

### Application/service tests

- Attempt amount is derived from the order snapshot, not a request field; the same idempotency key and fingerprint returns one attempt, while reuse with different semantics is rejected.
- External calls happen outside the critical DB transaction; failed or uncertain initiation does not mark an order paid or commit inventory.
- Successful verification marks the payment/order once and commits reservations once. Duplicate verification does not create duplicate ledger entries, transitions, notifications, or receipts.
- A verified but mismatched amount/currency, canceled/paid order, expired reservation, or other fulfillment conflict enters an explicit reconciliation path without silently committing inventory.
- Payment and order locking/unique constraints handle concurrent callbacks or multiple attempts. Run concurrency cases against the same DB engine and isolation model used in production where practical.

### Callback feature tests

Include success and negative cases for:

- correct signed lookup, `Status=OK`, matching persisted Authority, successful provider Verify
- missing, empty, malformed, wrong, and cross-attempt Authority; Verify must not be called
- `Status=NOK`, missing/unknown status; no paid transition
- `Status=OK` with provider verification rejection or exception; no order payment, inventory, or notification effect
- tampered callback amount, order ID, or reference; SDK receives the persisted payment amount and attempt
- invalid callback signature or unknown attempt; no provider call
- repeated successful callback; one payment/order transition and one downstream effect
- old/failed attempt callback after a newer attempt exists; it cannot alter the new attempt or order
- callback without customer session where the provider requires session independence
- browser result/success page with forged query data; no state mutation

Use assertions on persisted Payment, Order, inventory ledger/reservation, transaction/audit rows, and notification/outbox rows. A response-only assertion is not enough to prove financial safety.

Laravel applications can use [HTTP feature tests](https://laravel.com/docs/12.x/http-tests) and the app's isolated testing database. Preserve this distinction: SQLite feature coverage can prove common request semantics, but concurrency guarantees must be validated using the target relational DB when those guarantees depend on its locking behavior.

## Release checklist

- [ ] Production credentials are stored in the approved secret mechanism and are not present in test fixtures, screenshots, logs, URLs, or repository history.
- [ ] Production mode cannot point to sandbox; sandbox credentials/mode cannot be active in production.
- [ ] Public callback URL is HTTPS, stable behind the load balancer, and reachable without relying on one application node's local session.
- [ ] Attempt and Authority uniqueness constraints/indexes match the database engine and callback lookup pattern.
- [ ] Request, Verify, callback, replay, and concurrent-transition behavior is covered by isolated tests.
- [ ] Amount unit, currency, rounding, and API-version response codes are confirmed against current provider documentation and merchant configuration.
- [ ] Monitoring and a manual reconciliation procedure exist for provider timeout, successful transfer with failed local persistence, and payment/order mismatch.
- [ ] No routine verification test can reach production ZarinPal or the development database.

For ZarinPal integration-specific API facts, use the upstream sources in [sdk-and-sources.md](sdk-and-sources.md) and re-check them when upgrading the SDK/API version.

