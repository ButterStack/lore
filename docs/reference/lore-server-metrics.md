# Lore Server metrics and traces

## Synopsis

```text
<namespace>.<instrument>    { <attribute>=<value>, ... }
```

Lore Server (`loreserver`) emits metrics, traces, and logs through OpenTelemetry instrumentation. This page catalogs the metrics and spans the server exposes: their names, instrument types, units, and attributes. For the settings that turn export on and tune it, see the [Telemetry settings](lore-server-config.md#telemetry-settings) section of the Lore Server configuration reference.

Export happens over OTLP (gRPC) and only when `[telemetry.exporter]` is configured. Point the exporter at any OpenTelemetry-compatible collector or backend to scrape the metrics and spans below.

## How metric names are built

Most instruments are created through an internal `InstrumentProvider`, which prepends a per-subsystem **namespace** to the **instrument name** with a `.` separator. The exported metric name is `<namespace>.<instrument>`. For example, the S3 backend's namespace is `urc.aws.s3` and its latency instrument is `operation_duration`, so the exported metric is `urc.aws.s3.operation_duration`.

Two namespace prefixes coexist in the current codebase:

- `urc.*` is the original internal name for Lore and prefixes most instruments today.
- `lore.*` prefixes the newer v1 gRPC services.

The helper constructors fix each instrument's type, unit, and buckets:

| Constructor | Instrument | Unit |
| --- | --- | --- |
| `latency_histogram_ms` | `Histogram<f64>` | `milliseconds` |
| `size_histogram` | `Histogram<u64>` | `bytes` |
| `length_histogram` | `Histogram<u64>` | none (count) |
| `counter` | `Counter<u64>` | none |
| `gauge` | `Gauge<u64>` | none |

> [!NOTE]
> Many `urc.*` metrics name low-level internals (the async runtime, the QUIC transport, storage compaction). Treat instrument names as observability surface, not a stability contract: the legacy `urc.*` prefixes and internal instruments can change between releases. The request metrics and the operation-duration metrics in the first sections below are the most stable and the most useful for alerting.

## Shared attributes

Two attribute keys recur across most instruments:

| Attribute | Meaning |
| --- | --- |
| `context` | The operation being measured, for example `get`, `put`, `query`. A single namespace records one instrument across many `context` values. |
| `success` | `true` or `false`, set by the timing combinators from the operation's result. |

Result-aware combinators also set `option=some`/`option=none` for optional results. Every exported metric additionally carries the configured `[telemetry.additional_labels]` and the resource attributes below.

## Resource attributes

Every exported metric, span, and log carries these resource attributes, set once at startup:

| Attribute | Source | Default |
| --- | --- | --- |
| `service.name` | Environment variable `LORE_APP` | `lore` |
| `service.version` | Library version (`LORE_LIBRARY_VERSION`) | build version |
| `deployment.environment.name` | Environment variable `LORE_ENV` | unset |
| `instance` | Environment variable `PLATFORM_INSTANCE_ID` | unset |

Plugin-contributed resource detectors and every `[telemetry.additional_labels]` entry are merged in as additional resource attributes.

## Request metrics

### HTTP server

Recorded for every HTTP request by the HTTP metrics layer. Instrumentation scope `http`, using OpenTelemetry HTTP semantic-convention names.

| Metric | Type | Unit |
| --- | --- | --- |
| `http.server.active_requests` | `UpDownCounter<i64>` | none |
| `http.server.request.duration` | `Histogram<f64>` | `s` |

Attributes:

| Attribute | On | Notes |
| --- | --- | --- |
| `http.request.method` | both | Request method. |
| `http.route` | both | Matched route, leading `/` stripped. Unmatched requests record `unmatched_path`. |
| `user_agent.name` | both | Normalized user-agent. Non-matching agents record `<unknown>`; a missing header records `<none>`. |
| `http.response.status_code` | duration | Response status, recorded on completion. |

### gRPC server

Recorded for every gRPC call by the gRPC metrics layer. Instrumentation scope `grpc`, using OpenTelemetry HTTP and RPC semantic-convention names.

| Metric | Type | Unit |
| --- | --- | --- |
| `http.server.active_requests` | `UpDownCounter<i64>` | none |
| `http.server.request.duration` | `Histogram<f64>` | `s` |
| `rpc.server.duration` | `Histogram<f64>` | `s` |

Attributes: `http.request.method`, `http.route`, `rpc.service`, `rpc.method`, `user_agent.name`, plus `rpc.grpc.status_code` and `http.response.status_code` on completion.

> [!NOTE]
> Calls that return `Unimplemented` are deliberately not recorded, so probes for unsupported methods don't inflate the request metrics.

## Operation-duration metrics

A large set of subsystems share one pattern: a single `<namespace>.operation_duration` latency histogram (`Histogram<f64>`, unit `milliseconds`), split by a `context` attribute naming the operation and a `success` attribute. Use `context` to break latency down per operation and `success` to isolate the error path.

| Namespace | `context` values |
| --- | --- |
| `urc.store.immutable.local` | `obliterate`, `evict`, `compact` |
| `urc.store.immutable.aws` | `exist`, `get`, `put`, `exist_batch`, `obliterate`, `query`, `copy` |
| `urc.store.mutable.aws` | `list`, `load`, `store`, `compare_and_swap` |
| `urc.aws.s3` | `head_bucket`, `list_objects_v2`, `head_object`, `get_object`, `put_object`, `list_versions`, `delete_object` |
| `urc.aws.dynamodb` | `table_exists`, `get_item`, `batch_get_item`, `put_item`, `delete_item`, `transact_write_items`, `query_paginated`, `query_single` |
| `urc.quic.stream_handler` | `handle_message` |
| `urc.auth.jwk_service` | `get_keys` |
| `urc.authnz.urc_auth` | `lookup_user_permissions`, `check_user_permission` |
| `urc.authnz.rebac` | `create_resource`, `delete_resource` |
| `urc.replication.client` | `query`, `get`, `put`, `replication_put` |
| `urc.store.replicated` | `query`, `get`, `put`, `obliterate` (as `immutable.operation_duration`) |

The gRPC service handlers use a parallel per-message latency histogram named `stream.message.handler.duration` (`Histogram<f64>`, `milliseconds`) instead of `operation_duration`:

| Metric | `context` values |
| --- | --- |
| `urc.grpc.storage_service.stream.message.handler.duration` | `get`, `put`, `copy` |
| `urc.grpc.replication_service.stream.message.handler.duration` | `put` |

## Storage metrics

### Local immutable store

Namespace `urc.store.immutable.local`. Alongside its `operation_duration` histogram (see above), the local store reports compaction sizing and a health snapshot.

Compaction (recorded during compaction runs):

| Metric | Type |
| --- | --- |
| `urc.store.immutable.local.compaction_target_size` | `Gauge<u64>` |
| `urc.store.immutable.local.compaction_total_size` | `Gauge<u64>` |
| `urc.store.immutable.local.compaction_group_target_size` | `Gauge<u64>` |
| `urc.store.immutable.local.compaction_group_evicted_size` | `Gauge<u64>` |
| `urc.store.immutable.local.compaction_group_final_total_size` | `Gauge<u64>` |
| `urc.store.immutable.local.compaction_final_size` | `Gauge<u64>` |
| `urc.store.immutable.local.compaction_group_evicted_count` | `Counter<u64>` |

Health snapshot (sampled on the metrics interval; note the hyphen separators):

| Metric | Type | Meaning |
| --- | --- | --- |
| `urc.store.immutable.local.fragment-count` | `Gauge<u64>` | Fragments held by the local store. |
| `urc.store.immutable.local.used-bytes` | `Gauge<u64>` | Bytes used on disk. |
| `urc.store.immutable.local.available` | `Gauge<u64>` | Bytes available. |

### AWS DynamoDB

Namespace `urc.aws.dynamodb`. In addition to `operation_duration`, the DynamoDB backend reports request-shape histograms. All carry the `table_name` attribute; `put_item` also carries `conditional`.

| Metric | Type | Meaning |
| --- | --- | --- |
| `urc.aws.dynamodb.accumulation_operation_duration` | `Histogram<f64>` (ms) | Duration of a paginated accumulation. |
| `urc.aws.dynamodb.batch_operation_size` | `Histogram<u64>` | Items per batch operation. |
| `urc.aws.dynamodb.nested_api_calls.total` | `Histogram<u64>` | Underlying API calls per logical operation. |
| `urc.aws.dynamodb.unprocessed_retries.total` | `Histogram<u64>` | Retries for unprocessed items. |
| `urc.aws.dynamodb.query.accumulation.count` | `Histogram<u64>` | Items returned by an accumulating query. |
| `urc.aws.dynamodb.query.accumulation.scanned_count` | `Histogram<u64>` | Items scanned by an accumulating query. |

### AWS S3, mutable store, immutable store

Namespaces `urc.aws.s3`, `urc.store.mutable.aws`, and `urc.store.immutable.aws` each expose the shared `operation_duration` histogram documented above.

### Composite store

Namespace `urc.store.immutable.composite`. Carries a `replica_type` attribute.

| Metric | Type |
| --- | --- |
| `urc.store.immutable.composite.get` / `.put` / `.query` / `.exist` / `.exist_batch` | `Counter<u64>` |
| `urc.store.immutable.composite.local.caching_total` | `Counter<u64>` |
| `urc.store.immutable.composite.get.inflight.receiver` | `Counter<u64>` |
| `urc.store.immutable.composite.topology.refresh.num_changes` | `Counter<u64>` |
| `urc.store.immutable.composite.topology.refresh.num_peer_errors` | `Counter<u64>` |
| `urc.store.immutable.composite.topology.refresh.num_peers` | `Gauge<u64>` |
| `urc.store.immutable.composite.topology.refresh.iteration.duration` | `Histogram<f64>` (ms) |

### Replication

Namespaces `urc.store.replicated` and `urc.replication.client` expose the shared `operation_duration` histograms above, plus:

| Metric | Type | Meaning |
| --- | --- | --- |
| `urc.store.replicated.client.regenerate.duration` | `Histogram<f64>` (ms) | Client regeneration time. |
| `urc.replication.client.replication.message_dropped` | `Counter<u64>` | Dropped replication messages. |
| `urc.replication.client.replication.timeout` | `Counter<u64>` | Replication timeouts. |
| `urc.replication.client.replication.connect` / `.replication.disconnect` | `Counter<u64>` | Connect and disconnect events. |
| `urc.replication.client.replication.num_attempts` | `Histogram<u64>` | Attempts per replication. |

## QUIC transport metrics

The QUIC transport reports detailed connection health. These are low-level and the most likely to change between releases; they are most useful for transport debugging rather than long-term dashboards.

### Connection monitor

Namespace `urc.quic.client_monitor`, carrying a `context` attribute.

| Metric | Type | Unit |
| --- | --- | --- |
| `urc.quic.client_monitor.connection.path.rtt` | `Histogram<f64>` | `milliseconds` |
| `urc.quic.client_monitor.connection.path.cwnd` | `Histogram<u64>` | `bytes` |

Path gauges (`Gauge<u64>`): `.connection.path.congestion_events`, `.connection.path.lost_packets`, `.connection.path.lost_bytes`, `.connection.path.sent_packets`, `.connection.path.black_holes_detected`, `.connection.path.current_mtu`.

UDP gauges (`Gauge<u64>`): `.connection.udp.tx.datagrams`, `.connection.udp.tx.bytes`, `.connection.udp.tx.ios`, `.connection.udp.rx.datagrams`, `.connection.udp.rx.bytes`, `.connection.udp.rx.ios`.

Frame gauges (`Gauge<u64>`): `.connection.frame.tx.data_blocked`, `.connection.frame.tx.stream_data_blocked`, `.connection.frame.tx.streams_blocked_bidi`, `.connection.frame.tx.stream`, `.connection.frame.tx.reset_stream`, `.connection.frame.rx.data_blocked`, `.connection.frame.rx.stream_data_blocked`, `.connection.frame.rx.max_data`, `.connection.frame.rx.max_stream_data`.

### Quinn connection

Namespace `urc.quinn`.

| Metric | Type |
| --- | --- |
| `urc.quinn.connection.data_blocked` | `Gauge<u64>` |
| `urc.quinn.connection.max_bidi_streams` | `Gauge<u64>` |
| `urc.quinn.connection.stream_data_blocked` | `Gauge<u64>` |
| `urc.quinn.connection.streams_blocked_bidi` | `Gauge<u64>` |
| `urc.quinn.connection.duration_seconds` | `Histogram<u64>` |

### Stream handler and messages

Namespace `urc.quic.stream_handler` reports, besides `operation_duration`:

| Metric | Type |
| --- | --- |
| `urc.quic.stream_handler.stream.peak_pending_chunks` | `Histogram<u64>` |
| `urc.quic.stream_handler.stream.peak_pending_chunk_stall_duration` | `Histogram<u64>` |

Storage message payload and content sizes (`Histogram<u64>`, unit `bytes`):

| Metric |
| --- |
| `urc.quic.message.get.payload_size` |
| `urc.quic.message.get.content_size` |
| `urc.quic.message.put.payload_size` |
| `urc.quic.message.put.content_size` |

## Service counters

### Branch and repository counters

Branch operations increment counters on both the legacy and v1 revision services:

| Metric (legacy / v1 namespace) | Type |
| --- | --- |
| `urc.revision_service.num_branches_created` / `lore.revision.v1.revision_service.num_branches_created` | `Counter<u64>` |
| `urc.revision_service.num_branches_deleted` / `lore.revision.v1.revision_service.num_branches_deleted` | `Counter<u64>` |
| `urc.revision_service.num_branches_pushed` / `lore.revision.v1.revision_service.num_branches_pushed` | `Counter<u64>` |
| `urc.repository_service.num_repositories_created` / `lore.repository.v1.repository_service.num_repositories_created` | `Counter<u64>` |
| `urc.repository_service.num_repositories_deleted` / `lore.repository.v1.repository_service.num_repositories_deleted` | `Counter<u64>` |

The revision services also report revision-list timing, split by `start_type` and `list_strategy` attributes:

| Metric (per legacy / v1 namespace) | Type |
| --- | --- |
| `...revision_list.resolve_start.duration` | `Histogram<f64>` (ms) |
| `...revision_list.resolve_start.relative_age_seconds` | `Histogram<u64>` |
| `...revision_list.walk.duration` | `Histogram<f64>` (ms) |

### Lock service

Namespace `urc.lock_service`, split by `context` (`lock`, `status`, `unlock`, `admin_lock`).

| Metric | Type |
| --- | --- |
| `urc.lock_service.locking.request.resources.length` | `Histogram<u64>` |
| `urc.lock_service.status.request.resources.length` | `Histogram<u64>` |

### Task queue

Namespace `urc.taskqueue`, carrying `name` and `context` attributes.

| Metric | Type |
| --- | --- |
| `urc.taskqueue.submitted_tasks` | `Counter<u64>` |
| `urc.taskqueue.num_rate_limited` | `Counter<u64>` |
| `urc.taskqueue.num_task_result_send_failed` | `Counter<u64>` |
| `urc.taskqueue.num_task_work_send_failed` | `Counter<u64>` |
| `urc.taskqueue.active_tasks` | `Gauge<u64>` |
| `urc.taskqueue.pending_tasks` | `Gauge<u64>` |

### Peer discovery

Namespace `urc.hashicorp.service_peer_discovery`:

| Metric | Type |
| --- | --- |
| `urc.hashicorp.service_peer_discovery.refresh_peers_loop.iteration.duration` | `Histogram<f64>` (ms) |

### TLS certificate expiry

Namespace `urc.certs`. Attributes: `cert_path`, `subject`, `serial`, `expiry_timestamp`.

| Metric | Type | Unit | Description |
| --- | --- | --- | --- |
| `urc.certs.expiry_seconds` | `Gauge<i64>` | `seconds` | Seconds until the certificate expires (negative if expired). |

Alert on this gauge crossing your renewal threshold to catch expiring certificates before connections fail.

## Runtime and process metrics

### Server liveness

| Metric | Namespace | Type | Attributes |
| --- | --- | --- | --- |
| `urc.server.up` | `urc.server` | `Gauge<u64>` | `version`, `settings_hash` |

### Task instrumentation

Namespace `lore.runtime`, with `context_label`, `spawn_file`, and `spawn_line_number` attributes:

| Metric | Type |
| --- | --- |
| `lore.runtime.tasks.spawned.total` | `Counter<u64>` |
| `lore.runtime.tasks.running.total` | `UpDownCounter<i64>` |

### Async runtime bridge

When the server is built with `tokio_unstable`, it bridges Tokio runtime and per-task metrics into OpenTelemetry under the `tokio_runtime` and `quicsrv_tokio_task` scopes (the latter carries a `quinn_server_name` attribute). These are raw runtime instruments (worker counts, poll and park counts, busy ratios, queue depths, poll and delay durations) intended for runtime-level performance debugging. They are the most implementation-specific instruments the server emits; consult the Tokio runtime-metrics documentation for the meaning of individual counters.

## Traces

Lore Server starts a new trace at each inbound request and records spans across request handling, storage, replication, and the QUIC transport.

### Trace-root spans

| Span | Surface | Notable fields |
| --- | --- | --- |
| `lore_http_tracing` | HTTP requests | `http.method`, `http.path`, `correlation_id`, `user_agent`, `repository_id`, `user_id` |
| `lore_tracing` | gRPC calls | `rpc.system`, `rpc.service`, `rpc.method`, `correlation_id`, `user_agent`, `repository_id`, `user_id` |
| `urc` | tower-http layer | `correlation_id`, `method`, `uri` |

Child spans cover gRPC methods (per storage, replication, environment, admin, and notification service), per-fragment storage work, QUIC command dispatch (one span per wire command, such as `Get`, `Put`, `Query`, `Copy`), replication protocol operations, and branch-push work. Standard span field keys include `correlation_id`, `repository_id`, `partition_id`, `user_id`, `branch_id`, `revision`, `connection_id`, `transport`, `protocol`, and `opcode`.

### Sampling

Trace sampling is parent-based with a custom per-operation sampler. The base fraction is `[telemetry.traces].sample_rate` (default `0.05`); spans tagged as low-tier (high-volume) sample at `[telemetry.traces].sample_rate_low_tier` (default `0.001`). See [Metrics and traces](lore-server-config.md#metrics-and-traces) for the settings.

> [!NOTE]
> The spans named `urc-quic` and `ReplicationClient::PutStreamImplementation` are filtered out of OTLP trace export by design, so they don't appear in your tracing backend even when sampled.

## Examples

### Enable OTLP export

Add an exporter and metrics or traces tables to the server configuration, then point the endpoint at your collector. See the [Telemetry settings](lore-server-config.md#telemetry-settings) reference for every field.

```toml
[telemetry.exporter]
endpoint = "http://collector:4317"
queue_size = 2048
timeout = 5000

[telemetry.metrics]
sample_interval_millis = 10000

[telemetry.traces]
sample_rate = 0.05
service_name = "lore-server"
```

### Request latency, split by route

Query the `http.server.request.duration` histogram grouped by `http.route` and `http.response.status_code` to find slow or failing routes.

### Storage error rate

Filter any `operation_duration` metric (for example `urc.aws.s3.operation_duration`) by `success=false`, grouped by `context`, to see which storage operations are failing.

## See also

- [Lore Server configuration reference](lore-server-config.md) for the `[telemetry.*]` settings that turn export on and tune it.
- [OpenTelemetry documentation](https://opentelemetry.io/docs/) for OTLP collectors, exporters, and semantic conventions.
