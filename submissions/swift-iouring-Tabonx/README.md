# Swift / io_uring / SQLite

| | |
|---|---|
| Language | Swift 6.4.1, Swift 6 language mode, no Foundation, stdlib linked statically |
| Framework | none: io_uring through liburing 2.15, with an epoll fallback |
| SQLite | 3.54.0 amalgamation with `SQLITE_THREADSAFE=0`, built with profile-guided optimization |
| JSON / JWT | hand-written parser and writer, SHA-256 and HMAC-SHA256 in Swift |
| Nginx or direct | direct: serves `0.0.0.0:80` itself |

`install.sh` downloads the Swift 6.4.1 toolchain for Ubuntu 24.04 and checks its SHA-256. `build.sh`
downloads SQLite and liburing (both SHA-256 checked) and builds them from source. It trains the SQLite
profile on a synthetic database created from `../../schema.sql`, without seed data. If any PGO step
fails, it builds SQLite without a profile.

## Design

- **Connections live in io_uring, not in the descriptor table.** Each ring accepts with multishot
  accept into its own registered file table (direct descriptors, up to `RLIMIT_NOFILE` slots each).
  With the box's 65,535 limit, eight rings hold up to 524,280 connections, and the process itself keeps
  about a dozen descriptors open. A ring that is 1,024 slots from full cancels its accept, so the kernel
  hands new connections to the next ring. Without that, the first ring takes every connection.
- **One wait, one batch.** Receives are multishot with a provided-buffer ring of 4,096 × 2 KiB buffers
  per ring, so an idle connection owns no buffer. Under load a ring waits for 32 completions or 250 µs.
  When quiet, it wakes on the first completion. Requests are parsed in place in the kernel's buffers and
  the buffers go back to the ring after the batch.
- **Group commit.** All requests in a batch run inside one SQLite transaction, under one mutex shared by
  the rings. Responses, reads included, are submitted only after `COMMIT` succeeds. If it fails, every
  response in the batch becomes a 500 and its connection closes (rule 6).
- **Zero-copy responses.** Each batch writes its responses into an arena, headers in front of each
  body. Sends point straight into it, and the arena is reused only after its last send completes.
- **SQLite.** One connection with `locking_mode=EXCLUSIVE` and WAL, `synchronous=NORMAL`, a 128 MiB page
  cache, no mmap, `wal_autocheckpoint=0`, and a passive checkpoint after a commit once the WAL passes
  4,096 frames.
- **The feed.** It reads the newest 20 ids from `posts_created_at_id_idx`. When they are contiguous,
  one range scan of `likes_post_id_idx` is counted row by row (about 7 % cheaper than `GROUP BY` in
  SQLite) and one rowid range scan returns the rows in feed order. Otherwise the reference query runs.
- **No JOIN for a single post.** SQLite 3.54 re-prepares
  `... JOIN users u ON u.id = p.user_id WHERE p.id = ?1` on every step. A scalar subquery for the
  username keeps the prepared plan.
- **JWT on every authenticated request.** HS256 only, full JSON validation of header and payload,
  constant-time signature compare, `exp` checked, pads hashed once at startup.
- **Fallback.** If the kernel or a sandbox refuses io_uring, the server says so on stderr and runs the
  same request handling on a single-threaded epoll loop.

## Measurements

These are local numbers from Docker on an arm64 Mac, not from the benchmark VPS. The server was pinned
to one CPU with 2 GB, and the load generator ran on the other CPUs of the same VM.

CPU per request with the load.js mix under wrk (mean of two interleaved 20 s runs). The Hummingbird
row comes from an earlier session on the same machine:

| Server | µs/req, 256 conns | µs/req, 4096 conns |
|---|---:|---:|
| swift-hummingbird-denizak (batch 2: 48,000 users) | 69.0 | 72.6 |
| go-epoll-MegaPanchamZ (batch 2: 165,000 users) | 8.1 | 8.7 |
| this entry | 7.6 | 10.3 |

At 4,096 busy connections the Go entry batches better. With think-time users, where every connection
is mostly idle, the two used about the same CPU: 25 % for Go and 22 % for this entry at 60,000 users.

Think-time users, using a C reimplementation of `bench/load.js` (60 s ramp, 120 s hold, the
challenge's pass criteria):

| Users | req/s | p95 | p99 | errors | server CPU | server memory peak |
|---:|---:|---:|---:|---:|---:|---:|
| 100,000 | 10,582 | 9.7 ms | 65.5 ms | 0 | 36 % | 289 MB |
| 200,000 | 21,295 | 16.4 ms | 39.0 ms | 0 | 52 % | 564 MB |

At 260,000 and 300,000 users the run failed. The profile showed the shared kernel of the test VM
dominating: conntrack and established-socket lookups across both client and server sockets. 150,000
idle keep-alive connections cost 230 MB, and the process held 14 descriptors.

The io_uring path was tested on arm64 Linux 6.10. Under x86_64 emulation (Rosetta), io_uring isn't
available, so `test/run.sh` there passed 42/42 on the epoll fallback.

## Layout

```
Sources/CSQLite/include/   module map and helpers for sqlite3.h (fetched by build.sh)
Sources/CShim/             socket, epoll and rlimit setup in C
Sources/CUring/            liburing helpers: SQE preparation, waits with timeout, threads and the mutex
Sources/Turbo/             rings, epoll fallback, SQL, feed, routing, HTTP parsing, JSON, JWT, SHA-256
pgo/train.c                the SQLite training workload
```
