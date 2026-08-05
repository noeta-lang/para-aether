# para/aether

A first-party web framework for Noeta — controllers, reflection-based routing, dependency injection, middleware, sessions, and background tasks, in pure Noeta over `std.http`.

Controllers are classes whose methods are tagged `#[Get("/path")]` / `#[Post("/path")]` / … . The `App` autodiscovers those routes by reflection (`attributes_of`), and for each request it reflects the handler's parameters (`params_of`) and injects each one by its declared type — a path or query parameter parsed to the type it was declared as, a request body materialized into a typed struct, a bound service, the live `Request` — then dispatches by name (`invoke`). Laravel-style method injection, zero per-handler glue.

Because the whole route table is derived, so is its **OpenAPI document**: `para.aether.openapi` reads the same table plus each handler's reflected signature and writes the spec. Nothing is stated twice, so the spec cannot drift from the service.

## What it provides

Two modules — `para.aether` and `para.aether.openapi`:

- **Routing** — `#[Get(path)]` / `#[Post(path)]` / `#[Put(path)]` / `#[Patch(path)]` / `#[Delete(path)]` method attributes, plus `#[Group(prefix)]` on a controller for the prefix its routes share; `app.discover()` builds the route table by reflection; `app.serve(port)` runs it on the bundled HTTP server. Paths capture any number of `{name}` segments.
- **Dependency injection** — handler parameters are injected by declared type: a path or query parameter of the same **name** parsed to its declared scalar type (`?T` for "may be absent"), a typed request body (via `@derive(Deserialize<Json>)`), a service bound with `app.bind(name, service)` (injected at a `dyn`-typed parameter), or the live `Request`.
- **OpenAPI generation** — `openapi.document(app, info)` derives the whole 3.1 document from the route table and the handlers' signatures; `openapi.expose(app, "/openapi.json", info)` serves it as an ordinary route.
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
    pub fn new(): UsersController { return UsersController {} }

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

Methods are private by default from noeta 0.5 on, which is why the constructor is `pub`: wiring code outside the class calls it. Handler methods need no `pub` — the router reaches them by reflection, not as a caller — and a trait impl (`ServiceProvider.register`, `Middleware.handle`, `SessionStore.open`) is `pub` because a trait is an outward contract.

## Controllers and routing — the attributes are the route table

`Get`, `Post`, `Put`, `Patch` and `Delete` are `@attribute(Method)` structs, each carrying a `path`. There is one attribute per verb rather than one `#[Route(method: "GET", …)]` because `attributes_of` is a closed-world query **keyed by type** — a verb carried as a string could not be discovered at all, and the same property makes a misspelled `#[Gett]` a compile error instead of a route that never matches.

Register a controller instance under its type name with `app.register("UsersController", UsersController.new())` — the name must match the class, because it is the last-but-one segment of the reflected route target — then `app.discover()` sweeps every route attribute in the program into the table. Routing is exact-segment matching, with any number of `{name}` captures:

```noeta
#[Get("/users/{id}/posts/{slug}")]
fn post(id: int, slug: string): Post { ... }
```

Captures bind **by name**, not by position, so reordering the handler's parameters cannot silently swap them, and each is percent-decoded (`/tags/for%20sale` → `for sale`). The first capture in template order also feeds **route-model binding** (below): it is how a `bind_model`'d parameter gets loaded by id.

### Route groups — one prefix, stated once

`#[Group(prefix)]` on a controller is the prefix every route on it hangs under, so a version or a resource root is written once instead of repeated in each route attribute:

```noeta
#[Group("/api/v1/tags", tag: "Tags")]
class TagsController {
    #[Get("")]           // → /api/v1/tags
    fn index(): List<string> { ... }

    #[Get("/{name}")]    // → /api/v1/tags/{name}
    fn show(name: string): string { ... }
}
```

It is a **class** attribute, and that is the entire mechanism: reflection keys a class attribute by the class and a method attribute by `Class.method`, so a route finds its group by dropping the last segment of its own target. Nothing registers a group, and the same closed-world `attributes_of` query that discovers the routes discovers the prefix they share.

