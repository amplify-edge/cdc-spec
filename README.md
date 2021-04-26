# cdc-spec

TODO: Update the chat examples Proto and SQL to be matched. Curently they are not matching.

This Specification outlines a streaming CDC subsystem that allows the existing DB layer to have a Materialised View, such that upstream Modules and Clients get a change feed of the Materialised Views changes.

what is solves:

- Lowers the amount of complexity in the middle tier Golang code and also the Dart code, because the stateful CDC subsystem manages all this state.
  - The middle tier code is now stateless and so can be scaled out effotlessly.

- Restarts are much faster.
  - The  Materialised Views are durable and are effectively replacing caches.

- Serverless in that it allows Modules to be built that are not compiled but added at runtime and then reflected on. 
  - This is possible because the CDC subsystem is doing all the work, and the developers Protobuf in the Module described the data and services.

- Control and migrate using standard database SQL.
  - By storing all Write, Read and Change data in the Subsystem which itself is a SQL DB, you can write standard data migrations, and so ease the burden of keeping it all up to date.
  - The middle tier code refactoring effort is vastly reduced when the DB Schema changes.

- automatically enable scale:
  - you can do Master / Slave. The Master is the source write only DB. The Slaves are the Materialsied Views and change feeds.
  - you can have a master hot spare, in case your master falls over.

## General design and flow

What we want is to construct Materialised Views that are configured to update themselves when the source data they are pointing to changes, and for those Materialised vies to emit change feed when they change.

Currently we hold the source mutable data in the Genji DB and Minio S3. These are sources of data that is read from and written to currently.

The Source data becomes write only.

The Materialised View is readonly of course. All queries from your middle tier use these materialsied Views.

The middle tier is notified when the Materialised View changes, allowing it to react.

A basic example is a Chat system where you need the Flutter GUI to update automatically when users add messages to the chat with images. We will use this Chat example below to illustrate the concepts.

## 1 Protobuf Reflection

https://github.com/cloudwebrtc/nats-grpc/tree/reflection

This allows a Module developer to specify the Data and the Services as a standard Protobuf.

The subsystem is able to reflect on the Protobuf and use it as a generic API for the CDC sub system.

Example:

```protobuf
syntax = "proto3";

option go_package = "./chatpb";

package chatpb;

service ChatService {
    rpc JoinChannel(Channel) returns (stream Message) {}
    rpc SendMessage(stream Message) returns (MessageAck) {}
}

message Channel {
    string name = 1;
    string senders_name = 2;
}

message Message {
    string sender = 1;
    Channel channel = 2;
    string message = 3;
}

message MessageAck {
    string status = 1;
}
```

## 2 Data Sources (write only)

This is where data mutations are stored.

Support the following sources for the write data:

- Genji for SQL data. Genji can also hold blob btw and is used where Minio is too heavy.
- S3 for blob data like images and video.

For Genji, from the Protobuf the subsystem infers the database schema.

For S3 or genji blob, where we store images and video, we can use the existing Proto that is designed for files here: https://github.com/amplify-edge/sys-share/blob/master/sys-core/proto/v2/sys_core_file_services.proto. This protobuf is curently used for storing blobs in genji. It will need to be modified for the CDC i expect.

Change notifications need to be produced by these Data Sources so that the notifications be consumed by the system:

- For genji, we need to tap into its underlying system and produce a WAL change stream that can be consumed by the subsystem.
- For Minio, it has a good notification system that the subsytem can consume.

A Protobuf will need neded for these Change Notifications from sources.


Genji Example:

```sql
CREATE TABLE user
(
    user_id CHAR(32),
    user_login VARCHAR(255),
    user_password CHAR(64),
    user_email VARCHAR(400),
    PRIMARY KEY (user_id)
);

CREATE TABLE chat_message
(
    chat_message_id CHAR(32),
    chat_message_datetime DATETIME,
    chat_message_text TEXT,
    chat_message_chat_id CHAR(32),
    chat_message_user_id CHAR(32),
    PRIMARY KEY (chat_message_id)
);

CREATE TABLE user_chat
(
    user_chat_chat_id CHAR(32),
    user_chat_user_id CHAR(32),
    PRIMARY KEY (user_chat_chat_id,user_chat_user_id)
);

CREATE TABLE chat
(
    chat_id CHAR(32),
    chat_topic VARCHAR(32),
    chat_password CHAR(64),
    user_chat_user_id CHAR(32),
    PRIMARY KEY (chat_id)
);
```

ERD:

![alt text](https://github.com/amplify-edge/cdc-spec/blob/master/chat_erd.png?raw=true)

## 3 Materialised Views (read only)

This system listens to the subsystem's data change feed and updates its Materialised Views.

Example Materialised View is:

```sql
SELECT * from chat where chat_topic=<some uuid> 
```

For images and video the same CDC concepts apply, in that the Materialised Views hold the transcoded images and videos, and you can use SQL to query for them.

## 4 Materialsied Views Change Feed

When ever a Materialsied View changes, the subsystem sends that change to the users Module, so that they can react.

## Usage Patterns

But when you have a CDC subsystem like described above you dont really need GraphQL. Instead you can model every Screen and Widget of your flutter app in the CDC system itself.

At the Flutter level you new up a Page and the Widgets and give each the Protobuf endpoint for Mutations and Queries and Subscriptions.
Security is already modelled in the Genji DB, so you can also check if they are allowed to see data that the Protobuf represeents.


## Reference systems

Materialized is a very good example of the required sub system.

https://materialize.com/docs/overview/api-components/#sinks
