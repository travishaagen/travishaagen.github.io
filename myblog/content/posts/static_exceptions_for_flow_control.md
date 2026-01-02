+++
title = "Static Exceptions for Flow Control"
description = "Static Exceptions, when used for flow control, can dramatically reduce garbage generation and latency in Java and Kotlin applications."
date = 2026-01-02
[taxonomies]
tags = ["Java", "Kotlin", "JVM", "garbage collection", "latency", "performance", "Netty"]
+++

Static Exceptions, when used for flow control, can dramatically reduce garbage generation and latency in Java and Kotlin
applications.

## TL;DR

To create high performance exceptions for flow control in the JVM, simply extend the following `StaticException` class.
Then store the `Exception` instance in a `static final` field and throw it as needed. They're fast, because the call
stack doesn't need to be traversed and since we're creating reusable objects, there's less for the garbage collector
to do.

Java:

```java
public abstract class StaticException extends RuntimeException {

    private static final StackTraceElement[] emptyStackTrace = new StackTraceElement[0];

    public StaticException(String message) {
        super(message, null, false, true);
        setStackTrace(emptyStackTrace);
    }

    @Override
    public synchronized Throwable fillInStackTrace() {
        return this;
    }
}
```

Kotlin:

```kotlin
abstract class StaticException(
    message: String,
) : RuntimeException(message, null, false, true) {

    companion object {
        private val emptyStackTrace = arrayOf<StackTraceElement>()
    }

    init {
        stackTrace = emptyStackTrace
    }

    override fun fillInStackTrace(): Throwable {
        return this
    }
}
```

## Example Use Case

In a web application, conditions that trigger HTTP error responses can happen anywhere in the call chain. For example,
the client's bearer token has expired, so we return `401 Unauthorized`. A request is malformed, so validation
logic returns `400 Bad Request`. Likewise, deeper in the call chain, an asynchronous API call times out, so we return
`503 Service Unavailable`. None of these error conditions are particularly unexpected, and aside from the timeout,
may not even warrant logging.

For JVM applications it can be convenient to throw an exception that signifies "respond with an error". When the
exception is caught, we build the appropriate HTTP response. An example response exception might be,

```java
public class ResponseException extends RuntimeException {

    public final int statusCode;

    public ResponseException(String message, int statusCode) {
        super(message);
        this.statusCode = statusCode;
    }
}
```

Which could be caught in our web controller logic, or a framework mechanism such as a Spring
[@ExceptionHandler](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-exceptionhandler.html),
where the appropriate HTTP response status header and payload body would be constructed,

```json
{
  "code": 400,
  "message": "Bad Request"
}
```

## Discussion

