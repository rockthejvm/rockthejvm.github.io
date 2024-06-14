---
title: "WebSockets in Scala, Part 2: Integrating Redis and PostgreSQL"
date: 2024-05-23
header:
  image: "https://res.cloudinary.com/dkoypjlgr/image/upload/f_auto,q_auto:good,c_auto,w_1200,h_300,g_auto,fl_progressive/v1715952116/blog_cover_large_phe6ch.jpg"
tags: []
excerpt: "Learn how to use Redis in Scala using redis4cats and persist records on Postgres using skunk, adding new functionality to our chatroom application"
toc: true
toc_label: "In this article"
---

_by [Herbert Kateu](https://github.com/hkateu)_

## 1. Introduction
This article is a follow-up to the [websocket](/websockets-in-http4s/) article that was published previously. To recap, we created an in-memory chat application using WebSockets with the help of the Http4s library. The chat application had a variety of features implemented through commands directly in the chat window such as the ability to create users, create chat rooms, and switch between chat rooms. 

In this iteration, we'll be integrating Redis to keep track of the users and rooms and we'll also be persisting messages in Postgres so that new users can have access to previous conversations. Finally, We'll get rid of `chatState` and create a new protocol that interacts with Postgres and Redis.

Since this tutorial builds on the previous article, to follow along, we'll need to clone that [GitHub repo](https://github.com/hkateu/WebsocketChatApp) where we'll be making the necessary updates to build this new version.

## 2. Setting Up
We'll be using skunk and redis4Cats in our application so let's add them to our `build.sbt` file.

```scala
...
val RedisVersion = "1.5.2"
val SkunkVersion = "0.6.3"

lazy val root = (project in file("."))
  .settings(
    organization := "rockthejvm",
    name := "websockets",
    version := "0.0.1-SNAPSHOT",
    scalaVersion := "3.3.3",
    libraryDependencies ++= Seq(
      ...
      "dev.profunktor" %% "redis4cats-effects" % RedisVersion,
      "dev.profunktor" %% "redis4cats-streams" % RedisVersion,
      "org.tpolecat" %% "skunk-core" % SkunkVersion
    ),
    libraryDependencies ++= Seq(
      "io.circe" %% "circe-core",
      "io.circe" %% "circe-generic",
      "io.circe" %% "circe-parser"
    ).map(_ % CirceVersion)
  )
```
We also add circe core, generic and parser to `build.sbt`.

## 3. Creating the Domain
In this version, we'll add a `UUID` to the `User` and `Room` case classes and move the `validateutility` to its own file. Let's create a `validateutility.scala` in the following path, `src/main/scala/rockthejvm/websockets/domain`, and add the following code:

```scala
package rockthejvm.websockets.domain

import cats.data.Validated

object validateutility{
  def validateItem[F](
      value: String,
      userORRoom: F,
      name: String
  ): Validated[String, F] = {
    Validated.cond(
      (value.length >= 2 && value.length <= 10),
      userORRoom,
      s"$name must be between 2 and 10 characters"
    )
  }
}
```
Nothing has changed from the previous version. Next, we'll create a `user.scala` file in the following path, `src/main/scala/rockthejvm/websockets/domain`. This file will contain the new `User` case class.

```scala
package rockthejvm.websockets.domain

import java.util.UUID
import cats.Applicative
import cats.data.Validated
import rockthejvm.websockets.domain.validateutility.*
import cats.syntax.all.*

object user {
  case class UserId(id: UUID)
  case class UserName(name: String)
  case class User(id: UserId, name: UserName)
  object User {
    def apply[F[_]: Applicative](
        id: UUID,
        name: String
    ): F[Validated[String, User]] =
      validateItem(name, new User(UserId(id), UserName(name)), "User name")
        .pure[F]
  }
}
```
Here we create a `User` case class that contains the `UserId` and `UserName` that take a `UUID` and `String` respectively. In the `apply` method we still use `validateItem()` to create the `F[Validated[String, User]]`.

Let's also do the same for `room.scala` in the same path and add the following code:

```scala
package rockthejvm.websockets.domain

import java.util.UUID
import cats.data.Validated
import cats.syntax.all.*
import cats.*
import rockthejvm.websockets.domain.validateutility.*

object room {
  case class RoomId(id: UUID)
  case class RoomName(name: String)
  case class Room(id: RoomId, name: RoomName)
  object Room {
    def apply[F[_]: Applicative](
        id: UUID,
        name: String
    ): F[Validated[String, Room]] =
      validateItem(name, new Room(RoomId(id), RoomName(name)), "Room").pure[F]
  }
}
```
The code here is very similar to `user.scala`. We'll also need to insert messages into our database, for this we'll need another case class, create a `message.scala`  file under `domain`, and add the following code:

```scala
package rockthejvm.websockets.domain

import java.util.UUID
import java.time.LocalDateTime
import rockthejvm.websockets.domain.user.{UserId, User}
import rockthejvm.websockets.domain.room.RoomId

object message {
  case class MessageId(id: UUID)
  case class MessageText(value: String)
  case class MessageTime(time: LocalDateTime)
  case class InsertMessage(
      id: MessageId,
      value: MessageText,
      time: MessageTime,
      userId: UserId,
      roomId: RoomId
  )
}
```
The values in `InsertMessage` match the columns we'll have when we create the `messages` table in Postgres. We'll also need another case class that will hold the messages fetched from the database:


```scala
package rockthejvm.websockets.domain

...

object message {
  ...
  case class FetchMessage(value: MessageText, from: User)
}
```
The `FetchMessage` case class contains a `MessageText` and a `User` from whom the `message` was sent. The reason we need another case class is that only want to fetch two columns from Postgres.

## 4. Docker for Redis and PostgreSQL

We'll be using Docker images for Redis and Postgres. To follow along, you'll need [Docker](https://www.docker.com) and [Docker Compose](https://docs.docker.com/compose/) installed. We can install them by installing [Docker desktop](https://www.docker.com/products/docker-desktop/) on your system.

After installation, we can check if we have everything installed by running the following:
```bash
$ docker -v    
Docker version 25.0.3, build 4debf41

$ docker-compose -v
Docker Compose version v2.24.5
```
Next, we'll create the SQL commands to create the database and necessary tables for Postgres. In the root folder create setup.sql and add the following:

```sql
CREATE DATABASE websocket;

\c websocket;

CREATE TABLE users (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE rooms (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE messages (
    id UUID PRIMARY KEY,
    message TEXT NOT NULL,
    time TIMESTAMP DEFAULT NOW(),
    user_id UUID REFERENCES users (id),
    room_id UUID REFERENCES rooms (id)
);
```
The first command creates a database called `websocket`, `\c websocket` connects to it, then we create a `users` and `rooms` table each with an `id` of type `UUID` and `name` of type `VARCHAR(255)`, and finally we create the messages table with an `id`, `message` and `time` columns and it also references the `users` and `rooms` table `id`.

Then we need manage our docker stack using docker-compose. Let's create a `docker-compose.yaml` file in the root folder of our application and add the following:

```yaml
services:
  postgres:
    image: postgres:alpine
    container_name: postgres-server
    restart: always
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    ports:
      - 5432:5432
    volumes:
      - ./postgres_volume:/var/lib/postresql/data
      - ./setup.sql:/docker-entrypoint-initdb.d/setup.sql

  redis:
    image: redis:alpine
    container_name: redis-server
    restart: always
    ports:
      - 6379:6379

volumes:
  postgres_volume:
    driver: local
```
Here we define, two `services`, `postgres` and `redis`. Under `postgres`, we specify the image name as `postgres:alpine`, we also specify the container name as `postgres-server` and set it to restart `always`.

Postgres containers need a username and password which we specify as environment variables under the `environment` segment. Next, we give it a port number of `5432` for both internal and external and lastly, we specify the docker volume for the container as `postgres_volume`.

A docker volume helps keep the state of the container, in our case all the databases and tables will be backed up in case the container is destroyed.

Finally, to run our `setup.sql` file when the container is initialized, we add `./setup.sql:/docker-entrypoint-initdb.d/setup.sql` under volumes.

Under the Redis section, we only specify the image name, container name, restart policy, and port number.

Moving from the `services` to `volumes` section, we set the `postgres_volume` `driver` as `local` so that docker writes to the external disk. 

Finally, we can start our docker containers by running the following command in the root of our application.:
```bash
$ docker compose up
```
We can confirm that our containers have been created and are running with the following command:
```bash
$ docker ps
```
We can also confirm that our database and tables have been created by running the following:
```bash
$ docker exec -it postgres-server psql -U postgres                                                               
```
Then connect to the database and finally list the tables by running the following:
```bash
$ \c websocket
$ \d
```
## 5. Skunk for PostgreSQL Integration

In this section, we'll implement the protocols necessary for interacting with Postgres in our application using [Skunk](/skunk-complete-guide/).

First, we'll need to implement `Codec`s for the types in our domain. Create a `codecs.scala` file in the following path, `src/main/scala/rockthejvm/websockets/codecs/codecs.scala` and add the following code:

```scala
package rockthejvm.websockets.codecs

import skunk.*
import skunk.codec.all.*
import rockthejvm.websockets.domain.user.{UserId, UserName}
import rockthejvm.websockets.domain.room.{RoomId, RoomName}
import rockthejvm.websockets.domain.message.{
  MessageId,
  MessageText,
  MessageTime
}

object codecs {
  val userId: Codec[UserId] = uuid.imap[UserId](UserId(_))(_.id)
  val userName: Codec[UserName] =
    varchar(255).imap[UserName](UserName(_))(_.name)
  val roomId: Codec[RoomId] = uuid.imap[RoomId](RoomId(_))(_.id)
  val roomName: Codec[RoomName] =
    varchar(255).imap[RoomName](RoomName(_))(_.name)
  val messageId: Codec[MessageId] = uuid.imap[MessageId](MessageId(_))(_.id)
  val messageText: Codec[MessageText] =
    text.imap[MessageText](MessageText(_))(_.value)
  val messageTime: Codec[MessageTime] =
    timestamp.imap[MessageTime](MessageTime(_))(_.time)
}
```
Skunk provides several important codecs through the `skunk.codec.all.*` import such as `uuid`, `varchar`, `text`, and `timestamp`. We use the `imap` method to provide the `encode` and `decode` methods for each case class, here's the signature:

```scala
  def imap[B](f: A => B)(g: B => A): Codec[B] =
    Codec(b => encode(g(b)), decode(_, _).map(f), types)
```
Here we provide the methods from case class to postgres and the reverse. Here we provide codecs for case classes that take native types, however, later we shall see how to join codecs for types such as `InsertMessage`.

Now we'll create various methods that will interact with the Postgres database through `Skunk`. Let's create a `PostgresProtocol.scala` file under the `websockets` and add the following code:

```scala
package rockthejvm.websockets

import fs2.Stream
import java.time.LocalDateTime
import rockthejvm.websockets.domain.user.*
import rockthejvm.websockets.domain.room.*
import rockthejvm.websockets.domain.message.*

trait PostgresProtocol[F[_]] {
  def createUser(name: String): F[Either[String, User]]
  def createRoom(name: String): F[Either[String, Room]]
  def deleteRoom(roomId: RoomId): F[Unit]
  def deleteUser(userId: UserId): F[Unit]
  def saveMessage(
      message: String,
      time: LocalDateTime,
      userId: UserId,
      roomId: RoomId
  ): F[Unit]
  def fetchMessages(roomId: RoomId): F[Stream[F, FetchMessage]]
  def deleteRoomMessages(roomId: RoomId): F[Unit]
}
```
First, we create a `PostgresProtocol` trait with a number of methods that will later be used by our chat protocol:

```scala
...
import cats.effect.std.UUIDGen
import cats.effect.* 
import skunk.* 

trait PostgresProtocol[F[_]] {
  ...
}

object PostgresProtocol {
  def make[F[_]: UUIDGen: Concurrent](
      postgres: Resource[F, Session[F]]
      ): F[PostgresProtocol[F]] = 
          postgres.use {session => 
            new PostgresProtocol[F] {
            ???
            }.pure[F]
          }
}
private object SqlCommands {
  ???
}
```
We also provide a companion object with a `make()` method where we'll implement all the methods defined in the trait, below that we have the `SqlCommands` object which will contain other `Codecs` used in our implementations. Here we call `postgres.use()` to access the `session` which represents a live connection to the database, and inside that, we'll implement an `F[PostgresProtocol[F]]`.

First, we start with `createUser()`, which inserts a user into the database, to do this, we'll need a `Codec[User]` for this to work. 
```scala
import rockthejvm.websockets.codecs.codecs.*
...
private object SqlCommands {
    val usercodec: Codec[User] = 
        (userId ~ userName).imap {
            case i ~ n => User(i,n)
        }(u => (u.id, u.name))
}
```
Here we create a `twiddle` list, `(userId ~ userName)` which produces a `Codec[(UserId, UserName)]`, we then call `imap` and provide the two functions from `userId ~ userName` to `User` and the reverse

Let's also go ahead and implement the `Codec[Room]` since its very similar to `Codec[User]`:

```scala
private object SqlCommands {
  ...
  val roomcodec: Codec[Room] = 
      (roomId ~ roomName).imap {
          case i ~ n => Room(i,n)
      }(u => (u.id, u.name))
}
```

Next, we'll need to create its associate `Command` for the `INSERT` statement. In Skunk, `Select` statements are prepared with a `Query` while `Insert` and `Update` statements are prepared with a `Command` since it return no rows.

```scala
private object SqlCommands {
  ...
  val insertUser: Command[User] = 
      sql"""
          INSERT INTO users
          VALUES($usercodec)
      """
      .command
}
```
Here we prepare an insert statement, and pass the `usercodec` to `VALUES()`, then call the `command` method giving us a `Command[User]`. A `Command[User]`, means we need to pass a `User` inorder to execute the query:

Let's also create `Command[Room]` since its also similar to the above:

```scala
private object SqlCommands {
  ...
  val insertRoom: Command[Room] =
    sql"""
            INSERT INTO rooms
            VALUES($roomcodec)
        """.command
}
```
Now we can implement `createUser()`:

```scala
...
new PostgresProtocol[F] {
  override def createUser(name: String): F[Either[String, User]] =
    session.prepare(insertUser).flatMap { cmd =>
      UUIDGen.randomUUID.flatMap { id =>
        User(id, name).flatMap {
          case Valid(u) =>
            cmd.execute(u).as(Right(u))
          case Invalid(err) =>
            Left(err).pure[F]
        }
      }
    }
}.pure[F]
```
The `checkUser()` function takes a name of type `String` and returns an `F[Either[String, User]]`. Here we call the `prepare()` on `session` passing it the `insertUser` `Command`, this caches and prepares the query. 

The `prepare()` method produces a `F[PreparedCommand[F, User]]`, which we flatMap on to create our `User`. We use the  `UUIDGen.randomUUID` method from cats.effect to create our `UUID` which we pass to `User` along with the `name`.

If `User` creation succeeds, we call cmd.execute(u) which runs the `INSERT` query, and finally, we return a `Right(u)`. If the creation fails, we pass the error as `Left(err).pure[F]`.

Let's also implement createRoom() since it's similar:

```scala
...
new PostgresProtocol[F] {
  override def createRoom(name: String): F[Either[String, Room]] =
    session.prepare(insertRoom).flatMap { cmd =>
      UUIDGen.randomUUID.flatMap { id =>
        Room(id, name).flatMap {
          case Valid(r) =>
            cmd.execute(r).as(Right(r))
          case Invalid(err) =>
            Left(err).pure[F]
        }
      }
    }
}.pure[F]
```
Next is `deleteUser()` and `deleteRoom()`, like before, we'll need to first create sql statements of type `Command` for both scenarios. The `Command[UserId]` means that a `UserId` is needed as an argument:

```scala
private object SqlCommands {
  ...
  val delUser: Command[UserId] = 
      sql"""
          UPDATE users
          SET name = "deletedUser"
          WHERE id = $userId
      """
      .command
}
```
In our case instead of completely deleting the user, we replace the name with `deletedUser` with the help of an `UPDATE` statement, so that we keep the messages in the database even if the user doesn't exist:

However, for the case of `Room`s, we'll be completely deleting a `room` that matches a particular `RoomId` using a `DELETE` statement:
```scala
private object SqlCommands {
  ...
  val delRoom: Command[RoomId] = 
    sql"""
        DELETE FROM rooms
        WHERE id = $roomId
    """
    .command
}
```

Now let's define the `deleteUserOrRoom()` private function:

```scala
new PostgresProtocol[F] {
  ...
  private def deleteUserOrRoom[A](id: A, command: Command[A]): F[Unit] =
    session.prepare(command).flatMap { cmd =>
      cmd.execute(id).void
    }
}.pure[F]
```
This function takes a `command` of type `Command[A]` and an `id` of type `A` that we pass to prepare and execute methods respectively to produce an `F[Unit]`. Here's how we'd use it

```scala
...
new PostgresProtocol[F] {
  ...
  override def deleteUser(uId: UserId): F[Unit] =
    deleteUserOrRoom[UserId](uId, delUser)

  override def deleteRoom(rId: RoomId): F[Unit] =
    deleteUserOrRoom[RoomId](rId, delRoom)
}.pure[F]
```
Both these functions take either a `UserId` or a `RoomId`, which we pass to `deleteUserOrRoom()` along with it's respective `Command`.

Next, we have `saveMessage()`, which will be used to insert new messages into the database, this will require a message codec to decode and encode messages to and from Postgres:

```scala
private object SqlCommands {
  ...
  val messagecodec: Codec[InsertMessage] = 
      (messageId ~ messageText ~ messageTime ~ userId ~ roomId).imap {
          case mi ~ mt ~ mtm ~ ui ~ ri => InsertMessage(mi,mt,mtm,ui,ri)
      }(m => (((((m.id), m.value), m.time), m.userId), m.roomId))
}
```
Here we a `Codec[InsertMessage]`, the process is similar to how we created the `usercodec`, except that the decode function has to return a nested tuple as shown above.

Now we can create a `Command[InsertMessage]` to `INSERT` a message:

```scala
private object SqlCommands {
  ...
  val insertMessage: Command[InsertMessage] = 
      sql"""
          INSERT INTO messages
          VALUES($messagecodec)
      """
      .command
}
```
At this point, we can now implement the save message function as follows:
```scala
...
new PostgresProtocol[F] {
  ...
  override def saveMessage(
      message: String,
      time: LocalDateTime,
      userId: UserId,
      roomId: RoomId
  ): F[Unit] =
    session.prepare(insertMessage).flatMap { cmd =>
      UUIDGen.randomUUID.flatMap { id =>
        cmd
          .execute(
            InsertMessage(
              MessageId(id),
              MessageText(message),
              MessageTime(time),
              userId,
              roomId
            )
          )
          .void
      }
    }
}.pure[F]
```
Just like before, we create a random `UUID` then pass all the required values to `InsertMessage` within the execute method and finally call `void` to return an `F[Unit]`.

The `fetchMessages()` function is interesting since it requires a `JOIN` sql statement. This is the reason we needed a `FetchMessage` case class to retrieve messages. Like before, we'll need a codec:

```scala
private object SqlCommands {
  ...
  val fetchmessagecodec: Codec[FetchMessage] =
    (messageText ~ userId ~ userName).imap { case mt ~ ui ~ un =>
      FetchMessage(mt, User(ui, un))
    }(m => (((m.value), m.from.id), m.from.name))
}
```
What's unique about this codec, is we are directly passing `userId` and `userName` to `User` within `FetchMessage`, it also produces a nested tuple in the decode function. To use our codec we'll create a `SELECT` statement, which means we'll be using a `Query`.

```scala
private object SqlCommands {
  ...
  val getMessage: Query[RoomId, FetchMessage] =
    sql"""
            SELECT messages.message, users.id, users.name 
            FROM messages 
            INNER JOIN users 
            ON messages.user_id = users.id 
            WHERE messages.room_id = $roomId;
        """.query(fetchmessagecodec)
}
```
In this `SELECT` statement, we are selecting the `message` from the `messages` table and `id` and `name` from the `users` table by using an `INNER JOIN` with the id values. The results are filtered by `RoomId` using the `WHERE` clause to produce a `Query[RoomId, FetchMessage] `.

We can now use this in our function:

```scala
...
new PostgresProtocol[F] {
  ...
  override def fetchMessages(roomId: RoomId): F[Stream[F, FetchMessage]] =
    session.prepare(getMessage).map { cmd =>
      cmd.stream(roomId, 32)
    }
}.pure[F]
```
Here we use the `stream` method on `cmd`, passing it the roomId and `chunckSize` of `32`, this returns an `F[Stream[F, FetchMessage]]`. The `stream` method repeatedly emits chunks of data from Postgres based on our query.

Finally, let's implement the last function of `PostgresProtocol`. The `deleteRoomMessages()`, is a function that deletes messages by `RoomId`, this requires a `Command[RoomId]`.

```scala
private object SqlCommands {
  ...
    val delMessages: Command[RoomId] =
      sql"""
              DELETE FROM messages 
              WHERE room_id = $roomId
          """.command
}
```
The usage is similar to `deleteUserOrRoom()` with the help of the `execute` method:
```scala
...
new PostgresProtocol[F] {
  ...
  override def deleteRoomMessages(roomId: RoomId): F[Unit] =
    session.prepare(delMessages).flatMap { cmd =>
      cmd.execute(roomId).void
    }
}.pure[F]
```

## 6. Redis and Chat Protocols

Before we dive into the Redis implementation, we'll need an overview of how the schema will be. 
A Redis hash is a record type structured as a collection of field-value pairs, we'll need the following hashes for our application:

The `users` hash will be a collection of UUID string fields mapped to user name values.
The `rooms` hash will be a collection of UUID string fields mapped to room name values.
The `userroomid` hash will be a collection of user UUID string fields mapped to and room UUID string values.

A Redis set is a collection of unique string values, we'll need a Redis set for each `room` created in our application:
The sets will be named in the format, `room:<room id>`, and will be a collection of user UUID strings.

We'll be creating both the Redis Algebra and Chat Algebra in this section so that we can follow why each Redis function is needed.

First, we'll start by creating the Redis Algebra for our application, in the `websockets` folder create a `RedisPotocol.scala` file with the following contents:

```scala
package rockthejvm.websockets
import rockthejvm.websockets.domain.user.*
import rockthejvm.websockets.domain.room.*

trait RedisProtocol[F[_]] {
  def createUser(user: User): F[Unit]
  def createRoom(room: Room): F[Unit]
  def getRoomFromName(roomname: String): F[Option[Room]]
  def getUsersRoomId(user: User): F[Option[RoomId]]
  def usernameExists(username: String): F[Boolean]
  def listUserIds(roomid: RoomId): F[Set[UserId]]
  def listRooms: F[List[String]]
  def roomExists(roomid: RoomId): F[Boolean]
  def mapUserToRoom(userid: UserId, roomid: RoomId): F[Unit]
  def addUserToRoom(userid: UserId, roomid: RoomId): F[Unit]
  def removeUserFromRoom(roomid: RoomId, userid: UserId): F[Unit]
  def deleteUserRoomMapping(userid: UserId): F[Unit]
  def deleteRoom(roomid: RoomId): F[Long]
  def deleteUser(userid: UserId): F[Long]
  def getSelectedUsers(
      userid: UserId,
      rest: List[UserId]
  ): F[Option[List[User]]]
  def chatState: F[String]
}
```
Just like with Postgres, we'll be implementing each of these functions, and by the end of this section, we will have learned alot about Redis and the redis4cats package.

We'll also need to create the companion object for `RedisProtocol` along with a `make()` function.

```scala
import dev.profunktor.redis4cats.RedisCommands
 ...

object RedisProtocol {
  def make[F[_]](
      redis: RedisCommands[F, String, String]
  ): F[RedisProtocol[F]] = {
    new RedisProtocol[F] {
      ???
    }.pure[F]
  }
}
```
The `make()` function takes an argument, `redis` of type `RedisCommands[F, String, String]` as an argument. This contains all the methods for interacting with Redis, and returns an `F[RedisProtocol[F]]`.

Let's also create `ChatProtocol.scala` in the same folder and add the following code:

```scala
package rockthejvm.websockets

import rockthejvm.websockets.domain.user.*

trait ChatProtocol[F[_]] {
  def register(name: String): F[OutputMessage]
  def enterRoom(user: User, room: String): F[List[OutputMessage]]
  def chat(user: User, text: String): F[List[OutputMessage]]
  def help(user: User): F[OutputMessage]
  def listRooms(user: User): F[List[OutputMessage]]
  def listMembers(user: User): F[List[OutputMessage]]
  def disconnect(maybeuser: Option[User]): F[List[OutputMessage]]
  def chatState: F[String]
}
```
Similarly, we'll also need a companion object:

```scala
object ChatProtocol {

  def make[F[_]](
      redisP: RedisProtocol[F],
      postgresP: PostgresProtocol[F]
  ): F[ChatProtocol[F]] = {
    new ChatProtocol[F] {
      ???
    }.pure[F]
  }
}
```

### 6.1. Registering Users

Let's start with the `register()` function in `ChatProtocol`. To register someone, we'll need to first check if the user name exists in Redis, then add it to both the Redis and Postgres databases:

```scala
new RedisProtocol[F] {
  ...
  override def usernameExists(username: String): F[Boolean] = {
    redis.hVals("users").map { nameslist =>
      nameslist.exists(u => u.toLowerCase == username.toLowerCase)
    }
  }
}.pure[F]
```
The `usernameExists()` function takes a username of type `String` and returns an `F[Boolean]`. Here we use the `hVals()` function, which takes the name of the hash, and returns an `F[List[String]]` containing the list of values. We map on this value and call the `exists()` method which returns a `Boolean` depending on if a match is found or not. 

Next we'll implement the `createUser()`, and `createRoom()` functions in `RedisProtocol[F]` since they are very similar:

```scala
import cats.syntax.all.*
import cats.Monad

object RedisProtocol {
  def make[F[_]: Monad](
      redis: RedisCommands[F, String, String]
  ): F[RedisProtocol[F]] = {
    new RedisProtocol[F] {
      override def createUser(user: User): F[Unit] = {
        redis.hSet("users", user.toMap).void
      }

      override def createRoom(room: Room): F[Unit] = {
        redis.hSet("rooms", room.toMap).void
      }
    }.pure[F]
  }
}
```
Here we use the `hSet()` function to create the `users` and `rooms` Redis hashes. The method takes a key value which we supplied as `users` and `rooms`, these are the names of the hashes, the second argument is the `field` which is of type `Map[String, String]`.

Notice the new `toMap()` function on `Room` and `User`, that we used to create the `Map`. Let's implement them next:

```scala
  //in user.scala
  case class User(id: UserId, name: UserName) {
    def toMap = Map((id.id.toString, name.name))
  }

  //in room.scala
  case class Room(id: RoomId, name: RoomName) {
    def toMap = Map((id.id.toString, name.name))
  }
```
The `toMap` method creates a `Map` with a tuple of the `UUID` string and the name from `UserName` and `RoomName` respectively. We update these methods in `user.scala` and `room.scala`.

Now back the `ChatProtocol[F]` `register()` function:

```scala
import cats.Monad
import cats.syntax.all.*
import cats.effect.std.UUIDGen
import cats.effect.kernel.Concurrent

object ChatProtocol {
  def make[F[_]: Monad: UUIDGen: Concurrent](
      redisP: RedisProtocol[F],
      postgresP: PostgresProtocol[F]
  ): F[ChatProtocol[F]] = {
    new ChatProtocol[F] {
      override def register(name: String): F[OutputMessage] = {
        for {
          userExists <- redisP.usernameExists(name)
          maybeUser <- 
            if(userExists == true) {
              Left("User name exists").pure[F]
            } else {
              postgresP.createUser(name)
            }
          om <- maybeUser match {
            case Right(u: User) =>
              redisP.createUser(u) *>
                SuccessfulRegistration(u).pure[F]
            case Left(err: String) =>
              ParsingError(None, err)
                .pure[F]
          }
        } yield om
      }
    }.pure[F]
  }
}
```
We start by checking if the user name exists in Redis, if this is true we return a `Left("User name exists").pure[F]` other wise we create the user in Postgres by calling `postgresP.createUser(name)`.

If the creation was successful, we also create the user in Redis by calling `redisP.createUser(u)` which returns `SuccessfulRegistration(u).pure[F]`. In case of an error we return, `ParsingError(None, err).pure[F]`.

### 6.2. Entering a Room

To enter a room, we'll need to first get the user's current room, and then make the transfer to the new room. We'll receive the room name inform of a `String`, which we'll use to check against the Redis database:

```scala
import java.util.UUID
...

new RedisProtocol[F] {
  ...
  override def getRoomFromName(roomname: String): F[Option[Room]] = {
    redis.hGetAll("rooms").flatMap { rmap =>
      rmap.find(_._2 == roomname) match {
        case Some((i, n)) =>
          Room(
            UUID
              .fromString(i),
            n
          )
            .map(_.toOption)
        case None => None.pure[F]
      }
    }
  }
}.pure[F]
```
Here the `getRoomFromName()` function that takes a `roomname` of type `String` and returns an `F[Option[Room]]`, we use the `hGetAll("rooms")` method that returns a `Map` of all the key-values pairs in the `rooms` hash of type `F[Map[String, String]]`. We `flatMap` of this value and call `find()` on the `Map`, which returns an `Option` based on the predicate.

If we get a `Some`, we return an `F[Validated[String, Room]]` from the `apply` method on `Room`. We map this value and convert it to an `Option` with the `toOption` method. We can't get an `Invalid` since these values were verified before we put them into Redis. If we get a `None` were return a `None.pure[F]`. 

We'll also need the user's current roomid, so that we transfer the user from there:

```scala
new RedisProtocol[F] {
  ...
  override def getUsersRoomId(user: User): F[Option[RoomId]] = {
    redis.hGet("userroomid", user.id.id.toString).map {
      case Some(roomid) => Some(RoomId(UUID.fromString(roomid)))
      case None         => None
    }
  }
}.pure[F]
```
The `getUsersRoomId()` function takes a `User` and returns the `RoomId` of the `Room` a `User` is in. Here we use the `hGet()` function, which we supply with the user's id to return the room id from the `userroomid` hash. If we get a `Some`, we return a `Some` of `RoomId`. Otherwise, we return a `None`.

Now we can implement the enterroom() function in `ChatProtocol`:

```scala
new ChatProtocol[F] {
  ...
  override def enterRoom(
      user: User,
      room: String
  ): F[List[OutputMessage]] = {
    redisP.getRoomFromName(room).flatMap {
      case Some(r) =>
        redisP.getUsersRoomId(user).flatMap {
          case Some(usersroomid) =>
            if (usersroomid == r.id) {
              List(
                SendToUser(
                  user,
                  s"You are already in the ${r.name.name} room"
                )
              ).pure[F]
            } else {
              transferUserToRoom(redisP, postgresP, user, r)
            }
          case None =>
            addToRoom(redisP, postgresP, user, r)
        }
      case None =>
        createRoom(redisP, postgresP, room).flatMap {
          case Right(r) =>
            transferUserToRoom(redisP, postgresP, user, r)
          case Left(err) =>
            List(
              ParsingError(
                Some(user),
                err
              )
            ).pure[F]
        }

    }
  }
}.pure[F]
```
The `enterroom()` function takes a `user` of type `User` and a `room` of type `String` and returns an `F[List[OutputMessage]]`. There are several new functions here which we'll implement later.

We start by calling `redisP.getRoomFromName(room)`, if the room exists, we compare that RoomId with the user's `RoomId` which we get from `redisP.getUsersRoomId(user)`. If they are the same we inform the user, otherwise we transfer the user by calling `transferUserToRoom(redisP, postgresP, user, r)`.

If the user doesn't have a `RoomId`, we add the user to the requested room by calling `addToRoom(redisP, postgresP, user, r)`.
Now, if the room name doesn't exist, we create that room by calling `createRoom(redisP, postgresP, room)`, then transfer the user to the new room.

### 6.3. Moving to Another Room

This is a private function, that transfers the user to a new room:

```scala
object ChatProtocol {
  ...
  private def transferUserToRoom[F[_]: Monad: UUIDGen: Concurrent](
      redisP: RedisProtocol[F],
      postgresP: PostgresProtocol[F],
      user: User,
      room: Room
  ): F[List[OutputMessage]] =
    val leaveMessages = removeFromCurrentRoom(redisP, postgresP, user)
    val enterMessages = addToRoom(redisP, postgresP, user, room)
    for {
      leave <- leaveMessages
      enter <- enterMessages
    } yield leave ++ enter
}
```
To do this we remove the user from the current room by calling `removeFromCurrentRoom()`, then add the user to the requested room by calling `addToRoom(redisP, postgresP, user, room)`. This happens both in Redis and Postgres to keep everything in sync.

### 6.4. Removing a User from a Room

To remove a user from the current room, we'll need to remove the user from the Redis `room:<roomid>` set and remove the entry from the `userroomid` hash. 

Note that in Redis when the last member is deleted from a set, the entire set is deleted, therefore if this occurs, we'll also need to update the `rooms` hash:

```scala
new RedisProtocol[F] {
  ...
  override def removeUserFromRoom(
      roomid: RoomId,
      userid: UserId
  ): F[Unit] = {
    redis.sRem(s"room:${roomid.id.toString}", userid.id.toString).void
  }
}.pure[F]
```
The `removeUserFromRoom()` uses the `sRem` function to delete the user from the `room:<roomid>` set. It requires the room name and the user id:

```scala
new RedisProtocol[F] {
  ...
  override def deleteUserRoomMapping(userid: UserId): F[Unit] = {
    redis.hDel("userroomid", userid.id.toString).void
  }

  override def deleteRoom(roomid: RoomId): F[Long] = {
    redis.hDel("rooms", roomid.id.toString)
  }
}.pure[F]
```
The `deleteUserRoomMapping()` and `deleteRoom()`  uses the `hDel` function to delete an entry from the `userroomid` and `rooms` hashes, and only requires the user id and room id respectively.

Since Redis automatically deletes empty sets, we'll have to check if the `room:<roomid>` set still exists:

```scala
new RedisProtocol[F] {
  ...
  override def roomExists(roomid: RoomId): F[Boolean] = {
    redis.exists(s"room:${roomid.id.toString}")
  }
}.pure[F]
```
We use the `exists` function which checks if a key exists within redis.

Finally, we can implement the removeFromCurrentRoom() function:

```scala
object ChatProtocol {
  ...
  private def removeFromCurrentRoom[F[_]: Monad: UUIDGen](
        redisP: RedisProtocol[F],
        postgresP: PostgresProtocol[F],
        user: User
    ): F[List[OutputMessage]] = {
      redisP.getUsersRoomId(user).flatMap {
        case Some(roomid) =>
          for {
            _ <- redisP.removeUserFromRoom(roomid, user.id)
            _ <- redisP.roomExists(roomid).flatMap { b =>
              println(b)
              if (b == true) { ().pure[F] }
              else {
                redisP.deleteRoom(roomid) *>
                postgresP.deleteRoomMessages(roomid) *>
                  postgresP.deleteRoom(roomid)    
              }
            }
            _ <- redisP.deleteUserRoomMapping(user.id)
            om <- broadcastMessage(
              redisP,
              roomid,
              SendToUser(user, s"${user.name.name} has left the room")
            )
          } yield om
        case None =>
          List.empty[OutputMessage].pure[F]
      }
    }
}
```
Here we start by getting the user's current roomid by calling `redisP.getUsersRoomId(user)`, if the room doesn't exist we return `List.empty[OutputMessage].pure[F]` otherwise we use a for comprehension to do the following.

1. Remove the user from the `room:<roomid>` set by calling `redisP.removeUserFromRoom(roomid, user.id)`
1. Check if the room still exists by calling `redisP.roomExists(roomid)`. 
1. If it still exists, we return `().pure[F]` otherwise we delete the entry from the `rooms` hash, then delete those room messages and the room from Postgres by calling `postgresP.deleteRoomMessages(roomid)` and `postgresP.deleteRoom(roomid)` respectively.
1. We also delete the entry from `userroomid` by calling r`edisP.deleteUserRoomMapping(user.id)`
1. Finally we tell the old room member that the user has left the room.

### 6.5. Broadcasting a Message

To broadcast messages to members in a room, we first need to retrieve a list of user ids from a `room:<roomid>` set:
```scala
new RedisProtocol[F] {
  ...
  override def listUserIds(roomid: RoomId): F[Set[UserId]] = {
    redis.sMembers(s"room:${roomid.id.toString}").map { set =>
      set.map(id => UserId(UUID.fromString(id)))
    }
  }
}.pure[F]
```
The `listUserIds()` function takes a `RoomId` and returns an `F[Set[UserId]]`. Here we use the `sMembers()` function which returns all the members in a Redis set, it takes the name of the set, which we supplied as `room:<roomid>`. This returns an `F[Set[String]]` which we convert to a `Set[UserId]` using the `map` method.

Next we'll need to somehow convert those `UserId`s into `User`s:

```scala
new RedisProtocol[F] {
  ...
  override def getSelectedUsers(
      userid: UserId,
      rest: List[UserId]
  ): F[Option[List[User]]] = {
    val idMap: F[Map[String, String]] = 
      if (rest.isEmpty) {
        redis
          .hmGet(
            "users",
            userid.id.toString
          )
      } else {
        redis
          .hmGet(
            "users",
            userid.id.toString,
            rest.map(_.id.toString): _*
          )
      }

    idMap.flatMap { usermap =>
      usermap.toList.map { (id, name) =>
        User(UUID.fromString(id), name).map(_.toOption)
      }.sequence
    } map (_.sequence)
  }
}.pure[F]
```
Here we use the `hmGet()` function that returns multiple values associated with `UserId`s from the `users` hash. It takes the hash name, and one or more field values and returns an `F[Map[String, String]]`. 
If `rest` is empty we pass only `userid`, otherwise we pass both `rest`, and `userid` to `hmGet()`. We then `flatMap()` on `idMap`, convert it to a list and map each value to a `User`.
This produces a `List[F[Option[User]]]`, so we call `sequence` twice to convert it into an `F[Option[List[User]]]`.

Finally, let's implement the `broadcastMessage()` function.

```scala
object ChatProtocol {
  ...
  private def broadcastMessage[F[_]: Monad: UUIDGen](
      redisP: RedisProtocol[F],
      roomid: RoomId,
      om: OutputMessage
  ): F[List[OutputMessage]] = {
    redisP.listUserIds(roomid).flatMap { uset =>
      val userlist = uset.toList
      if(userlist.isEmpty){
        List.empty[OutputMessage].pure[F]
      } else{
        redisP.getSelectedUsers(userlist.head, userlist.tail).map { maybelist =>
          maybelist match {
            case Some(ulist) =>
              ulist.map { u =>
                om match {
                  case SendToUser(user, msg)  => SendToUser(u, msg)
                  case ChatMsg(from, to, msg) => ChatMsg(from, u, msg)
                  case _                      => DiscardMessage
                }
              }
            case None => List.empty[OutputMessage]
          }
        }
      }

    }
  }
}
```
We start by getting a list of user ids by calling `redisP.listUserIds(roomid)`, if its empty, we return `List.empty[OutputMessage].pure[F]`.

Otherwise, we retrieve the list of `User`s by calling `redisP.getSelectedUsers(userlist.head, userlist.tail)`. The rest of the implementation hasn't changed from before.

### 6.6. Adding a User to a Room

This function adds a user to a room, however, in Redis we'll need to add the user id to the `room:<roomid>` set, and add the `userid -> roomid` pair to the `userroomid` hash:

```scala
new RedisProtocol[F] {
  ...
  override def addUserToRoom(userid: UserId, roomid: RoomId): F[Unit] = {
    redis.sAdd(s"room:${roomid.id.toString}", userid.id.toString).void
  }
}.pure[F]
```
The `addUserToRoom()` function takes a `UserId`, and  `RoomId` and returns an F[Unit]. Here we use the `sAdd` function to add the userid to the `room:<roomid>` redis set. 

Next, the `mapUserToRoom()` function adds a `userid -> roomid` pair to the `userroomid` redis hash:
```scala
new RedisProtocol[F] {
  ...
  override def mapUserToRoom(userid: UserId, roomid: RoomId): F[Unit] = {
    val urmap: Map[String, String] = Map(
      (userid.id.toString, roomid.id.toString)
    )
    redis.hSet("userroomid", urmap).void
  }
}.pure[F]
```
The `mapUserToRoom()` function takes a `userid` and `roomid` and returns an `F[Unit]`. First, we create a `urmap`, which is a `Map` of these `String` values, then call the `hSet` method to add this pair to the `userroomid` hash.

Finally, here's the `addToRoom()` function:

```scala
object ChatProtocol {
  ...
  private def addToRoom[F[_]: Monad: UUIDGen: Concurrent](
      redisP: RedisProtocol[F],
      postgresP: PostgresProtocol[F],
      user: User,
      room: Room
  ): F[List[OutputMessage]] = {
    for {
      _ <- redisP.addUserToRoom(user.id, room.id)
      _ <- redisP.mapUserToRoom(user.id, room.id)
      previousMessages <- fetchRoomMessages(postgresP, room.id, user)
      om <- broadcastMessage(
        redisP,
        room.id,
        SendToUser(
          user,
          s"${user.name.name} has joined the ${room.name.name} room"
        )
      )
    } yield previousMessages ++ om
  }
}
```
Here we add the user to the `room:<roomid>` set, then add the pair to the `userroomid` hash, and finally call `fetchRoomMessages(postgresP, room.id, user)` to retrieve all the previous messages from postgres, we'll be implementing this next.

We then inform all the room members that the new user has joined the room and pass all the previous messages to the new user.

### 6.7. Fetching Current Messages

```scala
object ChatProtocol {
  ...
  private def fetchRoomMessages[F[_]: FlatMap: Concurrent](
    postgresP: PostgresProtocol[F],
    roomid: RoomId,
    user: User
  ): F[List[OutputMessage]] = 
    postgresP
      .fetchMessages(roomid)
      .flatMap {
        _.map { case FetchMessage(msg, from) =>
          ChatMsg(from, user, msg.value)
        }.compile.toList
      }
}
```
To fetch messages from Postgres, we call `postgresP.fetchMessages(roomid)`, this returns an `F[Stream[F, FetchMessage]]` which we map on to convert to `ChatMsg`, we then compile to list to return an `F[List[OutputMessage]]`.

### 6.8. Creating a Chat Room

We already looked at the `createRoom()` redis implementation in the `register()` function section, now let's look at how to implement it in `ChatProtocol`:

```scala
object ChatProtocol {
  ...
  private def createRoom[F[_]: Monad](
      redisP: RedisProtocol[F],
      postgresP: PostgresProtocol[F],
      room: String
  ): F[Either[String, Room]] =
    postgresP.createRoom(room).flatMap {
      case v @ Right(r) =>
        redisP.createRoom(r) *>
          v.pure[F]
      case l @ Left(err) => l.pure[F]
    }
}
```
Here we start by calling `postgresP.createRoom(room)` which returns an `F[Either[String, Room]]`, if the creation is successful, we call `redisP.createRoom(r)` then return a `Right(r)` otherwise we return a `Left(err)`.

### 6.9. Sending Messages

When we receive a chat message, we'll need to save it into Postgres and then broadcast it:

```scala
import java.time.LocalDateTime
...
new ChatProtocol[F] {
  ...
  override def chat(user: User, text: String): F[List[OutputMessage]] = {
    redisP.getUsersRoomId(user).flatMap {
      case Some(roomid) =>
        postgresP.saveMessage(text, LocalDateTime.now(), user.id, roomid) *>
          broadcastMessage(redisP, roomid, ChatMsg(user, user, text))
      case None =>
        List(SendToUser(user, "You are not currently in a room")).pure[F]
    }
  }
}.pure[F]
```
We start by getting the user's roomid, then calling `postgresP.saveMessage()` followed by the `broadcastMessage()` function. In case the user has no room id, we inform the user by returning `List(SendToUser(user, "You are not currently in a room")).pure[F]`.
 
### 6.10. The Help Prompt

The implementation of this function remains unchanged from before:

```scala
new ChatProtocol[F] {
  ...
  override def help(user: User): F[OutputMessage] = {
    val text = """Commands:
                    | /help             - Show this text
                    | /room <room name> - Change to specified room
                    | /rooms            - List all rooms
                    | /members          - List members in current room
                """.stripMargin
    SendToUser(user, text).pure[F]
  }
}.pure[F]
```

### 6.11. Listing Rooms

Here we'll need to retrieve a list of rooms from Redis:
```scala
new RedisProtocol[F] {
  ...
  override def listRooms: F[List[String]] = {
    redis.hVals("rooms")
  }
}.pure[F]
```
We do this through the `hVals` method which returns all the values from the `rooms` hash:

```scala
new ChatProtocol[F] {
  ...
  override def listRooms(user: User): F[List[OutputMessage]] = {
    redisP.listRooms.map { rooms =>
      val roomList = rooms.toList.sorted.mkString("Rooms:\n\t", "\n\t", "")
      List(SendToUser(user, roomList))
    }
  }
}.pure[F]
```
The `listRooms` `ChatProtocol[F]` method starts by calling `redisP.listRooms` function, then we sort and make a String from the resulting list and finally pass this value to the `SendToUser()` apply method.

### 6.12. Listing Members

This function gets the list of users in the room the user is in.
```scala
new ChatProtocol[F] {
  ...
  override def listMembers(user: User): F[List[OutputMessage]] = {
    val membersList: F[String] = redisP.getUsersRoomId(user).flatMap {
      case Some(roomid) =>
        redisP.listUserIds(roomid).flatMap { u =>
          redisP.getSelectedUsers(u.toList.head, u.toList.tail).map {
            case Some(lu) =>
              lu.map(_.name.name)
                .sorted
                .mkString("Room Members:\n\t", "\n\t", "")
            case None => ""
          }
        }
      case None => "You are not currently in a room".pure[F]
    }
    membersList.map(mlist => List(SendToUser(user, mlist)))
  }
}.pure[F]
```
We start by calling `redisP.getUsersRoomId(user)` to get the roomid of the user, if the user has no roomid, we return `You are not currently in a room".pure[F]`. 

Otherwise, we pass the roomid to `redisP.listUserIds(roomid)` to get a list of member ids which we pass to `redisP.getSelectedUsers(u.toList.head, u.toList.tail)` and convert to `User`s.

Finally, we produce a string of user names which we sequentially pass to the `SendToUser()` apply method

### 6.13. Disconnecting

We'll need to be able to remove a user from the `users` hash in Redis before we implement disconnect:

```scala
new RedisProtocol[F] {
  ...
  override def deleteUser(userid: UserId): F[Long] = {
    redis.hDel("users", userid.id.toString)
  }
}.pure[F]
```
Here we use the `hDel` function similarly to `deleteRoom()` that we implemented previously:

```scala
new ChatProtocol[F] {
  ...
  override def disconnect(
      maybeuser: Option[User]
  ): F[List[OutputMessage]] = {
    maybeuser match {
      case Some(user) =>
        postgresP.deleteUser(user.id) *>
          redisP.deleteUser(user.id) *>
          removeFromCurrentRoom(redisP, postgresP, user)
      case None => List.empty[OutputMessage].pure[F]   
    }
  }
}.pure[F]
```
We first delete the user from Postgres and Redis by calling `postgresP.deleteUser(user.id)` and `redisP.deleteUser(user.id)` then we finally call `removeFromCurrentRoom()` to remove the user from the current room.

### 6.14. Getting State 

This is the last function to implement, however, to get an overview from Redis, quite a lot has to be done:

```scala
new RedisProtocol[F] {
  ...
  override def chatState: F[String] = {
    for {
      maybeusers <- redis.hLen("users")
      mayberooms <- redis.hLen("rooms")
      roomIds <- redis.hKeys("rooms")
      usersPerRoom <- roomIds.traverse { id =>
        redis.sMembers(s"room:$id").flatMap { uSet =>
          redis.hGet("rooms", id).flatMap {
            case Some(r) =>
              val uids = uSet.map(id => UserId(UUID.fromString(id))).toList
              getSelectedUsers(uids.head, uids.tail)
                .map {
                  _.getOrElse(List.empty[User])
                    .map(_.name.name)
                    .mkString(s"$r Room Members:\n\t", "\n\t", "")
                }
            case None =>
              s"An error occured while fetching rooms".pure[F]
          }
        }
      }
    } yield s"""
                          |<!Doctype html>
                          |<title>Chat Server State</title>
                          |<body>
                          |<pre>Users: ${maybeusers.getOrElse("")}</pre>
                          |<pre>Rooms: ${mayberooms.getOrElse("")}</pre>
                          |<pre>Overview: 
                          |${usersPerRoom.mkString}
                          |</pre>
                          |</body>
                          |</html>
                    """.stripMargin
  }
}.pure[F]
```
From the top:
1. We call `hLen("users")` to get the number of users currently, returning `maybeusers` as an `Option[Long]`
1. We call `hLen("rooms")` to get the number of rooms currently, returning `mayberooms` as an `Option[Long]`
1. We call `hKeys("rooms")` to get a list of all the room ids and return roomIds as a `List[String]`
1. We then traverse on `roomIds` and for each roomid we get a `Set[String]`, (`uSet`) containing user ids
1. Next, we convert these user ids to `UserId`s forming a `List[UserId]` as `uids`
1. Then we call `getSelectedUsers(uids.head, uids.tail)` to retrieve a list of `User`'s and convert that to a `String`.
1. This forms a `List[String]` of users per room, `usersPerRoom`
1. In the yield section, we get the number of users and rooms by calling `maybeusers.getOrElse("")` and `mayberooms.getOrElse("")` respectively
1. Finally we convert `usersPerRoom` into a string as well

This whole function returns an `F[String]`:

```scala
new ChatProtocol[F] {
  ...
  override def chatState: F[String] = redisP.chatState
}.pure[F]
```
Lastly in ChatProtocol, we simply call `redisP.chatState`

## 7. Getting Input

Let's create an `InputMessage.scala` file in the websocket folder and add the following contents:

```scala
package rockthejvm.websockets

import cats.effect.kernel.Ref
import rockthejvm.websockets.domain.user.*

trait InputMessage[F[_]] {
  def parse(
      userRef: Ref[F, Option[User]],
      text: String
  ): F[List[OutputMessage]]
}
```
Here the `InputMessage[F[_]]`, trait contains a parse method whose signature remains unchanged, however now we went ahead and removed `defaultRoom`:

```scala
import cats.Monad
import cats.syntax.all.*

...
case class TextCommand(left: String, right: Option[String])

object InputMessage {
  def make[F[_]: Monad](
      chatP: ChatProtocol[F]
  ): F[InputMessage[F]] = {
    new InputMessage[F] {
      override def parse(
          userRef: Ref[F, Option[User]],
          text: String
      ): F[List[OutputMessage]] = {
        text.trim match {
          case "" => List(DiscardMessage).pure[F]
          case txt =>
            userRef.get.flatMap {
              case Some(user) => procesText4Reg(user, txt, chatP)
              case None => processText4UnReg(txt, chatP, userRef)
            }
        }
      }
    }.pure[F]
  }
}
```
The main difference here is that we have replaced `Protocol[F]` with `ChatProtocol[F]`, and since we don't have default room anymore we directly `flatMap` on `userRef.get` to produce either a `Some` or a `None`:

```scala
...
import cats.parse.Parser
import cats.parse.Parser.char
import cats.parse.Rfc5234.{alpha, sp, wsp}

...
object InputMessage {
  ...
  private def commandParser: Parser[TextCommand] = {
    val leftSide = (char('/').string ~ alpha.rep.string).string
    val rightSide: Parser[(Unit, String)] = sp ~ alpha.rep.string
    ((leftSide ~ rightSide.?) <* wsp.rep.?).map((left, right) =>
      TextCommand(left, right.map((_, s) => s))
    )
  }

  private def parseToTextCommand(
      value: String
  ): Either[Parser.Error, TextCommand] = {
    commandParser.parseAll(value)
  }
}
```
These two functions remain the same as before.

```scala
object InputMessage {
  ...
  private def processText4UnReg[F[_]: Monad](
      text: String,
      chatP: ChatProtocol[F],
      userRef: Ref[F, Option[User]]
  ): F[List[OutputMessage]] = {
    if (text.charAt(0) == '/') {
      parseToTextCommand(text).fold(
        _ =>
          List(
            ParsingError(
              None,
              "Characters after '/' must be between A-Z or a-z"
            )
          ).pure[F],
        v =>
          v match {
            case TextCommand("/name", Some(n)) =>
              chatP.register(n).flatMap{
                case sr @ SuccessfulRegistration(u,_) => 
                  for {
                    _ <- userRef.update(_ => Some(u))
                    om <- chatP.enterRoom(u, "Default")
                    help <- chatP.help(u)
                  } yield sr :: (help :: om)
                case ParsingError(None, err) =>
                  List(ParsingError(None, err)).pure[F]
                case _ =>  List(DiscardMessage).pure[F]
              }
            case _ =>
              List(UnsupportedCommand(None)).pure[F]
          }
      )
    } else {
      List(Register(None)).pure[F]
    }
  }
}
```
Here the changes in the `processText4UnReg()` function occur under `TextCommand("/name", Some(n))`, since we now call `chatP.register(n)` passing it the user name. 

In the case of `SuccessfulRegistration` we update the `userRef`, and call `chatP.enterRoom(u, "Default")` to enter the `Default` room and send the user the welcome message and help menu. The rest of the logic remains unchanged:

```scala
...
import cats.Applicative

object InputMessage {
  ...
  private def procesText4Reg[F[_]: Applicative](
      user: User,
      text: String,
      chatP: ChatProtocol[F],
  ): F[List[OutputMessage]] = {
    if (text.charAt(0) == '/') {
      parseToTextCommand(text).fold(
        _ =>
          List(
            ParsingError(
              None,
              "Characters after '/' must be between A-Z or a-z"
            )
          ).pure[F],
        v =>
          v match {
            case TextCommand("/name", Some(n)) =>
              List(ParsingError(Some(user), "You can't register again")).pure[F]
            case TextCommand("/room", Some(r)) =>
              chatP.enterRoom(user, r)
            case TextCommand("/help", None) =>
              chatP.help(user).map(List(_))
            case TextCommand("/rooms", None) =>
              chatP.listRooms(user)
            case TextCommand("/members", None) =>
              chatP.listMembers(user)
            case _ => List(UnsupportedCommand(Some(user))).pure[F]
          }
      )
    } else {
      chatP.chat(user, text)
    }
  }
}
```
The `procesText4Reg()` function also remains unchanged except for the new `ChatProtocol[F]` methods

## 8. The Web App Routes

In this section we'll continue to upgrade our application to the new `ChatProtocol[F]`:

```scala
package rockthejvm.websockets

import fs2.io.file.Files
import cats.effect.Temporal
import org.http4s.dsl.Http4sDsl
import org.http4s.server.websocket.WebSocketBuilder2
import cats.effect.std.Queue
import fs2.concurrent.Topic
import org.http4s.{HttpApp, HttpRoutes}
import org.http4s.StaticFile
import cats.syntax.all.*
import org.http4s.headers.`Content-Type`
import org.http4s.MediaType

class Routes[F[_]: Files: Temporal] extends Http4sDsl[F] {
  def service (
      wsb: WebSocketBuilder2[F],
      q: Queue[F, OutputMessage],
      t: Topic[F, OutputMessage],
      im: InputMessage[F],
      chatP: ChatProtocol[F]
  ): HttpApp[F] = {
    HttpRoutes.of[F] {
      case request @ GET -> Root / "chat.html" =>
        StaticFile
          .fromPath(
            fs2.io.file.Path("src/main/resources/chat.html"),
            Some(request)
          )
          .getOrElseF(NotFound())

      case GET -> Root / "metrics" =>
        def currentState: F[String] = {
          chatP.chatState
        }

        currentState.flatMap { currState =>
          Ok(currState, `Content-Type`(MediaType.text.html))
        }
    }
  }
}
```
The `/chat.html` route remains unchanged, however, the `/metrics` now produces the state of the application from `chatP.chatState`:

```scala
import cats.effect.kernel.Ref
import rockthejvm.websockets.domain.user.User
...
    HttpRoutes.of[F] {
      ...
      case GET -> Root / "ws" =>
        for {
          uRef <- Ref.of[F, Option[User]](None)
          uQueue <- Queue.unbounded[F, OutputMessage]
          ws <- wsb.build(
            send(t, uQueue, uRef),
            receive(chatP, im, uRef, q, uQueue)
          )
        } yield ws
    }
```
Under the `/ws` once again we replace `protocol` with `chatP` in the `recieve()` handle. Everything else remains unchanged.

```scala
...
import fs2.Stream
import org.http4s.websocket.WebSocketFrame
import scala.concurrent.duration.*

class Routes[F[_]: Files: Temporal] extends Http4sDsl[F] {
  ...
  private def handleWebSocketStream(
      wsf: Stream[F, WebSocketFrame],
      im: InputMessage[F],
      chatP: ChatProtocol[F],
      uRef: Ref[F, Option[User]]
  ): Stream[F, OutputMessage] = {
    for {
      sf <- wsf
      maybeuser <- Stream.eval(uRef.get)
      om <- Stream.evalSeq(
        sf match {
          case WebSocketFrame.Text(text, _) =>
            im.parse(uRef, text)
          case WebSocketFrame.Close(_) =>
            chatP.disconnect(maybeuser)
        }
      )
    } yield om
  }

  private def receive(
      chatP: ChatProtocol[F],
      im: InputMessage[F],
      uRef: Ref[F, Option[User]],
      q: Queue[F, OutputMessage],
      uQueue: Queue[F, OutputMessage]
  ): Pipe[F, WebSocketFrame, Unit] = { wsf =>
    handleWebSocketStream(wsf, im, chatP, uRef)
      .evalMap { m =>
        uRef.get.flatMap {
          case Some(_) =>
            q.offer(m)
          case None =>
            uQueue.offer(m)
        }
      }
      .concurrently {
        Stream
          .awakeEvery(30.seconds)
          .map(_ => KeepAlive)
          .foreach(uQueue.offer)
      }
  }
}
```
Here we now use the new `ChatProtocol[F]`, for both `handleWebSocketStream()` and `receive()` functions:

```scala
import io.circe.generic.auto.*
import io.circe.syntax.*

class Routes[F[_]: Files: Temporal] extends Http4sDsl[F] {
  ...
  private def filterMsg(
      msg: OutputMessage,
      userRef: Ref[F, Option[User]]
  ): F[Boolean] = {
    msg match {
      case DiscardMessage => false.pure[F]
      case sendtouser @ SendToUser(_, _) =>
        userRef.get.map { _.fold(false)(u => sendtouser.forUser(u)) }
      case chatmsg @ ChatMsg(_, _, _) =>
        userRef.get.map { _.fold(false)(u => chatmsg.forUser(u)) }
      case _ => true.pure[F]
    }
  }

  private def processMsg(msg: OutputMessage): WebSocketFrame = {
    msg match {
      case KeepAlive => WebSocketFrame.Ping()
      case msg @ _   => WebSocketFrame.Text(msg.asJson.noSpaces)
    }
  }

  private def send(
      t: Topic[F, OutputMessage],
      uQueue: Queue[F, OutputMessage],
      uRef: Ref[F, Option[User]]
  ): Stream[F, WebSocketFrame] = {
    def uStream =
      Stream
        .fromQueueUnterminated(uQueue)
        .filter {
          case DiscardMessage => false
          case _              => true
        }
        .map(processMsg)

    def mainStream =
      t.subscribe(maxQueued = 1000)
        .evalFilter(filterMsg(_, uRef))
        .map(processMsg)

    Stream(uStream, mainStream).parJoinUnbounded
  }
}
```
The rest of the above 3 functions also remain unchanged.

## 9. The Server

The server function is also now uses `ChatProtocol[F]`:

```scala
package rockthejvm.websockets

import cats.effect.kernel.Async
import fs2.io.file.Files
import fs2.io.net.Network
import cats.effect.std.Queue
import fs2.concurrent.Topic
import com.comcast.ip4s.*
import cats.syntax.all.*
import org.http4s.ember.server.EmberServerBuilder

object Server {
  def server[F[_]: Async: Files: Network](
      q: Queue[F, OutputMessage],
      t: Topic[F, OutputMessage],
      im: InputMessage[F],
      chatP: ChatProtocol[F]
  ): F[Unit] = {
    val host = host"0.0.0.0"
    val port = port"8080"
    EmberServerBuilder
      .default[F]
      .withHost(host)
      .withPort(port)
      .withHttpWebSocketApp(wsb =>
        new Routes().service(wsb, q, t, im, chatP)
      )
      .build
      .useForever
      .void
  }
}
```

## 10. The Main Program

In this section, we have several updates that involve initializing Redis and Postgres:

```scala
package rockthejvm.websockets

import cats.effect.IOApp
import cats.effect.kernel.Resource
import cats.effect.IO
import dev.profunktor.redis4cats.RedisCommands
import dev.profunktor.redis4cats.Redis
import dev.profunktor.redis4cats.effect.Log.Stdout.given

object Program extends IOApp.Simple {
  private def makeRedis: Resource[IO, RedisCommands[IO, String, String]] =
    Redis[IO].utf8("redis://127.0.0.1:6379")
}
```
Here we create the `makeRedis` function by calling `Redis[IO].utf8()` and passing it a redis uri value of `redis://127.0.0.1:6379`. This returns a `Resource` that we'll use in our program.

```scala
import skunk.Session
import natchez.Trace.Implicits.noop
...
object Program extends IOApp.Simple {
  ...
  private def makePostgres: Resource[IO, Resource[IO, Session[IO]]] = 
    Session
      .pooled[IO](
        host = "localhost",
        port = 5432,
        user = "postgres",
        password = Some("password"),
        database = "websocket",
        max = 10
      )
}
```
To create the Postgres `Resource`, we use the `Session.pooled[IO]` function where we provide the `host` as `localhost`, a `port`, `user`, `password`, and `database` as `5432`, `postgres`, `Some("password")` and `websocket` all tallying with what we provided in the docker container.

Lastly, we provide a `10` as the number of concurrent sessions for our Resource:

```scala
import cats.effect.std.Queue
import fs2.concurrent.Topic
import fs2.Stream
import scala.concurrent.duration.*
import Server.server
...

object Program extends IOApp.Simple {
  ...
  def program: IO[Unit] = {
    makeRedis.use { redis =>
      makePostgres.use { session =>
        for {
          postgresprotocol <- PostgresProtocol.make[IO](session)
          redisprotocol <- RedisProtocol.make[IO](redis)
          chatprotocol <- ChatProtocol.make[IO](redisprotocol, postgresprotocol)
          im <- InputMessage.make[IO](chatprotocol)
          q <- Queue.unbounded[IO, OutputMessage]
          t <- Topic[IO, OutputMessage]
          s <- Stream(
            Stream.fromQueueUnterminated(q).through(t.publish),
            Stream
              .awakeEvery[IO](30.seconds)
              .map(_ => KeepAlive)
              .through(t.publish),
            Stream.eval(server[IO](q, t, im, chatprotocol))
          ).parJoinUnbounded.compile.drain
        } yield s
      }
    }
  }

  override def run: IO[Unit] = program
}
```
Finally, in the `program` function, we took out `ChatState` and `Protocol` and added the `PostgresProtocol` and `RedisProtocol` which we provide as arguments to the `ChatProtocol` `make()` function.

## 11. Serving HTML

Since we upgraded the `User` case class, we'll also need to make changes to `chat.html`:

```javascript
        socket.onmessage = function (event) {
            ...
            } else if (obj.ChatMsg) {
                output.append(colorText(obj.ChatMsg.from.name.name + " : " + obj.ChatMsg.msg, "purple"))
            } 
            ...
        };
```
In the `obj.ChatMsg` branch we now add `obj.ChatMsg.from.name.name` to access the user name. Everything else remains the same.

Now to run our application, we first need to start our Redis and Postgres Docker containers using Docker-Compose, and finally our application server. The application should function closely to the original.

## 12. Conclusion

In conclusion, this article has gone in-depth on how to implement Redis and Postgres in a Scala application using the redis4cats and skunk libraries. Now we can persist our messages, and rip all the benefits of storing our information in Redis such as high availability and persistence. 

In this version we simply dump all the previous messages to the new user but this should be done progressively whenever the user scrolls up, however, this was beyound the scope of this tutorial.
As always the code for this application can be found over on [Github](https://github.com/hkateu/WebSocketChatApp2).