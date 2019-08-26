# Event loop

Javascript 基于 Event Loop 的运行方式与众不同，microtask queue、task queue 让很多人迷惑代码为什么这样运行。这里就来看一眼 Event Loop。

## Event loop 的执行过程

### ECMA-262

作为 Javascript 背后的标准，ECMA-262 里没有定义 Event Lopp ，没有 microtask queue 与 task queue 。（ECMA-262 同样也没有定义 `setTimeout` 等定时函数，也没有 I/O。）

它定义了 Job 与 Job queue ，并有一个顶层的 [RunJobs] 操作：

> 1. Perform ? InitializeHostDefinedRealm().
> 2. In an implementation-dependent manner, obtain the ECMAScript source texts (see clause 10) and any associated host-defined values for zero or more ECMAScript scripts and/or ECMAScript modules. For each such sourceText and hostDefined, do
>    1. If sourceText is the source code of a script, then
>       i. Perform EnqueueJob("ScriptJobs", ScriptEvaluationJob, « sourceText, hostDefined »).
>    2. Else sourceText is the source code of a module,
>       ii. Perform EnqueueJob("ScriptJobs", TopLevelModuleEvaluationJob, « sourceText, hostDefined »).
> 3. Repeat,
>    1. Suspend the running execution context and remove it from the execution context stack.
>    2. Assert: The execution context stack is now empty.
>    3. Let nextQueue be a non-empty Job Queue chosen in an implementation-defined manner. If all Job Queues are empty, the result is implementation-defined.
>    4. Let nextPending be the PendingJob record at the front of nextQueue. Remove that record from nextQueue.
>    5. Let newContext be a new execution context.
>    6. Set newContext's Function to null.
>    7. Set newContext's Realm to nextPending.[[Realm]].
>    8. Set newContext's ScriptOrModule to nextPending.[[ScriptOrModule]].
>    9. Push newContext onto the execution context stack; newContext is now the running execution context.
>    10. Perform any implementation or host environment defined job initialization using nextPending.
>    11. Let result be the result of performing the abstract operation named by nextPending.[[Job]] using the elements of nextPending.[[Arguments]] as its arguments.
>    12. If result is an abrupt completion, perform HostReportErrors(« result.[[Value]] »).

