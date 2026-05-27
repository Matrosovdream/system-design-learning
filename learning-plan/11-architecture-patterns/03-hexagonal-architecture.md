# Example 03 — Hexagonal architecture (ports & adapters)

The core insight: your **domain logic** should not depend on **anything external** — not the DB, not the framework, not the HTTP layer. Anything external is a "port" with an "adapter" that the domain doesn't know about.

Originally Alistair Cockburn's "hexagonal" because it drew the domain as a hexagon with adapters on each side. Same idea is called "ports and adapters" or "onion architecture".

## The shape

```
                  ┌─────────────────────┐
                  │     Domain Core     │
                  │  (entities, logic)  │
                  └──────┬───────┬──────┘
                        ports (interfaces)
              ┌──────────┴───────┴──────────┐
       inbound adapter            outbound adapter
       (HTTP, CLI, queue)         (DB, API, cache)
              ↑                          ↑
            world                       world
```

- The **domain core** has no imports of any framework, DB driver, HTTP library, queue client.
- **Ports** are interfaces defined by the domain. "I need to save an Order. Here's the interface I need fulfilled."
- **Adapters** are implementations of those interfaces. "Postgres adapter saves to Postgres. Mongo adapter saves to Mongo. In-memory adapter is for tests."

## A small Go example

### Domain core

```go
// package: order

type Order struct {
    ID     string
    UserID string
    Items  []Item
    Total  Money
    Status Status
}

type OrderRepository interface {        // ← port
    Save(ctx context.Context, o *Order) error
    FindByID(ctx context.Context, id string) (*Order, error)
}

type PaymentService interface {         // ← port
    Charge(ctx context.Context, amount Money, customer string) error
}

func (o *Order) Place(ctx context.Context, repo OrderRepository, pay PaymentService) error {
    if err := o.validate(); err != nil {
        return err
    }
    if err := pay.Charge(ctx, o.Total, o.UserID); err != nil {
        return err
    }
    o.Status = Placed
    return repo.Save(ctx, o)
}
```

Notice: the domain code knows about `OrderRepository` and `PaymentService` **interfaces** but knows nothing about Postgres, Stripe, or HTTP.

### Adapters

```go
// package: postgres

type PostgresOrderRepository struct{ db *sql.DB }

func (r *PostgresOrderRepository) Save(ctx context.Context, o *order.Order) error {
    _, err := r.db.ExecContext(ctx, "INSERT INTO orders ...", o.ID, o.UserID, ...)
    return err
}

// package: stripe

type StripePaymentService struct{ client *stripe.Client }

func (s *StripePaymentService) Charge(ctx context.Context, amount order.Money, cust string) error {
    _, err := s.client.Charges.New(&stripe.ChargeParams{ Amount: int64(amount), Customer: cust })
    return err
}
```

The adapters depend on the domain. The domain doesn't depend on them. **Dependency flows inward.**

### Composition root

The application's `main()` wires the adapters into the domain:

```go
func main() {
    db := postgres.NewDB(...)
    stripeClient := stripe.NewClient(...)

    repo := &postgres.PostgresOrderRepository{db: db}
    pay := &stripe.StripePaymentService{client: stripeClient}

    handler := http.NewOrderHandler(repo, pay)
    http.ListenAndServe(":8080", handler)
}
```

The domain doesn't even know that an HTTP handler exists. The handler is itself an **inbound adapter** that translates HTTP requests into domain calls.

## What this buys you

### 1. Testability

The domain can be tested **without any external dependency**:

```go
func TestPlaceOrder(t *testing.T) {
    repo := &fakeRepo{}                  // in-memory adapter for testing
    pay := &fakePayment{shouldSucceed: true}
    order := &order.Order{...}
    err := order.Place(ctx, repo, pay)
    assert.NoError(t, err)
    assert.Equal(t, order.Placed, order.Status)
}
```

No DB. No Stripe. No HTTP. Tests run in milliseconds.

This is the single biggest practical benefit. Most of your code becomes trivially testable.

### 2. Swap implementations

Switching from Postgres to Mongo? Write a new adapter. Domain code unchanged.

Switching from Stripe to Adyen? Write a new payment adapter. Domain unchanged.

Switching from REST to gRPC for the inbound API? Add a gRPC handler that calls the same domain logic.

### 3. Resistance to framework lock-in

Your domain doesn't import Symfony, Laravel, Echo, Gin. If you migrate frameworks, the domain comes with you.

### 4. Clear separation of concerns

Business logic is in one place (the domain). Plumbing is in another (adapters). Mixing the two is the most common source of "this codebase is hard to change" complaints.

## Common shapes of the layered model

You'll see different names for the same idea:

| Term                   | Source                              | Layers (outer → inner)                                |
|------------------------|-------------------------------------|--------------------------------------------------------|
| Hexagonal (ports/adapters) | Alistair Cockburn               | Adapters → Ports → Application → Domain                |
| Onion architecture     | Jeffrey Palermo                     | Infrastructure → App services → Domain services → Domain |
| Clean architecture     | Robert C. Martin                    | Frameworks → Interface adapters → Use cases → Entities |
| DDD layered            | Eric Evans                          | UI → Application → Domain → Infrastructure             |

All express the same principle: **dependencies point inward, toward business logic.**

## The dependency rule

**Code in an inner layer doesn't know about an outer layer.**

- Domain doesn't import the DB driver.
- Use cases don't import the HTTP framework.
- Entities don't import use cases.

When you need data from an outer layer, the inner layer **defines an interface** ("I need a thing that can save an order"), and the outer layer provides it.

## What's NOT in the domain

- DB columns, SQL.
- HTTP request/response shapes.
- JSON serialization annotations.
- Framework-specific decorators.
- Logging, tracing, metrics calls (these are infrastructure — wrap the domain at the edges).

If you're tempted to put `@Column(name = "...")` on a domain entity, you're leaking infrastructure into the domain.

A common compromise: have a separate "persistence model" struct that the adapter maps to/from the domain entity. Two structs, one mapping function. Slight duplication, clean boundary.

## What goes wrong without it

- **DB schema migrations require domain logic changes** because the domain is bound to the schema.
- **Tests require a DB / Stripe / HTTP server to run.** Slow, flaky.
- **You can't switch ORMs / frameworks** without a major rewrite.
- **Business rules are scattered** across controllers, services, model classes, frontend.

Every legacy "I can't change anything in this codebase" codebase has these symptoms.

## When NOT to apply hexagonal

- **Trivial CRUD app** with no business logic. The domain is just "save row". Hexagonal adds layers for no benefit.
- **Pure data pipeline** (ETL, ingestion). The "domain" is data transformations; ports/adapters add overhead.
- **Prototype / spike** where you're discovering the domain. Build it loose first; structure it later.

Hexagonal pays off when **the domain has real complexity** — calculations, workflows, business rules. Cars-as-a-service, healthcare, finance, logistics — all benefit. A blog comments system probably doesn't.

## A common middle ground

In many real systems, you apply hexagonal **for the complex aggregates** and write simpler "controller → ORM" code for the rest.

Order placement, inventory, billing → hexagonal.
User profile CRUD, settings page → simple.

This pragmatic mix beats both pure extremes.

## Architect's takeaway

- **Hexagonal architecture isolates business logic from infrastructure.**
- **Dependencies point inward** — domain doesn't know about DB, HTTP, framework.
- **Testability is the practical win**: trivial unit tests for complex logic.
- **Adapters are the cost** — one extra layer per external concern.
- **Apply to complex domains.** Skip for trivial CRUD.
- **Don't ideologize.** Use a mix: hexagonal where complexity warrants it; simpler patterns elsewhere.
