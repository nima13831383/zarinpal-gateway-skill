# SDK details and upstream sources

## Version-specific notes from the source project

The source repository's `composer.lock` pins `zarinpal/zarinpal-php-sdk` v2.0.0 at commit `7025d1562796938f08e95e54d7ac272d1fcec2c1`. Its wrapper uses `ZarinPal\Sdk\ZarinPal`, `Options`, `PaymentGateway`, `RequestRequest`, and `VerifyRequest`; it builds the redirect URL through the SDK rather than hard-coding a production/sandbox URL.

The source adapter maps request code `100` plus a non-empty Authority to successful initiation. Its verified tests map Verify codes `100` and `101` to verified success, with `101` representing an already-verified payment in the tested API behavior. Reconfirm these mappings against current ZarinPal docs and the installed SDK before copying them into another project; unknown codes must fail closed.

The SDK v2 project wrapper comments that nullable `mobile` and `email` fields serialize as keys with null values, which its API request path rejects. This is SDK-version-specific. Inspect the serializer and current API contract; omit absent optional metadata where supported. Do not fabricate customer phone numbers or email addresses just to satisfy serialization.

The source app treats persisted integer Rial as authoritative and configures currency `IRR`. Do not assume that convention applies to every SDK/API release. ZarinPal examples have historically used inconsistent unit wording; confirm the exact current contract, then align the order, request, Verify, display, refund, and reporting layers. Avoid floats and multiple implicit conversions.

## Upstream references

Prefer provider-maintained sources and inspect the exact installed version or commit rather than relying on copied snippets:

- [ZarinPal PHP SDK repository](https://github.com/ZarinPal/zarinpal-php-sdk) — primary repository for the PHP SDK; its README is brief, so inspect source and package version for actual behavior.
- [Pinned SDK source commit used in the source project](https://github.com/ZarinPal/zarinpal-php-sdk/tree/7025d1562796938f08e95e54d7ac272d1fcec2c1) — reproducible reference for the v2.0.0 package used when this skill was authored.
- [Packagist package](https://packagist.org/packages/zarinpal/zarinpal-php-sdk) — package metadata and release history; verify the resolved version from the application's lockfile.
- [ZarinPal-Lab REST API sample](https://github.com/ZarinPal-Lab/Zarinpal-RestAPI-Sample-php) — provider-associated PHP request/Verify examples. Treat examples as protocol pointers and validate against current docs.
- [ZarinPal-Lab API v4 PHP sample](https://github.com/ZarinPal-Lab/Api-V4-RestAPI-Sample-php) — older API-version example; do not assume its field names, host, units, or endpoints match a newer SDK.
- [ZarinPal GitHub organization](https://github.com/ZarinPal) — confirm which repositories are currently maintained and provider-owned.
- [Laravel 12 database transactions](https://laravel.com/docs/12.x/database) — transaction and locking reference for the source app's Laravel major version.
- [Laravel 12 HTTP tests](https://laravel.com/docs/12.x/http-tests) — request-level test reference.
- [PHP `hash_equals`](https://www.php.net/manual/en/function.hash-equals.php) — constant-time comparison for callback signatures and other secret values.

The provider's public documentation is authoritative for current request/Verify schemas, supported currencies, amount units and bounds, status codes, redirect URLs, sandbox rules, timeouts, and refunds. If the exact docs URL or version is not clear from the repository, locate the provider's current official documentation instead of treating a third-party blog, old README, or this skill as current protocol authority.
