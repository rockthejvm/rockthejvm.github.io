---
title: "ZIO Fullstack Webapp"
date: 2024-07-05
header:
  image: "https://res.cloudinary.com/dkoypjlgr/image/upload/f_auto,q_auto:good,c_auto,w_1200,h_300,g_auto,fl_progressive/v1715952116/blog_cover_large_phe6ch.jpg"
tags: [scala, zio, laminar, tapir]
excerpt: "Full stack Scala webapp with ZIO, Laminar, and Tapir"
toc: true
toc_label: "In this article"
---

# Full stack ZIO Scala WebApp

Leverage Scala / ScalaJS with ZIO and Laminar to build reactive webapp.

Build a full stack webapp application using the same language has been a never ending quest. 

In this journey, dynamically typed languages and their loosely compilation guarantees were always seen as untouchable because of:

* easy setup
* isomorphic by nature, no need for translation between back and front payload
* quick feedback loop, "just hit reload" 

On the opposite side, statically typed languages are perceived with somewhat good reason as pachyderm on a wire, where static code guarantees were eclipsed by:
* convoluted setup
* polyglot applications, and payload conversion (eg: Scala/Java -> Json)
* no/slow feedback during development.

In this article, I will demonstrate you how my ❤ Scala ecosystem changed the game.

Without compromising type safety guarantees of Scala and in the meantime being able to embed library from JavaScript/TypeScript ecosystem.

## 0. Prerequisites:

* **Sbt**: The build tool
* **NodeJS**: The JavaScript runtime
* **Docker**: For the database
* **VSCode**: The editor of choice


## 1. What is inside

* **ZIO**: Functional Effect System on both back and front side.
  * Quill for database access.
* **Tapir**: Providing and giving access to a Rest API in a type safe and incredible  elegant way.
* **Laminar**: "Simple, expressive, and safe UI library for Scala.js"


The approach is to build a full stack webapp with the same language, Scala, on both the client and the server side.

This is a very powerful combination, that allows to build a full stack webapp in a type safe manner. And in the mean time we can leverage the JavaScript ecosystem, without compromising type safety, architecture and scalability of the application.


## 2. Setup


Within a single repository, we will setup a multi-modules application using:

* **SBT**, Scala build tool
* **NPM**, Node Package manager
* **Vite**, development and build tool. 

For the very impatient, I've cooked a [g8](https://www.foundweekends.org/giter8/scaffolding.html) template 👇:

```bash
sbt new cheleb/zio-scalajs-laminar.g8 --name=zio-laminar-demo
```

This setup is a little complex because it handles:

* Incremental compilation both client and server side.
* Hot reload of UI
* Production packaging

If you are a vscode / metals user, you might be lucky:
  
```bash
  code zio-laminar-demo
```

