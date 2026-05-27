# Example 02 — REST vs gRPC vs GraphQL: three API styles compared

Three dominant ways for services to talk. They were designed for different problems. The right one depends on what's calling what.

## REST (over HTTP/1.1 or HTTP/2 with JSON)

The default of the web. Resources identified by URLs; verbs by HTTP methods (`GET`, `POST`, `PUT`, `DELETE`); state as JSON.

```http
GET /api/users/42
Authorization: Bearer abc
```

```json
{ "id": 42, "name": "Alice", "email": "alice@example.com" }
```

### Strengths

- **Universally supported.** Every browser, language, curl, Postman speaks HTTP.
- **Cacheable** at every layer (browser, CDN, proxy, app) because HTTP caching headers are standard.
- **Self-describing**: humans can read URLs and JSON.
- **Easy to start**: write a controller, ship.
- **Stateless** by design — scales horizontally.

### Weaknesses

- **Verbose** payloads (JSON has overhead, keys repeated).
- **Multiple endpoints** for related data → N+1 fetches from a client.
- **No strict schema** by default — both sides must agree out-of-band (or use OpenAPI).
- **Errors are loose** — every team invents its own error JSON shape.
- **Streaming is awkward** (Server-Sent Events, WebSocket, but not REST itself).

### Use REST for

- **Public APIs**. Universally understood.
- **Browser → server** when you don't need a special client.
- **Webhook delivery** (also typically HTTP+JSON POST).
- **Mobile → server** for simple use cases.

## gRPC (HTTP/2 + Protocol Buffers)

A modern RPC framework from Google. Schema-defined services, binary protocol, streaming built in.

```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
}

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
```

Code is **generated** from the `.proto` file for each language. Client and server share the same contract automatically.

### Strengths

- **Schema-first** — Protocol Buffers are strongly typed, versionable, generate native types in every language.
- **Binary protocol** — smaller payloads, much faster serialization than JSON.
- **HTTP/2 multiplexing** — many concurrent calls on one connection.
- **Streaming** — server-streaming, client-streaming, bidirectional streaming are first-class.
- **Built-in deadlines, cancellation, metadata.**
- **Cross-language**: generated stubs in Go, PHP, Java, Python, JavaScript, C++, Rust, etc.

### Weaknesses

- **Not browser-native** — needs gRPC-Web bridge for JS clients (extra moving part).
- **Not human-debuggable** — binary protocol; you need tools (`grpcurl`, BloomRPC).
- **HTTP caching doesn't work** — payload is opaque to caches.
- **Public APIs are harder** — your consumers need to handle protobuf.
- **Schema evolution discipline required** — backward compatibility rules to follow.

### Use gRPC for

- **Service-to-service** inside your infra.
- **High-throughput, low-latency** internal calls.
- **Polyglot environments** — same `.proto` generates Go and PHP clients consistently.
- **Streaming workloads** — chat, real-time updates, large file transfer.

### Real-world: gRPC is mostly internal

You rarely see public APIs over gRPC. You almost always see service-to-service over gRPC at companies that have adopted it (Google, Square, Slack, Lyft). The split is **REST for outside, gRPC for inside**.

## GraphQL

A query language for APIs. The client describes **exactly what it wants** in a single query.

```graphql
{
  user(id: 42) {
    name
    posts(last: 5) {
      title
      comments(first: 3) {
        body
        author { name }
      }
    }
  }
}
```

One request gets the user, their 5 latest posts, and 3 comments per post. **No over-fetching, no N+1.**

### Strengths

- **Client-driven**: no more "we need to add `?include=foo` to your endpoint". The client just asks.
- **No over-fetching**: get only the fields you need.
- **One round-trip** for related data.
- **Strong typing** via the schema (introspectable, documented).
- **Excellent for mobile apps** with variable, evolving needs.

### Weaknesses

- **Server-side complexity**: arbitrary queries → arbitrary DB load. You must implement query cost limits, depth limits, persisted queries, dataloaders for N+1 prevention.
- **Caching is harder**: every query is unique; HTTP caching doesn't apply (Apollo client-side cache compensates).
- **All requests are POST `/graphql`** — middlewares that filter by URL/method don't help.
- **Schema management at scale** — federating GraphQL across many backends (Apollo Federation, GraphQL Mesh) is a whole tool ecosystem.
- **Tooling has improved but is still heavier** than REST.

### Use GraphQL for

- **BFF for mobile / SPA** where clients vary in what they need.
- **Aggregating multiple downstream services** in one query.
- **Rapidly evolving frontend** that doesn't want to wait for backend endpoints.

### Real-world: GraphQL has plateaued

Hugely popular ~2017-2020, more mixed feelings since. Many teams have **moved back to REST** for public APIs, citing operational complexity. GraphQL remains strong as a **BFF** layer between rich frontends and many backend services.

## Comparison table

| Property            | REST                          | gRPC                                 | GraphQL                                       |
|---------------------|-------------------------------|--------------------------------------|------------------------------------------------|
| Transport           | HTTP/1.1 or HTTP/2            | HTTP/2 only                          | HTTP (usually POST)                            |
| Payload             | JSON (or XML, rarely)         | Protocol Buffers (binary)            | JSON                                           |
| Schema              | Optional (OpenAPI)            | Mandatory (`.proto`)                 | Mandatory (`.graphql`)                         |
| Generation          | Optional (OpenAPI codegen)    | First-class                          | First-class                                    |
| Browser-native      | Yes                           | No (need gRPC-Web)                   | Yes                                            |
| Streaming           | Awkward (SSE, WebSocket)      | Native (bi-directional)              | Subscriptions over WebSocket                   |
| Caching             | Excellent (HTTP-native)       | None (binary opaque)                 | Hard (client-side libraries help)              |
| Best for            | Public APIs, simple CRUD      | Internal service-to-service          | BFF / mobile / aggregating multiple services   |
| Over-fetching       | Common                        | Common                                | Solved                                         |
| Latency             | Good                          | Best                                  | Variable (depends on resolvers)                |
| Debuggability       | Excellent (curl, browser)     | Lower (grpcurl)                       | Good (GraphQL Playground)                      |

## A common multi-layer setup

```
Mobile / Web → GraphQL BFF → REST (public services) + gRPC (internal services)
```

- GraphQL aggregates for the client.
- REST for partner integrations.
- gRPC inside the data center.

**Pick per pair of services**, not per company.

## Architect's takeaway

- **REST for outside (public APIs, browsers, webhooks).** Cacheable, universal, simple.
- **gRPC for inside (service-to-service).** Fast, strict, generated clients.
- **GraphQL for client-driven aggregation** (mobile, SPAs hitting many services).
- **Don't standardize on one style** for everything — match the style to the caller.
- **Schema discipline** matters more than the style choice. Whether you use OpenAPI for REST or `.proto` for gRPC, document your contracts.
