# KArgs - Type-Safe Multi-Argument Wrapper

The Kotlin SDK uses `kargs()` to provide compile-time type safety for methods with multiple arguments.

## Why KArgs?

When calling activities, workflows, signals, or other Temporal operations with **2 or more arguments**, using varargs (`vararg args: Any?`) loses type safety - wrong argument types or counts are only caught at runtime.

The `kargs()` function wraps multiple arguments in a type-safe container that the compiler can verify against the target method signature.

## Argument Patterns

| Arguments | Pattern | Example |
|-----------|---------|---------|
| 0 args | `(method, options)` | `executeActivity(Activity::noArgs, options)` |
| 1 arg | `(method, arg, options)` | `executeActivity(Activity::oneArg, value, options)` |
| 2+ args | `(method, kargs(...), options)` | `executeActivity(Activity::twoArgs, kargs(a, b), options)` |

## Usage Examples

### Activities

```kotlin
// 0 arguments
KWorkflow.executeActivity(
    GreetingActivities::getDefault,
    KActivityOptions(startToCloseTimeout = 30.seconds)
)

// 1 argument - passed directly
KWorkflow.executeActivity(
    GreetingActivities::greet,
    name,
    KActivityOptions(startToCloseTimeout = 30.seconds)
)

// 2 arguments - use kargs()
KWorkflow.executeActivity(
    GreetingActivities::composeGreeting,
    kargs("Hello", name),
    KActivityOptions(startToCloseTimeout = 30.seconds)
)

// 3+ arguments
KWorkflow.executeActivity(
    OrderActivities::processOrder,
    kargs(orderId, customerId, items),
    KActivityOptions(startToCloseTimeout = 1.minutes)
)
```

### Child Workflows

```kotlin
// 1 argument
KWorkflow.executeChildWorkflow(
    ChildWorkflow::process,
    order,
    KChildWorkflowOptions(workflowId = "child-1")
)

// 2 arguments
KWorkflow.executeChildWorkflow(
    ChildWorkflow::processWithConfig,
    kargs(order, config),
    KChildWorkflowOptions(workflowId = "child-1")
)
```

### Client Workflow Execution

```kotlin
// 1 argument
client.executeWorkflow(
    OrderWorkflow::processOrder,
    order,
    KWorkflowOptions(workflowId = "order-123", taskQueue = "orders")
)

// 2 arguments
client.executeWorkflow(
    OrderWorkflow::processWithOptions,
    kargs(order, processingOptions),
    KWorkflowOptions(workflowId = "order-123", taskQueue = "orders")
)
```

### Signals

```kotlin
// 1 argument
handle.signal(OrderWorkflow::updateStatus, newStatus)

// 2 arguments
handle.signal(OrderWorkflow::updateWithReason, kargs(newStatus, reason))
```

### Queries

```kotlin
// 1 argument
val result = handle.query(OrderWorkflow::getItemByIndex, index)

// 2 arguments
val result = handle.query(OrderWorkflow::getItemsInRange, kargs(start, end))
```

### Updates

```kotlin
// 1 argument
val result = handle.executeUpdate(OrderWorkflow::addItem, item)

// 2 arguments
val result = handle.executeUpdate(
    OrderWorkflow::addItemWithPriority,
    kargs(item, priority),
    KUpdateOptions(updateId = "add-item-1")
)
```

### String-based (Untyped) APIs

The same pattern applies to string-based APIs:

```kotlin
// 1 argument
KWorkflow.executeActivity<String>(
    "greet",
    name,
    KActivityOptions(startToCloseTimeout = 30.seconds)
)

// 2 arguments
KWorkflow.executeActivity<String>(
    "composeGreeting",
    kargs("Hello", name),
    KActivityOptions(startToCloseTimeout = 30.seconds)
)
```

## Type Safety

The `kargs()` function creates typed wrapper classes that ensure compile-time verification:

```kotlin
// Compile error! Wrong argument types
KWorkflow.executeActivity(
    GreetingActivities::composeGreeting,  // expects (String, String)
    kargs(123, true),  // ✗ Type mismatch: expected String, String
    options
)

// Compile error! Wrong number of arguments
KWorkflow.executeActivity(
    GreetingActivities::composeGreeting,  // expects 2 args
    kargs("only one"),  // ✗ kargs() requires at least 2 arguments
    options
)
```

## KArgs Classes

The SDK provides `KArgs` classes for 2-6 arguments:

```kotlin
sealed interface KArgs

data class KArgs2<A1, A2>(val a1: A1, val a2: A2) : KArgs
data class KArgs3<A1, A2, A3>(val a1: A1, val a2: A2, val a3: A3) : KArgs
data class KArgs4<A1, A2, A3, A4>(val a1: A1, val a2: A2, val a3: A3, val a4: A4) : KArgs
data class KArgs5<A1, A2, A3, A4, A5>(val a1: A1, val a2: A2, val a3: A3, val a4: A4, val a5: A5) : KArgs
data class KArgs6<A1, A2, A3, A4, A5, A6>(val a1: A1, val a2: A2, val a3: A3, val a4: A4, val a5: A5, val a6: A6) : KArgs

// Factory functions with type inference
fun <A1, A2> kargs(a1: A1, a2: A2) = KArgs2(a1, a2)
fun <A1, A2, A3> kargs(a1: A1, a2: A2, a3: A3) = KArgs3(a1, a2, a3)
fun <A1, A2, A3, A4> kargs(a1: A1, a2: A2, a3: A3, a4: A4) = KArgs4(a1, a2, a3, a4)
fun <A1, A2, A3, A4, A5> kargs(a1: A1, a2: A2, a3: A3, a4: A4, a5: A5) = KArgs5(a1, a2, a3, a4, a5)
fun <A1, A2, A3, A4, A5, A6> kargs(a1: A1, a2: A2, a3: A3, a4: A4, a5: A5, a6: A6) = KArgs6(a1, a2, a3, a4, a5, a6)
```

## Best Practice: Single Parameter Objects

For methods with many parameters, consider using a single parameter object instead:

```kotlin
// Instead of many arguments...
data class OrderProcessingParams(
    val orderId: String,
    val customerId: String,
    val items: List<Item>,
    val priority: Priority = Priority.NORMAL,
    val expedited: Boolean = false
)

// Use a single parameter - no kargs() needed
KWorkflow.executeActivity(
    OrderActivities::processOrder,
    OrderProcessingParams(orderId, customerId, items),
    KActivityOptions(startToCloseTimeout = 1.minutes)
)
```

This approach:
- Avoids needing `kargs()` entirely
- Allows optional fields with defaults
- Is more readable for complex inputs
- Works well with Kotlin data classes

## Related

- [Activity Definition](./activities/definition.md) - Activity execution patterns
- [Child Workflows](./workflows/child-workflows.md) - Child workflow execution
- [Workflow Client](./client/workflow-client.md) - Client workflow execution

---

**Next:** [Kotlin Idioms](./kotlin-idioms.md)
