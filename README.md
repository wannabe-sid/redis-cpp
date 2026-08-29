# RedisCore

A from-scratch implementation of [Redis](https://github.com/redis/redis) — the popular in-memory key-value database — written in C++. Referred to as **RedisCore** throughout this document.

RedisCore speaks the real [RESP protocol](https://redis.io/docs/latest/develop/reference/protocol-spec/) and is fully compatible with the official [`redis-cli`](https://redis.io/docs/latest/develop/connect/cli/), so you can connect to it exactly as you would a real Redis server.

> **Goal of this project:** not to outperform official Redis, but to deeply understand how Redis — and server applications in C++ generally — actually work under the hood.

---

## Table of Contents

- [Supported Commands](#supported-commands)
- [Performance](#performance)
- [Building RedisCore](#building-rediscore)
- [Running RedisCore](#running-rediscore)
- [Talking to RedisCore](#talking-to-rediscore)
- [Key Design Choices](#key-design-choices)
- [Source Code Layout](#source-code-layout)
- [Improvements / Future Work](#improvements--future-work)
- [About](#about)

---

## Supported Commands

RedisCore supports the same set of operations as the original Redis had at its 2009 launch:

| Command | Description |
|---|---|
| [`SET`](https://redis.io/docs/latest/commands/set/) | Set a string value for a key |
| [`GET`](https://redis.io/docs/latest/commands/get/) | Get the string value of a key |
| [`EXISTS`](https://redis.io/docs/latest/commands/exists/) | Check whether a key exists |
| [`DEL`](https://redis.io/docs/latest/commands/del/) | Delete a key |
| [`INCR`](https://redis.io/docs/latest/commands/incr/) | Increment a key's integer value |
| [`DECR`](https://redis.io/docs/latest/commands/decr/) | Decrement a key's integer value |
| [`LPUSH`](https://redis.io/docs/latest/commands/lpush/) | Push a value onto the left of a list |
| [`RPUSH`](https://redis.io/docs/latest/commands/rpush/) | Push a value onto the right of a list |
| [`LRANGE`](https://redis.io/docs/latest/commands/lrange/) | Get a range of elements from a list |
| [`SAVE`](https://redis.io/docs/latest/commands/save/) | Persist current state to disk |

---

## Performance

Benchmarked with the official `redis-benchmark` tool (`-n 100000`, SET/GET only):

```bash
redis-benchmark -p 2000 -t set,get, -n 100000 -q
SET: 207468.88 requests per second
GET: 213675.22 requests per second
```

For comparison, official Redis on the same setup:

```bash
redis-benchmark -t set,get, -n 100000 -q
SET: 222222.23 requests per second
GET: 222717.16 requests per second
```

| Command | RedisCore | Official Redis | RedisCore vs. Official |
|---|---|---|---|
| `SET` | 207,468.88 req/s | 222,222.23 req/s | ~93.4% |
| `GET` | 213,675.22 req/s | 222,717.16 req/s | ~95.9% |

Despite using a simpler one-thread-per-client model instead of Redis's single-threaded event loop, RedisCore reaches **~93–96% of official Redis's throughput** on basic SET/GET workloads.

---

## Building RedisCore

Let `$REDIS_HOME` be the path where you cloned this repo:

```bash
export REDIS_HOME=/path/to/your/clone
cd $REDIS_HOME
mkdir ${REDIS_HOME}/bin
cd bin
cmake $REDIS_HOME
make
```

This creates the `redis` executable inside `${REDIS_HOME}/bin`.

---

## Running RedisCore

```bash
cd $REDIS_HOME
bin/redis
```

This starts the RedisCore server on port `2000` (configurable in `config.json`).

> **Important:** the executable must be run from `$REDIS_HOME`, since RedisCore expects a `config.json` file in its current working directory.

On first run, you should see:

```
$ bin/redis
State restoral failed! Continuing with empty state...
Server listening on port: 2000
```

*(See [Key Design Choices](#key-design-choices) below for more on state restoral.)*

---

## Talking to RedisCore

Connect using the official `redis-cli`:

```bash
redis-cli -p 2000
```

Example session:

```
127.0.0.1:2000> ping
PONG
127.0.0.1:2000> echo "this is RedisCore"
this is RedisCore
127.0.0.1:2000> set name1 ram
OK
127.0.0.1:2000> get name1
ram
127.0.0.1:2000> rpush statement RedisCore looks interesting!
(integer) 3
127.0.0.1:2000> lrange statement 0 -1
1) "RedisCore"
2) "looks"
3) "interesting!"
127.0.0.1:2000> save
OK
```

---

## Key Design Choices

RedisCore intentionally diverges from official Redis in a few places:

- **Threading model** — RedisCore uses **one thread per client connection**, unlike official Redis's single-threaded event-loop architecture.
  - Chosen mainly for implementation simplicity — async I/O in C++ turned out to be considerably messier to work with.
  - Being multi-threaded, RedisCore implements the necessary **locking** around shared data to prevent race conditions.
- **Persistence** — RedisCore dumps a snapshot of its in-memory state to a `state.json` file in `$REDIS_HOME`, instead of official Redis's binary `.rdb` format.
  - JSON was chosen for human-readability, to make it easy to visually inspect state changes after running `save`.
  - A move to a binary format (e.g. `protobuf`, or `rdb` itself) is a possible future improvement.
  - On startup, RedisCore loads `state.json` from the previous run if present; otherwise it starts with an empty state.
- **Configuration** — via `config.json` in `$REDIS_HOME`, supporting:
  - `port` — the port the server listens on
  - `snapshot_period` — how often (in minutes) the server automatically snapshots its state

---

## Source Code Layout

| File | Description |
|---|---|
| `CMakeLists.txt` | Build configuration for the RedisCore executable. |
| `server.cpp` | Main entry point — starts the server, spawns a new thread per client connection, and launches the periodic snapshot thread. |
| `RESPParser.cpp` / `.h` | Implements deserialization of client requests per the [RESP protocol](https://redis.io/docs/latest/develop/reference/protocol-spec/) (arrays of bulk strings). Uses an 8192-byte read cache per client to avoid excessive `read` syscalls, exposed via `read_new_request()`. |
| `redisstore.cpp` / `.h` | Core data-structure layer. `RedisStore` is a singleton exposing all the storage operations needed by the command layer. |
| `cmds.cpp` / `.h` | Implements each Redis command as a thin, validating wrapper around `RedisStore` methods. |
| `type.cpp` / `.h` | RESP output serialization. Defines a base `RObject` class, subclassed by each RESP data type (string, error, integer, bulk string, array), each implementing its own `serialize()`. |
| `config.cpp` / `.h` | Reads server configuration from `config.json`. |
| `common.cpp` / `.h` | Shared utilities used across the codebase. |
| `config.json` | Runtime configuration (port, snapshot period). |
| `state.json` | Persisted snapshot of RedisCore's in-memory state (created after the first `SAVE`). |

> **Design note:** Some RESP object-definition logic in `type.cpp` arguably overlaps with what `RESPParser` does. However, `RESPParser` requires substantially more validation logic during deserialization, so serialization and deserialization were deliberately kept in separate files rather than unified.

---

## Improvements / Future Work

- Limit the number of concurrent client connections, or explore a thread-pool architecture instead of one-thread-per-client.
- Move state persistence to a binary protocol (e.g. `protobuf` or Redis's own `rdb` format) instead of JSON.
- Support passing configuration via command-line arguments, not just `config.json`.
- Explore the specialized data-structure optimizations used in official Redis.

---

## About

Implementation of Redis in C++, built to understand Redis internals and C++ server design first-hand — not to compete with official Redis on performance.

**Author:** [wannabe-sid](https://github.com/wannabe-sid)
