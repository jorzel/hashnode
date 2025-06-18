---
title: "Building your Redis From Scratch: Multi-Node Architecture & Initial Communication"
seoTitle: "Building your Redis From Scratch: Multi-Node Architecture"
seoDescription: "In this post, we begin the journey into building a multi-node Redis architecture. "
datePublished: Wed Jun 18 2025 09:27:38 GMT+0000 (Coordinated Universal Time)
cuid: cmc1r0yqv000i02l181jacaya
slug: building-your-redis-from-scratch-multi-node-architecture-and-initial-communication
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1750238792286/b30fa403-5af4-4f9a-b924-867f5ae23087.png
tags: redis, programming, databases, software-engineering, learning-journey

---

In the previous post, we built a simple, single-node Redis-compatible server that supports basic command handling, a RESP protocol parser, and a lightweight in-memory storage engine. While this setup is functional and instructive, it doesn't reflect the challenges faced in real-world distributed environments.

In this post, we begin the journey into building a multi-node Redis architecture. This is a much broader and more complex topic, so we’ll cover it over a series of articles.

We’ll start here by:

* Introducing the concept of a distributed Redis setup,
    
* Explaining the roles of master and replica nodes,
    
* Discussing why multi-node replication is useful,
    
* Implementing initial communication between the master and replicas.
    

In the next posts, we’ll dive deeper into:

* The mechanics of command replication from the master to replicas,
    
* How acknowledgments (offset tracking) are used to ensure consistency,
    

Let’s begin by understanding why a distributed Redis setup is worth the added complexity.

## What is Multi-Node Setup?

In a single-node setup, we run just one instance of the Redis server. This node is responsible for handling all read and write operations, and it manages the entire dataset in memory. While this is straightforward and often sufficient for local development or small-scale applications, it introduces a major single point of failure and doesn’t scale well under heavy load.

To solve these limitations, Redis supports multi-node architectures. The most common multi-node pattern is a master-replica setup:

* A single master node handles all write operations (and can also serve reads).
    
* One or more replica nodes (also called slaves in older terminology) replicate data from the master and serve only read requests.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1750072241791/fb855a56-9ae8-4071-921e-ff114aad47e5.png align="center")

## Why Do You Need a Multi-Node Setup?

### Resilience & High Availability

Imagine your Redis server crashes unexpectedly, due to hardware failure, a bug, or a system upgrade gone wrong. With only a single Redis node, this means complete downtime. Your clients can't read from or write to the system. Services that depend on Redis will begin to fail.

Replication solves this by introducing redundancy. In a replicated setup:

* One node acts as the master, handling all writes.
    
* One or more nodes act as replicas (or secondaries), receiving and applying changes from the master.
    

If the master fails, a replica can be promoted to become the new master. This allows the system to continue functioning with minimal disruption and no data loss, assuming the replicas were fully synchronized.

Replicas can also act as live backups, improving disaster recovery and fault tolerance.

### Scalability (Especially for Read-Heavy Workloads)

Another key benefit of replication is horizontal scaling for read traffic.

In many systems, read operations far outnumber writes (e.g., fetching cached data, session information, configuration values). A single Redis instance might become a bottleneck as read traffic grows.

With read replicas:

* Clients can be distributed across multiple replica nodes.
    
* Load is balanced, and the master is freed up to focus on writes.
    
* The system can serve more concurrent users without degrading performance.
    

This setup enables Redis to handle massive throughput at low latency, which is especially important in web-scale applications.

## Multi-Node Start

In this post, we want to present a simple multi-node setup. At this stage, we're not yet concerned with data replication or consistency guarantees. Instead, the goal is to connect two Redis-like instances — one acting as a master, the other as a replica.

## Startup Configuration

To implement the replica node, we'll start by reusing most of the existing `MasterServer` code. At this stage, both master and replica share the same underlying server implementation — the primary distinction is their role. We’ll introduce a `role` flag (e.g., `"master"` or `"replica"`) to track this.

```go
type ReplicaServer struct {
	listener       net.Listener
	commandParser  protocol.CommandParser
	commandHandler commands.CommandHandler
	cfg         *config.Config
	role           string
}
```

In this post, the main functional difference is that the replica will initiate a handshake with the master when it starts up. Aside from this handshake logic, the master and the replica behave the same. In future posts, we'll begin to diverge their responsibilities more clearly, introducing different handlers and replication-specific logic depending on the role.

This incremental approach allows us to keep the architecture simple while building toward a fully functional multi-node setup.

In the `main` function, we use the presence of the `--replicaof` argument to determine whether the server should start as a **replica** or as a **master**.

```go
cfg, err := getInitSpecsFromArgs()
if err != nil {
	os.Exit(1)
}

var srv server.Server
if cfg.ReplicaOf == nil {
	srv, err = server.NewMasterServer(cfg)
} else {
	srv, err = server.NewReplicaServer(cfg)
}
err = srv.Start(ctx)
```

We need to modify our CLI to support an optional `--replicaof` flag. This is a straightforward change to the argument parser.

We extend our `getInitSpecsFromArgs()` function to parse the master address if provided:

