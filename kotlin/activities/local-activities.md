# Local Activities

Local activities use the same stub-less pattern as regular activities.

## Local Activity Execution

```kotlin
@ActivityInterface
interface ValidationActivities {
    suspend fun validate(input: String): Boolean
    fun sanitize(input: String): String  // Non-suspend also supported
}

val isValid = KWorkflow.executeLocalActivity(
    ValidationActivities::validate,
    input,
    KLocalActivityOptions(startToCloseTimeout = 5.seconds)
)

val sanitized = KWorkflow.executeLocalActivity(
    ValidationActivities::sanitize,
    input,
    KLocalActivityOptions(startToCloseTimeout = 1.seconds)
)
```

## KWorkflow Local Activity Methods

```kotlin
object KWorkflow {
    /**
     * Execute a local activity with type-safe method reference.
     */

    // 0 arguments
    suspend fun <T, R> executeLocalActivity(
        activity: KFunction1<T, R>,
        options: KLocalActivityOptions
    ): R

    // 1 argument - passed directly
    suspend fun <T, A1, R> executeLocalActivity(
        activity: KFunction2<T, A1, R>,
        arg1: A1,
        options: KLocalActivityOptions
    ): R

    // 2+ arguments - use kargs() wrapper for type safety
    suspend fun <T, A1, A2, R> executeLocalActivity(
        activity: KFunction3<T, A1, A2, R>,
        args: KArgs2<A1, A2>,
        options: KLocalActivityOptions
    ): R

    // ... up to 6 arguments with KArgs
}
```

## KLocalActivityOptions

```kotlin
/**
 * Kotlin-native local activity options.
 */
data class KLocalActivityOptions(
    val startToCloseTimeout: Duration? = null,
    val scheduleToCloseTimeout: Duration? = null,
    val scheduleToStartTimeout: Duration? = null,
    val localRetryThreshold: Duration? = null,
    val retryOptions: KRetryOptions? = null,
    val doNotIncludeArgumentsIntoMarker: Boolean = false,
    // Experimental
    @Experimental val summary: String? = null
)
```

## KActivityContext in Local Activities

`KActivityContext.current()` is available in local activities with limited functionality:

| Feature | Local Activity Behavior |
|---------|------------------------|
| `ctx.info` | Works (`info.isLocal` returns `true`) |
| `ctx.heartbeat()` | No-op (ignored) |
| `ctx.lastHeartbeatDetails<T>()` | Returns `null` |
| `ctx.taskToken` | Throws `UnsupportedOperationException` |
| `ctx.doNotCompleteOnReturn()` | Throws `UnsupportedOperationException` |

## Related

- [Activity Definition](./definition.md) - Interface patterns
- [Activity Implementation](./implementation.md) - Full implementation details

---

**Next:** [Client](../client/README.md)
