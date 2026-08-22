+++
title = 'Kotlin Coroutine의 동작 원리'
date = '2023-06-07T20:38:19+09:00'
draft = false
translationKey = 'kotlin-coroutine-internals'
slug = 'kotlin-coroutine-internals'
description = 'Kotlin coroutine이 state machine과 CPS(Continuation Passing Style)를 통해 suspension point를 만들어내는 원리를 다룹니다.'
tags = ['Kotlin', 'Coroutines', 'Concurrency', 'CPS', 'State Machine']
categories = ['Kotlin', 'Concurrency']
+++

이번 글은 Kotlin Coroutine의 디자인을 제안한 [Kotlin Proposals - Coroutines](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md)를 참고하여 coroutine의 동작 원리에 대해 다루어 보도록 하겠습니다.

## Coroutine

원문에서는 coroutine을 한 문장으로 `an instance of suspendable computation`이라고 설명하고 있습니다. 이처럼, coroutine의 가장 핵심적인 특징은 중단 가능하다는 것입니다. 그렇다면, 중단 가능하다는 것은 정확히 어떤 의미일까요?

## Suspension

원문에 따르면, 중단 가능하다는 것은 coroutine은 다른 coroutine의 코드를 실행할 수 있도록, 현재 thread에서 코드 실행을 잠시 중단하고 thread를 양보할 수 있다는 것을 의미합니다. 뿐만 아니라. 중단된 coroutine이 다른 thread에서 다시 재개될 수 있음을 뜻합니다.

## State Machine

coroutine을 하나의 거대한 함수라고 생각해보겠습니다. coroutine, 즉, 함수가 실행 도중에 중단될 수 있다는 것은 함수의 반환 값을 얻기 위해 동일한 함수를 수 차례에 걸쳐 반복적으로 실행해야 한다는 것을 의미합니다. 이는 일반적으로 우리에게 친숙한 제어 흐름은 아닙니다. 아래의 예시를 통해, 중단 가능한 함수가 대략적으로 어떤 모습을 갖고 있는지 살펴보도록 하겠습니다.

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

위 예시에서 `MyCoroutine` 함수는 중단 가능한 함수입니다. 함수를 중단 가능하도록 만들기 위해서는 함수의 실행과 관련된 변수(ex, program counter)를 별도로 관리할 필요가 있습니다. `MyCoroutine` 은  `label`이란 변수를 활용하여 내부적으로 중단에 따른 실행 흐름을 제어합니다.

이처럼 상태에 따라 명시적으로 실행 흐름을 제어하는 방법론을 state machine이라고 합니다. Kotlin의 coroutine도 state machine 방법론을 채택하여 구현되어 있습니다. 위 예시는 이를 비유적으로 표현하기 위한 간단한 예시입니다.

## Suspension point

앞선 예시에서는 코드 실행 중, 총 세 차례의 중단이 발생하였습니다. coroutine에서는 코드 실행 흐름에서 중단이 발생하는 지점을 suspension point라고 합니다. suspension point는 coroutine의 핵심 목표인 동시성 확보를 위해 매우 중요합니다.  coroutine은 suspension point에서 실행을 중단합니다. 중단된 coroutine이 위치한 thread에서는 새롭게 생성된 혹은 대기중인 coroutine을 실행합니다.

예를 들어, I/O 작업을 위해 CPU 유휴시간이 발생하는 지점을 suspension point로 설정하여, suspend가 발생한 coroutine을 대신하여 새로운 coroutine을 스케쥴링 함으로써, CPU 자원을 효율적으로 사용할 수 있습니다.

이 모습은 OS에서 이뤄지는 context switch와 유사하게 보입니다. 다만, 일반적으로 시분할 알고리즘 등을 활용하여 강제적으로 context switch가 일어나는 preemptive multitasking과 달리, coroutine은 coroutine switch가 일어나는 suspension point가 사용자의 코드에 의해 정의된다는 cooperative multitasking 방식에 해당합니다. Cooperative multitasking에서는 coroutine들에게 CPU가 효율적으로 분배될 수 있도록 적절하게 suspension point를 생성하는 것이 매우 중요합니다. 그렇다면, suspension point는 어떻게 만들어질까요?

앞선 예시를 일부 수정하여, `sayHello()` 함수만 suspension point로 만들어보도록 하겠습니다. 코드를 실행해보면, `myCoroutine` 함수의 최초 호출 시에는 “Hello World” 구문만 출력되고, 재 실행 시에, 나머지 구문들이 출력되는 것을 확인하실 수 있습니다.

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

실제 Kotlin에서는 Kotlin 컴파일러가 suspend function을 state machine 형태로 byte code를 생성합니다.  또한, `COROUTINE_SUSPEND`라는 특수한 enum 값을 사용하여, suspension point를 생성합니다.

## Suspending function

suspending function은 잠재적으로 suspension point가 될 수 있는 가능성을 갖고 있는 특수한 함수입니다. 그러나, 모든 suspending function이 suspension point를 포함하는 것은 아닙니다.

예를 들어, 아래의 `sayHello()` 함수는 suspension point로 동작하지 않습니다. 반면,  `sayHelloAfter()` 함수는 suspension point로서 동작합니다. 엄밀하게는, `SayHelloAfter` 함수의 `delay` 함수가 suspension point에 해당합니다.

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

Kotlin에서 suspension point가 어떻게 만들어지는지 보다 깊이 이해하기 위해서는 `Continuation` 에 대해 알아야 합니다.

## Continuation

Continuation은 coroutine에서 실행되는 code block과 execution context를 실질적으로 소유한 객체로, coroutine의 실행 및 중단과 관련된 핵심적인 역할을 수행합니다. 이런 점에서 coroutine은 연속적으로 이어진 하나 이상의 Continuation의 집합체라고도 할 수 있습니다.

