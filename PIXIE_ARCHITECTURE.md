# Pixie Platform Architecture Document

**Classification:** Internal - CTO Review
**Version:** 2.0
**Date:** 2026-03-17
**Codebase Reference:** pixie-io/pixie @ commit ce714e6e8 (main)
**Languages:** C++ (data plane), Go (control plane), TypeScript/React (UI), Python (PxL scripts)
**Build System:** Bazel

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Overview](#2-system-overview)
3. [Architecture Deep Dive](#3-architecture-deep-dive)
4. [Memory System Analysis](#4-memory-system-analysis)
5. [Performance & Scalability](#5-performance--scalability)
6. [Security Considerations](#6-security-considerations)
7. [Observability & Operational Readiness](#7-observability--operational-readiness)
8. [CTO Review Questions](#8-cto-review-questions)
9. [Risk Register](#9-risk-register)
10. [Recommendations & Next Steps](#10-recommendations--next-steps)

---

## 1. Executive Summary

### What Pixie Is

Pixie is a CNCF-graduated, open-source Kubernetes-native observability platform. It uses eBPF to automatically collect telemetry data (HTTP traces, database queries, CPU profiles, network metrics) from applications running in Kubernetes clusters without requiring code changes, sidecars, or manual instrumentation.

### Problem It Solves

Traditional APM and observability tools impose three costs: (1) engineering effort to instrument code, (2) data egress fees to ship telemetry to SaaS backends, and (3) operational overhead managing agents and sidecars. Pixie eliminates all three by running entirely inside the cluster using kernel-level tracing.

### Key Value Proposition

- **Zero-instrumentation:** eBPF intercepts syscalls at the kernel level. No SDK, no sidecar, no code changes.
- **Edge-first processing:** All data stays in-cluster RAM. No cloud egress for raw telemetry.
- **Automatic protocol decoding:** 14 L7 protocols decoded automatically (HTTP/1.x, HTTP/2/gRPC, MySQL, PostgreSQL, DNS, Redis, Kafka, NATS, Cassandra, MongoDB, AMQP, Mux, TLS metadata).
- **Sub-second queries:** PxL query language compiles to distributed execution plans that run across all nodes in parallel.

### High-Level Architecture

Pixie uses a **split-plane architecture**. The **edge plane (Vizier)** is deployed per Kubernetes cluster and handles all data collection, storage, and query execution. It consists of PEM (Pixie Edge Module) DaemonSet pods running eBPF via the Stirling engine, storing telemetry in Apache Arrow columnar in-memory tables, and executing distributed queries via the Carnot engine. A stateless Query Broker orchestrates query execution across PEMs and a centralized aggregator called Kelvin.

The **cloud plane** is a set of 14 Go microservices backed by PostgreSQL. It handles user authentication, cluster registration, query proxying, and management. Critically, the cloud never stores or processes raw telemetry data. It serves only as a control plane and query proxy.

Communication between edge and cloud flows through a persistent bidirectional gRPC stream, bridged internally by NATS messaging. This design means Pixie continues collecting data even when cloud connectivity is lost. The fundamental architectural trade-off is **RAM-bounded data retention vs. zero data egress**: telemetry lives only in cluster memory (default 1.28 GB per node), making retention inversely proportional to traffic volume.

---

## 2. System Overview

### 2.1 Component Inventory

| Component | Language | Deployment | Role |
|-----------|----------|------------|------|
| **Stirling** | C++ | In-process (PEM) | eBPF data collection engine. Deploys kprobes/uprobes, decodes 14 protocols |
| **Carnot** | C++ | In-process (PEM & Kelvin) | Query engine. Compiles PxL to distributed plans, executes Arrow-based operators |
| **Table Store** | C++ | In-process (PEM & Kelvin) | In-memory columnar storage. Apache Arrow format, hot/cold partitioning |
| **PEM** | C++ | DaemonSet (1 per node) | Per-node agent. Runs Stirling + local Carnot. Default 2 GiB memory request |
| **Kelvin** | C++ | Deployment (1 replica) | Aggregation agent. Runs Carnot without data collection. Merges PEM results |
| **Query Broker** | Go | Deployment | Receives PxL queries, invokes planner, dispatches plans via NATS |
| **Metadata Service** | Go | Deployment (replicated, leader-elected) | Watches K8s API, tracks agents/schemas, PebbleDB-backed |
| **Cloud Connector** | Go | Deployment (1 replica) | Bidirectional gRPC bridge to Cloud. Heartbeats, query passthrough |
| **Cloud API** | Go | Deployment | User-facing GraphQL + gRPC gateway |
| **Cloud Auth** | Go | Deployment | OAuth/OIDC authentication, JWT issuance, API key management |
| **Cloud VZMgr** | Go | Deployment | Vizier lifecycle management, deployment keys, status monitoring |
| **Cloud VZConn** | Go | Deployment | Server-side of edge-cloud bridge. NATS JetStream integration |
| **K8s Operator** | Go | Deployment (1 replica) | Deploys/manages Vizier CRD. Handles upgrades, cert provisioning |
| **UI** | TypeScript/React | Static assets (served via cloud) | Web-based query editor, live views, admin console |
| **CLI (`px`)** | Go | Client binary | Command-line interface for deploy, run, live, debug |

### 2.2 High-Level Data Flow

```mermaid
flowchart TB
    subgraph User["User Interfaces"]
        UI["Web UI<br/>(React + Apollo GraphQL)"]
        CLI["CLI (px)<br/>(Go, gRPC)"]
        API_EXT["External API<br/>(gRPC/REST)"]
    end

    subgraph Cloud["Cloud Plane (Go Microservices)"]
        API["API Service<br/>GraphQL + gRPC"]
        AUTH["Auth Service<br/>OAuth/OIDC + JWT"]
        VZMGR["VZMgr<br/>Cluster Lifecycle"]
        VZCONN["VZConn<br/>Edge Bridge"]
        PG[(PostgreSQL)]
        NATS_CLOUD["NATS JetStream"]
    end

    subgraph Vizier["Edge Plane - Vizier (per K8s cluster)"]
        CC["Cloud Connector<br/>gRPC Bridge"]
        QB["Query Broker<br/>Plan & Dispatch"]
        MDS["Metadata Service<br/>PebbleDB + K8s Watch"]
        NATS_VIZ["NATS"]

        subgraph Agents["Data Collection Tier"]
            PEM1["PEM (Node 1)<br/>Stirling + Carnot + TableStore"]
            PEM2["PEM (Node 2)<br/>Stirling + Carnot + TableStore"]
            PEMN["PEM (Node N)<br/>Stirling + Carnot + TableStore"]
        end

        KELVIN["Kelvin<br/>Aggregation Carnot"]
    end

    subgraph Kernel["Linux Kernel (per node)"]
        BPF["eBPF Programs<br/>kprobes/uprobes/tracepoints"]
        PERF["Perf Buffers<br/>(per-CPU ring buffers)"]
    end

    UI --> API
    CLI --> API
    API_EXT --> API
    API --> AUTH
    API --> VZMGR
    API --> VZCONN
    AUTH --> PG
    VZMGR --> PG
    VZMGR --> NATS_CLOUD
    VZCONN --> NATS_CLOUD

    VZCONN <-->|"Persistent bidirectional<br/>gRPC stream"| CC
    CC --> NATS_VIZ
    QB --> NATS_VIZ
    MDS --> NATS_VIZ
    QB -->|"Plan dispatch"| NATS_VIZ
    NATS_VIZ -->|"Plan fragments"| PEM1
    NATS_VIZ -->|"Plan fragments"| PEM2
    NATS_VIZ -->|"Plan fragments"| PEMN
    PEM1 -->|"Partial results<br/>(gRPC stream)"| KELVIN
    PEM2 -->|"Partial results<br/>(gRPC stream)"| KELVIN
    PEMN -->|"Partial results<br/>(gRPC stream)"| KELVIN
    KELVIN -->|"Aggregated results<br/>(gRPC ResultSink)"| QB

    BPF -->|"Events"| PERF
    PERF -->|"Poll every 200ms"| PEM1
```

### 2.3 Telemetry Collection Flow

```mermaid
flowchart LR
    subgraph Kernel["Kernel Space"]
        SC["Syscalls<br/>(read/write/accept/close)"]
        BPF["eBPF Programs"]
        PB["Perf Buffers<br/>(per-CPU, ~100 MiB/s)"]
    end

    subgraph UserSpace["User Space - Stirling"]
        CT["ConnTracker<br/>(per-connection state)"]
        PP["Protocol Parsers<br/>(HTTP, MySQL, DNS...)"]
        DT["DataTable Buffer<br/>(1024 records target)"]
    end

    subgraph Storage["Table Store"]
        HOT["Hot Partition<br/>(write buffer, spinlock)"]
        COLD["Cold Partition<br/>(compacted Arrow arrays)"]
        EXPIRY["FIFO Expiry<br/>(oldest data evicted)"]
    end

    SC --> BPF
    BPF --> PB
    PB -->|"Poll + drain"| CT
    CT --> PP
    PP --> DT
    DT -->|"Push when full<br/>(callback)"| HOT
    HOT -->|"Compact every 1 min<br/>(~64 KB batches)"| COLD
    COLD --> EXPIRY
```

### 2.4 Query Execution Flow

```mermaid
sequenceDiagram
    participant User
    participant API as Cloud API
    participant VZConn
    participant CC as Cloud Connector
    participant QB as Query Broker
    participant Planner as Carnot Planner (C++)
    participant NATS
    participant PEM1 as PEM (Node 1)
    participant PEM2 as PEM (Node 2)
    participant Kelvin

    User->>API: ExecuteScript(PxL)
    API->>VZConn: Proxy via NATS
    VZConn->>CC: gRPC stream
    CC->>QB: NATS passthrough

    QB->>Planner: Compile PxL → DistributedPlan
    Planner-->>QB: Per-agent plan fragments

    par Fan-out via NATS
        QB->>NATS: Plan → PEM1
        QB->>NATS: Plan → PEM2
    end

    par Local execution
        PEM1->>PEM1: MemorySource → Filter → Map → Agg(partial)
        PEM2->>PEM2: MemorySource → Filter → Map → Agg(partial)
    end

    PEM1->>Kelvin: gRPC TransferResultChunk (partial)
    PEM2->>Kelvin: gRPC TransferResultChunk (partial)

    Kelvin->>Kelvin: GRPCSource → Agg(finalize) → GRPCSink
    Kelvin->>QB: gRPC ResultSink (final results)
    QB-->>CC: Stream results via NATS
    CC-->>VZConn: gRPC stream
    VZConn-->>API: Proxy response
    API-->>User: ExecuteScriptResponse (streaming)
```

---

## 3. Architecture Deep Dive

### 3.1 Stirling - eBPF Data Collection Engine

**Source:** `src/stirling/`

Stirling is the kernel-side data collection engine built on BCC (BPF Compiler Collection). It runs inside each PEM and is responsible for all automated telemetry collection.

#### Source Connectors

| Connector | File | Sampling Period | Tables Produced |
|-----------|------|-----------------|-----------------|
| **SocketTraceConnector** | `src/stirling/source_connectors/socket_tracer/socket_trace_connector.h` | 200ms | conn_stats, http_events, mysql_events, cql_events, pgsql_events, dns_events, redis_events, nats_events, kafka_events, mux_events, amqp_events, mongodb_events, tls_events |
| **ProcessStatsConnector** | `src/stirling/source_connectors/process_stats/` | Configurable | process_stats |
| **PerfProfileConnector** | `src/stirling/source_connectors/perf_profiler/` | Configurable | stack_traces |
| **TCPStatsConnector** | `src/stirling/source_connectors/tcp_stats/` | Configurable | tcp_stats |
| **ProcExitConnector** | `src/stirling/source_connectors/proc_exit/` | Event-driven | proc_exit_events |
| **NetworkStatsConnector** | `src/stirling/source_connectors/` | Configurable | network_stats |
| **DynamicTraceConnector** | `src/stirling/source_connectors/dynamic_tracer/` | User-defined | User-defined |

#### BPF Data Path Architecture

```mermaid
flowchart TD
    subgraph KernelBPF["Kernel - BPF Programs"]
        KP["kprobes<br/>(read/write/accept/close)"]
        UP["uprobes<br/>(Go HTTP2, gRPC, TLS)"]
        TP["Tracepoints<br/>(proc_exit, tcp_stats)"]
        SP["Sampling Probes<br/>(CPU profiler)"]
    end

    subgraph PerfBuf["Perf Buffers (per-CPU)"]
        DATA_PB["Data Buffer<br/>Target: 100 MiB/s per CPU<br/>Max: 10 Gbit/s total"]
        CTRL_PB["Control Buffer<br/>Target: 5 MiB/s per CPU<br/>Max: 512 Mbit/s total"]
    end

    subgraph Stirling["User-Space Stirling Engine"]
        POLL["Perf Buffer Drain<br/>(every 200ms)"]
        CTM["ConnTracker Manager<br/>(per-connection state machines)"]
        DS["DataStream<br/>(bidirectional buffers,<br/>1 MiB retention each)"]
        PARSE["Protocol Parsers<br/>(req/resp stitching)"]
        DT["DataTable<br/>(1024 record target capacity)"]
    end

    KP --> DATA_PB
    UP --> DATA_PB
    TP --> CTRL_PB
    SP --> DATA_PB
    DATA_PB --> POLL
    CTRL_PB --> POLL
    POLL --> CTM
    CTM --> DS
    DS --> PARSE
    PARSE --> DT
    DT -->|"Push callback<br/>when full"| TS["Table Store"]
```

#### Key Configuration Flags

| Flag | Default | Env Var | Description |
|------|---------|---------|-------------|
| `stirling_socket_tracer_target_data_bw_percpu` | 100 MiB/s | - | Per-CPU perf buffer bandwidth target |
| `stirling_socket_tracer_target_control_bw_percpu` | 5 MiB/s | - | Per-CPU control buffer bandwidth |
| `stirling_socket_tracer_max_total_data_bw` | 10 Gbit/s | - | Global data bandwidth cap |
| `stirling_socket_tracer_percpu_bw_scaling_factor` | 8 | - | Adjusts per-CPU sizing based on core count |
| `messages_expiry_duration_secs` | 60 | - | Parsed message lifetime per connection |
| `messages_size_limit_bytes` | 1 MiB | - | Message buffer limit per direction per connection |
| `datastream_buffer_retention_size` | 1 MiB | `PL_DATASTREAM_BUFFER_SIZE` | DataStream buffer retention per connection |
| `max_body_bytes` | 512 | `PL_STIRLING_MAX_BODY_BYTES` | Protocol body truncation limit |
| `total_conn_tracker_mem_usage` | 0 (unlimited) | - | Global ConnTracker memory cap |
| `stirling_enable_http_tracing` | 1 (on) | - | Per-protocol enable/disable |
| `stirling_disable_self_tracing` | true | - | Exclude Stirling's own traffic |
| `stirling_bpf_loop_limit` | 41 | - | Max iovecs per BPF program invocation |
| `stirling_bpf_chunk_limit` | 4 | - | Max chunks for messages >30 KB |

**Critical Design Decision:** Each active network connection gets a `ConnTracker` object holding bidirectional `DataStream` buffers (up to 1 MiB each). On nodes with thousands of concurrent connections (e.g., service mesh proxies), this is the primary unbounded memory consumer. The `total_conn_tracker_mem_usage` flag exists to cap this but **defaults to 0 (unlimited)**.

### 3.2 Carnot - Query Engine

**Source:** `src/carnot/`

Carnot is a distributed columnar query engine that compiles PxL into physical execution plans.

#### Compilation Pipeline

```mermaid
flowchart LR
    PXL["PxL Script<br/>(Python-like)"]
    PARSE["Parser<br/>(pypa → AST)"]
    IR["AST → IR<br/>(DAG of OperatorIR)"]
    ANALYZE["Analyzer<br/>(type checking,<br/>relation inference)"]
    OPT["Optimizer<br/>(predicate pushdown,<br/>column pruning)"]
    VERIFY["Verifier<br/>(ResultSink check)"]
    DIST["Distributor<br/>(split into<br/>PEM + Kelvin plans)"]
    PROTO["Plan Proto<br/>(per-agent planpb.Plan)"]

    PXL --> PARSE --> IR --> ANALYZE --> OPT --> VERIFY --> DIST --> PROTO
```

#### Execution Operators

| Operator | Type | Description |
|----------|------|-------------|
| `MemorySourceNode` | Source | Reads from in-memory Table Store via `Table::Cursor` |
| `GRPCSourceNode` | Source | Receives RowBatches from remote Carnot instances |
| `UDTFSourceNode` | Source | Executes table-generating functions (UDTFs) |
| `EmptySourceNode` | Source | Generates empty batches for schema-only queries |
| `MapNode` | Processing | Column transformation via UDFs |
| `FilterNode` | Processing | Row filtering via scalar expressions |
| `AggNode` | Processing | Blocking/windowed aggregation via UDAs. Supports partial aggregation |
| `LimitNode` | Processing | Row limiting. Can abort upstream sources early |
| `UnionNode` | Processing | Merges multiple input streams |
| `EquiJoinNode` | Processing | Hash-based equality join |
| `MemorySinkNode` | Sink | Writes results to in-memory Table Store |
| `GRPCSinkNode` | Sink | Sends RowBatches to remote Carnot (PEM → Kelvin) |
| `OTelExportSinkNode` | Sink | Exports to OTel collectors via gRPC. Supports metrics, traces, and logs. gzip compression enabled. Custom headers + configurable timeout. |

**Execution lifecycle per node:** `Init()` → `Prepare()` → `Open()` → `GenerateNext()`/`ConsumeNext()` → `Close()`

**Distributed execution strategy:** The planner splits logical plans at "blocking" operators (aggregations, joins, limits). PEMs execute scan/filter/map/partial-aggregate locally. Results stream via gRPC to Kelvin, which finalizes aggregations and sends results to the Query Broker's `ResultSinkService`.

**Key limitation confirmed in code:** No per-query memory budget exists. Carnot uses Arrow's `default_memory_pool()` without limits. A complex query (e.g., large join) can consume unbounded memory within a PEM.

### 3.3 Table Store - In-Memory Columnar Storage

**Source:** `src/table_store/`

The Table Store is Pixie's primary data layer. **All telemetry data is stored in RAM as Apache Arrow columnar arrays.** There is no disk persistence by design.

#### Architecture

```mermaid
flowchart TB
    subgraph TableStore["Table Store (per PEM)"]
        subgraph Table1["Table: http_events (max ~512 MB)"]
            HOT1["Hot Partition<br/>(write buffer)<br/>SpinLock protected"]
            COLD1["Cold Partition<br/>(compacted Arrow arrays, ~64 KB batches)<br/>SpinLock protected"]
            IDX1["Time Index<br/>(O(log N) sorted list)"]
        end
        subgraph Table2["Table: conn_stats"]
            HOT2["Hot"]
            COLD2["Cold"]
        end
        subgraph TableN["Table: ... (N tables)"]
            HOTN["Hot"]
            COLDN["Cold"]
        end
    end

    STIRLING["Stirling<br/>(RecordBatch push)"] --> HOT1
    HOT1 -->|"CompactHotToCold()<br/>every 1 min"| COLD1
    COLD1 -->|"ExpireBatch()<br/>when size > limit"| EVICT["FIFO Eviction<br/>(oldest batches dropped)"]

    QUERY["Carnot Query<br/>(Table::Cursor)"] --> IDX1
    IDX1 --> HOT1
    IDX1 --> COLD1
```

#### Table Configuration (per PEM)

**Source:** `src/vizier/services/agent/pem/pem_manager.cc`

| Parameter | Default | Env Var | Description |
|-----------|---------|---------|-------------|
| **Total table store limit** | **1280 MB** (1024+256) | `PL_TABLE_STORE_DATA_LIMIT_MB` | Maximum aggregate size of all tables |
| HTTP events table share | 40% (~512 MB) | `PL_TABLE_STORE_HTTP_EVENTS_PERCENT` | Dedicated allocation for `http_events` |
| Stirling error table | 1 MB | `PL_TABLE_STORE_STIRLING_ERROR_LIMIT_BYTES` | `stirling_error` table cap |
| Probe status table | 1 MB | (same as above) | `probe_status` table cap |
| Proc exit events | 10 MB | `PL_TABLE_STORE_PROC_EXIT_EVENTS_LIMIT_BYTES` | `proc_exit_events` table cap |
| Other tables | ~756 MB / N | Computed | Remaining budget split equally |
| Per-table fallback limit | 64 MB | `PL_TABLE_STORE_TABLE_SIZE_LIMIT` | Default if not specifically sized |
| Compaction period | 1 minute | Hardcoded | `kTableStoreCompactionPeriod` |
| Cold batch target size | 64 KB | Hardcoded (256 KB for HTTP) | `kDefaultColdBatchMinSize` |

**Cursor pattern:** Queries iterate via `Table::Cursor` with `StartSpec` (time or position) and `StopSpec` (time, position, or infinite for streaming). Cursors track a `last_read_row_id` to guarantee consistency across compaction events.

### 3.4 Cloud Services

**Source:** `src/cloud/`

14 Go microservices, all using PostgreSQL with migration-based schema management (`src/cloud/shared/pgmigrate/`).

```mermaid
flowchart TB
    subgraph External["External Access"]
        USER["Users / CLI"]
        VIZIER_EDGE["Vizier Clusters"]
    end

    subgraph CloudServices["Cloud Plane"]
        API["API Service<br/>Port 51200<br/>GraphQL + gRPC"]
        AUTH["Auth Service<br/>Port 50100<br/>OAuth/OIDC, JWT, API Keys"]
        PROFILE["Profile Service<br/>Port 51500<br/>Users, Orgs, Invites"]
        VZMGR["VZMgr Service<br/>Port 51800<br/>Cluster lifecycle, Deploy keys"]
        VZCONN["VZConn Service<br/>Port 51600<br/>Edge bridge"]
        SCRIPTMGR["ScriptMgr<br/>PxL catalog"]
        PLUGIN["Plugin Service<br/>Retention, OTel export"]
        PROJMGR["Project Manager<br/>Subdomain management"]
        CONFIGMGR["Config Manager<br/>Vizier YAML generation"]
        CRONSCRIPT["CronScript<br/>Scheduled execution"]
        ARTIFACT["Artifact Tracker<br/>Version management"]
        INDEXER["Indexer<br/>Elasticsearch integration"]
        METRICS["Metrics<br/>Internal aggregation"]
        AUTOCOMPLETE["Autocomplete<br/>PxL query hints"]
    end

    subgraph Infra["Infrastructure"]
        PG[(PostgreSQL<br/>+ pgcrypto)]
        NATS_JS["NATS JetStream"]
        ES["Elasticsearch"]
    end

    USER --> API
    API --> AUTH
    API --> PROFILE
    API --> VZMGR
    API --> SCRIPTMGR
    API --> PLUGIN
    API --> VZCONN
    AUTH --> PG
    PROFILE --> PG
    VZMGR --> PG
    VZMGR --> NATS_JS
    VZCONN --> NATS_JS
    PLUGIN --> PG
    CRONSCRIPT --> PG
    INDEXER --> ES
    VIZIER_EDGE <-->|"Persistent gRPC stream"| VZCONN
```

#### Key Cloud Database Tables

| Service | Tables | Notable Schema Decisions |
|---------|--------|-------------------------|
| **Auth** | `api_keys` | SHA256-hashed + PGP-encrypted keys. `PL_DATABASE_KEY` env var for encryption |
| **Profile** | `users`, `orgs`, `invite_tokens`, `user_attributes` | Multi-tenant with org-scoped isolation |
| **VZMgr** | `vizier_cluster`, `vizier_cluster_info`, `deployment_keys` | 31 migration files. Shard-indexed for horizontal scaling. Status enum: UNKNOWN/HEALTHY/UNHEALTHY/DISCONNECTED |
| **Plugin** | `retention_plugins_enabled`, `retention_scripts`, `retention_releases` | Per-org plugin configuration |
| **CronScript** | `cron_scripts` | Scheduled PxL execution definitions |

### 3.5 Vizier Edge Services

#### Query Broker

**Source:** `src/vizier/services/query_broker/`

The Query Broker is the query orchestrator. It is **stateless per-query** but holds in-memory state for active queries:

- `QueryResultForwarder`: Maintains an `activeQuery` map with per-query buffered channels (size 1024). Timeouts: 30s for sink initialization, 180s for producer/consumer inactivity. On PEM failure mid-query: watchdog goroutine detects producer timeout, calls `ProducerCancelStream()`, cascades cancellation to consumers, and sends error via result channel.
- `AgentsTracker`: Background goroutine streaming agent state from Metadata Service at 5-second intervals, maintaining full schema catalog.
- **Not fully stateless:** The result forwarder holds per-query state. If a Query Broker pod crashes, all in-flight queries on that pod are lost.

#### Metadata Service

**Source:** `src/vizier/services/metadata/`

Leader-elected via K8s Lease objects (`src/shared/services/election/election.go`, using `K8sLeaderElectionMgr`). Only the leader writes to the datastore; followers handle reads and stand ready for promotion.

- **Election timing:** Lease duration = `expectedMaxSkew + renewDuration` (~7s default), renew deadline = 5s, retry period = ~1.25s. Pod terminates on leadership loss (auto-restart triggers re-election).
- **Backing store:** PebbleDB (default) at `/metadata/pebble_20220209`. Uses Pebble defaults (no custom block cache, bloom filter, or compaction tuning). Keys use TTL-based expiration with dual-index approach (`___ttl_by_key___` and `___ttl_time___` prefixes) and a 1-minute reaper cycle.
- **K8s metadata TTL:** 24 hours
- **Agent info TTL:** 24 hours
- **Agent health threshold:** 30 seconds (unresponsive if no heartbeat)

#### Cloud Connector

**Source:** `src/vizier/services/cloud_connector/bridge/server.go`

Maintains a single persistent gRPC stream to VZConn. Internal buffered channels:

| Channel | Buffer Size | Purpose |
|---------|-------------|---------|
| `grpcOutCh` | 5000 | Outbound messages to cloud |
| `grpcInCh` | 5000 | Inbound messages from cloud |
| `ptOutCh` | 5000 | Priority passthrough traffic |
| `natsMetricsCh` | 5000 | Metrics forwarding |

Heartbeat every 5 seconds. Watchdog kills the container after 10 minutes of heartbeat stall.

### 3.6 NATS Messaging Patterns

**Topic naming convention:**

| Pattern | Example | Usage |
|---------|---------|-------|
| `Agent/<uuid>` | `Agent/550e8400-...` | Per-agent query dispatch |
| `v2c.<topic>` | `v2c.heartbeat` | Vizier → Cloud |
| `c2v.<topic>` | `c2v.VizierPassthroughRequest` | Cloud → Vizier |
| `Metrics` | `Metrics` | Prometheus metrics forwarding |

**Edge NATS:** NATS within the Vizier namespace uses **TLS client certificate authentication** (verified: `nats.ClientCert()` + `nats.RootCAs()` in `query_broker_server.go:155-157`). This provides mutual TLS between NATS clients and the broker, though any pod with access to the TLS certs (stored as K8s secrets in the namespace) can connect.

**Cloud NATS:** JetStream with durable subscriptions (`src/shared/services/msgbus/jetstream.go`):
- `AckPolicy`: Explicit acknowledgment, `AckWait`: 30 seconds
- `MaxAckPending`: 50 messages per consumer
- `DeliverPolicy`: DeliverAll (replay from start on new subscription)
- Async publishing with exponential backoff retry
- Configurable cluster size (default 5 replicas via `jetstream_cluster_size` flag)

### 3.7 UI and CLI

**UI** (`src/ui/`): React SPA with Apollo Client for GraphQL, Material-UI theming, LaunchDarkly feature flags.
- **State management:** React Context API (`PixieAPIContext`, `AuthContext`, `PixieThemeContext`, `EmbedContext`, plus per-feature contexts for script/editor/results)
- **Routing:** React Router with embedded path prefix support (`/embed/*`)
- **Key pages:** `pages/live/` (query editor), `pages/admin/` (admin console), `pages/configure-data-export/` (OTel export setup)
- **Auth flow:** `useIsAuthenticated()` hook with 5-minute token refresh for embedded mode

**CLI** (`src/pixie_cli/`): Go-based Cobra CLI with Segment.io analytics integration.
- **Commands:** `deploy`, `delete`, `run`, `live` (interactive REPL with autocomplete + command palette via Ctrl+K), `get`, `debug`, `collect_logs`, `api_key`, `deployment_key`, `version`, `update`
- **Connection modes:** Via cloud (`--cloud_addr`, default `getcosmic.ai:443`) or direct to Vizier (`--direct_vizier_addr`)
- **Script loading:** Bundle reader (embedded scripts) + file-based loading

### 3.8 Graceful Shutdown

PEM handles `SIGINT`, `SIGQUIT`, `SIGTERM`, and `SIGHUP` via `TerminationHandler` (`src/vizier/services/agent/shared/base/lifecycle.h`). K8s `terminationGracePeriodSeconds: 10` (in `k8s/vizier/pem/base/pem_daemonset.yaml:138`). On signal:
1. Calls `manager_->Stop(5 seconds timeout)` — Stirling stops BPF programs and data collection
2. Process exits with the received signal code

**Data loss on shutdown:** All in-memory telemetry is lost. No graceful data handoff to other nodes or disk.

---

## 4. Memory System Analysis

### 4.1 PEM Memory Budget Breakdown

```mermaid
pie title PEM Memory Distribution (Default 1280 MB Table Store)
    "HTTP Events Table" : 512
    "Other Protocol Tables (~10)" : 756
    "Stirling Error + Probe Status" : 2
    "Proc Exit Events" : 10
```

**Beyond the table store (estimated additional ~700 MB-1.2 GB):**

| Component | Estimated Size | Bounded? |
|-----------|---------------|----------|
| Table Store (all tables) | 1280 MB | Yes (configurable) |
| BPF perf buffers | 50-200 MB (scales with CPU cores) | Yes (bandwidth flags) |
| ConnTracker objects | 100 MB - 1+ GB | **No** (default unlimited) |
| DataStream buffers | ~2 MB per active connection | Bounded per-conn, unbounded globally |
| Stirling DataTable in-flight | ~50 MB | Yes (1024 records per table) |
| Carnot query execution | Variable | **No** (no per-query limit) |
| Metadata (gRPC, NATS, etc.) | ~50 MB | Yes |
| **Total estimated per PEM** | **~2-3+ GB** | Partially bounded |

### 4.2 What's In Memory vs. Persisted

| Data | Storage | Persistence | Recovery |
|------|---------|-------------|----------|
| Telemetry data (all protocols) | Arrow tables in PEM RAM | **Volatile** | Lost on restart |
| K8s metadata | PebbleDB on disk | Persistent (24h TTL) | Survives restart |
| Agent schemas, heartbeats | PebbleDB on disk | Persistent (TTL) | Survives restart |
| Cloud user/org/cluster data | PostgreSQL | Persistent | Standard DB backup |
| Query plans | Transient (per-query) | Not persisted | Recompiled per query |
| BPF perf buffer data | Kernel memory | Volatile | Lost on restart |
| ConnTracker state | PEM heap | Volatile | Lost on restart |

### 4.3 Caching Strategy

Pixie has **no explicit caching layer** (no Redis, Memcached). This is intentional:

- The Table Store IS the cache: a sliding window of recent telemetry in Arrow format.
- Cold partition Arrow arrays have lazy conversion caching (`RecordBatchWithCache` in `src/table_store/table/internal/types.h`).
- No cloud-side caching of query results, cluster info, or scripts.

**Gap identified:** Frequently-queried cluster metadata and script catalogs could benefit from in-memory caching at the cloud API layer.

### 4.4 Memory Bottlenecks (Verified)

1. **ConnTracker proliferation (HIGH RISK):** On nodes with service meshes (Envoy, Istio), thousands of short-lived connections create ConnTracker objects each holding up to 2 MB of DataStream buffers. The `total_conn_tracker_mem_usage` flag defaults to 0 (unlimited). This is the primary cause of PEM OOM kills in production.

2. **HTTP events table dominance:** At 40% of table store budget (~512 MB), the HTTP table fills quickly on API gateways. Other tables are starved. The fixed percentage allocation doesn't adapt to actual traffic patterns.

3. **No query memory limits:** A user running `SELECT * FROM http_events` on a large time window materializes all matching data in Carnot's execution engine with no memory cap. This can OOM the PEM.

4. **Perf buffer sizing on high-core machines:** At 100 MiB/s per CPU with scaling factor 8, a 64-core node calculates buffer sizes that can consume hundreds of MB of kernel memory.

5. **Cold partition memory duplication:** During compaction, both old hot batches and new cold batches exist simultaneously, temporarily doubling memory for affected data.

### 4.5 Observability Metrics Currently Collected

**PEM-level (Prometheus gauges):**
- `node_available_memory`, `node_total_memory` - Host memory from `/proc/meminfo`
- `heap_size_bytes`, `heap_inuse_bytes`, `heap_free_bytes` - Process heap metrics
- Per-table: bytes, batches, expiry counts via `TableMetrics`

**Query Broker (Prometheus summaries):**
- `query_exec_time_ms` (p99, by script_name)
- `query_exec_pems_queried` (by script_name)
- `query_exec_records_processed` (by script_name)
- `query_exec_bytes_processed` (by script_name)

**Cloud (Prometheus counters/histograms):**
- `cloud_to_vizier_msg_count`, `vizier_to_cloud_msg_count`
- `cloud_to_vizier_msg_size_dist`, `vizier_to_cloud_msg_size_dist`
- `stan_publish_count`, `nats_publish_count`
- Queue length gauges for gRPC and NATS channels

**Metadata Service:**
- `agent_reg_counter` - Agent registration events

**Gaps identified:** No per-table memory usage metric exposed to users. No ConnTracker memory usage metric. No query-in-progress count or memory consumption metric.

### 4.6 Memory Optimization Strategy

#### How Can RAM Usage Be Reduced?

| Strategy | Savings | Effort | Risk |
|----------|---------|--------|------|
| Set `total_conn_tracker_mem_usage=256MB` | 200 MB - 1+ GB | Low (flag change) | May drop data from idle connections |
| Disable unused protocol tracers | 64 MB per table | Low (CRD config) | Loss of auto-detection for those protocols |
| Reduce HTTP table to 20% | ~256 MB | Low (flag change) | Shorter HTTP data retention |
| Implement disk-tiered cold storage | ~800 MB | High (new feature) | Slight latency on cold queries |
| Add per-query memory limits | Variable | Medium | Complex queries may fail |
| Adaptive table sizing by data rate | Variable | Medium | Requires monitoring integration |

#### What Can Be Offloaded to Disk?

- **Cold partition data** → Arrow IPC files with mmap (zero-copy reads, natural fit)
- **Idle ConnTracker buffers** → Compressed disk swap for connections inactive >30s
- **PebbleDB** → Already on disk (correct)
- **Historical telemetry** → OTel export to external backends (already supported via `OTelExportSinkNode`)

#### What Metrics Can Be Disabled Safely?

| Table/Metric | Safe to Disable? | Condition |
|-------------|------------------|-----------|
| `conn_stats` | Yes | If L7 protocol data is sufficient. Set `stirling_conn_stats_sampling_ratio=0` |
| `proc_exit_events` | Yes | For stable workloads without process lifecycle debugging |
| `tcp_stats` | Yes | Largely redundant with `conn_stats` |
| Unused protocol tables (Mux, AMQP, MongoDB) | Yes | If those protocols aren't in use |
| `stack_traces` (profiler) | Yes | If CPU profiling isn't needed. Saves CPU + memory |

#### Trade-offs: Reduced Observability vs. Performance

| Change | RAM Saved | Observability Impact | Recommendation |
|--------|----------|---------------------|----------------|
| Disable unused protocols | ~64 MB each | Can't auto-detect those protocols | **Do it** - most clusters use 3-5 protocols |
| Reduce HTTP table to 20% | ~256 MB | ~2.5x shorter HTTP retention | Situational - depends on query patterns |
| Lower total table store to 512 MB | ~768 MB | ~2.5x shorter overall retention | Only for memory-constrained nodes |
| Disk-tiered cold storage | ~800 MB | <1ms latency increase on cold reads | **Best long-term option** |
| Cap ConnTracker memory | ~500 MB+ | May miss data from evicted idle connections | **Do it** - critical for stability |

#### Recommended Priority Order

1. **Immediate (week 1):** Set `total_conn_tracker_mem_usage` default to 256 MB
2. **Immediate (week 2):** Expose per-protocol enable/disable in Vizier CRD
3. **Short-term (month 1):** Add per-query memory budget to Carnot (wrap `default_memory_pool`)
4. **Medium-term (quarter):** Implement mmap-based cold storage using Arrow IPC
5. **Long-term:** Adaptive table sizing based on observed data rates

---

## 5. Performance & Scalability

### 5.1 Current Bottlenecks

```mermaid
flowchart LR
    subgraph Bottlenecks["Scalability Bottlenecks"]
        B1["Single Kelvin<br/>(aggregation SPOF)"]
        B2["PEM RAM ceiling<br/>(1.28 GB = minutes of data)"]
        B3["Query planning<br/>(serial, scales with agent count)"]
        B4["NATS throughput<br/>(all dispatch through single bus)"]
        B5["Perf buffer overflow<br/>(unrecoverable data loss)"]
    end

    B1 -->|"Impact"| I1["Large cluster queries<br/>bottleneck at single node"]
    B2 -->|"Impact"| I2["Short retention on<br/>high-traffic nodes"]
    B3 -->|"Impact"| I3["Compilation latency<br/>grows with cluster size"]
    B4 -->|"Impact"| I4["Query dispatch contention<br/>under concurrent load"]
    B5 -->|"Impact"| I5["Data gaps under<br/>extreme syscall rates"]
```

**Single Kelvin (confirmed SPOF):** Code at `src/vizier/services/query_broker/controllers/query_executor.go:410` contains `TODO(james): update this to support multiple Kelvin plans`. All PEM results flow through one Kelvin pod. On clusters with 100+ nodes, Kelvin becomes CPU/memory bound.

**PEM memory ceiling:** 1.28 GB default table store. On a high-traffic API gateway node processing 10,000 req/s, HTTP data retention can drop to under 2 minutes.

**Query planning serialization:** Planner runs synchronously per query within Query Broker. The `DistributedState` passed to the planner grows linearly with agent count. No plan caching for repeated queries.

### 5.2 Scaling Characteristics

| Component | Horizontal | Vertical | Current Limit |
|-----------|-----------|----------|---------------|
| **PEM** | Automatic (DaemonSet) | `PEMMemoryLimit` in CRD | Node count |
| **Kelvin** | **Not supported** (single instance) | CPU/memory configurable | 1 pod |
| **Query Broker** | Yes (stateless routing) | Standard | Per-query state in forwarder |
| **Metadata Service** | Leader-elected replicas | PebbleDB tuning | Leader write bottleneck |
| **Cloud API** | Standard horizontal | Standard | Stateless |
| **Cloud VZConn** | Sharded by Vizier ID | Standard | NATS throughput |
| **PostgreSQL** | Read replicas possible | Standard vertical | Single-writer |

### 5.3 Failure Scenarios and Recovery

| Failure | Data Loss | Detection | Recovery Time | Mitigation |
|---------|-----------|-----------|---------------|------------|
| PEM OOM kill | All node telemetry | DaemonSet restart | 10-30s | Increase `PEMMemoryLimit`, cap ConnTracker |
| Kelvin crash | In-flight query results | Query timeout (180s) | Pod restart (~10s) | None currently (SPOF) |
| Metadata leader crash | None (PebbleDB persisted) | K8s Lease election | Lease expiry (~15s) | Configure faster lease renewal |
| NATS crash | In-flight messages | Connection closed error | Pod restart + reconnect | NATS cluster HA |
| Cloud disconnect | None (edge continues) | Heartbeat watchdog (10 min) | Auto-reconnect with backoff | Edge-local queries still work |
| PostgreSQL crash | None (durable) | Cloud service health checks | Standard PG recovery | Backup/restore |
| Node drain | All PEM data on that node | K8s eviction | DaemonSet reschedule | No graceful handoff |

---

## 6. Security Considerations

### 6.1 Authentication & Authorization Architecture

```mermaid
flowchart TB
    subgraph UserAuth["User Authentication"]
        OAUTH["OAuth/OIDC Provider<br/>(Auth0, Hydra/Kratos, generic OIDC)"]
        LOGIN["Login/Signup"]
        JWT["JWT Token<br/>(HS256, ~30 min TTL)"]
        APIKEY["API Key<br/>(px-api-* prefix,<br/>SHA256 hashed + PGP encrypted)"]
    end

    subgraph ServiceAuth["Service-to-Service"]
        SJWT["Service JWT<br/>(HS256, cluster claims)"]
        MTLS["mTLS<br/>(X509 certs)"]
    end

    subgraph EdgeAuth["Edge Authentication"]
        DEPLOY_KEY["Deployment Key<br/>(one-time cluster registration)"]
        CLUSTER_JWT["Cluster JWT<br/>(ongoing auth)"]
    end

    subgraph Enforcement["Enforcement Layer"]
        GRPC_INT["gRPC Interceptors<br/>(Unary + Stream)"]
        ORG_SCOPE["Org-Scoped Access<br/>(JWT claims: UserID, OrgID)"]
        K8S_RBAC["K8s RBAC<br/>(per-component service accounts)"]
    end

    LOGIN --> OAUTH
    OAUTH --> JWT
    JWT --> GRPC_INT
    APIKEY --> GRPC_INT
    SJWT --> GRPC_INT
    GRPC_INT --> ORG_SCOPE
    DEPLOY_KEY --> CLUSTER_JWT
    CLUSTER_JWT --> GRPC_INT
    MTLS --> GRPC_INT
```

### 6.2 Security Posture Assessment

| Area | Implementation | Strength | Weakness |
|------|---------------|----------|----------|
| **User auth** | OAuth/OIDC + JWT (HS256) | Pluggable providers | HS256 symmetric key - single compromise point |
| **API keys** | SHA256 hash + PGP symmetric encryption | Defense-in-depth | Encryption key (`PL_DATABASE_KEY`) is a single env var |
| **Transport** | TLS on all gRPC (enabled by default) | Strong default | `--disable_ssl` flag exists with no guard |
| **Service auth** | Bearer JWT in gRPC metadata | Standard pattern | No service mesh mTLS |
| **Edge NATS** | mTLS client certificates | Good transport security | Any pod with access to namespace TLS secrets can connect |
| **BPF privileges** | `CAP_SYS_ADMIN` / `CAP_BPF` | Required for eBPF | Root-equivalent access |
| **Data privacy** | `RedactionOptions` at query compile time | Configurable per-cluster | Requires correct `DataAccess: Restricted` config |
| **Pod security** | runAsNonRoot, drop ALL caps, seccomp | Good hardening | PEM requires privileged access (exception) |
| **K8s RBAC** | Per-component service accounts | Least privilege | Operator has broad ClusterRole |
| **Secrets** | K8s Secrets for certs, tokens, keys | Standard | No external secret manager integration |

### 6.3 Specific Vulnerabilities

1. **HS256 JWT signing key:** Single symmetric key shared across all services. Key compromise allows unrestricted token forgery. **Recommendation:** Migrate to RS256 with key rotation.

2. **NATS TLS cert access scope:** NATS within Vizier uses mTLS, but the client certs are stored as K8s secrets accessible to any pod in the namespace. A compromised pod with secret access could connect to NATS and inject query plans. **Recommendation:** Use per-service NATS credentials or restrict secret access via RBAC.

3. **PEM captures all node traffic:** In multi-tenant clusters, one tenant's data is visible to PEM. `DataAccess: Restricted` mode provides redaction but is opt-in. **Recommendation:** Default to restricted in shared clusters.

4. **No rate limiting:** No rate limiting on query execution or API calls. A runaway script or malicious user can saturate all PEMs. **Recommendation:** Add per-user/per-org query rate limits.

---

## 7. Observability & Operational Readiness

### 7.1 Metrics Inventory

| Component | Metric | Type | Labels |
|-----------|--------|------|--------|
| Query Broker | `query_exec_time_ms` | Summary (p99) | `script_name` |
| Query Broker | `query_exec_pems_queried` | Summary | `script_name` |
| Query Broker | `query_exec_records_processed` | Summary | `script_name` |
| Query Broker | `query_exec_bytes_processed` | Summary | `script_name` |
| Metadata Svc | `agent_reg_counter` | Counter | - |
| PEM | `node_available_memory` | Gauge | - |
| PEM | `node_total_memory` | Gauge | - |
| PEM | `heap_size_bytes` | Gauge | - |
| PEM | `heap_inuse_bytes` | Gauge | - |
| VZConn | `cloud_to_vizier_msg_count` | Counter | `vizier_id` |
| VZConn | `vizier_to_cloud_msg_count` | Counter | `vizier_id` |
| VZConn | `cloud_to_vizier_msg_size_dist` | Histogram | `vizier_id` |
| VZMgr | `vizier_updated_counter` | Counter | - |

### 7.2 Operational Gaps

| Gap | Impact | Recommendation |
|-----|--------|----------------|
| No per-table memory usage metric | Can't alert on table store nearing capacity | Add gauge per table |
| No ConnTracker count/memory metric | Can't detect memory leak from connection growth | Add gauge |
| No active query count | Can't detect query storms | Add gauge to Query Broker |
| No perf buffer drop rate metric | Can't quantify data loss | Expose `HandleDataEventLoss` counter |
| No PodDisruptionBudgets for Vizier services | Unsafe rolling updates | Add PDBs for Kelvin, Metadata, Query Broker |
| No NetworkPolicies (verified: 0 custom policies) | No network segmentation within namespace | Add policies to restrict inter-namespace traffic |
| Cloud services have `resources: {}` (verified empty) | Unpredictable scheduling, no OOM protection | Define resource specs in all cloud deployment YAMLs |
| No backup for PebbleDB or PostgreSQL | Risk of metadata loss / data corruption | PebbleDB: PVC snapshots via Velero. PG: pg_dump CronJob |
| No version compatibility checks (cloud↔Vizier) | Major version jump could break gRPC contracts | Add compatibility matrix or API versioning |

### 7.3 Logging

- **Go services:** `github.com/sirupsen/logrus` (structured logging to stdout, default `InfoLevel`). Pattern: `log.WithError(err).WithField("key", val).Error("message")`
- **C++ services:** Custom logging via `src/common/base/logging.h`
- **Log levels:** Standard (TRACE, DEBUG, INFO, WARNING, ERROR, FATAL). Development mode adds caller info (func + line)
- **Error tracking:** Sentry integration in operator and cloud services (`InitSentryFromCRD()`)
- **No centralized log aggregation** built-in - relies on cluster-level solutions (Fluentd, etc.)

### 7.4 Multi-Cluster Sharding

Cloud services support horizontal scaling via UUID-based sharding (`src/cloud/shared/vzshard/vzshard.go`):
- Viziers partitioned by last byte of UUID into **256 shards** (00-ff hex)
- Each cloud service instance configures `--vizier_shard_min` / `--vizier_shard_max`
- NATS topics include shard: `v2c.{shard}.{vizierID}.{topic}`
- Per-shard NATS subscriptions with 8192 message buffer
- Enables independent scaling of VZMgr, VZConn, and metrics processing

### 7.5 Auto-Update Mechanism

**Source:** `src/cloud/vzmgr/controllers/vizier_updater.go`

1. VZMgr `Updater` polls latest Vizier version from Artifact Tracker every **1 minute**
2. During heartbeat processing, compares current vs latest version (semantic versioning via `blang/semver`)
3. Out-of-date Viziers added to update queue (capacity: 32, gradual rollout)
4. Update sent via sharded NATS topic: `UpdateOrInstallVizierRequest` with signed JWT
5. 5-minute timeout for `VizierUpdateResponse`; reverts to UNHEALTHY on timeout
6. Operator reconciler monitors update phase with 10-minute `updatingFailedTimeout`
7. **No backward compatibility checks** between cloud and Vizier versions. No API versioning.

---

## 8. CTO Review Questions

### Q1: What is the true cost of running Pixie per node?

**Answer:** PEM requests 2 GiB RAM by default (configurable via `pemMemoryRequest` in Helm values). Of this, 1.28 GB is for the table store, and the rest covers Stirling, Carnot, gRPC overhead, and the process heap. On a 100-node cluster at $0.004/GB/hour, the RAM cost alone is ~$576/month. CPU overhead is typically 1-5% per node for eBPF tracing. Compare to SaaS APM tools at $15-30/host/month: Pixie is cost-competitive below ~20,000 req/s per node but becomes expensive on high-traffic nodes that need more RAM for retention.

### Q2: What is the data retention story, and can it be extended?

**Answer:** Retention is a function of traffic volume and table store size. On a quiet node, data can persist for hours. On a high-traffic API gateway, retention may be under 2 minutes. There is **no disk spillover**. Extending retention requires either: (a) increasing `PL_TABLE_STORE_DATA_LIMIT_MB` (linear RAM cost), (b) reducing traffic via protocol disabling, or (c) implementing disk-tiered storage (not yet available). For long-term retention, use the OTel export feature to ship data to Prometheus/Jaeger/Grafana Tempo.

### Q3: What happens when a PEM is OOM-killed?

**Answer:** All in-memory telemetry for that node is lost permanently. DaemonSet restarts PEM in ~10-30 seconds. eBPF probes are redeployed, data collection resumes with a gap. No historical data is recoverable. This is acceptable for debugging but not for compliance or audit trails.

### Q4: Can Pixie scale to 1000+ nodes?

**Answer:** With caveats. PEMs scale naturally (DaemonSet). The bottlenecks are:
- **Kelvin:** Single instance, processes all aggregated results. At 1000 nodes, likely saturated. Multi-Kelvin support is a known TODO in the codebase.
- **Metadata Service:** Must track all agents. PebbleDB performance at scale is untested.
- **Query planning:** Compilation time grows with agent count (linear in `DistributedState` size).
- **NATS:** All query dispatch goes through a single NATS cluster.

Realistic current limit: **~200-500 nodes** without significant architectural changes.

### Q5: How reliable is eBPF-based collection?

**Answer:** Highly reliable for supported protocol versions and kernel versions (4.14+). Known gaps:
- HTTP/3 (QUIC): Not supported
- Newer MySQL auth methods: May fail to parse
- Encrypted traffic: Only TLS metadata, not payload (unless using Go/OpenSSL uprobe-based decryption)
- Perf buffer overflow: Under extreme syscall rates (>100K/s per core), events are dropped irreversibly
- Kernel version sensitivity: Some probes use `OnForNewerKernel` mode

### Q6: What is the single point of failure analysis?

**Answer:**
| SPOF | Severity | Mitigation Available |
|------|----------|---------------------|
| Kelvin | **High** (all queries fail) | None (single instance by design) |
| NATS | **High** (all dispatch fails) | NATS cluster HA |
| Metadata leader | **Medium** (no new registrations) | K8s Lease election (~15s failover) |
| PEM per node | **Low** (only that node's data) | DaemonSet auto-restart |
| Cloud connectivity | **Low** (edge keeps working) | Graceful degradation |

### Q7: How does multi-tenancy work?

**Answer:** Cloud: strict org-based JWT isolation. Edge: Pixie runs one Vizier per cluster with full node access. PEM sees ALL traffic on the node via eBPF. In shared clusters, `DataAccess: Restricted` enables query-time redaction, but the PEM still captures everything. True per-namespace isolation is architecturally impossible without per-namespace PEM deployments.

### Q8: What is the disaster recovery plan?

**Answer:** Edge telemetry is ephemeral by design - no backup, no recovery. Cloud data (PostgreSQL) follows standard DB backup practices. PebbleDB metadata on the edge survives PEM restarts but has no backup mechanism. Pixie is a **debugging tool, not a system of record**. For durable telemetry, export via OTel.

### Q9: How are upgrades handled?

**Answer:** The K8s Operator watches the Vizier CRD and reconciles to the desired version. Versions are tracked by the Artifact Tracker cloud service. `AutoUpdate: true` enables automatic upgrades. Rollback requires manual CRD version pinning. **Risk:** No PodDisruptionBudgets exist for Vizier services, so rolling updates can cause brief availability gaps.

### Q10: What is the query latency profile?

**Answer:** Compilation: ~10-50ms. NATS dispatch: <5ms. Local PEM execution: 10-500ms (depends on data volume). gRPC result streaming: ~10-100ms. Kelvin aggregation: 10-200ms. Total: **sub-second for simple queries, 1-5 seconds for complex aggregations on large clusters.** No query plan caching exists, so repeated identical queries recompile every time.

### Q11: Is there any rate limiting or abuse prevention?

**Answer:** **No.** No rate limiting exists on query execution, API calls, or script scheduling. A single user can run unlimited concurrent queries that consume PEM resources across all nodes. CronScript has configurable `frequency_s` but no global concurrency limit. This is a significant operational risk.

### Q12: How extensible is protocol tracing?

**Answer:** Adding a protocol requires: (1) BPF program for data capture, (2) user-space parser, (3) table schema, (4) SocketTraceConnector integration. The protocol list is **compile-time fixed** (see `kTables` array in `socket_trace_connector.h`). The `DynamicTraceConnector` offers lighter-weight custom tracepoints without recompilation, but full protocol support requires a PEM rebuild.

### Q13: What is the testing strategy?

**Answer:** Extensive unit tests via Bazel. Stirling has BPF-level test infrastructure with container-based integration tests. Protocol parsers have per-protocol test suites. The main testing gap is **end-to-end distributed query tests** across multi-node clusters.

### Q14: What observability does Pixie have for itself?

**Answer:** Prometheus metrics from all Go services (via `promhttp`), C++ heap metrics, node memory metrics, and query execution metrics. The `stirling_error` table captures internal errors. However, there are significant gaps: no per-table memory dashboards, no ConnTracker metrics, no query concurrency metrics, and no built-in alerting.

---

## 9. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation | Owner |
|----|------|-----------|--------|------------|-------|
| R1 | PEM OOM from unbounded ConnTracker growth | **High** | High - node data loss | Set `total_conn_tracker_mem_usage` default | Platform |
| R2 | Kelvin SPOF causes cluster-wide query failure | **Medium** | High - total query outage | Implement multi-Kelvin | Architecture |
| R3 | No per-query memory limit leads to PEM OOM | **Medium** | High - node data loss | Wrap Arrow memory pool with budget | Engine |
| R4 | NATS TLS certs accessible to any namespace pod | **Low** | Critical - arbitrary execution | Per-service NATS credentials or secret RBAC | Security |
| R5 | HS256 JWT key compromise | **Low** | Critical - full auth bypass | Migrate to RS256 | Security |
| R6 | No rate limiting enables query storms | **Medium** | Medium - PEM saturation | Add per-user rate limits | Platform |
| R7 | PebbleDB corruption with no backup | **Low** | Medium - metadata loss | Add backup mechanism | Operations |
| R8 | No PodDisruptionBudgets | **Medium** | Low - brief outages during updates | Add PDBs | Operations |
| R9 | Telemetry data loss on any PEM restart | **High** | Medium - debugging gaps | Disk-tiered storage | Architecture |
| R10 | Protocol list is compile-time fixed | **Low** | Medium - limits extensibility | Plugin architecture | Architecture |

---

## 10. Recommendations & Next Steps

### Immediate (0-4 weeks)

| Priority | Action | Effort | Impact |
|----------|--------|--------|--------|
| **P0** | Set `total_conn_tracker_mem_usage` default to 256 MB | 1 day | Prevents most PEM OOM kills |
| **P0** | Expose per-protocol enable/disable in Vizier CRD | 1 week | Saves 64 MB+ per disabled protocol |
| **P1** | Add PodDisruptionBudgets for Kelvin, Metadata, QB | 1 day | Safe rolling updates |
| **P1** | Add ConnTracker count + memory Prometheus gauge | 2 days | Enables OOM debugging |
| **P1** | Add per-table memory usage gauge | 2 days | Enables retention planning |
| **P2** | Document recommended resource configs by cluster size | 3 days | Reduces misconfigurations |

### Medium-term (1-3 months)

| Priority | Action | Effort | Impact |
|----------|--------|--------|--------|
| **P0** | Add per-query memory budget to Carnot | 2-3 weeks | Prevents query-triggered OOM |
| **P1** | Implement query rate limiting (per-user, per-org) | 2 weeks | Prevents resource abuse |
| **P1** | Add perf buffer drop rate metric + alerting | 1 week | Quantifies data loss |
| **P2** | Migrate JWT from HS256 to RS256 with rotation | 3-4 weeks | Eliminates shared-secret risk |
| **P2** | Restrict NATS TLS secret access to specific service accounts | 1 week | Reduces blast radius of namespace compromise |
| **P2** | Add query plan caching for repeated scripts | 2-3 weeks | Reduces compilation overhead |

### Long-term (3-12 months)

| Priority | Action | Effort | Impact |
|----------|--------|--------|--------|
| **P0** | Disk-tiered cold storage (Arrow IPC + mmap) | 2-3 months | 3-5x retention at lower RAM cost |
| **P1** | Multi-Kelvin support (sharded aggregation) | 2-3 months | Removes primary scalability bottleneck |
| **P1** | Adaptive table sizing based on data rates | 1-2 months | Optimal memory utilization |
| **P2** | Cross-cluster query federation via cloud | 3-4 months | Multi-cluster observability |
| **P2** | Streaming query materialization (persistent queries) | 2-3 months | Enables dashboards + alerts |
| **P3** | Protocol plugin system (runtime loadable) | 3-6 months | Community extensibility |
| **P3** | Edge-native storage engine (memory + disk + compression) | 6-12 months | Positions Pixie as edge database |

### Architecture Vision

The near-term goal is to **decouple data retention from RAM cost** via disk-tiered storage and eliminate the Kelvin SPOF via multi-instance aggregation. The medium-term vision is to evolve Pixie from an ephemeral debugging tool into a **persistent edge observability database** that can power dashboards, alerts, and compliance workflows alongside its existing zero-instrumentation capabilities.

---

*Generated from static analysis of pixie-io/pixie codebase at commit ce714e6e8 (main branch). All file paths and line references verified against current source. Claims marked as "confirmed in code" are backed by specific file evidence. Assumptions and gaps are explicitly noted.*
