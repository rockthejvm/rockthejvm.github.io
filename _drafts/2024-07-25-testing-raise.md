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

At RockTheJvm, we deeply understand the power of the Kotlin Arrow library and the Raise DSL, and we've previously shared our insights in our article on [Functional Error Handling in Kotlin, Part 3: The Raise DSL](https://blog.rockthejvm.com/functional-error-handling-in-kotlin-part-3/). Now, we're ready to delve into the crucial topic of testing applications that use the Raise DSL. To fully grasp the concepts we'll be discussing, it's essential to have a solid understanding of Kotlin, Arrow library, and the Raise DSL. We recommend revisiting our previous posts or resources if you need to refresh your knowledge. Get ready to explore the testing world of Arrow Raise with confidence and understanding.

## 1. Setting up the project

First things first, let's set up a Kotlin project. We can use Gradle or Maven, but for this article, we'll focus on Gradle. As always, use the `gradle init` command from a shell to create a Gradle application from scratch. We'll then add the following dependencies to the `build.gradle.kts` file, providing you with practical examples and clear explanations along the way:

```kotlin
dependencies {
    implementation("io.arrow-kt:arrow-core:1.2.4")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.9.0-RC")
    testImplementation("org.jetbrains.kotlin:kotlin-test-junit5")
    testImplementation("org.junit.jupiter:junit-jupiter-engine:5.10.0")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.9.0-RC")
    testImplementation("org.assertj:assertj-core:3.26.3")
    testImplementation("io.kotest:kotest-runner-junit5:5.9.0")
    testImplementation("io.kotest.extensions:kotest-assertions-arrow:1.4.0")
    testImplementation("io.kotest.extensions:kotest-assertions-arrow-fx-coroutines:1.4.0")
    testImplementation("org.mockito.kotlin:mockito-kotlin:5.4.0")
}
```

Notice that we're using version `1.9.0-RC` of the coroutines library since we need support for the Kotlin compiler's version 2.0.0.

We need to enable the usage of context receivers since they're still an experimental feature in Kotlin 2.0.0. We need to add the following code to the `build.gradle.kts` file to do so:

```kotlin
tasks.withType<KotlinCompile>().configureEach {
    compilerOptions {
        freeCompilerArgs.add("-Xcontext-receivers")
    }
}
```

