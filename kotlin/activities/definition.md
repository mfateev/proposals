# Activity Definition

## String-based Activity Execution

For calling activities by name (useful for cross-language interop or dynamic activity names):

```kotlin
// Execute activity by string name - 1 argument passed directly
val result = KWorkflow.executeActivity<String>(
    "activityName",
    arg,
    KActivityOptions(
        startToCloseTimeout = 30.seconds,
        retryOptions = KRetryOptions(
            initialInterval = 1.seconds,
            maximumAttempts = 3
        )
    )
)

// Multiple arguments - use kargs() wrapper
val result = KWorkflow.executeActivity<String>(
    "activityName",
    kargs(arg1, arg2),
    KActivityOptions(startToCloseTimeout = 30.seconds)
)

// Parallel execution - use standard coroutineScope { async {} }
val results = coroutineScope {
    val d1 = async { KWorkflow.executeActivity<String>("activity1", arg1, options) }
    val d2 = async { KWorkflow.executeActivity<String>("activity2", arg2, options) }
    awaitAll(d1, d2)  // Returns List<String>
}
```

## Typed Activities

The typed activity API uses direct method references - no stub creation needed. This approach:
- Provides full compile-time type safety for arguments and return types
- Allows different options (timeouts, retry policies) per activity call
- Works with both Kotlin `suspend` and Java non-suspend activity interfaces
- Similar to TypeScript and Python SDK patterns

```kotlin
// Define activity interface
@ActivityInterface
interface GreetingActivities {
    suspend fun composeGreeting(greeting: String, name: String): String

    suspend fun sendEmail(email: Email): SendResult

    suspend fun log(message: String)
}

// Use @ActivityMethod only when customizing the activity name
@ActivityInterface
interface CustomNameActivities {
    @ActivityMethod(name = "compose-greeting")
    suspend fun composeGreeting(greeting: String, name: String): String
}

// In workflow - direct method reference, no stub needed
// Argument order: method reference, arguments, options
val greeting = KWorkflow.executeActivity(
    GreetingActivities::composeGreeting,  // Direct reference to interface method
    kargs("Hello", "World"),              // Multiple args use kargs() wrapper
    KActivityOptions(startToCloseTimeout = 30.seconds)
)

// Single argument - passed directly (no kargs needed)
val result = KWorkflow.executeActivity(
    GreetingActivities::sendEmail,
    email,
    KActivityOptions(
        startToCloseTimeout = 2.minutes,
        retryOptions = KRetryOptions(maximumAttempts = 5)
    )
)

// Void activities work too
KWorkflow.executeActivity(
    GreetingActivities::log,
    "Processing started",
    KActivityOptions(startToCloseTimeout = 5.seconds)
)
```

## Parameter Restrictions

**Default parameter values are allowed for methods with 0 or 1 arguments.** For methods with 2+ arguments, defaults are not allowed. This is validated at worker registration time.

```kotlin
// ✓ ALLOWED - 1 argument with default
suspend fun processOrder(priority: Int = 0): OrderResult

// ✗ NOT ALLOWED - 2+ arguments with defaults
suspend fun processOrder(orderId: String, priority: Int = 0)  // Error!

// ✓ CORRECT for 2+ arguments - use a parameter object with optional fields
data class ProcessOrderParams(
    val orderId: String,
    val priority: Int? = null
)

suspend fun processOrder(params: ProcessOrderParams)
```

