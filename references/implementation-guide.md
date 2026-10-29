# Implementation guide

## Provider-neutral Laravel shape

A robust Laravel integration separates three responsibilities:

- **SDK client:** translates typed application inputs to the installed ZarinPal SDK and converts SDK responses/exceptions into a small normalized shape. This is the only layer that should know SDK namespaces, response classes, endpoint selection, and SDK-specific serialization quirks.
- **Gateway adapter:** implements the app's gateway contract, maps request/verify results to application DTOs, constructs a trusted callback URL, obtains the provider redirect URL, and returns safe failure results.
- **Payment application service:** creates durable attempts from authoritative order snapshots, enforces idempotency, applies locked state transitions, reconciles order/inventory eligibility, records transaction history, and dispatches post-commit effects.

Keep controllers thin: authorize the customer/order, call the application service, redirect to the provider or result page, and pass callback inputs to a service. Do not put pricing, payment state transitions, inventory commit, or SDK request construction in a controller, Blade template, or JavaScript.

A useful contract has methods equivalent to `initiate(PaymentAttempt)` and `verify(PaymentAttempt)`, with application-owned result values. Keep the provider alias persisted on each attempt so a later settings change cannot make an old callback invoke a different provider.

## Initiation sequence

1. Authorize the order and establish it is payable. Calculate the amount from the order's stored total, including its already-snapshotted tax/discount/shipping results. Never accept amount, currency, or callback URL from the browser.
2. In a short transaction, lock/re-read the order, check payable and inventory-reservation rules, then create a payment attempt with a unique idempotency key/fingerprint and immutable amount/currency.
3. Call ZarinPal after the transaction commits. Do not hold database locks while waiting on the network.
4. Treat only the documented request-success outcome plus a non-empty Authority as successful initiation. Persist the Authority, provider response facts needed for support, and processing state before returning the redirect URL.
5. Build the callback URL from trusted application routing/configuration. In production it must resolve to the public HTTPS origin. Configure proxy/trusted-host behavior correctly; never use a browser-supplied Host header as the payment callback origin.
6. If the provider call fails, record a safe failure or uncertain state according to whether the outcome is known. A transport timeout is not necessarily a provider rejection. Keep retryable/unknown outcomes distinct from definitive provider declines so support can recover safely.

A local attempt is needed before the external call so the callback can be bound to a known order and amount. If persistence of the provider Authority fails after ZarinPal created the request, retain enough local diagnostics to reconcile the uncertain attempt; do not tell the customer that payment succeeded.

## Callback and verification sequence

1. Make the provider callback independent of a customer login/session when ZarinPal must reach it directly. Scope the route narrowly and exempt only the appropriate provider callback from browser CSRF expectations; use the application's routing and signature protections without pretending they authenticate the provider.
2. Resolve exactly one persisted attempt by an opaque public ID or signed lookup token. Confirm it belongs to the ZarinPal gateway and is in a state eligible for verification.
3. Require a recognized callback status that requests verification, and require a non-empty Authority exactly equal to the Authority stored on this attempt. Reject a missing, malformed, or other attempt's Authority before calling Verify. Ignore callback amount, order ID, ref ID, and customer identity as financial inputs.
4. Call ZarinPal Verify with the persisted Authority and persisted original amount. This external request happens outside the DB transaction. Normalize the result based on current official API/SDK semantics. Do not infer success from HTTP 200 alone.
5. In a transaction, lock and re-read the Payment and Order. Re-check that this same attempt remains eligible and that the verification result is still applicable. Record the Verify transaction once. Handle terminal paid/failed attempts as idempotent replays.
6. Check the order's state, amount/currency, reservation expiry/coverage, and other fulfillment rules. A verified provider transfer that cannot safely complete the order needs an explicit reconciliation/refund workflow; it must not silently commit stock or overwrite a canceled order.
7. For a normal success, atomically mark the Payment paid, update the Order payment/fulfillment states, and commit inventory according to the commerce domain's transaction boundaries. Emit a domain event or outbox record after commit for notifications/receipts. Make consumers idempotent.
8. Render the customer result from persisted state only. The result page itself does not call Verify or mutate state just because the customer visits it. If the browser and provider callback race, both paths must converge through the same idempotent service.

Use database transactions and locking supported by the configured database. In Laravel, see the official [database transaction documentation](https://laravel.com/docs/12.x/database). Ensure queues/events are dispatched after commit when they depend on committed records.

## Settings and secrets

Use the target application's existing runtime configuration contract. In the source project, gateway selection, enabled state, sandbox mode, and merchant ID are registered core Site Settings. Additive migrations create missing rows without overwriting existing values; admins can edit values but not key/group/type metadata. Runtime code reads values through `SettingsService` and a dedicated `PaymentGatewayConfiguration` resolver, then passes an immutable settings object into the SDK adapter. The merchant credential is a secret setting encrypted at rest and masked in Filament. This keeps direct setting queries out of provider code and makes registration testable.

For another app, use its established secret manager, encrypted settings, or environment config. Keep environment infrastructure distinct from application settings. Secret fields must be encrypted at rest where stored, masked in administration UIs, and omitted from logs/status output. Validate merchant ID shape and mode, but still rely on ZarinPal for authoritative acceptance.

## Amount, currency, and response mapping

The source project stores customer prices in integer Rial and configures ZarinPal currency as `IRR`. Other apps or API versions may use a different unit convention. Before shipping, reconcile the amount unit across order storage, API request, verification, invoices, refunds, admin display, and logs. Use integer money or a money value object; never float. Add an explicit test with a distinctive amount to catch a factor-of-ten error.

Provider response data should be normalized conservatively. Keep the provider's exact response code and safe fee/reference fields where support needs them. Do not put full raw payloads or card data in a general JSON column. If the provider does not return a field, don't synthesize it and describe it as provider-confirmed. The application may separately record which persisted amount it sent for the successful verification.

## Availability and operations

- Set explicit connection timeouts and bounded retry behavior in the SDK/HTTP client if supported. Avoid automatic retries for non-idempotent request operations unless the provider supports an idempotency token or reconciliation proves safe.
- Keep provider calls outside DB transactions. For recoverable async verification/reconciliation jobs, use attempt identity as the idempotency key, bounded retries/backoff, and an operational queue that is monitored.
- Track counts and latency for request/verify operations, unresolved attempts, callback Authority mismatches, and reconciliation-required payments. Exclude credentials, raw callback signatures, and PII.
- Retain enough payment transaction/audit history to explain each state transition. A log is not the payment ledger.
- Provide a support runbook: inspect the local attempt and transaction history, query/inquire with the provider using authorized tooling, reconcile under controlled authorization, and document any manual refund/state repair. Never solve a mismatch by manually marking an order paid from callback parameters.

## Source-project implementation pattern

The source app uses `PaymentGatewayInterface` for providers, `ZarinPalClientInterface` for the SDK boundary, `ZarinPalPaymentGateway` for protocol mapping, `PaymentGatewayRegistry` for selection, `PaymentGatewayConfiguration` for resolved settings, and `PaymentService` for transactional payment/order behavior. Its public callback resolves by persisted payment number, checks a constant-time HMAC lookup signature and stored Authority, and calls the same `PaymentService::verify()` used by the application. Treat these as examples of separations and invariants, not mandatory class names for other repositories.

