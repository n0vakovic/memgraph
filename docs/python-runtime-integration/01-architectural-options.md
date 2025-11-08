# Python Runtime Integration: Architectural Options

**Date:** 2025-11-08
**Status:** Analysis
**Purpose:** Analyze architectural options for integrating Python runtime into Memgraph platform for RAG and agentic applications

---

## Executive Summary

This document analyzes architectural options for deep Python runtime integration into Memgraph, building upon the existing MAGE (Memgraph Advanced Graph Extensions) model. The goal is to enable Memgraph as a comprehensive RAG (Retrieval-Augmented Generation) platform with full Python ecosystem integration, customizability, and support for autonomous agents.

**Key Findings:**
- Memgraph already has sophisticated Python integration via the MAGE model
- Multiple integration levels are possible, from MAGE-compatible to deeply embedded
- Security and sandboxing require careful architectural choices
- Query execution pipeline offers multiple extension points for Python runtime

---

## Current State: MAGE Architecture

### Overview

Memgraph currently supports Python through a mature, production-ready extension system:

**Architecture:**
```
┌─────────────────────────────────────────────────────────┐
│                    Cypher Query                         │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              Query Execution Engine                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Operator   │→ │ CallProcedure│→ │   Operator   │  │
│  └──────────────┘  └──────┬───────┘  └──────────────┘  │
└─────────────────────────────┼───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              Module Registry (C++)                      │
│         (Thread-safe, RWLock-protected)                 │
└────────┬───────────────────────────────────┬────────────┘
         │                                   │
         ▼                                   ▼
┌──────────────────┐              ┌──────────────────────┐
│ SharedLibModule  │              │   PythonModule       │
│   (.so files)    │              │   (.py files)        │
│                  │              │                      │
│  C/C++ via       │              │  Embedded CPython    │
│  dlopen()        │              │  Interpreter         │
└──────────────────┘              └──────────┬───────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────┐
│              Python Layer (2-tier)                      │
│  ┌────────────────────────────────────────────────┐    │
│  │  High Level: mgp.py (Pythonic API)             │    │
│  │  - Decorators: @mgp.read_proc, @mgp.write_proc │    │
│  │  - Classes: Graph, Vertex, Edge, Path          │    │
│  │  - Type hints and Python idioms                │    │
│  └──────────────────┬─────────────────────────────┘    │
│                     │                                   │
│  ┌──────────────────▼─────────────────────────────┐    │
│  │  Low Level: _mgp C Extension                   │    │
│  │  - Direct Python C API bindings                │    │
│  │  - PyObject ↔ mgp_value conversions           │    │
│  │  - Memory management                           │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Key Characteristics

**Strengths:**
- ✅ **Mature & Production-Ready**: Battle-tested in real deployments
- ✅ **Type-Safe**: Strong type system with validation
- ✅ **Memory-Managed**: Custom allocators with per-query limits
- ✅ **Thread-Safe**: Concurrent procedure execution
- ✅ **Hot-Reloadable**: Dynamic module loading without restart
- ✅ **Multi-Language**: Supports C, C++, Python, Rust
- ✅ **Stream-Capable**: Special support for Kafka/Pulsar transformations

**Limitations:**
- ❌ **No Process Isolation**: Procedures run in-process
- ❌ **No Resource Limits**: CPU/time not constrained (only memory)
- ❌ **No Filesystem Restrictions**: Full file system access
- ❌ **No Network Restrictions**: Can make arbitrary network calls
- ❌ **Coarse-Grained**: Procedure-level, not expression-level Python

**API Surface:**

```python
# Current Python API Example
import mgp

@mgp.read_proc
def analyze_graph(ctx: mgp.ProcCtx,
                  depth: int = 3) -> mgp.Record(score=float):
    graph = ctx.graph
    for vertex in graph.vertices:
        # Full graph access
        neighbors = vertex.out_edges
        # Complex Python logic
        score = complex_algorithm(vertex, neighbors, depth)
        yield mgp.Record(score=score)

