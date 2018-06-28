# Encapsulate successful or failed function execution 

* **Type**: Standard Library API proposal
* **Author**: Roman Elizarov
* **Contributors**: Andrey Breslav, Ilya Gorbunov
* **Status**: Submitted
* **Prototype**: In progress

## Summary

Kotlin language provides exceptions that are used to represent an arbitrary failure of a function and include 
ability to attach additional information pertaining to this failure. Exceptions are sequential in nature and work 
great in any kind of sequential code, including code for a single coroutine or in other case where one piece 
of work in being sequentially decomposed. Exceptions ensure that the first failure in a sequentially performed work
stops further progress and is propagated up to the caller. However, sequential nature of exceptions
complicates their use in cases where some kind of parallel decomposition of work is needed or multiple
failures need to be retained for later processing. 

We'd like to introduce a type in the Kotlin standard library that is effectively a union type between successful
and failed outcome of execution of Kotlin function. If union types were natively supported by Kotlin 
language we would just represent this union as `T | Throwable`, where `T` represents a successful result of some 
type `T` and `Throwable` represents a failure. But since there is no direct support for union types in the Kotlin
language we model it as a separate generic class `SuccessOrFailure<T>` in the standard library.  

## Use cases

This section lists motivating use-cases.

### Continuation and similar callbacks

The primary driver for inclusion of this class into the Standard Library is `Continuation<T>` callback interface
that should get invoked on the successful or failed execution of an asynchronous operation.
We'd like to be able to have only a single function with "success or failure" union type as its parameter:

```kotlin
interface Continuation<in T> {
    fun resumeWith(result: SuccessOrFailure<T>)
}
```  

### Asynchronous parallel decomposition

Another example here is parallel execution of multiple asynchronous operations that must capture 
successful or failed execution of each individual piece to analyze and reach decision on the outcome of a 
larger piece of work:

```kotlin
val deferreds: List<Deferred<T>> = List(n) { 
    async { 
        /* Do something that produces T or fails */ 
    } 
}
val outcomes1: List<T> = deferreds.map { it.await() } // BAD -- crash on the first (by index) failure
val outcomes2: List<T> = deferreds.awaitAll() // BAD -- crash on the earliest (by time) failure 
val outcomes3: List<SuccessOrFailure<T>> = deferreds.map { it.awaitCatching() } // !!! <= THIS IS THE ONE WE WANT  
```     

Where `awaitCatching` could be defined like this:

```kotlin
suspend fun <T> Deferred<T>.awaitCatching(): SuccessOrFailure<T> = 
    runCatching { await() }
```

### Functional bulk manipulation of failures

Kotlin encourages writing code in a functional style. It works well as long as business-specific failures are
represented with nullable types or sealed class hierarchies, while other kinds of failures 
(that are represented by exceptions) do not require any special local handling. However, when interfacing with
Java-style APIs that rely heavily on exceptions or otherwise having a need to somehow process exceptions locally
(as opposed to propagating them up the call stack), we see a clear lack of primitives in the Kotlin standard library.

Consider writing a function `readFiles` that receives a list of files, reads all of them, and returns a 
list of results. We are given the following function to read single file contents:

```kotlin
fun readFileData(file: File): Data
```

This reading function throws exception if file is not found or parsing of a file had somehow failed. Normally that would
be fine and the first failure of this kind would terminate the whole program with a stacktrace and explanatory message. 
However, for `readFiles` we'd explicitly like to be able to continue after the failure to collect and report all failures.
Moreover, we'd like to be able to have a functional implementation of `readFiles` like this:

```kotlin
fun readFilesCatching(files: List<File>): List<SuccessOrFailure<Data>> =
    files.map { 
        runCatching { 
            readFileData(it)
        }
    }
```

> This function is named `readFileCatching` to make it explicit to the caller that all encountered failures
were _caught_ and encapsulated in `SuccessOrFailure` and it is caller responsibility to process those failures.

Now, consider making some transformation of `readFilesCatching` results that we'd like to express functionally, 
while preserving accumulated failures:

```kotlin                                                     
readFilesCatching(files).map { result: SuccessOrFailure<Data> -> // type explicitly written here for clarity
    result.map { it.doSomething() } // Operates on Success case, while preserving Failure
}
```

If `doSomething`, in turn, can potentially fail and we are interested in keeping this failure per each individual
file we can write it using `mapCatching` instead of `map`:

```kotlin
readFilesCatching(files).map { result: SuccessOrFailure<Data> -> 
    result.mapCatching { it.doSomething() }
}
```

