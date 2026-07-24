# AGENTS.md

Guidance for coding agents working in this repo — the standalone repo of the **para/aether** Noeta package (a first-party web framework, pure Noeta), extracted from the noeta monorepo. Toolchain issues (the language, the `noeta` binary, `std.*`) belong in the monorepo at github.com/noeta-lang/noeta, not here.

## Repo layout

- `noeta.toml` — the package manifest (`name = "para/aether"`). No `native` key: this package is pure Noeta.
- `aether.noe` — the whole surface: route attributes (`Get`/`Post`), `App` (reflection routing + DI), `ServiceProvider`, `Middleware`/`Next`, `Config`, `Store`, sessions (`SessionStore`/`CookieSessions`), and the background-task helpers (`work`/`background`/`every`).
- `examples/*/` — each a standalone package depending on this repo via `para = { path = "../.." }`, with its own committed `noeta.lock`.
- `.github/workflows/` — CI (`ci.yml`) and the tag-triggered registry publish (`release.yml`).

## Build & test

Pure Noeta — no cargo anywhere in this repo.

- `noeta check <file>.noe` / `noeta test <file>.noe` in each `examples/*` directory is the test suite. `noeta test` never runs top-level statements, so server examples are safe to test (they only `serve` under `noeta run`).
- Server examples block on a real socket under `noeta run` — don't `noeta run` them in an automated session unless you intend to.

## Conventions

- `noeta.lock` files under `examples/` **are committed** — leave resolved locks in place; don't delete or regenerate them gratuitously.
- Markdown never hard-wraps lines.
- **American English** throughout — code, comments, and docs (`behavior`, not `behaviour`).
- **Conventional commits** for all commit titles. Commit each green slice as it completes, but **never `git push` without explicit authorization**. Never move a published `v*` tag — a release is a new tag.
- Implement in full — no stubs or TODOs; new functionality lands with tests.
- Keep `README.md` and this file up to date when layout or behavior changes.

## CI

`ci.yml` checks and tests every example with a pinned released `noeta`; `release.yml` publishes the tag to the hosted registry (`noeta publish`, keyless Sigstore provenance via GitHub OIDC). Both go green only once the toolchain repo is published under github.com/noeta-lang/noeta.
