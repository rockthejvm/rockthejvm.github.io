---
title: "Project Loom: Structured Concurrency in Java"
date: 2024-10-10
header:
    image: "https://res.cloudinary.com/dkoypjlgr/image/upload/f_auto,q_auto:good,c_auto,w_1200,h_300,g_auto,fl_progressive/v1715952116/blog_cover_large_phe6ch.jpg"
tags: [java, loom, structured-concurrency]
excerpt: ""
toc: true
toc_label: "In this article"
---

_By [Riccardo Cardin](https://github.com/rcardin)_

TODO ForkJoinPool

Concurrency is a beast that every developer must face at some point in their career. It is a complex topic that requires a deep understanding of the underlying mechanisms of the programming language and the runtime environment. Java developers have been dealing with concurrency for a long time, and the Java platform provides a rich set of tools to manage it. However, writing concurrent code in Java is not an easy task, and developers often struggle to write correct and efficient concurrent programs. Fortunately, Project Loom is here to help. We already covered the introduction of virtual threads in the JVM in the article [The Ultimate Guide to Java Virtual Threads](https://blog.rockthejvm.com/ultimate-guide-to-java-virtual-threads/). In this article, we will explore the concept of structured concurrency and how Project Loom simplifies writing concurrent code in Java. 

## 1. Setting up the project

Java 23 was released in mid-September, and it brought a lot of new features and improvements to the Java platform. We're interested in the third preview of structured concurrency, which is given a dedicated JEP: [JEP 480: Structured Concurrency (Third Preview)](https://openjdk.org/jeps/480). As you can imagine, we need to enable the preview features with a dedicated directive to the compiler. We'll use Maven as the build tool for our project. So, to enable structured concurrency in our project, we need to add the following configuration to the `pom.xml` file:

```xml
<build>
  <pluginManagement>
    <plugin>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.10.1</version>
      <configuration>
        <release>23</release>
        <compilerArgs>--enable-preview</compilerArgs>
      </configuration>
    </plugin>
  </pluginManagement>
</build>
```

At the end of the article, we'll share the whole `pom.xml` file for your reference.

Once we have set up the project, we can start writing some code to explore structured concurrency in Java. In detail, we need an example to work with. This time, we'll take as an example the interaction with GitHub APIs (or at least, a simplified version of them). So, without further ado, let's start writing some code.

## 2. The problem with concurrency

We'll model the retrieval of the information needed to display the repositories of a GitHub user with some other data of the user itself. The final data structure we want to make available is the following:

```java
record GitHubUser(User user, List<Repository> repositories) {}

record User(UserId userId, UserName name, Email email) {}

record UserId(long value) {}

record UserName(String value) {}

record Email(String value) {}

record Repository(String name, Visibility visibility, URI uri) {}

enum Visibility {
    PUBLIC,
    PRIVATE
}
```

Unfortunately, Java still lacks of value classes, so we need to use regular types to model data. Once [project Valhalla](https://openjdk.org/projects/valhalla/) will be completed the `UserName`, `Email`, and `Repository` classes will be replaced with proper value classes.

We define also a logger that we'll use to print some useful information to the console:

```java
private static final Logger LOGGER = LoggerFactory.getLogger("GitHubApp");
```

How can we retrieve the needed information? Well, the GitHub API provides a RESTful interface that we can use to retrieve the information we need. In details, we need to perform two requests: one to retrieve the user information and one to retrieve the repositories of the user. From the first one, we retrieve the username and the email address, while from the second one, we retrieve the repositories of the user.

We don't need to implement the actual interaction with the GitHub API. So, we'll use a simplified version of the clients. Here is the interfaces we'll use:

```java
interface FindUserByIdPort {
  User findUser(UserId userId) throws InterruptedException;
}

interface FindRepositoriesByUserIdPort {
  List<Repository> findRepositories(UserId userId) throws InterruptedException;
}
```

The implementations of these interfaces are straightforward. They simply return fixed information and simulate a delay in the response. Here is the implementation of both the interfaces:

```java
class GitHubRepository implements FindUserByIdPort, FindRepositoriesByUserIdPort {
  @Override
  public User findUser(UserId userId) throws InterruptedException {
    LOGGER.info("Finding user with id '{}'", userId);
    delay(Duration.ofMillis(500L));
    LOGGER.info("User '{}' found", userId);
    return new User(userId, new UserName("rcardin"), new Email("rcardin@rockthejvm.com"));
  }
  @Override
  public List<Repository> findRepositories(UserId userId) throws InterruptedException {
    LOGGER.info("Finding repositories for user with id '{}'", userId);
    delay(Duration.ofSeconds(1L));
    LOGGER.info("Repositories found for user '{}'", userId);
    return List.of(
        new Repository(
                "raise4s", Visibility.PUBLIC, URI.create("https://github.com/rcardin/raise4s")),
        new Repository(
                "sus4s", Visibility.PUBLIC, URI.create("https://github.com/rcardin/sus4s")));
  }
}

private static void delay(Duration duration) throws InterruptedException {
  Thread.sleep(duration);
}
```

We added a convenient `delay` method wrapping the common `Thread.sleep` method. 

As you might have noticed, we leave the `InterruptedException` unhandled, and we've not wrapped it into a `RuntimeException`. We did it on purpose. In fact, leaving the `InterruptedException` in the signature is a signal that the thread can suspend (or can be interrupted). We can call it a capability of the method. Although checked exceptions are not the best way to signal capabilities since they don't compose well, they're still the best approach in Java so far to implement an effect system in the language.

Now that we have our clients, we can start writing the code to retrieve the information we need. We'll start with the sequential version of the code.

```java
interface FindGitHubUserUseCase {
  GitHubUser findGitHubUser(UserId userId) throws InterruptedException;
}

class FindGitHubUserSequentialService implements FindGitHubUserUseCase {
  private final FindUserByIdPort findUserByIdPort;
  private final FindRepositoriesByUserIdPort findRepositoriesByUserIdPort;
  public FindGitHubUserSequentialService(
      FindUserByIdPort findUserByIdPort,
      FindRepositoriesByUserIdPort findRepositoriesByUserIdPort) {
    this.findUserByIdPort = findUserByIdPort;
    this.findRepositoriesByUserIdPort = findRepositoriesByUserIdPort;
  }
  
  @Override
  public GitHubUser findGitHubUser(UserId userId) throws InterruptedException {
    var user = findUserByIdPort.findUser(userId);
    var repositories = findRepositoriesByUserIdPort.findRepositories(userId);
    return new GitHubUser(user, repositories);
  }
}
```

Despite all the ceremony needed by Java, the above code is quite simple. The tasks are performed in sequence, and we expect to wait at least 1.5 seconds to get the result. 

Although the code is not efficient, it has a lot of good features. First, it's easy to understand. Second, the computation has a clear scope. In fact, we know for sure that after the completion of the `findGitHubUser` method the computation is done and the JVM knows it can clean up all the resources used by the computation. Third, the exception handling management is straightforward. Let's say the `findUserByIdPort.findUser(userId)` call throws an exception. The exception is propagated to the caller, and the `findRepositoriesByUserIdPort.findRepositories(userId)` will never start. No resources are wasted in this case.

We can now write the concurrent version of the code since the call to the two external APIs have no dependency. Here is the code:

```java
@Override
public GitHubUser findGitHubUser(UserId userId)
    throws InterruptedException, ExecutionException {
  try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var user = executor.submit(() -> findUserByIdPort.findUser(userId));
    var repositories =
        executor.submit(() -> findRepositoriesByUserIdPort.findRepositories(userId));
    return new GitHubUser(user.get(), repositories.get());
  }
}
```

We're using virtual threads in the above solution. We guess everybody is familiar with them since they have been available for a while now. (However, if you're not familiar with virtual thread, please refer to our previous article [The Ultimate Guide to Java Virtual Threads](https://blog.rockthejvm.com/ultimate-guide-to-java-virtual-threads/)).

The code is quite simple. We use an `ExecutorService` to create a dedicated virtual thread for every submitted execution. The result is a `Future<T>` object that we can use to retrieve the result of the computation with the `get` method. We added the `ExecutionException` to our method signature because it's possible that we'll try to retrieve a value from a `Future` that has been completed exceptionally. Clearly, we need to add the exception also to the interface method's signature.

We can try to define a `main` method to test the code we've written so far. Here is the code:

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
  final GitHubRepository gitHubRepository = new GitHubRepository();
  FindGitHubUserConcurrentService service =
      new FindGitHubUserConcurrentService(gitHubRepository, gitHubRepository);
  final GitHubUser gitHubUser = service.findGitHubUser(new UserId(1L));
  LOGGER.info("GitHub user: {}", gitHubUser);
}
```

The output of the execution of the above code is the following:

```
08:50:34.159 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
08:50:34.159 [virtual-20] INFO GitHubApp -- Finding user with id 'UserId[value=1]'
08:50:34.679 [virtual-20] INFO GitHubApp -- User 'UserId[value=1]' found
08:50:35.179 [virtual-22] INFO GitHubApp -- Repositories found for user 'UserId[value=1]'
08:50:35.183 [main] INFO GitHubApp -- GitHub user: GitHubUser[user=User[userId=UserId[value=1], name=UserName[value=rcardin], email=Email[value=rcardin@rockthejvm.com]], repositories=[Repository[name=raise4s, visibility=PUBLIC, uri=https://github.com/rcardin/raise4s], Repository[name=sus4s, visibility=PUBLIC, uri=https://github.com/rcardin/sus4s]]]
```

The output is what we expect. Both thread starts together and their execution interleaves. After more or less 0.5 seconds, the user is found, and after more or less 1 second, the repositories are found. So far so good.

What if the `findUser` method throws an exception? Maybe the network has a glitch, or the GitHub API is down. Let's simulate it by throwing an exception in the `findUser` method. Here is the code:

```java
@Override
public User findUser(UserId userId) throws InterruptedException {
  LOGGER.info("Finding user with id '{}'", userId);
  delay(Duration.ofMillis(100L));
  throw new RuntimeException("Socket timeout");
}
```

We still want to wait a bit before throwing the exception to simulate a real-world scenario. We want to make sure that the `findRepositories` method is not started. We can run the `main` method again and see what happens. The output is the following:

```
08:39:41.945 [virtual-20] INFO GitHubApp -- Finding user with id 'UserId[value=1]'
08:39:41.947 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
08:39:42.969 [virtual-22] INFO GitHubApp -- Repositories found for user 'UserId[value=1]'
Exception in thread "main" java.util.concurrent.ExecutionException: java.lang.RuntimeException: Socket timeout
	at java.base/java.util.concurrent.FutureTask.report(FutureTask.java:122)
	at java.base/java.util.concurrent.FutureTask.get(FutureTask.java:191)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindGitHubUserConcurrentService.findGitHubUser(GitHubApp.java:115)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp.main(GitHubApp.java:125)
Caused by: java.lang.RuntimeException: Socket timeout
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$GitHubRepository.findUser(GitHubApp.java:68)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindGitHubUserConcurrentService.lambda$findGitHubUser$0(GitHubApp.java:112)
```

We should start to understand the problem with plain-old concurrency. The exception is thrown, but the `findRepositories` method is still started and executed till completion. This is a problem because we're wasting resources. We call such a situation as a thread leak. However, it can be worse. Imagine for example that the thread running the `ExecutorService` throws an exception after submitting the two computation. Here is the code:

```java
@Override
public GitHubUser findGitHubUser(UserId userId) throws InterruptedException, ExecutionException {
  try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var user = executor.submit(() -> findUserByIdPort.findUser(userId));
    var repositories =
            executor.submit(() -> findRepositoriesByUserIdPort.findRepositories(userId));
    throw new RuntimeException("Something went wrong"); 
  }
}
```

If we try to run again the `main` method, we'll see the following output:

```
08:52:18.455 [virtual-20] INFO GitHubApp -- Finding user with id 'UserId[value=1]'
08:52:18.455 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
08:52:18.975 [virtual-20] INFO GitHubApp -- User 'UserId[value=1]' found
08:52:19.476 [virtual-22] INFO GitHubApp -- Repositories found for user 'UserId[value=1]'
Exception in thread "main" java.lang.RuntimeException: Something went wrong
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindGitHubUserConcurrentService.findGitHubUser(GitHubApp.java:115)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp.main(GitHubApp.java:136)
```

Even if the `main` thread went in error, both the executions of `findUser` and `findRepositories` methods didn't stop and complete their execution. Again, we found a thread leak. What we expected is that the `main` thread should be the root of the computation, the parent thread, and if it goes in error, all the computations started from it, aka its children thread, should stop. However, using plain-old concurrency, this is not the case. We can't express any association within threads.

We can try to implement our own some sort of policy to fix the above problem. However, it's not easy to do it. We need to keep track of all the threads started by the `ExecutorService` and stop them when the parent thread goes in error. It's a lot of work, and it's error-prone. We can't rely on the JVM to do it for us. We need to do it manually.

Fortunately, Project Loom provides a solution to the problem. It's called structured concurrency.

## 3. Structured concurrency

Now that we understood the concerns with what we called plain-old concurrency, let's step back to the sequential solution for a moment:

```java
@Override
public GitHubUser findGitHubUser(UserId userId) throws InterruptedException {
  var user = findUserByIdPort.findUser(userId);
  var repositories = findRepositoriesByUserIdPort.findRepositories(userId);
  return new GitHubUser(user, repositories);
}
```

As we said, despite being sequential and then not optimized, the above computation has some nice features. 

The computation has a clear scope, and the exception handling is straightforward. We can think about the calls to `findUserByIdPort.findUser(userId)` and `findRepositoriesByUserIdPort.findRepositories(userId)` methods as they were children of the `findGitHubUser` method. The language guarantees that when the `findGitHubUser` method completes, all its children computation are completed too. We cannot have children computation that outlive the parent one. 

We cannot have resource starvation or execution leaks neither. Again, it's the stack structure of modern programming languages that guarantees it. Every time a function terminates its execution, the runtime can clean up all the resources used by the computation. Moreover, if the `findUserByIdPort.findUser(userId)` method throws an exception, the `findRepositoriesByUserIdPort.findRepositories(userId)` method is not started.

We can say that the syntactic structure of the code reflects the semantic structure of the computation. We can say that the code is structured. 

Structured concurrency is a concurrency model that satisfies the above properties also in case of code that executes concurrently. It was introduced by Martin Sústrik in his blog post [Structured Concurrency](https://250bpm.com/blog:71/), and then popularized by Nathaniel J. Smith in his blog post [Notes on structured concurrency, or: Go statement considered harmful](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/). 

At the core, its main objective is to provide a way to structure a concurrent computation in a way that the syntactic structure of the code reflects the semantic structure of the concurrency, satisfying the properties we've seen so far for the sequential code. We say that an operation can fork some children tasks to run concurrently. The relationships between the parent and the children tasks form a tree. 

![Structured Concurrency Tasks Tree](/images/loom-structured-concurrency/structured-concurrency.png)

The execution of the task in a node is guaranteed to be completed only when all the children tasks are completed. If a task throws an exception, all the sibling tasks are stopped, and the exception is propagated to the parent task.

In our example, once we'll set up the structured concurrency, the execution of the `findGitHubUser` method will be completed only when the `findUserByIdPort.findUser(userId)` and `findRepositoriesByUserIdPort.findRepositories(userId)` methods are completed.

Structured concurrency was already implemented in many programming languages. If we focused on the JVM, we can mention the [Kotlin Coroutines](https://blog.rockthejvm.com/kotlin-coroutines-101/), Scala [Cats Effects Fibers](https://typelevel.org/cats-effect/docs/concepts#structured-concurrency), and [ZIO Fibers](https://zio.dev/reference/fiber/#structured-concurrency). Now, Project Loom introduces structured concurrency also for Java.

Java implements structured concurrency though the `java.util.concurrent.StructuredTaskScope` type. Let's lean how to use it directly through coding. First, we need to create the scope into which we want the properties of structured concurrency to be satisfied. Here is the code:

```java
@Override
public GitHubUser findGitHubUser(UserId userId) throws ExecutionException {
  try (var scope = new StructuredTaskScope<>()) {
    // TODO
    return null;
  }
}
```

The type is a generic type `T` that represents the result of the computations spawned from it. Since we'll often spawn more than one task with different return type, we usually bound the type variable to `Object`. The type implements the `AutoCloseable` interface, which means we can use it inside a try-with-resources block:

```java
public class StructuredTaskScope<T> implements AutoCloseable
```

The try-with-resources block delimits the scope of the structured concurrency area. We'll see in a moment that the computation will leave the `try` block only when all the subtask created inside it are completed.

Now, we need to create the subtasks. Using the structured concurrency terminology we introduced so far, the thread creating the `StructuredTaskScope` is the parent task, and the subtasks are the children tasks. Here is the code:

```java
@Override
public GitHubUser findGitHubUser(UserId userId) throws ExecutionException {
  try (var scope = new StructuredTaskScope<>()) {
    var user = scope.fork(() -> findUserByIdPort.findUser(userId));
    var repositories = scope.fork(() -> findRepositoriesByUserIdPort.findRepositories(userId));
    // TODO
    return null;
  }
}
```

As we can see, we can spawn or fork a new child task using the `fork` method of the `StructuredTaskScope` type, which is defined as follows:

```java
// Java SDK
public class StructuredTaskScope<T> implements AutoCloseable { 
  public <U extends T> Subtask<U> fork(Callable<? extends U> task)
}
```

The `fork` method takes a `Callable<T>` object as argument and returns a `Subtask<T>` object. The `Subtask<T>` object is a handle to the child task. To be fair, the `Subtask<T>` is defined as implementation of the `Supplier<T>` interface, which means we can retrieve the result of the computation using the `get` method:

```java
// Java SDK
public sealed interface Subtask<T> extends Supplier<T> permits SubtaskImpl
```

If we don't need to use any of the methods specific to the `Subtask<T>` type, and we only need to retrieve the result of the computation, we should use it as a `Supplier<T>` object. Java architects decided not to return a `java.util.concurrent.Future<T>` instance from the `fork` method to avoid confusion with the computations which are not structured, and give a clear-cut with the past.

Before getting the result from a `Subtask<T>` object, we need to make sure that the computation is completed. Since we're using the structured concurrency model, we want to synchronize on the executions of all the children tasks. So, we call the method `join` method on the `scope` object:

```java
@Override
public GitHubUser findGitHubUser(UserId userId) throws ExecutionException, InterruptedException {
  try (var scope = new StructuredTaskScope<>()) {
    var user = scope.fork(() -> findUserByIdPort.findUser(userId));
    var repositories = scope.fork(() -> findRepositoriesByUserIdPort.findRepositories(userId));
    scope.join();
    LOGGER.info("Both forked task completed");
    return null;
  }
}
```

After joining all the forked subtask, we can finally retrieve the results, since we know for sure that all the computations are completed. Here is the piece of our puzzle:

```java
@Override
public GitHubUser findGitHubUser(UserId userId) throws ExecutionException, InterruptedException {
  try (var scope = new StructuredTaskScope<>()) {
    var user = scope.fork(() -> findUserByIdPort.findUser(userId));
    var repositories = scope.fork(() -> findRepositoriesByUserIdPort.findRepositories(userId));
    scope.join();
    LOGGER.info("Both forked task completed");
    return new GitHubUser(user.get(), repositories.get());
  }
}
```

If we run the `main` method again, we'll see the following output:

```
11:06:36.350 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
11:06:36.350 [virtual-20] INFO GitHubApp -- Finding user with id 'UserId[value=1]'
11:06:36.874 [virtual-20] INFO GitHubApp -- User 'UserId[value=1]' found
11:06:37.374 [virtual-22] INFO GitHubApp -- Repositories found for user 'UserId[value=1]'
11:06:37.377 [main] INFO GitHubApp -- Both forked task completed
11:06:37.380 [main] INFO GitHubApp -- GitHub user: GitHubUser[user=User[userId=UserId[value=1], name=UserName[value=rcardin], email=Email[value=rcardin@rockthejvm.com]], repositories=[Repository[name=raise4s, visibility=PUBLIC, uri=https://github.com/rcardin/raise4s], Repository[name=sus4s, visibility=PUBLIC, uri=https://github.com/rcardin/sus4s]]]
```

First, we can see that the two forked computation effectively interleave each other. Then, you might have noticed that the `StructuredTaskScope` class uses virtual threads under the hood, as it can be seen from the thread names.

Remember, calling the `join` method before exiting the `try` block is mandatory. If we don't do it, we'll get a `java.lang.IllegalStateException` exception at runtime. For example, we can remove the call to the `join` method from the previous example:

```java
@Override
public GitHubUser findGitHubUser(UserId userId) throws ExecutionException, InterruptedException {
  try (var scope = new StructuredTaskScope<>()) {
    var user = scope.fork(() -> findUserByIdPort.findUser(userId));
    var repositories = scope.fork(() -> findRepositoriesByUserIdPort.findRepositories(userId));
    LOGGER.info("Both forked task completed");
    return new GitHubUser(user.get(), repositories.get());
  }
}
```

As we said, the execution of the above code leads to the following output:

```
21:21:03.690 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=42]'
21:21:03.690 [virtual-20] INFO GitHubApp -- Finding user with id 'UserId[value=42]'
Exception in thread "main" java.lang.IllegalStateException: Owner did not join after forking subtasks
	at java.base/java.util.concurrent.StructuredTaskScope.newIllegalStateExceptionNoJoin(StructuredTaskScope.java:440)
	at java.base/java.util.concurrent.StructuredTaskScope.ensureJoinedIfOwner(StructuredTaskScope.java:478)
	at java.base/java.util.concurrent.StructuredTaskScope$SubtaskImpl.get(StructuredTaskScope.java:931)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindGitHubUserStructuredConcurrencyService.findGitHubUser(GitHubApp.java:263)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp.main(GitHubApp.java:396)
```

It would be better to avoid such invalid sequence of steps at compile time, but I guess it's a trade-off to keep the API simple.

So, we just finish covering the happy path of structured concurrency, the way Project Loom implements it. Now, it's time to go deeper in which are the available policies for synchronization of forked tasks, and how to handle exceptions.

## 4. Synchronization Policies

Let's say that our `findUserByIdPort.findUser(userId)` method will throw an exception, as we previously simulated when we talked about plain-old concurrency. What happens to our structured concurrency computation? Let's see it in action, and executes it:

```
11:07:17.590 [virtual-20] INFO GitHubApp -- Finding user with id 'UserId[value=1]'
11:07:17.590 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
11:07:18.616 [virtual-22] INFO GitHubApp -- Repositories found for user 'UserId[value=1]'
11:07:18.632 [main] INFO GitHubApp -- Both forked task completed
Exception in thread "main" java.lang.IllegalStateException: Result is unavailable or subtask did not complete successfully
	at java.base/java.util.concurrent.StructuredTaskScope$SubtaskImpl.get(StructuredTaskScope.java:940)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindGitHubUserStructuredConcurrencyService.findGitHubUser(GitHubApp.java:156)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp.main(GitHubApp.java:166)
```

The above log is very interesting. First, we see that parent thread waits for the end of both tasks before continuing. So, the tree structure of the computation is respected. The computation went in error when we tried to get the result from the failed computation. Moreover, the second task was not canceled once the first had gone in error. It seems we didn't achieve much with structured concurrency until now.

The issue is the type of the scope we chose to use. In fact, the `StructuredTaskScope<T>` doesn't implement any advanced policy for error handling. It's a simple blueprint to build advanced policies. At the moment of writing, project Loom comes with two implementations of the `StructuredTaskScope<T>` type: `java.util.concurrent.StructuredTaskScope.ShutdownOnFailure` and `java.util.concurrent.StructuredTaskScope.ShutdownOnSuccess`

Let's start with analyzing the first one. We'll do using it directly into our example and see what happens. Here is the code:

```java
@Override
public GitHubUser findGitHubUser(UserId userId)
    throws ExecutionException, InterruptedException {
  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var user = scope.fork(() -> findUserByIdPort.findUser(userId));
    var repositories = scope.fork(() -> findRepositoriesByUserIdPort.findRepositories(userId));
    scope.join();
    LOGGER.info("Both forked task completed");
    return new GitHubUser(user.get(), repositories.get());
  }
}
```

First, we execute the case in which no exception is thrown. The output is the following:

```
08:21:05.040 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
08:21:05.041 [virtual-20] INFO GitHubApp -- Finding user with id 'UserId[value=1]'
08:21:05.583 [virtual-20] INFO GitHubApp -- User 'UserId[value=1]' found
08:21:06.087 [virtual-22] INFO GitHubApp -- Repositories found for user 'UserId[value=1]'
08:21:06.108 [main] INFO GitHubApp -- Both forked task completed
08:21:06.111 [main] INFO GitHubApp -- GitHub user: GitHubUser[user=User[userId=UserId[value=1], name=UserName[value=rcardin], email=Email[value=rcardin@rockthejvm.com]], repositories=[Repository[name=raise4s, visibility=PUBLIC, uri=https://github.com/rcardin/raise4s], Repository[name=sus4s, visibility=PUBLIC, uri=https://github.com/rcardin/sus4s]]]
```

As we can see, there is nothing interesting. The behavior is the same of the execution with the `StructuredTaskScope` type as expected. Now, let's see what happens when the `findUserByIdPort.findUser(userId)` method throws an exception:

```
08:22:42.466 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
08:22:42.466 [virtual-20] INFO GitHubApp -- Finding user with id 'UserId[value=1]'
08:22:42.590 [main] INFO GitHubApp -- Both forked task completed
Exception in thread "main" java.lang.IllegalStateException: Result is unavailable or subtask did not complete successfully
	at java.base/java.util.concurrent.StructuredTaskScope$SubtaskImpl.get(StructuredTaskScope.java:940)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindGitHubUserStructuredConcurrencyService.findGitHubUser(GitHubApp.java:157)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp.main(GitHubApp.java:167)
```

Well, the output looks very promising. The second task was stopped (canceled) as soon as the first one went in error. The parent task was not stop and a `java.lang.IllegalStateException` exception was thrown when we tried to get the result from the failed computation. So, we solved one of the problems of unstructured concurrency, thread leaks and resources starvation. A good step forward.

However, the thrown exception was not the original exception thrown by the child computation in error. We completely lost the original cause of error. However, we can do better. The `StructuredTaskScope.ShutdownOnFailure` scope adds a method to the available ones, `throwIfFailed`. As the documentation said, the method throws if any of the subtasks failed. The method throws an `java.util.concurrent.ExecutionException` exception set with the original exception as its cause. If no subtask failed, the method returns normally.

We change first our code to see the new behavior in action:

```java
@Override
public GitHubUser findGitHubUser(UserId userId)
    throws ExecutionException, InterruptedException {
  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var user = scope.fork(() -> findUserByIdPort.findUser(userId));
    var repositories = scope.fork(() -> findRepositoriesByUserIdPort.findRepositories(userId));

    LOGGER.info("Both forked task completed");
    
    scope.join().throwIfFailed();
    
    return new GitHubUser(user.get(), repositories.get());
  }
}
```

If we run the code, we'll notice that the behavior is the expected one. The output is the following:

```
08:34:53.701 [main] INFO GitHubApp -- Both forked task completed
08:34:53.701 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
08:34:53.700 [virtual-20] INFO GitHubApp -- Finding user with id 'UserId[value=1]'
Exception in thread "main" java.util.concurrent.ExecutionException: java.lang.RuntimeException: Socket timeout
	at java.base/java.util.concurrent.StructuredTaskScope$ShutdownOnFailure.throwIfFailed(StructuredTaskScope.java:1324)
	at java.base/java.util.concurrent.StructuredTaskScope$ShutdownOnFailure.throwIfFailed(StructuredTaskScope.java:1301)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindGitHubUserStructuredConcurrencyService.findGitHubUser(GitHubApp.java:156)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp.main(GitHubApp.java:168)
Caused by: java.lang.RuntimeException: Socket timeout
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$GitHubRepository.findUser(GitHubApp.java:68)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindGitHubUserStructuredConcurrencyService.lambda$findGitHubUser$0(GitHubApp.java:151)
	at java.base/java.util.concurrent.StructuredTaskScope$SubtaskImpl.run(StructuredTaskScope.java:893)
	at java.base/java.lang.VirtualThread.run(VirtualThread.java:329)
```

If we want to change the type of the exception thrown by the `throwIfFailed`, the method comes with an override that takes a function as input to map the exception. Here is the code:

```java
// Java SDK
public <X extends Throwable> void throwIfFailed(Function<Throwable, ? extends X> esf) throws X
```

Let's say we want to rethrow exactly the original exception. We can do it using the identity function:

```java
@Override
public GitHubUser findGitHubUser(UserId userId)
    throws Throwable {
  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var user = scope.fork(() -> findUserByIdPort.findUser(userId));
    var repositories = scope.fork(() -> findRepositoriesByUserIdPort.findRepositories(userId));
    
    LOGGER.info("Both forked task completed");
    
    scope.join().throwIfFailed(Function.identity());
    
    return new GitHubUser(user.get(), repositories.get());
  }
}
```

If we run the code, we'll see the following output:

```
08:46:38.451 [main] INFO GitHubApp -- Both forked task completed
08:46:38.450 [virtual-20] INFO GitHubApp -- Finding user with id 'UserId[value=1]'
08:46:38.451 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
Exception in thread "main" java.lang.RuntimeException: Socket timeout
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$GitHubRepository.findUser(GitHubApp.java:70)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindGitHubUserStructuredConcurrencyService.lambda$findGitHubUser$0(GitHubApp.java:153)
	at java.base/java.util.concurrent.StructuredTaskScope$SubtaskImpl.run(StructuredTaskScope.java:893)
	at java.base/java.lang.VirtualThread.run(VirtualThread.java:329)
```

As we can see from the log, we successfully rethrow the original exception. However, there is an issue we should be aware of. The `throwIfFailed` method takes a function with a `Throwable` instance as input to `Throwable`, which means that the function will be called also for terminal errors, such as `OutOfMemoryError` or `StackOverflowError`. As a rule of thumb, we should avoid catching such errors, and let the JVM handle them. So, if we need to process the error with the `throwIfFailed` method, remember to check if the input is an instance of `Exception` or `Error` and act accordingly:

```java
scope.join().throwIfFailed(throwable -> {
  if (throwable instanceof Exception) {
    // Handle the exception
  }
  else throw (Error) throwable; 
});
```

It's easy to use the `StructuredTaskScope.ShutdownOnFailure` policy to implement an structured concurrency primitive that is available in all the library implementing some form of concurrency: We're talking about the `par` function. The `par` function takes two tasks and returns the result of both, or it stops if any of the two computation fails. For who is familiar with Scala Cats Effects or ZIO libraries, the `par` function is a common primitive. Here is the code:

```java
record Pair<T1, T2>(T1 first, T2 second) {}

static <T1, T2> Pair<T1, T2> par(Callable<T1> first, Callable<T2> second)
    throws InterruptedException, ExecutionException {
  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var firstTask = scope.fork(first);
    var secondTask = scope.fork(second);
    scope.join().throwIfFailed();
    return new Pair<>(firstTask.get(), secondTask.get());
  }
}
```

If you have noticed, the above code is exactly what we did in the `findGitHubUser` method. We can now refactor the method to use the `par` function:

```java
@Override
public GitHubUser findGitHubUser(UserId userId) 
    throws ExecutionException, InterruptedException {
  var result =
      par(
          () -> findUserByIdPort.findUser(userId),
          () -> findRepositoriesByUserIdPort.findRepositories(userId)
      );
  return new GitHubUser(result.first(), result.second());
}
```

The `StructuredTaskScope.ShutdownOnFailure` is not the only policy available. The JDK comes with another built-in policy, the `StructuredTaskScope.ShutdownOnSuccess`. The policy is similar to the `ShutdownOnFailure` one, but it stops the computation at first subtask that completes successfully. Let's build an example to analyze the function in detail.

Imagine that we noticed that retrieving all the repositories of a user is a very expensive operation. Moreover, the repositories of a user change very rarely. So, we decide to build a cached version of the `findRepositories` method. First, we need to define an implementation of the port that uses a cache to store the result of the computation. Here is the code:

```java
class FindRepositoriesByUserIdCache implements FindRepositoriesByUserIdPort {
    
  private final Map<UserId, List<Repository>> cache = new HashMap<>();

  public FindRepositoriesByUserIdCache() {
    cache.put(
          new UserId(42L),
          List.of(
              new Repository(
                  "rockthejvm.github.io",
                  Visibility.PUBLIC,
                  URI.create("https://github.com/rockthejvm/rockthejvm.github.io"))));
  }
  
  @Override
  public List<Repository> findRepositories(UserId userId) throws InterruptedException {
    // Simulates access to a distributed cache (Redis?)
    delay(Duration.ofMillis(100L));
    final List<Repository> repositories = cache.get(userId);
    if (repositories == null) {
      LOGGER.info("No cached repositories found for user with id '{}'", userId);
      throw new NoSuchElementException(
          "No cached repositories found for user with id '%s'".formatted(userId));
    }
    return repositories;
  }
  
  public void addToCache(UserId userId, List<Repository> repositories)
      throws InterruptedException {
    // Simulates access to a distributed cache (Redis?)
    delay(Duration.ofMillis(100L));
    cache.put(userId, repositories);
  }
}
```

As you can see, we're simulating a distributed cache like Redis with an in-memory map and a delay to simulate the network latency. Moreover, we're not paying attention to concurrent access to the map since the main topic of the article is not the concurrent access of data structures. Please, don't use the above code in production. Moreover, the behaviour in case the repositories are not found in the cache is a bit rude. The method throws a `NoSuchElementException` exception. However, it'll be clear in a moment why we did it.

At startup time, the only cached repositories are those of the user with `UserId(42L)`.

Now, we can implement a pimped version of our original `findRepositories` method. It'll spawn two tasks: one to retrieve the repositories from the cache and one to retrieve the repositories from the GitHub API. The first task that completes successfully will stop the computation. Here is the code:

```java
static class GitHubCachedRepository implements FindRepositoriesByUserIdPort {
    
  private final FindRepositoriesByUserIdPort repository;
  private final FindRepositoriesByUserIdCache cache;

  GitHubCachedRepository(
      FindRepositoriesByUserIdPort repository, FindRepositoriesByUserIdCache cache) {
    this.repository = repository;
    this.cache = cache;
  }
  
  @Override
  public List<Repository> findRepositories(UserId userId)
      throws InterruptedException, ExecutionException {
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<List<Repository>>()) {
      scope.fork(() -> cache.findRepositories(userId));
      scope.fork(
          () -> {
            final List<Repository> repositories = repository.findRepositories(userId);
            cache.addToCache(userId, repositories);
            return repositories;
          });
      return scope.join().result();
    }
  }
}
```

We used the `StructuredTaskScope.ShutdownOnSuccess` policy to implement our new use case. Let's spot the differences with the previous policy we used. First, the type of the scope is `StructuredTaskScope.ShutdownOnSuccess<List<Repository>>`. The type parameter is the type of the result of the computation. Then, we didn't give much attention to the `Subtask` objects returned by the `fork` method. In fact, as we can see, we call the `result` method on the `scope` object to retrieve the result of the computation. In fact, we can't know which of the two tasks completed successfully first. The `result` method returns the result of the first task that completed successfully.

Let's test the new implementation. We can use the `main` method to do it:

```java
public static void main() throws ExecutionException, InterruptedException {
  final GitHubRepository gitHubRepository = new GitHubRepository();
  final FindRepositoriesByUserIdCache cache = new FindRepositoriesByUserIdCache();
  final FindRepositoriesByUserIdPort gitHubCachedRepository =
      new GitHubCachedRepository(gitHubRepository, cache);
  final List<Repository> repositories = gitHubCachedRepository.findRepositories(new UserId(1L));
  
  LOGGER.info("GitHub user's repositories: {}", repositories);
}
```

We expect that the `cache` to throw an exception since the repositories of the user with `UserId(1L)` are not cached, and the `repository` to complete successfully the execution. As we said, the `StructuredTaskScope.ShutdownOnSuccess` scope waits for the first task that completes successfully. The output of the execution is in fact the following:

```
09:43:21.679 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
09:43:21.779 [virtual-20] INFO GitHubApp -- No cached repositories found for user with id 'UserId[value=1]'
09:43:22.702 [virtual-22] INFO GitHubApp -- Repositories found for user 'UserId[value=1]'
09:43:22.812 [main] INFO GitHubApp -- GitHub user's repositories: [Repository[name=raise4s, visibility=PUBLIC, uri=https://github.com/rcardin/raise4s], Repository[name=sus4s, visibility=PUBLIC, uri=https://github.com/rcardin/sus4s]]
```

As we can see, the `cache` task throws an exception, and the `repository` task completes successfully. The computation stops, and the result is the one we expect. 

Now, we can simulate that the `cache` completes successfully, finding the user's repositories in-memory. As you might remember, the repositories of the user with `UserId(42L` are put in cache at startup time. Let's change our `main` method to test the new scenario:

```java
public static void main() throws ExecutionException, InterruptedException {
  final GitHubRepository gitHubRepository = new GitHubRepository();
  final FindRepositoriesByUserIdCache cache = new FindRepositoriesByUserIdCache();
  final FindRepositoriesByUserIdPort gitHubCachedRepository =
          new GitHubCachedRepository(gitHubRepository, cache);

  final List<Repository> repositories = gitHubCachedRepository.findRepositories(new UserId(42L));
  LOGGER.info("GitHub user's repositories: {}", repositories);
}
```

The output of the execution is the following:

```
21:36:32.901 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=42]'
21:36:33.014 [main] INFO GitHubApp -- GitHub user's repositories: [Repository[name=rockthejvm.github.io, visibility=PUBLIC, uri=https://github.com/rockthejvm/rockthejvm.github.io]]
```

As we can see, the computation that doesn't use the cache started. However, since the computation that uses the cache completed successfully, the other computation in canceled to avoid wasting resources. We've achieved our goal. 

What if both tasks throw an exception? We can see it in action by changing the `findRepositories` of the `GitHubRepository` class to throw an exception. Here is the code:

```java
@Override
@Override
public List<Repository> findRepositories(UserId userId) throws InterruptedException {
  LOGGER.info("Finding repositories for user with id '{}'", userId);
  delay(Duration.ofSeconds(1L));
  throw new RuntimeException("Socket timeout");
}
```

Now, we need to switch the user using during the search to let the `cache` crash again. 

```java
public static void main() throws ExecutionException, InterruptedException {
  final GitHubRepository gitHubRepository = new GitHubRepository();
  final FindRepositoriesByUserIdCache cache = new FindRepositoriesByUserIdCache();
  final FindRepositoriesByUserIdPort gitHubCachedRepository =
      new GitHubCachedRepository(gitHubRepository, cache);
  final List<Repository> repositories = gitHubCachedRepository.findRepositories(new UserId(1L));
  LOGGER.info("GitHub user's repositories: {}", repositories);
}
```

There we go! Let's execute the code:

```
16:18:21.615 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
16:18:21.714 [virtual-20] INFO GitHubApp -- No cached repositories found for user with id 'UserId[value=1]'
Exception in thread "main" java.util.concurrent.ExecutionException: java.util.NoSuchElementException: No cached repositories found for user with id 'UserId[value=1]'
	at java.base/java.util.concurrent.StructuredTaskScope$ShutdownOnSuccess.result(StructuredTaskScope.java:1152)
	at java.base/java.util.concurrent.StructuredTaskScope$ShutdownOnSuccess.result(StructuredTaskScope.java:1116)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$GitHubCachedRepository.findRepositories(GitHubApp.java:104)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp.main(GitHubApp.java:257)
```

As we can see, the log reports the exception thrown by the `cache` task, the first one that was thrown. The second exception, the `RuntimeException("Socket timeout")` was caught by the `scope`, but it was suppressed. In fact, the `StructuredTaskScope.ShutdownOnSuccess` rethrows the first exception thrown if all the tasks throw an exception.

We can remap the exception thrown by the `result` method as we did for the `throwIfFailed` method. The `result` method comes with an override that takes a function as input to map the exception. Here is the code:

```java
scope.join().result(Function.identity());
```

Clearly, the same warnings or remapping the exceptions we gave for the `throwIfFailed` method apply to the `result` method.

We can use the `StructuredTaskScope.ShutdownOnSuccess` to implement another concurrency primitive present in other well-known libraries such as ZIO and Softwaremill Ox: The `raceAll` function. The `raceAll` function takes two tasks and returns the result of the first task that completes successfully. If both tasks throw an exception, the first exception thrown is rethrown. Here is how we can implement it in Java using the `StructuredTaskScope.ShutdownOnSuccess` policy:

```java
static <T> T raceAll(Callable<T> first, Callable<T> second)
    throws InterruptedException, ExecutionException {
  try (var scope = new StructuredTaskScope.ShutdownOnSuccess<T>()) {
    scope.fork(first);
    scope.fork(second);
    return scope.join().result();
  }
}
```

Let's rewrite the `GitHubCachedRepository.findRepositories` method using the `raceAll` function we just implemented:

```java
@Override
public List<Repository> findRepositories(UserId userId)
    throws InterruptedException, ExecutionException {
  return raceAll(
      () -> cache.findRepositories(userId),
      () -> {
        final List<Repository> repositories = repository.findRepositories(userId);
        cache.addToCache(userId, repositories);
        return repositories;
      });
}
```

If you're familiar with concurrency primitives, you might have asking how we can implement the `race` primitive. The `race` function should execute two tasks concurrently and return the result of the first task that completes, both if successful or not. Both ZIO and Cats Effects libraries provide such a primitive, and it is the building block to create the `timeout` function. However, the available subclasses of the `StructuredTaskScope` don't provide a way to implement the `race` function directly. We need to create our own policy to implement it.

Let's do it, and let's deepen our knowledge of structured concurrency internal mechanisms in the meantime.

## 5. Implementing a custom policy

As we said, we want to implement a structured concurrency policy that mimic the behaviour of the `race` function. The `race` function should execute two tasks concurrently and return the result of the first task that completes, both if successful or not. Let's do it one step at time. 

Implementing a custom policy always start extending the `StructuredTaskScope` type. In fact, the base class implements all the mechanisms to handle the synchronization of the forked tasks. So, first of all, we need to extend the `StructuredTaskScope` type:

```java
class ShutdownOnResult<T> extends StructuredTaskScope<T> {
    // TODO
}
```

We want our policy to return a result of type `T`, similarly to what is done by the `ShutdownOnSuccess<T>` type. Then, we made the new policy generic on `T`. The `StructuredTaskScope` is a concrete class, there is no abstract method to implement. The documentation tells us that we need to override the `handleComplete` method:

```java
static class ShutdownOnResult<T> extends StructuredTaskScope<T> {
  
  @Override
  protected void handleComplete(Subtask<? extends T> subtask) {
    // TODO
  }
}
```

The method takes a `Subtask<T>` in input. In fact, the scope calls the `handleComplete` every time a subtask forked by the scope completes. The input subtask is the one that completes.

Now, we can extract the result of the computation from the subtask. The `Subtask` type exposes two methods. The first is the `T get()` method we used so far and that returns the computed value in case of success. If you remember, the `get` methods comes from the `Supplier<T>` interface that the `Subtask<T>` implements. The second method is the `Throwable exception()`, which returns the exception thrown by the computation in case of failure.

However, as we already saw in the previous examples, we can't call `get` if the computation failed otherwise the method will throw an `IllegalStateException`. On the other hand, if the computation succeeded, we can't call the `exception` method, which will have the same behavior. 

We need a way to know if the computation completed successfully or not. The `Subtask` type comes with the method `State state()` that returns the state of the computation as an enum. The enum has three values: `UNAVAILABLE`, `SUCCESS`, and `FAILED`.

A subtask will be in the `SUCCESS` state after the computation completed successfully. The `get` method will return the result of the computation. A subtask will be in the `FAILED` state after the computation completed with an exception. The `exception` method will return the exception thrown by the computation. A subtask will be in the `UNAVAILABLE` state if the computation is not completed yet, for example if we call the `get` method before the having `join` the scope or if the subtask was canceled (we'll see in the next sections what does it means).

We need to use some state that can store the result of the first successful exception or the exception thrown by the first task that failed:

```java
static class ShutdownOnResult<T> extends StructuredTaskScope<T> {
  private T firstResult;
  private Throwable firstException;

  @Override
  protected void handleComplete(Subtask<? extends T> subtask) {
    // TODO
  }
}
```

We need to synchronize the access to the state since the `handleComplete` method is called concurrently by the forked tasks. There are a million way to do it. The only one we don't want to use is calling a `synchronized` block. [As you might remember](https://blog.rockthejvm.com/ultimate-guide-to-java-virtual-threads/#6-pinned-virtual-threads), virtual threads don't work well with `synchronized` blocks, since the carrier thread is pinned to the virtual thread and so it blocks waiting for the lock to be released. 

We can use a `ReentrantLock` to synchronize the access to the state instead, which doesn't suffer of the above problem:

```java
static class ShutdownOnResult<T> extends StructuredTaskScope<T> {
  private final Lock lock = new ReentrantLock();
  private T firstResult;
  private Throwable firstException;

  @Override
  protected void handleComplete(Subtask<? extends T> subtask) {
    // TODO
  }
```

We want to synchronize the access to the state both in reading and writing. Probably, we can optimize further the access to the mutable state, but the aim of the article is to show how to implement a custom policy for structured concurrency. So, we'll keep the code simple:

```java
@Override
protected void handleComplete(Subtask<? extends T> subtask) {
  switch (subtask.state()) {
    case FAILED -> {
      lock.lock();
      try {
        if (firstException == null) {
          firstException = subtask.exception();
          shutdown();
        }
      } finally {
        lock.unlock();
      }
    }
    case SUCCESS -> {
      lock.lock();
      try {
        if (firstResult == null) {
          firstResult = subtask.get();
          shutdown();
        }
      } finally {
        lock.unlock();
      }
    }
    case UNAVAILABLE -> super.handleComplete(subtask);
  }
}
```

As we can see, the `handleComplete` method just check the state of the given `subtask`, and set the proper variable representing our state accordingly. After that, the method calls the `shutdown` method. The `shutdown` method is a method of the `StructuredTaskScope` type that stops all the pending subtasks forked by the scope. We will see how it works in detail in the next sections.

Now that we have the first result or the first exception thrown by the forked tasks, we need a way to retrieve it. We can add a method to the `ShutdownOnResult` type that returns the result of the computation or throws the exception thrown by the computation. The method looks like a mix between the `ShutdownOnFailure.throwIfFailed` and the `ShutdownOnSuccess.result` methods, and we called it `resultOrThrow`:

```java
public T resultOrThrow() throws ExecutionException {
  if (firstException != null) {
    throw new ExecutionException(firstException);
  }
  return firstResult;
}
```

Finally, the complete code of the `ShutdownOnResult` policy is the following:

```java
static class ShutdownOnResult<T> extends StructuredTaskScope<T> {
  private final Lock lock = new ReentrantLock();
  private T firstResult;
  private Throwable firstException;
  
  @Override
  protected void handleComplete(Subtask<? extends T> subtask) {
    switch (subtask.state()) {
      case FAILED -> {
        lock.lock();
        try {
          if (firstException == null) {
            firstException = subtask.exception();
            shutdown();
          }
        } finally {
          lock.unlock();
        }
      }
      case SUCCESS -> {
        lock.lock();
        try {
          if (firstResult == null) {
            firstResult = subtask.get();
            shutdown();
          }
        } finally {
          lock.unlock();
        }
      }
      case UNAVAILABLE -> super.handleComplete(subtask);
    }
  }
  
  @Override
  public ShutdownOnResult<T> join() throws InterruptedException {
    super.join();
    return this;
  }
  
  public T resultOrThrow() throws ExecutionException {
    if (firstException != null) {
      throw new ExecutionException(firstException);
    }
    return firstResult;
  }
}
```

Let's try our new policy in action. For example, say that we want to retrieve the repositories of a user within a given interval or give up otherwise. Obviously, we want the computation to be properly shutdown in both cases, as we did in all the previous examples. We can use the new and shiny `ShutdownOnResult` policy to implement the use case:

```java
static class FindRepositoriesByUserIdWithTimeout {
  final FindRepositoriesByUserIdPort delegate;
  FindRepositoriesByUserIdWithTimeout(FindRepositoriesByUserIdPort delegate) {
    this.delegate = delegate;
  }
  List<Repository> findRepositories(UserId userId, Duration timeout)
      throws InterruptedException, ExecutionException {
    try (var scope = new ShutdownOnResult<List<Repository>>()) {
      scope.fork(() -> delegate.findRepositories(userId));
      scope.fork(
          () -> {
            delay(timeout);
            throw new TimeoutException("Timeout of %s reached".formatted(timeout));
          });
      return scope.join().resultOrThrow();
    }
  }
}
```

We can now see the effect of the timeout trying to get the repositories of a user within 500 milliseconds:

```java
public static void main() throws ExecutionException, InterruptedException {
  final GitHubRepository gitHubRepository = new GitHubRepository();
  final FindRepositoriesByUserIdWithTimeout findRepositoriesWithTimeout =
      new FindRepositoriesByUserIdWithTimeout(gitHubRepository);
  final List<Repository> repositories =
      findRepositoriesWithTimeout.findRepositories(new UserId(1L), Duration.ofMillis(500L));
  LOGGER.info("GitHub user's repositories: {}", repositories);
}
```

As we might have expected, the output of the execution is the following, saying that the computation retrieving the repositories started but couldn't complete within the given interval, and it was canceled.

```
09:13:08.611 [virtual-20] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
Exception in thread "main" java.util.concurrent.ExecutionException: java.util.concurrent.TimeoutException: Timeout of PT0.5S reached
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$ShutdownOnResult.resultOrThrow(GitHubApp.java:334)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindRepositoriesByUserIdWithTimeout.findRepositories(GitHubApp.java:64)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp.main(GitHubApp.java:365)
Caused by: java.util.concurrent.TimeoutException: Timeout of PT0.5S reached
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindRepositoriesByUserIdWithTimeout.lambda$findRepositories$1(GitHubApp.java:62)
	at java.base/java.util.concurrent.StructuredTaskScope$SubtaskImpl.run(StructuredTaskScope.java:893)
	at java.base/java.lang.VirtualThread.run(VirtualThread.java:329)
```

Whereas, if we increase the timeout to 1.5 seconds, the computation retrieves the repositories successfully:

```
09:15:42.083 [virtual-20] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
09:15:43.100 [virtual-20] INFO GitHubApp -- Repositories found for user 'UserId[value=1]'
09:15:43.122 [main] INFO GitHubApp -- GitHub user's repositories: [Repository[name=raise4s, visibility=PUBLIC, uri=https://github.com/rcardin/raise4s], Repository[name=sus4s, visibility=PUBLIC, uri=https://github.com/rcardin/sus4s]]
```

Moreover, at the moment, we have all the building blocks to develop our `race` function, which returns the result of the first task that completes, both if successful or not. Here is the code:

```java
static <T> T race(Callable<T> first, Callable<T> second)
    throws InterruptedException, ExecutionException {
  try (var scope = new ShutdownOnResult<T>()) {
    scope.fork(first);
    scope.fork(second);
    return scope.join().resultOrThrow();
  }
}
```

Once we have the `race` function, we can use it to implement the `timeout` function. The `timeout` function takes a task and a duration as input. The function returns the result of the task if the task completes within the given duration. If the task doesn't complete within the given duration, the function throws a `TimeoutException` exception. Here is the code:

```java
static <T> T timeout(Duration timeout, Callable<T> task)
    throws InterruptedException, ExecutionException {
  return race(
      task,
      () -> {
        delay(timeout);
        throw new TimeoutException("Timeout of %s reached".formatted(timeout));
      });
}
```

We can rewrite the above example using the `timeout` function, thus avoiding the definition of the `FindRepositoriesByUserIdWithTimeout` class at all:

```java
public static void main() throws ExecutionException, InterruptedException {
  final GitHubRepository gitHubRepository = new GitHubRepository();
  final List<Repository> repositories =
      timeout(Duration.ofMillis(500L), () -> gitHubRepository.findRepositories(new UserId(1L)));
  
  LOGGER.info("GitHub user's repositories: {}", repositories);
}
```

Building up a function that implements a timeout on concurrent computation was fun, and we learnt a lot during the process. However, to be fair, the `StructuredTaskScope` type already implement it. In fact, there is an override of the `join` function that takes a `Instant` as input:

```java
public StructuredTaskScope<T> joinUntil(Instant deadline)
```

The function waits for the scope to complete for the given duration. If the scope doesn't complete within the given duration, the function throws a `TimeoutException` exception. Then, we can rewrite the `timeout` function just using the `ShutdownOnFailure` policy and the `joinUntil` method:

```java
static <T> T timeout2(Duration timeout, Callable<T> task)
        throws InterruptedException, TimeoutException {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        var result = scope.fork(task);
        scope.joinUntil(Instant.now().plus(timeout));
        return result.get();
    }
}
```

By the way, we were able to implement also the `race` function through the `ShutdownOnResult` policy. So, it was not wasted time after all.

## 6. Cancelling a computation

We saw in the previous section that the `StructuredTaskScope` type comes with a `shutdown` method. We called the method in the `handleComplete` method of the `ShutdownOnResult` policy to stop the computation as soon as the first task completed successfully or failed. The `shutdown` method stops all the pending subtasks forked by the scope. 

All the available policies of the `StructuredTaskScope` calls the `shutdown` method in case of error or success. The `ShudownOnFailure` policy, as the name suggests, calls the `shutdown` method in case of error of one of the forked computation:

```java
// Java SDK
@Override
protected void handleComplete(Subtask<?> subtask) {
    if (subtask.state() == Subtask.State.FAILED
            && firstException == null
            && FIRST_EXCEPTION.compareAndSet(this, null, subtask.exception())) {
        super.shutdown();
    }
}
```

As we did for our `ShutdownOnResult` implementation, the `shutdown` method is called in the `handleComplete` method, which means when a forked task completes. In the above case, the shutdown policy testes if an exception was thrown by the computation, and in case it was the first exception thrown, it calls the `shutdown` method.

The `ShutdownOnSuccess` policy works in a similar way. It calls the `shutdown` method when it gets the first successful result from one of the forked tasks:

```java
// Java SDK
@Override
protected void handleComplete(Subtask<? extends T> subtask) {
    if (firstResult != null) {
        // already captured a result
        return;
    }
    if (subtask.state() == Subtask.State.SUCCESS) {
        // task succeeded
        T result = subtask.get();
        Object r = (result != null) ? result : RESULT_NULL;
        if (FIRST_RESULT.compareAndSet(this, null, r)) {
            super.shutdown();
        }
    } else if (firstException == null) {
        // capture the exception thrown by the first subtask that failed
        FIRST_EXCEPTION.compareAndSet(this, null, subtask.exception());
    }
}
```

As you should remember, failures accumulates and don't force the scope to shut down.

Last but not least, if the policy haven't called the `shutdown` method and the execution of the scope completes, the `shutdown` method is called by the `close` method:

```java
// Java SDK
@Override
public void close() {
    ensureOwner();
    int s = state;
    if (s == CLOSED)
        return;
    try {
        if (s < SHUTDOWN)
            implShutdown();
        flock.close();
    } finally {
        state = CLOSED;
    }
    // throw ISE if the owner didn't attempt to join after forking
    if (forkRound > lastJoinAttempted) {
        lastJoinCompleted = forkRound;
        throw newIllegalStateExceptionNoJoin();
    }
}
```

Despite a lot of implementation details, the above code says that if the scope is still open when the `close` method is called, the `shutdown` method is called. 

As you can see, also the scope has an internal status that is checked to see if the scope is still open or not. The possible statuses are:

```java
// Java SDK
// states: OPEN -> SHUTDOWN -> CLOSED
private static final int OPEN     = 0;   // initial state
private static final int SHUTDOWN = 1;
private static final int CLOSED   = 2;

private volatile int state;
```

A scope starts in the `OPEN` state at creation time since the variable that store the `state` is an `int` and `int`s initial value is always set to zero by the JVM. Then, if the `shutdown` method is called, the scope moves to the `SHUTDOWN` state:

```java
// Java SDK
public void shutdown() {
    ensureOwnerOrContainsThread();
    int s = ensureOpen();  // throws ISE if closed
    if (s < SHUTDOWN && implShutdown())
        flock.wakeup();
}

private boolean implShutdown() {
    shutdownLock.lock();
    try {
        if (state < SHUTDOWN) {
            // prevent new threads from starting
            flock.shutdown();
            // set status before interrupting tasks
            state = SHUTDOWN;
            // interrupt all unfinished threads
            interruptAll();
            return true;
        } else {
            // already shutdown
            return false;
        }
    } finally {
        shutdownLock.unlock();
    }
}
```

Finally, the `close()` method move the state of the scope to `CLOSE`, as we saw. The curious reader should have noticed that the scope uses `jdk.internal.misc.ThreadFlock` to manage threads forked by a scope. `ThreadFlock` are a low-level mechanism to manage correlated threads in the JDK. It's to low-level that is even hard to find documentation associated with them.

By the way, we said that calling the `shutdown` method stops all the pending subtasks forked by the scope. The above implementation shows how the method stops the task, calling the `interruptAll` private method. Here is its implementation of the core of the method:

```java
// Java SDK
private void implInterruptAll() {
    flock.threads()
            .filter(t -> t != Thread.currentThread())
            .forEach(t -> {
                try {
                    t.interrupt();
                } catch (Throwable ignore) { }
            });
}
```

As we can see, there is no magic under the stop of uncompleted computation. The (virtual) threads owning the tasks are just interrupted by the scope. As you might remember, [interruption (or cancelling) is a cooperative mechanism in Java](https://rockthejvm.com/articles/the-ultimate-guide-to-java-virtual-threads/#the-scheduler-and-cooperative-scheduling). A thread is eligible for interruption if it calls a method that throws an `InterruptedException` exception or if it checks the interruption status of the thread. Luckily, almost any blocking operation in the JDK can be interrupted, so the interruption mechanism works well in the JDK. However, it's easy to create a computation that can't be interrupted when dealing with CPU-intensive tasks.

Let's make an example. Imagine that we want to mine bitcoins while we're waiting for the repositories of a user. We can implement the `mineBitcoins` method as follows:

```java
record Bitcoin(String hash) {}

static Bitcoin mineBitcoin() {
  LOGGER.info("Mining Bitcoin...");
  while (alwaysTrue()) {
    // Empty body
  }
  LOGGER.info("Bitcoin mined!");

  return new Bitcoin("bitcoin-hash");
}
private static boolean alwaysTrue() {
```

Now, we can make it race with a thread that retrieves the repositories of a user:

```java
public static void main() throws ExecutionException, InterruptedException {
  final GitHubRepository gitHubRepository = new GitHubRepository();
  
  var repositories = race(
          () -> gitHubRepository.findRepositories(new UserId(42L)), 
          () -> mineBitcoin()
  );
  
  LOGGER.info("GitHub user's repositories: {}", repositories);
}
```

We expect to get the repositories of the user before the bitcoin is mined and see the `race` function to interrupt the `mineBitcoin` computation. We already tested the `race` function, and we know it works as expected. However, the output of the execution is the following:

```
08:49:09.118 [virtual-22] INFO GitHubApp -- Mining Bitcoin...
08:49:09.118 [virtual-20] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=42]'
08:49:10.135 [virtual-20] INFO GitHubApp -- Repositories found for user 'UserId[value=42]'
(infinite waiting)
```

The `main` method executes forever, despite the `race` function should interrupt the `mineBitcoin` computation and all the good promises of structured concurrency. The problem is that the `mineBitcoin` computation doesn't have a  check point for the interruption status of its thread. Since interruption is a cooperative mechanism, the `mineBitcoin` computation doesn't know that the scope wants to stop it.

It's easy to fix the problem. We can add a check point in the `mineBitcoin` computation to check if the thread was interrupted. We can check if a thread was interrupted with the `isInterrupted` method on the `Thread` class:

```java
static Bitcoin mineBitcoinWithConsciousness() {
  LOGGER.info("Mining Bitcoin...");
  while (alwaysTrue()) {
    if (Thread.currentThread().isInterrupted()) {
      LOGGER.info("Bitcoin mining interrupted");
      return null;
    }
  }
  LOGGER.info("Bitcoin mined!");
  return new Bitcoin("bitcoin-hash");
}
```

Now, we can retry the `race` function with the `mineBitcoinWithConsciousness` computation:

```java
public static void main() throws ExecutionException, InterruptedException {
  final GitHubRepository gitHubRepository = new GitHubRepository();\
    
  var repositories = race(
          () -> gitHubRepository.findRepositories(new UserId(42L)),
          () -> mineBitcoinWithConsciousness()
  );
  
  LOGGER.info("GitHub user's repositories: {}", repositories);
}
```

If we run the code again, we can see from the output that the `mineBitcoinWithConsciousness` computation is interrupted after the user's repositories are retrieved and the `race` function semantic is respected:

```
09:02:10.116 [virtual-22] INFO GitHubApp -- Mining Bitcoin...
09:02:10.133 [virtual-20] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=42]'
09:02:11.156 [virtual-20] INFO GitHubApp -- Repositories found for user 'UserId[value=42]'
09:02:11.164 [virtual-22] INFO GitHubApp -- Bitcoin mining interrupted
09:02:11.165 [main] INFO GitHubApp -- GitHub user's repositories: [Repository[name=raise4s, visibility=PUBLIC, uri=https://github.com/rcardin/raise4s], Repository[name=sus4s, visibility=PUBLIC, uri=https://github.com/rcardin/sus4s]]
```

Calling the `shutdown` method prevents also the scope from forking new tasks. The `fork` method will not simply start any new task if the scope is in the `SHUTDOWN` state. Let's try it with a simple example:

```java
public static void main() throws ExecutionException, InterruptedException {
  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    scope.shutdown();
    scope.fork(() -> {
      LOGGER.info("Hello, structured concurrency!");
      return null;
    });
    scope.join().throwIfFailed();
  }
  LOGGER.info("Completed");
}
```

The execution of the above code will never print the message "Hello, structured concurrency!" since the scope is in the `SHUTDOWN` state when the `fork` method is called.