### Functional error handling

In mostly functional code `try { ... } catch(e: Throwable) { ... }` construct looks
out of style. For example, consider this piece of code that uses Rx for asynchronous processing.
It invokes `doSomethingAsync` that returns `Single<Data>` and processes potential error in a functional style:

```kotlin
doSomethingAsync()
    .doOnError { showErrorDialog(it) }
    .doOnSuccess { processData(it) }
```

Now, if `doSomethingSync` is a synchronous function, then handling its errors looks quite visually different,
which is problematic for the code that mixes both approaches:

```kotlin
try { 
    val data = doSomethingSync()
    processData(data) 
} catch(e: Throwable) { 
    showErrorDialog(e) 
}
```  

Also note, that the code with `try/catch` has different semantics, since it also catches exceptions that could have
been thrown by `processData`. Preserving functional-style error-handling semantics using `try/catch` 
is quite non-trivial (see [Error handling alternative](#error-handling-alternative) section).

Instead, we'd like to be able to write the same code in a more functional way:

```kotlin
runCatching { doSomethingSync() }
    .onFailure { showErrorDialog(it) }
    .onSuccess { processData(it) }
```

## Alternatives

There is a number of community-supported libraries that provide this kind of success or failure union type, 
but we cannot use any of them for the `Continuation` callback interface that is defined in the Standard Library.

### Continuation alternative
 
Alternative signatures for the `Continuation` interface are listed below.

**Two methods** as in current experimental version of coroutines:

```kotlin
interface Continuation<in T> {
    fun resume(value: T)
    fun resumeWithException(exception: Throwable)
}
```  

This solution was tried in experimental version of coroutine and the following problems were identified:

* All implementations have to implement both methods and there is no easy shortcut to provide a builder with
  a lambda like `Continuation { ... body ... }`.
* Some implementations need to capture "success or failure" in their state and pass on captured success or failure
  to another delegate continuation at a later time.
* Some implementations have a common piece of logic that should be executed on both success and failure 
  with minor differences for successful and failed cases. These implementations have to immediately forward both
  `resume` and `resumeWithException` to some internal function like `doResume`, thus increasing stack size and still
  forcing implementor to figure out a way to represent both success and failure in one method.    

**One method with two parameters**:

```kotlin
interface Continuation<in T> {
    fun resume(value: T?, exception: Throwable?)
}
```  

The downside here is that both parameters here are nullable and there is no larger type-safety nor 
a clear indication of intent to have only one of them set.

**One method with Any? parameter**:

```kotlin
interface Continuation<in T> {
    fun resume(result: Any?) // result: T | Throwable
}
```  

This solution completely lacks any type-safety on Kotlin side.

### Error handling alternative

Let's see what it takes to rewrite the code with Rx-style error handling without resorting to 3rd party libraries.

**Non-nullable value type**:

If the result of `doSomethingSync` is non-nullable, then we can write somewhat concise code:

```kotlin
val data: Data? = try { 
        doSomethingSync() 
    } catch(e: Throwable) { 
        showErrorDialog(e)
        null 
    }
if (data != null)    
    processData(data) 
```

**Nullable value type**:

If the result of `doSomethingSync` is nullable, then one possible alternative is shown below: 

```kotlin
var data: Data? = null
val success = try { 
        data = doSomethingSync()
        true 
    } catch(e: Throwable) { 
        showErrorDialog(e)
        false 
    }
if (success)    
    processData(data) 
```

## API Details

The following snippet gives summary of all the public APIs:

```kotlin
class SuccessOrFailure<out T> /* internal constructor */ {
    val isSuccess: Boolean
    val isFailure: Boolean
    fun getOrThrow(): T
    fun getOrNull(): T?
    fun exceptionOrNull(): Throwable?
    
    companion object {
        fun <T> success(value: T): SuccessOrFailure<T>
        fun <T> failure(exception: Throwable): SuccessOrFailure<T>
    }
}

inline fun <R> runCatching(block: () -> R): SuccessOrFailure<R>
inline fun <T, R> T.runCatching(block: T.() -> R): SuccessOrFailure<R>

inline fun <R, T : R> SuccessOrFailure<T>.getOrElse(defaultValue: () -> R): R

inline fun <R, T> SuccessOrFailure<T>.map(transform: (T) -> R): SuccessOrFailure<R>
inline fun <R, T: R> SuccessOrFailure<T>.recover(transform: (Throwable) -> R): SuccessOrFailure<R>

inline fun <R, T> SuccessOrFailure<T>.mapCatching(transform: (T) -> R): SuccessOrFailure<R>
inline fun <R, T: R> SuccessOrFailure<T>.recoverCatching(transform: (Throwable) -> R): SuccessOrFailure<R>

inline fun <T> SuccessOrFailure<T>.onFailure(block: (Throwable) -> Unit): SuccessOrFailure<T>
inline fun <T> SuccessOrFailure<T>.onSuccess(block: (T) -> Unit): SuccessOrFailure<T>
```

All of the functions have self-explanatory consistent names that follow established tradition in Kotlin Standard library
and establish the following additional conventions:

* Functions that can throw previously suppressed (captured) exception are named 
  with explicit `OrThrow` suffix like `getOrThrow`.
* Functions that capture thrown exception and encapsulate it into `SuccessOrFailure` instance are named 
  with explicit `Catching` suffix like `runCatching` and `mapCatching`.
* A traditional `map` transformation function that works on successful cases 
  is augmented with a `recover` function that similarly transforms exceptional cases. 
  A failure inside either `map` or `recover` transform aborts operation like a traditional function, 
  but `mapCatching` and `recoverCatching` encapsulate failure in transform into the resulting `SuccessOrFailure`.
* Functions to query the case are naturally named `isSuccess` and `isFailure`. 
* Functions that act on the success or failure cases are named `onSuccess` and `onFailure` and return their receiver
  unchanged for further chaining according to tradition established by `onEach` extension from the Standard Library.   

## Dependencies

This library depends on 
[`inline class`](https://github.com/zarechenskiy/KEEP/blob/master/proposals/inline-classes.md) 
language feature for its efficient implementation. 

## Binary contract and implementation details

`SuccessOrFailure` is implemented by an `inline class` and is optimized for a successful case. Success is stored as
a value of `SuccessOrFailure` directly, without additional boxing, while failure exception is wrapped into an internal 
`SuccessOrFailure.Failure` class. `SuccessOrFailure` class has the following internal published APIs that 
represent its binary interface on JVM in addition to its public [API](#api-details):

```kotlin
inline class SuccessOrFailure<out T>  @PublishedApi internal constructor(
    @PublishedApi internal val value: Any? // internal value -- either T or Failure
) : Serializable {
    @PublishedApi internal class Failure(
        @JvmField val exception: Throwable
    ) : Serializable
} 
```  

## Error-handling style and exceptions

The `SuccessOrFailure` class is not designed to be used as the result type of general functions and we'd like 
to explicitly discourage using it like this:

```kotlin
fun findUserByName(name: String): SuccessOrFailure<User>
```

In general, if some API requires its invokers to handle invocation failures locally, then it should use nullable types
(if failures do not carry additional business meaning) or domain-specific data types to represent its results and 
failures.  

In the above example, if the only kind of failure we might be interested in handling is the failure to find 
the user with the given name, then the following signature shall be used:

```kotlin
fun findUserByName(name: String): User?
```   
   
If there is a business need to distinguish different failures and process these different failures in distinct ways
on each invocation site, then the following kind of signature shall be considered:

```kotlin
sealed class FindUserResult {
    data class Found(val user: User) : FindUserResult()
    data class NotFound(val name: String) : FindUserResult()
    data class MalformedName(val name: String) : FindUserResult()
    // other cases that need different handling
}

fun findUserByName(name: String): UserResult
```

> This is one of the reasons that `SuccessOrFailure` was not named `Result`. That is, to prevent its abuse
  as a "universal result class" and to encourage declaration of more domain-specific and more explanatory 
  result classes.
 
Exceptions in Kotlin are designed for the failures that usually do not require local handling at each call site.
This includes several broad areas. Logic and programming errors like index bounds problems and various checks
for internal invariants and preconditions, environment problems, out of memory conditions, etc. 
These failures are usually non-recoverable (or are not supposed to be recovered from) and are handled in some 
centralized way by logging or otherwise reporting them for troubleshooting, typically terminating application
or, sometimes, attempting to restart or to reinitialize an application as a whole or just its failing subsystem.       
This is where default exceptions behaviour to abort current operation and propagate it up the call stack comes in handy. 

External environment problems like network or file Input/Output errors represent a corner case here. 
It is cumbersome to require their local handling by the caller as it complicates sequential business 
logic by obscuring it with code to handle IO errors, so it is idiomatic in Kotlin to use exceptions (like `IOException`) 
for these. However, they are often handled at a more granular level than some global error-handling code. 
These errors often require some specific user-interaction and can require domain-specific retry or recovery code.

So, in case when `findUserByName` failure does not require local handling by the caller, then its failure 
should be represented by exception and its signature should look like this:

```kotlin
fun findUserByName(name: String): User
```

If invoker of this function wants to perform multiple concurrent operations and process their failures afterwards,
it can always use `runCatching { findUserByName(name) }` to make it explicit that a failure is being caught and
encapsulated into `SuccessOrFailure` instance. If the need to do so happen often, then it is fine to define
the function named `findUserByNameCatching` that returns `SuccessOrFailure<User>`
in _addition_ to `findUserByName`, following the convention of `Catching` suffix in the name of the function
that encapsulates failures.

## Similar API review

Kotlin Standard Library provides rich collection of transformations for nullable types that are idiomatic in Kotlin
to indicate failure when no additional information about the failure is needed. However, there is no build-in support
for non-standard exception handling in the Standard Library -- exceptions always terminate operation and propagate to the caller.

Other programming languages include a similar facility to represent a union of success and failure in their standard 
library with the following names:

* [`Try[T]`](https://www.scala-lang.org/api/current/scala/util/Try.html) in Scala is similar to the proposed `SuccessOrFailure<T>`.
* [`Result<T, E>`](https://doc.rust-lang.org/std/result/) in Rust (also parametrized by the type of error).
* [`Exceptional e t`](https://wiki.haskell.org/Exception) in Haskell (also parametrized by the type of error).  
* [`expected<E, T>`](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4015.pdf) in C++ (proposed, also parametrized by the type of error).

Existing Kotlin libraries that provide similar functionality:

* [`Try<T>`](https://arrow-kt.io/docs/datatypes/try/) in Arrow library.

For comparison of Scala's `Try` and its Kotlin analogue in Arrow library with this `SuccessOrFailure` class
see [Appendix](#appendix:-why-flatmap-is-missing). 

## Placement

This API shall be placed into the Kotlin Standard Library. Since the proposed API is fairly small and does not
clearly belong to any larger group of APIs, it should be placed directly into `kotlin` package.  

## Unresolved questions

**Name of the class**:

The proposed name is `SuccessOrFailure` was chosen because it clearly express that this class is a union of
success and failed outcome. Some alternative names like `Result` and `Try` ware discussed and the 
following objections were raised:

* A shorter name does not clearly indicate what this wrapper class actually represents. 
  Neither `Result` nor `Try` have the corresponding connotations. 
  `Exception` or `Expected` were not at the table and the time of naming discussion.
* A shorter name would encourage abuse of this class as a return type of a function in cases where simply throwing
  an exception and letting a caller decide works better
  (see [Error-handling style and exceptions](#error-handling-style-and-exceptions)).
  
**Parametrization by the base error type**:

Parametrizing this class by the type of exception like `SuccessOrFailure<T, E>` is possible, but raises the
following problems:
    
* It increases verboseness without providing improvement for any of the named [Use cases](#use-cases).
* Kotlin currently lacks facility to specify default values for generic type parameters.
* It leads to abuse in cases where a user-provided API-specific sealed class would work better.   

It is possible to define a separate class that is parametrized by `<T, E>` and then define
`SuccessOrFailure<T>` and a `typealias` to that class with `E = Throwable`. However, it complicates the naming
and IDE support with respect to typealiases. Moreover, we cannot succinctly define `runCatching` function 
and other `Catching` functions to make them usable both with and without explicit caught type specification.
We'll have to have two different names for a function with an additional `E: Throwable` type parameter
that must be specified and without it. Moreover, specifying `E` on call site requires specifying return type, too,
since partial type parameter specification is not possible in Kotlin.  

All in all, it does not seem that the costs outweigh whatever benefits it might bring.

**Name for the Catching suffix**:

We need to clearly mark functions that return `SuccessOrFailure` and this KEEP suggests `Catching` suffix
like in `runCatching`. Alternatives:

* `OrCatch` suffix as in `runOrCatch` (inspired by `OrNull` suffix).
* `OrFailure` suffix as in `runOrFailure` (same).

**Success or failure must be used**:

Using `SuccessOrFailure` as the return type poses a problem that it might accidentally get lost, thus loosing
unhandled exception in case the invocation was a failure. The problem is somewhat alleviated by naming 
all such function with `Catching` suffix, but finding lost exception in the code that uses `SuccessOrFailure` 
is still non trivial challenge.  

Consider this code from [Functional bulk manipulation of failures](#functional-bulk-manipulation-of-failures):

```kotlin                                                     
readFilesCatching(files).map { result ->
    result.map { it.doSomething() } 
}
```

If `doSomething` throws an exception, then all exceptions that were returned in a list by `readFilesCatching` are lost.

Some IDE inspections can be designed to detect these kinds problems. It is an open question how exactly they should
work and and whether it is really a big problem after all.

## Future advancements

**Representing as a sealed class**:

Kotlin `inline` classes cannot be currently used with `sealed class` construct. 
If that is somehow supported in the future, then we could change implementation of 
`SuccessOrFailure` like this without affecting its public APIs and binary interfaces:

```kotlin
sealed inline class SuccessOrFailure<T> {
    inline class Success<T>(val value: T) : SuccessOrFailure<T>()
    inline class Failure<T>(val exception: Throwable) : SuccessOrFailure<T>()
}
``` 

These would make it possible to use `result is Success` and `result is Failure` checks and get advantage of
smart casts instead of `result.isSuccess` and `result.isFailure` that are currently provided and which do not work 
with smart casts.

**Parametrizing by the base error type**:

If `Kotlin` adds some form of support for type parameter default values and partial type inference,
then we can consider extending `SuccessOrFailure` class with an addition type parameter `E: Throwable` that represents
the base class for caught exceptions. For example, in input/output code there may be a desire to catch only 
`IOException` and its subclasses, while aborting on any other exception using something like
`runCatching<_, IOException> { code }` assuming that return type can be still inferred
(potential partial type inference syntax here is used for illustration only).

**Integration into the language**:

Kotlin nullable types have extensive support in Kotlin via operators `?.`, `?:`, and `!!` and `T?` type constructor
syntax. We can envision a better integration of `SuccessOrFailure` into the Kotlin language in the future.
However, unlike nullable types, that are often used to represent _non signalling_ failure that does not cary
additional information, `SuccessOrFailure` instances also carry additional information and, in general, shall be
always handled in some way. Making `SuccessOrFailure` an integral part of the language also requires a 
considerable effort on improving type system to ensure proper handling of encapsulated exceptions.

## Appendix: Why flatMap is missing

You can skip this appendix is you are not familiar with Scala's or Arrow's `Try` monad that provides very 
similar functionality to this `SuccessOrFailure` class. 

If you familiar with `Try` monad, then you might ask why there is no `flatMap` 
function on the `SuccessOrFailure` class with the following natural signature:

```kotlin
inline fun <R, T> SuccessOrFailure<T>.flatMap(transform: (T) -> SuccessOrFailure<R>): SuccessOrFailure<R>
```  

The usual reason to have `flatMap` is to avoid "nesting" of `SuccessOrFailure` types when combining multiple
functions that return `SuccessOrFailure`:

```kotlin
d.awaitCatching().map { it.doSomethingCatching() } // : SuccessOrFailure<SuccessOrFailure<Data>> -- oops!
``` 
 
Functional code that uses `Try` monad gets quickly polluted
with `flatMap` invocations and in order to make such code manageable a language has to be extended 
with some kind of special monad comprehension syntax to hide those `flatMap` invocations.  

However, we discourage writing functions that return `SuccessOrFailure` as 
a [matter of style](#error-handling-style-and-exceptions) and if those function are written, they should
only be present in addition to regular ones. It means that instead of this kind of code that
uses monad comprehension over `Try` monad 
(which is adapted from a guide 
[here](https://danielwestheide.com/blog/2012/12/26/the-neophytes-guide-to-scala-part-6-error-handling-with-try.html)):
 
```scala
def getURLContent(url: String): Try[Iterator[String]] =
  for {
    url <- parseURL(url)
    connection <- Try(url.openConnection())
    input <- Try(connection.getInputStream)
    source = Source.fromInputStream(input)
  } yield source.getLines()
```

one can write a natural Kotlin code that have the same semantics of aborting further progress on the first failure:

```kotlin
fun getURLContent(url: String): List<String> {
    val url = parseURL(url)
    val connection = url.openConnection()
    val input = connection.getInputStream()
    val source = Source.fromInputStream(input)
    return source.getLines()
}
```

Notice, that monad comprehension over a `Try` monad is basically built into the Kotlin langauge.
That is how imperative control flow works in Kotlin out of the box and there is no need to emulate it
via monad comprehensions.

