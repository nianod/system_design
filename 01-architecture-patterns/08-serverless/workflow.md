# Serverless: Execution Workflows

## Overview

This document walks through the concrete execution paths a serverless system handles: a synchronous HTTP request, an asynchronous event trigger, an orchestrated multi-step business process, and the difference a cold start makes to each. It builds on the order-processing example introduced in [readme.md](./readme.md) — `CreateOrder`, `DynamoDB`, `EventBridge`, and `SendEmail`.

## Table of Contents

- [Synchronous: HTTP Request via API Gateway](#synchronous-http-request-via-api-gateway)
- [Asynchronous: Event-Triggered Execution](#asynchronous-event-triggered-execution)
- [Orchestrated: Multi-Function Business Process](#orchestrated-multi-function-business-process)
- [Cold Start vs Warm Start Execution Paths](#cold-start-vs-warm-start-execution-paths)

## Synchronous: HTTP Request via API Gateway

The client makes a request and blocks waiting for a response. API Gateway terminates the HTTP connection, invokes the function synchronously, and returns the function's result directly to the client.

```mermaid
sequenceDiagram
    participant C as Client
    participant AG as API Gateway
    participant L as Lambda: CreateOrder
    participant DB as DynamoDB

    C->>AG: POST /orders {items, customerId}
    AG->>L: Invoke (sync)
    L->>DB: PutItem(order)
    DB-->>L: Success
    L-->>AG: 201 Created {orderId}
    AG-->>C: 201 Created {orderId}
```

Because the client is waiting, this path is latency-sensitive — a cold start here is directly visible to the user. This is the pattern to reach for provisioned concurrency on (see [challenges.md](./challenges.md#cold-starts)) if the endpoint is on a critical user-facing path.

## Asynchronous: Event-Triggered Execution

The producer doesn't wait for the consumer. A function publishes an event or an object lands in storage, and one or more downstream functions are invoked independently, off the request path.

### Object Storage Trigger

```mermaid
sequenceDiagram
    participant U as User Upload
    participant S3 as S3 Bucket
    participant L as Lambda: ProcessImage

    U->>S3: PUT product-image.jpg
    S3-->>L: ObjectCreated event (async)
    L->>L: Resize / generate thumbnails
    L->>S3: PUT thumbnails/product-image-thumb.jpg
```

### Event Bus Trigger

```mermaid
sequenceDiagram
    participant L1 as Lambda: CreateOrder
    participant EB as EventBridge
    participant L2 as Lambda: SendEmail
    participant L3 as Lambda: UpdateInventory

    L1->>EB: Publish OrderCreated
    Note over L1: CreateOrder already<br/>returned 201 to the client
    EB->>L2: Deliver OrderCreated
    EB->>L3: Deliver OrderCreated
    L2->>L2: Send confirmation email
    L3->>L3: Decrement stock
```

The key property: `CreateOrder` returns to the caller as soon as it has published the event, without waiting for `SendEmail` or `UpdateInventory` to finish. If either of those fails, the platform retries the invocation automatically (and eventually routes to a dead-letter queue if configured) — the caller is never blocked on, or aware of, that retry.

## Orchestrated: Multi-Function Business Process

A single event fan-out (as above) works for independent side effects, but a business process with sequencing, branching, and compensation logic — an order-fulfillment saga — needs an explicit orchestrator rather than implicit choreography through an event bus. Step Functions (AWS) / Durable Functions (Azure) / Workflows (GCP) fill this role.

```mermaid
graph TB
    Start([OrderCreated]) --> Reserve[Lambda: ReserveInventory]
    Reserve -->|Success| Charge[Lambda: ChargePayment]
    Reserve -->|Failure| Fail1[Lambda: NotifyOutOfStock]

    Charge -->|Success| Ship[Lambda: ScheduleShipping]
    Charge -->|Failure| Compensate1[Lambda: ReleaseInventory]

    Ship --> Confirm[Lambda: SendConfirmation]
    Compensate1 --> Fail2[Lambda: NotifyPaymentFailed]

    Confirm --> End([Order Fulfilled])
    Fail1 --> EndFail([Order Rejected])
    Fail2 --> EndFail
```

```json
{
  "Comment": "Order fulfillment saga",
  "StartAt": "ReserveInventory",
  "States": {
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ReserveInventory",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "NotifyOutOfStock" }],
      "Next": "ChargePayment"
    },
    "ChargePayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ChargePayment",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "ReleaseInventory" }],
      "Next": "ScheduleShipping"
    },
    "ScheduleShipping": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ScheduleShipping",
      "Next": "SendConfirmation"
    },
    "SendConfirmation": { "Type": "Task", "Resource": "arn:aws:lambda:us-east-1:123456789012:function:SendConfirmation", "End": true },
    "ReleaseInventory": { "Type": "Task", "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ReleaseInventory", "Next": "NotifyPaymentFailed" },
    "NotifyPaymentFailed": { "Type": "Task", "Resource": "arn:aws:lambda:us-east-1:123456789012:function:NotifyPaymentFailed", "End": true },
    "NotifyOutOfStock": { "Type": "Task", "Resource": "arn:aws:lambda:us-east-1:123456789012:function:NotifyOutOfStock", "End": true }
  }
}
```

The orchestrator tracks state, retries, and compensation transitions explicitly — the individual functions stay simple and stateless, and the saga's control flow lives in one auditable definition instead of being implicitly scattered across event handlers.

## Cold Start vs Warm Start Execution Paths

```mermaid
graph TB
    Invoke[Invocation Arrives] --> Lookup{Warm environment<br/>exists for this function?}

    Lookup -->|Yes| WarmPath[Reuse environment]
    WarmPath --> RunHandler1[Run handler code]
    RunHandler1 --> ReturnWarm[Return response<br/>~1-50ms total]

    Lookup -->|No| ColdPath[Download deployment package]
    ColdPath --> StartRuntime[Start language runtime]
    StartRuntime --> RunInit[Run module-level init code<br/>DB clients, SDK setup]
    RunInit --> RunHandler2[Run handler code]
    RunHandler2 --> ReturnCold[Return response<br/>100ms-several seconds total]

    style WarmPath fill:#d4edda
    style ColdPath fill:#f8d7da
```

Two practical takeaways from this diagram:

1. **Module-level initialization only runs on a cold start.** Code placed outside the handler function (creating a database client, loading configuration) executes once per cold environment and is reused by every subsequent warm invocation on that instance — so expensive setup should live there, not inside the handler.
2. **Concurrency, not request rate, determines how many cold starts you see.** A steady trickle of requests mostly hits warm instances; a sudden burst that exceeds currently-running instances forces the platform to cold-start new ones to keep up, regardless of how low the average request rate is.
