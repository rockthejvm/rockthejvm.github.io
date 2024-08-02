---
title: "ZIO Fullstack Webapp"
date: 2024-07-05
header:
  image: "https://res.cloudinary.com/dkoypjlgr/image/upload/f_auto,q_auto:good,c_auto,w_1200,h_300,g_auto,fl_progressive/v1715952116/blog_cover_large_phe6ch.jpg"
tags: [scala, zio, laminar, tapir]
excerpt: "Full stack Scala webapp with ZIO, Laminar, and Tapir"
---

# Full stack ZIO Scala WebApp

Leverage Scala / ScalaJS with ZIO and Laminar to build reactive app.

Build a full stack webapp application using the same language has been a never ending quest. 

In this journey, JS dynamically typed languages and their loosely compilation guarantees were always seen as untouchable because of:

* easy setup
* isomorphic by nature, no need for translation between back and front payload
* quick feedback loop, "just hit reload" 

On the opposite side, statically typed languages are perceived with somewhat good reason as pachyderm on a wire, where static code guarantees were eclipsed by:
* convoluted setup
* polyglot applications, and payload conversion (Scala/Java -> Json)
* no/slow feedback during development.

In this article, I will demonstrate you how my ❤ Scala ecosystem changed the game.

Without compromising type safety guarantees of Scala and in the meantime being able to embed library from JavaScript/TypeScript ecosystem.

## 1. What is inside

* **ZIO**: Functional Effect System on both back and front side.
* **Tapir**: Providing and giving access to a Rest API in a type safe and incredible  elegant way.
* **Laminar**: "Simple, expressive, and safe UI library for Scala.js"

## 2. Setup

Within a single repository, we will setup a multi-modules application using:

* **SBT**, Scala build tool
* **NPM**, Node Package manager
* **Vite**, development and build tool. 

For the very impatient 👇, I've cooked a g8 template:

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

3 processes are running here:

* **fastlink**: Scala to JS compilation/transpilation 
* **npmDev**: http://localhost:5173/ now serves your client web app in development mode, hot reloading with Vite. 
* **serverRun**: http://localhost:8080/docs/ is the Swagger of the API, hot reloading too.

## 3. The build

I won't dive too much into details, see comment in the project.

### 3.1. Scala/ScalaJS world 

Scala build here is driven by SBT in a rather classic way, except for the `shared` project that is cast in two different worlds `.jvm` and `.js`
Generated build contains documentation on details. Please take a look and don't hesitate to ask question/provide PR on the g8.
For ScalaJS

* "org.scala-js" % "sbt-scalajs" % "1.16.0" 
* "ch.epfl.scala" % "sbt-scalajs-bundler" % "0.21.1" will bundle the app
* "ch.epfl.scala" % "sbt-web-scalajs-bundler" % "0.21.1" will expose the bundle as classpath resource (websjar).

### 3.2. Cross building
"org.portable-scala" % "sbt-scalajs-crossproject" % "1.3.2"

This is the plugin that will allow to spread project resources between JS and JVM worlds.
Shared project configuration 👇

* ☕️ Server project aka Scala JVM depends `share.jvm` variant
* 🧭 Client project aka Scala JS depends on `shared.js` variant

ScalaJS and ScalaJS bundler smart enough to crosscompile and deploy `module/shared/src/main/scala` sources in both target.

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


# 5. ZIO, Laminar and Tapir

Match in heaven:

* ZIO and Laminar are both functional and reactive
* Tapir is a type safe way to define API endpoints, then provide or consume them.
* Scala3 extension methods are used to provide a more fluent API.

## 5.1. ZIO

A library for asynchronous and concurrent programming in Scala.

At the type level **ZIO** is a data type that represents an effectful program:
* that can fail with an error of type `E`
* produce a value of type `A`
* may require an element of type `R` to execute.

In a glance ZIO[–R,+E,+A] smart variance annotations allow a very convenient constructions like:

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
        FlywayServiceLive.configuredLayer,
        PersonServiceLive.layer
      )
}
```

Two ZIOs **(1)** and **(2)** combine in a for comprehension **(3)** producing a new ZIO **(4)** that accumulate the dependencies:

ZIO[**FlywayService**, …] ⊕ ZIO[**PersonService & Server**, …] resulting in 

 ZIO[**FlywayService & PersonService & Server**, …]

To be runnable those dependencies need to be satisfied **(5)**, this is what layers are made for.

**ZLayer** are another data type, that abstract services construction and have the same wonderful combinaison properties.

```scala
object FlywayServiceLive {
  def live: RLayer[FlywayConfig, FlywayService] = ZLayer( // (1)
    for {
      config <- ZIO.service[FlywayConfig]
      flyway <- ZIO.attempt(Flyway.configure().dataSource(config.url, config.user, config.password).load())
    } yield new FlywayServiceLive(flyway)
  )

