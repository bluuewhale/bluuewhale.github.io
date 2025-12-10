+++
title = 'Kotlin Coroutine Internals: Suspension, Continuation, CPS'
date = '2023-07-07T00:00:00+09:00'
draft = false
slug = 'kotlin-coroutine-internals'
description = 'How Kotlin coroutines build suspension points with state machines and CPS.'
tags = ['Kotlin', 'Coroutines', 'Concurrency', 'CPS', 'State Machine']
categories = ['Kotlin', 'Concurrency']
+++

This post explains how coroutines work, referencing the design proposal [Kotlin Proposals - Coroutines](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md).

## Coroutine

The proposal describes a coroutine in one sentence as `an instance of suspendable computation`. The essential trait of a coroutine is its ability to suspend. So what exactly does “suspendable” mean?

## Suspension

According to the proposal, suspendable means a coroutine can pause execution on the current thread, yield the thread so another coroutine can run, and later resume—possibly on a different thread.

## State Machine

Think of a coroutine as one big function. If that function can be suspended mid-execution, you must call the same function repeatedly to eventually get its return value. That’s not the control flow we’re used to. Let’s look at a simple example of what a suspendable function might look like.

```kotlin
class MyCoroutine {
    private var label = 0
    private var result: Result<Any?> = Result.success(null)
    
    operator fun invoke(): Result<Any?> {
    	result = when (label) {
            0 -> {
                label = 1
                Result.success(sayHello())
            } 
            1 -> {
                label = 2
                Result.success(saySomething())
            }
            2 -> {
                label = 3
                Result.success(sayBye())
            }
            3 -> {
                label = -1
                Result.success("Done!")
            }
            else -> Result.failure(Error("Too many invokation"))
        }
        
        return result
    }
    
    private fun sayHello() {
        println("Hello, World!")
    }
    
    private fun saySomething() {
    	println("I can suspend!")
    }
    
    private fun sayBye() {
        println("Bye!")
    }
}

fun main() {
    val myCoroutine = MyCoroutine()
    myCoroutine() // Hello, World!
    myCoroutine() // I can suspend!
    myCoroutine() // Bye!
    
    val result = myCoroutine()
    println(result.getOrNull()) // Done!
}
```

In this example, `MyCoroutine` is a suspendable function. To make a function suspendable, you need to manage execution-related variables (e.g., a program counter) separately. `MyCoroutine` uses a `label` variable to control its flow when suspending.

Explicitly steering execution based on state is the state machine approach. Kotlin coroutines also adopt this state machine model. The example above is a simplified analogy.

## Suspension point

In the previous example, execution paused three times. The points where execution pauses are called suspension points. They’re crucial for achieving concurrency—the core goal of coroutines. A coroutine stops at a suspension point, and the thread can run a new or waiting coroutine.

For example, if I/O would leave the CPU idle, you can place a suspension point there and schedule another coroutine instead, using CPU resources efficiently.

This looks similar to an OS context switch. However, unlike preemptive multitasking (where time-slicing or similar forces context switches), coroutines use cooperative multitasking: suspension points are defined in user code. In cooperative multitasking, placing suspension points well is key to distributing CPU time effectively. So how do we create a suspension point?

Let’s tweak the earlier example so only `sayHello()` becomes a suspension point. When you run it, the first call to `myCoroutine` prints only “Hello World,” and the subsequent call prints the remaining lines.

```kotlin
val COROUTINE_SUSPEND = "FOO" // Special Symbol

class MySuspendingFunction() {
    private var label = 0

    operator fun invoke(): Result<Any?>? {
        while (true) {
            when (label) {
                0 -> {
                    label = 1
                    val result = sayHello()
                    if (result == COROUTINE_SUSPEND) return null
                }

                1 -> {
                    label = 2
                    val result = saySomething()
                    if (result == COROUTINE_SUSPEND) return null
                }

                2 -> {
                    label = 3
                    val result = sayBye()
                    if (result == COROUTINE_SUSPEND) return null
                }

                3 -> {
                    label = -1
                    return Result.success("Done!")
                }

                else -> return Result.failure(Error("Too many invokation"))
            }
        }
        
        label++
    }

    private fun sayHello(): Any {
        println("Hello, World!")
        return COROUTINE_SUSPEND
    }

    private fun saySomething(): Any {
        println("I can suspend!")
        return Unit
    }

    private fun sayBye(): Any {
        println("Bye!")
        return Unit
    }
}

fun main() {
    val fn = MySuspendingFunction()
    
    fn() // Hello, World!
    println("================================")
    val result = fn() // I can suspend! + Bye!
    println(result?.getOrNull()) // Done!
}

// Hello, World!
// ================================
// I can suspend!
// Bye!
// Done!
```

