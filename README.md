# Elixir SDK Generator Template

A GitHub template for generating Elixir SDKs from OpenAPI specifications using [openapi-generator](https://openapi-generator.tech/).

## Features

✨ **Automated Generation**: One-command SDK generation with zero manual intervention
🔄 **Auto-Regeneration**: GitHub Actions automatically regenerate SDK when specs change
🏊 **Connection Pooling**: Built-in Finch connection pooling for optimal performance
♻️ **Retry Logic**: Automatic retries with exponential backoff (idempotent requests only)
📊 **Telemetry**: Integrated telemetry for monitoring and observability
🧪 **Test Infrastructure**: Test helpers (Bypass mock server, Mox, fixtures) that survive regeneration
📈 **Code Coverage**: Coverage tracking with a configurable threshold (via `coveralls.json`)
🔍 **Breaking Changes**: Spec-level breaking change detection with [oasdiff](https://github.com/oasdiff/oasdiff)
📦 **Hex.pm Ready**: Pre-configured publishing workflow
🚀 **Git-ops Releases**: Conventional commits → automated version bump, changelog, tag, and publish (via [git_ops](https://hex.pm/packages/git_ops))
🔄 **Spec Sync**: Weekly workflow checks your upstream spec URL and opens a PR when it changes
🤖 **Agent-Ready**: AGENTS.md/CLAUDE.md, Claude Code permissions, and skills — `/setup-sdk` mints the SDK (and cleans itself up afterwards), `/regenerate` stays for maintenance
✅ **Self-Testing**: The template repo runs its own end-to-end smoke test in CI

## Quick Start

> **Prerequisites**: Erlang/Elixir as pinned in `.tool-versions` (currently
> Elixir 1.20.2 / OTP 29 — generated SDKs stay compatible down to Elixir 1.18),
> Java 11+ (for OpenAPI Generator), and OpenAPI Generator itself:
> `brew install openapi-generator` (macOS/Linux) or
> `npm install -g @openapitools/openapi-generator-cli`.
> `jq` is required only for non-interactive setup (`--config`).

### 1. Use This Template

Click the "Use this template" button on GitHub to create your own repository.

### 2. Run Setup

```bash
# Interactive
./scripts/setup.sh

# Non-interactive (see setup.example.json for the format)
./scripts/setup.sh --config my-setup.json

# Skip git initialization (e.g. in CI)
./scripts/setup.sh --config my-setup.json --no-git
```

Setup will ask for:
- Package name (e.g., `my_api_client`)
- Module namespace (optional PascalCase, e.g. `MyAPIClient`; defaults to the
  camelized package name, e.g. `example_api` → `ExampleAPI`)
- Description (optional; falls back to the spec's `info.description`)
- GitHub repository details
- API base URL (optional)
- OpenAPI specification location (optional; keeps the bundled example
  otherwise). **Provide a URL to enable the weekly spec-sync workflow** —
  it gets recorded in `.spec-source`, and GitHub `/blob/` URLs are converted
  to raw URLs automatically.
- License SPDX id (default `MIT`)

Setup **fresh-initializes the release-facing files for your SDK** — a new
CHANGELOG.md (with the release-automation marker), a minimal README.md filled
with your package details, a LICENSE with your name, and removal of
template-only docs. Your SDK's history starts at zero rather than inheriting
this template's. Opt out with `--keep-template-docs`.

Alternatively, run the **`/setup-sdk`** skill in Claude Code — it interviews
you and runs the whole setup + first generation + docs update flow. The skill
is one-shot: setup removes it (along with the template smoke-test workflow)
once your SDK exists, leaving only the `/regenerate` maintenance skill.

### 3. Generate SDK

```bash
./scripts/regenerate.sh
```

This will validate your spec, generate the SDK, run post-processing (including
starter test files for each API module), install dependencies, format the code,
and run the tests.

### 4. Review and Test

```bash
mix check      # unused deps, hex.audit, warnings-as-errors, format, credo --strict, tests
mix dialyzer   # run separately (slow)
```

All of these pass on a freshly generated SDK.

## Project Structure

```
elixir-sdk-generator/
├── .claude/                 # Claude Code permissions + skills:
│                            #   /setup-sdk (one-shot, removed by setup.sh)
│                            #   /regenerate (kept for SDK maintenance)
├── AGENTS.md                # Agent guidance (CLAUDE.md points here)
├── .github/workflows/       # CI/CD pipelines
│   ├── template-smoke.yml   # End-to-end template test (template repo only;
│   │                        #   removed by setup.sh)
│   ├── test.yml.disabled    # Test on every push (enabled by setup.sh)
│   ├── spec-sync.yml.disabled       # Weekly upstream spec check → PR
│   ├── regenerate-sdk.yml.disabled  # Auto-regenerate SDK
│   ├── publish.yml.disabled         # Publish to Hex.pm
│   └── breaking-changes.yml.disabled # Detect breaking changes
├── .openapi-generator/
│   └── templates/           # COMPLETE vendored template set (see below)
├── config/                  # Elixir configuration
├── lib/                     # Generated SDK code (disposable)
├── scripts/                 # Automation scripts
├── test/                    # Tests (persistent, never regenerated)
│   ├── unit/
│   ├── integration/
│   ├── fixtures/
│   └── support/
├── coveralls.json           # Coverage threshold configuration
├── generator-config.yaml    # OpenAPI Generator config
├── setup.example.json       # Example config for non-interactive setup
└── openapi-spec.yaml        # Your OpenAPI specification
```

## Usage

### Basic Example

```elixir
# Create a connection (module namespace derived from your package name)
conn = MySDK.Connection.new()

# Make API calls — responses decode into typed model structs
{:ok, %MySDK.Model.User{} = user} = MySDK.Api.Users.get_user(conn, user_id)
```

### Custom Configuration

```elixir
# Custom base URL
conn = MySDK.Connection.new(base_url: "https://api.example.com")

# Custom timeout
conn = MySDK.Connection.new(timeout: 60_000)

# Custom retry configuration
conn = MySDK.Connection.new(retry: [max_retries: 5, delay: 200, max_delay: 10_000])

# Disable retries
conn = MySDK.Connection.new(retry: false)

# Extra middleware
conn = MySDK.Connection.new(middleware: [{Tesla.Middleware.Logger, []}])
```

### Retry Semantics

Retries use exponential backoff and are **limited to idempotent HTTP methods**
(GET, HEAD, OPTIONS, PUT, DELETE) on status 408/429/5xx or transport errors.
POST requests are never retried automatically, so requests with side effects
are never replayed. Override with a custom predicate:

```elixir
conn = MySDK.Connection.new(retry: [should_retry: fn result, env, _ctx -> ... end])
```

### Runtime Configuration

```elixir
# config/runtime.exs
config :my_sdk,
  base_url: System.get_env("API_BASE_URL", "https://api.example.com"),
  pool_size: String.to_integer(System.get_env("HTTP_POOL_SIZE", "25")),
  pool_count: 1,
  connect_timeout: 5_000
```

## Development Workflow

1. Edit `openapi-spec.yaml` with your API changes (or let the weekly
   spec-sync workflow pull them from your recorded spec URL)
2. Run `./scripts/regenerate.sh` — or the `/regenerate` skill in Claude Code,
   which also reviews the diff, updates tests, and drafts the CHANGELOG entry
3. Flesh out the starter tests created in `test/unit/` for new endpoints
4. Commit and push

### Keeping up with an upstream spec

If setup recorded a spec URL in `.spec-source`, the **spec-sync** workflow
checks it every Monday, regenerates when it changed, and opens a PR with an
oasdiff changelog of the API changes. Run it on demand from the Actions tab.

## GitHub Actions Workflows

Workflows ship disabled (`.disabled` suffix) so they don't run on the template
repo; `./scripts/setup.sh` enables them and removes the template-only smoke
test. See [.github/workflows/README.md](.github/workflows/README.md) for
details, including the required `HEX_API_KEY` secret and a note about PR
creation with `GITHUB_TOKEN` not triggering CI.

## Advanced Features

### Attribution

Generated files carry a header crediting OpenAPI Generator and this template,
and the Hex package links include a "Generated with" entry. Please keep them —
they help others find the tooling.

### Custom Templates

`.openapi-generator/templates/` contains the **complete** template set for the
elixir generator (vendored from openapi-generator 7.23.0, the latest release) —
the elixir generator does not fall back to built-in templates when
`templateDir` is set, so partial template directories do not work. The
customized templates are:

- `connection.ex.mustache` — Finch adapter, timeouts, telemetry, and
  idempotent-only retries on top of the stock connection (auth support and
  `request/2` preserved)
- `mix.exs.mustache` — dev/test tooling, docs, coverage, and the
  `Application` supervisor wiring
- `application.ex.mustache` — Finch pool supervisor (registered as an extra
  supporting file via the `files:` section of `generator-config.yaml`)
- `model.mustache` — moduledoc fallback for schemas without descriptions

If you upgrade openapi-generator, re-vendor the stock templates and re-apply
those customizations (see the note at the top of `generator-config.yaml`).

### Protected Files

Files listed in `.openapi-generator-ignore` are never overwritten during
regeneration: configuration, tests, scripts, workflows, docs, and this
README.

### Coverage Threshold

`coveralls.json` ships with `minimum_coverage: 0` so a freshly generated SDK
passes CI. Raise it once you've added tests for your API operations:

```json
{ "coverage_options": { "minimum_coverage": 80 } }
```

## Testing

```bash
mix test                     # all tests
mix coveralls                # with coverage (threshold from coveralls.json)
mix coveralls.html           # HTML report
mix test test/integration/   # integration tests only
```

## Releasing & Publishing

Releases are git-ops driven: PR titles are validated against
[Conventional Commits](https://www.conventionalcommits.org) (use squash
merges), and the **Release** workflow (or `mix git_ops.release` locally)
derives the next version from the commit history, updates `@version` and
CHANGELOG.md, tags, and triggers the Hex.pm publish workflow:

- `fix: ...` → patch, `feat: ...` → minor, `feat!:` / `BREAKING CHANGE` → major
- `@version` in mix.exs is the version source of truth — regeneration
  preserves it automatically
- Requires the `HEX_API_KEY` secret (from `mix hex.user auth`); the
  `<!-- changelog -->` marker it needs is already in the CHANGELOG.md that
  setup writes

Manual fallbacks still work: `./scripts/publish.sh` locally, or push a
`v*.*.*` tag to trigger publish.yml directly.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for development guidelines.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history.

## Resources

- [OpenAPI Generator](https://openapi-generator.tech/)
- [Elixir Tesla](https://github.com/elixir-tesla/tesla)
- [Finch](https://github.com/sneako/finch)
- [Hex.pm](https://hex.pm/)
