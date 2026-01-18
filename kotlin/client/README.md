# Client API

This section covers the Kotlin client API for interacting with Temporal.

## Overview

The Kotlin SDK provides `KClient`, a unified client (like Python's `Client` and .NET's `TemporalClient`) with suspend functions and type-safe APIs for workflows, schedules, and async activity completion.

## Documents

| Document | Description |
|----------|-------------|
| [Client](./workflow-client.md) | KClient, starting workflows, schedules |
| [Workflow Handles](./workflow-handle.md) | Typed/Untyped handles, signals, queries, results |
| [Activity Completion](./activity-completion.md) | Async activity completion from external services |
| [Advanced Operations](./advanced.md) | SignalWithStart, UpdateWithStart |

## Quick Reference

### Creating a Client

```kotlin
val client = KClient.connect(
    KClientOptions(
        target = "localhost:7233",
        namespace = "default"
    )
)
```

### Starting Workflows

```kotlin
// Execute and wait for result
val result = client.executeWorkflow(
    GreetingWorkflow::getGreeting,
    "Temporal",
    KWorkflowOptions(
        workflowId = "greeting-123",
        taskQueue = "greetings"
    )
)

// Start async and get handle
val handle = client.startWorkflow(
    GreetingWorkflow::getGreeting,
    "Temporal",
    KWorkflowOptions(
        workflowId = "greeting-123",
        taskQueue = "greetings"
    )
)
val result = handle.result()
```

### Interacting with Workflows

```kotlin
val handle = client.workflowHandle<OrderWorkflow>("order-123")

// Signal
handle.signal(OrderWorkflow::updatePriority, Priority.HIGH)

// Query
val status = handle.query(OrderWorkflow::status)

// Update
val result = handle.executeUpdate(OrderWorkflow::addItem, newItem)

// Cancel
handle.cancel()
```

### Key Patterns

| Pattern | API |
|---------|-----|
| Execute workflow | `client.executeWorkflow(Interface::method, arg, options)` |
| Start workflow | `client.startWorkflow(Interface::method, arg, options)` |
| Get handle by ID | `client.workflowHandle<T>(workflowId)` |
| Signal with start | `client.signalWithStart(...)` |
| Update with start | `client.executeUpdateWithStart(...)` |

## Related

- [Workflow Handles](./workflow-handle.md) - Interacting with running workflows
- [Advanced Operations](./advanced.md) - Atomic operations

---

**Next:** [Workflow Client](./workflow-client.md)