The prefix is joined once, when the table row is built, so the table holds the **full** path and every consumer agrees by construction: the router matches it, `{…}` captures inside it are injected by name like any other, and the OpenAPI document reports the endpoint where it is actually served. There is no second, unprefixed spelling of a route anywhere — a grouped route is served at its joined path and nowhere else.

A prefix is normalized (`"api/v1"`, `"/api/v1"` and `"/api/v1/"` are the same group), so a group's spelling is never the reason a route 404s, and a handler whose own path is empty **is** the group: `#[Group("/api/v1/tags")]` with `#[Get("")]` is served at `/api/v1/tags`, without a trailing slash. The optional `tag:` names the group in the generated document, where an ungrouped controller's operations are filed under the controller's own name.

Groups compose with everything else on a route rather than replacing it: `#[Status]`, `#[Summary]` and parameter injection all join on the handler's reflection target, which grouping does not touch. Middleware is still app-wide (`app.use_middleware`) — a group moves paths, it does not scope a pipeline.

A handler's return value becomes the reply at the one point the runtime value exists, so the declared return type may be a value type, `string`, `Response`, or `dyn`:

| handler returns | reply |
| --- | --- |
| `string` | a `200` with that body, `content-type: text/plain` |
| `Response` (from `std.http`) | sent verbatim — status and headers survive, so a handler can set a `Set-Cookie` or a `404` itself |
| anything else | a `200` with the value rendered by `json.stringify`, `content-type: application/json` |
| `none` (from a `?T` handler) | a `404 Not Found` — "may be absent" is what the type says, and 404 is what HTTP calls absent |
| `Err(e)` (from a `Result` handler) | a `500` carrying the message — the `Ok` continues as any other value |

`#[Status(code)]` on a handler sets the status of the reply the framework builds (a `Response` a handler builds itself already says its own, and a `none` is a 404 regardless). It lives beside the route attributes rather than with the spec generator on purpose: the router answers with it and the document reports it, so an annotation cannot describe a status the service does not send.

The third row is what lets a handler be written as `fn show(id: int): Pet` — the shape a caller wants, and the shape the OpenAPI generator reads a response schema off.

An unmatched path is a `404 Not Found`; a route whose controller was never registered, an invocation error, or a parameter that cannot be injected is a `500` whose body names the problem. A path or query parameter that is missing or will not parse is a `400` naming the parameter and the type it had to be — a client error, reported as one.

## Dependency injection — parameters resolved by declared type

For each request, `App` reflects the matched handler's parameters (`params_of`) and builds the argument list one parameter at a time:

| parameter type | injected with |
| --- | --- |
| `dyn Trait` | the service registered under that interface with `app.bind(name, service)` |
| `Request` | the live HTTP request (server path only) |
| `string` / `int` / `float` / `f32` / `f64` / `bool` | the route's `{name}` capture, else the query parameter of that name, parsed |
| `?T` of any of those | the same, injected as `none` when absent instead of failing |
| a `bind_model`'d type | the model loaded from the store by the route's first `{…}` capture (see the `Store` seam below) |
| any other struct/class | the request body, decoded via `json.decode_typed` — the type needs `@derive(Deserialize<Json>)` |

Scalars resolve **by name**, path before query — a `{id}` in the template is part of the route's identity, so a client appending `?id=other` cannot redirect a handler that matched on the first one. A required scalar that the request does not carry is a `400`, and `?T` is how a handler says an absent value is a legitimate request. That is the same distinction the generated document reports as `required: true` / `required: false`, from the same declaration.

Services are keyed by the interface's **short trait name**: a `dyn SessionStore` parameter resolves the instance bound as `app.bind("SessionStore", sessions)`, whether the trait was imported from `para.aether` or defined locally. Re-binding a name replaces the service. A `dyn` parameter with no registered service is a *configuration* error, not a client error — the request fails with a `500` naming the interface and how to bind it, rather than silently injecting the wrong thing.

Injecting the store into a handler works the same way: `App.new()` registers its `Store` in the container, and `app.set_store(...)` keeps that entry in step, so a `dyn Store` parameter always resolves to the live store.