@mgp.transformation
def process_stream(messages: mgp.Messages) -> mgp.Record(params=mgp.Map):
    for msg in messages:
        data = json.loads(msg.payload())
        yield mgp.Record(params=transform(data))
```

---

## Architectural Option 1: Enhanced MAGE Model (Evolutionary)

**Approach:** Extend the current MAGE architecture with incremental improvements

### Proposed Enhancements

#### 1.1 Python REPL Integration

**Architecture:**
```
┌──────────────────────────────────────────────────────┐
│  Interactive Python REPL (via Bolt/WebSocket)       │
│                                                      │
│  > py_session = CREATE_PYTHON_SESSION()             │
│  > CALL py.eval(py_session, "import numpy as np")   │
│  > CALL py.exec(py_session, "results = []")         │
│  > MATCH (n) CALL py.process(py_session, n)         │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────┐
│       Python Session Manager (C++)                   │
│  - Session lifecycle (create, execute, destroy)     │
│  - Isolated Python sub-interpreters (PEP 554)       │
│  - Context sharing between Cypher and Python        │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────┐
│       CPython Sub-Interpreters                       │
│  - Per-session state isolation                      │
│  - Shared graph accessor                            │
│  - Variable persistence across calls                │
└──────────────────────────────────────────────────────┘
```

**Implementation:**
- Location: `/home/user/memgraph/src/query/procedure/py_session.{hpp,cpp}`
- New procedures: `py.create_session()`, `py.eval()`, `py.exec()`, `py.close_session()`
- Uses Python 3.12+ sub-interpreters for isolation
- Session state stored in `ExecutionContext` or transaction metadata

**Benefits:**
- ✅ Interactive data exploration
- ✅ Stateful Python scripts across multiple queries
- ✅ Jupyter-like notebook experience
- ✅ Minimal changes to core architecture

**Challenges:**
- ⚠️ Sub-interpreter stability (improving in Python 3.12+)
- ⚠️ Session lifecycle management (when to clean up?)
- ⚠️ Transaction boundaries (sessions vs transactions)

#### 1.2 Enhanced Trigger System with Python Callbacks

**Architecture:**
```
┌──────────────────────────────────────────────────────┐
│  CREATE TRIGGER notify_on_create                     │
│  ON CREATE VERTEX                                    │
│  EXECUTE PYTHON FUNCTION "hooks.on_vertex_created"  │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────┐
│       Trigger Store (Extended)                       │
│  - TriggerEventType (existing)                       │
│  - TriggerExecutionType (NEW)                        │
│    * CYPHER_QUERY (existing)                         │
│    * PYTHON_CALLBACK (new)                           │
│    * PROCEDURE_CALL (new)                            │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────┐
│       Python Trigger Executor                        │
│  - Direct callback invocation                        │
│  - Pass TriggerContext as Python object              │
│  - Handle exceptions gracefully                      │
└──────────────────────────────────────────────────────┘
```

**Implementation:**
- Location: `/home/user/memgraph/src/query/trigger.{hpp,cpp}`
- Extend `Trigger` struct with `python_callback_` field
- Add `PythonTriggerExecutor` class
- Register Python functions via `REGISTER PYTHON TRIGGER FUNCTION`

**Example:**
```python
# In trigger_module.py
import mgp

def on_vertex_created(context: mgp.TriggerContext):
    """Called when vertices are created"""
    for vertex in context.created_vertices:
        # Send notification to external service
        notify_service(vertex.id, vertex.labels)
        # Log to audit system
        audit_log.record("vertex_created", vertex.properties)
```

**Benefits:**
- ✅ Lower latency than Cypher trigger queries
- ✅ Direct Python integration for reactive logic
- ✅ Easier debugging and testing
- ✅ Natural fit for external integrations

#### 1.3 Transaction Lifecycle Hooks

**Hook Points:**
```python
# Python transaction hooks API
import mgp

