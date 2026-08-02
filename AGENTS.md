# AGENTS.md

Guidance for coding agents working in this repo — the standalone repo of the **para/aether** Noeta package (a first-party web framework, pure Noeta), extracted from the noeta monorepo. Toolchain issues (the language, the `noeta` binary, `std.*`) belong in the monorepo at github.com/noeta-lang/noeta, not here.

## Repo layout

- `noeta.toml` — the package manifest (`name = "para/aether"`). No `native` key: this package is pure Noeta.
- `aether.noe` — the routing + DI core: route attributes (`Get`/`Post`/`Put`/`Patch`/`Delete`/`Query`, `Status`, and the controller-level `Group`, whose prefix `route_for` joins onto every route on that class), `App` (reflection routing + DI), `ServiceProvider`, `Middleware`/`Next`, `Config`, `Store`, sessions (`SessionStore`/`CookieSessions`), and the background-task helpers (`work`/`background`/`every`).
- `openapi.noe` — `para.aether.openapi`: the OpenAPI 3.1 document derived from the route table and the handlers' reflected signatures (`params_of` / `returns_of` / `field_specs_of`), plus `expose(app, path, info)` which serves it as an ordinary route. It imports `para.aether`; `para.aether` must never import it back, or the two modules cycle.
- `examples/*/` — each a standalone package depending on this repo via `para = { path = "../.." }`, resolving its own (untracked) `noeta.lock`.
- `.github/workflows/` — CI (`ci.yml`: a `fmt` job and an `examples` job) and the tag-triggered registry publish (`release.yml`, which reuses `ci.yml` as its gate).

## Build & test

Pure Noeta — no cargo anywhere in this repo.

- `noeta check <file>.noe` / `noeta test <file>.noe` in each `examples/*` directory is the test suite. `noeta test` never runs top-level statements, so server examples are safe to test (they only `serve` under `noeta run`).
- `noeta fmt --check .` from the repo root is the other gate, and CI runs both. Format before committing; a block that is deliberately hand-laid out (a table) opts out with `// fmt: off` / `// fmt: on` rather than being left unformatted.
- Server examples block on a real socket under `noeta run` — don't `noeta run` them in an automated session unless you intend to.

## Conventions

- `noeta.lock` files under `examples/` are **not** committed (`.gitignore` excludes them): the examples are demos resolved fresh against the current toolchain, not consumers pinning a build. A package root's own lock would be tracked; this repo has none.
- Markdown never hard-wraps lines.
- **American English** throughout — code, comments, and docs (`behavior`, not `behaviour`).
- **Conventional commits** for all commit titles. Commit each green slice as it completes, but **never `git push` without explicit authorization**. Never move a published `v*` tag — a release is a new tag.
- Implement in full — no stubs or TODOs; new functionality lands with tests.
- Keep `README.md` and this file up to date when layout or behavior changes.

## CI

`ci.yml` checks and tests every example with a pinned released `noeta`; `release.yml` publishes the tag to the hosted registry (`noeta publish`, keyless Sigstore provenance via GitHub OIDC). Both go green only once the toolchain repo is published under github.com/noeta-lang/noeta.
