---
title: "Distributed Remote Code Execution Engine"
date: 2024-06-14
header:
    image: "https://res.cloudinary.com/dkoypjlgr/image/upload/f_auto,q_auto:good,c_auto,w_1200,h_300,g_auto,fl_progressive/v1715952116/blog_cover_large_phe6ch.jpg"
tags: [scala,pekko,pekko-http,pekko-stream,pekko-cluster,docker,docker-compose,scala3]
excerpt: "Practical guide to building the distributed remote code execution engine in Scala and Pekko"
---

_by [Anzori (Nika) Ghurtchumelia](https://github.com/ghurtchu)_

## 1. Introduction

After a long hiatus, I am back with renewed passion and energy, eager to delve deeper into the Scala ecosystem. This time, I am committed to building something tangible and useful with the tools available. Let's embark on this exciting journey of exploration and learning together.

The greatest benefit of small side projects is the unique knowledge boost which can potentially be handy later in career.

In this article we will attempt to build the remote code execution engine - the backend platform for websites such as [Hackerrank](https://hackerrank.com), [Leetcode](https://leetcode.com) and others.

If, for some reason you're unfamiliar with the websites mentioned above, the basic usage flow is described below:
- Client sends code
- Backend runs it and responds with output

There you go, sounds simple, right?

Right, right... 

Can you imagine how many things can go wrong here? It's the devil smirking in the corner, knowing, that the possibilities for failure are endless, however, we should address at least some of them.

To give you a quick idea: a separate blog post can be written only about the security, not to mention scalability, extensibility and a few other compulsory properties to make it production ready.

The goal isn't to build the best one, nor it is to compete with the existing ones. 

Put simply, the goal of this project is to get familiar with `Pekko` and its modules such as `pekko-http`, `pekko-stream`, `pekko-cluster` and a few interesting concepts revolving around actor model concurrency, such as: 
- cluster nodes and formation
- cluster aware routers
- remote worker actors
- actor lifecycle and hierarchy
- actor supervision strategies
- actor location transparency
- message serialization
- and so on...

Let's get started then, shall we?

## 2. Project Structure

We will use `Scala 3.4.1`, `sbt 1.9.9`, `docker`, `pekko` and its modules to complete our project.

The initial project skeleton looks like the following:

![alt "Project skeleton"](../images/braindrill/project_skeleton.png)

- `assets` folder contains images mainly for setting up good `README.md`
- `src` contains production and test code
- `target` will store the compiled `.jar` file that we will run later
- `.gitignore` for `git` to ignore some changes
- `.scalafmt.conf` for formatting Scala code
- `build.sbt` for defining build steps, external library dependencies and so on
- `deploy.sh` is a useful script for deploying the project locally within using `docker-compose`
- `docker-compose.yaml` for defining apps, their configuration and properties
- `Dockerfile` blueprint for running the app inside container
- `README.md` instructions for setting up the project locally

Nothing fancy, let's move on `build.sbt`:
```scala
ThisBuild / scalaVersion := "3.4.1"

val PekkoVersion = "1.0.2"
val PekkoHttpVersion = "1.0.1"
val PekkoManagementVersion = "1.0.0"

assembly / assemblyMergeStrategy := {
  case PathList("META-INF", "versions", "9", "module-info.class") => MergeStrategy.discard
  case PathList("module-info.class")                              => MergeStrategy.discard
  case x =>
    val oldStrategy = (assembly / assemblyMergeStrategy).value
    oldStrategy(x)
}

libraryDependencies ++= Seq(
  "org.apache.pekko" %% "pekko-actor-typed" % PekkoVersion,
  "org.apache.pekko" %% "pekko-stream" % PekkoVersion,
  "org.apache.pekko" %% "pekko-http" % PekkoHttpVersion,
  "org.apache.pekko" %% "pekko-cluster-typed" % PekkoVersion,
  "org.apache.pekko" %% "pekko-serialization-jackson" % PekkoVersion,
  "ch.qos.logback" % "logback-classic" % "1.5.6"
)

lazy val root = (project in file("."))
  .settings(
    name := "braindrill",
    assembly / assemblyJarName := "braindrill.jar",
    assembly / mainClass := Some("BrainDrill")
  )
```

As shown:
- we mainly use `pekko` libraries for main work and `logback-classic` for logging
- `root` definition which defines the `name`, `assemblyJarName` and `mainClass`
- all of which is used by `sbt assembly` to turn our code into `braindrill.jar`

and a one line definition in `project/plugins.sbt`:
```scala
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "2.1.1") // SBT plugin for using assembly command
```

## 3. Project Architecture

After a few iterations I came up with the architecture that can be horizontally scaled, if required. 
We have a `master` node and its role is to be the task distributor among `worker` nodes. 
`http` is exposed on `master` node, acting as a gateway to outside world.

### 3.1 Architecture Diagram

![alt "Project skeleton"](../images/braindrill/diagram.png)

Let's explain step by step what happens during code execution:

- User sends request at POST `http://localhost/python` with `python` code attached to request body
- `http` processes the request
- randomly chosen `load balancer` sends message to `worker router` on some node
- `worker router` sends message to `worker` which starts execution
- ... skipping lots of details ...
- `worker` responds to `load balancer` once the code is executed
- `load balancer` forwards message to `http` and user receives the output

Lot of details were skipped here, but they will be covered in the later parts of the blog post.

## 4. Configuration

Before writing any code let's check `resources/application.conf`:
```hocon
pekko {
  actor {
    provider = cluster

    serialization-bindings {
      "serialization.CborSerializable" = jackson-cbor
    }
  }
  http {
    host-connection-pool {
        max-open-requests = 256
    }
  }
  remote {
    artery {
      canonical.hostname = ${clustering.ip}
      canonical.port = ${clustering.port}
      large-message-destinations=[
        "/temp/load-balancer-*"
      ]
    }
  }
  cluster {
    seed-nodes = [
      "pekko://"${clustering.cluster.name}"@"${clustering.seed-ip}":"${clustering.seed-port}
    ]
    roles = [${clustering.role}]
    downing-provider-class = "org.apache.pekko.cluster.sbr.SplitBrainResolverProvider"
  }
}

http {
  port = 8080
  host = "0.0.0.0"
}


clustering {
 ip = "127.0.0.1"
 ip = ${?CLUSTER_IP}
 port = 1600
 port = ${?CLUSTER_PORT}
 role = ${?CLUSTER_ROLE}
 seed-ip = "127.0.0.1"
 seed-ip = ${?CLUSTER_IP}
 seed-ip = ${?SEED_IP}
 seed-port = 1600
 seed-port = ${?SEED_PORT}
 cluster.name = ClusterSystem
}
```

This file configures various settings for the Pekko application, including actor system properties, HTTP settings, remote communication, and clustering parameters:

**1. Pekko Actor System Configuration**
- `provider = cluster`: This setting specifies that the actor system will use clustering capabilities.
- `serialization-bindings`: This section defines serialization bindings for specific classes. Here, any class implementing `serialization.CborSerializable` will be serialized using the `jackson-cbor` serializer.

**2. HTTP Configuration**
- `host-connection-pool.max-open-requests`: This setting specifies the maximum number of open requests in the HTTP host connection pool. It is set to 256.

**3. Remote Communication Configuration**
- `remote.artery`: This section configures the Artery transport (a remoting mechanism in Pekko).
- `canonical.hostname`: This sets the hostname for the actor system, which is derived from `clustering.ip`.
- `canonical.port`: This sets the port for the actor system, which is derived from `clustering.port`.
- `large-message-destinations`: Specifies destinations for large messages. In this case, any destination matching the pattern `/temp/load-balancer-*` will be treated as a large message destination.

**4. Cluster Configuration**
- `seed-nodes`: Defines the initial contact points for the cluster, using placeholders for cluster name, seed IP, and seed port.
- `roles`: Specifies the roles of the cluster node, derived from clustering.role.
- `downing-provider-class`: Specifies the class for handling split-brain scenarios. Here, it's set to `SplitBrainResolverProvider`.
 
**5. Http Server Configuration**
- `http.port`: Sets the HTTP server port to 8080.
- `http.host`: Sets the HTTP server host to 0.0.0.0, meaning it will bind to all available network interfaces.

**6. Clustering Variables**
- `ip`: Default IP address for clustering is 127.0.0.1. It can be overridden by the environment variable `CLUSTER_IP`.
- `port`: Default port for clustering is 1600. It can be overridden by the environment variable `CLUSTER_PORT`.
- `role`: Role of the cluster node, which can be set using the environment variable `CLUSTER_ROLE`.
- `seed-ip`: Default seed IP address is 127.0.0.1. It can be overridden by `CLUSTER_IP` or `SEED_IP`.
- `seed-port`: Default seed port is 1600. It can be overridden by the environment variable `SEED_PORT`.
- `cluster.name`: Name of the cluster, set to `ClusterSystem`.

This configuration file is designed to be flexible, allowing for environment-specific overrides while providing sensible defaults, it plays crucial role with running the pekko cluster within the containers.

### 4.1 transformation.conf

The project also requires a special configuration for capturing properties for domain objects such as `worker`, `load-balancer` and so on.

```hocon
include "application"

transformation {
  workers-per-node = 32
  load-balancer = 3
}
```

Here, it simply means that each node will have 32 worker actors and master node will have 3 load balancer actors.

## 5. Cluster System




