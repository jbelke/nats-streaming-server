# NATS Streaming Server

NATS Streaming is an extremely performant, lightweight reliable streaming platform built on [NATS](https://nats.io).

[![License][License-Image]][License-Url] [![ReportCard][ReportCard-Image]][ReportCard-Url] [![Build][Build-Status-Image]][Build-Status-Url] [![Coverage][Coverage-Image]][Coverage-Url]

NATS Streaming provides the following high-level feature set.
- Log based.
- At-Least-Once Delivery model, giving reliable message delivery.
- Rate matched on a per subscription basis.
- Replay/Restart
- Last Value Semantics

# Table of Contents

- [Important Changes](#important-changes)
- [Concepts](#concepts)
    * [Relation to NATS](#relation-to-nats)
    * [Client Connections](#client-connections)
    * [Channels](#channels)
        * [Message Log](#message-log)
        * [Subscriptions](#subscriptions)
            * [Regular](#regular)
            * [Durable](#durable)
            * [Queue Group](#queue-group)
            * [Redelivery](#redelivery)
    * [Store Interface](#store-interface)
    * [Clustering](#clustering)
    * [Fault Tolerance](#fault-tolerance)
        * [Active Server](#active-server)
        * [Standby Servers](#standby-servers)
        * [Shared State](#shared-state)
        * [Failover](#failover)
    * [Partitioning](#partitioning)
    * [Monitoring](#monitoring)
        * [Enabling](#enabling)
        * [Endpoints](#endpoints)
- [Getting Started](#getting-started)
    * [Building](#building)
    * [Running](#running)
- [Configuring](#configuring)
    * [Command line arguments](#command-line-arguments)
    * [Configuration file](#configuration-file)
    * [Store Limits](#store-limits)
        * [Limits inheritance](#limits-inheritance)
    * [Securing](#securing)
        * [Authorization](#authorization)
        * [TLS](#tls)
    * [Persistence](#persistence)
        * [File Store](#file-store)
            * [File Store Options](#file-store-options)
- [Clients](#clients)
- [License](#license)

# Important Changes

## Version `0.5.0`

The Store interface was updated. There are 2 news APIs:

* `GetChannels()`: Returns a map of `*ChannelStore`, keyed by channel names.<br>
The implementation needs to return a copy to make it safe for the caller to manipulate
the map without a risk of concurrent access.
* `GetChannelsCount()`: Returns the number of channels currently stored.

## Version `0.4.0`

The Store interface was updated. There are 2 news APIs:

* `Recover()`: The recovery of persistent state was previously done in the constructor of the store implementation.<br>
It is now separate and specified with this API. The server will first instantiate the store, in
which some initialization or checks can be made.<br>
If no error is reported, the server will then proceed with calling `Recover()`, which will returned the recovered state.<br>
* `GetExclusiveLock()`: In Fault Tolerance mode, when a server is elected leader, it will attempt to get an exclusive
lock to the shared storage before proceeding.<br>

Check the [Store interface](https://github.com/nats-io/nats-streaming-server/blob/master/stores/store.go) for more information.

# Concepts

## Relation to NATS

NATS Streaming Server by default embeds a [NATS](https://github.com/nats-io/gnatsd) server. That is, the Streaming server is not a server per-se, but instead, a client to a NATS Server.<br>
It means that Streaming clients are not directly connected to the streaming server, but instead communicate with the streaming server *through* NATS Server.

This detail is important when it comes to Streaming clients connections to the Streaming server. Indeed, since there is no direct
connection, the server knows if a client is connected based on heartbeats.

***It is therefore strongly recommended for clients to close their connection when the application exit, otherwise the server
will consider these clients connected (sending data, etc...) until it detects missing heartbeats.***

The streaming server creates internal subscriptions on specific subjects to communicate with its clients and/or other servers.

Note that NATS clients and NATS Streaming clients cannot exchange data between each other. That is, if a streaming client
publishes on `foo`, a NATS client subscribing on that same subject will not receive the messages. Streaming messages are NATS
messages made of a protobuf. The streaming server is expected to send ACKs back to producers and receive ACKs from consumers.
If messages were freely exchanged with the NATS clients, this would cause problems.

## Client Connections

As described, clients are not directly connected to the streaming server. Instead, they send connection requests. The request
includes a `client ID` which is used by the server to uniquely identify, and restrict, a given client. That is, no two
connections with the same client ID will be able to run concurrently.

This client ID links a given connection to its published messages, subscriptions, especially durable subscriptions. Indeed, durable
subscriptions are stored as a combination of the client ID and durable name. More on durable subscriptions later.

It is also used to resolve the issue of not having direct client connections to the server. For instance, say that a client crashes
without closing the connection. It later restarts with the same client ID. The server will detect that this client ID is already
in-use. It will try to contact that known client to its original private inbox. If the server does not receive a response - which
would be the case if the client crashed - it will replace the old client with this new one.<br>
Otherwise, the server would reject the connection request since the client ID is already in-use.

## Channels

Channels are at the heart of the NATS Streaming Server. Channels are subjects clients send data to and consume from.

***Note: NATS Streaming server does not support wildcard for channels, that is, one cannot subscribe on `foo.*`, or `>`, etc...***

The number of channels can be limited (and is by default) through configuration. Messages produced to a channel are stored
in a message log inside this channel.

### Message Log

You can view a message log as a ring buffer. Messages are appended to the end of the log. If a limit is set globally for all channels, or specifically for this channel, when the limit is reached, older messages are removed to make room for the new ones.

But except for the administrative size/age limit set for a message log, messages are not removed due to consumers consuming them.
In fact, messages are stored regardless of the presence of subscriptions on that channel.

### Subscriptions

A client creates a subscription on a given channel. Remember, there is no support for wildcards, so a subscription is really tied to
one and only one channel. The server will maintain the subscription state on behalf of the client until the later closes the subscription (or its connection).

If there are messages in the log for this channel, messages will be sent to the consumer when the subscription is created. The server will
send up to the maximum number of inflight messages as given by the client when creating the subscription.

When receiving ACKs from the consumer, the server will then deliver more messages, if more are available.

A subscription can be created to start at any point in the message log, either by message sequence, or by time.

There are several type of subscriptions:

#### Regular

The state of these subscriptions is removed when they are unsubscribed or closed (which is equivalent for this type of subscription) or
the client connection is closed (explicitly by the client, or closed by the server due to timeout). They do, however, survive a *server*
failure (if running with a persistent store).

#### Durable

If an application wishes to resume message consumption from where it previously stopped, it needs to create a durable subscription.
It does so by providing a durable name, which is combined with the client ID provided when the client created its connection. The server then
maintain the state for this subscription even after the client connection is closed.

***Note: The starting position given by the client when restarting a durable subscription is ignored.***

When the application wants to stop receiving messages on a durable subscription, it should close - but *not unsubscribe*- this subscription.
If a given client library does not have the option to close a subscription, the application should close the connection instead.

When the application wants to delete the subscription, it must unsubscribe it. Once unsubscribed, the state is removed and it is then
possible to re-use the durable name, but it will be considered a brand new durable subscription, with the start position being the one
given by the client when creating the durable subscription.

#### Queue Group

When consumers want to consume from the same channel but each receive a different message, as opposed to all receiving the same messages,
they need to create a queue subscription. When a queue group name is specified, the server will send each messages from the log to a single
consumer in the group. The distribution of these messages is not specified, therefore applications should not rely on an expected delivery
scheme.

After the first queue member is created, any other member joining the group will receive messages based on where the server is in the message log for that particular group. That means that starting position given by joining members is ignored by the server.

When the last member of the group leaves (subscription unsubscribed/closed/or connection closed), the group is removed from the server.
The next application creating a subscription with the same name will create a new group, starting at the start position given in the subscription request.

Queue subscriptions can also be durables. In this case, the client provides also a durable name. The behavior is, as you would expect,
a combination of queue and durable subscriptions. The main difference is that when the last member leaves the group, the state of
the group will be maintained by the server. Later, when a member rejoins the group, the delivery will resume.

***Note: For a durable queue subscription, the last member to * unsubscribe * (not simply close) causes the group to  be removed from the server.***

#### Redelivery

When the server sends a message to a consumer, it expects to receive an ACK from this consumer. The consumer is the one specifying
how long the server should wait before resending all unacknowledged messages to the consumer.

When the server restarts and recovers unacknowledged messages for a subscription, it will first attempt to redelivery those
messages before sending new messages. However, if during the initial redelivery some messages don't make it to the client,
the server cannot know that and will enable delivery of new messages.

***So it is possible for an application to receive redelivered messages mixed with new messages. This is typically what happens
outside of the server restart scenario.***

For queue subscriptions, if a member has unacknowledged messages, when this member `AckWait` (which is the duration given to the
server before the server should attempt to redeliver unacknowledged messages) time elapses, the messages are redelivered to any
other member in the group (including itself).

If a queue member leaves the group, its unacknowledged messages are redistributed to other queue members.

## Store Interface

Every store implementation follows the [Store interface](https://github.com/nats-io/nats-streaming-server/blob/master/stores/store.go).

On startup, the server creates a unique instance of the `Store`. The constructor of a store implementation can do some
initialization and configuration check, but *must not* access, or attempt to recover, the storage at this point. This is important
because when the server runs on Fault Tolerance mode, the storage must be shared across many servers but only one server can be
using it.

After instantiating the store, the server will then call `Recover()` in order to recover the persisted state. For implementations
that do not support persistence, such as the provided `MemoryStore`, this call will simply return `nil` (without error) to
indicate that no state was recovered.

The `Store` is used to add/delete clients, create/lookup channels, etc...

Creating/looking up a channel will return a `ChannelStore`, which points to two other interfaces, the `SubStore` and `MsgStore`. These stores, for a given channel, handle subscriptions and messages respectively.

If you wish to contribute to a new store type, your implementation must include all these interfaces. For stores that allow recovery (such as file store as opposed to memory store), there are additional structures that have been defined and should be returned by `Recover()`.

The memory and the provided file store implementations both use a generic store implementation to avoid code duplication.
When writing your own store implementation, you can do the same for APIs that don't need to do more than what the generic implementation provides.
You can check [MemStore](https://github.com/nats-io/nats-streaming-server/blob/master/stores/memstore.go) and [FileStore](https://github.com/nats-io/nats-streaming-server/blob/master/stores/filestore.go) implementations for more details.

## Clustering

NATS Streaming Server does not support clustering at this time. The confusion is that it can run with a cluster of NATS Servers,
but as previously explained, the NATS Servers are the backbone of the streaming systems. So it is possible to run a single
streaming server attached to a cluster of NATS Servers and streaming clients attached to any of the NATS Server in the NATS cluster.

Streaming clustering with full replication of data across a set of streaming servers is planned for later this year.

## Fault Tolerance

To minimize the single point of failure, NATS Streaming server can be run in Fault Tolerance mode. It works by having a group
of servers with one acting as the active server (accessing the store) and handling all communication with clients, and all others
acting as standby servers.

To start a server in Fault Tolerance (FT) mode, you specify an FT group name.

Here is an example on how starting 2 servers in FT mode running on the same host and embedding the NATS servers:

```
nats-streaming-server -store file -dir datastore -ft_group "ft" -cluster nats://localhost:6222 -routes nats://localhost:6223 -p 4222

nats-streaming-server -store file -dir datastore -ft_group "ft" -cluster nats://localhost:6223 -routes nats://localhost:6222 -p 4223
```

### Active Server

There is a single Active server in the group. This server was the first to obtain the exclusive lock for storage.
For the `FileStore` implementation, it means trying to get an advisory lock for a file located in the shared datastore. If the
elected server fails to grab this lock because it is already locked, it will go back to standby.

***Only the active server accesses the store and service all clients.***

### Standby servers

There can be as many as you want standby servers on the same group. These servers do not access the store and do not receive any data from the streaming clients. They are just running waiting for the detection of the active server failure.

### Shared State

Actual file replication to multiple disks is not handled by the Streaming server. This - if required - needs to be handled by the user. For the FileStore implementation that we currently provide, the data store needs to be mounted by all servers in the FT group (e.g. an NFS Mount (Gluster in Google Cloud or EFS in Amazon).

### Failover

When the active server fails, all standby servers will try to activate. The process consists of trying to get an exclusive lock
on the storage.

The first server that succeeds will become active and go through the process of recovering the store and service clients.
It is as if a server in standalone mode was automatically restarted.

All other servers that failed to get the store lock will go back to standby mode and stay in this mode until they stop
receiving heartbeats from the current active server.

It is possible that a standby trying to activate is not able to immediately acquire the store lock. When that happens,
it goes back into standby mode, but if it fails to receive heartbeats from an active server, it will try again to
acquire the store lock. The interval is random but as of now set to a bit more than a second.

## Partitioning

It is possible to limit the list of channels a server can handle. This can be used to:

* Prevent creation of unwanted channels
* Share the load between several servers running with the same cluster ID

In order to do so, you need to enable the `partitioning` parameter in the configuration file,
and also specify the list of allowed channels in the `channels` section of the `store_limits` configuration.

Channels don't need to override any limit, but they need to be specified for the server to service only these channels.

Here is an example:

```
partitioning: true
store_limits: {
    channels: {
        "foo": {}
        "bar": {}
        # Use of wildcards in configuration is allowed. However, applications cannot
        # publish to, or subscribe to, wildcard channels.
        "baz.*": {}
    }
}
```

When partitioning is enabled, multiple servers with the same cluster ID can coexist on the same NATS network,
each server handling its own set of channels. ***Note however that in this mode, state is not replicated
(as it will be in the full clustering feature to come). The only communication between servers is to report
if a given channel is handled in more than one server***.

### Wildcards

NATS Streaming does not support sending or subscribing to wildcard channels (such as `foo.*`).

However, it is possible to use wildcards to define the partition that a server can handle.
For instance, with the following configuration:
```
partitioning: true
store_limits: {
    channels: {
        "foo.*": {}
        "bar.>": {}
    }
}
```
The streaming server would accept subscriptions or published messages to channels such as:
1. `foo.bar`
2. `bar.baz`
3. `bar.baz.bat`
4. ...

But would ignore messages or subscriptions on:

1. `foo`
2. `foo.bar.baz`
3. `bar`
4. `some.other.channel`
5. ...

### A given channel must be defined in a single server

When a server starts, it sends its list of channels to all other servers on the same cluster in an attempt
to detect duplicate channels. When a server receives this list and finds that it has a channel in
common, it will return an error to the emitting server, which will then fail to start.

However, on startup, it is possible that the underlying NATS cluster is not fully formed. The server would
not get any response from the rest of the cluster and therefore start successfully and service clients.
Anytime a Streaming server detects that a NATS server was added to the NATS cluster, it will resend its list
of channels. It means that currently running servers may suddenly fail with a message regarding duplicate channels.
Having the same channel on different servers means that a subscription would be created on all servers
handling the channel, but only one server will receive and process message acknowledgements. Other servers
would then redeliver messages (since they would not get the acknowledgements), which would cause duplicates.

***In order to avoid issues with channels existing on several servers, it is ultimately the responsibility
of the administrator to ensure that channels are unique.***

### Fault Tolerance and Partitioning

You can easily combine the Fault Tolerance and Partitioning feature.

To illustrate, suppose that we want two partitions, one for `foo.>` and one for `bar.>`.

The configuration for the first server `foo.conf` would look like:
```
partitioning: true
store_limits: {
    channels: {
        foo.>: {}
    }
}
```

The second configuration `bar.conf` would be:
```
partitioning: true
store_limits: {
    channels: {
        bar.>: {}
    }
}
```

If you remember, Fault Tolerance is configured by specifying a name (`ft_group_name`). Suppose there is
an NFS mount called `/nss/datastore` on both `host1` and `host2`.

Starting an FT pair for the partition `foo` could look like this:
```
host1$ nats-streaming-server -store file -dir /nss/datastore/foodata -sc foo.conf -ft_group_name foo -cluster nats://host1:6222 -routes nats://host2:6222,nats://host2:6223

host2$ nats-streaming-server -store file -dir /nss/datastore/foodata -sc foo.conf -ft_group_name foo -cluster nats://host2:6222 -routes nats://host1:6222,nats://host1:6223
```
Notice that each server on each note points to each other (the `-routes` parameter). The reason why
we also point to `6223` will be explained later. They both listen for routes connections on their
host's `6222` port.

We now start the FT pair for `bar`. Since we are running from the same machines (we don't have to),
we need to use a different port:
```
host1$ nats-streaming-server -store file -dir /nss/datastore/bardata -sc bar.conf -ft_group_name bar -p 4223 -cluster nats://host1:6223 -routes nats://host2:6222,nats://host2:6223

host2$ nats-streaming-server -store file -dir /nss/datastore/bardata -sc bar.conf -ft_group_name bar -p 4223 -cluster nats://host2:6223 -routes nats://host1:6222,nats://host1:6223
```
You will notice that the `-routes` parameter points to both `6222` and `6223`, this is so that
both partitions belong to the same cluster and be view as "one" by a Streaming application connecting
to this cluster. Effectively, we have created a full mesh of 4 NATS servers that can all communicate
with each other. Two of these servers are backups for servers running on the same FT group.

### Applications behavior

When an application connects, it specifies a cluster ID. If several servers are running with that same
cluster ID, the application will be able to publish/subscribe to any channel handled by the cluster (as long
as those servers are all connected to the NATS network).

A published message will be received by only the server that has that channel defined.
If no server is handling this channel, no specific error is returned, instead the publish call will timeout.
Same goes for message acknowledgements. Only the server handling the subscription on this channel should
receive those.

However, other client requests (such has connection and subscription requests) are received by all servers.
For connections, all servers handle them and the client library will receive a response from all servers in the
cluster, but use the first one that it received.<br>
For subscriptions, a server receiving the request for a channel that it does not handle will simply ignore
the request. Again, if no server handle this channel, the client's subscription request will simply time out.

## Monitoring

To monitor the NATS streaming system, a lightweight HTTP server is used on a dedicated monitoring port.
The monitoring server provides several endpoints, all returning a JSON object.

The NATS monitoring endpoints support JSONP and CORS, making it easy to create single page monitoring web applications.

### Enabling

To enable the monitoring server, start the NATS Streaming Server with the monitoring flag -m (or -ms) and
specify the monitoring port.

Monitoring options
```
-m, --http_port PORT             HTTP PORT for monitoring
-ms,--https_port PORT            Use HTTPS PORT for monitoring (requires TLS cert and key)
```
To enable monitoring via the configuration file, use `http: "host:port"` or `https: "host:port"` (there is
no explicit configuration flag for the monitoring interface).

For example, after running this:
```
nats-streaming-server -m 8222
```
you should see that the NATS Streaming server starts with the HTTP monitoring port enabled:

```
[19015] 2017/06/07 14:29:43.328760 [INF] STREAM: Starting nats-streaming-server[test-cluster] version 0.5.0
(...)
[19015] 2017/06/07 14:29:43.329011 [INF] Starting http monitor on 0.0.0.0:8222
[19015] 2017/06/07 14:29:43.329085 [INF] Listening for client connections on 0.0.0.0:4222
(...)
```
You can then point your browser (or curl) to [http://localhost:8222/streaming](http://localhost:8222/streaming)

### Endpoints

The following sections describe each supported monitoring endpoint: serverz, storez, clientsz, and channelsz.

#### /serverz

The endpoint [http://localhost:8222/streaming/serverz](http://localhost:8222/streaming/serverz) reports
various general statistics.
```
{
  "cluster_id": "test-cluster",
  "server_id": "J3Odi0wXYKWKFWz5D5uhH9",
  "version": "0.5.0",
  "go": "go1.7.5",
  "state": "STANDALONE",
  "now": "2017-06-07T14:45:29.472535266+02:00",
  "start_time": "2017-06-07T14:44:49.936077607+02:00",
  "uptime": "39s",
  "clients": 11,
  "subscriptions": 10,
  "channels": 1,
  "total_msgs": 85350,
  "total_bytes": 12785990
}
```

#### /storez

The endpoint [http://localhost:8222/streaming/storez](http://localhost:8222/streaming/storez) reports
information about the store.
```
{
  "cluster_id": "test-cluster",
  "server_id": "J3Odi0wXYKWKFWz5D5uhH9",
  "now": "2017-06-07T14:45:46.738570607+02:00",
  "type": "MEMORY",
  "limits": {
    "max_channels": 100,
    "max_msgs": 1000000,
    "max_bytes": 1024000000,
    "max_age": 0,
    "max_subscriptions": 1000
  },
  "total_msgs": 130691,
  "total_bytes": 19587140
}
```

#### /clientsz

The endpoint [http://localhost:8222/streaming/clientsz](http://localhost:8222/streaming/clientsz) reports
more detailed information about the connected clients.

It uses a paging mechanism which defaults to 1024 clients.

You can control these via URL arguments (limit and offset). For example: [http://localhost:8222/streaming/clientsz?limit=1&offset=1](http://localhost:8222/streaming/clientsz?limit=1&offset=1).
```
{
  "cluster_id": "test-cluster",
  "server_id": "J3Odi0wXYKWKFWz5D5uhH9",
  "now": "2017-06-07T14:47:44.495254605+02:00",
  "offset": 1,
  "limit": 1,
  "count": 1,
  "total": 11,
  "clients": [
    {
      "id": "benchmark-sub-0",
      "hb_inbox": "_INBOX.jAHSY3hcL5EGFQGYmfayQK"
    }
  ]
}
```
You can also report detailed subscription information on a per client basis using `subs=1`.
For example: [http://localhost:8222/streaming/clientsz?limit=1&offset=1&subs=1](http://localhost:8222/streaming/clientsz?limit=1&offset=1&subs=1).
```
{
  "cluster_id": "test-cluster",
  "server_id": "J3Odi0wXYKWKFWz5D5uhH9",
  "now": "2017-06-07T14:48:06.157468748+02:00",
  "offset": 1,
  "limit": 1,
  "count": 1,
  "total": 11,
  "clients": [
    {
      "id": "benchmark-sub-0",
      "hb_inbox": "_INBOX.jAHSY3hcL5EGFQGYmfayQK",
      "subscriptions": {
        "foo": [
          {
            "inbox": "_INBOX.jAHSY3hcL5EGFQGYmfayvC",
            "ack_inbox": "_INBOX.J3Odi0wXYKWKFWz5D5uhem",
            "is_durable": false,
            "max_inflight": 1024,
            "ack_wait": 30,
            "last_sent": 505597,
            "pending_count": 0,
            "is_stalled": false
          }
        ]
      }
    }
  ]
}
```
You can select a specific client based on its client ID with `client=<id>`, and get also get detailed statistics with `subs=1`.
For example: [http://localhost:8222/streaming/clientsz?client=me&subs=1](http://localhost:8222/streaming/clientsz?client=me&subs=1).
```
{
  "id": "me",
  "hb_inbox": "_INBOX.HG0uDuNtAPxJQ1lVjIC2sr",
  "subscriptions": {
    "foo": [
      {
        "inbox": "_INBOX.HG0uDuNtAPxJQ1lVjIC389",
        "ack_inbox": "_INBOX.Q9iH2gsDPN57ZEvqswiYSL",
        "is_durable": false,
        "max_inflight": 1024,
        "ack_wait": 30,
        "last_sent": 0,
        "pending_count": 0,
        "is_stalled": false
      }
    ]
  }
}
```

#### /channelsz

The endpoint [http://localhost:8222/streaming/channelsz](http://localhost:8222/streaming/channelsz) reports
the list of channels.
```
{
  "cluster_id": "test-cluster",
  "server_id": "J3Odi0wXYKWKFWz5D5uhH9",
  "now": "2017-06-07T14:48:41.680592041+02:00",
  "offset": 0,
  "limit": 1024,
  "count": 2,
  "total": 2,
  "names": [
    "bar"
    "foo"
  ]
}
```
It uses a paging mechanism which defaults to 1024 channels.

You can control these via URL arguments (limit and offset).
For example: [http://localhost:8222/streaming/channelsz?limit=1&offset=1](http://localhost:8222/streaming/channelsz?limit=1&offset=1).
```
{
  "cluster_id": "test-cluster",
  "server_id": "J3Odi0wXYKWKFWz5D5uhH9",
  "now": "2017-06-07T14:48:41.680592041+02:00",
  "offset": 1,
  "limit": 1,
  "count": 1,
  "total": 2,
  "names": [
    "foo"
  ]
}
```
You can also get the list of subscriptions with `subs=1`.
For example: [http://localhost:8222/streaming/channelsz?limit=1&offset=0&subs=1](http://localhost:8222/streaming/channelsz?limit=1&offset=0&subs=1).
```
{
  "cluster_id": "test-cluster",
  "server_id": "J3Odi0wXYKWKFWz5D5uhH9",
  "now": "2017-06-07T15:01:02.166116959+02:00",
  "offset": 0,
  "limit": 1,
  "count": 1,
  "total": 2,
  "channels": [
    {
      "name": "bar",
      "msgs": 0,
      "bytes": 0,
      "first_seq": 0,
      "last_seq": 0,
      "subscriptions": [
        {
          "inbox": "_INBOX.S7kTJjOcToXiJAzGWgINit",
          "ack_inbox": "_INBOX.Y04G5pZxlint3yPXrSTjTV",
          "is_durable": false,
          "max_inflight": 1024,
          "ack_wait": 30,
          "last_sent": 0,
          "pending_count": 0,
          "is_stalled": false
        }
      ]
    }
  ]
}
```
You can select a specific channel based on its name with `channel=name`.
For example: [http://localhost:8222/streaming/channelsz?channel=foo](http://localhost:8222/streaming/channelsz?channel=foo).
```
{
  "name": "foo",
  "msgs": 649234,
  "bytes": 97368590,
  "first_seq": 1,
  "last_seq": 649234
}
```
And again, you can get detailed subscriptions with `subs=1`.
For example: [http://localhost:8222/streaming/channelsz?channel=foo&subs=1](http://localhost:8222/streaming/channelsz?channel=foo&subs=1).
```
{
  "name": "foo",
  "msgs": 704770,
  "bytes": 105698990,
  "first_seq": 1,
  "last_seq": 704770,
  "subscriptions": [
    {
      "inbox": "_INBOX.jAHSY3hcL5EGFQGYmfayvC",
      "ack_inbox": "_INBOX.J3Odi0wXYKWKFWz5D5uhem",
      "is_durable": false,
      "max_inflight": 1024,
      "ack_wait": 30,
      "last_sent": 704770,
      "pending_count": 0,
      "is_stalled": false
    },
    {
      "inbox": "_INBOX.jAHSY3hcL5EGFQGYmfaywG",
      "ack_inbox": "_INBOX.J3Odi0wXYKWKFWz5D5uhjV",
      "is_durable": false,
      "max_inflight": 1024,
      "ack_wait": 30,
      "last_sent": 704770,
      "pending_count": 0,
      "is_stalled": false
    },
    (...)
  ]
}
```

# Getting Started

The best way to get the NATS Streaming Server is to use one of the pre-built release binaries which are available for OSX, Linux (x86-64/ARM), Windows. Instructions for using these binaries are on the GitHub releases page.

Of course you can build the latest version of the server from the master branch. The master branch will always build and pass tests, but may not work correctly in your environment. You will first need Go installed on your machine (version 1.5+ is required) to build the NATS server.

See also the NATS Streaming Quickstart [tutorial](https://nats.io/documentation/streaming/nats-streaming-quickstart/).

## Building

Building the NATS Streaming Server from source requires at least version 1.6 of Go, but we encourage the use of the latest stable release. Information on installation, including pre-built binaries, is available at http://golang.org/doc/install. Stable branches of operating system packagers provided by your OS vendor may not be sufficient.

Run `go version` to see the version of Go which you have installed.

Run `go build` inside the directory to build.

Run `go test ./...` to run the unit regression tests.

A successful build produces no messages and creates an executable called `nats-streaming-server` in the current directory. You can invoke that binary, with no options and no configuration file, to start a server with acceptable standalone defaults (no authentication, memory store).

Run go help for more guidance, and visit http://golang.org/ for tutorials, presentations, references and more.

## Running

The NATS Streaming Server embeds a NATS Server. Starting the server with no argument will give you a server with default settings and a memory based store.

```
> ./nats-streaming-server
[18648] 2017/06/27 12:28:25.989594 [INF] STREAM: Starting nats-streaming-server[test-cluster] version 0.5.0
[18648] 2017/06/27 12:28:25.989669 [INF] STREAM: ServerID: csSwT119qazk62IIen0oyB
[18648] 2017/06/27 12:28:25.989672 [INF] STREAM: Go version: go1.7.5
[18648] 2017/06/27 12:28:25.990300 [INF] Starting nats-server version 0.9.6
[18648] 2017/06/27 12:28:25.991620 [INF] Listening for client connections on 0.0.0.0:4222
[18648] 2017/06/27 12:28:25.992153 [INF] Server is ready
[18648] 2017/06/27 12:28:26.279424 [INF] STREAM: Message store is MEMORY
[18648] 2017/06/27 12:28:26.279491 [INF] STREAM: ---------- Store Limits ----------
[18648] 2017/06/27 12:28:26.279496 [INF] STREAM: Channels:                  100 *
[18648] 2017/06/27 12:28:26.279499 [INF] STREAM: --------- Channels Limits --------
[18648] 2017/06/27 12:28:26.279503 [INF] STREAM:   Subscriptions:          1000 *
[18648] 2017/06/27 12:28:26.279506 [INF] STREAM:   Messages     :       1000000 *
[18648] 2017/06/27 12:28:26.279509 [INF] STREAM:   Bytes        :     976.56 MB *
[18648] 2017/06/27 12:28:26.279512 [INF] STREAM:   Age          :     unlimited *
[18648] 2017/06/27 12:28:26.279515 [INF] STREAM: ----------------------------------
```

The server will be started and listening for client connections on port 4222 (the default) from all available interfaces. The logs will be displayed to stderr as shown above.

Note that you do not need to start the embedded NATS Server. It is started automatically when you run the NATS Streaming Server. See below for details on how you secure the embedded NATS Server.

# Configuring

## Command line arguments

The NATS Streaming Server accepts command line arguments to control its behavior. There is a set of parameters specific to the NATS Streaming Server and some to the embedded NATS Server.

***Note about parameters types***

| Type | Remark |
|:----|:----|
|`<bool>`| For booleans, either simply specify the parameter with value to enable (e.g `-SD`), or specify `=false` to disable|
|`<size>` | You can specify as a number `1024` or as a size `1KB`|
|`<duration>` | Values must be expressed in the form `_h_m_s`, such as `1h` or `20s` or `1h30m`, or `1.5h`, etc...|

```
Usage: nats-streaming-server [options]

Streaming Server Options:
    -cid, --cluster_id  <string>     Cluster ID (default: test-cluster)
    -st,  --store <string>           Store type: MEMORY|FILE (default: MEMORY)
          --dir <string>             For FILE store type, this is the root directory
    -mc,  --max_channels <int>       Max number of channels (0 for unlimited)
    -msu, --max_subs <int>           Max number of subscriptions per channel (0 for unlimited)
    -mm,  --max_msgs <int>           Max number of messages per channel (0 for unlimited)
    -mb,  --max_bytes <size>         Max messages total size per channel (0 for unlimited)
    -ma,  --max_age <duration>       Max duration a message can be stored ("0s" for unlimited)
    -ns,  --nats_server <string>     Connect to this external NATS Server URL (embedded otherwise)
    -sc,  --stan_config <string>     Streaming server configuration file
    -hbi, --hb_interval <duration>   Interval at which server sends heartbeat to a client
    -hbt, --hb_timeout <duration>    How long server waits for a heartbeat response
    -hbf, --hb_fail_count <int>      Number of failed heartbeats before server closes the client connection
          --ack_subs <int>           Number of internal subscriptions handling incoming ACKs (0 means one per client's subscription)
          --ft_group <string>        Name of the FT Group. A group can be 2 or more servers with a single active server and all sharing the same datastore.

Streaming Server File Store Options:
    --file_compact_enabled <bool>        Enable file compaction
    --file_compact_frag <int>            File fragmentation threshold for compaction
    --file_compact_interval <int>        Minimum interval (in seconds) between file compactions
    --file_compact_min_size <size>       Minimum file size for compaction
    --file_buffer_size <size>            File buffer size (in bytes)
    --file_crc <bool>                    Enable file CRC-32 checksum
    --file_crc_poly <int>                Polynomial used to make the table used for CRC-32 checksum
    --file_sync <bool>                   Enable File.Sync on Flush
    --file_slice_max_msgs <int>          Maximum number of messages per file slice (subject to channel limits)
    --file_slice_max_bytes <size>        Maximum file slice size - including index file (subject to channel limits)
    --file_slice_max_age <duration>      Maximum file slice duration starting when the first message is stored (subject to channel limits)
    --file_slice_archive_script <string> Path to script to use if you want to archive a file slice being removed
    --file_fds_limit <int>               Store will try to use no more file descriptors than this given limit
    --file_parallel_recovery <int>       On startup, number of channels that can be recovered in parallel

Streaming Server TLS Options:
    -secure <bool>                   Use a TLS connection to the NATS server without
                                     verification; weaker than specifying certificates.
    -tls_client_key <string>         Client key for the streaming server
    -tls_client_cert <string>        Client certificate for the streaming server
    -tls_client_cacert <string>      Client certificate CA for the streaming server

Streaming Server Logging Options:
    -SD, --stan_debug=<bool>         Enable STAN debugging output
    -SV, --stan_trace=<bool>         Trace the raw STAN protocol
    -SDV                             Debug and trace STAN
    (See additional NATS logging options below)

Embedded NATS Server Options:
    -a, --addr <string>              Bind to host address (default: 0.0.0.0)
    -p, --port <int>                 Use port for clients (default: 4222)
    -P, --pid <string>               File to store PID
    -m, --http_port <int>            Use port for http monitoring
    -ms,--https_port <int>           Use port for https monitoring
    -c, --config <string>            Configuration file

Logging Options:
    -l, --log <string>               File to redirect log output
    -T, --logtime=<bool>             Timestamp log entries (default: true)
    -s, --syslog <string>            Enable syslog as log method
    -r, --remote_syslog <string>     Syslog server addr (udp://localhost:514)
    -D, --debug=<bool>               Enable debugging output
    -V, --trace=<bool>               Trace the raw protocol
    -DV                              Debug and trace

Authorization Options:
        --user <string>              User required for connections
        --pass <string>              Password required for connections
        --auth <string>              Authorization token required for connections

TLS Options:
        --tls=<bool>                 Enable TLS, do not verify clients (default: false)
        --tlscert <string>           Server certificate file
        --tlskey <string>            Private key for server certificate
        --tlsverify=<bool>           Enable TLS, verify client certificates
        --tlscacert <string>         Client certificate CA for verification

NATS Clustering Options:
        --routes <string, ...>       Routes to solicit and connect
        --cluster <string>           Cluster URL for solicited routes

Common Options:
    -h, --help                       Show this message
    -v, --version                    Show version
        --help_tls                   TLS help.
```

## Configuration file

You can use a configuration file to configure the options specific to the NATS Streaming server.

Use the `-sc` or `-stan_config` command line parameter to specify the file to use.

For the embedded NATS Server, you can use another configuration file and pass it to the Streaming server using `-c` or `--config` command line parameters.

However, since options do not overlap, it is possible to combine all options into a single file and specify this file using either the `-sc` or `-c` command line parameter.

Note the order in which options are applied during the start of a NATS Streaming server:

1. Start with some reasonable default options.
2. If a configuration file is specified, override those options
  with all options defined in the file. This include options that are defined
  but have no value specified. In this case, the zero value for the type of the
  option will be used.
3. Any command line parameter override all of the previous set options.

In general the configuration parameters are the same as the command line arguments.
But see an example configuration file below for more details:

```
# Define the cluster name.
# Can be id, cid or cluster_id
id: "my_cluster_name"

# Store type
# Can be st, store, store_type or StoreType
# Possible values are file or memory (case insensitive)
store: "file"

# When using a file store, need to provide the root directory.
# Can be dir or datastore
dir: "/path/to/storage"

# Debug flag.
# Can be sd or stand_debug
sd: false

# Trace flag.
# Can be sv or stan_trace
sv: false

# If specified, connects to an external NATS server, otherwise
# starts and embedded server.
# Can be ns, nats_server or nats_server_url
ns: "nats://localhost:4222"

# This flag creates a TLS connection to the server but without
# the need to use a TLS configuration (no NATS server certificate verification).
secure: false

# Interval at which the server sends an heartbeat to a client,
# expressed as a duration.
# Can be hbi, hb_interval, server_to_client_hb_interval
hb_interval: "10s"

# How long the server waits for a heartbeat response from the client
# before considering it a failed hearbeat. Expressed as a duration.
# Can be hbt, hb_timeout, server_to_client_hb_timeout
hb_timeout: "10s"

# Count of failed hearbeats before server closes the client connection.
# The actual total wait is: (fail count + 1) * (hb interval + hb timeout).
# Can be hbf, hb_fail_count, server_to_client_hb_fail_count
hb_fail_count: 2

# Normally, when a client creates a subscription, the server creates
# an internal subscription to receive its ACKs.
# If lots of subscriptions are created, the number of internal
# subscriptions in the server could be very high. To curb this growth,
# use this parameter to configure a pool of internal ACKs subscriptions.
# Can be ack_subs_pool_size, ack_subscriptions_pool_size
ack_subs_pool_size: 10

# In Fault Tolerance mode, you can start a group of streaming servers
# with only one server being active while others are running in standby
# mode. The FT group is named.
# Can be ft_group, ft_group_name
ft_group: "ft"

# In order to use partitioning, this parameter needs to be set to true
# and the list of channels defined in store_limits/channels section.
# This section then serves two purposes, overriding limits for a given
# channel or adding it to the partition.
partitioning: true

# Define store limits.
# Can be limits, store_limits or StoreLimits.
# See Store Limits chapter below for more details.
store_limits: {
    # Define maximum number of channels.
    # Can be mc, max_channels or MaxChannels
    max_channels: 100

    # Define maximum number of subscriptions per channel.
    # Can be msu, max_sybs, max_subscriptions or MaxSubscriptions
    max_subs: 100

    # Define maximum number of messages per channel.
    # Can be mm, max_msgs, MaxMsgs, max_count or MaxCount
    max_msgs: 10000

    # Define total size of messages per channel.
    # Can be mb, max_bytes or MaxBytes. Expressed in bytes
    max_bytes: 10240000

    # Define how long messages can stay in the log, expressed
    # as a duration, for example: "24h" or "1h15m", etc...
    # Can be ma, max_age, MaxAge.
    max_age: "24h"
}

# TLS configuration.
tls: {
    client_cert: "/path/to/client/cert_file"
    client_key: "/path/to/client/key_file"
    # Can be client_ca or client_cacert
    client_ca: "/path/to/client/ca_file"
}

# Configure file store specific options.
# Can be file or file_options
file: {
    # Enable/disable file compaction.
    # Can be compact or compact_enabled
    compact: true

    # Define compaction threshold (in percentage)
    # Can be compact_frag or compact_fragmemtation
    compact_frag: 50

    # Define minimum interval between attempts to compact files.
    # Expressed in seconds
    compact_interval: 300

    # Define minimum size of a file before compaction can be attempted
    # Expressed in bytes
    compact_min_size: 10485760

    # Define the size of buffers that can be used to buffer write operations.
    # Expressed in bytes
    buffer_size: 2097152

    # Define if CRC of records should be computed on reads.
    # Can be crc or do_crc
    crc: true

    # You can select the CRC polynomial. Note that changing the value
    # after records have been persisted would result in server failing
    # to start complaining about data corruption.
    crc_poly: 3988292384

    # Define if server should perform "file sync" operations during a flush.
    # Can be sync, do_sync, sync_on_flush
    sync: true

    # Define the file slice maximum number of messages. If set to 0 and a
    # channel count limit is set, then the server will set a slice count
    # limit automatically.
    # Can be slice_max_msgs, slice_max_count, slice_msgs, slice_count
    slice_max_msgs: 10000

    # Define the file slice maximum size (including the size of index file).
    # If set to 0 and a channel size limit is set, then the server will
    # set a slice bytes limit automatically.
    # Expressed in bytes.
    # Can be slice_max_bytes, slice_max_size, slice_bytes, slice_size
    slice_max_bytes: 67108864

    # Define the period of time covered by a file slice, starting at when
    # the first message is stored. If set to 0 and a channel age limit
    # is set, then the server will set a slice age limit automatically.
    # Expressed as a duration, such as "24h", etc..
    # Can be  slice_max_age, slice_age, slice_max_time, slice_time_limit
    slice_max_age: "24h"

    # Define the location and name of a script to be invoked when the
    # server discards a file slice due to limits. The script is invoked
    # with the name of the channel, the name of data and index files.
    # It is the responsibility of the script to then remove the unused
    # files.
    # Can be slice_archive_script, slice_archive, slice_script
    slice_archive_script: "/home/nats-streaming/archive/script.sh"

    # Channels translate to sub-directories under the file store's root
    # directory. Each channel needs several files to maintain the state
    # so the need for file descriptors increase with the number of
    # channels. This option instructs the store to limit the concurrent
    # use of file descriptors. Note that this is a soft limit and there
    # may be cases when the store will use more than this number.
    # A value of 0 means no limit. Setting a limit will probably have
    # a performance impact.
    # Can be file_descriptors_limit, fds_limit
    fds_limit: 100

    # When the server starts, the recovery of channels (directories) is
    # done sequentially. However, when using SSDs, it may be worth
    # setting this value to something higher than 1 to perform channels
    # recovery in parallel.
    parallel_recovery: 4
}
```

## Store Limits

The `store_limits` section in the configuration file (or the command line parameters
`-mc`, `-mm`, etc..) allow you to configure the global limits.

These limits somewhat offer some upper bound on the size of the storage. By multiplying
the limits per channel with the maximum number of channels, you will get a total limit.

It is not the case, though, if you override limits of some channels. Indeed, it is possible
to define specific limits per channel. Here is how:

```
...
store_limits: {
    # Override some global limits
    max_channels: 10
    max_msgs: 10000
    max_bytes: 10MB
    max_age: "1h"

    # Per channel configuration.
    # Can be channels, channels_limits, per_channel, per_channel_limits or ChannelsLimits
    channels: {
        "foo": {
            # Possible options are the same than in the store_limits section, except
            # for max_channels. Not all limits need to be specified.
            max_msgs: 300
            max_subs: 50
        }
        "bar": {
            max_msgs:50
            max_bytes:1KB
        }
        "baz": {
            # Set to 0 for ignored (or unlimited)
            max_msgs: 0
            # Override with a lower limit
            max_bytes: 1MB
            # Override with a higher limit
            max_age: "2h"
        }
        # When using partitioning, channels need to be listed.
        # They don't have to override any limit.
        "bozo": {}

        # Wildcards are possible in configuration. This says that any channel
        # that will start with "foo" but with at least 2 tokens, will be
        # able to store 400 messages. Other limits are inherited from global.
        "foo.>": {
            max_msgs:400
        }
        # This one says that if the channel name starts with "foo.bar" but has
        # at least one more token, the sever will hold it for 2 hours instead
        # of one. The max number of messages is inherited from "foo.>", so the
        # limit will be 400. All other limits are inherited from global.
        "foo.bar.>": {
            max_age: "2h"
        }
    }
}
...
```

Note that the number of defined channels cannot be greater than the stores' maximum number
of channels. ***This is true only for channels without wildcards.***

Channels limits can override global limits by being either higher, lower or even set to
unlimited.

***An unlimited value applies to the specified limit, not to the whole channel***

That is, in the configuration above, `baz` has the maximum number of messages set
to 0, which means ignored or unlimited. Yet, other limits such as max bytes, max age
and max subscriptions (inherited in this case) still apply. What that means is that
the store will not check the number of messages but still check the other limits.

For a truly unlimited channel *all* limits need to be set to 0.

### Limits inheritance

When starting the server from the command line, global limits that are not specified
(configuration file or command line parameters) are inherited from default limits
selected by the server.

Per-channel limits that are not explicitly configured inherit from the corresponding
global limit (which can itself be inherited from default limit).

If a per-channel limit is set to 0 in the configuration file (or negative value
programmatically), then it becomes unlimited, regardless of the corresponding
global limit.

On startup the server displays the store limits. Notice the `*` at the right of a
limit to indicate that the limit was inherited from the default store limits.

For channels that have been configured, their name is displayed and only the
limits being specifically set are displayed to minimize the output.

#### Wildcards

Wildcards are allowed for channels configuration. Limits for `foo.>`
will apply to any channel that starts with `foo` (but has at least one more token).
If `foo.bar.>` is specified, it will inherit from `foo.>` and from global limits.

Below is what would be displayed with the above store limits configuration. Notice
how `foo.bar.>` is indented compared to `foo.>` to show the inheritance.

```
[63762] 2017/04/19 14:47:36.149341 [INF] STREAM: ---------- Store Limits ----------
[63762] 2017/04/19 14:47:36.149352 [INF] STREAM: Channels:                   10
[63762] 2017/04/19 14:47:36.149355 [INF] STREAM: --------- Channels Limits --------
[63762] 2017/04/19 14:47:36.149359 [INF] STREAM:   Subscriptions:          1000 *
[63762] 2017/04/19 14:47:36.149362 [INF] STREAM:   Messages     :         10000
[63762] 2017/04/19 14:47:36.149365 [INF] STREAM:   Bytes        :      10.00 MB
[63762] 2017/04/19 14:47:36.149368 [INF] STREAM:   Age          :        1h0m0s
[63762] 2017/04/19 14:47:36.149371 [INF] STREAM: -------- List of Channels ---------
[63762] 2017/04/19 14:47:36.149374 [INF] STREAM: foo.>
[63762] 2017/04/19 14:47:36.149378 [INF] STREAM:  |-> Messages                   400
[63762] 2017/04/19 14:47:36.149383 [INF] STREAM:  foo.bar.>
[63762] 2017/04/19 14:47:36.149387 [INF] STREAM:   |-> Age                    2h0m0s
[63762] 2017/04/19 14:47:36.149390 [INF] STREAM: bar
[63762] 2017/04/19 14:47:36.149393 [INF] STREAM:  |-> Messages                    50
[63762] 2017/04/19 14:47:36.149572 [INF] STREAM:  |-> Bytes                  1.00 KB
[63762] 2017/04/19 14:47:36.149581 [INF] STREAM: baz
[63762] 2017/04/19 14:47:36.149585 [INF] STREAM:  |-> Messages             unlimited
[63762] 2017/04/19 14:47:36.149589 [INF] STREAM:  |-> Bytes                  1.00 MB
[63762] 2017/04/19 14:47:36.149592 [INF] STREAM:  |-> Age                     2h0m0s
[63762] 2017/04/19 14:47:36.149595 [INF] STREAM: bozo
[63762] 2017/04/19 14:47:36.149599 [INF] STREAM: -----------------------------------
```


## Securing

### Authorization

If you use basic authorization, that is a single user defined through the configuration file or command line parameters, nothing else is required. The embedded NATS Server is started with those credentials and the NATS Streaming Server uses them.

However, if you define multiple users, then you **must** specify the username and password (or token) corresponding to the NATS Streaming Server's user.

For instance, suppose that your configuration file `server.cfg` has the following content:

```
listen: 127.0.0.1:4233
http: 127.0.0.1:8233

authorization {
    users = [
      {user: alice, password: foo}
      {user: bob,   password: bar}
    ]
}
```
and you start the NATS Streaming Server this way:
```sh
nats-streaming-server -config server.cfg
```
then the server would fail to start. You **must** specify the user used by the streaming server. For instance:
```sh
nats-streaming-server -config server.cfg -user alice -pass foo
```

### TLS

While there are several TLS related parameters to the streaming server, securing the NATS Streaming server's connection is straightforward when you bear in mind that the relationship between the NATS Streaming server and the embedded NATS server is a client server relationship.  To state simply, the streaming server is a client of it's embedded NATS server.

That means two sets of TLS configuration parameters must be used:  TLS server parameters for the embedded NATS server, and TLS client parameters for the streaming server itself.

The streaming server specifies it's TLS client certificates with the following three parameters:
```    
    -tls_client_key              Client key for the streaming server

    -tls_client_cert             Client certificate for the streaming server

    -tls_client_cacert           Client certificate CA for the streaming server
```

These could be the same certificates used with your NATS streaming clients.

The embedded NATS server specifies TLS server certificates with these:

```
        --tlscert <file>             Server certificate file

        --tlskey <file>              Private key for server certificate

        --tlscacert <file>           Client certificate CA for verification
```

The server parameters are used the same way you would secure a typical NATS server. See [here](https://github.com/nats-io/gnatsd#securing-nats).

Proper usage of the NATS Streaming Server requires the use of both client and server parameters.

e.g.:

```sh
nats-streaming-server -tls_client_cert client-cert.pem -tls_client_key client-key.pem -tls_client_cacert ca.pem -tlscert server-cert.pem -tlskey server-key.pem -tlscacert ca.pem
```

Further TLS related functionality can be found in [usage](https://github.com/nats-io/gnatsd#securing-nats), and should specifying cipher suites be required, a configuration file for the embedded NATS server can be passed through the `-config` command line parameter.

## Persistence

By default, the NATS Streaming Server stores its state in memory, which means that if the streaming server is stopped, all state is lost. Still, this level of persistence allows applications to stop and later resume the stream of messages, and protect against applications disconnect (network or applications crash).

### File Store

For a higher level of message delivery, the server should be configured with a file store. NATS Streaming Server comes with a basic file store implementation. Various file store implementations may be added in the future.

To start the server with a file store, you need to provide two parameters:

```sh
nats-streaming-server -store file -dir datastore
```
The parameter `-store` indicates what type of store to use, in this case `file`. The other (`-dir`) indicates in which directory the state should be stored.

The first time the server is started, it will create two files in this directory, one containing some server related information (`server.dat`) another to record clients information (`clients.dat`).

When a streaming client connects, it uses a client identification, which the server registers in this file. When the client disconnects, the client is cleared from this file.

When the client publishes or subscribe to a new subject (also called channel), the server creates a sub-directory whose name is the subject. For instance, if the client subscribes to `foo`, and assuming that you started the server with `-dir datastore`, then you will find a directory called `datastore/foo`. In this directory you will find several files: one to record subscriptions information (`subs.dat`), and a series of files that logs the messages `msgs.1.dat`, etc...

The number of sub-directories, which again correspond to channels, can be limited by the configuration parameter `-max_channels`. When the limit is reached, any new subscription or message published on a new channel will produce an error.

On a given channel, the number of subscriptions can also be limited with the configuration parameter `-max_subs`. A client that tries to create a subscription on a given channel (subject) for which the limit is reached will receive an error.

Finally, the number of stored messages for a given channel can also be limited with the parameter `-max_msgs` and/or `-max_bytes`. However, for messages, the client does not get an error when the limit is reached. The oldest messages are discarded to make room for the new messages.

#### File Store Options

As described in the [Configuring](https://github.com/nats-io/nats-streaming-server#configuring) section, there are several options that you can use to configure a file store.

Regardless of channel limits, you can configure message logs to be split in individual files (called file slices). You can configure
those slices by number of messages it can contain (`--file_slice_max_msgs`), the size of the file - including the corresponding index file
(`--file_slice_max_bytes`), or the period of time that a file slice should cover - starting at the time the first message is stored in
that slice (`--file_slice_max_age`). The default file store options are defined such that only the slice size is configured to 64MB.

Note: If you don't configure any slice limit but you do configure channel limits, then the server will automatically
set some limits for file slices.

When messages accumulate in a channel, and limits are reached, older messages are removed. When the first file slice
becomes empty, the server removes this file slice (and corresponding index file).

However, if you specify a script (`--file_slice_archive_script`), then the server will rename the slice files (data and index)
with a `.bak` extension and invoke the script with the channel name, data and index file names.<br>
The files are left in the channel's directory and therefore it is the script responsibility to delete those files when done.
At any rate, those files will not be recovered on a server restart, but having lots of unused files in the directory may slow
down the server restart.

For instance, suppose the server is about to delete file slice `datastore/foo/msgs.1.dat` (and `datastore/foo/msgs.1.idx`),
and you have configured the script `/home/nats-streaming/archive_script.sh`. The server will invoke:

``` bash
/home/nats-streaming/archive_script.sh foo datastore/foo/msgs.1.dat.bak datastore/foo/msgs.2.idx.bak
```
Notice how the files have been renamed with the `.bak` extension so that they are not going to be recovered if
the script leave those files in place.

As previously described, each channel corresponds to a sub-directory that contains several files. It means that the need
for file descriptors increase with the number of channels. In order to scale to ten or hundred thousands of channels,
the option `fds_limit` (or command line parameter `--file_fds_limit`) may be considered to limit the total use of file descriptors.

Note that this is a soft limit. It is possible for the store to use more file descriptors than the given limit if the
number of concurrent read/writes to different channels is more than the said limit. It is also understood that this
may affect performance since files may need to be closed/re-opened as needed.

## Clients

Here is the list of NATS Streaming clients, supported by Apcera. We may add additional supported streaming clients in the future, and encourage community-contributed clients.

- [C#](https://github.com/nats-io/csharp-nats-streaming)
- [Go](https://github.com/nats-io/go-nats-streaming)
- [Java](https://github.com/nats-io/java-nats-streaming)
- [Node.js](https://github.com/nats-io/node-nats-streaming)

## License

The MIT License (MIT)

Copyright (c) 2016-2017 Apcera Inc.

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.


[License-Url]: http://opensource.org/licenses/MIT
[License-Image]: https://img.shields.io/badge/License-MIT-blue.svg
[Build-Status-Url]: http://travis-ci.org/nats-io/nats-streaming-server
[Build-Status-Image]: https://travis-ci.org/nats-io/nats-streaming-server.svg?branch=master
[Coverage-Url]: https://coveralls.io/r/nats-io/nats-streaming-server?branch=master
[Coverage-image]: https://coveralls.io/repos/github/nats-io/nats-streaming-server/badge.svg?branch=master&t=kIxrDE
[ReportCard-Url]: http://goreportcard.com/report/nats-io/nats-streaming-server
[ReportCard-Image]: http://goreportcard.com/badge/github.com/nats-io/nats-streaming-server
[github-release]: https://github.com/nats-io/nats-streaming-server/releases/
