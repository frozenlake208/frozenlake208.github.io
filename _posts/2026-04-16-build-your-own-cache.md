---
title: "Build Your Own Cache: Mutexes, TTL, and Surviving OOM Crashes"
description: "Build an in-memory cache in Go with thread-safe operations and TTL eviction"
date: 2026-04-16 08:35:00 +0700
categories: [Build From Scratch]
tags: [golang, caching, backend, build-from-scratch]
math: true 
mermaid: true 
image: https://pikuma.com/images/blog/understanding-computer-cache/thumbnail-990x540.jpg
---

## 🛠️ What is a Cache, *Really*?

In technical interviews, the easiest way to solve a read-heavy bottleneck is to confidently draw a box, label it "Redis," and say: *"I will add a cache in front of the database."* The interviewer nods. You nod. Everyone is happy.

But what if the interviewer follows up with: *"Okay, how does that cache actually work under the hood? What happens when two threads write to it at the same time? How does it not eat all our RAM?"* If you are a read-only developer, panic sets in. You know it stores data in memory, but the actual mechanics are a black box 😱.

Let's strip away the network protocols and the magic. At its absolute core, an in-memory cache is literally just a Hash Map (a dictionary) sitting in your server's RAM. However, if you deploy a raw Hash Map to production, two things will inevitably happen:
1. Two CPU threads will try to write to it at the exact same millisecond, corrupting the memory and crashing your app (Race Condition).
2. It will store data forever until your server runs out of physical memory and the OS aggressively kills your process (OOM Crash).

