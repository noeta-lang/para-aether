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

A `POST /users` with a JSON body arrives at `create` as a typed `CreateUser` — decoding, routing, and dispatch are all derived from the signature and the attributes. The rest of this README walks each layer of that machinery.

## Controllers and routing — the attributes are the route table

`Get` and `Post` are `@attribute(Method)` structs, each carrying a `path`. Register a controller instance under its type name with `app.register("UsersController", UsersController.new())` — the name must match the class, because it is the leading segment of the reflected route target — then `app.discover()` sweeps every `#[Get]` / `#[Post]` in the program into the route table. Routing is exact-segment matching, with one `{param}` capture per pattern:

```noeta
#[Get("/users/{id}")]
fn show(user: User): string { ... }
```

The captured segment feeds **route-model binding** (below): it is how a `bind_model`'d parameter gets loaded by id. One positional capture per route is the current slice; named or multiple captures are not supported yet.

A handler's return value becomes the reply at the one point the runtime value exists, so the declared return type may be `string`, `Response`, or `dyn`:

| handler returns | reply |
| --- | --- |
| `string` | a `200` with that body, `content-type: text/plain` |
| `Response` (from `std.http`) | sent verbatim — status and headers survive, so a handler can set a `Set-Cookie` or a `404` itself |

An unmatched path is a `404 Not Found`; a route whose controller was never registered, an invocation error, or a parameter that cannot be injected is a `500` whose body names the problem.

## Dependency injection — parameters resolved by declared type

For each request, `App` reflects the matched handler's parameters (`params_of`) and builds the argument list one parameter at a time:

| parameter type | injected with |
| --- | --- |
| `dyn Trait` | the service registered under that interface with `app.bind(name, service)` |
| `Request` | the live HTTP request (server path only) |
| a `bind_model`'d type | the model loaded from the store by the route's `{id}` (see the `Store` seam below) |
| any other struct/class | the request body, decoded via `json.decode_typed` — the type needs `@derive(Deserialize<Json>)` |

Services are keyed by the interface's **short trait name**: a `dyn SessionStore` parameter resolves the instance bound as `app.bind("SessionStore", sessions)`, whether the trait was imported from `para.aether` or defined locally. Re-binding a name replaces the service. A `dyn` parameter with no registered service is a *configuration* error, not a client error — the request fails with a `500` naming the interface and how to bind it, rather than silently injecting the wrong thing.

Injecting the store into a handler works the same way: `App.new()` registers its `Store` in the container, and `app.set_store(...)` keeps that entry in step, so a `dyn Store` parameter always resolves to the live store.

> [!WARNING]
> `Request` is matched by the extern type's short name, so a request-body struct must not be named `Request` — it would be caught by the request-injection arm instead of being decoded from the body.

## Service providers — modular wiring

A `ServiceProvider` packages one feature's setup — its controllers, config, bindings, middleware — behind a single `register(app)`:

```noeta
use para.aether.{App, ServiceProvider}

class UsersProvider {
    fn new(): UsersProvider { return UsersProvider {} }
    impl ServiceProvider {
        fn register(app: App): void {
            app.register("UsersController", UsersController.new())
            app.configure("users.max", "100")
        }
    }
}

app = App.new()
app.provider(UsersProvider.new())
app.discover()
```

`app.provider(p)` runs the provider's `register` immediately, so an app composes features by installing a list of providers before `discover()`.

## Middleware — an onion, not before/after hooks

A middleware is a layer: `handle` receives the request *and* `next` — the rest of the pipeline, with the routed handler at the bottom. `next.run(req)` continues inward; the innermost `run` bottoms out in the app's dispatch. That shape lets one layer do any of three things: rewrite the request before continuing (`next.run(rewritten)`), inspect or replace the response on the way out, or answer *without ever calling `next`* — an auth reject, a cache hit, a preflight — which a before/after hook pair cannot express.

```noeta
use para.aether.{Middleware, Next}
use std.http.server
use std.http.{Request, Response}

class Stamp {
    fn new(): Stamp { return Stamp {} }

    impl Middleware {
        fn handle(req: Request, next: Next): Response {
            return next.run(req).with_header("x-app", "aether")
        }
    }
}

app.use_middleware(Stamp.new())
```

Layers run in registration order: the first middleware registered is the outermost — it sees the request first and the response last.

## Sessions — one interface, stateless by default

Handlers read and write sessions through the `SessionStore` interface — `open(req)` yields the request's `Session` (an empty one when the cookie is absent, forged, or expired, so the caller has a single correct path), and `attach(resp, session)` persists it onto the reply. `Session` is `std.session`'s own type, so backends are interchangeable at the call site.

The bundled backend is `CookieSessions`: the whole session rides in a signed, HMAC'd cookie. No server state, nothing to configure, and correct under parallel serving — every worker can read a cookie none of them wrote. `CookieSessions.new(keys, max_age, secure)` takes a `std.session` `Keyring`, a max age in seconds, and whether the cookie is https-only.