Please take care that the way we can provide options to the compiler has changed since the article we wrote on context receivers (see [Kotlin Context Receivers: A Comprehensive Guide ](https://blog.rockthejvm.com/kotlin-context-receivers/) for further details).

Once we set up the project, we need some helpful use cases to test since this article will focus on testing the Raise DSL. We'll take a typical example. We want to create some _business logic_ to create a stock portfolio for a user. Initially, we want to make an empty portfolio for the user, adding some initial amount of money. We can model the use case as follows:

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

Let's analyze the above code briefly since the function's signature tells us a lot of helpful information. First, we have a `suspend` function, which tells us that the function will perform some effectful operation. It will likely persist the newly created portfolio in some databases. Once the new portfolio is created, the function returns its identifier, representing the happy path. Finally, the function can raise a `DomainError` in case of failure since it's declared in the context of `Raise<DomainError>` (again, check out the article [Functional Error Handling in Kotlin, Part 3: The Raise DSL](https://blog.rockthejvm.com/functional-error-handling-in-kotlin-part-3/) for further details).

We can start implementing the use case now that we have set up. Since we are diligent and well-behaved developers, **we want to write tests before implementing the use case**, following the Test-Driven Development (TDD) approach.

## 2. Testing the use case

First, we want to test the happy path, meaning the use case creates a new portfolio for the user. We need to implement our use case interface with a concrete class, which we usually call a service:

```kotlin
fun createPortfolioUseCase(): CreatePortfolioUseCase =
    object : CreatePortfolioUseCase {
        context (Raise<DomainError>)
        override suspend fun createPortfolio(model: CreatePortfolio): PortfolioId = TODO()
    }
```

As you might have noticed, the `createPortfolioUseCase` method is nothing more than a factory method.

We'll use different testing frameworks. Let's begin with a setup that should be familiar to developers addicted to Kotlin and Spring: **JUnit 5 as the test runtime and AssertJ for assertions**.

```kotlin
internal class CreatePortfolioUseCaseJUnit5Test {
    
    private val underTest = createPortfolioUseCase()

    @Test
    internal fun `given a userId and an initial amount, when executed, then it create the portfolio`() = 
        runTest { 
            TODO("Not yet implemented") 
        }
}
```

For convention, we call the unit we want to test `underTest`. Moreover, we used a typical given-when-then structure for the test's name. Furthermore, we must wrap the whole test inside a `runTest` call that gives us the coroutine context required to run a suspending function.

Now, we have to test that, given some inputs, the function will return the expected output. However, here we have a complication. The function is declared in the `Raise<DomainError>` context, which means that the code that calls it should be able to handle a possible error of type `DomainError`. The below implementation will not even compile:

```kotlin
@Test
internal fun `given a userId and an initial amount, when executed, then it create the portfolio`() = 
    runTest { 
        val actualResult: PortfolioId = 
            underTest.createPortfolio(CreatePortfolio(UserId("bob"), Money(1000.0))) 
    }
```

The compiler gives us the following error:

```
No context receiver for 'arrow.core.raise.Raise<in.rcard.arrow.raise.testing.DomainError>' found.
```

Nothing more true. Unfortunately, we can't build our instance of the `Raise<E>` type. So, we need to use what the Arrow library gives us. One handful approach is **to transform the `Raise<E>.() -> A` function in a more manageable `() -> Either<E, A>` function**. Arrow gives us all the builders to create wrapped types from a function with a `Raise<E>` context, so let's use them:

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

The `assertThat` function is a static method of the `org.assertj.core.api.Assertions`, and we take advantage of the `getOrNull` method of the `Either` type to get the value of the `Right` side of the `Either` type. The AssertJ library has no built-in support for Arrow types. However, we can use the assertions imported by the `assertj-arrow-core` library (if you're asking, the library's author is me). **The library has a lot of fluent assertions tailored to the Arrow library**. For example, we can use the assertions on the `Either<E, A>` type directly, changing the above test as follows:

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

Now, we can implement the function `createPortfolio` to make the test pass. Let's do it in a dumb way, returning a fixed value:

```kotlin
fun createPortfolioUseCase(): CreatePortfolioUseCase =
    object : CreatePortfolioUseCase {
        context(Raise<DomainError>)
        override suspend fun createPortfolio(model: CreatePortfolio): PortfolioId = PortfolioId("1")
    }
```

If we run our test, it should be green.

Instead of transforming the `Raise<E>.() -> A` function in a `() -> Either<E, A>` function, **we can use the `fold` function provided by the Arrow library**:

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

However, the above code is cumbersome and less readable than the previous one. Moreover, we must apply a `fold` function whenever we want to test a function declared in a `Raise<E>` context. Fortunately, the `assertj-arrow-core` does it for us, defining some handful of assertions that use the `fold` function under the hood:

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

The test is less readable than the one with the `either` builder because the library doesn't support suspending functions (at least for now). However, the `RaiseAssert` class is a powerful tool to test functions declared in a `Raise<E>` context. As we said, the `succeedWith` uses the `fold` function under the hood, so we don't need to worry about it.

We have used JUnit 5 until now. However, we can switch to Kotest. **Kotest is a robust testing framework for Kotlin**, which is very close to ScalaTest for the Scala language. Kotest also has a set of tailored assertions for some of the available types in the Arrow library (see the [documentation](https://kotest.io/docs/assertions/arrow.html) for further details).

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

This article is not a tutorial on how to use Kotest. However, we can notice that tests are not methods of the class, as in JUnit5. Tests are lambdas with receiver given to the `ShouldSpec` class, one of the available contexts. If you extend the `ShouldSpec` class, you can access the `context` and `should` methods, declared functions in the `ShouldSpecRootScope`. The lambda you give to the `ShouldSpec` constructor is defined with the `ShouldSpecRootScope` context. As you might notice, we didn't use the `runTest` or `runBlocking` functions since the `should` method accepts a suspending function itself.

In this test, we used the approach of converting the `Raise<E>.() -> A` context into an `() -> Either<E, A>`, and then we took advantage of the `shouldBeRight` assertions given by the `kotest-assertions-arrow` library.

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

So, we briefly introduce how to test a function that can raise an error of type `E`. What if the function calls another function that can raise an error, too? We should not depend on code outside the unit we're testing. At the end of the day, we focus on unit testing. Let's see what the available options are.

## 3. Mocking the Raise DSL

When dealing with unit tests and depending on a component outside our unit under test, we have at least two choices: fake objects and mocks. But before going deep into the topic, let's prepare the playground.

Let's say we only have one portfolio per user. So, we need to check if the user already has a portfolio before creating a new one. **If we want to use a hexagonal architecture pattern, the use case should retrieve the information through an interface we call _a port_**. A port is a way for our business logic (or application) to be driven by or to drive external systems without depending directly on them. Okay, too much theory for today. Let's jump into the code.

First, we now have a way for our use case to fail: A portfolio for a user may already exist. So, we need to add a new error:

```kotlin
sealed interface DomainError {
    data class PortfolioAlreadyExists(val userId: UserId) : DomainError
}
```

Then, we can define the new port interface. Given a `userId`, we can count the user's portfolios.

```kotlin
interface CountUserPortfoliosPort {
    context(Raise<DomainError>)
    suspend fun countByUserId(userId: UserId): Int
}
```

Finally, let's wire all the things together, starting using our port into the use case:

```kotlin
fun createPortfolioUseCase(countUserPortfolios: CountUserPortfoliosPort): CreatePortfolioUseCase =
    object : CreatePortfolioUseCase {
        context(Raise<DomainError>)
        override suspend fun createPortfolio(model: CreatePortfolio): PortfolioId {
            ensure(countUserPortfolios.countByUserId(model.userId) == 0) {
                raise(DomainError.PortfolioAlreadyExists(model.userId))
            }
            return PortfolioId("1")
        }
    }
```

We're passing the port directly as an input parameter of the factory function `createPortfolioUseCase`.

As you might expect, we need to change our tests too. When creating the use case instance under test, we must understand what to pass to the `createPortfolioUseCase` function. As we said, we have at least two choices. The first one is called fake objects. Martin Fowler defines a fake object as follows (see the articles [Test Double](https://martinfowler.com/bliki/TestDouble.html) for further details):

> Fake objects actually have working implementations, but they usually take some shortcut, which makes them unsuitable for production.

In our case, we need to implement the port for our test. For example, we want to implement a test verifying the use case when the user already has a portfolio. We can implement the port as follows:

```kotlin
private val fakeCountUserPortfolios: CountUserPortfoliosPort =
    object : CountUserPortfoliosPort {
        context(Raise<DomainError>)
        override suspend fun countByUserId(userId: UserId): Int = 
            if (userId == UserId("bob")) 0 else 1
    }
```

So, if the given `userId` is not `bob`, the fake implementation will show the user has a portfolio. Otherwise, it returns that the user has no portfolio. We can use the `fakeCountUserPortfolios` object to create the use case under test and implement the test we need using Kotest:

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

Implementing the test in JUnit 5 is the same, so we'll omit it.

So, we didn't find many concerns about using fake objects. The only annoying thing is that we must implement our port twice: one for production execution and one for tests. Many developers prefer something other than this trade-off and use stubs and mocks.

We start again from the definition of mocks given by Martin Fowler to introduce stubs and mocks:

> **Stubs** provide canned answers to calls made during the test, usually not responding at all to anything outside what's programmed in for the test.
> **Mocks** are pre-programmed with expectations which form a specification of the calls they are expected to receive. They can throw an exception if they receive a call they don't expect and are checked during verification to ensure they got all the calls they were expecting. 

For the sake of simplicity, we'll call both test doubles as *mocks*. So, **mocks are partial implementations of an interface that we can program to respond to a given set of inputs (signals), and we can verify that they received the expected signals**.

There are a lot of libraries that can help us to create mocks. The most famous in Java is [Mockito](https://site.mockito.org/). In the Kotlin ecosystem, its counterpart is called [MockK](https://mockk.io/) instead. MockK is a mocking library that is idiomatic to Kotlin. It's mighty and easy to use. Again, this article is intended to be something other than a tutorial on how to use MockK. So, we'll focus solely on how to use it to mock a function defined with a `Raise<E>` context.

**Mocking a dependency is a three-step process**. First, you need to retrieve from the library an empty mock of the dependency:

```kotlin
val countUserPortfoliosMock: CountUserPortfoliosPort = mockk()
```

The `mockk()` factory function provides a proxy to the port we can use to instrument our needs. The second step is the instrumentation of the mock indeed. Based on what the tutorials said on the net, we should instrument the `countUserPortfoliosMock` in the following way:

```kotlin
coEvery { 
    countUserPortfoliosMock.countByUserId(UserId("bob"))
} returns 0
```

The above code translates to the following sentence: "Every time we call the method `countByUserId` of the instance `countUserPortfoliosMock` with input equals to `UserId("bob")`, we'll get the value `0` as a result." Note that we are using the `coEvery` function of MockK since we declared the `countByUserId` function to be a suspending function. Despite that, we get an error if we try to compile it:

```
No context receiver for 'arrow.core.raise.Raise<in.rcard.arrow.raise.testing.DomainError>' found.
```

The compiler tells the truth. We defined the `countByUserId` function using the `Raise<DomainError>` context. We should remember that **declaring a context receiver is like adding an implicit input parameter to the list of explicitly declared input parameters**. So, the compiler tells us we're not giving enough parameters for the function to be mocked.

We need to add the missing parameter with a matcher, as with any other input parameter. The only difference is that we need the `Raise<DomainError>` at the scope level. Then, we can use the `with` scope function once again:

```kotlin
coEvery {
    with(any<Raise<DomainError>>()) {
        countUserPortfoliosMock.countByUserId(UserId("bob"))
    }
} returns 0
```

Now, the compiler is happier, and we can proceed with the rest of the test code:

```kotlin
should("create a portfolio for a user (using mockk") {
    val countUserPortfoliosMock: CountUserPortfoliosPort = mockk()
    val underTestWithMock = createPortfolioUseCase(countUserPortfoliosMock)
    coEvery {
        with(any<Raise<DomainError>>()) {
            countUserPortfoliosMock.countByUserId(UserId("bob"))
        }
    } returns 0
    val actualResult: Either<DomainError, PortfolioId> =
        either {
            underTestWithMock.createPortfolio(CreatePortfolio(UserId("bob"), Money(1000.0)))
        }
    actualResult.shouldBeRight(PortfolioId("1"))
}
```

There is another way to achieve the same result. We need a `Raise<E>` instance to mock our function. Using the matchers inside the scoped function `with` is one way. Another way is to provide an actual instance of a `Raise<E>`. As we might know, the only way to retrieve one is using the `Raise.fold` function or one of the builder methods, like the `Raise.either` we saw. In our case, we were already using the `either` function. So, it's enough to extend its scope:

```kotlin
should("create a portfolio for a user (using mockk and either") {
    val countUserPortfoliosMock: CountUserPortfoliosPort = mockk()
    val underTestWithMock = createPortfolioUseCase(countUserPortfoliosMock)
    val actualResult: Either<DomainError, PortfolioId> =
        either {
            coEvery {
                countUserPortfoliosMock.countByUserId(UserId("bob"))
            } returns 0
            underTestWithMock.createPortfolio(CreatePortfolio(UserId("bob"), Money(1000.0)))
        }
    actualResult.shouldBeRight(PortfolioId("1"))
}
```

We left out only the case in which we want to mock a function that raises an error. As you can imagine, it's close to what we've just done. The main difference is that we cannot use the `returns` function on the MockK scope. We don't want to throw an exception. So, the `throws` function is not suitable too. We need to use the `answers` generic function indeed.

First, we need to add a new error to our errors hierarchy:

```kotlin
sealed interface DomainError {
    data class PortfolioAlreadyExists(
        val userId: UserId,
    ) : DomainError

    data class GenericError(
        val throwable: Throwable,
    ) : DomainError
}
```

The `GenericError` represents an unexpected error, such as a problem with the network. Then, we can implement the test:

```kotlin
should("return a PortfolioAlreadyExists error for an existing user") {
    val countUserPortfoliosMock: CountUserPortfoliosPort = mockk()
    val underTestWithMock = createPortfolioUseCase(countUserPortfoliosMock)
    val exception = RuntimeException("Ooops!")
    val actualResult: Either<DomainError, PortfolioId> =
        either {
            coEvery {
                countUserPortfoliosMock.countByUserId(UserId("bob"))
            } answers {
                raise(GenericError(exception))
            }
            underTestWithMock.createPortfolio(CreatePortfolio(UserId("bob"), Money(1000.0)))
        }
    actualResult.shouldBeLeft(GenericError(exception))
}
```

The above test verifies the behavior of the use case when there was an error during the execution of the port. We used the `answers` function, the most generic one available in MockK.

For completeness, we can translate the above test using Mockito to understand the differences between the two libraries. In detail, we'll use the library `mockito-kotlin` on top of Mockito to have a more idiomatic look and feel:

```kotlin
@Test
fun `given a userId and an initial amount, when executed with error, then propagates the error properly`() {
    runTest {
        val exception = RuntimeException("Ooops!")
        val actualResult =
            either {
                val countUserPortfoliosPort =
                    mock<CountUserPortfoliosPort> {
                        onBlocking { countByUserId(UserId("bob")) } doAnswer { raise(GenericError(exception)) }
                    }
                val underTest = createPortfolioUseCase(countUserPortfoliosPort)
                underTest.createPortfolio(CreatePortfolio(UserId("bob"), Money(1000.0)))
            }
        EitherAssert.assertThat(actualResult).containsOnLeft(GenericError(exception))
    }
}
```

As we can see, the only part that has changed is the DSL offered by the mocking library. The rest of the test remains more or less the same.

## 4. Conclusions

In this article, we saw how to test a function declared in a `Raise<E>` context. We introduced several approaches and used many libraries. We saw how to write a test using JUnit 5 and the Kotest testing framework. We saw how to use the `assertj-arrow-core` library to test functions using the Raise DSL. Then, we translated the same tests using the Kotest extension asserts for Arrow. Then, we focused on mocking. The first approach we approached was using fake objects. Despite some initial programming, the fake objects approach is natural and ergonomic. We also saw how to mock a function declared in a `Raise<E>` context using the MockK and Mockito libraries. Mocking feels less idiomatic and more complex due to the `Raise<E>` context, which neither libraries manage natively. As of today, we still miss the need to include Mockito's extensions to handle functions using the Raise DSL natively, which would make the mocking process more straightforward.

## 5. Appendix A

As many of you might know, Kotlin _context receivers_ were deprecated in more recent versions of Kotlin, and they are eligible to be deleted in future versions in favor of [context parameters](https://github.com/Kotlin/KEEP/issues/367). It was also assured by Kotlin staff that there would be no gap between the two (see [this YouTrack issue](https://youtrack.jetbrains.com/issue/KT-8087/Make-it-possible-to-suppress-warnings-globally-in-compiler-via-command-line-option#focus=Comments-27-10137847.0-0) for further details).

Given the above information, it's perfectly right to say you want to use context receivers. Let's see how we can rewrite the use case and the test to continue using the Raise DSL without context receivers.

As we know, a single context receiver is equivalent to declaring it as a function receiver. So, the definition of our use case becomes as follows:

```kotlin
interface CreatePortfolioUseCaseWoContextReceivers {
    suspend fun Raise<DomainError>.createPortfolio(model: CreatePortfolio): PortfolioId
}

fun createPortfolioUseCaseWoContextReceivers(): CreatePortfolioUseCaseWoContextReceivers =
    object : CreatePortfolioUseCaseWoContextReceivers {
        override suspend fun Raise<DomainError>.createPortfolio(model: CreatePortfolio): PortfolioId = PortfolioId("1")
    }
```

Easy-peasy. Now, we need to change the code to use the new version of the use case. We can't call the `createPortfolio` function directly on the `CreatePortfolioUseCaseWoContextReceivers` type, which now becomes a receiver. We need to use some scope function to create a scope with an instance of the `CreatePortfolioUseCaseWoContextReceivers` type. Usually, we use the `with` scope function for this purpose. Here is how the JUnit 5 test changes:

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