우리가 흔히 coroutine builder로 알고 있는 `launch` 와 같은 함수들도 내부적으로는 `Continuation`을 생성합니다. Kotlin에서 내부적으로 coroutine을 생성하기 위해 사용되는 `startCoroutine()`도 실제로는 `Continuation` 을 생성합니다.

```kotlin
// https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/coroutines/Continuation.kt#L112
public fun <T> (suspend () -> T).startCoroutine(completion: Continuation<T>) {
    createCoroutineUnintercepted(completion).intercepted().resume(Unit)
}
```

그렇다면, Continuation은 과연 어떤 모습일까요? Continuation은 interface입니다. Continuation은 code block 실행을 재개하기 위한 resumeWith() 메서드를 제공합니다.

p.s. `CoroutineContext` 는 Continuation 그 자체의 실행 환경을 담고 있는 객체로, 앞서 설명드린 code block의 execution context와는 차이가 있습니다. 실제 code block의 execution context는 구현체에서 내부 변수로서 저장하여 사용합니다. 이와 관련된 내용은 이후에 조금 더 자세히 다루도록 하겠습니다.

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

Continuation interface에서는 왜 `suspend()` 와 같은 메서드를 제공하지 않을까요? 혹은, 왜 `resumeWith()` 라는 메서드를 제공하는 걸까요?

먼저, 첫 번째 질문에 대한 답은 suspension point 생성 여부는 suspend function에서 결정합니다. 앞서 설명드린 바와 같이, Kotlin 컴파일러는 suspend function을 state machine 형태로 변환하여, suspend function의 반환 값에 따라 suspend 여부를 결정합니다. 따라서, Continuation을 suspend function으로 주입하여 `suspend` 와 같은 함수를 명시적으로 호출하도록 만들 필요가 없습니다. 단지, suspension point를 생성하고 싶은 경우, suspend function에서 `COROUTINE_SUSPENDED` 를 반환하면 그만입니다.

그렇다면 , `resumeWith()` 메서드는 어떤 용도로 사용되는 걸까요? resumeWith는 I/O와 같이 CPU 유휴 시간을 발생시키는 외부 함수가 작업 완료 후, coroutine을 재개할 수 있도록 callback 용도로 제공하는 함수입니다.

Kotlin 표준 라이브러리에서 제공하는 가장 대표적인 suspend function인 `delay` 는 대략 다음과 같은 형태로 구현되어 있습니다.

```kotlin
suspend fun delay(ms: Long) = suspendCoroutine { continuation ->
    Timer().schedule(object : TimerTask() {
        override fun run() {
            continuation.resume(Unit)
        }
    }, ms)
}
```

`suspendCoroutine` 은 Kotlin 표준 라이브러리에서 제공하는 함수로, suspend function에서 Continuation 객체를 주입 받아 이를 직접적으로 제어할 수 있는 callback을 생성하는 함수입니다. 만약 `suspendCoroutine` 함수 내부에서 `Continuation.resumeWith()` 를 명시적으로 호출하지 않는 경우, 해당 suspend function은 `COROUTINE_SUSPENDED` 를 반환하도록 구현되어 있습니다.  위 예시에서는 `Continuation.resumeWith()` 함수가 Timer에 의해 별도의 thread에 스케쥴링 됩니다. 따라서, `delay()` 함수는 내부적으로 `COROUTINE_SUSPENDED` 를 반환하여 suspension point로 동작하게 되는 것입니다.

이를 응용하면, 비동기로 파일을 읽어오는 함수도 suspend function으로 구현할 수 있습니다.

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

Kotlin에서는 suspend function이 `COROUTINE_SUSPENDED` 를 반환하여 suspension point를 생성하고, `Continuation.resumeWith()` 을 호출하여(주로, 외부 thread에 위임) coroutine을 재개시키는 방식으로 concurrency를 구현하였습니다.

suspend function이 Continuation 객체를 제어하기 위해서는 Continuation instance에 대한 reference가 필요합니다. 이를 위해, Kotlin 컴파일러는 변환 과정에서 모든 suspend function에 `Continuation`  을 매개변수로 추가합니다. 이를 CPS(Continuaiton Passing Style) 변환이라고 합니다.

```kotlin
suspend fun <T> sayHello(): T
```

CPS 변환을 거친 후의 코드는 다음과 같습니다.

```kotlin
fun <T> sayHello(continuation: Continuation<T>): Any?
```

함수의 변환 타입이 `Any?` 가 된 이유는, 모든 suspend function은 `COROUTINE_SUSPENDED` 라는 특수한 값을 반환할 수 있도록 설계되었기 때문입니다. Kotlin에서는 CPS 변환을 채택하여 suspend function에서 `Continuation.resumeWith()` 를 호출할 수 있도록 하였습니다.

## Implementation

지금까지의 내용을 모두 활용하여, 간단하게 다음과 같이 coroutine을 구현할 수 있습니다. 실제로는 suspend된 coroutine을 다시 실행할 thread를 지정하는 `Dispatcher` , 취소 기능, structured concurrency 등 훨씬 많은 기능들을 포함하고 있습니다. 이해를 돕기 위해 본 예시는 coroutine의 구현에 핵심적인 역할을 담당하는 `state machine` 과 `CPS` 에 대한 내용을 중점으로 작성하였습니다.

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

## 마무리

이번 글에서는 Kotlin coroutine의 동작 원리를 `state machine` 과 `CPS` 를 중점으로 살펴보았습니다. 다음 글에서는 coroutine의 스케쥴링에 핵심적인 역할을 수행하는 `CoroutineContext` 와 `ContinuationInterceptor` 에 대해 알아보도록 하겠습니다.