Leveraging the [tasks vscode](https://code.visualstudio.com/docs/editor/tasks) support, and  after few downloads, your new full stack development environment should look like:

![tasks](../images/zio-fullstack/tasks.png)

4 processes are running here:

* Server side:
  * **docker**: Start a Postgres database in a docker container.
  * **serverRun**: http://localhost:8080/docs/ is the Swagger of the API, hot reloading too.
* Client side:
  * **fastlink**: Scala to JS compilation/transpilation 
  * **npmDev**: http://localhost:5173/ now serves your client web app in development mode, hot reloading with Vite. 

## 3. The build

I won't dive too much into details, see comment in the project for details.

### 3.1. Scala/ScalaJS world 

Scala build here is driven by SBT in a rather classic way, except for the `shared` project that will be cast in two different worlds `.jvm` and `.js`
Generated build contains documentation on details. Please take a look and don't hesitate to ask question/provide PR on the g8.

For **ScalaJS**

* "org.scala-js"  % "[sbt-scalajs](http://www.scala-js.org/doc/sbt-plugin.html)" % "1.16.0" 
* "ch.epfl.scala" % "[sbt-scalajs-bundler](https://scalacenter.github.io/scalajs-bundler/index.html)" % "0.21.1" will bundle the app
* "ch.epfl.scala" % "[sbt-web-scalajs-bundler](https://scalacenter.github.io/scalajs-bundler/getting-started.html#sbt-web)" % "0.21.1" will expose the bundle as classpath resource for sbt-web (websjar).

### 3.2. Cross building
"org.portable-scala" % "[sbt-scalajs-crossproject](https://github.com/portable-scala/sbt-crossproject)" % "1.3.2"

This is the plugin that will allow to spread project resources between JS and JVM worlds.
Shared project configuration 👇

* ![Scala](../images/zio-fullstack/scala.png) Server project aka Scala JVM depends `share.jvm` variant
* ![ScalaJS](../images/zio-fullstack/scalajs.png) Client project aka Scala JS depends on `shared.js` variant

ScalaJS and ScalaJS bundler are smart enough to crosscompile and deploy `module/shared/src/main/scala` sources in both target.

### 3.3. JS world

JS integration is also a classical one with NPM and Vite fellow, here a again specific configuration is limited to vite.config.js

```javascript
import { defineConfig } from "vite";
import scalaJSPlugin from "@scala-js/vite-plugin-scalajs";

export default defineConfig({
    plugins: [scalaJSPlugin({
        // path to the directory containing the sbt build
        // default: '.'
        cwd: '../..',

        // sbt project ID from within the sbt build to get fast/fullLinkJS from
        // default: the root project of the sbt build
        projectID: 'client',

        // URI prefix of imports that this plugin catches (without the trailing ':')
        // default: 'scalajs' (so the plugin recognizes URIs starting with 'scalajs:')
        uriPrefix: 'scalajs',
    })],
    build: {
        sourcemap: true,
    },
    // base: "/zio-laminar-demo",

});
```

Where most of the work is achieved by `@scala-js/vite-plugin-scalajs` plugin.

This setting will allow Vite to hot reload changes on the fly, giving an incredible developer experience.

# 4. Project structure

Thanks to SBT power and multi module support, it easy to compile and package all together. 

<table>
  <tr>
    <td>
     <img src="../images/zio-fullstack/layout.png" />
   </td>
   <td>
    <strong>Client Module</strong>, UI:
   <ul>
    <li>Laminar is the reactive UI</li>
    <li>Sttp client leverages the same Tapir endoint definitions</li>
   </ul>
  <strong>Server</strong> Module, ZIO HTTP: 
  <ul> 
    <li>Tapir ServerEndpoint from shared endpoint definitions.</li>
    <li>Swagger exposition gracefully provided by Tapir http://localhost:8080/docs/</li>
    <li>Prometheus metrics http://localhost:8080/metrics/</li>
  </ul>
  <strong>Shared</strong> module:
    <ul>
      <li>model with Json support</li>
      <li>API endpoint definitions</li>
    </ul>
  </td>
  </tr>
</table>


Also, with SBT it is easy to [generate some static files](https://github.com/cheleb/zio-laminar-demo/blob/master/build.sbt#L63) that are copied in the resources of some module. Here I use [sbt-twirl](https://github.com/playframework/twirl) to generate the index.html of the server (bff) module.



Let's see how ZIO, Laminar and Tapir can be combined to build a full stack webapp.

# 5. Server side

From now we will focus on the server side, but the same principles will apply to the client side. 

Bottom up, we will see how ZIO will combine:
* data access
* services
* Rest API 

## 5.1. ZIO

A library for asynchronous and concurrent programming in Scala.

At the type level **ZIO** is a data type that represents an effectful program that:
* either
  * produce a value of type `A`
  * fail with an error of type `E`
* may require an element of type `R` to execute.

Sometime an oversimplified mental model is to represent ZIO as:
  
```scala
  type ZIO[R, E, A] = R => Either[E, A]
```

But ZIO is much more than that, it is a full fledged library that provides a lot of utilities to handle effects, blocking, concurrency, retry, and so on.

Very important, ZIO is a combinaison of monad, and as such it provides a lot of combinators to handle effects, errors and dependencies.

In a glance ZIO[–R,+E,+A] smart variance annotations allow a very convenient constructions like our Server side application:

```scala
object HttpServer extends ZIOAppDefault {

  val runMigrations: ZIO[FlywayService, Throwable, Unit] = [...]              // (1)
  
  val server: ZIO[PersonService & Server, Nothing, Unit] = [...]              // (2)
  
  val program: ZIO[FlywayService & PersonService & Server, Throwable, Unit] = // (4)
    for {
      _ <- runMigrations                                                      // (3)
      _ <- server
    } yield ()

  override def run =
    program
      .provide( // (5)
         Server.default,
        // Service layers
        PersonServiceLive.layer,
        FlywayServiceLive.configuredLayer,
        // Repository layers
        UserRepositoryLive.layer,
        Repository.dataLayer
        // Uncomment to get a mermaid diagram of the dependencies is compilation logs.
        //,       ZLayer.Debug.mermaid 
      )
}
```

* Two ZIO effects:
  * **(1)** `runMigrations` is a ZIO[**FlywayService**, …] that depends on a FlywayService, and that will handle the database migrations.
  * **(2)** `server` is a ZIO[**PersonService & Server**, …] that depends on a PersonService and a Server and that will handle the HTTP server.
* Those two effects combine in a for comprehension **(3)** to produce a new ZIO **(4)** that accumulate the dependencies:

 ZIO[**FlywayService & PersonService & Server**, …]

To be runnable those dependencies need to be satisfied **(5)**, this is what layers are made for.



Let see how ZIO, Layer per layer assembles the application.

### 5.1.1. Repository data layer

Repository, the layer that abstract the database access, is the first layer of the application.

In this project, classic Repository object connects the database through classical JDBC DataSource.


```scala

object Repository {

  def quillLayer: URLayer[DataSource, Postgres[SnakeCase.type]] =
                                    Quill.Postgres.fromNamingStrategy(SnakeCase) // (1)

  private def datasourceLayer: TaskLayer[DataSource] =
                                    Quill.DataSource.fromPrefix("db")            // (2)

  def dataLayer: TaskLayer[Postgres[SnakeCase.type]] =
                                    datasourceLayer >>> quillLayer               // (3)

}
```

* **(1)** Quill layer, that will provide a Quill Postgres service, with a SnakeCase naming strategy for the table mapping.
> **URLayer** is a type alias for a ZLayer that does require a dependency and cannot fail:
>
>  type **URLayer**[-R, +A] = ZLayer[R, **Nothing**, A]

* **(2)** DataSource layer, that will provide a DataSource service from the configuration.
> TaskLayer is a type alias for a ZLayer that does not require a dependency and can fail with a Throwable:
>
>  type **TaskLayer**[+A] = ZLayer[Any, **Throwable**, A]

* **(3)** Data layer, that will provide a Quill Postgres service from the DataSource service.

ZIO is rich of type alias that after a while will make the code very readable.

### 5.1.2. User Repository 

User Repository is a classic repository that will handle the User entity.



```scala
trait UserRepository:                            // (1)
  def create(user: User): Task[User]
  def getById(id: Long): Task[Option[User]]
 
                                                 // (2)
class UserRepositoryLive private (quill: Quill.Postgres[SnakeCase]) extends UserRepository {

  import quill.*

  inline given SchemaMeta[User] = schemaMeta[User]("users")     // (3)
  inline given InsertMeta[User] = insertMeta[User](_.id)
  inline given UpdateMeta[User] = updateMeta[User](_.id, _.creationDate)

  override def create(user: User): Task[User] =
    run(query[User].insertValue(lift(user)).returning(r => r)) // (4a)
  override def getById(id: Long): Task[Option[User]] =         // (4b)
    run(query[User].filter(_.id == lift(Option(id)))).map(_.headOption)
}

object UserRepositoryLive: // (5)                                        (6)
  def layer: RLayer[Postgres[SnakeCase], UserRepositoryLive] = ZLayer.derive[UserRepositoryLive]

```

* **(1)** Repository trait
* **(2)** Repository implementation, with Quill Postgres injection
* **(3)** Quill Meta, to map the User case class to the users table
* **(4)** Quill queries
* **(5)** Repository layer
* **(6)** One of the many ZIO macro that will generate the layer for you, from the constructor of the class (2).


💡 Note how `ZLayer.derive[UserRepositoryLive]` is used to generate the layer from the constructor of the class, and how this layer is then depends on the `Postgres[SnakeCase]`.


Quill is a compile-time query language for Scala and SQL. It allows to write queries in Scala and compile them to SQL.


Quill compiles **(4a)** and **(4b)** to SQL like this:


```sql
INSERT INTO users (name,email,age,creation_date) VALUES (?, ?, ?, ?) RETURNING id, name, email, age, creation_date;
SELECT x4.id, x4.name, x4.email, x4.age, x4.creation_date AS creationDate FROM users x4 WHERE x4.id IS NULL AND ? IS NULL OR x4.id = ?;
```


### 5.1.3. Flyway Service

In the same way, Flyway service is a layer that will handle the database migrations.

```scala
object FlywayServiceLive {
  def live: RLayer[FlywayConfig, FlywayService] = ZLayer( // (1)
    for {
      config <- ZIO.service[FlywayConfig]
      flyway <- ZIO.attempt(Flyway.configure()
                   .dataSource(config.url, config.user, config.password).load())
    } yield new FlywayServiceLive(flyway)
  )

  val configuredLayer: TaskLayer[FlywayService] = Configs.makeConfigLayer[FlywayConfig]("db.dataSource") 
                                                             >>> live  // (2)
}
```
  
* **(1)** is a layer that will provide a **FlywayService**, it depends on the availability of **FlywayConfig** service.
* **(2)** Inject a **FlywayConfig** in a the previous layer and the remove the dependency on the config.

### 5.1.4. Person Service

Person service is a layer that will handle the business logic of the application.

```scala
trait PersonService {             // Service trait
  def register(person: Person): Task[User]
}
                                  // Dependency injection
class PersonServiceLive private (userRepository: UserRepository) extends PersonService {
  def register(person: Person): Task[User] =
    userRepository.create(
      User(
        id = None,
        name = person.name,
        email = person.email,
        age = person.age,
        creationDate = ZonedDateTime.now()
      )
    )
}

object PersonServiceLive {           // Layer derivation (macro) from the constructor
  val layer: RLayer[UserRepository, PersonService] = ZLayer.derive[PersonServiceLive]
}
```


### 5.1.5. Server layer

Server layer is a layer that will handle the HTTP server, is provided by ZIO Http with convenient default configuration.

It is a dependency that will be provided to the server effect.

```scala
private val server: ZIO[PersonService & Server, IOException, Unit] =
    for {
      _         <- Console.printLine("Starting server...")
      endpoints <- HttpApi.endpointsZIO                // (1)
      docEndpoints = SwaggerInterpreter()              // (2)
                       .fromServerEndpoints(endpoints, "zio-laminar-demo", "1.0.0")
      _ <- Server.serve(                               // (3)
             ZioHttpInterpreter(serverOptions)
               .toHttp(metricsEndpoint :: webJarRoutes :: endpoints ::: docEndpoints)
           )
    } yield ()
```

This is a ZIO effect that will:

* **(1)** gather the endpoints from the HttpApi
* **(2)** generate the documentation endpoints from the endpoints
* **(3)** start the server, and serve the endpoints.

This endpoints are a combination of the metrics endpoint, the webjar routes, the API endpoints and the documentation endpoints.


```scala
object HttpApi {
  private def gatherRoutes(
    controllers: List[BaseController]
  ): List[ServerEndpoint[Any, Task]] =
    controllers.flatMap(_.routes)

  private def makeControllers: URIO[PersonService, List[BaseController]] = for {
    healthController <- HealthController.makeZIO
    personController <- PersonController.makeZIO
  } yield List(healthController, personController)

  val endpointsZIO: URIO[PersonService, List[ServerEndpoint[Any, Task]]] = makeControllers.map(gatherRoutes)
}

```

And last part of puzzle, PersonController that will provide the API endpoints.

As it leverages Tapir, see next section [Tapir server side](#62-server-side) for more details.


In the meantime, enjoy the Mermaid [diagram of the dependencies](https://mermaid-js.github.io/mermaid-live-editor/edit/#eyJjb2RlIjoiZ3JhcGggQlRcbkwwKFwiU2VydmVyLmRlZmF1bHRcIilcbkwxKFwiUmVwb3NpdG9yeS5kYXRhTGF5ZXJcIikgLS0+IEwyKFwiVXNlclJlcG9zaXRvcnlMaXZlLmxheWVyXCIpXG5MMVxuTDIgLS0+IEwzKFwiUGVyc29uU2VydmljZUxpdmUubGF5ZXJcIilcbkw0KFwiRmx5d2F5U2VydmljZUxpdmUuY29uZmlndXJlZExheWVyXCIpIiwibWVybWFpZCI6ICJ7XCJ0aGVtZVwiOiBcImRlZmF1bHRcIn0ifQ==) compiled by ZIO.


## 6. Tapir

Tapir is a library for defining API endpoint as immutable data structures.

This is a huge win, because it allows to:

* define the API once
* generate the client and server side
* generate the documentation
* generate the tests

Here is an example of Tapir endpoints definition:

### 6.1. Shared Endpoint definitions

In the shared module, the API endpoints are defined as immutable data structures.

```scala
object PersonEndpoints {
  val get Endpoint[Unit, String, Person, Any] = endpoint.get
    .in("person")
    .out(jsonBody[Person])
    .errorOut(stringBody)
    .summary("Get a person by id")

  val create: Endpoint[Unit, Person, Throwable, User, Any] = baseEndpoint
    .tag("person")
    .name("person")
    .post
    .in("person")
    .in(
      jsonBody[Person]
        .description("Person to create")
        .example(Person("John", 30, Left(Cat("Fluffy"))))
    )
    .out(jsonBody[User])
    .description("Create person")

}
```

This is a simple GET endpoint that will return a Person and a POST endpoint that will create a Person and return a User.

From this definition, Tapir will help to generate the client and server side code, the documentation and the tests.


### 6.2. Server side

On **server side**, Tapir Endpoints.create will be implemented by a ZIO effect, through the **zServerLogic** combinator.

```scala
                                           // (1)
class PersonController private (personService: PersonService) extends BaseController {
  [...]

  val create: ServerEndpoint[Any, Task] = PersonEndpoint.createEndpoint
    .zServerLogic( // (2)
      p => personService.register(p)
    )

  val routes: List[ServerEndpoint[Any, Task]] =
    List(create)
}

object PersonController {
  def makeZIO: URIO[PersonService, PersonController] = // (3)
    ZIO.service[PersonService].map(new PersonController(_))
}

```

* **(1)** Dependency Injection.
* **(2)** Controller implementation, where json codec are transparently handled.
* **(3)** Service instantiation:  URIO[PersonService, PersonController]


Everything is IO, URIO[PersonService, PersonController] is an IO that needs a PersonService and will produce a PersonController and cannot fail.

> **URIO** is a type alias for a ZIO that cannot fail:
>
>   type **URIO**[R, A] = ZIO[R, **Nothing**, A]`

**zServerLogic** is part of the Tapir ZIO support, and bridges a ZIO in the Rest API.

**zServerLogic** is a combinator that will transform a ZIO effect into a server logic, it will handle the error and the success case.


### 6.3. Client side

On **client side**, Tapir will allow to interact as client, il will leverage the same endpoint definition to generate the client side code.

It will smartly generate the client side code, that will be used in the Laminar UI.


## 7. Laminar

Laminar is a 100% Scala reactive UI library. In few word, Laminar provide reactive variable:

* **Signal**, to watch an unique element
* **EventStream**, to consume stream of events
* **Var**: to watch and update an element
* **EventBus**: to consumes and produces events.

Laminar also provide Dom manipulation facilities, and bindings to WebComponent libraries (UI5 in this example).


## 7.1. ZIO, Tapir and Laminar

On **client side**, Tapir will allow to interact as client, is a very straigh forward way: 

```scala
 Button(
  "-- Post -->",
  onClick --> { _ => // (1)
    PersonEndpoint // (2)
      .createEndpoint(personVar.now()) // (2a andThen 2b)
      .emitTo(userBus) // (3a and 3b)
  }
)
```


* **(1)** from an Laminar UI event
* **(2)** a HTTP endpoint
  * **(2a)** reads the current value of a Laminar Var
  * **(2b)** turns into ZIO effect to handle the HTTP request
* **(3)** this effect
  * **(3a)** this effect will run
  * **(3B)** its result is consumed by the Laminar UI


Whoah, this is a lot of magic in a single line.

In the same way that on server side **zServerLogic** bridges ZIO in Tapir through an extension method, on client side two extension methods are used:
* From laminar to ZIO
* From ZIO to Laminar back.

This is the power of Scala3 extension methods, that allow to extend existing classes with new methods.

A first extension, turn this endpoint value to a function ( def **apply …** ):

```scala
extension [I, E <: Throwable, O](endpoint: Endpoint[Unit, I, E, O, Any])
    /**
     * Call the endpoint with a payload, and get a ZIO back.
     * @param payload
     * @return
     */
    def apply(payload: I): RIO[BackendClient, O] =
      ZIO.service[BackendClient]
        .flatMap(_.endpointRequestZIO(endpoint)(payload))
```

Function, when called that turns this pimped endpoint to a function that:

* take arbitrary input I
* use a provided  **BackendClient** (with ZIO.service[**BackendClient**])
* to produce a ZIO effect producing an arbitrary output O

Simply said, with this extension in scope, now createEndpoint can be used as a function:

```scala
def createEndpoint[I, O]( payload: I) : RIO[BackendClient, O]
```

> Note: **RIO** is just a type alias where error are Throwable:
>
>  type **RIO**[R, A]= ZIO[R, **Throwable**, A] 


This effect, needs a BackendClient to run (ZIO.service[BackendClient]).

This is what the second extension will satisfy in a glimpse of Scala:

```scala
extension [E <: Throwable, A](zio: ZIO[BackendClient, E, A])
    /**
     * Emit the result of the ZIO to an EventBus.
     *
     * @param bus
     */
    def emitTo(bus: EventBus[A]): Unit =
      Unsafe.unsafe { implicit unsafe =>                           // (1)
        Runtime.default.unsafe.fork(                               // (2)
          zio
            .tapError(e => Console.printLineError(e.getMessage())) // (3)
            .tap(a => ZIO.attempt(bus.emit(a)))                    // (4)
            .provide(BackendClientLive.configuredLayer)            // (5)
        )
      }
```


This extension will allow to run the ZIO effect and emit the result to an EventBus.

* **(1)** Unsafe is a ZIO utility to run unsafe code.
* **(2)** Runtime is the ZIO runtime forked (worker maybe ?).
* **(3)** tapError will log the error if any.
* **(4)** tap will emit the result to the EventBus.
* **(5)** provide the BackendClient service to the effect, with a layer.

`BackendClientLive.configuredLayer` is the layer that will provide the BackendClient service, this is where most of the magic happens.

```scala
object BackendClientLive:
  val layer = ZLayer.derive[BackendClientLive]

  val configuredLayer: ULayer[BackendClient] = {
    val backend: SttpBackend[Task, ZioStreams] = FetchZioBackend()
    val interpreter                            = SttpClientInterpreter()
    val config                                 = BackendClientConfig(Some(uri"${Constants.backendBaseURL}"))

    ZLayer.succeed(backend) ++ ZLayer.succeed(interpreter) ++ ZLayer.succeed(config) >>> layer
  }
```




In short term, in this one liner we:

* open a channel to a Rest API in a full type safe manner
* serialised the case class to a JSON payload
* wait for the result.
* piped the deserialised result in UI.



# All together

Here is the [full code](https://github.com/cheleb/zio-laminar-demo/blob/master/modules/client/src/main/scala/com/example/ziolaminardemo/app/demos/scalariform/ScalariformDemoPage.scala#L32) of the page that will create a Person and display the registred User in the UI.

```scala
package com.example.ziolaminardemo.app.demos.scalariform

// imports ...


object ScalariformDemoPage:
  def apply() =
    val personVar = Var(Person("Alice", 42, Left(Cat("Fluffy"))))         // (1)
    val userBus   = EventBus[User]()                                      // (2)

    div(
      styleAttr := "width: 100%; overflow: hidden;",
      div(
        styleAttr := "width: 600px; float: left;",
        Form.renderVar(personVar)                                         // (3)
      ),
      div(
        styleAttr := "width: 100px; float: left; margin-top: 200px;",
        Button(
          "-- Post -->",
          onClick --> { _ =>
            // scalafmt:off

            PersonEndpoint                                                // (4)
              .createEndpoint(personVar.now())
              .emitTo(userBus) 

            // scalafmt:on

          }
        )
      ),
      div(
        styleAttr := "width: 600px; float: left;",
        h1("Databinding"),
        child.text <-- personVar.signal.map(p => s"$p"),                  // (5)
        h1("Response"),
        child <-- userBus.events.map(renderUser)                          // (6)
      )
    )

  def renderUser(user: User) =
    div(
      h2("User"),
      div(s"Name: ${user.name}"),
      div(s"Age: ${user.age}"),
      div(s"Pet: ${user.pet}"),
      div(s"Creation Date: ${user.creationDate}")
    )
```

* **(1)** A Laminar Var that will hold the Person to create
* **(2)** An EventBus that will receive the User created
* **(3)** A Form that will bind the Person to the UI see [here](https://cheleb.github.io/laminar-form-derivation/docs/index.html)
* **(4)** A Button that will call the PersonEndpoint.createEndpoint with the current value of the Person Var
* **(5)** A div that will display the current value of the Person Var (debug purpose)
* **(6)** A div that will display the User created, reacting to the EventBus events.

This is a lot of magic in a single page.


# Interacting with JavaScript ecosystem

We can leverage the JavaScript ecosystem in a type safe manner. 

Either by using Scala.js facade or by using the JavaScript API directly.

To illustrate this,  we are using a Laminar binding to a SAP UI5 component.

But more interestingly, you will find in the project a facade to the [Chart.js](https://www.chartjs.org/) library.

This being is generated by [Scalablytyped](https://scalablytyped.org/), a tool that will generate Scala.js facade from TypeScript definition.

This sample is a shamy cut and paste from Sébastien Doeraene demonstration: [Getting started with Scala.js, Laminar and ScalablyTyped](https://www.youtube.com/watch?v=UePrOa_1Am8)

```sbt
addSbtPlugin("org.scalablytyped.converter" % "sbt-converter" % "1.0.0-beta44")
```

Some configuration, devDependencies point to TypeScript whereas runtime dependencies point javascript:
```json
{
  "name": "scalablytyped",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "dependencies": {
    "chart.js": "2.9.4"
  },
  "devDependencies": {
    "typescript": "5.4.5",
    "@types/chart.js": "2.9.29"
  }
}
```

In this example Chart.js binding are generated from the TypeScript definition and fully accessible from ScalaJS at compile time, whereas the chart.js will be used at runtime.
Generated classes are pushed in your local (ivy) repo, hence generated only one.

Notice, this package.json is not the application one, because I want to limit this binding generation to strict necessary.

Then you can refer to TypeScript library very easily, see [here](https://github.com/cheleb/zio-laminar-demo/blob/master/modules/client/src/main/scala/com/example/ziolaminardemo/app/demos/ScalablytypedDemoPage.scala#L103).


# Short conclusion

This is a very simple example, but it shows how ZIO, Laminar and Tapir can be combined to build a full stack webapp. 

* ZIO provides a type safe way to handle effects, and to build a full stack application both on the client and the server side.

* Laminar provides a type safe way to build a reactive UI.

* Tapir provides a type safe way to define API endpoints.


This is a very powerful combination, that allows to build a full stack webapp in a type safe manner. And in the mean time we can leverage the JavaScript ecosystem, without compromising type safety, architecture and scalability of the application.


# Roadmap

## More to come

I will soon push in this g8:
* JWT token handling

## I failed with:

* spliting JS output in multiple files.

## I love to add:

* WASM sample.

# In the end

Hope you appreciate, this introduction.

Many more patterns to discover in the [ZIO rite of passage](https://rockthejvm.com/p/zio-rite-of-passage).

I really love the combo ZIO, Laminar and Tapir, ScalaJs is able to leverage both Scala and JS ecosystem to build fully type safe, reactive webapp with a minimum of boiler plate, thanks to Scala3 new derivation and extension mechanism.

Cheers, and happy coding.

References:
* Rock the JVM: [https://rockthejvm.com](https://rockthejvm.com)
* Laminar: [https://laminar.dev](https://laminar.dev)
* ZIO: [https://zio.dev](https://zio.dev)
* Scalablytyped demo: [https://www.youtube.com/watch?v=UePrOa_1Am8](https://www.youtube.com/watch?v=UePrOa_1Am8)
* UI5 Laminar bindings: [https://github.com/sherpal/LaminarSAPUI5Bindings](https://github.com/sherpal/LaminarSAPUI5Bindings)
* HTML Form derivation: [https://cheleb.github.io/laminar-form-derivation/docs/index.html](https://cheleb.github.io/laminar-form-derivation/docs/index.html)

All the g8 scafolding code is available on [github zio-scalajs-laminar.g8](https://github.com/cheleb/zio-scalajs-laminar.g8)