@mgp.on_transaction_start
def validate_transaction(tx_ctx: mgp.TransactionContext):
    """Called before transaction starts"""
    if not is_authorized(tx_ctx.user):
        raise mgp.AbortTransaction("Unauthorized")

@mgp.on_pre_commit
def pre_commit_hook(tx_ctx: mgp.TransactionContext):
    """Called before commit, can modify or abort"""
    # Complex validation logic
    if not validate_constraints(tx_ctx.changes):
        raise mgp.AbortTransaction("Constraint violation")

    # Add computed properties
    for vertex in tx_ctx.created_vertices:
        vertex.properties["created_at"] = datetime.now()

@mgp.on_post_commit
def post_commit_hook(tx_ctx: mgp.TransactionContext):
    """Called after successful commit (read-only)"""
    # Send to external systems
    sync_to_elasticsearch(tx_ctx.changes)
    emit_event_stream(tx_ctx.transaction_id, tx_ctx.changes)

@mgp.on_rollback
def rollback_hook(tx_ctx: mgp.TransactionContext):
    """Called on transaction rollback"""
    log_rollback(tx_ctx.transaction_id, tx_ctx.error)
```

**Implementation:**
- Location: `/home/user/memgraph/src/query/interpreter.cpp`
- Add hook registry: `TransactionHookRegistry`
- Call hooks at appropriate points in `Interpreter::CommitTransaction()`, etc.
- Pass `TransactionContext` with changes, user info, transaction metadata

**Benefits:**
- ✅ Audit logging
- ✅ External system synchronization
- ✅ Complex validation logic
- ✅ Event-driven architectures

#### 1.4 Custom Operators (Python-backed)

**Architecture:**
```
┌──────────────────────────────────────────────────────┐
│  MATCH (n:User)                                      │
│  PYTHON FILTER "filters.is_premium_user(n)"         │
│  RETURN n                                            │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────┐
│       Query Planner                                  │
│  - Recognize PYTHON keyword                          │
│  - Create PythonFilterOperator                       │
│  - Include in query plan                             │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────┐
│       PythonFilterOperator : LogicalOperator         │
│  - MakeCursor() → PythonFilterCursor                 │
│                                                      │
│  class PythonFilterCursor : Cursor {                 │
│    bool Pull(Frame &, ExecutionContext &) override { │
│      while (input_cursor->Pull(...)) {               │
│        if (python_predicate(frame)) return true;     │
│      }                                               │
│      return false;                                   │
│    }                                                 │
│  }                                                   │
└──────────────────────────────────────────────────────┘
```

**Implementation:**
- Location: `/home/user/memgraph/src/query/plan/operator.{hpp,cpp}`
- New operators: `PythonFilter`, `PythonMap`, `PythonAggregate`
- Cypher syntax extensions: `PYTHON FILTER`, `PYTHON MAP`, `PYTHON REDUCE`
- Optimize by compiling Python to bytecode once

**Example:**
```cypher
MATCH (u:User)-[:PURCHASED]->(p:Product)
PYTHON FILTER "ml_model.predict_churn(u) > 0.8"
PYTHON MAP "enrichment.add_demographics(u)"
RETURN u, collect(p) as products
```

**Benefits:**
- ✅ Tight integration with query execution
- ✅ Better optimization opportunities
- ✅ Push complex logic down to execution layer
- ✅ Reduced data movement

**Challenges:**
- ⚠️ Parser changes required
- ⚠️ Optimization complexity
- ⚠️ Error handling across language boundary

### Summary: Enhanced MAGE Model

**Implementation Effort:** Medium (3-6 months)
**Risk:** Low (builds on proven architecture)
**Compatibility:** High (backward compatible)

**Recommended for:**
- Organizations already using Memgraph
- Production deployments requiring stability
- Incremental capability enhancement
- Teams familiar with MAGE model

---

## Architectural Option 2: Deep Embedded Python (Revolutionary)

**Approach:** Make Python a first-class language alongside Cypher

### Vision

Transform Memgraph into a true polyglot database where Python and Cypher are equally supported:

```python
# Python-first API
from memgraph import Graph