```go
func getInitSpecsFromArgs() (*config.Config, error) {
	replicaOfString := flag.String("replicaof", "", "Address of the master server")
    // validation
    replicaOf := ...
    
    return &config.Config{
         ...
         replicaOf: replicaOf,
    } 
```

## Handshake

> ***⚠️ Disclaimer:*** The handshake process is relatively complex, involving several Redis-specific commands such as `REPLCONF`, `PSYNC`, and `FULLRESYNC`. To avoid overwhelming the reader with detailed implementation code at this stage, we focus on the overall flow and responsibilities of each participant in the handshake. For those interested in the internal mechanics or source code, the full implementation is available in the GitHub repository accompanying this series.

The handshake is the initial phase that establishes communication between a replica and a master.

It is triggered automatically when a node is started with the `--replicaof` flag, indicating it should act as a replica. The replica initiates the handshake by opening a TCP connection to the master.

Here is a simple sequence diagram for the handshake between a replica and the master.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1750087572936/d4f3f3f4-af16-4279-b522-d5c15124ee3d.png align="center")

1. `PING` – Sent by the **replica** to verify that the master is reachable.
    
2. `REPLCONF listening-port <port>` – Sent by the **replica** to inform the master of its port.
    
3. `REPLCONF capa psync2` – Sent by the **replica** to declare its replication capabilities.
    
4. `PSYNC ? -1` – Sent by the **replica** to request full synchronization.
    
5. `FULLRESYNC <replid> <offset>` – Sent by the **master** in response to the `PSYNC`, indicating a full sync will occur.
    
6. `Bulk of data representing RDB snapshot` - Sent by the **master**
    

The `REPLCONF` commands are simplified at this stage of development. They are received sequentially by the master server, and `+OK` response is sent to the replica.

In the response for `PSYNC` command sent by the replica, the master sends `FULLRESYNC` command with `replid` and `offset`. In Redis, the `replid` (replication ID) is a unique identifier assigned to a Redis master instance during its lifecycle, while `offset` representing the number of bytes propagated from the master to replicas. More about these two fields will be provided in the next post about data replication.

Right after the master sends the `FULLRESYNC` response, it immediately follows with a binary dump of its current dataset, typically in RDB (Redis Database) format.

Although the connection between the master and replica is established successfully, the replica cannot rely on its standard connection listener to receive data from the master. This is because the replica's listener is designed to accept *new* incoming connections, whereas the TCP connection to the master has already been initiated by the replica itself. Therefore, the master does not trigger the replica's listener when it sends data.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1750106737094/ae7399bf-b03c-4073-96e5-044d4ec8ff07.png align="center")

To handle this, the replica must set up a separate mechanism to process incoming data on the already-open connection with the master. In Go, this can be done by launching a dedicated goroutine (just after the handshake process completes) that continuously reads and handles replication data sent from the master over this existing connection.

```go
go rs.handleReplicationConnection(ctx, handshakeConn)
```

Within this new handler, the replica should be able to parse several types of incoming data from the master: the `FULLRESYNC` message (sent as a simple string), an RDB file (typically encoded in raw bytes or as a bulk string), and any subsequent replicated commands (sent using the `BulkArray` format).

In the previous post, we implemented a simple parser capable of handling `BulkArray`\-formatted commands. Now, to support the replication handshake, we need to enhance this parser to also handle the `FULLRESYNC` response and the optional RDB file that follows.

```go
case '*':
	// parse Bulk Array command
    // e.g. *3\r\n$3\r\nSET\r\n$5\r\nmykey\r\n$8\r\nmyvalue\r\n
case '+':
	// parse Simple String msg 
    // e.g. +FULLRESYNC 8f9e9d1e5a6b2cfb7c1d2e3f4a5b6c7d8e9f0a1b 0\r\n
case '$':
	// parse RDB file
    // e.g. $11\r\nREDIS0009
```

It’s important to note that your parser must be capable of handling **multiple commands within a single TCP message** - for example, a `FULLRESYNC` followed immediately by an RDB file. This can be tricky at first and was a source of confusion for me initially.

## Demonstration

Now we can demonstrate how the initial communication between master and replica takes place. At this stage, we haven't implemented data replication yet, so the only way to verify the process is by analyzing the logs of the master and replica servers respectively.

First, we start a master server:

```bash
./run.sh
{"level":"info","config":{"replica_of":null,"port":6379},"time":1750237929437,"message":"Starting configuration"}
{"level":"info","address":"[::]:6379","role":"master","time":1750237929437,"message":"Server listening on..."}
```

The server is listening for connections and has `role: master` .

In the separate terminal we run a replica:

```bash
./run.sh --port 6782 --replicaof "0.0.0.0 6379"
{"level":"info","config":{"replica_of":{"host":"0.0.0.0","port":6379},"port":6782},"time":1750238045519,"message":"Starting configuration"}
{"level":"info","replica_of":"0.0.0.0:6379","time":1750238045519,"message":"Server configured as replica of"}
{"level":"info","time":1750238045521,"message":"Connected to master server, starting handshake"}
{"level":"info","remote_addr":"127.0.0.1:6379","time":1750238045521,"message":"Connected to master server"}
{"level":"info","time":1750238045522,"message":"Ping successful, proceeding with REPLCONF handshake"}
{"level":"info","time":1750238045522,"message":"REPLCONF listening-port successful, proceeding with REPLCONF capa psync2"}
{"level":"info","time":1750238045522,"message":"REPLCONF capa psync2 successful, proceeding with PSYNC handshake"}
{"level":"info","time":1750238045522,"message":"Handshake initiated successfully"}
{"level":"info","address":"[::]:6782","role":"replica","time":1750238045522,"message":"Server listening on..."}
{"level":"info","role":"replica","local_addr":"127.0.0.1:50052","op":"handle_replication_conn","time":1750238045522,"message":"Handling new connection for replication"}
{"level":"info","role":"replica","local_addr":"127.0.0.1:50052","op":"handle_replication_conn","command":"FULLRESYNC","args":["test","0"],"index":0,"time":1750238045522,"message":"Parsed command"}
{"level":"info","command":"FULLRESYNC","offset":0,"repl_id":"test","time":1750238045522,"message":"Handling FULLRESYNC command"}
{"level":"debug","role":"replica","local_addr":"127.0.0.1:50052","op":"handle_replication_conn","data":"+FULLRESYNC test 0\r\n$88\r\nREDIS0011\ufffd\tredis-ver\u00057.2.0\ufffd\nredis-bits\ufffd@\ufffd\u0005ctime\ufffdm\b\ufffde\ufffd\bused-mem°\ufffd\u0010\u0000\ufffd\baof-base\ufffd\u0000\ufffd\ufffdn;\ufffd\ufffd\ufffdZ\ufffd","time":1750238045522,"message":"Received data for replication"}
{"level":"info","role":"replica","local_addr":"127.0.0.1:50052","op":"handle_replication_conn","time":1750238045522,"message":"Received RDB dump from master server"}
```

In the command line, we passed the master address with the port `0.0.0.0 6379` . While the master uses the default `6379` port, the replica runs on `6782`.

We see the sequence of commands sent as a part of the handshake. At the very end, the replica handles `FULLRESYNC` command and `rdb dump` file.

At the same time, on the master side, we also have new logs:

```bash
{"level":"info","remote_addr":"127.0.0.1:50052","time":1750238045520,"message":"Handling new connection"}
{"level":"debug","remote_addr":"127.0.0.1:50052","data":"*1\r\n$4\r\nPING\r\n","time":1750238045521,"message":"Received data"}
{"level":"info","remote_addr":"127.0.0.1:50052","command":"PING","args":[],"time":1750238045522,"message":"Parsed command"}
{"level":"debug","remote_addr":"127.0.0.1:50052","data":"*3\r\n$8\r\nREPLCONF\r\n$14\r\nlistening-port\r\n$4\r\n6782\r\n","time":1750238045522,"message":"Received data"}
{"level":"info","remote_addr":"127.0.0.1:50052","command":"REPLCONF","args":["listening-port","6782"],"time":1750238045522,"message":"Parsed command"}
{"level":"debug","remote_addr":"127.0.0.1:50052","data":"*3\r\n$8\r\nREPLCONF\r\n$4\r\ncapa\r\n$6\r\npsync2\r\n","time":1750238045522,"message":"Received data"}
{"level":"info","remote_addr":"127.0.0.1:50052","command":"REPLCONF","args":["capa","psync2"],"time":1750238045522,"message":"Parsed command"}
{"level":"debug","remote_addr":"127.0.0.1:50052","data":"*3\r\n$5\r\nPSYNC\r\n$1\r\n?\r\n$2\r\n-1\r\n","time":1750238045522,"message":"Received data"}
{"level":"info","remote_addr":"127.0.0.1:50052","command":"PSYNC","args":["?","-1"],"time":1750238045522,"message":"Parsed command"}
{"level":"info","time":1750238045522,"message":"Sending DB file to replica"}
```

The master handled `REPLCONF` commands and `PSYNC` . To conclude the handshake process, it sent `rdb file` to the replica.

## Conclusion

In this post, we laid the groundwork for supporting a multi-node Redis-like architecture.

We then implemented the first step of this architecture: the initial handshake between a replica and a master. While the nodes currently do not replicate data in real time, we demonstrated how to establish roles (master vs. replica), connect the nodes, and complete the replication handshake using RESP-based commands like `REPLCONF`, `PSYNC`, and `FULLRESYNC`.

At this point, both the master and replica are aware of each other and can exchange foundational sync data, including a stubbed RDB snapshot. This setup prepares us for the next phase: streaming real-time data changes from master to replicas, handling offsets, and building a consistent replication backlog.

In the next post, we’ll dive into the core mechanics of **command replication**. We’ll continue building toward a fully functional, fault-tolerant, and horizontally scalable Redis-compatible cluster.

As always, the full implementation and code examples are available in the [GitHub repository](https://github.com/jorzel/myredis/tree/blogpost2) accompanying this series.

Stay tuned!