这里，首先根据脚本的类型，将 [ScriptEvaluationJob](https://www.ecma-international.org/ecma-262/#sec-scriptevaluationjob) 或 [TopLevelModuleEvaluationJob](https://www.ecma-international.org/ecma-262/#sec-toplevelmoduleevaluationjob) 加入 `ScriptJob` 队列。然后开始循环处理每一个非空 Job queue 。

在同一个 Job queue 内部，任务是严格先进先出的。但是，选择哪一个是实现决定的。然而，ECMA-262 仅定义了两个 Job queue ，一个是上面的顶层的 `ScriptJob` ，只用于放（唯一一个）顶层任务；另一个是 `PromiseJob` ，用于 Promise 。这两个 Job queue 不会同时非空，因而执行顺序其实是确定的。

[EnqueuJob](https://www.ecma-international.org/ecma-262/#sec-enqueuejob)被用于想 Job queue 加入 Job。

### HTML

在 HTML 中使用的 Javascript，并没有使用上面提到的 RunJobs ，以及 Job queue ，而是定义了自己的 [Event Loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop) ，以及 [task queue](https://html.spec.whatwg.org/multipage/webappapis.html#task-queue)，[microtask queue](https://html.spec.whatwg.org/multipage/webappapis.html#microtask-queue)。

HTML 的 Event Loop 执行如下的[操作](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)：

> 1. Let taskQueue be one of the event loop's task queues, chosen in a user-agent-defined manner, with the constraint that the chosen task queue must contain at least one runnable task. If there is no such task queue, then jump to the microtasks step below.
> 2. Let oldestTask be the first runnable task in taskQueue, and remove it from taskQueue.
> 3. Report the duration of time during which the user agent does not execute this  loop by performing the following steps:
>    1. Set event loop begin to the current high resolution time.
>    2. If event loop end is set, then let top-level browsing contexts be the set of all top-level browsing contexts of all Document objects associated with the event loop. Report long tasks, passing in event loop end, event loop begin, and top-level browsing contexts.
> 4. Set the event loop's currently running task to oldestTask.
> 5. Perform oldestTask's steps.
> 6. Set the event loop's currently running task back to null.
> 7. Remove oldestTask from its task queue.
> 8. Microtasks: Perform a microtask checkpoint.
> 9. Let now be the current high resolution time. [HRT]
> 10. Report the task's duration by performing the following steps:
>     1. ...
> 11. 以下省略

注意，microtask queue 不是 task queue。task queue 不是队列，因为它不是先进先出的。从第一步可以看出从 task queue 中取出的任务并不一定是最先进入 task queue 的任务。

在 Event Loop 中，完成一个 task queue 的任务之后，会[Perform a microtask checkpoint](https://html.spec.whatwg.org/multipage/webappapis.html#perform-a-microtask-checkpoint) :

> 1. If the event loop's performing a microtask checkpoint is true, then return.
> 2. Set the event loop's performing a microtask checkpoint to true.
> 3. While the event loop's microtask queue is not empty:
>    1. Let oldestMicrotask be the result of dequeuing from the event loop's microtask queue.
>    2. Set the event loop's currently running task to oldestMicrotask.
>    3. Run oldestMicrotask.
>    4. Set the event loop's currently running task back to null.
> 4. For each environment settings object whose responsible event loop is this event loop, notify about rejected promises on that environment settings object.
> 5. Cleanup Indexed Database transactions.
> 6. Set the event loop's performing a microtask checkpoint to false.

在这里，将会按照先进先出的顺序，执行所有 microtask queue 中的任务（包括在此过程中新进入 microtask queue 的任务）， 直到 microtask queue 为空。

所以执行过程是，执行一个 task queue 的任务，然后执行 microtask queue 中的所有任务，然后进入 Event Loop 下一个循环，再执行一个 task queue 中的任务。

ECMA-262 中由 EnqueuJob 加入的 Job ，[在 HMTL 中全部进入 microtask queue](https://html.spec.whatwg.org/multipage/webappapis.html#enqueuejob(queuename,-job,-arguments)).

如果仅有由 Promise 生成的 microtask 的话，上述的执行过程与 RunJobs 基本是一致的。所以以后的讨论，都基于 HTML 的 Event Loop 与 microtask / task 的定义进行。

## Task vs Microtask

那么，microtask 跟 task 都各有那些呢？ 这里挑一部分来介绍，应该可以解决很多网上的“为什么输出会是这样”的疑问。

### Microtask

1. 所有 ECMA-262 中的 Promise 相关的 Job。包括：
   1. `then`、`catch` 的回调
      * 如果 Promise 在调用 `then` 或 `catch` 时已经 settle（状态已确定），那么相应的回调函数直接加入 microtask queu，参见 [PerformPromiseThen](https://www.ecma-international.org/ecma-262/#sec-performpromisethen) 。否则，回到被记录，并在 Promise settle 时，通过[TriggerPromiseActions](https://www.ecma-international.org/ecma-262/#sec-triggerpromisereactions) 加入 micro task queue。
      * `await` 是由 Promise 实现的，每一个 `await` ，都会通过 Promise.then 执行 await 结束之后的操作（即使 `await` 的对象不是 Promise）。（参见[Await](https://www.ecma-international.org/ecma-262/#await)）
   2. 由一个 Promise (P1) 去 resolve 另一个 Promise （P2）的时候，自动生成的一个对 `P2.then` 的调用。该调用会被加入 microtask queue 。
      * 参见 [Promise Resolve Functions](https://www.ecma-international.org/ecma-262/#sec-promise-resolve-functions)
      * `P2.then` 执行后，会使 `P2.then` 的回调成为另一个 microtask
      * 该 `P2.then` 的回调，将 resolve 或 reject P1
2. [Mutation Observers](https://dom.spec.whatwg.org/#mutation-observers)

### Task

1. `SetTimeout`、`SetInterval` 的回调。即使时间设置为 `0`。
   * 参见 [timer initialisation steps](https://html.spec.whatwg.org/#timer-initialisation-steps)
   * 回调会在延时完成后被加入 task queue 。