In real Kotlin, the compiler generates bytecode that turns a suspend function into a state machine. It also uses the special enum value `COROUTINE_SUSPEND` to create suspension points.

## Suspending function

A suspending function is a special function that can potentially become a suspension point. But not every suspending function actually contains one.

For example, the `sayHello()` function below is not a suspension point. In contrast, `sayHelloAfter()` is—specifically, the `delay` call inside it is the suspension point.

```kotlin
import kotlinx.coroutines.delay

// this is not a suspension point
suspend fun sayHello() {
  println("Hello!")
}

// this is a suspension point
suspend fun sayHelloAfter(ms: Long) {
  delay(ms)
  println("Hello")
}
 
```

To understand how Kotlin forms suspension points, we need to look at `Continuation`.

## Continuation

Continuation owns the code block and execution context of a coroutine and plays a central role in running and suspending it. In that sense, you can think of a coroutine as a chain of one or more Continuations.

Coroutine builders such as `launch` create a `Continuation` internally. Kotlin’s `startCoroutine()` also ends up creating a `Continuation`.

```kotlin
// https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/coroutines/Continuation.kt#L112
public fun <T> (suspend () -> T).startCoroutine(completion: Continuation<T>) {
    createCoroutineUnintercepted(completion).intercepted().resume(Unit)
}
```

So what does a Continuation look like? It’s an interface that provides `resumeWith()` to resume execution of the code block.

p.s. `CoroutineContext` holds the environment of the Continuation itself; it differs from the execution context of the code block described earlier. Concrete implementations store the actual execution context as internal state. We’ll cover this in more detail later.

```kotlin
// https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/coroutines/Continuation.kt
public interface Continuation<in T> {
    /**
     * The context of the coroutine that corresponds to this continuation.
     */
    public val context: CoroutineContext

    /**
     * Resumes the execution of the corresponding coroutine passing a successful or failed [result] as the
     * return value of the last suspension point.
     */
    public fun resumeWith(result: Result<T>)
}
```

Why doesn’t the Continuation interface provide a `suspend()`-like method? And why does it offer `resumeWith()`?

First question: whether to create a suspension point is decided inside the suspend function. As explained earlier, the Kotlin compiler transforms suspend functions into state machines and decides on suspension based on their return values. There’s no need to inject a Continuation and call a `suspend` method explicitly. If you want a suspension point, just return `COROUTINE_SUSPENDED` from the suspend function.

Then what is `resumeWith()` for? It’s a callback for external operations that cause CPU idle time (like I/O) to resume a coroutine after completion.

`delay`, one of the most representative suspend functions in the Kotlin standard library, is implemented roughly like this:

```kotlin
suspend fun delay(ms: Long) = suspendCoroutine { continuation ->
    Timer().schedule(object : TimerTask() {
        override fun run() {
            continuation.resume(Unit)
        }
    }, ms)
}
```

`suspendCoroutine` is provided by the Kotlin standard library. It injects a Continuation into a suspend function and lets you control it via a callback. If `suspendCoroutine` doesn’t explicitly call `Continuation.resumeWith()` inside, that suspend function returns `COROUTINE_SUSPENDED`. In the example above, `Continuation.resumeWith()` is scheduled on a separate thread by a Timer. Thus `delay()` internally returns `COROUTINE_SUSPENDED` and acts as a suspension point.

By extension, you can implement an async file read as a suspend function:

