# para/aether

A first-party web framework for Noeta — controllers, reflection-based routing, dependency injection, middleware, sessions, and background tasks, in pure Noeta over `std.http`.

Controllers are classes whose methods are tagged `#[Get("/path")]` / `#[Post("/path")]`. The `App` autodiscovers those routes by reflection (`attributes_of`), and for each request it reflects the handler's parameters (`params_of`) and injects each one by its declared type — a request body materialized into a typed struct, a bound service, the live `Request` — then dispatches by name (`invoke`). Laravel-style method injection, zero per-handler glue.

## What it provides

One module, `para.aether`:

- **Routing** — `#[Get(path)]` / `#[Post(path)]` method attributes; `app.discover()` builds the route table by reflection; `app.serve(port)` runs it on the bundled HTTP server.
- **Dependency injection** — handler parameters are injected by declared type: a typed request body (via `@derive(Deserialize<Json>)`), a service bound with `app.bind(name, service)` (injected at a `dyn`-typed parameter), or the live `Request`.
- **Service providers** — the `ServiceProvider` trait (`register(app)`) modularizes a feature's wiring; installed with `app.provider(...)`.
- **Middleware** — the `Middleware`/`Next` onion (`app.use_middleware(...)`): a layer receives the request *and* the rest of the chain, so it can rewrite the request, inspect the response, or short-circuit.
- **Config** — the `Config` registry: `app.configure(key, value)` / `app.setting(key, fallback)`.
- **Sessions** — the `SessionStore` interface plus the stateless `CookieSessions` store (the whole session in a signed cookie, via `std.session`). The stored, revocable upgrade is the companion package [para/aether_db](https://github.com/noeta-lang/para-aether-db).
- **Background tasks** — `work(rx)` (a worker loop), `background(tx, job)` (queue a job off the request path), and `every(ms, action)` (a scheduled tick), all running alongside `serve` in one `concurrent` nursery.

## Installation

```toml
[dependencies]
para = { version = "^0.1", package = "para/aether" }
```

The package is keyed `para`, so its module addresses as `para.aether`. It is pure Noeta — no `[trust]` entry needed.

## Usage

```noeta
use para.aether.{App, Get, Post}
use std.json

@derive(Deserialize<Json>)
struct CreateUser {
    name: string
    age: int
}

class UsersController {
    fn new(): UsersController { return UsersController {} }

    #[Post("/users")]
    fn create(body: CreateUser): string {
        return "created ${body.name} (${body.age})"
    }

    #[Get("/health")]
    fn health(): string { return "ok" }
}

app = App.new()
app.register("UsersController", UsersController.new())
app.discover()
app.serve(8080)
```

A `POST /users` with a JSON body arrives at `create` as a typed `CreateUser` — decoding, routing, and dispatch are all derived from the signature and the attributes.

## Examples

- [`examples/aether-demo/`](examples/aether-demo) — service providers, DI, routing, and config in one app.
- [`examples/aether-sessions/`](examples/aether-sessions) — request context, `Response` handlers, the middleware onion, and stateless cookie sessions.
- [`examples/aether-background/`](examples/aether-background) — background jobs and a scheduled tick alongside `serve`.

## Requirements

None beyond the `noeta` toolchain — this package is pure Noeta.

## Development

Each directory under `examples/` is its own small package depending on this repo by path; run `noeta check` / `noeta test` there. See [AGENTS.md](AGENTS.md) for the repo layout and environment details.

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or <http://opensource.org/licenses/MIT>)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
