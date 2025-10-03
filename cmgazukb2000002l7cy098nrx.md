---
title: "Redis and the Magic of ZSET"
seoTitle: "Redis and the Magic of ZSET"
seoDescription: "The post shows how Redis utilizes skiplist to get a powerful zset  data structure"
datePublished: Fri Oct 03 2025 15:23:21 GMT+0000 (Coordinated Universal Time)
cuid: cmgazukb2000002l7cy098nrx
slug: redis-and-the-magic-of-zset
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1759504852708/a58adaba-4f6d-4cbc-bf5a-7f7f4881c4ab.png
tags: redis, programming, databases, software-engineering, datastructure

---

Redis is often praised as one of the simplest yet most powerful databases. Its key-value model is blazing fast for direct lookups, but when it comes to querying by ranges, ranks, or sorting, Redis doesn’t give you much out of the box. Unless, of course, you know about **ZSETs**.

A ZSET (sorted set) is Redis’ answer to the question: “How do I efficiently query ordered data?” With ZSETs you can get things like *top N players by score*, *all values between X and Y*, or *the rank of a specific member* in logarithmic time. Under the hood, ZSETs are powered by a surprisingly elegant data structure: the **skiplist**.

This post breaks down how skiplists work, why Redis chose them, and how insertion, removal, and rank queries actually operate. By the end, you’ll understand why ZSETs can feel like magic when combined with normal Redis keys.

## Skiplist Basics: A Tower of Linked Lists

At first glance, a skiplist looks like a bunch of linked lists stacked on top of each other. Each element (called a **node**) may appear in multiple levels. The higher the level, the fewer nodes exist there, acting like express lanes for fast searching.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1759504511396/051f8782-c253-4b84-bbf2-f5b32aa06e59.png align="center")

Here’s what makes a skiplist tick:

* **Multilevel Nodes**: Each node randomly decides how tall it is. Some are short, some are skyscrapers. Tall nodes act as shortcuts when searching.
    
* **Probability** `p`: This is the coin flip that determines how high a node grows. Redis uses `p = 0.25`. That means ~25% of nodes appear on level 2, ~6% on level 3, ~1.5% on level 4, etc.
    
* **MaxLevel**: The upper bound on how tall nodes can be. Redis picked 32. With billions of elements, you’re still safe from degenerating into O(n).
    
* **Forward pointers**: At each level, nodes point to the next node in order.
    
* **Spans**: Each forward pointer doesn’t just link to the next node, it also says *how many nodes are being skipped at this level*. That’s the secret sauce for rank queries.
    
* **Backward pointer**: At the lowest level, nodes also point back to their predecessor. This helps with reverse scans.
    

The result: searching, inserting, and deleting are all expected **O(log n)** operations. Unlike trees, skiplists achieve this with very simple code and without painful balancing logic. The trade-off is that performance is *probabilistic*, but the odds of a bad case are astronomically small.

In the next few paragraphs, I would like to show how skiplists work. I am going to keep implementation out of the main text to avoid overloading readers, but it is included in the [repository](https://github.com/jorzel/skiplist).

## How Random Levels Are Chosen

When inserting a new element, we don’t deterministically assign a level. Instead, we flip a weighted coin until it comes up tails.

* Start at level 1.
    
* While `rand() < p`, increase level.
    
* Stop when tails, or when `maxLevel` is reached.
    

```go
rnd := rand.New(rand.NewSource(time.Now().UnixNano()))
// randomLevel generates a random level for a new node
// with probability p for each level
// the lower the p, the less likely to have high levels
func randomLevel() int {
	lvl := 1
	for lvl < maxLevel && rnd.Float64() < p {
		lvl++
	}
	return lvl
}
```

This randomness ensures a geometric distribution of node heights. Most nodes are small, a few are tall, and a tiny handful become “towers” that accelerate searches. With `p=0.25` and `maxLevel=32`, you expect fewer than one node out of 10^18 to hit the max.

---

## Insert Member

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1759502791502/727f4f7a-170b-4d26-aed9-28ab45789ac3.png align="center")

To insert `(score, member)`:

1. Start from the header (`head`) node (a sentinel that has no real value).
    
2. At each level, move forward until you would overshoot the target.
    
3. Keep track of these **insertion points**.
    
4. Flip coins to decide the level of the new node.
    
5. Splice it in by updating forward pointers and spans.
    

Because spans tell us how many nodes each jump covers, they are carefully adjusted during insertion.

## Remove Member

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1759502818993/55525a4c-b01f-48f8-af60-7b2d8a1b8d8f.png align="center")

Removal is similar:

1. Find the **nodes to remove** at each level.
    
2. Update their forward pointers to skip over the target (the node before `node-to-remove` should now point to `noid-to-remove.forward`.
    
3. Adjust spans accordingly.
    
4. Fix backward pointers.
    

If the removed node was the tallest, the overall skiplist level may shrink.

---

## Rank: Counting Skips

Rank is where spans shine. Imagine asking: *What’s the index of member X in sorted order?*

During the search for `(score, member)`:

* Every time we move forward at level `i`, we add that edge’s `span[i]` to a running total.
    
* When we finally land on the node, the accumulated total is its rank.
    

That’s O(log n) rank calculation, something that would otherwise require walking the entire base linked list.

### **Case 1:** `GetRank(8, "8")`

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1759502471849/06a695a6-2b35-44d2-a75f-1db149041291.png align="center")