![meme1](https://images.viblo.asia/d2b7c4de-4e8e-4f7b-810b-12ddcf749779.jpg)

Today, we are opening our IDEs and writing Go. We are going to build our own in-memory Key-Value cache from scratch. We will start with a simple Go map, watch it break under concurrency, fix it with Mutex locks, and stop it from exploding by building a background TTL (Time-To-Live) sweeper.

Time to write some code.

---

## 🗺️ The Core Engine: A Raw Hash Map 
To build a cache, we need a data structure that allows us to retrieve data instantly. We cannot use an Array or a Linked List because searching them takes $O(N)$ time. If our cache has a million keys, iterating through them on every single user request would crush our CPU.

We need $O(1)$ time complexity. So, we start with a Hash Map (in Go, a `map`).

Let's write the initial, naive version of our cache: 

```go
package main

type Cache struct {
    data map[string]any
}

func NewCache() *Cache {
    return &Cache{
        data: make(map[string]any),
    }
}

func (c *Cache) Set(key string, value any) {
    c.data[key] = value
}

func (c *Cache) Get(key string) (any, bool) {
    val, found := c.data[key]
    return val, found
}
```

This looks perfect, right? We encapsulate a `map` inside a struct. We use `any` (Go's empty interface) so our cache can store strings, integers, or complex JSON structs. We have instant $O(1)$ reads and writes. If you test this with a single user, it works flawlessly. 

### 🛑 The Concurrency Trap
But remember, we are building a backend system, not a local script. In production, you don't have one user—you have thousands of concurrent requests. In Go, every incoming HTTP request is handled by its own lightweight thread (a goroutine).

What happens if Goroutine A tries to `Set("user_1")` and at the exact same nanosecond, Goroutine B tries to `Set("user_2")` 🤔?

They both attempt to mutate the exact same physical block of RAM simutaneously. This is a classic **Race Condition**.

![meme2](https://miro.medium.com/v2/resize:fit:640/format:webp/1*nil8GVReukeh6tHVWGZXtg.gif)

Most standard Hash Maps in programming languages (including Go's map) are **not thread-safe**. They are designed for speed, not concurrency. If Go's runtime detects two threads mutating a map at the same time, it doesn't just throw a warning—it intentionally panics and crashs your entire server tp prevent memory corruption: 

```plaintext
fatal error: concurrent map writes
```

Our raw cache is completely unsafe for production. To fix this, we need to physically force the CPU threads to wait in line. We need a Mutex.

## 🔒 Fixing the Crash: Mutex Locks
To stop the race condition, we need to guarantee that only one goroutine can mutate our map at any given time. We do this using a **Mutex** (Mutual Exclusion).

Think of a Mutex like the single key to a coffee shop bathroom. If Goroutine A has the key, it goes inside and locks the door. If Goroutine B arrives, it physically cannot enter. It must stand outside and wait until Goroutine A unlocks the door and hands over the key.

![mutex](https://media2.dev.to/dynamic/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2F6yp4y67e3yts9shfcr0t.png)

In Go, we use the `sync` package. But because this is a cache, we are going to make a crucial backend optimization. We won't use a standard `sync.Mutex`; we will use a `sync.RWMutex` (Read/Write Mutex).

Why 🧐? Because caches are heavily read-biased. You might read from a cache 1,000 times for every 1 time you write to it. An `RWMutex` allows infinite goroutines to *read* simutaneously (`RLock`), but the moment a goroutine needs to *write*, it locks out everyone else (`Lock`) until the write is finished. 

```go
package main 

import "sync"

type Cache struct {
    data map[string]any
    mu   sync.RWMutex // 👈 Add the Mutex
}

func NewCache() *Cache {
    return &Cache{
        data: make(map[string]any),
    }
}

func (c *Cache) Set(key string, value any) {
    c.mu.Lock()         // 👈 Lock the door for writers
    defer c.mu.Unlock() // 👈 Ensure it unlocks when the function finishes

    c.data[key] = value
}

func (c *Cache) Get(key string) (any, bool) {
    c.mu.RLock()         // 👈 Lock the door for readers (multiple allowed)
    defer c.mu.RUnlock() 

    val, found := c.data[key]
    return val, found
}
```

By adding these simple locks, our cache is now 100% thread-safe. We can blast it with 10,000 concurrent requests, and the Go runtime will coordinate the queue perfectly. 

### 📉 The Trade-off: Safety Costs Speed
This safety comes at a cost called **Lock Contention**. If you have a massive spike in traffic and 5,000 threads all try to `Set()` data at the same time, 4,999 of them are stuck waiting in line. The CPU is spending more time managing the line than actually write data.

This is the fundamental trade-off of concurrent programming: **Shared state requires locks, and locks create bottlenecks**. But at least our server isn't crashing anymore. However, we still have a ticking time bomb. Our cache is thread-safe, but it stores data forever. If we leave this running in production, it will slowly consume every byte of RAM until the server suffers an OOM (Out Of Memory) death 💥. 

To prevent this, we need to build a self-cleaning mechanism. We need to implement TTL.

---

## ⏳ Preventing OOM: Implementing TTL (Time-To-Live)
Right now, our map is just `map[string]any`. To support expirations, we can't just store the raw value anymore. We need to store the value *and* the exact timestamp when that value should die.

Let's update our core data structure by wrapping our value in an `item` struct: 

```go
import "time"

type item struct {
    value   any
    expires int64 // Unix timestamp (nanoseconds) of when this item dies
}

type Cache struct {
    data map[string]item // 👈 Now stores our new item struct
    mu   sync.RWMutex
}
```

Next, we update our `Set` and `Get` methods to respect time.

When we `Set` an item, we calculate its expiration. When we `Get` an item, we check if the current time has passed the expiration time. If it has, we pretend it doesn't exist. 

```go
// We add a TTL duration (e.g., 5 * time.Minute)
func (c *Cache) Set(key string, value any, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.data[key] = item{
        value:   value,
        expires: time.Now().Add(ttl).UnixNano(), // 👈 Calculate exact death time
    }
}

func (c *Cache) Get(key string) (any, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    item, found := c.data[key]
    if !found {
        return nil, false
    }

    if time.Now().UnixNano() > item.expires {
        return nil, false
    }

    return item.value, true
}
```

### 🧹 The Garbage Collector (Active Sweeper)
We have a problem. Our `Get` method correctly hides expired data from user (this is called **Lazy Eviction**). But what if a user `Set` a 100MB JSON payload, it expires, and *nobody ever tries to `Get` it again?*

The data will sit in our map, eating 100MB of RAM forever. Lazy eviction alone will still result in an OOM crash.

To fix this, we need **Active Eviction**. We must spin up a background worker—a separate goroutine—that wakes up every few seconds, scans the entire map, and physically deletes the expired keys from memory. 

Let's write our sweeper: 

```go
// Call this function right after creating a NewCache
func (c *Cache) startSweeper(interval time.Duration) {
    go func() {
        // Create an infinite loop that ticks based on the interval
        ticker := time.NewTicker(interval)
        for range ticker.C {
            c.sweep()
        }
    }()
}

func (c *Cache) sweep() {
    c.mu.Lock() // 👈 Stop the world! No reads or writes allowed.
    defer c.mu.Unlock()

    now := time.Now().UnixNano()
    for key, item := range c.data {
        if now > item.expires {
            delete(c.data, key) // 👈 Physically free the RAM
        }
    }
}
```

We now have a self-cleaning, thread-safe cache. It safely handles concurrent requests and actively cleans up its own garbage to prevent the server from crashing. 

### 🛑 The "Stop The World" Bottleneck
But wait...look closely at our `sweep()` function.

To safely iterate over the map and delete items, we have to call `c.mu.Lock()`. This is a hard write-lock. If we have 1,000,000 keys in our cache, the sweeper has to check every single one of them. While it loops through those million keys, **every single user request trying to read or write to the cache is completely blocked**. Your latency will randomly spike every few seconds because the background worker is "stopping the world" to take out the trash. This is the exact reason why real-world tools like **Redis** don't scan the entire cache at once—they use randomized algorithms to evict keys in small, fast batches. 

However, even with batching, there is one final edge case we haven't solved. What if your server gets hit with a massive spike of traffic, and we run out of RAM *before* the TTL expires 🤔?

We need an absolute hard limit on memory, and we need to know exactly who to kick out when the club gets too full. We need an Eviction Policy.

---

## 🗑️ Eviction Policies: Bounding Memory with LRU
TTL handles the dimension of *time*, but physical RAM is bounded by *volumn*. If your server has 2GB of memory allocated to the cache, and a sudden traffic spike pushes 3GB of data before any TTLs expire, your server will still OOM crash.

We must define a strict `capacity` limit (e.g., maximum 10,000 keys). If a user calls `Set()` and the cache is already at capacity, we must delete an existing key to make room. 

But which key do we delete 🙄? 

If we delete a random key, we might delete highly active data, causing immediate cache misses and slamming our database. The industry standard solution is **LRU (Least Recently Used)**.

The LRU algorithm operates on a simple assumption: **data accessed recently will likely be accessed again soon. Data ignored for a long time is safe to drops.**

![lru](https://media.geeksforgeeks.org/wp-content/uploads/20240909142802/Working-of-LRU-Cache-copy-2.webp)

### ⚙️ The Architecture: Hash Map + Doubly Linked List
Here is the engineering challenge: How do we find the "Least Recently Used" item 🤔?

If we add a `lastAccessed` timestamp to our struct and scan the map to find the oldest time, that is an $O(N)$ operation. If our cache has 10,000 items, scanning it on every single `Set()` operation while holding a write-lock will severely bottleneck the CPU. We need to identify and delete the LRU item in $O(1)$ constant time.

To achieve this, we combine the two data structures: a **Hash Map** and a **Doubly Linked List**.

1. **The Doubly Linked List** maintains the chronological order of usage. The "Head" node is the most recently used. The "Tail" node is the least recently used. 
2. **The Hash Map** provides $O(1)$ lookups. Instead of storing the raw value, the map stores a physical memory pointer to the specific node inside the linked list. 

Here is what the underlying structs actually look like: 

```go
// A node in the doubly linked list
type Node struct {
    key   string
    value any
    prev  *Node
    next  *Node
}

// The LRU Cache
type LRUCache struct {
    capacity int
    data     map[string]*Node // 👈 Map stores pointers to list nodes
    head     *Node            // Most recently used
    tail     *Node            // Least recently used
    mu       sync.RWMutex
}
```

### 🔄 How LRU Executes in O(1) Time
Let's look at the mechanical steps of how this hybrid data structure perfectly optimizes the hardware: 

- **When you `Get(key)`**: The map instantly finds the node pointer in $O(1)$. Because it is a doubly linked list, we can detach the node from its current position and move it to the `head` (marking it as most recently used) by simply updating a few pointers. No memory shifting required. Still $O(1)$.

- **When you `Set(key, value)` at capacity**: We look directly at the `tail` pointer. This is our least recently used item. We detach the tail node from the list in $O(1)$, use its `key` to delete it from the hash map in $O(1)$, and then insert our new item at the `head` in $O(1)$.

By wiring a Map and a Linked List together, we enforce a strict memory boundary without sacrificing a single CPU cycle to scanning.

---

## 🏗️ Putting It All Together: The Final Architecture

We have discussed Mutexes, TTL, and LRU eviction in isolation. Now, let's wire them together into a single, unified system.

Below is the complete architecture. It combines the `RWMutex` for thread safety, the `Hash Map` and `Doubly Linked List` for $O(1)$ capacity management, and the active background sweeper to purge expired data.

```go
package main

import (
	"sync"
	"time"
)

// 1. The Doubly Linked List Node (Now includes TTL expiration)
type Node struct {
	key     string
	value   any
	expires int64
	prev    *Node
	next    *Node
}

// 2. The Core Engine
type LRUCache struct {
	capacity int
	data     map[string]*Node
	head     *Node
	tail     *Node
	mu       sync.RWMutex
}

func NewLRUCache(capacity int, sweepInterval time.Duration) *LRUCache {
	cache := &LRUCache{
		capacity: capacity,
		data:     make(map[string]*Node),
	}
	
	// Start the background garbage collector
	go cache.startSweeper(sweepInterval)
	return cache
}

// 3. Thread-Safe Read with Lazy Eviction
func (c *LRUCache) Get(key string) (any, bool) {
	c.mu.Lock() // Using write-lock because reading an LRU modifies node positions
	defer c.mu.Unlock()

	node, found := c.data[key]
	if !found {
		return nil, false
	}

	// Lazy Eviction check
	if time.Now().UnixNano() > node.expires {
		c.removeNode(node)
		delete(c.data, key)
		return nil, false
	}

	c.moveToHead(node) // Mark as recently used
	return node.value, true
}

// 4. Thread-Safe Write with Capacity Limits
func (c *LRUCache) Set(key string, value any, ttl time.Duration) {
	c.mu.Lock()
	defer c.mu.Unlock()

	// Update existing item
	if node, found := c.data[key]; found {
		node.value = value
		node.expires = time.Now().Add(ttl).UnixNano()
		c.moveToHead(node)
		return
	}

	// Enforce Capacity (Evict LRU)
	if len(c.data) >= c.capacity {
		// The tail is the Least Recently Used
		delete(c.data, c.tail.key)
		c.removeNode(c.tail)
	}

	// Insert new item
	newNode := &Node{
		key:     key,
		value:   value,
		expires: time.Now().Add(ttl).UnixNano(),
	}
	c.data[key] = newNode
	c.addToHead(newNode)
}

// 5. The Active Sweeper
func (c *LRUCache) startSweeper(interval time.Duration) {
	ticker := time.NewTicker(interval)
	for range ticker.C {
		c.mu.Lock()
		now := time.Now().UnixNano()
		
		for key, node := range c.data {
			if now > node.expires {
				c.removeNode(node)
				delete(c.data, key)
			}
		}
		c.mu.Unlock()
	}
}

// Note: Linked list helper methods (addToHead, removeNode, moveToHead) 
// are omitted here to keep the core logic clear, but they execute in O(1) time.
```

By instantiating `NewLRUCache(10000, 5*time.Minute)`, you immediately guarantee that your server will never hold more than 10,000 items in RAM, it will physically garbage-collect expired data every 5 minutes without you triggering it, and it will safely handle thousands of concurrent API requests without corrupting the map.

---
## 🚀 Conclusion: From Theory to Engineering
We started with a theoretical box labeled "Cache." By opening it up and writing the code, we moved from surface-level theory to actual engineering.

You now understand that a cache is not magic. It is simply a Hash Map bounded by physical hardware constraints.

- To prevent race conditions and memory corruption, we apply **Mutex locks**.
- To prevent stale data and delayed OOM crashes, we build **background sweeper goroutines (TTL)**.
- To enforce absolute memory limits without sacrificing $O(1)$ speed, we orchestrate **Hash Maps and Doubly Linked Lists (LRU)**.

If you are asked about caching in a technical interview now, you don't just have to parrot "I'll use Redis." You can explain exactly how the CPU manages concurrent writes, how background eviction threads lock the server, and how memory layout dictates performance.

In the next post, we will tackle the second project on our roadmap: **Build Your Own Message Queue**. We will explore Pub/Sub, why consumers crashing ruins systems, and why explicit Acknowledgements (ACKs) are mandatory.

Until then, stop memorizing. Start engineering. 🛠️

![the-end](https://media0.giphy.com/media/v1.Y2lkPTc5MGI3NjExMWJpeDFtcjc5dGRnbmZtbDhkc241eXRmbGpoNmQ5MG95M3M1MWR6aSZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/xT5LMRgWdqopR1Ar04/giphy.gif)




