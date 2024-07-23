---
title: "Raise Your Tests: Testing Arrow Raise"
date: 2024-04-10
header:
    image: "https://res.cloudinary.com/dkoypjlgr/image/upload/f_auto,q_auto:good,c_auto,w_1200,h_300,g_auto,fl_progressive/v1715952116/blog_cover_large_phe6ch.jpg"
tags: [kotlin, arrow]
excerpt: ""
toc: true
toc_label: "In this article"
---

_By [Riccardo Cardin](https://github.com/rcardin)_

At RockTheJvm we know very well the power of the Kotlin Arrow library. In details, we love the Raise DSL and its features. We already wrote about it (check out [Functional Error Handling in Kotlin, Part 3: The Raise DSL](https://blog.rockthejvm.com/functional-error-handling-in-kotlin-part-3/)), but now it's time to understand how to test an application that uses the Raise DSL. Fasten your seatbelt, because we are going to dive into the testing world of Arrow Raise.

## 1. Setting up the project

First of all, we need to set up a Kotlin project. We can use Gradle or Maven, but in this article we are going to use Gradle. As usual, use the `gradle init` command from a shell to create a Gradle application from scratch. We need to add the following dependencies to the `build.gradle.kts` file:

```kotlin
dependencies {
    implementation("io.arrow-kt:arrow-core:1.2.4")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.9.0-RC")
    testImplementation("org.jetbrains.kotlin:kotlin-test-junit5")
    testImplementation(libs.junit.jupiter.engine) <<-- Understnd this
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.9.0-RC")
    testImplementation("org.assertj:assertj-core:3.26.3")
    testImplementation("io.kotest:kotest-runner-junit5:5.9.0")
    testImplementation("io.kotest.extensions:kotest-assertions-arrow:1.4.0")
    testImplementation("io.kotest.extensions:kotest-assertions-arrow-fx-coroutines:1.4.0")
}
```

Notice that we're using version `1.9.0-RC` of coroutines library since we need the support for version 2.0.0 of the Kotlin compiler

We need to enable the usage of context receivers since they're still an experimental feature in Kotlin 2.0.0. We need to add the following code to the `build.gradle.kts` file to do so:

```kotlin
tasks.withType<KotlinCompile>().configureEach {
    compilerOptions {
        freeCompilerArgs.add("-Xcontext-receivers")
    }
}
```

Please, take care that way we can provide options to compiler is changed since the article we wrote on context receivers (see [Kotlin Context Receivers: A Comprehensive Guide ](https://blog.rockthejvm.com/kotlin-context-receivers/) for further details).

By the way, in the appendix you can find the full `build.gradle.kts` file.

Once we set up the project, we need some useful use case to test since this article will focus on testing the Raise DSL. We'll take a common example. We want to create some _business logic_ to create a stocks' portfolio for a user. Initially, we want to create a empty portfolio for the user, adding some initial amount of money. We can model the use case as follows:

```kotlin
interface CreatePortfolioUseCase {
    context (Raise<DomainError>)
    suspend fun createPortfolio(model: CreatePortfolio): PortfolioId
}

data class CreatePortfolio(
    val userId: UserId,
    val amount: Money,
)

@JvmInline
value class UserId(val id: String)

@JvmInline
value class PortfolioId(val id: String)

sealed interface DomainError
```

Let's analyze the above code for a second since the signature of the function tells us a lot of useful information. First, we have a `suspend` function, which tells us that the function will perform some effectful operation. We can guess that it will persist the newly created portfolio into some kind of database. Once created the new portfolio, the function returns its identifier, which represents the happy path. Finally, the function can raise a `DomainError` in case of failure since it's declared in the context of `Raise<DomainError>` (again, check out the article [Functional Error Handling in Kotlin, Part 3: The Raise DSL](https://blog.rockthejvm.com/functional-error-handling-in-kotlin-part-3/) for further details).

Now that we set up the use case, we can start implementation. Since we're diligent and well behaved developers, we want to write some tests before implementing the use case, following the Test-Driven Development (TDD) approach.

## 2. Testing the use case

First, we want to test the happy-path, which means that the use case creates a new portfolio for the user. We need to implement our use case interface with a concrete class, which usually we call a service:

```kotlin
fun createPortfolioUseCase(): CreatePortfolioUseCase =
    object : CreatePortfolioUseCase {
        context (Raise<DomainError>)
        override suspend fun createPortfolio(model: CreatePortfolio): PortfolioId = TODO()
    }
```

As you might have noticed, the `createPortfolioUseCase` method is nothing more than a factory method.

We'll use different testing frameworks. Let's begin with a setup that should be quite familiar to developers that uses Kotlin together with Spring: JUnit 5 as the test runtime and AssertJ for assertions.

```kotlin
internal class CreatePortfolioUseCaseJUnit5Test {
    
    private val underTest = createPortfolioUseCase()

    @Test
    internal fun `given a userId and an initial amount, when executed, then it create the portfolio`() = runTest {
        TODO("Not yet implemented")
    }
}
```

For convention, we call the unit we want to test with the name `underTest`. Moreover,  we used a common given-when-then structure for the name of the test. Moreover, not that we need to wrap the whole test inside a `runTest` call that gives us the coroutine context needed to run a suspending function.

Now, we have to test that given some inputs the function will return the expected output. However, here we have a complication. The function is declared in the `Raise<DomainError>` context, which means that the code that calls it should be able to handle a possible error of type `DomainError`. In fact, the below implementation will not even compile:

```kotlin
@Test
internal fun `given a userId and an initial amount, when executed, then it create the portfolio`() = runTest {
    val actualResult: PortfolioId =
        underTest.createPortfolio(CreatePortfolio(UserId("bob"), Money(1000.0)))
}
```

In fact, the compiler gives us the following error:

```
No context receiver for 'arrow.core.raise.Raise<in.rcard.arrow.raise.testing.DomainError>' found.
```

Nothing more true. Unfortunately, we can't build an instance of the `Raise<E>` type our own. So, we need to use what the Arrow library gives us. One handful approach is to transform the `Raise<E>.() -> A` function in a more manageable `() -> Either<E, A>` function. Arrow gives us all the builders to create wrapped types from a function with a `Raise<E>` context, so, let's use them:

```kotlin
val actualResult: Either<DomainError, PortfolioId> =
    either {
        underTest.createPortfolio(CreatePortfolio(UserId("bob"), Money(1000.0)))
    }
```

Now that we have a manageable function, we can add the assertion to our test, which becomes the following:

```kotlin
@Test
internal fun `given a userId and an initial amount, when executed, then it create the portfolio`() =
    runTest {
        val actualResult: Either<DomainError, PortfolioId> =
            either {
                underTest.createPortfolio(CreatePortfolio(UserId("bob"), Money(1000.0)))
            }
        Assertions.assertThat(actualResult.getOrNull()).isEqualTo(PortfolioId("1"))
    }
```

The `assertThat` function is a static method of the `org.assertj.core.api.Assertions`, and we take advantage of the `getOrNull` method of the `Either` type to get the value of the `Right` side of the `Either` type. In fact, the AssertJ library has no built-in support for Arrow types. However, we can use the assertions imported by the `assertj-arrow-core` library (if you're asking, the author of the library is me). The library as a lot of fluent assertions tailored on the Arrow library. For example, we can use the assertions on the `Either<E, A>` type directly, changing the above test as follows:

```kotlin
@Test
internal fun `given a userId and an initial amount, when executed, then it create the portfolio (using AssertJ-Arrow-Core)`() =
    runTest {
        val actualResult: Either<DomainError, PortfolioId> =
            either {
                underTest.createPortfolio(CreatePortfolio(UserId("bob"), Money(1000.0)))
            }
        EitherAssert.assertThat(actualResult).containsOnRight(PortfolioId("1"))
    }
```

As we can see, we don't need to extract the value from the `Either<E, A>` anymore. 

Now, we can implement the function `createPortfolio` to make the test pass. Let's do it in the dumb way, returning a fixed value:

```kotlin
fun createPortfolioUseCase(): CreatePortfolioUseCase =
    object : CreatePortfolioUseCase {
        context(Raise<DomainError>)
        override suspend fun createPortfolio(model: CreatePortfolio): PortfolioId = PortfolioId("1")
    }
```

If we run our test, it should be green.

Instead of transforming the `Raise<E>.() -> A` function in a `() -> Either<E, A>` function, we can use the `fold` function provided by the Arrow library:

```kotlin
@Test
internal fun `given a userId and an initial amount, when executed, then it create the portfolio (using fold)`() =
    runTest {
        fold(
            block = { underTest.createPortfolio(CreatePortfolio(UserId("bob"), Money(1000.0))) },
            recover = { Assertions.fail("The use case should not fail") },
            transform = { Assertions.assertThat(it).isEqualTo(PortfolioId("1")) },
        )
    }
```

However, the above code is cumbersome and less readable than the previous one. Moreover, we need to apply a `fold` function every time we want to test a function declared in a `Raise<E>` context. Fortunately the `assertj-arrow-core` does it for us, defining some handful assertions that use the `fold` function under the hood:

```kotlin
@Test
internal fun `given a userId and an initial amount, when executed, then it create the portfolio (using RaiseAssert)`() =
    RaiseAssert
        .assertThat {
            runBlocking {
                underTest.createPortfolio(
                    CreatePortfolio(
                        UserId("bob"),
                        Money(1000.0),
                    ),
                )
            }
        }.succeedsWith(PortfolioId("1"))
```

The test seems to be less readable than the previous just because the library doesn't support suspending functions (at least for now). However, the `RaiseAssert` class is a powerful tool to test functions declared in a `Raise<E>` context. As we said, the `succeedWith` uses the `fold` function under the hood, so we don't need to worry about it. 

We used JUnit 5 until now. However, we can switch to Kotest for sure. Kotest is a powerful testing framework for Kotlin, which is very closed to ScalaTest for the Scala language. Kotest has also a set of tailored assertions for some of the available types in Arrow library (see the [documentation](https://kotest.io/docs/assertions/arrow.html) for further details).

So, let's translate the above tests in Kotest notation.

```kotlin
internal class CreatePortfolioUseCaseKotestTest : ShouldSpec({

    val underTest = createPortfolioUseCase()

    context("The create portfolio use case") {
        should("create a portfolio for a user") {
            val actualResult: Either<DomainError, PortfolioId> =
                either {
                    underTest.createPortfolio(CreatePortfolio(UserId("bob"), Money(1000.0)))
                }

            actualResult.shouldBeRight(PortfolioId("1"))
        }
    }
})
```

This article is not a tutorial on how to use Kotest. However, we can notice that tests are not methods of the class, as in JUnit5. Tests are created lambda given to the `ShouldSpec` class, which is one of the available context. If you extend the `ShouldSpec` class, you have access to the `context` and `should` method, which are declared as function in the `ShouldSpecRootScope`. In fact, the lambda you give to the `ShouldSpec` constructor is defined with the `ShouldSpecRootScope` context. As you might notice, we didn't use the `runTest` or `runBlocking` functions since the `should` method accepts a suspending function itself.

In this test we used the approach of converting the `Raise<E>.() -> A` context into an `Either<E, A>`, and then we take advantage of the `shouldBeRight` assertions given by the `kotest-assertions-arrow` library.

Unfortunately, Kotest has no extension to test a `Raise<E>.() -> A` function directly. So, if we don't want to convert it into an `Either<E, A>`, we need to use the approach with the `fold` method:

```kotlin
should("create a portfolio for a user (using fold)") {
    fold(
        block = { underTest.createPortfolio(CreatePortfolio(UserId("bob"), Money(1000.0))) },
        recover = { fail("The use case should not fail") },
        transform = { it.shouldBe(PortfolioId("1")) },
    )
}
```

So, we briefly introduce how to test a function that can raise an error of type `E`. What if the function calls another function that can raise an error too? We should not depend on code that is outside the unit we're testing. At the end of the day, we focus on unit test. Let's see what're the available options.

## Mocking the Raise DSL

When we're dealing with unit tests, and we depend on a component that is outside our unit under test, we have at least two choices: Fake objects and mocks. But, before going deep into the topic, let's prepare the playground. 

Let's say that we have the constraint that we can't have more than one portfolio per user. So, we need to check if the user already has a portfolio before creating a new one. If we want to use a hexagonal architecture pattern, the use should retrieve the information through an interface we call _a port_. To keep it simple, a port is a way for our business logic (or application) to be driven by or to drive external system, without depending directly from them. Okay, too much theory for today. Let's jump into the code.

First, we now have a way for our use case to fail: A portfolio for a user may already exists. So, we need to add a new error:

```kotlin
sealed interface DomainError {
    data class PortfolioAlreadyExists(val userId: UserId) : DomainError
}
```

Then, we can define the new port interface. Given a `userId`, we can retrieve the number of portfolios the user has.

```kotlin
interface CountUserPortfoliosPort {
    context(Raise<DomainError>)
    suspend fun countByUserId(userId: UserId): Int
}
```

Finally,. let's wire all the things together, starting using our port into the use case:

```kotlin
fun createPortfolioUseCase(countUserPortfolios: CountUserPortfoliosPort): CreatePortfolioUseCase =
    object : CreatePortfolioUseCase {
        context(Raise<DomainError>)
        override suspend fun createPortfolio(model: CreatePortfolio): PortfolioId =
            if (countUserPortfolios.countByUserId(model.userId) > 0) {
                raise(DomainError.PortfolioAlreadyExists(model.userId))
            } else {
                PortfolioId("1")
            }
    }
```

We're passing the port directly as an input parameter of the factory function called `createPortfolioUseCase`.

As you might expect, we need to change our tests too. We need to understand what to pass to the `createPortfolioUseCase` function when creating the instance of the use case under test. As we said, we have at least two choices. The first one is called fake objects. Martin Fowler defines a fake object as follows (see the articles [Test Double](https://martinfowler.com/bliki/TestDouble.html) for further details):

> Fake objects actually have working implementations, but usually take some shortcut which makes them not suitable for production.

In our case, we need to implement the port we defined in a way suitable for our test. For example, let's say we want to implement a test verifying the use case when the user already has a portfolio. We can implement the port as follows:

```kotlin
private val fakeCountUserPortfolios: CountUserPortfoliosPort =
    object : CountUserPortfoliosPort {
        context(Raise<DomainError>)
        override suspend fun countByUserId(userId: UserId): Int = if (userId == UserId("bob")) 0 else 1
    }
```

So, if the given `userId` is anything than `bob`, the fake implementation returns that the user has a portfolio. Otherwise, it returns that the user has no portfolio. We can use the `fakeCountUserPortfolios` object to create the use case under test, and implement the test we need using Kotest:

```kotlin
should("raise a PortfolioAlreadyExists") {
    val alice = UserId("alice")
    val actualResult: Either<DomainError, PortfolioId> =
        either {
            underTest.createPortfolio(CreatePortfolio(alice, Money(1000.0)))
        }
    actualResult.shouldBeLeft(PortfolioAlreadyExists(alice))
}
```

The implementation of the test in JUnit 5 is more or less the same, so we'll omit it.


## XX. Appendix A

As many of you might know, Kotlin _context receiver_ were deprecated in more recent versions of Kotlin, and they are eligible to be deleted in future versions in favor of [context parameters](https://github.com/Kotlin/KEEP/issues/367). It was also assured by Kotlin staff that there will be no gap between the two (see [this YouTrack issue](https://youtrack.jetbrains.com/issue/KT-8087/Make-it-possible-to-suppress-warnings-globally-in-compiler-via-command-line-option#focus=Comments-27-10137847.0-0) for further details). 

In face of the above information, it's perfectly right to say you want to use context receivers. Let's see together how we can rewrite the use case and the test to continue using the Raise DSL without context receivers.

As we know, a single context receiver is equivalent to declare it as a function receiver. So, the definition of our use case becomes as follows:

```kotlin
interface CreatePortfolioUseCaseWoContextReceivers {
    suspend fun Raise<DomainError>.createPortfolio(model: CreatePortfolio): PortfolioId
}

fun createPortfolioUseCaseWoContextReceivers(): CreatePortfolioUseCaseWoContextReceivers =
    object : CreatePortfolioUseCaseWoContextReceivers {
        override suspend fun Raise<DomainError>.createPortfolio(model: CreatePortfolio): PortfolioId = PortfolioId("1")
    }
```

Easy-peasy. Now, we need to change the code that uses the new version of the use case. We can't call the `createPortfolio` function directly anymore on the `CreatePortfolioUseCaseWoContextReceivers` type, which now becomes a receiver. We need to use some scope function to create a scope with an instance of the `CreatePortfolioUseCaseWoContextReceivers` type. Usually, we use the `with` scope function for this purpose. Here is how the JUnit 5 test changes:

```kotlin
internal class CreatePortfolioUseCaseWoContextReceiversJUnit5Test {
    private val underTest = createPortfolioUseCaseWoContextReceivers()

    @Test
    internal fun `given a userId and an initial amount, when executed, then it create the portfolio`() =
        runTest {
            val actualResult: Either<DomainError, PortfolioId> =
                with(underTest) {
                    either {
                         createPortfolio(CreatePortfolio(UserId("bob"), Money(1000.0)))
                    }
                }
            Assertions.assertThat(actualResult.getOrNull()).isEqualTo(PortfolioId("1"))
        }
}
```

From here on, everything should run quite smoothly.
