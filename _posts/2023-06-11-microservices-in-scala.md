---
title: "Architecting Scala: Microservices with the Typelevel stack"
date: 2023-06-11
header:
  image: "/images/blog cover.jpg"
tags: [ microservices, architecture, scala, typelevel ]
excerpt: "A guide to building microservices in Scala with the Typelevel stack."
---

# Scala in Production: Microservices with the Typelevel stack

# 1. Introduction

While ScalaJS has made the full stack Scala application possible, that is far from the norm. In fact, out in the wild
you will find that most Scala code is inside highly performant, and easily Scalable microservice. In this article we
will explore a few different ways of architecting our applications.

## 1.1 Requirements

This article assumes you are familiar with the basics of Scala and have at least _some_ experience with the Typelevel
stack. You should understand:

- What is an `IO`
- How to define endpoints with `http4s`
- Declare and consume streams with `fs2`

If you are not familiar with some of these concepts, you can find articles covering each of them right here on
RockTheJVM. And if you're up for a challenge, check out the Typelevel Rite of Passage to see just how far you can go
with these technologies.

# 2. On microservices

First of all, we should make sure we're all on the same page, and that we mean the same thing when we say the word "
microservice".

## 2.1 Why microservices?

We can picture all apps existing on a spectrum. On the one side, we have our good old monoliths. These apps are all
written in one language, deployed into a single artifact and run on a single runtime. If you've ever made a solo project
where you created a functional app where you have a single repo, which spans all layers of the stack from the UI to the
database, you've made a monolith.