```kotlin
import kotlin.coroutines.*
import java.io.File
import java.nio.ByteBuffer
import java.nio.channels.AsynchronousFileChannel
import java.nio.channels.CompletionHandler
import java.nio.file.StandardOpenOption

suspend fun readAsync(file: File, buffer: ByteBuffer): Int = suspendCoroutine { continuation ->
    val channel = AsynchronousFileChannel.open(file.toPath(), StandardOpenOption.READ)

    channel.read(buffer, 0L, continuation, object : CompletionHandler<Int, Continuation<Int>> {
        override fun completed(result: Int, attachment: Continuation<Int>) {
            continuation.resume(result)
            channel.close()
        }

        override fun failed(exc: Throwable, attachment: Continuation<Int>) {
            continuation.resumeWithException(exc)
            channel.close()
        }
    })
}
```

## Continuation Passing Style

In Kotlin, a suspend function creates a suspension point by returning `COROUTINE_SUSPENDED`, and resumes the coroutine by calling `Continuation.resumeWith()` (often on another thread) to achieve concurrency.

For a suspend function to control a Continuation, it needs a reference to one. During compilation, Kotlin adds a `Continuation` parameter to every suspend function. This is the CPS (Continuation Passing Style) transformation.

```kotlin
suspend fun <T> sayHello(): T
```

After CPS transformation, the function looks like this:

```kotlin
fun <T> sayHello(continuation: Continuation<T>): Any?
```

The return type becomes `Any?` because every suspend function can return the special value `COROUTINE_SUSPENDED`. Kotlin adopts CPS so that suspend functions can call `Continuation.resumeWith()`.

## Implementation

Putting it all together, we can implement a coroutine like this. In reality, coroutines include many more features—`Dispatcher` to choose the thread for resumption, cancellation, structured concurrency, and more. For clarity, this example focuses on the key pieces: the state machine and CPS.

```kotlin
import java.util.*

const val COROUTINE_SUSPENDED = "foo"

// continuations
interface Continuation<T> {
    fun resumeWith(result: T): Any?
}

class SayHello(private val continuation: Continuation<Any?>): Continuation<Any?> {
    override fun resumeWith(result: Any?): Any? {
        println("Hello, World!")
        return delay(3000, continuation)
    }
}

class SaySomething(private val continuation: Continuation<Any?>): Continuation<Any?> {
    override fun resumeWith(result: Any?): Any? {
        println("I can suspend!")
        return continuation.resumeWith(null)
    }
}

class SayBye(private val continuation: Continuation<Any?>): Continuation<Any?> {
    override fun resumeWith(result: Any?): Any? {
        println("Bye!")
        return continuation.resumeWith(null)
    }
}

// suspending functions (CPS applied)
class MySuspendingFunction(private val continuation: Continuation<Any?>): Continuation<Any?> {
    var label = 0
    override fun resumeWith(result: Any?): Any? {
        return when (label) {
            0 -> {
                label = 1
                SayHello(this).resumeWith(null)
            }

            1 -> {
                label = 2
                SaySomething(this).resumeWith(null)
            }

            2 -> {
                label = 3
                SayBye(this).resumeWith(null)
            }

            3 -> {
                label = -1
                continuation.resumeWith(null)
            }

            else -> Error("Too many invokation")
        }
    }
}

// util methods
fun delay(ms: Long, continuation: Continuation<Any?>): Any? {
    Timer().schedule(object : TimerTask() {
        override fun run() {
            continuation.resumeWith(Result.success(Unit))
        }
    }, ms)

    return COROUTINE_SUSPENDED
}

class EmptyContinuation: Continuation<Any?>{
    override fun resumeWith(result: Any?){
        println("We are done now")
    }
}


fun main() {
    MySuspendingFunction(EmptyContinuation()).resumeWith(null)
    println("I'm not blocking!")
    println("==========================")

    Thread.sleep(4000) // prevent program being terminated
}

// Hello, World!
// I'm not blocking!
// ===========  (after 3 seconds..) =============
// I can suspend!
// Bye!
// We are done now
```