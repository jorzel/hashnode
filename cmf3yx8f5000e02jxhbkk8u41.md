---
title: "Generating Unique and Sortable IDs in Distributed Systems"
seoTitle: "Generating Unique and Sortable IDs in Distributed Systems"
seoDescription: "How to generate a unique and sortable ID. Autoincrement, UUID, KSUID, ULID, Snowflake ID."
datePublished: Wed Sep 03 2025 12:43:21 GMT+0000 (Coordinated Universal Time)
cuid: cmf3yx8f5000e02jxhbkk8u41
slug: generating-unique-and-sortable-ids-in-distributed-systems
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/IUQH0SQ4jmk/upload/a78f628979500f6f87e593c87da2bba5.jpeg
tags: software-development, programming, software-architecture, distributed-system, software-engineering

---

When building distributed systems, generating unique identifiers becomes tricky. But why do we need them to be both unique AND sortable?

Uniqueness is obvious - you can't have duplicate IDs causing data corruption or overwriting records. But sortability brings some real benefits, e.g:

* **Natural ordering** - IDs created later sort after earlier ones, giving you chronological order for free
    
* **Database performance** - sorted inserts are faster, less index fragmentation, better query performance on ranges
    
* **Easier debugging** - you can instantly see which records are newer just by looking at the ID.
    

In the following post, I would like to present possible solutions for this problem.

## The Usual Suspects

Most developers reach for one of two standard approaches when they need unique IDs. Both seem reasonable at first, but they often show their limitations in distributed environments.

**The Database auto-increment** works great until you realize that properly divided services don't share databases. Each service would generate conflicting IDs, thereby breaking the uniqueness of the system.

**UUIDs** guarantee global uniqueness but don't provide sortability. Random UUIDs don't preserve creation order, which is often exactly what you need.

## Separate ID Service

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1756900772162/d0234404-ae39-4e76-b46a-c360cf36daea.png align="center")

You could create a dedicated service with access to an auto-increment database, essentially serving as a single source of truth for all other systems that generate IDs.

But here's the issue: it becomes a **single point of failure** for your entire system. It also adds a large service dependency to other systems. A simple operation like generating an ID suddenly gets additional network latency. It seems like great overhead for our task.

## Timestamp-Based IDs (The Smart Move)

Using a timestamp as part of the ID and leaving some bytes to randomize the ID seems more straightforward. Depending on your use case, you can choose one of the following options:

### **ULID**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1756901512841/617d0299-1d25-4fb1-b77d-7c335f11394a.png align="center")

A 128-bit identifier that combines a millisecond timestamp with randomness. Encoded in Base32, it produces a 26-character string that is globally unique and sortable by creation time without coordination.

```python
# pip install python-ulid

import ulid
import time

id1 = ulid.new()
id2 = ulid.new()

print(f"ULID 1: {id1}")  # 01K47RE8HM22176GF3QEA5VXC4
print(f"ULID 2: {id2}")  # 01K47REDKRVYZ67YBB275ZFHJB
print(f"Sortable: {id1 < id2}")  # True - chronological order preserved
```

### KSUID

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1756901562236/c3ea3fb6-b6a2-4428-bbf9-a5c741b80a95.png align="center")

A 160-bit identifier with a second-precision timestamp and 128 bits of randomness. Encoded in Base62 as a 27-character string, it’s highly collision-resistant and naturally ordered by time, making it well-suited for event logs and analytics.

```python
# pip install ksuid

from ksuid import ksuid
import time

# KSUID with second precision
id1 = ksuid()
time.sleep(1)  # 1 second delay for visibility
id2 = ksuid()

print(f"KSUID 1: {id1}")  # 1545e3c6325d397529c1124712f93aeaa3b2bf5c
print(f"KSUID 2: {id2}")  # 1545e3cac9c006f4bb64c753c68ff9143af33539
print(f"Sortable: {str(id1) < str(id2)}")  # True
```

### Snowflake ID

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1756903168857/681d2986-6155-4f1e-9797-de66e4f876b4.png align="center")

Snowflake fits everything into a 64-bit integer: 41 bits for timestamp, 10 bits for machine ID, and 12 bits for sequence number (plus 1 control bit at the very beginning). This gives you strictly monotonic, numeric IDs that are perfect for database primary keys. The catch? You need to coordinate machine IDs across your fleet.

```python
import time
import threading

class Snowflake:
    def __init__(self, machine_id: int):
        self.machine_id = machine_id & 0x3FF   # 10 bits for machine id
        self.sequence = 0
        self.last_timestamp = -1
        self.lock = threading.Lock()

    def _timestamp(self):
        return int(time.time() * 1000)  # milliseconds

    def next_id(self):
        with self.lock:
            timestamp = self._timestamp()

            if timestamp == self.last_timestamp:
                self.sequence = (self.sequence + 1) & 0xFFF  # 12 bits
                if self.sequence == 0:
                    while timestamp <= self.last_timestamp:
                        timestamp = self._timestamp()
            else:
                self.sequence = 0

            self.last_timestamp = timestamp
            return ((timestamp << 22) | (self.machine_id << 12) | self.sequence)


gen = Snowflake(machine_id=1)
id1 = gen.next_id()
id2 = gen.next_id()
print(id1) # 7368985504009687040
print(id2) # 7368985504009687041
print(id1 < id2) # True
```

## Summary

* **Snowflake**: Optimized for **compact, numeric IDs** (good for DB primary keys, ordered event streams). Requires coordination (machine IDs). Use **Snowflake** if you want **small, numeric IDs** (e.g., for databases, primary keys, social media posts).
    
* **ULID**: Aimed at being a **UUID replacement** → globally unique, sortable, and human-readable. No coordination needed. Use **ULID** if you want a **drop-in replacement for UUIDs** with better sortability.
    
* **KSUID**: Similar to ULID, but with **more randomness** and **Base62 encoding**, making it great for logs and analytics. Use **KSUID** if you want **extreme collision resistance** and don't mind slightly larger IDs.
    

Choosing the right ID scheme depends on your system’s priorities. By understanding the trade-offs (e.g. collision-free guarantees over time resolution), you can pick the approach that keeps your distributed system both scalable and easy to reason about.