---
name: zarinpal-gateway
description: Implement, review, or debug ZarinPal payment gateway integrations in PHP/Laravel and similar server-side applications. Use for secure payment request, redirect, callback, verification, idempotency, and production readiness; not for unrelated payment providers.
---

# ZarinPal gateway integration

Use this skill to implement or review a complete server-side ZarinPal payment flow. Preserve the target application's existing payment architecture. Keep provider-specific protocol code behind an adapter and keep order, payment, inventory, and customer state transitions in the application's authoritative domain layer.

## Non-negotiable payment invariant

A browser return is never proof of payment. Callback values such as `Status`, `Authority`, amount, order ID, reference ID, or a success-looking URL are untrusted. They may request verification, but only a successful server-to-server ZarinPal Verify for the persisted payment attempt, using that attempt's stored Authority and original persisted amount, can authorize a paid transition.

The paid transition must be atomic and idempotent. Repeated, concurrent, malformed, mismatched, or late callbacks must not repeat inventory commits, order transitions, receipts, notifications, or other financial effects.

## Workflow

1. **Inspect before changing anything.** Read repository instructions, payment documentation, routes, models, migrations, gateway contracts, services, settings/configuration, and tests. Inspect the installed SDK version from both Composer constraints and lockfile, then read the SDK's actual implementation. Extend the current pattern instead of replacing working payment code.
2. **Map the trust and state boundaries.** Identify who owns the authoritative amount, which record represents an attempt, current status transitions, order cancellation and reservation rules, callback/session requirements, and post-payment effects. Decide what happens when provider verification succeeds but the order can no longer be fulfilled.
3. **Define provider-neutral results.** Normalize initiation and verification into application-owned results. Keep SDK response objects and provider-specific codes inside the adapter. Do not claim the provider returned data it does not actually return.
4. **Implement short, durable state transitions.** Persist an attempt and its order-derived amount before contacting the provider. Make the network call outside a database transaction. Persist the returned Authority before redirecting the customer. On callback, verify outside a long database transaction, then lock and re-read the payment and order before applying the transition.
5. **Test the protocol and the boundary.** Use a fake client for deterministic tests. Exercise the SDK serialization separately. Cover forged callback inputs, failed verification, replay, concurrency, and downstream effects. Use an isolated test database; never reset a development or production database to make payment tests pass.
6. **Document operations.** Record sandbox/production configuration, callback URL and proxy requirements, safe troubleshooting signals, and a reconciliation path for ambiguous provider outcomes. Never run a production charge as a test; perform an external sandbox smoke request only when the user requested that check.

For the detailed architecture and flow, read [references/implementation-guide.md](references/implementation-guide.md). For callback threats and regression coverage, read [references/security-and-testing.md](references/security-and-testing.md). For SDK-specific details and upstream links, read [references/sdk-and-sources.md](references/sdk-and-sources.md).

## Design requirements

- **Amount:** derive it from the persisted order or immutable order snapshot, never the browser. Store and send integer amounts in one explicitly chosen unit. Confirm ZarinPal's current amount unit and currency semantics for the selected API/SDK version; Rial and Toman are not interchangeable. Convert at one documented boundary only.
- **Attempt identity:** persist a unique local payment-attempt identifier, gateway alias, amount, currency, idempotency key/fingerprint, status, Authority, provider reference, and timestamps needed for audit/reconciliation. Protect applicable uniqueness invariants with database constraints, not only PHP checks.
- **Callbacks:** bind the callback to one persisted attempt. Require a non-empty Authority that exactly matches that attempt before Verify. Use constant-time comparison for signatures or secret tokens. A signed callback URL can protect lookup from guessing, but it does not prove provider payment and never replaces Verify.
- **Verification:** send the persisted Authority and original persisted amount to ZarinPal. Map only documented success outcomes for the SDK/API version in use. Treat network errors, malformed responses, unknown codes, wrong Authority, and provider rejection as not verified. Distinguish definitive declines from transient/unknown outcomes so a timeout does not accidentally become an unrecoverable terminal failure. Persist a provider reference only from the authoritative response.
- **Concurrency:** use transactions and database locks or equivalent database-level concurrency controls around terminal transitions. Re-check status after the external call. A unique event/transaction constraint or equivalent guard should prevent duplicate side effects across workers and app nodes.
- **Effects:** commit inventory and update order payment/fulfillment state only after verified success and any domain reconciliation checks. Dispatch notifications and other non-transactional effects after commit, preferably through an outbox or idempotent queued listener where the application requires durable delivery.
- **Ambiguous timeouts:** a request timeout can happen after ZarinPal created a payment request. Do not blindly create repeated external attempts. Use provider idempotency/query capabilities when available; otherwise preserve the attempt as uncertain and provide an inquiry/reconciliation path.
- **Configuration:** load merchant credentials and mode through the application's established secret/configuration mechanism. Fail closed when required configuration is incomplete. Do not place secrets in source, logs, URLs, client code, or exception messages. Prevent sandbox mode from being enabled in production.
- **Logging:** log safe local payment/order IDs, operation, provider outcome code, and exception class. Avoid raw provider payloads, credentials, customer PII, card data, and callback secrets.
- **User experience:** customer result pages render persisted payment state only. A page visit must never mutate payment state merely because query parameters say payment succeeded.

If a framework-specific implementation is needed, use the target framework's current official documentation. The Laravel examples and SDK caveats in the references reflect the project used to develop this skill; verify them against the target app and installed versions.

