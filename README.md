# ZarinPal Gateway Skill

A reusable Codex skill for implementing, reviewing, and debugging server-side ZarinPal payment integrations, with a focus on PHP and Laravel.

## Install

Copy this folder into your Codex skills directory:

```text
~/.codex/skills/zarinpal-gateway/
```

Then invoke it with:

```text
$zarinpal-gateway
```

The skill adapts to the target project's payment architecture and SDK version. It covers provider-neutral gateway boundaries, persisted payment attempts, callback trust, server-side verification, idempotency, concurrency, reconciliation, and testing.

## Contents

- `SKILL.md` — entrypoint and core payment invariants
- `references/implementation-guide.md` — Laravel-oriented flow and production architecture
- `references/security-and-testing.md` — callback threat model and test coverage
- `references/sdk-and-sources.md` — source-project SDK notes and upstream references

## Upstream references

- [ZarinPal PHP SDK](https://github.com/ZarinPal/zarinpal-php-sdk)
- [ZarinPal REST API sample](https://github.com/ZarinPal-Lab/Zarinpal-RestAPI-Sample-php)
- [ZarinPal API v4 PHP sample](https://github.com/ZarinPal-Lab/Api-V4-RestAPI-Sample-php)
- [Laravel database documentation](https://laravel.com/docs/12.x/database)
- [Laravel HTTP testing documentation](https://laravel.com/docs/12.x/http-tests)
- [PHP `hash_equals`](https://www.php.net/manual/en/function.hash-equals.php)

Verify current provider documentation and the installed SDK before relying on version-specific API details.
