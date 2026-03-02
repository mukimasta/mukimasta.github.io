---
author : ['Mukii']
title: "Distributed System Blog 01 – Exactly-Once RPC"
description: "Implementing Exactly-Once RPC over unreliable networks"
date: 2026-01-20
lastmod: 2026-01-20
type: post
draft: false
translationKey: distributed_systems_blog1
coffee: 3
tags: ['distributed_systems']
categories: ['distributed_systems']
---



# Distributed System Blog 01 – Exactly-Once RPC

**Implementing Exactly-Once RPC over unreliable networks**

## 0  Preface

### 0.1  Distributed Systems

We use distributed systems every day:
- Open WeChat → your phone connects to Tencent's servers
- Scroll TikTok → videos stream from ByteDance's servers to you
- Shop online → order data lives in Alibaba's databases

**Distributed system = multiple independent computing nodes collaborating over a network to achieve a common goal**

**Why do we need distributed systems?**
- Single machine can't hold it all: Facebook has 3 billion users; one machine can't store all that data
- Single machine can't handle the load: Black Friday sees hundreds of thousands of orders per second; one machine can't process that
- Single machines fail: servers break, disks burn out, we need backups

But multiple machines bring new problems: how do they communicate and collaborate over the network?

### 0.2  Remote Procedure Call (RPC)

Think about how you use apps:

```
Tap a friend's avatar → see their profile
Refresh feed → see latest updates
Send a message → recipient gets it
```

Behind the scenes, these operations are like calling a remote function.

Imagine you're writing code:
```python
# Local function call
profile = get_profile(user_id)
```

This function runs in your program and reads data from local memory.

But what if the data lives on another machine?

```python
# Remote function call (RPC)
profile = remote_get_profile(user_id)
```

This function:
1. Sends the request over the network to the server
2. The server executes the function
3. Sends the result back over the network

It looks like we just added "remote," but once the network is involved, everything changes.

### 0.3  A Transfer Story

Imagine this scenario:
```
You open the app, transfer $100 to a friend
Tap "Confirm Transfer"
loading...
loading...
10 seconds pass, still loading
```

What do you do?
- Tap again? What if the first one succeeded? Then you'd transfer $200
- Give up? What if it never went through?
- Wait? Wait for how long?

**This is the core dilemma of distributed systems: the network is unreliable**

### 0.4  How Unreliable Is the Network?

You sent "transfer $100" and got no reply. What happened?
```
Case 1: The request was lost in transit
Client ----X---> Server
(Server never received it)

Case 2: Server executed it, but the reply was lost
Client --------> Server ✔️
Client <---X---- Server
(Money transferred, but you don't know)

Case 3: The network is slow
Client ----😴---> Server 🕒
(Still on the way, looks like it was lost)
```

**Key dilemma: You cannot distinguish these cases, nor can you guarantee the order in which messages arrive.**

### 0.5  Our Goal: Exactly-Once RPC

Regardless of network conditions, we want to guarantee: each operation is executed exactly once (Exactly-Once).

How do we achieve "Exactly-Once"?

**Exactly-Once = At-Least-Once + At-Most-Once**

Breaking it down:
- **Cannot be 0 times** → execute at least 1 time (At-Least-Once)
- **Cannot be 2+ times** → execute at most 1 time (At-Most-Once)

## 1  At-Least-Once RPC

Let's implement At-Least-Once RPC first—ensuring each operation is executed by the Server at least once, and the Client always receives the result.

At-Least-Once is implemented at the Client layer.

### 1.1  Retry Mechanism

First, let's solve the case where the request may be lost—i.e., the Server never received it:
```
  Client    ----X--->    Server
Transfer $100  (packet loss)  (never received)

Result: 0 executions
```

What to do?

An intuitive idea: retry.

We resend the request at intervals until we successfully receive a reply.

```
  Client    ----X--->    Server
Transfer $100  (packet loss)  (never received)

  Client    -------->    Server
Transfer $100   (retry)   (executed successfully)

  Client    <--------    Server
 (got reply)             (reply with result)

Result: 1 execution
```

Seems reasonable, but this creates a problem.

Consider this scenario:

```
T1: Client sends "transfer $100"
T2: (timeout, no reply)
T3: Client retries, sends "transfer $100" again
T4: Server replies "success"
T5: Client sends new command "query balance"
T6: Server replies "success"

Question: Which command is this "success" a reply to?
- The transfer?
- The balance query?
```

