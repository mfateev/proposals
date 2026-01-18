# Async Activity Completion

Complete activities asynchronously from outside the activity execution context.

## Overview

When an activity calls `KActivityContext.current().doNotCompleteOnReturn()`, it signals that the activity will be completed externally—either by a different service, a callback, or another process. Use `KActivityCompletionClient` to complete these activities.

## Use Cases

- **Callbacks**: Activity starts an operation and completes when a webhook/callback arrives
- **Human-in-the-loop**: Activity waits for manual approval
- **Cross-service**: One service starts an activity, another service completes it
- **Long-polling**: Activity polls external system; separate process completes it when ready

## Basic Usage

```kotlin
// Inside activity: mark for external completion
class PaymentActivitiesImpl : PaymentActivities {
    override suspend fun processPayment(payment: Payment): PaymentResult {
        val ctx = KActivityContext.current()

        // Save task token for external completion
        saveTaskToken(payment.id, ctx.taskToken)

        // Tell Temporal not to complete when this returns
        ctx.doNotCompleteOnReturn()

        // Start async operation (e.g., send to payment processor)
        paymentProcessor.startAsync(payment)

        // Return value is ignored when doNotCompleteOnReturn() is called
        return PaymentResult.PENDING
    }
}

// External completion (e.g., from webhook handler)
suspend fun handlePaymentWebhook(paymentId: String, result: PaymentResult) {
    val taskToken = loadTaskToken(paymentId)
    val completionClient = client.newActivityCompletionClient()

    if (result.success) {
        completionClient.complete(taskToken, result)
    } else {
        completionClient.completeExceptionally(taskToken, PaymentFailedException(result.error))
    }
}
```

## KActivityCompletionClient

Obtained from `KClient.newActivityCompletionClient()`:

```kotlin
/**
 * Client for completing activities asynchronously from outside the activity execution.
 *
 * Example:
 * ```kotlin
 * val completionClient = client.newActivityCompletionClient()
 *
 * // Complete by task token (most common)
 * completionClient.complete(taskToken, "result")
 *
 * // Or get a handle for repeated operations
 * val handle = completionClient.forTaskToken(taskToken)
 * handle.heartbeat("progress update")
 * handle.complete("final result")
 * ```
 */
class KActivityCompletionClient {
    /**
     * Creates a handle for completing an activity by task token.
     * This is the most common way to complete async activities.
     */
    fun forTaskToken(taskToken: ByteArray): KActivityCompletionHandle

    /**
     * Creates a handle for completing an activity by workflow and activity identifiers.
     * Use when task token is not available but you know the workflow/activity IDs.
     */
    fun forActivity(
        workflowId: String,
        activityId: String,
        runId: String? = null
    ): KActivityCompletionHandle

    // ==================== Convenience Methods ====================
    // For one-off completions without creating a handle

    /** Completes the activity successfully by task token. */
    suspend fun <R> complete(taskToken: ByteArray, result: R)

    /** Completes the activity with failure by task token. */
    suspend fun completeExceptionally(taskToken: ByteArray, exception: Exception)

    /** Records a heartbeat by task token. */
    suspend fun <V> heartbeat(taskToken: ByteArray, details: V)

    /** Reports cancellation by task token. */
    suspend fun <V> reportCancellation(taskToken: ByteArray, details: V)
}
```

## KActivityCompletionHandle

Handle for performing multiple operations on the same activity:

```kotlin
/**
 * Handle for completing a specific activity asynchronously.
 *
 * Obtain via [KActivityCompletionClient.forTaskToken] or
 * [KActivityCompletionClient.forActivity].
 */
sealed class KActivityCompletionHandle {
    /** Completes the activity successfully with the given result. */
    suspend fun <R> complete(result: R)

    /** Completes the activity with a failure. */
    suspend fun completeExceptionally(exception: Exception)

    /** Records a heartbeat for the activity. */
    suspend fun <V> heartbeat(details: V)

    /** Reports that the activity was cancelled. */
    suspend fun <V> reportCancellation(details: V)
}
```

## Completion Methods

### Complete Successfully

```kotlin
// By task token (most common)
completionClient.complete(taskToken, result)

// By workflow/activity IDs
val handle = completionClient.forActivity(
    workflowId = "order-123",
    activityId = "process-payment"
)
handle.complete(PaymentResult(success = true))
```

