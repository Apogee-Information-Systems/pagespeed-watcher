# AGENTS.md

## Cursor Cloud specific instructions

This repo is a single PHP/Laravel Composer package (`apogee/laravel-pagespeed-watcher`), not a standalone app. There is no long-running server, frontend, or database process to start for development. Dev work is: install deps, run PHPUnit, run lint, and exercise the Artisan commands via Orchestra Testbench.

Runtime: PHP 8.3 (satisfies `^8.2`) is installed. Composer is available both as the global `composer` command and as the committed PHAR at `bin/composer`. Dependencies live in `vendor/` (not committed); the startup update script runs `composer install`.

### Standard commands (already documented in `composer.json` / `.github/workflows/ci.yml`)
- Tests: `vendor/bin/phpunit` (or `composer test`). Uses Orchestra Testbench + in-memory SQLite; no external services needed.
- Validate manifest: `composer validate --strict`.
- Lint: `composer lint` — note this is best-effort (`|| true`) and runs `vendor/bin/phpcs`, but `squizlabs/php_codesniffer` is NOT a declared dependency, so out of the box it prints "phpcs: not found" and passes. To actually run PSR-12 checks, install phpcs (e.g. `composer global require squizlabs/php_codesniffer`) and run `phpcs --standard=PSR12 src`. The current `src/` has pre-existing PSR-12 violations (trailing whitespace, missing final newlines).

### Running the Artisan commands (no host app in this repo)
The package ships two Artisan commands, `watcher:test-page` and `watcher:usage`. To run them, use the Testbench console which boots a minimal Laravel app that auto-discovers `WatcherServiceProvider`:

```bash
export DB_CONNECTION=sqlite DB_DATABASE=/tmp/watcher.sqlite
export APP_KEY="base64:P5w0pH2K2s0hY4QyY8jL7d6mA9cB3fE0vN5oT2uR1wM="
touch /tmp/watcher.sqlite
vendor/bin/testbench migrate:fresh          # creates watcher_* tables
vendor/bin/testbench watcher:usage
vendor/bin/testbench watcher:test-page --url=https://example.com
```

Gotchas:
- Use a FILE-based SQLite DB (not `:memory:`) when running multiple `testbench` invocations. Each invocation is a separate process, so an in-memory DB is discarded between commands and you'll get "no such table" errors.
- `watcher:test-page` needs a Google PageSpeed Insights key via `PSI_API_KEY` for a successful run. Keyless requests to the public PSI API are rate-limited (HTTP 429). It is optional — the test suite mocks the PSI HTTP client, so no key is required for `composer test`.
- Known pre-existing code bug (do not "fix" as part of setup): `WatcherApiUsage::incrementRequests()` uses the SQL function `GREATEST(...)`, which does not exist in SQLite. So any real (non-mocked) PSI request that reaches usage recording fails with `no such function: GREATEST` on SQLite. This does not affect the test suite because tests mock the PSI client and never execute that raw SQL against SQLite.
