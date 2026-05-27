# Example 05 — Clean architecture: the dependency rule

Robert C. Martin's "Clean Architecture" is essentially a more prescriptive cousin of hexagonal architecture, with named layers and one big rule: **dependencies always point inward**.

## The four concentric layers

```
        ┌─────────────────────────────────────────────────┐
        │      Frameworks & Drivers (outermost)           │
        │   HTTP, DB, message queue, web UI, devices       │
        │   ┌─────────────────────────────────────────┐    │
        │   │     Interface Adapters                   │   │
        │   │   Controllers, presenters, repositories  │   │
        │   │   ┌────────────────────────────────┐     │   │
        │   │   │     Use Cases / Application    │     │   │
        │   │   │   PlaceOrder, CancelOrder      │     │   │
        │   │   │   ┌──────────────────────┐     │     │   │
        │   │   │   │      Entities         │     │     │   │
        │   │   │   │  Order, Customer,    │     │     │   │
        │   │   │   │  Product (domain)    │     │     │   │
        │   │   │   └──────────────────────┘     │     │   │
        │   │   └────────────────────────────────┘     │   │
        │   └─────────────────────────────────────────┘    │
        └─────────────────────────────────────────────────┘
```

### Entities (innermost)

The core business objects. Have **no dependencies** — pure data + invariants.

```go
type Order struct {
    ID         OrderID
    CustomerID CustomerID
    Items      []LineItem
    Status     OrderStatus
}

func (o *Order) Cancel() error {
    if o.Status == Shipped {
        return ErrCannotCancelShipped
    }
    o.Status = Cancelled
    return nil
}
```

No framework, no DB, no HTTP. Just business rules.

### Use cases (application)

Orchestrate entities to fulfill specific user goals. "Place an order", "Cancel an order", "Send a notification".

```go
type PlaceOrderUseCase struct {
    orderRepo OrderRepository    // interface
    payment   PaymentService     // interface
    eventBus  EventPublisher     // interface
}

func (uc *PlaceOrderUseCase) Execute(ctx context.Context, input PlaceOrderInput) (*Order, error) {
    order := NewOrder(input.CustomerID, input.Items)
    if err := uc.payment.Charge(ctx, order.Total, input.CustomerID); err != nil {
        return nil, err
    }
    order.Status = Placed
    if err := uc.orderRepo.Save(ctx, order); err != nil {
        return nil, err
    }
    uc.eventBus.Publish(ctx, OrderPlacedEvent{OrderID: order.ID})
    return order, nil
}
```

Use cases depend on entities (allowed; entities are inward). They depend on **interfaces** (`OrderRepository`, `PaymentService`) — not concrete implementations.

### Interface adapters

Implement the interfaces that use cases need. Translate between use cases' data shape and the outside world's shape (DB rows, HTTP requests, queue messages).

```go
// HTTP controller (adapter to HTTP world)
type OrderController struct {
    placeOrder *PlaceOrderUseCase
}

func (c *OrderController) HandlePost(w http.ResponseWriter, r *http.Request) {
    var input PlaceOrderInput
    json.NewDecoder(r.Body).Decode(&input)
    order, err := c.placeOrder.Execute(r.Context(), input)
    ...
    json.NewEncoder(w).Encode(order)
}

// DB repository (adapter to Postgres)
type PostgresOrderRepo struct{ db *sql.DB }

func (r *PostgresOrderRepo) Save(ctx context.Context, o *Order) error {
    _, err := r.db.ExecContext(ctx, "INSERT INTO orders ...", ...)
    return err
}
```

These adapters depend on the inner layers (they implement use-case interfaces, work with entities).

### Frameworks & drivers (outermost)

The concrete external stuff: HTTP server, DB driver, queue client, framework wiring.

```go
func main() {
    db := sql.Open("postgres", ...)
    repo := &PostgresOrderRepo{db: db}
    payment := &StripePayment{...}
    bus := &KafkaPublisher{...}

    placeOrder := &PlaceOrderUseCase{
        orderRepo: repo, payment: payment, eventBus: bus,
    }
    controller := &OrderController{placeOrder: placeOrder}

    http.HandleFunc("/orders", controller.HandlePost)
    http.ListenAndServe(":8080", nil)
}
```

