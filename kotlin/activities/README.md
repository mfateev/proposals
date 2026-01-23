# Activities

This section covers defining and implementing Temporal activities in Kotlin.

## Overview

Activities are the building blocks for interacting with external systems. The Kotlin SDK provides type-safe activity execution with suspend function support.

## Documents

| Document | Description |
|----------|-------------|
| [Definition](./definition.md) | Activity interfaces, typed and string-based execution |
| [Implementation](./implementation.md) | Implementing activities, KActivity API, heartbeating |
| [Local Activities](./local-activities.md) | Short-lived local activities |

## Quick Reference

### Basic Activity

```kotlin
// Define activity interface - @ActivityMethod is optional
@ActivityInterface
interface GreetingActivities {
    suspend fun composeGreeting(greeting: String, name: String): String
}

// Use @ActivityMethod only when customizing the activity name
@ActivityInterface
interface CustomNameActivities {
    @ActivityMethod(name = "compose-greeting")
    suspend fun composeGreeting(greeting: String, name: String): String
}

// Implement activity
class GreetingActivitiesImpl : GreetingActivities {
    override suspend fun composeGreeting(greeting: String, name: String): String {
        return "$greeting, $name!"
    }
}
```

### Calling Activities from Workflows

```kotlin
// Type-safe method reference - single argument passed directly
val greeting = KWorkflow.executeActivity(
    GreetingActivities::greet,
    name,
    KActivityOptions(startToCloseTimeout = 30.seconds)
)

// Multiple arguments use kargs() wrapper for type safety
val greeting = KWorkflow.executeActivity(
    GreetingActivities::composeGreeting,
    kargs("Hello", "World"),
    KActivityOptions(startToCloseTimeout = 30.seconds)
)

// String-based (for cross-language interop)
val result = KWorkflow.executeActivity<String>(
    "composeGreeting",
    kargs("Hello", "World"),
    KActivityOptions(startToCloseTimeout = 30.seconds)
)
```

### Key Patterns

| Pattern | API |
|---------|-----|
| Execute activity | `KWorkflow.executeActivity(Interface::method, arg, options)` |
| Execute by name | `KWorkflow.executeActivity<R>("name", arg, options)` |
| Local activity | `KWorkflow.executeLocalActivity(Interface::method, arg, options)` |
| Heartbeat | `KActivityContext.current.heartbeat(details)` |

## Related

- [Implementation](./implementation.md) - Suspend activity patterns
- [Local Activities](./local-activities.md) - Short-lived activities

---

**Next:** [Activity Definition](./definition.md)