> [!WARNING]
> `Request` is matched by the extern type's short name, so a request-body struct must not be named `Request` — it would be caught by the request-injection arm instead of being decoded from the body.

## Service providers — modular wiring

A `ServiceProvider` packages one feature's setup — its controllers, config, bindings, middleware — behind a single `register(app)`:

```noeta
use para.aether.{App, ServiceProvider}

class UsersProvider {
    pub fn new(): UsersProvider { return UsersProvider {} }
    impl ServiceProvider {
        pub fn register(app: App): void {
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
    pub fn new(): Stamp { return Stamp {} }

    impl Middleware {
        pub fn handle(req: Request, next: Next): Response {
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
    pub fn new(): Home { return Home {} }

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

## OpenAPI — the document the app already contains

A specification is not a second artifact to keep in step with a service; it is a *projection* of it. Every fact an OpenAPI document states — which paths exist, which verb reaches each one, what a parameter is called, where it lives, what type it is and whether it is required, what the request body decodes into, what the handler answers — is something the program already declares and the compiler already knows. `para.aether.openapi` reads those facts back out by reflection and writes the document:

```noeta
use para.aether.openapi

info = openapi.Info { title: "Pet Proxy", version: "1.0.0", servers: ["http://localhost:8099"] }

app.discover()
openapi.expose(app, "/openapi.json", info)   // serve it as an ordinary route
echo openapi.document(app, info)             // or print it, to commit and diff
```

| document element | derived from |
| --- | --- |
| path, HTTP method | the `#[Get]` / `#[Post]` / … attribute, under its controller's `#[Group]` prefix |
| `operationId`, `tags` | the handler's name, and the controller's `#[Group(tag:)]` or else the controller's own |
| path parameters | the `{…}` segments of the route, joined to the handler's parameters by name |
| query parameters | every other scalar parameter of the handler |
| `required` | the parameter's type — a `?T` is optional, everything else is required; a path parameter always is |
| parameter and property schemas | the declared types (`int` → `integer`, `List<T>` → an array of `T`, `i32` → `integer/int32`, `?T` → `anyOf: [T, null]`) |
| `requestBody` | the handler's struct parameter, `$ref`'d into `components.schemas` |
| response schema | the handler's **return type** (`returns_of`) — a `string` return is `text/plain`, a `Response` is left unstated, a value type is its schema |
| a `404` response | a `?T` return: the router answers 404 for `none`, so the document states it, from the same declaration |
| a `500` response | a `Result<T, E>` return: the router answers 500 for the `Err` |
| the success status | `#[Status(code)]`, which the router itself uses |
| `components.schemas` | `field_specs_of` on every struct the walk meets, recursively; `required` is the fields that declared neither `?T` nor a default |
| `description` | the handler's `@doc` block, when the `doc` tier is live |

Two attributes exist for the two facts a signature genuinely cannot carry, and nothing else — and `#[Status]` is a *routing* attribute the document merely reports, so the two cannot disagree:

```noeta
#[Status(201, description: "Created")]     // a created resource answers 201
#[Summary("Add a pet")]                    // prose, in a build where @doc blocks are stripped
#[Post("/pets")]
fn create(body: NewPet): PetSummary { ... }
```

The document is OpenAPI **3.1** — that version's schema vocabulary is the one that can say `anyOf: [T, null]`, and Noeta's optionality is a type, not a flag. Output is deterministic (map keys are sorted), so a generated spec can be committed and its diff read.

Because the document is generated from the same table the router dispatches against, the loop closes: feed it to [para/api](https://github.com/noeta-lang/para-api)'s `@openapi("spec.json")` directive and you get a typed client for the service, generated from the service.

## Examples

- [`examples/aether-demo/`](examples/aether-demo) — service providers, DI, routing, and config in one app.
- [`examples/aether-rest/`](examples/aether-rest) — every verb, path and query parameters injected by name, JSON replies, a `#[Group]`'d resource beside an ungrouped one, and the generated OpenAPI document asserted against the code it came from.
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
