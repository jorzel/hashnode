---
title: "The subtle difference between shallow and deep copy"
seoTitle: "The subtle difference between shallow and deep copy"
seoDescription: "Failing to understand the difference between deep and shallow copying can lead to unintended side effects, especially in concurrent environment."
datePublished: Fri Jun 21 2024 20:52:56 GMT+0000 (Coordinated Universal Time)
cuid: clxp63w7700010amcewum0y73
slug: the-subtle-difference-between-shallow-and-deep-copy
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/4yOgRb_b_i4/upload/e50aa5639121ff0748960a25cde86240.jpeg
tags: software-development, golang, software-engineering

---

## Introduction

In many programming languages, using the assignment operator (`=`) to duplicate arrays or maps (or similar data structures) does not create a new, independent copy. Instead, it creates a new reference to the same underlying data. This can lead to unintended side effects when the original data is modified. In software terminology, we use terms "deep copy" and "shallow copy" to describe different methods of duplicating objects:

* A deep copy of an object creates a new object and recursively copies all objects found in the original. This means that the new object is a complete copy, with no shared references to the objects in the original.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1719163113030/d749e871-4e34-403f-89f2-a2ea842a5b71.png align="center")

* A shallow copy of an object creates a new object but inserts references into it to the objects found in the original. Therefore, the new object is a copy of the original object's structure but not of the objects that the structure references.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1719163101055/3607e08b-b8c4-40fd-bebc-2930d2bfb89d.png align="center")

In this short article, I would like to show how using the wrong copying approach could lead to unintended consequences in our programs.

## Simple example

*NOTE: We use Golang to present it, but the same behavior could be seen in other popular languages like Javascript, Python, etc.*

What happens when we assign a hash map to another one?

```go
package main

import "fmt"

func main() {
	print("SHALLOW COPY")
	headers := map[string]string{
		"Content-Type": "application/json",
	}
	fields := headers
	fmt.Println("Fields:", fields)
	fmt.Println("Headers:", headers)
	fmt.Println("ADD AUTHORIZATION")
	fields["Authorization"] = "Bearer 123"
	fmt.Println("Fields:", fields)
	fmt.Println("Headers:", headers)
}
```

We initialized a `headers` map, assigned `headers` to `fields`, and eventually added `Authorization` to `fields`. Our intention was probably to copy values from `headers` to `fields` and made changes that affect `fields` but not `headers`. However, we get something like that:

```bash
SHALLOW COPY
Headers: map[Content-Type:application/json]
ADD AUTHORIZATION
Fields: map[Authorization:Bearer 123 Content-Type:application/json]
Headers: map[Authorization:Bearer 123 Content-Type:application/json]
```

By adding `Authorization` to `fields` we modified not only `fields` but also `headers`. The reason is that the assignment operator did shallow copy and we did not duplicate the object but only copied the reference to `headers`. Consequently, `headers` and `fields` pointed to the same object.

```go
package main

import "fmt"

func main() {
	fmt.Println("DEEP COPY")
	headers := map[string]string{
		"Content-Type": "application/json",
	}
	fmt.Println("Headers:", headers)
	fields := deepCopy(headers)
	fmt.Println("ADD AUTHORIZATION")
	fields["Authorization"] = "Bearer 123"
	fmt.Println("Fields:", fields)
	fmt.Println("Headers:", headers)
}

func deepCopy(m map[string]string) map[string]string {
	newMap := make(map[string]string)
	for k, v := range m {
		newMap[k] = v
	}
	return newMap
}
```

We replaced the assignment operator (shallow copy) by `deepCopy` function (deep copy). Now `headers` and `fields` just after making a copy have the same values, but are represented by two different objects.

```go
DEEP COPY
Headers: map[Content-Type:application/json]
ADD AUTHORIZATION
Fields: map[Authorization:Bearer 123 Content-Type:application/json]
Headers: map[Content-Type:application/json]
```

The `headers` is not bound to `fields` and any modification of `fields` will not affect `headers`.

## Concurrent access

The above example was trivial and straightforward. However, things get tricky when handling many concurrent requests in our applications. In such scenarios, errors become non-deterministic, and identifying the cause of unexpected behavior becomes much harder to debug.

```go
package main

import (
	"fmt"
	"time"

	"math/rand/v2"
)

func main() {
	fmt.Println("CONCURRENCY")
	headers := map[string]string{
		"Content-Type": "application/json",
	}
	c := client{
		headers: headers,
	}

	operations := 10
	for i := 0; i < operations; i++ {
		simulateLatency()
		go c.sendRequest(fmt.Sprintf("Bearer %d", i))
	}
	time.Sleep(1 * time.Second)
}

type client struct {
	headers map[string]string
}

func (c *client) sendRequest(key string) {
	fields := c.headers
	fields["Authorization"] = key
	simulateLatency()
    fmt.Println(fmt.Sprintf("key: %s, fields: %s", key, fields))
}


func simulateLatency() {
	interval := rand.IntN(5)
	time.Sleep(time.Duration(interval) * time.Millisecond)
}
```

In this example, there is a small time gap between setting `Authorization` field and printing values (or sending a request over the network in a real system). This gap and using a shallow copy of the `headers` map can lead to a situation where another concurrent operation overwrites the `Authorization` field. In the end, the `Authorization` field is not set to the expected value.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1719163128380/33a941e0-9266-4b2d-ad62-be9b0810c11c.png align="center")

When we run the code, we get something like this:

```bash
CONCURRENCY
key: Bearer 0, fields: map[Authorization:Bearer 0 Content-Type:application/json]
key: Bearer 1, fields: map[Authorization:Bearer 1 Content-Type:application/json]
key: Bearer 2, fields: map[Authorization:Bearer 2 Content-Type:application/json]
key: Bearer 3, fields: map[Authorization:Bearer 4 Content-Type:application/json]
key: Bearer 5, fields: map[Authorization:Bearer 4 Content-Type:application/json]
key: Bearer 4, fields: map[Authorization:Bearer 4 Content-Type:application/json]
key: Bearer 6, fields: map[Authorization:Bearer 6 Content-Type:application/json]
key: Bearer 7, fields: map[Authorization:Bearer 7 Content-Type:application/json]
key: Bearer 8, fields: map[Authorization:Bearer 9 Content-Type:application/json]
key: Bearer 9, fields: map[Authorization:Bearer 9 Content-Type:application/json]
```

As we expected, some of the requests have incorrect `Authorization` field, because `headers` has been shared between concurrent operations.

The solution here is simple: use a deep copy instead of a shallow one. Although using deep copy seems straightforward when dealing with concurrent access to shared resources, even experienced developers can make mistakes. It is crucial to be extremely careful when duplicating mutable structures to avoid errors.

## Conclusion

Failing to understand the difference between deep and shallow copying can lead to unintended side effects, especially in concurrent or multi-threaded environments where a shared state can be inadvertently modified. It is crucial to choose the correct type of copying based on the needs of your application to ensure data integrity and prevent subtle bugs.