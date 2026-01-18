# Child Workflows

Child workflows are invoked using direct method references - no stub creation needed.

## Execute and Wait

```kotlin
// Simple case - execute child workflow and wait for result
// Argument order: method reference, argument, options
override suspend fun parentWorkflow(): String {
    return KWorkflow.executeChildWorkflow(
        ChildWorkflow::doWork,
        "input",
        KChildWorkflowOptions(workflowId = "child-workflow-id")
    )
}

// With retry options
override suspend fun parentWorkflowWithRetry(): String {
    return KWorkflow.executeChildWorkflow(
        ChildWorkflow::doWork,
        "input",
        KChildWorkflowOptions(
            workflowId = "child-workflow-id",
            workflowExecutionTimeout = 1.hours,
            retryOptions = KRetryOptions(maximumAttempts = 3)
        )
    )
}

// Multiple arguments use kargs() wrapper for type safety
override suspend fun parentWithMultipleArgs(): String {
    return KWorkflow.executeChildWorkflow(
        ChildWorkflow::processWithConfig,
        kargs(input, config),
        KChildWorkflowOptions(workflowId = "child-workflow-id")
    )
}
```

## Parallel Execution

Use standard `coroutineScope { async {} }` for parallel child workflows:

```kotlin
override suspend fun parentWorkflowParallel(): String = coroutineScope {
    // Start child and activity in parallel using standard Kotlin async
    val childDeferred = async {
        KWorkflow.executeChildWorkflow(
            ChildWorkflow::doWork,
            "input",
            KChildWorkflowOptions(workflowId = "child-workflow-id")
        )
    }
    val activityDeferred = async {
        KWorkflow.executeActivity(
            SomeActivities::doSomething,
            KActivityOptions(startToCloseTimeout = 30.seconds)
        )
    }

    // Wait for both using standard awaitAll
    val (childResult, activityResult) = awaitAll(childDeferred, activityDeferred)
    "$childResult - $activityResult"
}
```

## Child Workflow Handles

For cases where you need to interact with a child workflow (signal, query, cancel) rather than just wait for its result, use `startChildWorkflow` to get a handle:

```kotlin
// Start child workflow and get handle for interaction
override suspend fun parentWorkflowWithHandle(): String {
    val handle = KWorkflow.startChildWorkflow(
        ChildWorkflow::doWork,
        "input",
        KChildWorkflowOptions(workflowId = "child-workflow-id")
    )

    // Can signal the child workflow
    handle.signal(ChildWorkflow::updateProgress, 50)

    // Wait for result when ready
    return handle.result()
}

// Parallel child workflows with handles for interaction
override suspend fun parallelChildrenWithHandles(): List<String> = coroutineScope {
    val handles = listOf("child-1", "child-2", "child-3").map { id ->
        KWorkflow.startChildWorkflow(
            ChildWorkflow::doWork,
            "input",
            KChildWorkflowOptions(workflowId = id)
        )
    }

    // Can interact with any child while they're running
    handles.forEach { handle ->
        handle.signal(ChildWorkflow::updatePriority, Priority.HIGH)
    }

    // Wait for all results
    handles.map { async { it.result() } }.awaitAll()
}
```

## KWorkflow Child Workflow Methods

```kotlin
object KWorkflow {
    /**
     * Execute a child workflow and wait for its result.
     * For fire-and-wait cases where you don't need to interact with the child.
     */

    // 0 arguments
    suspend fun <T, R> executeChildWorkflow(
        workflow: KFunction1<T, R>,
        options: KChildWorkflowOptions
    ): R

    // 1 argument - passed directly
    suspend fun <T, A1, R> executeChildWorkflow(
        workflow: KFunction2<T, A1, R>,
        arg: A1,
        options: KChildWorkflowOptions
    ): R

    // 2+ arguments - use kargs() wrapper for type safety
    suspend fun <T, A1, A2, R> executeChildWorkflow(
        workflow: KFunction3<T, A1, A2, R>,
        args: KArgs2<A1, A2>,
        options: KChildWorkflowOptions
    ): R

    // ... up to 6 arguments with KArgs

    /**
     * Start a child workflow and return a handle for interaction.
     * Use this when you need to signal, query, or cancel the child workflow.
     * For simple fire-and-wait cases, prefer executeChildWorkflow() instead.
     */

    // 0 arguments
    suspend fun <T, R> startChildWorkflow(
        workflow: KFunction1<T, R>,
        options: KChildWorkflowOptions
    ): KChildWorkflowHandle<T, R>

    // 1 argument - passed directly
    suspend fun <T, A1, R> startChildWorkflow(
        workflow: KFunction2<T, A1, R>,
        arg: A1,
        options: KChildWorkflowOptions
    ): KChildWorkflowHandle<T, R>

    // 2+ arguments - use kargs() wrapper for type safety
    suspend fun <T, A1, A2, R> startChildWorkflow(
        workflow: KFunction3<T, A1, A2, R>,
        args: KArgs2<A1, A2>,
        options: KChildWorkflowOptions
    ): KChildWorkflowHandle<T, R>

    // ... up to 6 arguments with KArgs
}
```

## KChildWorkflowHandle API

```kotlin
/**
 * Handle for interacting with a started child workflow.
 * Returned by startChildWorkflow() with result type captured from method reference.
 *
 * @param T The child workflow interface type
 * @param R The result type of the child workflow method
 */
interface KChildWorkflowHandle<T, R> {
    /** The child workflow's workflow ID (available immediately) */
    val workflowId: String

    /**
     * Get the child workflow's run ID.
     * Suspends until the child workflow starts.
     */
    suspend fun runId(): String

    /**
     * Wait for the child workflow to complete and return its result.
     * Suspends until the child workflow finishes.
     */
    suspend fun result(): R

    // Signals - type-safe method references
    suspend fun signal(method: KFunction1<T, *>)
    suspend fun <A1> signal(method: KFunction2<T, A1, *>, arg: A1)
    suspend fun <A1, A2> signal(method: KFunction3<T, A1, A2, *>, args: KArgs2<A1, A2>)
    // ... up to 6 arguments with KArgs

    /**
     * Request cancellation of the child workflow.
     * The child workflow will receive a CancellationException at its next suspension point.
     */
    suspend fun cancel()
}
```

## KChildWorkflowOptions

```kotlin
// All fields are optional - null values inherit from parent workflow or use Java SDK defaults
KChildWorkflowOptions(
    workflowId = "child-workflow-id",
    taskQueue = "child-queue",                      // Optional: defaults to parent's task queue
    workflowExecutionTimeout = 1.hours,
    workflowRunTimeout = 30.minutes,
    retryOptions = KRetryOptions(
        initialInterval = 1.seconds,
        maximumAttempts = 3
    ),
    parentClosePolicy = ParentClosePolicy.PARENT_CLOSE_POLICY_TERMINATE,  // Optional
    cancellationType = ChildWorkflowCancellationType.WAIT_CANCELLATION_COMPLETED  // Optional
)

// Minimal options - just use defaults
val result = KWorkflow.executeChildWorkflow(
    ChildWorkflow::processData,
    inputData,
    KChildWorkflowOptions()
)
```

> **Note:** For simple parallel execution where you only need the result, use standard `coroutineScope { async { executeChildWorkflow(...) } }`. Use `startChildWorkflow` only when you need to interact with the child workflow while it's running.

## Related

- [External Workflows](./external-workflows.md) - Signal/cancel workflows in other executions
- [Cancellation](./cancellation.md) - How cancellation propagates to child workflows
- [Workflow Definition](./definition.md) - Basic workflow patterns

---

**Next:** [External Workflows](./external-workflows.md)
