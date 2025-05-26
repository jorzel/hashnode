---
title: "Understanding Consistent Hashing: Concepts, Pitfalls, and Real-World Use Cases"
seoTitle: "Understanding Consistent Hashing: Concepts, Pitfalls, and Use-cases"
seoDescription: "Consistent hashing is a well-established technique that addresses this challenge with minimal disruption."
datePublished: Mon May 26 2025 11:13:24 GMT+0000 (Coordinated Universal Time)
cuid: cmb4zodbr000709lec60841nz
slug: understanding-consistent-hashing-concepts-pitfalls-and-real-world-use-cases
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1748257869868/c8dff6b7-901b-44f7-bb6f-7e22c8bffdae.png
tags: software-development, programming-blogs, programming, software-architecture, software-engineering

---

### 📌 TL;DR

* **Consistent hashing** is a strategy for evenly assigning keys (like `user_id`, `session_id`) to servers with minimal disruption when servers are added or removed.
    
* **Naive approaches** (e.g., `hash(key) % N`) cause massive remapping on any change to the server pool.
    
* A **hash ring** maps keys and servers into a circular hash space, improving assignment stability.
    
* **Virtual nodes** (multiple positions per server) help distribute load more evenly and avoid hot spots.
    
* Common use cases:
    
    * Sticky session routing (stateful services)
        
    * Database sharding
        
    * Canary or feature-flag rollouts
        
* Choose a good **hash function** (e.g., MurmurHash, xxHash) to avoid key clustering and ensure fair distribution.
    

## Intro

In distributed systems, assigning keys (like `user_id`, `session_id`, or data records) to servers efficiently and reliably is a fundamental challenge, especially as the set of available servers changes over time. **Consistent hashing** is a well-established technique that addresses this challenge with minimal disruption.

In this post, we’ll explore how consistent hashing works, starting with naive approaches and the problems they create. We’ll then walk through improvements using hash rings and virtual nodes to enhance stability and balance. Finally, we'll dive into practical use cases where consistent hashing proves invaluable, such as sticky session routing, data sharding, and canary-style rollouts.

## Problem Definition

You are designing a distributed system (e.g., a web backend, cache cluster, or queue processor).  
You receive an unknown, arbitrary key, such as:

* `user_id`
    
* `session_id`
    
* `IP address`
    
* `file_hash`
    
* etc.
    

Your goal is to deterministically and evenly assign that key to a physical server instance, such that:

* The same key always maps to the same server
    
* The assignment is balanced across all servers
    
* When servers are added or removed, only a small number of keys are remapped
    

## Naive Approach

The first approach could be to use simple `modulo` function to determine a server node based on an incoming hash of incomming key.

```python
class NaiveSharding:
    def __init__(self, nodes):
        self.nodes = nodes

    def get_node(self, key):
        idx = hash(key) % len(self.nodes)
        return self.nodes[idx]
```

It provides uniform distribution of keys (assuming a good hash function), which makes it effective for even load balancing across nodes. However, it has a critical weakness. Adding or removing a node reshuffles nearly all keys. See the following example of how key assignments change when we change the number of servers from 3 to 5.

| `user_id` | `server_instance` (3 nodes) | `server_instance`(5 nodes) |
| --- | --- | --- |
| 101 | `hash(101) % 3 = 2` → **Server-2** | `hash(101) % 5 = 1` → **Server-1** |
| 202 | `hash(202) % 3 = 1` → **Server-1** | `hash(202) % 5 = 2` → **Server-2** |
| 303 | `hash(303) % 3 = 0` → **Server-0** | `hash(303) % 5 = 3` → **Server-3** |
| 404 | `hash(404) % 3 = 1` → **Server-1** | `hash(404) % 5 = 4` → **Server-4** |
| 505 | `hash(505) % 3 = 1` → **Server-1** | `hash(505) % 5 = 0` → **Server-0** |

When the number of servers changes from 3 → 5, every single user is reassigned to a different server.

## Consistent Hashing

Consistent hashing is a strategy for assigning keys to servers in a way that ensures stable mappings and minimizes disruption when servers are added or removed.

The core idea behind consistent hashing is the **hash ring**. Imagine the hash space as a circle (ring) from 0 to MAX\_HASH.

* Each server is hashed and placed on the ring (large dot).
    
* Each key is also hashed and placed on the ring (small dots).
    
* A key is assigned to the first server clockwise on the ring (small dots are assigned to large dots with the same color).
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1748252751116/97e65ec7-d04f-46be-92f2-28f23641a719.png align="center")

