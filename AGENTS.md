# AGENTS.md

## Project

This service is a Rust + Axum + Chromiumoxide screenshot API. It renders web pages with headless Chromium and returns PNG, JPEG or WebP bytes.

Important endpoints:

- `GET /health`
- `GET /screenshot` (query params)
- `POST /screenshot` (JSON body)

The implementation lives in `src/`. Keep request parsing and validation in `src/request.rs`, HTTP wiring in `src/main.rs`, and Chromium work in `src/screenshot.rs`.

## Commands

Run these before handing off changes:

```bash
cargo fmt --all -- --check
cargo clippy --locked --all-targets -- -D warnings
cargo test --locked
docker build -t screenshot-service:rust .
```

Run a single test: `cargo test --locked applies_defaults_like_the_go_service`.

Use `docker compose up -d --build` for a full local container smoke test, then clean it up with `docker compose down`. Stay aligned with the pinned `Cargo.lock`.

## Architecture

Five modules under `src/` — keep the responsibilities separated:

- `main.rs` — Axum router, 8 MiB body-size limit, `PORT` binding (default `8080`), startup banner, graceful shutdown (Ctrl+C / SIGTERM). `process_screenshot` is the shared post-validation pipeline that calls `take_screenshot`, then attaches `Content-Type` / `Content-Length` / `Content-Disposition` / `Cache-Control` headers to the raw image bytes.
- `request.rs` — `ScreenshotRequest` (POST JSON shape) and `ScreenshotQuery` (GET shape, where `headers` and `clip` arrive as JSON-encoded strings and are parsed into the request). All defaulting and clamping lives in `apply_defaults`; bounds checks live in `validate`. Both run in `process_screenshot` before any browser work.
- `screenshot.rs` — Chromium lifecycle. Each request launches a fresh `Browser` with a per-request `tempfile` user-data dir, spawns a handler task to drain CDP events, then runs `capture_page` under a `tokio::time::timeout` derived from `req.timeout`. Browser and handler are torn down on all paths; the page is closed explicitly unless the request timed out, in which case closing the browser takes it down (teardown errors are only logged at debug). Three capture modes: full-page (re-overrides device metrics to the layout content size, height capped at 16384px, uses `capture_beyond_viewport`), `clip`, or plain viewport. `wait_for` polls `find_element` + `bounding_box` every 100 ms until the element is visible or the request timeout elapses.
- `error.rs` — `AppError::BadRequest` → 400, `AppError::Screenshot` → 500. Always return the `{"error": "..."}` JSON shape.
- `logging.rs` — tracing init (`logging::init`, called first in `main`). Uses `RUST_LOG` when set, otherwise `screenshot_service=info,tower_http=info,warn`, and formats events with a custom colored `[time][LEVEL][component]` formatter (component = module name, `http` for `tower_http`).

## Conventions

- Preserve the existing HTTP contract documented in `README.md`.
- Keep the service compatible with the pinned Docker Rust toolchain.
- Do not reintroduce Go files, `go.mod`, or `go.sum`. The repo was rewritten from Go; the old files are deleted in the working tree but still visible in `git log`.
- Keep abstractions small — the five-module layout is intentional.
- Image bytes flow through `axum::body::Body` directly; do not buffer them through additional copies.
- Keep generated artifacts out of git: `target/`, `.idea/`, local screenshots, and logs.
- Prefer small, focused changes over broad rewrites.

## Runtime Notes

Local non-Docker runs require Chrome or Chromium on PATH. Set `CHROME_BIN` (or `CHROMIUM_BIN` / `CHROME_PATH`) when auto-detection is not enough.

The Docker image installs Alpine Chromium and runs as a non-root user behind `dumb-init`.

## Git commits

All commit subjects must follow:

```text
[Type] Short description starting with capital letter
```

Allowed types:

| Type      | Usage                                                 |
|-----------|-------------------------------------------------------|
| `[Feat]`  | New feature or capability                             |
| `[Fix]`   | Bug fix                                               |
| `[Chore]` | Maintenance, refactoring, dependency or build changes |
| `[Docs]`  | Documentation-only changes                            |

Rules:

- Description starts with a capital letter.
- Use imperative mood: `Add ...`, not `Added ...`.
- No trailing period.
- Keep the subject at or below roughly 70 characters.
- **Agent attribution uses the standard Git `Co-authored-by:` trailer in the commit body, not a free-form `Agent:` line.** This makes GitHub render the co-author avatar on the commit page. The trailer must be on its own line, separated from the subject by a blank line, in the form `Co-authored-by: <Display Name> <email>`. Suggested values per agent:
  - Claude (any 4.x): `Co-authored-by: Claude Opus 4.7 <noreply@anthropic.com>` (substitute the actual model, e.g. `Claude Sonnet 4.6`, `Claude Haiku 4.5`)
  - Codex: `Co-authored-by: Codex <noreply@openai.com>`
  - Copilot: `Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>`

Examples from this repo's history:

```text
[Chore] Update dependencies
[Chore] Configure Dependabot updates
[Chore] Rewrite service in Rust with Axum and chromiumoxide
```

## GitHub Actions workflows

Use the standardized workflow layout in `.github/workflows`:

- `ci.yml` runs on `main` pushes, pull requests targeting `main`, and manual dispatch.
- Rust CI order: `cargo fmt --all -- --check`, `cargo check --locked --all-targets`, `cargo clippy --locked --all-targets -- -D warnings`, then `cargo test --locked`.
- `release.yml` is the standard release build entrypoint. It runs on `v*` tags and manual dispatch, builds release artifacts, uploads them with `actions/upload-artifact`, and publishes GitHub Release assets on tag pushes.
- `docker.yml` is the standard Docker entrypoint. It runs on `main` pushes, `v*` tags, PRs that touch Docker/build inputs, and manual dispatch. PRs build only; non-PR runs push GHCR images with lowercase image names and Docker metadata tags.

Workflow maintenance rules:

- Keep workflow filenames and top-level names aligned: `CI`, `Release`, `Docker`, and optional package-specific names.
- Use `actions/checkout@v6`, `dtolnay/rust-toolchain@stable`, `Swatinem/rust-cache@v2`, `actions/upload-artifact@v7`, `actions/download-artifact@v8`, `softprops/action-gh-release@v3`, and current Docker actions (`setup-buildx@v4`, `login@v4`, `metadata@v6`, `build-push@v7`).
- Keep `permissions` minimal: `contents: read` for CI/Docker build-only work, `contents: write` for release publishing, and `packages: write` only when pushing container images.
- Use workflow `concurrency` keyed by workflow name and ref, with release jobs using `release-${{ github.ref_name }}` and `cancel-in-progress: false`.
- Do not reintroduce legacy workflow names such as `rust-ci.yml`, `build.yml`, `release-build.yml`, `docker-build.yml`, or `docker-release.yml` unless a package-specific workflow already exists and is intentionally preserved.