```noeta
use para.aether.{App, Get, Post, CookieSessions, SessionStore}
use std.http.server
use std.http.{Request, Response}
use std.session

class Home {
    fn new(): Home { return Home {} }

    #[Get("/")]
    fn index(req: Request, sessions: dyn SessionStore): Response {
        s = sessions.open(req)
        who = s.get("user") ?? "guest"
        return server.response(200, "hello, ${who}")
    }

    #[Post("/login")]
    fn login(req: Request, sessions: dyn SessionStore): Response {
        s = sessions.open(req).set("user", "ada")
        return sessions.attach(server.response(200, "logged in"), s)
    }
}

keys = session.keyring(["0123456789abcdef0123456789abcdef"])
app = App.new()
app.register("Home", Home.new())
app.bind("SessionStore", CookieSessions.new(keys, 3600, false))
app.discover()
app.serve(8080)
```

Because the store arrives by injection, swapping the stateless default for a stored, revocable backend — [para/aether_db](https://github.com/noeta-lang/para-aether-db), which keeps only a small id in the cookie and the data server-side — is a one-line change at the `bind` site, with no handler touched.

> [!NOTE]
> `secure: false` is a dev setting for plain-http localhost. In production load the keyring secret from the environment and pass `secure: true`. `max_age` bounds a stolen token's lifetime — which matters precisely because a stateless session has nothing to revoke against.

## Config — an app-wide string registry

`Config` is a key/value store shared through the app: providers write it during setup (`app.configure("users.max", "100")`), handlers and wiring code read it with a fallback (`app.setting("users.max", "50")`). Values are strings — the caller parses richer shapes (a port to `int`, a JSON blob via `json.decode`).

## The `Store` seam — route-model binding and unit-of-work

aether is deliberately database-agnostic: it depends on the `Store` interface, not on any concrete driver. Back it with [para/db](https://github.com/noeta-lang/para-db) over SQLite/Postgres, an in-memory fake in tests, or anything else, and wire it in with `app.set_store(...)`.

| `Store` method | role |
| --- | --- |
| `load(model, id): dyn` | load a bound model instance by its route id (route-model binding) |
| `stage(model, data): void` | stage a JSON write into the request's unit-of-work |
| `flush(): void` | commit every staged write — called once by aether at end-of-request |

Two Laravel-style features ride this seam:

- **Route-model binding** — `app.bind_model("User")` marks the type; a handler parameter typed `User` on a `{id}` route is then loaded via `Store.load("User", id)` and injected, instead of decoded from the body.
- **Unit-of-work** — a handler stages writes with `stage(...)` during the request; aether calls `flush()` once, after the response is produced, so a request's writes commit as one batch, off the hot path.

An unconfigured app has a no-op store: `stage`/`flush` do nothing, and `load` fails with a message telling you to call `app.set_store(...)`.

## Background tasks — a worker and a scheduler beside `serve`

Three helpers cover work that must not ride the request path: `work(rx)` is a worker loop draining a job channel, `background(tx, job)` enqueues a fire-and-forget thunk (the handler returns immediately), and `every(ms, action)` runs `action` on a fixed tick, forever. The serve loop drives sibling spawned tasks, so all of them run alongside the server in one `concurrent` nursery — no async-serve primitive needed:

```noeta
use para.aether.{work, every, background}
use std.http.server
use std.http.{Request, Response}

(tx, rx) = channel::<() -> void>(16)

fn process(): void { echo "  [bg] processed a job" }
fn heartbeat(): void { echo "  [scheduler] tick" }

async fn handle(req: Request) use (tx): Response {
    background(tx, process).await   // queued; the reply does not wait for it
    return server.response(202, "accepted")
}

async fn main() use (rx, tx): void {
    concurrent {
        spawn work(rx)
        spawn every(2, heartbeat)
        server.serve(8080, handle)
        tx.close()   // once the server stops, drain the worker and exit
    }
}
main().await
```

The same nursery shape composes with a full `App` — spawn the workers and hand the router to the server: `concurrent { spawn work(rx); spawn every(ms, action); server.serve(port, app.route) }`.

## Testing — drive the whole app with strings

`app.handle(method, path, body)` is the string-testable entry point: it resolves and dispatches exactly as the server does — DI, route-model binding, body decoding included — and returns the response *body* as a plain string (`"404 Not Found"` and `"500 …"` are the bodies of real 404/500 responses). `app.dispatch_request(method, path, body)` is the same call plus the end-of-request `Store.flush()`, so it rehearses the full request cycle with no server and no sockets:

```noeta
@test {
    fn creates_a_user(): void {
        assert(app.handle("POST", "/users", "{\"name\":\"Ada\",\"age\":36}") == "created Ada (36)")
    }
    fn unknown_route(): void { assert(app.handle("GET", "/nope", "") == "404 Not Found") }
}
```

> [!NOTE]
> There is no live request on the string path, so a handler that declares a `Request` parameter must be exercised through `serve` — `handle` fails it loudly rather than fabricating an empty request. `noeta test` never runs top-level statements, so a file that ends in `app.serve(8080)` is still safe to test.

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