What happens when we add 2 servers?

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1748252880408/cec68f29-627c-499c-be6d-33371193dedb.png align="center")

Only a part of the keys assigned previously to GREEN and BLUE servers are now assigned to the new YELLOW and VIOLET servers (~33% of keys) were remapped. Only keys between two points on the ring (servers) are affected, and most key-to-server assignments remain unchanged.

Although consistent hashing offers excellent stability when adding or removing servers, it introduces a trade-off: the distribution of keys across nodes can become uneven. In a standard hash ring, each server is hashed once and placed on the ring at a single point. However, this placement is entirely dependent on the output of the hash function. As a result, some servers may end up responsible for much larger segments of the ring than others. See the following example of 3 servers.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1748253326475/8485b4dd-503b-449a-9fb1-0112779e747f.png align="center")

Almost all keys landed on the GREEN server (~81%), while only ~2% landed on the RED one.

This imbalance leads to **hot spots**. Certain servers receive a disproportionately high share of the traffic, while others remain underutilized.

## Virtual Nodes

To address the problem of uneven distribution, many implementations introduce the concept of virtual nodes (or replicas). Instead of placing each physical server on the ring once, we hash it multiple times using different suffixes (e.g., `server-1#1`, `server-1#2`, etc.). Each of these virtual nodes is placed independently on the ring. See the example of 3 servers with 25 virtual nodes for each.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1748253732843/3e4a5f5c-35ae-48ce-bc68-5f261c6c1674.png align="center")

Now we have quite an even distribution (31-35% for each server). We are also flexible to add and remove servers without remapping a large part of the keys. Let’s examine what happens when we add 2 additional servers.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1748254080807/90fe134d-51df-4052-923d-d81c69544b8f.png align="center")

The distribution is still even, and ~43% of keys were remapped.

The number of virtual nodes per server controls how evenly the load is distributed. More virtual nodes lead to a smoother, more uniform key distribution and reduce the chance of any one server becoming a hot spot. They also improve stability by minimizing the number of keys that need to be reassigned when a node is added or removed.

Beyond around 100–200 virtual nodes per physical server, the improvements in load distribution and assignment stability become minimal. Increasing virtual nodes further mainly adds memory and computational overhead without meaningful benefits, so it’s usually not worth the extra cost

## Common Use-cases

The three most common applications of consistent hashing are load balancing with sticky sessions, data sharding, and canary-like traffic control.

In the first case, sticky session load balancing, we assign each client (based on something like `user_id` or IP) to a specific server and consistently route them there for future requests. This is especially useful when servers are stateful or maintain in-memory session data, as it avoids expensive lookups or session rehydration.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1748257396675/9ea14da0-fb5a-4087-a422-34478a6c31cb.png align="center")

The second use case is sharding, where we split large, loosely related datasets (e.g., by `user_id` or `client_id`) across multiple database instances to improve scalability and performance. Each shard is responsible for a portion of the keyspace, ensuring efficient distribution and parallelism.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1748257407852/af2ca68e-5b18-482f-9393-2c731590f8ae.png align="center")

The third application is canary-style traffic splitting. Here, we use consistent hashing to deterministically assign a hash of each `user_id` to a percentage-based threshold. For example, if we want to enable a feature for 40% of users, only those whose hash falls below the 40% mark will be enrolled. This ensures a stable, user-specific rollout, rather than random per-request percentage splits, so each user consistently sees the same experience.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1748257416713/f2c0fe8d-3f53-4f98-b4ed-6a8085121fe0.png align="center")

Consistent hashing is used in real-world systems like Amazon DynamoDB, Apache Cassandra, and distributed caches like Memcached or Redis Cluster to ensure resilience and balance at scale.

## Conclusion

Consistent hashing is a powerful and versatile technique that every software engineer should have in their toolbox. It provides a resilient and scalable way to assign keys to nodes in dynamic environments, ensuring minimal disruption when nodes are added or removed. Whether you're building stateful web services, sharding databases, or rolling out new features gradually, consistent hashing helps keep your system balanced and predictable.

Keep in mind, though, that the quality of your hashing algorithm directly impacts how well consistent hashing performs. Choosing a hash function with uniform distribution (like MurmurHash or xxHash) can significantly reduce hot spots and improve fairness across the ring. While cryptographic hashes (like SHA-1) are commonly used, they may add unnecessary overhead if you don’t need security guarantees.

In short, combining a solid hash function with a well-designed hash ring (and virtual nodes) gives you a simple yet powerful tool for building reliable distributed systems.