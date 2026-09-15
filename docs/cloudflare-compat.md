# Cloudflare compatibility

celld implements the Cloudflare Workers APIs that this page links to. Read the
Cloudflare documentation for the standard behavior of each API.

This page lists only an unavailable feature, a celld-specific limit, or an
observable difference. If an entry has no note, celld intends to match the
linked Cloudflare API.

- **Yes** means that celld implements the API, except for the listed differences.
- **Partial** means that a substantial part of the API is unavailable.
- **Experimental** means that celld can change the API without notice.
- **No** means that celld does not implement the API.

celld must reject an unsupported configuration or API at deployment or first
use. An unsupported feature that does not cause an error is a defect.

## Services

| service | status |
| --- | --- |
| [Workers](#workers) | **Yes** |
| [Durable Objects](#durable-objects) | **Yes** |
| [Durable Object Facets](#durable-object-facets) | **Yes** |
| [Containers](#containers) | **Experimental** |
| [Static assets](#static-assets) | **Yes** |
| [Cron Triggers](#cron-triggers) | **Yes** |
| [Dynamic Workers](#dynamic-workers) | **Yes** |
| [KV](#kv) | **Yes** |
| [Queues](#queues) | **Yes** |
| [D1](#d1) | **Yes** |
| [Workflows](#workflows) | **Yes** |
| [R2](#r2) | **Yes** |
| Workers AI | **No** |
| Vectorize | **No** |
| Hyperdrive | **No** |
| Browser Rendering | **No** |
| Email Workers | **No** |
| Python Workers | **No** |

### [Workers](https://developers.cloudflare.com/workers/runtime-apis/)

- celld does not manage a custom domain or terminate TLS. Terminate TLS in the
  ingress proxy.
- celld does not supply a Workers AI binding. Call an AI provider from the
  application. The process rejects `CELLD_AI_BINDING` and `CELLD_AI_URL`, and a
  deployment rejects an `ai` declaration.

### [Durable Objects](https://developers.cloudflare.com/durable-objects/)

- A Durable Object event keeps pending I/O active after the handler returns, so
  a timer or a subrequest does not require `ctx.waitUntil()`.
- An RPC stub cannot cross an isolate boundary. See [RPC](#rpc).
- An outbound WebSocket does not continue after the object moves to another
  node.
- celld refuses invalid UTF-8 from a SQLite `TEXT` value. Store arbitrary bytes
  in a `BLOB`.
- `SqlStorage.Cursor.toArray()` gives a celld-specific error near the V8 heap
  limit.
- `storage.sync()` waits for the object store or the fleet ensemble to hold all
  earlier committed writes. The operation uses the shorter of the 10-second
  `CELLD_LTX_DURABILITY_TIMEOUT_SECS` default and the 15-second
  `CELLD_OPERATION_DEADLINE_MS` default.
- `storage.sync()` rejects during an open transaction and after an object abort.
  Without an object store, it resolves after the local commit.
- A transaction and `blockConcurrencyWhile()` have a 30-second limit. A timeout
  resets the object and rolls back an open transaction.
- A failed handler still waits for the durability of any value or write that it
  can expose. A celld-generated failure that exposes no object value returns
  immediately.
- Outside an explicit transaction, a SQL write cursor must finish before a
  response, an outbound effect, or `storage.sync()`. An unfinished `RETURNING`
  cursor holds uncommitted writes, so celld rejects that output with an error.
  A read cursor can remain open.

### [Containers](https://developers.cloudflare.com/containers/)

Containers are experimental. The configuration keys, the `ctx.container`
surface, the node-side defaults, and the security boundary can change
without notice, and a change appears on this page.

Configuration and images:

- A `containers` entry accepts `class_name`, `image`, `name`,
  `instance_type`, and `max_instances`. Each other key stops the deployment.
  The class must be a SQLite-backed Durable Object class of the same script.
- `image` is a Dockerfile path in the project or an image reference. `celld
  deploy` and `celld dev` build or pull the image with the `docker` CLI on
  `PATH`, or the CLI that `CELLD_DOCKER` names. A Podman CLI works.
- `celld deploy` builds and pulls for `linux/amd64`, as Wrangler does, so a
  deploy from an ARM machine runs on x86-64 nodes. Set
  `CELLD_CONTAINER_PLATFORM` to build for another platform. `celld dev`
  builds for the local machine.
- `celld deploy` saves each image to the bucket at `deploy/images/<key>.tar`,
  where the key hashes the image layers and config, so a node never contacts
  a registry. A node loads an image into its own engine the first time a
  cell of the class starts, and a redeploy of an unchanged image uploads
  nothing. The `celld-fence` image that the node needs rides with every
  deployment that has a container.
- `instance_type` sets the CPU and memory limits of the Cloudflare instance
  type with that name. An entry without one gets the `dev` type, a sixteenth
  of a CPU and 256 MiB, as on Cloudflare. The disk size of the type is not
  enforced.
- `max_instances` caps the containers a class runs across the fleet.
  Cloudflare enforces the cap centrally; celld has no coordinator, so each
  node publishes its running container count per class in its node record,
  and a node about to start a container sums that count across the fleet
  from the shared node sample and adds its own live count. A start over the
  cap fails, and the object's `monitor()` reports the limit. The sample can
  lag one refresh, so two nodes that start the same class at once can
  briefly exceed the cap and converge on the next refresh.

The node and its engine:

- Each node that serves a container class needs a Docker or Podman daemon.
  celld uses the unix socket that `DOCKER_HOST` names, or the default socket
  of Docker, Docker Desktop, OrbStack, or Podman. A node without an engine
  rejects each `start()` with an error, and it serves every other class.
- `CELLD_CONTAINER_RUNTIME` names the OCI runtime for every container on the
  node, for example `runsc` for gVisor or `kata` for a virtual machine. A
  `containers` entry can name a `runtime` of its own, which overrides the
  node setting for that class; celld adds this key, which Cloudflare's
  config does not have. The daemon must have the runtime, or every `start()`
  of a container that names it fails and `monitor()` rejects with the
  daemon's error, so a class runs only where its isolation exists. Under the
  default runtime a container is a kernel namespace boundary, not a virtual
  machine. For code you did not write, set a runtime that gives each
  container its own kernel, and keep secrets in the Worker, as the Sandbox
  SDK recommends.
- A container that runs under a named runtime gets a public resolver in its
  `/etc/resolv.conf`, `1.1.1.1` by default and the `CELLD_CONTAINER_DNS`
  resolvers when the operator sets them. gVisor cannot reach the container
  engine's built-in resolver, so a container under it would resolve no
  hostname otherwise; the fence permits the public resolver. A container on
  the node's default runtime keeps the engine's resolver.
- A container runs on the node that owns the cell. An idle eviction keeps the
  container running for the `setInactivityTimeout()` window, or for 10
  minutes without one, and the next activation on the same node reconnects to
  it. A move to another node, a node restart, and a reset destroy it, so the
  container disk is ephemeral, as on Cloudflare. A node stop destroys every
  container of the node, and Ctrl-C on `celld dev` does the same while it
  keeps the local state.

Every container gets the same limits and the same boundary:

- All Linux capabilities are dropped, `no-new-privileges` is set, the
  daemon's default seccomp profile applies, and the container can hold 1024
  processes. Two containers on one node cannot open connections to each
  other.
- The node fences its container bridges before the first container starts,
  with nftables rules on the node, outside every container and every
  runtime. A container with `enableInternet: true` reaches the Internet and
  nothing of the node's own: it cannot open a connection to the node, to
  another node, to the private ranges `10/8`, `172.16/12`, `192.168/16`, and
  `100.64/10`, or to the link-local range, so the node's internal listener
  and a cloud metadata service are out of reach. celld installs the rules
  through the daemon with a one-shot privileged container of the
  `celld-fence` image. A node that cannot install the rules starts no
  container, and a deployment from a celld without the fence image starts
  no container on a node with it. IPv6 bridges are not created, so only IPv4
  is fenced.
- `enableInternet: false` attaches the container to an internal bridge with
  no route out. On macOS the node reaches a container only through
  published ports, and an internal network publishes none, so `celld dev` on
  macOS keeps egress on and logs a warning; the fence still applies.

The `ctx.container` surface:

- `running`, `start()`, `monitor()`, `destroy()`, `signal()`,
  `getTcpPort()`, `exec()`, and `setInactivityTimeout()` work. `start()`
  accepts `entrypoint`, `env`, `enableInternet`, and `labels`; it ignores
  `hardTimeout`. `inspect()`, the snapshot methods, and the outbound
  interception methods reject with an error.
- A `monitor()` promise and an `exec()` process settle only inside the event
  that created them. After the handler answers, they do not keep the object
  active, so a drain can move the object. `running` always reports the
  engine's state, so an object that lost its `monitor()` sees the exit at
  its next event. A `start()` that fails is logged by the node as
  `container_start_failed`.
- `getTcpPort(port).fetch()` reaches the container over the node's route to
  it. On Linux the route is the container's bridge address, so every port
  works. On macOS the route is a published loopback port, so the image must
  declare the port with `EXPOSE`, as `wrangler dev` requires. A fetch with
  an `Upgrade: websocket` header gives a 101 response with a `webSocket`,
  as on Cloudflare.
- `getTcpPort(port).connect()` gives a socket with the lifetime of the
  event that opened it. See [TCP sockets](#tcp-sockets).

The published packages:

- `@cloudflare/containers` runs as published. Its `sleepAfter` alarm stops
  the container, and `getContainer()`, `getRandom()`, and `switchPort()`
  work.
- `@cloudflare/sandbox` runs as published on the `cloudflare/sandbox` image:
  `exec()`, files, processes, sessions, and the code interpreter run inside
  the container, over the HTTP and the RPC transport. A move of the object
  to another node stops its container, so the first call after the move can
  fail with the SDK's `OperationInterruptedError`, as after a container
  restart on Cloudflare. The next call starts a fresh container. Preview
  URLs need a wildcard hostname at the operator's load balancer, tunnels
  run cloudflared inside the container, and bucket mounts use the
  credentials the application passes. See
  [`examples/sandbox`](../examples/sandbox).

### [Static assets](https://developers.cloudflare.com/workers/static-assets/)

- celld does not compress an asset response. Use a compressing ingress proxy
  when a client needs gzip or brotli.
- celld has no edge cache. Each node keeps a 512 MiB disk cache, configured by
  `CELLD_ASSET_CACHE_BYTES`, and requires browser revalidation.
- A node checks the deployment pointer after an asset miss at most once in five
  seconds.
- A `_headers` file cannot change `connection`, `content-length`, or
  `transfer-encoding`.
- A deployment can contain 20,000 assets, 25 MiB for each asset, and 1 GiB in
  total. Each `_headers` or `_redirects` file has a 100 KiB limit.
- `celld deploy` does not read `.assetsignore`. It also refuses a symbolic link,
  a special file, a non-UTF-8 name, an unsafe decoded path, and a `_worker.js`
  entry.

### [Cron Triggers](https://developers.cloudflare.com/workers/configuration/cron-triggers/)

- celld rejects a descending range such as `SAT-SUN` and a list that contains
  `*`.
- celld runs one handler for each occurrence across the fleet. After downtime,
  it runs only the most recent missed occurrence.
- celld serializes the handlers for one script. It retries a failure until the
  next occurrence unless the handler calls `noRetry()`.
- A service-binding target cannot run its own Cron Triggers.

### [Dynamic Workers](https://developers.cloudflare.com/dynamic-workers/)

- The process limit is 256 live Dynamic Workers, and each script generation can
  use 255 slots. Every loader and script shares the process limit.
- The process rejects the removed `CELLD_MAX_LOADED_WORKERS` environment
  variable.
- `getEntrypoint()` and `getDurableObjectClass()` support only the `props`
  option. A structured-clone encoded `props` value can be at most 1 MiB.
- A `globalOutbound` Fetcher cannot use `connect()` or a WebSocket.
- celld rejects `limits`, `tails`, and `allowExperimental`. It does not enforce
  the `cpuMs` or `subRequests` values inside `limits`.
- `WorkerCode.env` accepts structured-clone values and Service Binding
  capabilities. The encoded values and the capability props can total 1 MiB.
- A loaded Worker entrypoint cannot transfer to another Worker. Awaitable and
  pipelined properties are also unavailable.

### [Durable Object Facets](https://developers.cloudflare.com/dynamic-workers/usage/durable-object-facets/)

- Facets require Dynamic Workers and a class from a Worker Loader binding. A
  class from `ctx.exports` or a Durable Object binding is unavailable.
- Each facet has a separate SQLite database, which celld replicates with the
  root Durable Object.
- celld rejects an outbound effect from a facet while a root storage transaction
  holds an uncommitted facet image.
- An explicit transaction keeps its uncommitted writes in the facet. A rollback
  discards those writes, and a commit makes them available for root replication.
- The `clone()` method is unavailable.

### [KV](https://developers.cloudflare.com/kv/api/)

- celld has no edge cache. `cacheTtl` has no effect, and `cacheStatus` is
  `null`.
- A value above 1 MiB requires a fleet bucket.
- A namespace has one writer. Use more namespaces to increase write capacity.

### [Queues](https://developers.cloudflare.com/queues/configuration/javascript-apis/)

- A queue has one writer. Use more queues to increase write capacity.
- A Queue owner accepts at most 256 concurrent producer calls. It refuses an
  additional call, which the producer can retry.
- celld retains a message for four days. This period is not configurable.
- Pull consumers, the Queues HTTP API, dashboard controls, manual consumer
  attachment, R2 event notifications, and Queue event subscriptions are
  unavailable.

### [D1](https://developers.cloudflare.com/d1/worker-api/d1-database/)

- A binding result can contain at most 100,000 rows or 32 MiB.
- celld refuses invalid UTF-8 from a SQLite `TEXT` value. Store arbitrary bytes
  in a `BLOB`.

### [Workflows](https://developers.cloudflare.com/workflows/build/workers-api/)

- celld retains a successful or failed instance for 30 days by default. Each
  duration in the `retention` option can be at most 30 days.
- `locationHint` accepts the Cloudflare values, but fleet ownership selects the
  cell location.
- Non-step work cannot remain pending for more than 60 seconds.
- A step result, an event payload, and the workflow parameters each have a
  1 MiB limit.
- Rollback, a sensitive step result, and a `ReadableStream` step result are
  unavailable.

### [R2](https://developers.cloudflare.com/r2/api/workers/workers-api-reference/)

- An R2 binding uses the fleet bucket under `r2/<bucket_name>/`.
- An object `version` equals its content ETag, so identical content produces
  the same version.
- `ssecKey` and `jurisdiction` are unavailable.
- A conditional write cannot use a streamed body larger than 8 MiB.
- `createMultipartUpload()` does not accept a checksum.
- A multipart upload cannot resume on another node or after a restart. celld
  also cannot replace a part that the object store already holds.
- Out-of-order parts can use at most 256 MiB of memory, and completion cannot
  change the stored part order.

## Runtime APIs

| API | status |
| --- | --- |
| [Fetch, Request, Response, and Headers](#fetch-request-response-and-headers) | **Yes** |
| [Bindings](#bindings) | **Yes** |
| [Context](#context) | **Yes** |
| [Handlers](#handlers) | **Yes** |
| [RPC](#rpc) | **Yes** |
| [Streams](#streams) | **Yes** |
| Encoding | **Yes** |
| [WebSockets](#websockets) | **Yes** |
| [Web Crypto](#web-crypto) | **Yes** |
| [Web standards](#web-standards) | **Yes** |
| WebAssembly | **Yes** |
| [Performance and timers](#performance-and-timers) | **Yes** |
| Console | **Yes** |
| [Node.js compatibility](#nodejs-compatibility) | **Partial** |
| [Cache](#cache) | **Partial** |
| HTMLRewriter | **Yes** |
| [TCP sockets](#tcp-sockets) | **Yes** |
| EventSource | **Yes** |
| MessageChannel | **Yes** |
| BroadcastChannel | **No** |

### [Fetch, Request, Response, and Headers](https://developers.cloudflare.com/workers/runtime-apis/fetch/)

- The `cache` request option is unavailable.
- An inbound `Request` has a `cf` object without Cloudflare edge fields. celld
  cannot prove the geolocation, colo, or TLS metadata.
- celld removes `Content-Length` from a Worker response, except for a `HEAD`
  response.
- A remote Durable Object call cannot retry after request-body transmission
  starts because celld keeps no replay copy.
- A remote Durable Object call waits for a rejected owner generation to change
  for at most `CELLD_OPERATION_DEADLINE_MS`.

### [Bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/)

The [services table](#services) lists the available binding types. Each binding
type that the table does not list is unavailable.

### [Context](https://developers.cloudflare.com/workers/runtime-apis/context/)

- `passThroughOnException()` has no effect because celld has no CDN fallback.
- `ctx.facets` is available only inside a Durable Object.

### [Handlers](https://developers.cloudflare.com/workers/runtime-apis/handlers/)

The `tail` and `email` handlers are unavailable.

### [RPC](https://developers.cloudflare.com/workers/runtime-apis/rpc/)

- An RPC stub cannot cross an isolate boundary.
- An `AbortSignal` in a Durable Object RPC call does not cross a node boundary.
- A remote RPC retries only when the failed peer attempt did not start the
  method. Use a stable operation ID for an application retry.

### [Streams](https://developers.cloudflare.com/workers/runtime-apis/streams/)

- celld expires an unclaimed and inactive HTTP stream after 60 seconds. A
  successful stream operation starts a new 60-second period.
- An expired or unknown stream reports an error instead of EOF.

### [WebSockets](https://developers.cloudflare.com/workers/runtime-apis/websockets/)

- An outbound Worker socket closes when its event and `waitUntil` work end. A
  socket returned in the response stays open.
- Each isolate-polled input queue has a 1 MiB budget for non-terminal frames. A
  message larger than 1 MiB uses the complete budget.
- If the isolate stops polling, celld discards unread frames during cleanup. A
  later pull reports an abnormal close.
- A WebSocket transport cannot move to a new cell owner. A client must reconnect
  with the same application operation ID.
- A tunneled connection forwards the owner's Close without an additional Close.
  If the owner connection fails between frames before a Close, the ingress sends
  code 1012. If a frame is incomplete, the ingress closes the transport instead.
- `acceptWebSocket()` throws above 90 percent of the V8 heap limit.

### [Web Crypto](https://developers.cloudflare.com/workers/runtime-apis/web-crypto/)

- HMAC accepts MD5, SHA-1, SHA-224, SHA-256, SHA-384, and SHA-512.
- ECDSA supports only the P-256 curve with SHA-256.
- AES-GCM accepts authentication tags from 96 through 128 bits in 8-bit steps.
- RSA-OAEP accepts SHA-1, SHA-256, SHA-384, and SHA-512. A nonempty label must
  contain valid UTF-8.
- A secret key cannot use `jwk` with `exportKey()` or `wrapKey()`.

### [Web standards](https://developers.cloudflare.com/workers/runtime-apis/web-standards/)

### [Performance and timers](https://developers.cloudflare.com/workers/runtime-apis/performance/)

`performance.timeOrigin` is `0`, and `performance.now()` matches `Date.now()`.
Both clocks advance at an I/O boundary and stay fixed during JavaScript
execution.

### [Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/)

- celld implements `node:assert`, `node:async_hooks`, `node:buffer`,
  `node:diagnostics_channel`, `node:events`, `node:fs`, `node:os`, `node:path`,
  `node:stream`, `node:timers/promises`, and `node:util`.
- `node:diagnostics_channel` does not export messages to a tail Worker.
- `node:crypto` does not implement Diffie-Hellman, streaming signatures,
  ciphers, RSA-PSS, or DSA signatures and key generation.
- `node:zlib` implements only the synchronous gzip and deflate functions.
- `node:fs` provides `access`, `mkdir`, `realpath`, `stat`, `lstat`, and
  `readFile`. It exposes an empty, request-local `/tmp` and a read-only
  `/bundle` that contains the Worker modules.
- The celld bundler supports a synchronous CommonJS `require()` of a Node.js
  built-in module. A raw ESM Worker has no global `require()`.
- An import of another Node.js module succeeds, but its first call throws an
  error.

### [Cache](https://developers.cloudflare.com/workers/runtime-apis/cache/)

celld provides an always-miss cache because it has no shared edge cache.
`put()` validates and consumes a response but stores nothing, `match()` returns
`undefined`, and `delete()` returns `false`.

### [TCP sockets](https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/)

- A socket cannot outlive its event. A Durable Object must reconnect during a
  later event.
- celld verifies a TLS server against its bundled Mozilla root store.
- celld does not block the destination ports that Cloudflare blocks. The fleet
  network controls the egress policy.

### BroadcastChannel

celld defines the class so that a bundle can load, but its constructor throws an
error.

## Compatibility flags

celld honors these compatibility switches:

- `delete_all_deletes_alarm`
- `js_rpc`
- `fetcher_no_get_put_delete`
- `sqlite_vec`
- `websocket_standard_binary_type`
- The static-assets navigation flags

celld accepts each other compatibility flag without effect.
`Cloudflare.compatibilityFlags` reports only the flags that celld honors.

## Wrangler configuration

`celld deploy` accepts `wrangler.jsonc` or `wrangler.json`. It does not accept
`wrangler.toml`.

The `name` value must contain 1 to 63 lowercase ASCII letters, digits, or
internal hyphens. It cannot start or end with a hyphen.

The deployment accepts these top-level keys:

- `$schema`, `name`, `main`, and `no_bundle`
- `compatibility_date` and `compatibility_flags`
- `durable_objects` and `migrations`
- `assets`, `services`, `triggers`, and `vars`
- `d1_databases`, `kv_namespaces`, `queues`, `workflows`, and
  `r2_buckets`
- `worker_loaders` and `containers`

Each other top-level key, including `routes`, stops the deployment.

An asset-only project can omit `main`. The native deploy command refuses an
unsafe asset path, and `.assetsignore` requires Wrangler.

See [Limitations](limitations.md) for the operating-system, networking,
security, pressure, and update boundaries.