If the Client mistakenly thinks "success" is the reply to "query balance" when it's actually the delayed reply from the first "transfer," the system is in chaos.

**Problem essence: replies and requests have lost their correspondence.**

### 1.2  RequestID for Request Identification

How do we establish correspondence? Tag each request.

The tag must satisfy:

- Stay the same on retry (so we know it's the same request)
- Change for new commands (so we can distinguish different requests)

**Natural choice: monotonically increasing request_id**

```
Client maintains a counter:
request_id = 0

Send new command:
  request_id += 1
  send (request_id, command)

On retry:
  send the same (request_id, command)
  (request_id unchanged!)
```

The Server's reply must also include request_id, otherwise the Client cannot match them.

Concrete scenario:

```
T1: Client sends (request_id=1, "transfer $100")
T2: (timeout, about to retry) 
T3: Client sends (request_id=1, "transfer $100")  ← ID unchanged
T4: Server replies (request_id=1, "success")
T5: Client sends (request_id=2, "query balance") ← ID incremented
T6: Server replies (request_id=1, "success")

Client checks: request_id=1 ≠ current request_id=2
→ This is an old reply, ignore it
→ Keep waiting for reply with request_id=2
```

Complete message format:

```
Request:
  request_id: which request (sequence number)
  command:    what to execute

Reply:
  request_id: which request this replies to
  result:     execution result
```

### 1.3  Complete At-Least-Once Logic:

```
Client state:
- current_request_id: ID of current request
- current_command:    command of current request
- timer:              retry timer

Send new command:
  current_request_id += 1
  current_command = command
  send Request(current_request_id, command)
  start timer

On receiving Reply(reply_request_id, result):
  if reply_request_id == current_request_id:
      stop timer
      return result
  else:
      ignore (this is a reply to an old request)

On timer timeout:
  resend Request(current_request_id, current_command)
  restart timer
```


**Thus we achieve At-Least-Once: as long as the Client keeps retrying, the request will eventually reach and execute, and the Client will eventually receive the reply.**


## 2  At-Most-Once RPC

Client retries cause the Server to receive multiple identical requests. At-Most-Once must ensure each request is executed only once.

At-Most-Once is implemented at the Server layer.

### 2.1  Deduplication Mechanism

How does the Server know it's a duplicate request?

Use what we already have: **request_id**

```
Server maintains a record:
seen = {1, 2, 5, 7, ...}

On receiving Request(request_id, command):
  if request_id in seen:
      this is a duplicate request
      ...
  else:
      this is a new request
      ...
```

For multiple Clients:

The Server's record key becomes (client_id, request_id):

In network communication, every message naturally carries the sender's address. We use this address as client_id.

So the complete unique identifier for a request is: (client_id, request_id), which is what the Server layer maintains.
Where:
- **client_id**: provided by the network framework (sender's address)
- **request_id**: maintained by our application layer (monotonically increasing sequence number)

### 2.2  Request → Result Mapping Table

What to do when receiving a duplicate request?

```
T1: Client sends Request(request_id=1, command)
    Server executes, reply is lost

T2: Client retries, sends Request(request_id=1, command)
    Server finds: seen (client_id, 1) before
    Then what?
```

Server cannot:
- Re-execute → violates at-most-once
- Not reply → Client will wait forever

**Server must: return the previous result**

For each request (client_id, request_id), save the corresponding result.

**HashMap: (client_id, request_id) -> Result**

Use a "request → result" mapping table; when an old request arrives, return the cached result.

The Server cannot save history indefinitely.

### 2.3  Garbage Collection Under the Single-Request Assumption

Client single-request assumption: the Client will not send a new command before receiving the reply to the current request.

```
Client behavior:
send req=1 → wait for reply → receive
send req=2 → wait for reply → receive
send req=3 → ...
```

Under this assumption:
```
Client sent request_id=N+1
→ means it has received replies for request_id≤N
→ Server can discard records for request_id≤N
```

So we only need to keep the last one:
```
history = {
  client_A: (last_id, last_result),
  client_B: (last_id, last_result),
}

On receiving new request N:
  history[client_id] = (N, result)
  // automatically overwrites old, keeps only latest
```

**HashMap: client_id -> (request_id, Result)**

### 2.4  Complete At-Most-Once Logic

```
Server state:
- history: map[client_id → (last_request_id, last_result)]

On receiving Request(request_id, command) from client_id:
  
  record = history.get(client_id)
  
  if record does not exist:
      // first time seeing this Client
      result = execute(command)
      history[client_id] = (request_id, result)
      send Reply(request_id, result)
      return
  
  if request_id <= record.last_request_id:
      // duplicate or out-of-order old request
      send Reply(request_id, record.last_result)
      return
  
  // new request (request_id > last_request_id)
  result = execute(command)
  history[client_id] = (request_id, result)  // overwrite old record
  send Reply(request_id, result)
```


## 3  Exactly-Once RPC Complete Architecture

```
Client Node                    Server Node
┌──────────────┐              ┌────────────────┐
│ Retry Logic  │   Command    │ AMOApplication │
│    (ALO)     │  ─────────→  │     (AMO)      │
│              │    Result    │       ↓        │
│              │  ←─────────  │  Application   │
│              │              │   (business)   │
└──────────────┘              └────────────────┘
```

Detailed version:

```
Client Node                              Server Node
┌─────────────────────────┐             ┌──────────────────────────┐
│ Retry Logic (ALO impl)  │             │ AMOApplication (AMO impl)│
│                         │             │                          │
│ • request_id: 1,2,3...  │  Command    │ • Dedup check:           │
│ • Timer: retry timeout  │  +request_id│   req_id <= last_req_id? │
│ • Match reply: ID match?│             │                          │
│                         │ ─────────→  │ • Save history:          │
│                         │             │   map[client_id→         │
│                         │             │       (req_id, result)]  │
│                         │  Result     │                          │
│                         │  +request_id│   ↓ if new request       │
│ Match → stop retry      │ ←─────────  │                          │
│                         │             │ Application (Business)   │
│                         │             │ • Get/Put/Append         │
│                         │             │ • Transfer/Query         │
│                         │             │ • ... (any business)     │
└─────────────────────────┘             └──────────────────────────┘
         ↑                                         ↑
client_id provided by network          Get client_id from message
framework (sender address)             (sender address)
```

**Server internal layering (how it works)**

```
Server internals:
┌─────────────────────┐
│  AMOApplication     │ ← deduplication layer
│      (wrapper)       │   only cares about request_id
│                     │   doesn't care about business logic
│        ↓            │
│  Application        │ ← business layer
│   (concrete logic)   │   only cares about business logic
│  • KVStore          │   doesn't care about deduplication
│  • GameServer       │
│  • ShoppingCart     │
└─────────────────────┘
```

##  4  Extensions

Not all operations fear repetition

### 4.1  Side Effects and Idempotence


**Side Effect = operations that change system state**

```
No side effects (read operations):
- GET(key)          - read-only, doesn't change state
- QUERY_BALANCE()   - read-only, doesn't change state

With side effects (write operations):
- PUT(key, value)   - modifies state
- TRANSFER(amount)  - modifies state
- APPEND(key, val)  - modifies state
```

If an operation has side effects, repeating it may produce wrong results.

**Idempotence = executing multiple times has the same effect as executing once**

```
Idempotent operations:
- GET(key)            - read multiple times, same result
- PUT(key, "value")   - set to fixed value, repeat sets still that value
- DELETE(key)         - delete (deleting already deleted = still deleted)

Non-idempotent operations:
- APPEND(key, value)  - each append makes it longer
- INCREMENT(counter)  - each adds 1, gets bigger
- TRANSFER(amount)    - each deducts money, gets smaller
```

**Why does it matter?**

If operations are idempotent, the system can be simplified:

```
Client: not sure if it succeeded? Just retry
Server: repeating execution is fine
→ no need for complex AMO mechanism, at-least-once is enough
```

This is why systems like DNS, NFS can use simple at-least-once.

### 4.2  Scope of Our Solution

Applies to:
- Non-idempotent operations (need AMO protection)
    - APPEND, INCREMENT, TRANSFER, etc.
- Client single-request mode (garbage collection depends on this assumption)
    - one Client sends only one request at a time
- Stateful services (need to remember history)
    - services that require persistent storage

## 5  Conclusion

Lab 1 looks simple: implement an RPC.

But it reveals the core challenge of distributed systems: **how do we build determinism from uncertainty?**

Our answer is not to eliminate uncertainty (we can't), but rather:

- Acknowledge the network is unreliable
- Decompose the problem (exactly-once = at-least + at-most)
- Layered design (Client retry + Server deduplication)
- Leverage assumptions (single-request → garbage collection mechanism)
- Combine simple mechanisms into complex guarantees

This is not just a Lab solution—it's the fundamental philosophy of distributed system design.

Next: Lab 2 will introduce Primary-Backup, achieving reliable service on unreliable servers.