with Graph.connect("bolt://localhost:7687") as g:
    # Pythonic graph operations
    for user in g.vertices.filter(labels=["User"]):
        if ml_model.predict_churn(user) > 0.8:
            user.properties["churn_risk"] = "HIGH"

            # Mixed: Execute Cypher from Python
            recommendations = g.query("""
                MATCH (u:User {id: $user_id})-[:SIMILAR_TO]->(other:User)
                MATCH (other)-[:PURCHASED]->(p:Product)
                WHERE NOT (u)-[:PURCHASED]->(p)
                RETURN p
            """, user_id=user.id)

            # Back to Python
            send_retention_campaign(user, recommendations)

    # Commit transaction
    g.commit()
```

### Architecture

```
┌───────────────────────────────────────────────────────────┐
│                 Unified Query Interface                   │
│  ┌─────────────────────┐  ┌──────────────────────┐        │
│  │   Cypher Queries    │  │   Python Scripts     │        │
│  └──────────┬──────────┘  └──────────┬───────────┘        │
└────────────┼────────────────────────┼─────────────────────┘
             │                        │
             ▼                        ▼
┌───────────────────────────────────────────────────────────┐
│              Unified Execution Engine                     │
│  ┌────────────────────┐  ┌──────────────────────┐         │
│  │  Cypher Planner    │  │  Python Executor     │         │
│  │  & Optimizer       │  │  (JIT compiled)      │         │
│  └────────┬───────────┘  └──────────┬───────────┘         │
│           │                         │                     │
│           └──────────┬──────────────┘                     │
│                      ▼                                    │
│           ┌─────────────────────┐                         │
│           │  Unified Operator   │                         │
│           │  Execution Layer    │                         │
│           └──────────┬──────────┘                         │
└──────────────────────┼────────────────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────────────────┐
│              Storage Engine (v2)                          │
│         MVCC, Indices, Durability                         │
└───────────────────────────────────────────────────────────┘
```

### Key Components

#### 2.1 Python Query Language

**Full Python as query language:**
- Python scripts are transactions
- Direct graph manipulation API
- Implicit MVCC and transaction management
- JIT compilation for performance

**Implementation:**
- Embed Python interpreter deeply in query layer
- Provide Pythonic graph API (similar to NetworkX)
- Auto-generate transaction boundaries
- Use PyPy or Cython for JIT compilation

#### 2.2 Unified Type System

```python
# Shared type system between Cypher and Python
TypedValue = Union[
    None,                    # Null
    bool,                    # Boolean
    int,                     # Integer
    float,                   # Float
    str,                     # String
    list,                    # List
    dict,                    # Map
    datetime.date,           # Date
    datetime.time,           # LocalTime
    datetime.datetime,       # LocalDateTime
    Vertex,                  # Graph node
    Edge,                    # Graph relationship
    Path,                    # Graph path
]
```

**Zero-copy conversions:** Python objects directly backed by storage layer

#### 2.3 Cross-Language Optimization

```
Python: for v in g.vertices.filter(age > 30)
   ↓
AST Analysis
   ↓
Detect: "Simple filter on indexed property"
   ↓
Rewrite to: Cypher index lookup
   ↓
