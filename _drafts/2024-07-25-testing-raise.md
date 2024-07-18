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
internal class CreatePortfolioUseCaseTest {
    
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