### Complete with Failure

```kotlin
// By task token
completionClient.completeExceptionally(taskToken, PaymentFailedException("Card declined"))

// By workflow/activity IDs
val handle = completionClient.forActivity(
    workflowId = "order-123",
    activityId = "process-payment"
)
handle.completeExceptionally(TimeoutException("Payment processor timeout"))
```

### Heartbeat

Keep the activity alive and report progress for long-running external operations:

```kotlin
val handle = completionClient.forTaskToken(taskToken)

// Report progress periodically
handle.heartbeat("Starting processing")
// ... later ...
handle.heartbeat(ProgressDetails(percent = 50))
// ... later ...
handle.heartbeat(ProgressDetails(percent = 100))
handle.complete(result)
```

### Report Cancellation

Confirm that the activity was cancelled:

```kotlin
// Activity was cancelled externally
handle.reportCancellation(CancellationDetails("User requested cancellation"))
```

## Complete by Workflow/Activity IDs

When the task token is not available, use workflow and activity identifiers:

```kotlin
val handle = completionClient.forActivity(
    workflowId = "order-123",
    activityId = "send-notification",
    runId = "optional-run-id"  // Optional, for disambiguation
)

handle.complete("Notification sent")
```

## Error Handling

```kotlin
try {
    completionClient.complete(taskToken, result)
} catch (e: ActivityCompletionException) {
    when {
        e.isNotFound -> logger.warn("Activity not found - may have timed out")
        else -> throw e
    }
}
```

## Full Example: Human Approval Workflow

```kotlin
// Activity interface
@ActivityInterface
interface ApprovalActivities {
    suspend fun requestApproval(request: ApprovalRequest): ApprovalResult
}

// Activity implementation - starts approval, completes externally
class ApprovalActivitiesImpl(
    private val approvalService: ApprovalService,
    private val tokenStore: TaskTokenStore
) : ApprovalActivities {

    override suspend fun requestApproval(request: ApprovalRequest): ApprovalResult {
        val ctx = KActivityContext.current()

        // Store task token for later completion
        tokenStore.save(request.id, ctx.taskToken)

        // Mark for external completion
        ctx.doNotCompleteOnReturn()

        // Create approval request in external system
        approvalService.createRequest(request)

        // Return value ignored
        return ApprovalResult.PENDING
    }
}

// Webhook handler - completes activity when approval decision is made
class ApprovalWebhookHandler(
    private val client: KClient,
    private val tokenStore: TaskTokenStore
) {
    suspend fun handleApprovalDecision(requestId: String, approved: Boolean, comment: String?) {
        val taskToken = tokenStore.load(requestId)
            ?: throw IllegalStateException("No task token for request $requestId")

        val completionClient = client.newActivityCompletionClient()

        val result = ApprovalResult(
            approved = approved,
            comment = comment,
            decidedAt = Instant.now()
        )

        completionClient.complete(taskToken, result)

        // Clean up stored token
        tokenStore.delete(requestId)
    }
}
```

## KClient Addition

```kotlin
class KClient {
    // ... existing methods ...

    /**
     * Creates a new activity completion client for completing activities asynchronously.
     *
     * Use this when activities call `doNotCompleteOnReturn()` and need to be
     * completed from outside the activity execution context.
     *
     * Example:
     * ```kotlin
     * val completionClient = client.newActivityCompletionClient()
     * completionClient.complete(taskToken, result)
     * ```
     */
    fun newActivityCompletionClient(): KActivityCompletionClient
}
```

## Comparison with Java SDK

| Java SDK | Kotlin SDK |
|----------|------------|
| `client.newActivityCompletionClient()` | `client.newActivityCompletionClient()` |
| `completionClient.complete(taskToken, result)` | `completionClient.complete(taskToken, result)` |
| `completionClient.completeExceptionally(taskToken, ex)` | `completionClient.completeExceptionally(taskToken, ex)` |
| N/A | `completionClient.forTaskToken(taskToken)` → handle |
| N/A | `completionClient.forActivity(workflowId, activityId)` → handle |

The Kotlin SDK adds handle-based APIs for cleaner repeated operations on the same activity.

## Related

- [Activity Implementation](../activities/implementation.md) - `doNotCompleteOnReturn()` and task tokens
- [Client](./workflow-client.md) - KClient reference

---

**Next:** [Advanced Operations](./advanced.md)