Execute: Optimized index scan
```

**Optimizations:**
- Detect common Python patterns
- Translate to optimized operators
- Use indices even from Python code
- Push filters and projections down

### Implementation Challenges

**Major Undertaking:**
- ❌ **High Complexity**: Requires rearchitecting query layer
- ❌ **Performance Risk**: Python overhead vs native C++
- ❌ **GIL Limitations**: Python Global Interpreter Lock
- ❌ **Compatibility**: Breaking changes to existing API

**Estimated Effort:** 12-24 months, 3-5 engineers

### When This Makes Sense

**Use Cases:**
- New product/platform (not extending existing Memgraph)
- Python-first user base (data scientists, ML engineers)
- Notebook-centric workflows (Jupyter integration critical)
- Competing directly with Neo4j's Python experience

**Risk Profile:** High risk, high reward

---

## Architectural Option 3: Microservices Model (Distributed)

**Approach:** Python runtime as separate service(s) communicating with Memgraph

### Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  Memgraph Core                           │
│              (Graph Database + Cypher)                   │
└────────┬─────────────────────────────────────────┬───────┘
         │                                         │
         │ Bolt/HTTP/gRPC                         │
         │                                         │
    ┌────▼──────────┐                      ┌──────▼──────────┐
    │  Python       │                      │   Python        │
    │  Compute      │◄────Message Bus─────►│   Agents        │
    │  Service      │      (Kafka/NATS)    │   Service       │
    └────┬──────────┘                      └──────┬──────────┘
         │                                         │
         │                                         │
    ┌────▼─────────────────────────────────────────▼──────┐
    │          Shared State / Cache                       │
    │          (Redis, Memcached)                         │
    └─────────────────────────────────────────────────────┘
```

### Components

#### 3.1 Python Compute Service

**Responsibilities:**
- Execute Python procedures remotely
- Manage Python dependencies and environments
- Scale horizontally for compute-intensive tasks
- Cache compiled bytecode and models

**Protocol:**
```protobuf
service PythonCompute {
  rpc ExecuteProcedure(ProcedureRequest) returns (stream ResultRow);
  rpc LoadModule(ModuleDefinition) returns (LoadStatus);
  rpc HealthCheck(Empty) returns (HealthStatus);
}

message ProcedureRequest {
  string module_name = 1;
  string procedure_name = 2;
  repeated Value arguments = 3;
  GraphSnapshot context = 4;  // Serialized graph subset
}
```