**Rationale:** This aligns with Python, .NET, and Ruby SDKs which support defaults. For complex inputs with multiple parameters, the parameter object pattern avoids serialization ambiguity and cross-language issues. See [full discussion](../open-questions.md#default-parameter-values).

## Type Safety

The API uses `KFunction` reflection to extract method metadata and provides compile-time type checking:

```kotlin
// Compile error! Wrong argument types
KWorkflow.executeActivity(
    GreetingActivities::composeGreeting,
    kargs(123, true),  // ✗ Type mismatch: expected String, String
    options
)
```

## Parallel Execution

Use standard `coroutineScope { async { } }` for concurrent execution:

```kotlin
override suspend fun parallelGreetings(names: List<String>): List<String> = coroutineScope {
    names.map { name ->
        async {
            KWorkflow.executeActivity(
                GreetingActivities::composeGreeting,
                kargs("Hello", name),
                KActivityOptions(startToCloseTimeout = 10.seconds)
            )
        }
    }.awaitAll()  // Standard kotlinx.coroutines.awaitAll
}

// Multiple different activities in parallel
val (result1, result2) = coroutineScope {
    val d1 = async { KWorkflow.executeActivity(Activities::operation1, arg1, options) }
    val d2 = async { KWorkflow.executeActivity(Activities::operation2, arg2, options) }
    awaitAll(d1, d2)
}
```

> **Note:** We use standard Kotlin `async` instead of a custom `startActivity` method. The workflow's deterministic dispatcher ensures correct replay behavior.

## Java Activity Interoperability

Method references work regardless of whether the activity is defined in Kotlin or Java:

```kotlin
// Java activity interface works seamlessly
// public interface JavaPaymentActivities {
//     PaymentResult processPayment(String orderId, BigDecimal amount);
// }

val result: PaymentResult = KWorkflow.executeActivity(
    JavaPaymentActivities::processPayment,
    kargs(orderId, amount),
    KActivityOptions(startToCloseTimeout = 2.minutes)
)
```

## Activity Execution API

The `KWorkflow` object provides type-safe overloads using `KFunction` types:

```kotlin
object KWorkflow {
    // 0 arguments
    suspend fun <T, R> executeActivity(
        activity: KFunction1<T, R>,
        options: KActivityOptions
    ): R

    // 1 argument - passed directly
    suspend fun <T, A1, R> executeActivity(
        activity: KFunction2<T, A1, R>,
        arg1: A1,
        options: KActivityOptions
    ): R

    // 2+ arguments - use kargs() wrapper for type safety
    suspend fun <T, A1, A2, R> executeActivity(
        activity: KFunction3<T, A1, A2, R>,
        args: KArgs2<A1, A2>,
        options: KActivityOptions
    ): R

    // ... up to 6 arguments with KArgs

    // String-based overloads (untyped)
    // 0 arguments
    suspend inline fun <reified R> executeActivity(
        activityName: String,
        options: KActivityOptions
    ): R

    // 1 argument - passed directly
    suspend inline fun <reified R, A> executeActivity(
        activityName: String,
        arg: A,
        options: KActivityOptions
    ): R

    // 2+ arguments - use kargs() wrapper
    suspend inline fun <reified R, A1, A2> executeActivity(
        activityName: String,
        args: KArgs2<A1, A2>,
        options: KActivityOptions
    ): R

    // ... up to 6 arguments with KArgs
}
```

## Related

- [Local Activities](./local-activities.md) - Short-lived local activities
- [Workflows](../workflows/README.md) - Calling activities from workflows

---

## Open Questions (Decision Needed)

### Interfaceless Activity Definition

**Status:** Decision needed | [Full discussion](../open-questions.md#interfaceless-workflows-and-activities)

Currently, activities require interface definitions:

```kotlin
// Current approach - requires interface
@ActivityInterface
interface GreetingActivities {
    suspend fun composeGreeting(greeting: String, name: String): String
}

class GreetingActivitiesImpl : GreetingActivities {
    override suspend fun composeGreeting(greeting: String, name: String) = "$greeting, $name!"
}
```

**Proposal:** Allow defining activities directly on implementation classes without interfaces, similar to Python SDK:

```kotlin
// Proposed approach - no interface required
class GreetingActivities {
    suspend fun composeGreeting(greeting: String, name: String) = "$greeting, $name!"
}

// In workflow - call using method reference to impl class
val result = KWorkflow.executeActivity(
    GreetingActivities::composeGreeting,
    kargs("Hello", "World"),
    KActivityOptions(startToCloseTimeout = 30.seconds)
)
```

**Benefits:**
- Reduces boilerplate (no separate interface file)
- More similar to Python SDK experience
- Kotlin-only feature, no Java SDK changes required

**Trade-offs:**
- Different from Java SDK convention
- Activity name derived from method name (convention-based, respects `@ActivityMethod(name = "...")`)

---

### Type-Safe Activity Arguments

**Status:** Decided - Option C (KArgs wrapper classes)

For compile-time type-safe activity arguments:

- **0 arguments:** Just method reference and options
- **1 argument:** Passed directly (most common case)
- **2+ arguments:** Use `kargs()` wrapper for type safety

```kotlin
// 0 arguments
KWorkflow.executeActivity(
    GreetingActivities::getDefaultGreeting,
    KActivityOptions(startToCloseTimeout = 30.seconds)
)

// 1 argument - passed directly
KWorkflow.executeActivity(
    GreetingActivities::greet,
    name,
    KActivityOptions(startToCloseTimeout = 30.seconds)
)

// 2+ arguments - use kargs() wrapper
KWorkflow.executeActivity(
    GreetingActivities::composeGreeting,
    kargs("Hello", "World"),  // KArgs2<String, String> - type checked!
    KActivityOptions(startToCloseTimeout = 30.seconds)
)
```

See [full discussion](../open-questions.md#type-safe-activityworkflow-arguments) for rationale.

---

**Next:** [Activity Implementation](./implementation.md)