Start: `x=head`, `rank=0`.

**Level 4**

* `head.forward[4] = 11`.
    
* Is `11 < 8`? → no.
    
* Stay put.
    

**Level 3**

* Same check: `11 < 8`? → no.
    
* Stay put.
    

**Level 2**

* `head.forward[2] = 8`.
    
* `8 < 8`? → no.
    
* Stay put.
    

**Level 1**

* `head.forward[1] = 8`.
    
* `8 < 8`? → no.
    
* Stay put.
    

**Level 0**

* `head.forward[0] = 5`.
    
* `5 < 8`? → yes → take jump.
    
    * `rank += head.span[0] = 1`
        
    * `rank = 1`
        
    * `x = 5`
        
* Next forward: `x.forward[0] = 8` → `8 < 8`? → no.
    

After loop: `x = x.forward[0] = 8`. Target matches → return `rank = 1`.

### Case 2: `GetRank(16, "16")`

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1759502541198/dd9b894e-ed32-4bd6-a950-ed9dc0958455.png align="center")

Start: `x=head`, `rank=0`.

**Level 4**

* `head.forward[4] = 11`.
    
* `11 < 16`? → yes → take jump.
    
    * `rank += head.span[4] = 3`
        
    * `rank = 3`
        
    * `x = 11`
        
* Next forward: `x.forward[4] = nil` → stop inner loop.
    

**Level 3**

* From `x=11`, `x.forward[3] = nil`. → stay.
    

**Level 2**

* From `x=11`, `x.forward[2] = nil`. → stay.
    

**Level 1**

* From `x=11`, `x.forward[1] = 16`.
    
* `16 < 16`? → no. → stay.
    

**Level 0**

* From `x=11`, `x.forward[0] = 16`.
    
* `16 < 16`? → no.
    

After loop: `x = x.forward[0] = 16`. Target matches → return `rank = 3`.

## Redis’ Trick

Redis doesn’t just use a skiplist. It combines:

* A **dict (hash table)**: `member → node`, for O(1) direct lookups.
    
* A **skiplist**: keeps members ordered by score and member name for O(log n) range queries and rank calculations.
    

This hybrid gives the best of both worlds. Combining those two structures allows us to easily implement basic zset operations like: `ZAdd`, `ZRem`, `ZRank` and `ZRange`.

```go
import "github.com/jorzel/skiplist/skiplist"

type Zset struct {
    sl skiplist.Skiplist
    dict map[string]float64
}

// Add or update member
// ZAdd https://redis.io/docs/latest/commands/zadd/
func (z *Zset) Add(score float64, member string) {
    if n, ok := z.dict[member]; ok {
        z.sl.Remove(n.score, member)
    }
    n := z.sl.Insert(score, member)
    z.dict[member] = n
}

// Remove member
// ZRem https://redis.io/docs/latest/commands/zadd/
func (z *Zset) Remove(member string) bool {
    n, ok := z.dict[member]
    if !ok {
        return false
    }
    ok = z.sl.Remove(n.score, member)
    if ok {
        delete(z.dict, member)
    }
    return ok
}

// Rank returns zero-based rank, or -1 if not present
// ZRank https://redis.io/docs/latest/commands/zrank/
func (z *ZSet) Rank(member string) int {
    score, ok := z.dict[member]
    if !ok { return -1 }
    return z.sl.GetRank(score, member)
}

// Range by rank inclusive [start,stop], supports negative indices like Redis
// ZRange https://redis.io/docs/latest/commands/zrank/
func (z *ZSet) Range(start, stop int) []string {
    n := z.sl.length
    if n == 0 { return nil }

    // normalize negative indices
    if start < 0 { start = n + start }
    if stop < 0 { stop = n + stop }
    if start < 0 { start = 0 }
    if stop < 0 { return nil }
    if start > stop || start >= n { return nil }
    if stop >= n { stop = n-1 }

    res := make([]string, 0, stop-start+1)
    node := z.sl.GetByRank(start)
    for i := start; i <= stop && node != nil; i++ {
        res = append(res, node.member)
        node = node.forward[0]
    }
    return res
}
```

This mirrors how Redis’ ZSET is implemented in C: `dict + zskiplist`.

## Conclusion

ZSETs are surprisingly versatile once you know what powers them:

* Leaderboards (players ranked by score).
    
* Time-series queries (timestamps as scores).
    
* Priority queues (scores as priorities).
    
* Secondary indexes (scores encode derived values).
    

Even though Redis doesn’t give you SQL-like querying, a clever combination of keys + ZSETs makes it possible to implement efficient searching patterns.

Redis ZSETs look simple on the surface, but under the hood, they rely on a beautifully clever data structure: the skiplist. With random levels, forward pointers, spans, and a hybrid design that pairs a dict with the skiplist, Redis achieves fast lookups, inserts, deletes, and rank queries—all in O(log n).

Compared to balanced trees, skiplists are easier to code, almost as fast, and probabilistically robust. That’s why Redis chose them. And now, hopefully, you see why ZSET is one of the most powerful features in Redis: it turns a plain key-value store into a fast, query-capable data structure.

Once again, the `skiplist` implementation can be found [here](https://github.com/jorzel/skiplist/tree/main).