Monoliths can still be found in abundance in the wild, specially in old legacy systems. In fact Prime Video recently
moved from microservices back to a Monolith architecture (you can read about
that [here](https://www.primevideotech.com/video-streaming/scaling-up-the-prime-video-audio-video-monitoring-service-and-reducing-costs-by-90?utm_source=thenewstack&utm_medium=website&utm_content=inline-mention&utm_campaign=platform)).

While monoliths can work great, specially when working at low scale, they can become a nightmare when the app starts to
scale, both in size and traffic. Common issues when scaling a monolith include:

1. Difficult to maintain: while in the greenfield phase development will probably go fast, your codebase will soon reach
   a critical mass where making changes to it becomes very difficult. And while following principles such as GRASP or
   SOLID might help, the underlying issue is that the app has become to big for a normal human to hold in their heads at
   once. This means that the chances of unintended side effects increase, making change slow and uncertain.
2. All or nothing scaling: Because the app is coupled together in a single artifact, it is impossible to scale only the
   parts that need it. For example, lets take an e-commerce app. During the holidays there is a spike in traffic, but
   only in the checkout and payment parts of the app. However, because the whole app is coupled together, we have to
   scale the whole thing, even the parts that don't need it. This means we have to pay for more servers than we need,
   and that we have to wait for the whole app to scale before we can handle the traffic.
3. Fault tolerance: Because the app is one single thing, one module crashing could potentially crash the entire thing.
   Failure isn't isolated, and an uncaught exception could go up the stack till the entire monolith topples down.

We could probably gripe on monoliths a bit more, but hopefully at this time you are convinced that if microservices can
solve at least some of these issues then they are the way to go. At least for some types of apps.

## 2.2 What is a microservice?

The solution to these problems is in fact quite simple: just take your monolith and split it into parts.

A microservice architecture is one where instead of one single app running, our backend is instead a collection of
services which all communicate with each other. Each service is a separate app, with its own codebase, its own runtime
and its own deployment. This seemingly innocuous change solves our problems:

1. Easier to maintain: Because the app is now split into several smaller services, it is now easier to make small
   localized changes. This is because while our meager human brains can't hold the complexity of the entire system, they
   can hold the complexity of a single microservice. As long as we don't change how the microservice looks to the
   outside, it becomes easier and faster to change and iterate.
2. Granular scaling: Because our app is now separate, we can scale each component on its own based on traffic. Going
   back to our e-commerce example, we can now scale _only_ the checkout and payment services, and leave the rest of the
   app alone. This means we can handle more traffic with fewer servers, and that we can handle traffic spikes faster.
3. Isolated failure: Because each microservice is a separate app, a failure in one of them doesn't have to crash the
   entire system. It still may, but because the contact points are clear it becomes easier to isolate the failure and
   prevent it from cascading.

## 2.3 The microservice premium

At this point I feel compelled the mandatory:
_[beware the microservice premium](https://martinfowler.com/bliki/MicroservicePremium.html)_ warning. Microservices are
not a silver bullet, and they come with a lot of associated costs that make it not worth it unless you're hitting a
certain scale, both in terms of traffic and team size.

If you don't head this warning and still decide to go with microservices for your small to do app, beware you will be
paying a hefty price in terms of:

1. Increase complexity: Your design will have to be much more robust, and you will have to deal with a lot more moving
   parts. This means that you will have to spend a lot more time thinking about your design, and that you will have to
   deal with a lot more failure modes.
2. Simple requirements become difficult to implement: If your design isn't built to be extensible from the ground up,
   something as simple as adding a [user's birthday](https://www.youtube.com/watch?v=y8OnoxKotPQ) to the app could
   become months of work.
3. Coordination of work between teams: this I feel is a big one that often goes understated. Big, fundamental change
   becomes difficult if not planned for from the beginning. Why? Because now you have to orchestrate that change between
   multiple teams, and one team delaying means the entire feature gets delayed.

Granted, all of these problems can be either solved or at least mitigated with good design, but if you're not careful
you might find yourself going down a rabbit hole that you won't know how to work your way out of.

# 3. How to design our microservices

Now we know what microservices are, you're probably wondering how to split your app into services, ideally in a way that
removes or minimizes the microservice premium. Worry not, we'll discuss this shortly. But the main thing you should keep
in mind is that designing microservices is highly dependent on context and business needs.

## 3.1 Principles of microservice design

There is no algorithm where you can input your monolith and get a microservice architecture out. There are however,
certain guidelines we can keep in mind that will generally take us in the right track.

### Single Responsibility

When designing microservices, the main thing is to go small. There's a reason they are called _micro_services. There is
however, such a thing as going too small. The main thing to keep in mind is that a microservice should fulfill a single
function within the app. It should do one thing really well.

If your microservice is too big, you start losing the maintainability benefits. If your microservices are too granular,
your system will become very complex and the microservice premium will skyrocket. It's a fine line to walk in.

### Loose coupling

Your microservices should be very loosely coupled. If your microservices are strongly coupled, you don't have
microservices; you have a distributed monolith, which just takes the worst of all worlds. Besides the schema and the
API, your microservices should be completely independent of each other.

The caveat here is that this might lead to some code duplication (especially regarding business rules and validations),
but this is a small price to pay to avoid the cost of strong coupling. Trust me, if you think tracking bugs in a
monolith is hard, try tracking them in a distributed monolith.

### Complete encapsulation

Each microservice should encapsulate its own data. This means that each microservice is responsible for its own data
storage, and no other microservice should care how this is implemented. This is important because it allows us to change
the implementation of a microservice without affecting the rest of the system. Having to coordinate deployments between
teams is hard enough as it is, we don't need to add to the complexity by having to coordinate database migrations.

### Total ownership

In my experience, microservices work best when there is a team that is explicitly made responsible for each part of the
system. This clear separation of concerns allows us to track down and tackle issues quickly. This does not mean that
teams can't help each other or that this formal control structure is set in stone, but it does mean that there is a
clear line of responsibility. Beware however of using this as an excuse to create silos of knowledge, there should be
regular transfers of knowledge that allow each team to understand how their service fits into the hole. Also, the
purpose if this is not to assign blame when something goes wrong, but to make sure that the right people are working on
the right things and giving them autonomy to work in the way they think is best.

## 3.2 Functional Domain Driven Design and Microservices in Scala

So far, we have seen a lot of theory, but it's time to get it down to earth. Luckily, we are functional programmers,
which means that we inherently work with small, loosely coupled components. This means that we are already halfway
there.

So how do we take these ideas and translate them into the world of microservices? Well, the first thing we do is start
thinking in terms of functional domain driven design. We'll try to refrain from thinking about our system as a
collection of services, and instead think about it as a series of transformations over some data. In this way, each
microservice becomes somewhat of a big function that performs one of these transformations.

Of course, we live in the real world. Because of this, some (if not most) of our transformations will have to contain
side effects. Luckily for us, our chosen stack comes with a great abstraction for handling side effects out of the box.

Also, notice that all of our services will be following the same patterns:

1. Handle an incoming request
2. Decode the data
3. Scatter the data into multiple parallel computations
4. Gather the data of all the parallel computations
5. Encode the data
6. Pass the message along to the following service

This type of Handle, Decode, Scatter -> Gather, Encode, Respond is in
fact [the intended use case for Cats-Effect](https://youtu.be/qgfCmQ-2tW0?t=109).

# 4. A practical guide: Let's build something together

This isn't an article on DDD, so I'm not going to go in too deep. Just deep enough for it to be serviceable to our
architectural design.

For this to be informative, we're going to build something a bit more complex than a to-do app. Let's assume we've been
hired to build a platform for a company that sells furniture on demand.

Before we can design the system we need to properly understand the requirements, so I'm going to make them up right now:

- They sell furniture in two distinct modes:
    - In bulk to distributors. This includes worldwide shipping through some third party company that luckily offers an
      API which can handle all the shipping logistics.
    - To individuals. This is done locally and the delivery is handled by the company itself.
- While they do keep a minimum stock of their most popular items, most of their furniture is made to order.
- In both cases they need to create electronic invoices for their customers. These must then be sent to the government
  for tax purposes, via an API they provide.

We could make this more complicated, but I think this is enough to get the point across. So let's start designing our
system. We can always make up more requirements as we go (and we probably will, since requirements change all the time).

## 4.1 The domain

So the first thing we need to do is identify the flow of information: when does our system kick into gear? Well, this
flow kicks in whenever a customer places an order. So let's start there.

We have two types of customers: wholesalers (who buy in bulk) and individuals (who buy one or two items at a time). So
let's start by defining our domain. Let's drill down on the requirements, and ask around to see how these people shop.

Assuming I asked around, this is what I found out:

1. Wholesalers get the furniture catalogue every month. To place an order they send in a form via the mail, where they
   place the product code of what they are interested in and the quantity they want. They also include their company
   information and the address where they want the furniture delivered.
2. Individuals buy through a regular e-commerce website. They can browse the catalogue, add items to their cart and then
   place an order. They also need to provide their personal information and the address where they want the furniture
   delivered.

Because of this separation I'm deciding to build two separate UIs, one for the wholesalers and an e-commerce site. That
way if the e-commerce site breaks the wholesalers can still place their orders, which is where most of the company
revenue comes from.

Our flow of information now becomes much more clear:

1. A customer places an order
2. The order is validated
3. We check in our inventory if we have enough stock to fulfill the order
4. If we do, we mail those items to the customer
5. If we don't, we contact the factory to make the items
    1. Once the items get manufactured, we still send them to the customer
6. Whatever the case, we need to create an invoice for the customer
7. Throughout the entire process, the customer should be notified of each step

Each one of these steps is actually hiding a lot of complexity. But if we abstract over all of that we get this rather
simple graph:

You can see how the information flows down from one section to the next. More over, each unit is a discrete _concept_
that is connected to the others only via inputs and outputs.

This is what we are aiming for when we split our system into microservices. Each microservice should be a discrete
concept that is connected to the others only via inputs and outputs.

-- Insert image here --

Each of the nodes in our graph represents a _subdomain_, a section of the domain which concerns itself with one specific
thing.

Once again, splitting your domain into subdomains is not an exact science, there's a creative element to it. The keys to
properly identifying subdomains are similar to those of properly identifying microservices:

1. Each subdomain should be concerned with a single process.
2. Each subdomain should have everything it needs to perform the necessary transformations to the data.
3. Subdomains should communicate via Data Transfer Objects.
4. Since we're modelling this as a transformation of data, communication should be one way. If your systems are speaking
   back and forth, they're too coupled to be separate subdomains.

## 4.2 The services

Now that we know our workflow at a very high level, we're going to take our subdomains and map each of them to a
microservice.

So this means we'll get the following services:

1. The e-commerce site
2. The wholesaler site
3. The inventory service
4. The factory service
5. The shipping service
6. The invoicing service
7. The tax service
8. The notification service

We get the following, high level architecture diagram:

-- Insert image here --

Now we need to define our APIs, and determine how each service will communicate with one another.

-- Endpoint definitions

# Leveraging Scala: Event Driven Architecture and Streaming

So far so good, you have everything to do microservices in the most general level. But we can take it a step further.

Generally, communications between microservices is done via HTTP, sending JSONs from one end to another. While that
works, as you scale to millions of users this might get a bit slow, or at least not as fast as it could be.

You see, JSON is a _big_ format. This comes from the fact that the data schema is sent over along with the data. This is
great for flexibility and decoupling, but it's not so great for performance.

For the small price of some coupling however, you can change this and eliminate all your overhead. How? Via message
queues. There are several message brokers we can consider depending on the specific of our use case, but since that is
not the focus of this article I'm just going to use Kafka.

Using a message queue is a double-edged sword: by including the schema in our code, we save ourselves a lot of traffic
by not sending it over the network. But we may also increase our coupling. A normal way to handle that is to extract
the message schemas into a common library which can then be imported into microservices. This makes it so that changes
to the schema can be reviewed by all involved teams, and we have a single source of truth for all of our messages.

With this in mind, now our architecture changes, at least conceptually. Instead of being a bunch of systems
communicating between each other, our architecture becomes a bunch of systems which are listening for certain events.
Each microservice listens to the events that are relevant to their domain, which we can organize via different queues.

-- Insert image here --

And because we're using queues, we can still scale horizontally by having multiple consumers (read: multiple instances
of the same microservice) reading off the same queue.


