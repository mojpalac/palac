# Kotlin Coroutines Deep Dive

## Part 1: Understanding KotlinCoroutines

### Why Kotlin Coroutines?

Threads issues:

- There is no mechanism here to cancel these threads, so we
  often face memory leaks.
- Making so many threads is costly.
- Frequently switching threads is confusing and hard to manage.
- The code will unnecessarily get bigger and more complicated.

Callbacks:

- Getting data parallelized, is not so straightforward with callbacks
- supporting cancellation requires a lot of additional effort.
- The increasing number of indentations make this code hard to
  read (code with multiple callbacks is often considered highly
  unreadable). Such a situation is called “callback hell”.
- When we use callbacks, it is hard to control when things are triggered.

Rxjava/ReactiveStreams

- You need to learn different functions, like subscribeOn, observeOn, map, or subscribe,
- Cancelling needs to be explicit.
- Functions need to return objects wrapped inside Observable or Single classes.
- In practice, when we introduce RxJava, we need to reorganize our code a lot.

## Using Kotlin coroutines

core functionality is the ability to **suspend** a coroutine at some point and resume it in the future.
Suspended coroutine released the thread, so thread is not blocked!

`suspendCoroutine<Unit> {continuation ->  continuation.resume(Unit}`

suspendCoroutine needs to be called with resume to progress, otherwise it will always be suspended.

**Suspending a coroutine, not a function**

we suspend a coroutine, not a function. Suspending functions are not coroutines, just
functions that can suspend a coroutine.

## Coroutines under the hood

- suspend fun in reality are fun with additionally parameter at the end: continuation
- Suspending functions are like state machines, with a possible state at the beginning of the function and after each
  suspending
  function call.
- Both the number identifying the state and the local data are kept in the continuation object.
- Continuation of a function decorates a continuation of its caller function; as a result, all these continuations
  represent a call stack that is used when we resume or a resumed function completes.

## Part 2: Kotlin Coroutines library

### Coroutine builders

Suspend function can't be called from normal function, it can either be started in another suspended function or from
coroutine builder - bridge from the normal to the suspending world.

most common used:

- launch
- runBlocking
- async

**launch**
The way `launch` works is conceptually similar to starting a new `thread` (thread function)
`launch` is an extension function on the `CoroutineScope` interface. This is part of an important mechanism called
**structured concurrency**, whose purpose is to build a relationship between the parent coroutine and a child coroutine.
To some degree, how `launch` works is similar to a `daemon thread` but much cheaper.

```kotlin
fun main() {
    thread(isDaemon = true) {
        Thread.sleep(1000L)
        println("World!")
    }
    thread(isDaemon = true) {
        Thread.sleep(1000L) println ("World!")
    }
    thread(isDaemon = true) {
        Thread.sleep(1000L)
        println("World!")
    }
    println("Hello,")
    Thread.sleep(2000L)
}
```

```kotlin
fun main() {
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    Thread.sleep(2000L)
}
```

**runBlocking builder** (currently rarely used)
`runBlocking` is a very atypical builder. It blocks the thread it has been started on whenever its `coroutine` is
suspended (similar to suspending main).
This means that `delay(1000L)` inside runBlocking will behave like `Thread.sleep(1000L)`:

```kotlin
fun main() {
    Thread.sleep(1000L)
    println("World!")
    Thread.sleep(1000L)
    println("World!")
    Thread.sleep(1000L)
    println("World!")
    println("Hello,")
}
```

```kotlin
fun main() {
    runBlocking {
        delay(1000L)
        println("World!")
    }
    runBlocking {
        delay(1000L)
        println("World!")
    }
    runBlocking {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
}
```

Use cases in which `runBlocking` is used:

- main function, where we need to block the thread, because otherwise the program will end.
- unit tests, where we need to block the thread for the same reason.

**async builder**
Similar to `launch`, but produce a value.
The `async` function returns an object of type `Deferred<T>`, where `T` is the type of the produced value.
`Deferred` has a suspending method await, which returns this value once it is ready.
Just like the `launch` builder, `async` starts a coroutine immediately when it is called. So, it is a way to start a few
processes at once and then `await` all their results. The returned `Deferred` stores a value inside itself once it is
produced, so once it is ready it will be immediately returned from `await`. However, if we call `await` before the value
is
produced, we are suspended until the value is ready.

```kotlin
fun main() = runBlocking {
    val res1 = GlobalScope.async {
        delay(1000L)
        "Text 1"
    }
    val res2 = GlobalScope.async {
        delay(3000L)
        "Text 2"
    }
    val res3 = GlobalScope.async {
        delay(2000L)
        "Text 3"
    }
    println(res1.await())
    println(res2.await())
    println(res3.await())
}
```

How the `async` builder works is very similar to `launch`, but it has additional support for returning a value. If all
`launch` functions were replaced with `async`, the code would still work fine. But don’t do that! `async` is about
producing a
value, so if we don’t need a value, we should use `launch`.

```kotlin
fun main() = runBlocking {
// Don't do that!
// this is misleading to use async as launch 
    GlobalScope.async {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    delay(2000L)
}
```

The async builder is often used to parallelize two processes, such as obtaining data from two different places, to
combine them together.

```kotlin
scope.launch {
    val news = async {
        newsRepo.getNews()
            .sortedByDescending { it.date }
    }
    val newsSummary = newsRepo.getNewsSummary() // we could wrap it with async as well,
// but it would be redundant
    view.showNews(
        newsSummary,
        news.await()
    )
}
```

**Structured Concurrency**

If a coroutine is started on `GlobalScope`, the program will not wait for it. As previously mentioned, coroutines do not
block any threads, and nothing prevents the program from ending. This is why, in the below example, an additional delay
at the end of `runBlocking` needs to be called if we want to see “World!” printed.

```kotlin
fun main() = runBlocking {
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    GlobalScope.launch {
        delay(2000L)
        println("World!")
    }
    println("Hello,")
    //    delay(3000L)
}

// Hello,
```

Why do we need this `GlobalScope` in the first place?
It is because launch and async are extension functions on `CoroutineScope`.
However, if you take a look at the definitions of these and of `runBlocking`, you will see that the
block parameter is a function type whose receiver type is also `CoroutineScope`.
This means we don't need `GlobalScope`, instead, `launch` can be called on the receiver provided by
`runBlocking`.

```kotlin
fun main() = runBlocking {
    this.launch { // same as just launch
        delay(1000L)
        println("World!")
    }
    launch { // same as this.launch
        delay(2000L)
        println("World!")
    }
    println("Hello,")
}
```

A parent provides a scope for its children, and they are called in this scope.
This builds a relationship that is called a structured concurrency.
Here are the most important effects of the parent-child relationship:
• children inherit context from their parent (but they can also overwrite)
• a parent suspends until all the children are finished
• when the parent is cancelled,its child coroutines are cancelled too
• when a child raises an error,it destroys the parent as well

Notice that, unlike other coroutine builders, runBlocking is not an extension function on CoroutineScope. This means
that it cannot be a child: it can only be used as a root coroutine (the parent of all the children in a hierarchy).
This means that runBlocking will be used in different cases than other coroutines. As we mentioned before, this is very
different from other builders.