**Benefits:**
- ✅ Process isolation
- ✅ Independent scaling
- ✅ Language version flexibility
- ✅ Fault isolation (Python crash doesn't crash DB)

#### 3.2 Python Agents Service

**Autonomous agents with goals:**
```python
# Agent definition
class DataQualityAgent(Agent):
    async def run(self):
        while True:
            # Monitor graph for quality issues
            async with memgraph.transaction() as tx:
                issues = await tx.query("""
                    MATCH (n) WHERE n.email IS NULL
                    RETURN n.id as id
                """)

                for issue in issues:
                    # Autonomous remediation
                    await self.fix_missing_email(issue['id'])

            await asyncio.sleep(60)  # Run every minute
```

**Architecture:**
- Event-driven agent execution
- Trigger-based activation
- Schedule-based execution
- Goal-seeking behavior

#### 3.3 Communication Patterns

**Pattern 1: Request-Response**
```
Cypher Query → Call Python Procedure → RPC to Compute Service → Execute → Stream Results
```

**Pattern 2: Event-Driven**
```
Graph Change → Trigger → Publish Event → Agent Consumes → Take Action → Update Graph
```

**Pattern 3: Continuous Processing**
```
Kafka Stream → Transform (Python) → Cypher Query → Memgraph
```

### Trade-offs

**Pros:**
- ✅ **Isolation**: Services can crash independently
- ✅ **Scalability**: Scale Python compute separately from DB
- ✅ **Flexibility**: Use different Python versions, dependencies
- ✅ **Security**: Network-level boundaries
- ✅ **Polyglot**: Easy to add other language runtimes

**Cons:**
- ❌ **Latency**: Network overhead for every call
- ❌ **Complexity**: Distributed system challenges (discovery, coordination)
- ❌ **Serialization**: Must serialize graph data over the wire
- ❌ **Transactions**: Distributed transaction coordination
- ❌ **Debugging**: Harder to trace across services

**When to Use:**
- Large-scale deployments
- Untrusted Python code (multi-tenant)
- Compute-heavy workloads (ML inference)
- Need for independent scaling

---

## Architectural Option 4: Hybrid Model (Pragmatic)

**Approach:** Combine in-process (MAGE) for low-latency, microservices for isolation

### Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    Memgraph Core                           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │            In-Process Python (MAGE++)                │  │
│  │  - Low-latency procedures                            │  │
│  │  - Trusted code only                                 │  │
│  │  - Performance-critical paths                        │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │            Sandbox Manager                           │  │
│  │  - Decides: in-process vs remote execution           │  │
│  │  - Based on: trust level, resource limits, SLA      │  │
│  └──────────────┬───────────────────────────────────────┘  │
└─────────────────┼──────────────────────────────────────────┘
                  │
      ┌───────────┴───────────┐
      │                       │
      ▼                       ▼
┌─────────────┐         ┌──────────────┐
│  Sandboxed  │         │   Agent      │
│  Python     │         │   Runtime    │
│  (Remote)   │         │   (Remote)   │
└─────────────┘         └──────────────┘
```

### Decision Matrix

**Execute In-Process If:**
- ✅ Code is trusted (vetted by security team)
- ✅ Latency requirement < 10ms
- ✅ Memory usage < 100MB
- ✅ No long-running computation (< 1s)
- ✅ No external network calls

**Execute Remotely If:**
- ✅ User-provided code (untrusted)
- ✅ Large dependencies (ML models, etc.)
- ✅ Long-running (> 1s)
- ✅ Requires specific Python version
- ✅ Makes external API calls

### Implementation

```python
# Decorator determines execution strategy
from memgraph import procedure

@procedure.trusted  # Runs in-process
def fast_filter(ctx, threshold: float) -> bool:
    return ctx.vertex.properties["score"] > threshold

@procedure.sandboxed  # Runs remotely
def analyze_with_ml(ctx, model_name: str):
    # Loads large ML model, takes seconds
    model = load_model(model_name)  # 500MB model
    return model.predict(ctx.vertex)

@procedure.agent  # Autonomous execution
async def monitoring_agent(ctx):
    while True:
        await check_data_quality()
        await asyncio.sleep(60)
```

### Benefits

**Best of Both Worlds:**
- ✅ Low latency for trusted, simple procedures
- ✅ Isolation for untrusted or complex procedures
- ✅ Gradual migration path
- ✅ Flexibility in deployment

**Recommended for:**
- Production platforms with mixed workloads
- Multi-tenant environments
- Phased rollout strategy

---

## Comparison Matrix

| Aspect | Enhanced MAGE | Deep Embedded | Microservices | Hybrid |
|--------|--------------|---------------|---------------|--------|
| **Implementation Effort** | Medium (3-6 mo) | Very High (12-24 mo) | High (6-12 mo) | High (6-12 mo) |
| **Performance** | Excellent (in-process) | Good (JIT helps) | Poor (network) | Excellent/Good |
| **Isolation** | None | None | Excellent | Good |
| **Scalability** | Limited (single node) | Limited | Excellent | Good |
| **Compatibility** | High | Low | Medium | Medium |
| **Complexity** | Low | Very High | High | Medium-High |
| **Use Case Fit** | General-purpose | Data science | Multi-tenant | Enterprise |
| **Risk** | Low | High | Medium | Medium |

---

## Recommendations

### For RAG Platform (Primary Use Case)

**Recommended: Enhanced MAGE Model (Option 1) + Selective Hybrid (Option 4)**

**Rationale:**
1. **RAG Requirements:**
   - Vector similarity search (already supported via MAGE modules)
   - LLM integration (Python procedures calling OpenAI, Anthropic APIs)
   - Embedding generation (Python + HuggingFace/OpenAI)
   - Document chunking and preprocessing (Python text processing)
   - Context retrieval (Cypher queries with Python post-processing)

2. **Phase 1 (Months 1-3): Enhanced MAGE**
   - Add Python REPL support for interactive exploration
   - Enhance triggers with Python callbacks for RAG pipeline orchestration
   - Transaction hooks for embedding updates and cache invalidation
   - Custom operators for vector operations

3. **Phase 2 (Months 4-6): Selective Hybrid**
   - Sandbox manager for untrusted code (user-provided RAG customizations)
   - Remote execution for heavy ML inference (large embedding models)
   - Agent runtime for autonomous RAG pipeline maintenance

**Example RAG Workflow:**
```python
# Stored procedure for RAG retrieval
@mgp.read_proc
def rag_retrieve(ctx: mgp.ProcCtx,
                 query: str,
                 top_k: int = 5) -> mgp.Record(context=str, score=float):
    # Generate query embedding (calls remote service if large model)
    embedding = embedding_service.encode(query)

    # Vector similarity search (uses Memgraph indices)
    results = ctx.graph.query("""
        MATCH (d:Document)
        WHERE d.embedding IS NOT NULL
        WITH d, vector.cosine_similarity(d.embedding, $embedding) as score
        ORDER BY score DESC
        LIMIT $top_k
        RETURN d.content as context, score
    """, embedding=embedding, top_k=top_k)

    for row in results:
        yield mgp.Record(context=row['context'], score=row['score'])

# Trigger for automatic embedding generation
@mgp.on_create_vertex
def auto_embed(ctx: mgp.TriggerContext):
    for vertex in ctx.created_vertices:
        if 'Document' in vertex.labels and 'content' in vertex.properties:
            # Async embedding generation
            embedding_job_queue.enqueue(vertex.id, vertex.properties['content'])
```

### Implementation Roadmap

#### Q1 2026: Foundation
- [ ] Python REPL session management
- [ ] Enhanced trigger system with Python callbacks
- [ ] Transaction lifecycle hooks
- [ ] Documentation and examples

#### Q2 2026: Performance & Scale
- [ ] Custom operators (PYTHON FILTER, PYTHON MAP)
- [ ] Query optimization for Python expressions
- [ ] Benchmark and optimize GIL overhead
- [ ] Connection pooling for remote execution

#### Q3 2026: Isolation & Security
- [ ] Sandbox manager implementation
- [ ] Remote execution service (basic)
- [ ] Resource quota enforcement
- [ ] Security audit and hardening

#### Q4 2026: Agents & Autonomy
- [ ] Agent runtime framework
- [ ] Event-driven agent activation
- [ ] Goal-based agent orchestration
- [ ] Monitoring and observability

---

## Appendix: Key Implementation Files

### Core Python Integration
- `/home/user/memgraph/src/query/procedure/py_module.{hpp,cpp}` - Python module system
- `/home/user/memgraph/src/py/py.hpp` - Python C API wrappers
- `/home/user/memgraph/include/mgp.py` - User-facing Python API

### Extension Points
- `/home/user/memgraph/src/query/trigger.{hpp,cpp}` - Trigger system
- `/home/user/memgraph/src/query/interpreter.{hpp,cpp}` - Transaction management
- `/home/user/memgraph/src/query/plan/operator.{hpp,cpp}` - Query operators

### APIs
- `/home/user/memgraph/include/mg_procedure.h` - C procedure API
- `/home/user/memgraph/include/mgp.hpp` - C++ procedure API

---

## Conclusion

Memgraph's existing MAGE architecture provides a solid foundation for Python runtime integration. For RAG platform use cases, an evolutionary approach (Enhanced MAGE + Selective Hybrid) offers the best balance of:
- **Speed to market** (leverage existing infrastructure)
- **Performance** (in-process for hot paths)
- **Safety** (remote execution for untrusted code)
- **Flexibility** (support diverse RAG workflows)

The revolutionary approaches (Deep Embedded, Pure Microservices) are better suited for greenfield projects or dramatically different use cases than the current Memgraph architecture supports.
