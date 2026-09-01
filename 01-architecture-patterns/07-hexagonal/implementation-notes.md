# Hexagonal Architecture: Implementation Notes

## Overview

This document walks through structuring a real codebase around hexagonal architecture: how to lay out folders, where wiring happens, how to keep the core honest about not depending on infrastructure, and the pitfalls teams most commonly hit the first time they try it.

## Table of Contents

- [Recommended Folder Structure](#recommended-folder-structure)
- [Step-by-Step Implementation](#step-by-step-implementation)
- [The Composition Root](#the-composition-root)
- [Keeping the Core Framework-Agnostic](#keeping-the-core-framework-agnostic)
- [Migrating an Existing Layered App](#migrating-an-existing-layered-app)
- [Common Pitfalls](#common-pitfalls)
- [Related Documents](#related-documents)

## Recommended Folder Structure

A structure that makes the dependency rule hard to violate by accident — the `core/` directory should never import from `adapters/`:

```
order-service/
├── core/
│   ├── domain/                  # Entities, value objects, domain events
│   │   ├── order.ts
│   │   └── order-status.ts
│   ├── ports/                   # Interfaces the core defines
│   │   ├── place-order.usecase.ts       # primary port
│   │   ├── order-repository.ts          # secondary port
│   │   └── notification-sender.ts       # secondary port
│   └── use-cases/                # Implementations of primary ports
│       ├── place-order.ts
│       └── cancel-order.ts
├── adapters/
│   ├── primary/
│   │   ├── http/
│   │   │   └── order-controller.ts
│   │   └── cli/
│   │       └── cancel-order-command.ts
│   └── secondary/
│       ├── postgres/
│       │   └── postgres-order-repository.ts
│       ├── in-memory/
│       │   └── in-memory-order-repository.ts
│       └── smtp/
│           └── smtp-notification-sender.ts
├── composition-root.ts           # Wires adapters into use cases
└── main.ts                       # Starts the HTTP server, reads config
```

A simple lint rule enforces the boundary: forbid any import in `core/**` from `adapters/**`. Most monorepo/module boundary tools (ESLint's `import/no-restricted-paths`, Nx module boundaries, ArchUnit for Java) can express this directly.

## Step-by-Step Implementation

1. **Model the domain first, without a database in mind.** Write `Order`, `OrderStatus`, and the rules that govern valid state transitions as plain classes/functions.
2. **Write the primary ports as use-case interfaces**, one per user-facing capability (`PlaceOrderUseCase`, `CancelOrderUseCase`).
3. **Write the secondary ports the use cases need**, expressed in domain terms (`OrderRepository.findPendingOlderThan(cutoff)`, not `db.query(sql)`).
4. **Implement the use cases against the secondary ports only.** At this point the core has zero dependency on Express, Postgres, or any other library.
5. **Write one primary adapter** (usually HTTP) that translates requests into calls on the primary ports.
6. **Write one secondary adapter per external dependency** (a real database adapter, plus an in-memory one for tests).
7. **Wire everything together in a composition root**, and keep that file as the *only* place that imports both a concrete use case and a concrete adapter.

## The Composition Root

The composition root is the single seam where the abstract (ports) meets the concrete (adapters). Nothing else in the codebase should construct a concrete adapter directly.

```typescript
// composition-root.ts
export function buildApp(config: AppConfig) {
  // Secondary adapters
  const db = new Pool({ connectionString: config.databaseUrl });
  const orderRepository: OrderRepository = new PostgresOrderRepository(db);
  const notificationSender: NotificationSender = new SmtpNotificationSender(config.smtp);

  // Use cases (depend only on ports)
  const placeOrder: PlaceOrderUseCase = new PlaceOrder(orderRepository, notificationSender);
  const cancelOrder: CancelOrderUseCase = new CancelOrder(orderRepository, notificationSender);

  // Primary adapters
  const orderController = new OrderController(placeOrder, cancelOrder);

  const app = express();
  app.post("/orders", orderController.handlePost.bind(orderController));
  app.delete("/orders/:id", orderController.handleDelete.bind(orderController));
  return app;
}
```

`main.ts` becomes trivial: read configuration, call `buildApp(config)`, start listening. Tests call `buildApp` with fake config and in-memory adapters instead.

## Keeping the Core Framework-Agnostic

A few concrete habits keep infrastructure from creeping into `core/`:

- **No framework decorators in domain classes.** An `Order` entity should not carry `@Entity()` or `@Column()` decorators from an ORM — map between a plain domain object and a persistence model inside the secondary adapter instead.
- **No `Request`/`Response` objects past the primary adapter.** A use case takes a plain input DTO, not an Express `Request`.
- **Errors are domain errors, not HTTP errors.** The core throws `OrderNotFoundError`, not a `404`; the primary adapter is responsible for translating domain errors into transport-specific responses.
- **Configuration is injected, not read from `process.env` inside the core.** Reading environment variables belongs in the composition root or `main.ts`.

```typescript
// Bad: core depends on Express
export class PlaceOrder {
  execute(req: Request): Promise<void> { /* ... */ }
}

// Good: core depends only on a plain input type
export class PlaceOrder implements PlaceOrderUseCase {
  execute(input: PlaceOrderInput): Promise<Order> { /* ... */ }
}
```

## Migrating an Existing Layered App

Most layered applications are closer to hexagonal than they look — the migration is usually incremental, not a rewrite:

1. Identify the service-layer classes that currently call a concrete repository or external client directly.
2. Extract an interface for each dependency, matching the service's existing method signatures as closely as possible.
3. Move the concrete implementation behind that interface into an `adapters/secondary/` folder.
4. Inject the interface instead of the concrete class (constructor injection is enough — no DI framework required).
5. Repeat per dependency, one at a time, keeping the app shippable between each step.
6. Once every outward dependency of the service layer is behind an interface, you have a hexagonal core — the "hexagon" was there all along, just undocumented.

## Common Pitfalls

⚠️ **Defining ports after writing adapters.** If you write `PostgresOrderRepository` first and derive the interface from its method signatures, you'll end up with a port shaped like Postgres, not like the domain.

⚠️ **One giant "core" port that does everything.** Prefer several small, cohesive ports over one `OrderService` interface with 40 methods — it defeats the purpose of substitutability.

⚠️ **Skipping the in-memory adapter.** Writing a fake secondary adapter early (even before the real one) is often faster and immediately proves whether the port is well-shaped, since a fake is trivial to implement against a good interface and painful against a bad one.

⚠️ **Treating hexagonal as free.** For a small CRUD service, the layer of interfaces can genuinely be more code than the problem justifies — see [readme.md](./readme.md#trade-offs) for when to skip it.

## Related Documents

- **[readme.md](./readme.md)** — pattern overview and a full worked example
- **[ports-and-adapters.md](./ports-and-adapters.md)** — deeper treatment of port design and anti-patterns
- **[testing-strategy.md](./testing-strategy.md)** — how this structure is tested at each layer