  val configuredLayer: TaskLayer[FlywayService] = Configs.makeConfigLayer[FlywayConfig]("db.dataSource") >>> live  // (2)
}
```
  
* **(1)** is a layer that will provide a **FlywayService**, it depends on the availability of **FlywayConfig** service.
* **(2)** Inject a **FlywayConfig** in a the previous layer and the remove the dependency on the config.

Event better, with some macro magic ... you can get rid of the boilerplate in simple cases.

## 5.2. Laminar

Laminar is a 100% Scala reactive UI library. In few word, Laminar provide reactive variable:
* **Signal**, to watch an unique element
* **EventStream**, to consume stream of events
* **Var**: to watch and update an element
* **EventBus**: to consumes and produces events.

Laminar also provide Dom manipulation facilities, and bindings to WebComponent libraries (UI5 in this example).

## 5.3. Tapir

Tapir is a library for defining API endpoint as immutable data structures.

This is a huge win, because it allows to:

* define the API once
* generate the client and server side
* generate the documentation
* generate the tests

Here is an example of Tapir endpoints definition:

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

# 6. All together

Now that presentation are made, let see how ZIO, Laminar and Tapir can be combined to build a full stack webapp.

## 6.1. ZIO and Tapir

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

It is made available by an extension method on the endpoint.

## 6.2. ZIO, Tapir and Laminar

On **client side**, Tapir will generate the client side code, then: 

* **(1)** from an Laminar UI event
* **(2)** a HTTP endpoint
  * **(2a)** consumes the current value of a Laminar Var
  * **(2b)** turns into ZIO effect to handle the HTTP request
* **(3)** this effect
  * **(3a)** this effect will run
  * **(3B)** its result is consumed by the Laminar UI

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
* add a dependency to **BackendClient** with ZIO.service[**BackendClient**]
* return a ZIO effect producing an arbitrary output O

Simply said, with this extension in scope, now createEndpoint can be used as a function:

```scala
def createEndpoint[I, O]( payload: I) : RIO[BackendClient, O]
```

> Note: RIO is just a type alias where error are Throwable:
>  type RIO[R, A]= ZIO[R, Throwable, A] 


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
      Unsafe.unsafe { implicit unsafe =>
        Runtime.default.unsafe.fork(
          zio
            .tapError(e => Console.printLineError(e.getMessage()))
            .tap(a => ZIO.attempt(bus.emit(a)))
            .provide(BackendClientLive.configuredLayer)
        )
      }
```

`BackendClientLive.configuredLayer` is a layer that will provide the BackendClient service, this is where most of the magic happens.

This extension will allow to run the ZIO effect and emit the result to an EventBus.


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
    val personVar = Var(Person("Alice", 42, Left(Cat("Fluffy")))) // (1)
    val userBus   = EventBus[User]() // (2)

    div(
      styleAttr := "width: 100%; overflow: hidden;",
      div(
        styleAttr := "width: 600px; float: left;",
        Form.renderVar(personVar)  // (3)
      ),
      div(
        styleAttr := "width: 100px; float: left; margin-top: 200px;",
        Button(
          "-- Post -->",
          onClick --> { _ =>
            // scalafmt:off

            PersonEndpoint  // (4)
              .createEndpoint(personVar.now())
              .emitTo(userBus) 

            // scalafmt:on

          }
        )
      ),
      div(
        styleAttr := "width: 600px; float: left;",
        h1("Databinding"),
        child.text <-- personVar.signal.map(p => s"$p"), // (5)
        h1("Response"),
        child <-- userBus.events.map(renderUser) // (6)
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
* **(3)** A Form that will bind the Person to the UI
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


# andThen

## More to come

I will soon push in this g8:
* database integration
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
* Rock the JVM: https://rockthejvm.com
* Laminar: https://laminar.dev
* ZIO: https://zio.dev
* Scalablytyped demo: https://www.youtube.com/watch?v=UePrOa_1Am8
* UI5 Laminar bindings: https://github.com/sherpal/LaminarSAPUI5Bindings
* HTML Form derivation: https://cheleb.github.io/laminar-form-derivation/docs/index.html

All the g8 scafolding code is available on [github zio-scalajs-laminar.g8](https://github.com/cheleb/zio-scalajs-laminar.g8)