The composition root — where wiring happens. This is the only place that knows about everything.

## The dependency rule, restated

**Source code dependencies must point only inward.**

- Entities don't import use cases.
- Use cases don't import controllers.
- Controllers don't import DB drivers (they use repository interfaces defined in inner layers).

If you find yourself importing outward, you're violating the rule. The fix is usually to define an interface in the inner layer and let the outer layer implement it.

## Why this matters

### Testability

You can unit-test entities and use cases with **mock implementations** of the interfaces. No DB, no HTTP. Fast tests.

```go
func TestPlaceOrder(t *testing.T) {
    repo := &mockRepo{}
    payment := &mockPayment{succeed: true}
    bus := &mockBus{}

    uc := &PlaceOrderUseCase{repo, payment, bus}
    order, err := uc.Execute(ctx, PlaceOrderInput{...})

    assert.NoError(t, err)
    assert.Equal(t, Placed, order.Status)
    assert.True(t, bus.Published("OrderPlacedEvent"))
}
```

### Frameworks become details

Switching from Echo to Gin to Fiber? Replace the HTTP adapter. The use cases and entities are unchanged.

Switching from REST to gRPC? Add a new adapter. Same.

### Business rules survive technology changes

Postgres → Mongo: rewrite the repo adapter.
Stripe → Adyen: rewrite the payment adapter.
RabbitMQ → Kafka: rewrite the publisher adapter.

In all cases, **the entities and use cases (the actual business value) are untouched.**

## The most common violation

A lot of "clean architecture" code in the wild looks like this:

```go
// "use case" that imports SQL
func PlaceOrder(input Input) (*Order, error) {
    db, _ := sql.Open(...)
    db.Exec("INSERT INTO orders ...", ...)
    ...
}
```

The use case directly imports `database/sql`. **Dependency points outward.** That's not clean architecture — it's just code in a use-case folder.

The fix: define `type OrderRepository interface { Save(*Order) error }` in the use case layer. Implement it in an outer adapter. Wire at composition root.

## Pragmatic vs ideological

Clean architecture taken to the extreme adds many layers, many DTOs, many mappings. For small apps, it's overkill — the layering overhead is bigger than the layered code.

A pragmatic interpretation:

- **Always isolate the domain** (entities + invariants) from the framework.
- **Use cases** are a useful boundary for orchestration — but for trivial CRUD, the use case is sometimes just "call the repo".
- **Separate persistence model from domain entity** only when the schemas genuinely differ. Otherwise, one struct is fine for both.

This balances structural cleanliness with practical code volume.

## Differences from hexagonal

- **Hexagonal**: "domain core surrounded by adapters", no further internal structure prescribed.
- **Clean architecture**: explicitly four layers (entities, use cases, adapters, frameworks). More prescriptive.

In practice, they're variants of the same idea. Most "clean architecture" projects could also be called "hexagonal". The pattern matters; the name doesn't.

## A worked-PHP layout

For a PHP/Laravel-style project applying clean architecture:

```
src/
  Domain/                    ← entities + interfaces (innermost)
    Order/
      Order.php
      OrderId.php
      OrderRepository.php   (interface)
  Application/               ← use cases
    Order/
      PlaceOrderUseCase.php
      CancelOrderUseCase.php
  Infrastructure/            ← adapters
    Persistence/
      EloquentOrderRepository.php
    Payment/
      StripePaymentService.php
  Presentation/              ← framework/HTTP wiring
    Http/
      Controllers/
        OrderController.php
      Routes.php
```

`Domain/` has zero Laravel imports. `Application/` imports `Domain/` only. `Infrastructure/` implements interfaces from `Domain/`. `Presentation/` wires it all.

## Architect's takeaway

- **Clean architecture's core rule: dependencies point inward.** Entities don't know about use cases; use cases don't know about controllers.
- **Business logic survives technology changes** — DB, framework, transport can be swapped.
- **Testability is the practical win**: fast tests with no infrastructure.
- **Don't over-layer for trivial code.** Apply with judgment.
- **The framework is a detail.** This is the most important shift in mindset.
- Clean architecture, hexagonal, onion — same idea with different diagrams.
