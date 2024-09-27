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

Once we have set up the project, we can start writing some code to explore structured concurrency in Java. In detail, we need an example to work with.