We're using exceptions for _flow control_. Unfortunately, from a performance perspective, exceptions are expensive.
The bulk of the cost comes from `Throwable.fillInStackTrace()` which traverses the
[call stack](https://en.wikipedia.org/wiki/Call_stack), and is exacerbated by **massively** long call stacks created by
libraries and frameworks that we use to make life easier.

Fortunately, there is a solution. We can preinitialize exceptions and either generate the call stack _once_, or
provide an empty stack. I first learned about this technique from Norman Maurer's blog post
entitled “[The hidden performance costs of instantiating Throwables](http://normanmaurer.me/blog/2013/11/09/The-hidden-performance-costs-of-instantiating-Throwables/)”.
Norman is one of the creators of [Netty](https://github.com/netty/netty) which is an abstraction over Java's internal
sockets and byte buffers. It's a technique that has been used sparingly in Netty itself.

However, if you do it wrong, it can lead to problems. Let's look at this constructor for `Throwable`,

```java,hl_lines=4 14-16
protected Throwable(
    String message,
    Throwable cause,
    boolean enableSuppression,
    boolean writableStackTrace
) {
    if (writableStackTrace) {
        fillInStackTrace();
    } else {
        stackTrace = null;
    }
    detailMessage = message;
    this.cause = cause;
    if (!enableSuppression) {
        suppressedExceptions = null;
    }
    if (jfrTracing) {
        ThrowableTracer.traceThrowable(getClass(), message);
    }
}
```

It's most essential that we set `enableSuppression` to `false`. Otherwise, our own code or magical side effects from
libraries could invoke the `addSuppressed` method, and potentially add objects to the `suppressedExceptions`
collection **every time** we throw the reusable exception. The collection would grow infinitely.

```java,hl_lines=7-8 13
public final synchronized void addSuppressed(Throwable exception) {
    if (exception == this)
        throw new IllegalArgumentException(SELF_SUPPRESSION_MESSAGE, exception);

    Objects.requireNonNull(exception, NULL_CAUSE_MESSAGE);

    if (suppressedExceptions == null) // Suppressed exceptions not recorded
        return;

    if (suppressedExceptions == SUPPRESSED_SENTINEL)
        suppressedExceptions = new ArrayList<>(1);

    suppressedExceptions.add(exception);
}
```

This happened to me (`:faceplant:`). It turns out that [Project Reactor](https://projectreactor.io/docs) invokes
`addSuppressed` in a class called
[FluxOnAssembly](https://github.com/travishaagen/reactor-core/blob/main/reactor-core/src/main/java/reactor/core/publisher/FluxOnAssembly.java#L588)
every time an `Exception` passes through.

## Benchmarks

Norman included some [Java Microbenchmark Harness (JMH)](https://github.com/openjdk/jmh) results in the
beforementioned [blog post](http://normanmaurer.me/blog/2013/11/09/The-hidden-performance-costs-of-instantiating-Throwables/).
I also ran across
“[Why Consuming Stack Traces is Noticeably Slower in Java 11 Compared to Java 8: JMH Benchmark Results.](https://www.javaspring.net/blog/consuming-stack-traces-noticeably-slower-in-java-11-than-java-8/)”
It makes some interesting points and I didn't realize that Java 11 introduced a change that was meant to improve
average performance by lazily traversing the call stack for logging. Because of this lazy traversal, I included
some additional `*AndGetStacktrace` JMH benchmarks below.

```text
Benchmark                         Mode  Cnt           Score         Error  Units
staticException                  thrpt    5  1799330562.898 ± 3931377.639  ops/s
staticExceptionAndGetStacktrace  thrpt    5   105525288.622 ±  404717.391  ops/s
newException                     thrpt    5     1529637.197 ±    2700.762  ops/s
newExceptionAndGetStacktrace     thrpt    5      328081.037 ±    1212.503  ops/s
```

To summarize,

1. Throwing and catching a reusable `StaticException` reached 1,799,330,563 operations/second
2. Reusable `StaticException` followed by a call to `e.getStackTrace()` was 105,525,289 operations/second
3. Throwing a new `RuntimeException` every time was 1,529,637 operations/second
4. And `RuntimeException` with `e.getStackTrace()` was 328,081 operations/second

Comparing the _flow control_ `static` use case #1 with `new` #3 shows **1000x better performance** for
`StaticException`.
Both the `static` and `new` cases are seriously hampered whenever you choose to log the stacktrace.
The result for `staticExceptionAndGetStacktrace` is surprising, because the only additional operation is calling
`clone()`
on our empty `StackTraceElement[]` array. Long call stacks were not simulated with the benchmarks either.

Above benchmark code available at
[https://github.com/travishaagen/blog-static-exceptions-for-flow-control](https://github.com/travishaagen/blog-static-exceptions-for-flow-control)

## Final Thoughts

Throwing exceptions has a cost in the JVM. For optimal performance, exceptions should be thrown sparingly.
When a failure condition is not unexpected, your application's latency and throughput can benefit
from static exceptions.

## References

1. Maurer, N. (2013, November 9). The hidden performance costs of instantiating Throwables. The Thoughts of Norman
   Maurer. [http://normanmaurer.me/blog/2013/11/09/The-hidden-performance-costs-of-instantiating-Throwables/](http://normanmaurer.me/blog/2013/11/09/The-hidden-performance-costs-of-instantiating-Throwables/)
2. Why Consuming Stack Traces is Noticeably Slower in Java 11 Compared to Java 8: JMH Benchmark Results.
   (2025, November 26). javaspring.net.
   [https://www.javaspring.net/blog/consuming-stack-traces-noticeably-slower-in-java-11-than-java-8/](https://www.javaspring.net/blog/consuming-stack-traces-noticeably-slower-in-java-11-than-java-8/)
