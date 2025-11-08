# Sandboxed Execution and Autonomous Agents

**Date:** 2025-11-08
**Status:** Design Proposal
**Purpose:** Design secure sandboxing and autonomous agent execution for Memgraph Python runtime

---

## Executive Summary

Current Memgraph Python integration (MAGE) runs Python code **in-process without sandboxing**, which poses security risks for:
- Multi-tenant deployments
- User-provided code execution
- Autonomous agents with elevated privileges
- RAG platforms with untrusted customizations

This document proposes multi-layered sandboxing strategies and autonomous agent architectures suitable for production deployments, inspired by Claude Code's artifact sandboxing and modern agentic platforms.

---

## Current Security Posture

### Existing Protections (Limited)

**What Memgraph Currently Has:**
- ✅ **Memory Limits**: Per-procedure memory tracking and limits
- ✅ **Transaction Isolation**: MVCC ensures read/write isolation
- ✅ **Privilege System**: Procedures can require specific privileges
- ✅ **Cooperative Abort**: Procedures can check `mgp_must_abort()` for timeout

**What Memgraph Currently LACKS:**
- ❌ **Process Isolation**: Python runs in same process as database
- ❌ **CPU/Time Limits**: No hard timeout enforcement
- ❌ **Filesystem Restrictions**: Full filesystem access
- ❌ **Network Restrictions**: Can make arbitrary network calls
- ❌ **System Call Filtering**: No seccomp or similar restrictions

### Security Risks

**Risk Matrix:**

| Risk | Likelihood | Impact | Severity |
|------|-----------|--------|----------|
| Infinite loop DoS | High | High | **CRITICAL** |
| Resource exhaustion | High | High | **CRITICAL** |
| Data exfiltration | Medium | High | **HIGH** |
| Privilege escalation | Low | High | **HIGH** |
| Code injection | Medium | Medium | **MEDIUM** |
| Side-channel attacks | Low | Medium | **LOW** |

---

## Sandboxing Strategy: Defense in Depth

### Level 1: In-Process Sandboxing (Quick Win)

**Objective:** Minimal changes, basic protection

#### 1.1 Resource Limits (Enhanced)

```python
# /home/user/memgraph/src/query/procedure/sandbox.hpp

struct ResourceLimits {
    size_t max_memory_bytes = 256 * 1024 * 1024;  // 256 MB
    std::chrono::milliseconds max_cpu_time{5000};   // 5 seconds
    size_t max_file_descriptors = 10;
    size_t max_threads = 1;
    size_t max_subprocesses = 0;  // No subprocesses
};

class ResourceLimiter {
public:
    explicit ResourceLimiter(const ResourceLimits& limits);

    // Install limits before procedure execution
    void Install();

    // Check and enforce limits during execution
    void CheckLimits();

    // Remove limits after execution
    void Remove();

private:
    void SetCPULimit(std::chrono::milliseconds max_time);
    void SetMemoryLimit(size_t max_bytes);  // Already exists
    void SetFileDescriptorLimit(size_t max_fds);
    void MonitorThreadCreation();
};
```

**Implementation:**
- Location: `/home/user/memgraph/src/query/procedure/sandbox.{hpp,cpp}`
- Uses `setrlimit()` on Linux for CPU, memory, FD limits
- Timer thread for cooperative timeout enforcement
- Hooks into Python thread creation to block multi-threading

**Limitations:**
- ⚠️ Can be bypassed with sufficient effort
- ⚠️ No network isolation
- ⚠️ No filesystem isolation
- ⚠️ Python C extensions can bypass limits

#### 1.2 Restricted Python Environment

```python
# /home/user/memgraph/query_modules/_sandbox.py

"""
Restricted Python environment for untrusted code

Removes dangerous builtins and modules
"""

import sys
from types import ModuleType

# Dangerous builtins to remove
BLOCKED_BUILTINS = {
    'compile', 'eval', 'exec', 'execfile',  # Code execution
    'open', 'file', 'input', 'raw_input',   # File I/O
    '__import__', 'reload',                 # Module loading
    'vars', 'dir', 'globals', 'locals',     # Introspection
}

# Dangerous modules to block
BLOCKED_MODULES = {
    'os', 'sys', 'subprocess', 'multiprocessing',  # System access
    'socket', 'urllib', 'http', 'ftplib',          # Network
    'ctypes', 'cffi', '_ctypes',                   # Foreign function interface
    'importlib', 'imp',                            # Dynamic imports
    'pickle', 'shelve', 'marshal',                 # Serialization (code exec)
}

# Safe modules (allowlist)
ALLOWED_MODULES = {
    'math', 'random', 'datetime', 'collections',
    'itertools', 'functools', 'operator',
    'json', 're', 'string', 'textwrap',
    'numpy', 'pandas', 'scipy',  # Data science
    'mgp',  # Memgraph API
}

class SandboxedPythonEnvironment:
    """Creates a restricted Python environment"""

    def __init__(self, allowed_modules=None):
        self.allowed_modules = allowed_modules or ALLOWED_MODULES
        self.original_builtins = None
        self.original_import = None

    def __enter__(self):
        """Enter sandboxed environment"""
        import builtins

        # Save originals
        self.original_builtins = dict(builtins.__dict__)
        self.original_import = builtins.__import__

        # Remove dangerous builtins
        for name in BLOCKED_BUILTINS:
            if name in builtins.__dict__:
                del builtins.__dict__[name]

        # Install restricted import
        builtins.__import__ = self._restricted_import

        return self

    def __exit__(self, *args):
        """Exit sandboxed environment"""
        import builtins

        # Restore originals
        builtins.__dict__.update(self.original_builtins)
        builtins.__import__ = self.original_import

    def _restricted_import(self, name, *args, **kwargs):
        """Restricted import function"""
        # Check if module is allowed
        if name.split('.')[0] not in self.allowed_modules:
            raise ImportError(f"Import of module '{name}' is not allowed in sandbox")

        # Use original import
        return self.original_import(name, *args, **kwargs)


# Usage in procedure execution
@mgp.read_proc
def untrusted_procedure(ctx: mgp.ProcCtx):
    with SandboxedPythonEnvironment():
        # User code runs here
        # Cannot import os, subprocess, etc.
        import numpy as np  # OK
        # import os  # Would raise ImportError
        pass
```

**Limitations:**
- ⚠️ Can be bypassed via C extensions
- ⚠️ Doesn't prevent resource exhaustion
- ⚠️ Complex to maintain allowlist

#### 1.3 Python Sub-Interpreters (Python 3.12+)

```cpp
// /home/user/memgraph/src/query/procedure/py_subinterpreter.hpp

class PythonSubInterpreter {
public:
    PythonSubInterpreter();
    ~PythonSubInterpreter();

    // Execute code in isolated sub-interpreter
    py::Object Execute(const std::string& code);

    // Call procedure in isolated sub-interpreter
    mgp_value* CallProcedure(const std::string& module_name,
                            const std::string& proc_name,
                            const mgp_list* args);

private:
    PyInterpreterState* interp_state_;
    PyThreadState* thread_state_;
};
```

**Implementation:**
- Uses PEP 554 sub-interpreters (Python 3.12+)
- Each untrusted procedure runs in separate sub-interpreter
- Sub-interpreters share GIL but have separate state
- Can be destroyed/recreated on errors

**Benefits:**
- ✅ Isolated Python state (no cross-contamination)
- ✅ Can destroy sub-interpreter on error
- ✅ No serialization overhead

**Limitations:**
- ⚠️ Still in-process (can crash main process)
- ⚠️ GIL shared (performance impact)
- ⚠️ Requires Python 3.12+ (sub-interpreters stabilized)

### Level 2: Process Isolation (Recommended)

**Objective:** True isolation via separate processes

#### 2.1 Architecture

```
┌───────────────────────────────────────────────────────────┐
│                    Memgraph Main Process                  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │          Sandbox Manager                            │  │
│  │  - Spawns sandbox processes                         │  │
│  │  - Manages lifecycle                                │  │
│  │  - Enforces quotas                                  │  │
│  │  - Monitors health                                  │  │
│  └──────────────────┬──────────────────────────────────┘  │
└─────────────────────┼─────────────────────────────────────┘
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│  Sandbox Process │    │  Sandbox Process │
│  #1 (trusted)    │    │  #2 (untrusted)  │
│                  │    │                  │
│  Python Runtime  │    │  Python Runtime  │
│  + Minimal       │    │  + Hardened      │
│    Restrictions  │    │    Environment   │
│                  │    │                  │
│  Fast execution  │    │  Seccomp filter  │
│  Low overhead    │    │  Resource limits │
│                  │    │  Network blocked │
└──────────────────┘    └──────────────────┘
```

#### 2.2 Sandbox Process Manager

```cpp
// /home/user/memgraph/src/query/procedure/sandbox_manager.hpp

enum class SandboxTrustLevel {
    TRUSTED,      // In-process, minimal restrictions
    UNTRUSTED,    // Separate process, full sandboxing
    ISOLATED,     // Separate process + network namespace
};

struct SandboxConfig {
    SandboxTrustLevel trust_level;
    ResourceLimits limits;
    std::vector<std::string> allowed_network_hosts;
    std::filesystem::path allowed_file_paths;
    bool enable_seccomp;
};

class SandboxProcess {
public:
    SandboxProcess(const SandboxConfig& config);
    ~SandboxProcess();

    // Execute procedure in sandbox
    Future<mgp_value*> ExecuteProcedure(
        const std::string& module_name,
        const std::string& proc_name,
        const mgp_list* args,
        const mgp_graph* graph
    );

    // Health check
    bool IsAlive() const;

    // Terminate sandbox
    void Terminate();

private:
    pid_t pid_;
    int ipc_fd_;  // IPC socket
    SandboxConfig config_;

    void SpawnProcess();
    void ApplySandboxing();
    void SetupSeccomp();
    void SetupNamespaces();
};

class SandboxManager {
public:
    static SandboxManager& Instance();

    // Get or create sandbox for procedure
    std::shared_ptr<SandboxProcess> GetSandbox(const SandboxConfig& config);

    // Pool management
    void WarmupPool(size_t count);
    void CleanupIdle(std::chrono::seconds max_idle);

private:
    std::map<SandboxConfig, std::vector<std::shared_ptr<SandboxProcess>>> pool_;
    utils::RWLock lock_;
};
```

#### 2.3 IPC Protocol

```protobuf
// /home/user/memgraph/src/query/procedure/sandbox.proto

syntax = "proto3";

package memgraph.sandbox;

// Request to execute procedure
message ExecuteRequest {
    string module_name = 1;
    string procedure_name = 2;
    repeated Value arguments = 3;
    GraphSnapshot graph_snapshot = 4;
    ResourceLimits limits = 5;
}

// Response from procedure execution
message ExecuteResponse {
    oneof result {
        ProcedureResult success = 1;
        Error error = 2;
    }
    ExecutionStats stats = 3;
}

message ProcedureResult {
    repeated ResultRow rows = 1;
}

message ResultRow {
    map<string, Value> fields = 1;
}

message Error {
    ErrorCode code = 1;
    string message = 2;
    string traceback = 3;
}

enum ErrorCode {
    UNKNOWN = 0;
    TIMEOUT = 1;
    MEMORY_LIMIT = 2;
    PERMISSION_DENIED = 3;
    RUNTIME_ERROR = 4;
}

message ExecutionStats {
    uint64 cpu_time_us = 1;
    uint64 memory_peak_bytes = 2;
    uint64 wall_time_us = 3;
}

// Graph snapshot (subset of graph for procedure)
message GraphSnapshot {
    repeated Vertex vertices = 1;
    repeated Edge edges = 2;
    map<string, Value> parameters = 3;
}
```

#### 2.4 Seccomp Filtering

```cpp
// /home/user/memgraph/src/query/procedure/seccomp_filter.cpp

#include <linux/seccomp.h>
#include <linux/filter.h>
#include <sys/prctl.h>

void InstallSeccompFilter() {
    // Allow: read, write, close, exit, exit_group, etc. (safe syscalls)
    // Block: execve, fork, clone, socket (except whitelisted), open (restricted)

    struct sock_filter filter[] = {
        // Load syscall number
        BPF_STMT(BPF_LD | BPF_W | BPF_ABS, offsetof(struct seccomp_data, nr)),

        // Allow safe syscalls
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_read, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),

        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),

        // ... more allowed syscalls ...

        // Block dangerous syscalls
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_execve, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL_PROCESS),

        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_fork, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL_PROCESS),

        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_clone, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL_PROCESS),

        // Default: deny
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ERRNO | EPERM),
    };

    struct sock_fprog prog = {
        .len = sizeof(filter) / sizeof(filter[0]),
        .filter = filter,
    };

    // Install filter
    prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
    prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);
}
```

#### 2.5 Namespace Isolation

```cpp
// /home/user/memgraph/src/query/procedure/namespace_setup.cpp

#include <sched.h>
#include <sys/mount.h>

void SetupNamespaces() {
    // Create new namespaces
    if (unshare(CLONE_NEWNET | CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWIPC) != 0) {
        throw std::runtime_error("Failed to create namespaces");
    }

    // Mount tmpfs for /tmp (isolated filesystem)
    mount("tmpfs", "/tmp", "tmpfs", MS_NOSUID | MS_NODEV, "size=100M");

    // Make root filesystem read-only
    mount(nullptr, "/", nullptr, MS_BIND | MS_REMOUNT | MS_RDONLY, nullptr);

    // Only allow specific paths to be writable
    mount("tmpfs", "/tmp/sandbox", "tmpfs", 0, "size=50M");
}
```

**Benefits:**
- ✅ **True Isolation**: Process crash doesn't affect Memgraph
- ✅ **Resource Control**: Hard limits via cgroups
- ✅ **Network Isolation**: Can block or whitelist network access
- ✅ **Filesystem Isolation**: Can mount read-only or tmpfs
- ✅ **Syscall Filtering**: Block dangerous operations

**Trade-offs:**
- ❌ **Latency**: Process spawn overhead (mitigated by pooling)
- ❌ **Serialization**: Must serialize graph data (optimized with shared memory)
- ❌ **Complexity**: More moving parts to manage

### Level 3: Container-Based Sandboxing (Maximum Isolation)

**Objective:** Defense against sophisticated attacks

#### 3.1 gVisor Integration

```yaml
# Sandbox using gVisor (userspace kernel)

apiVersion: v1
kind: Pod
metadata:
  name: python-sandbox
spec:
  runtimeClassName: gvisor  # Use gVisor runtime
  containers:
  - name: python-executor
    image: memgraph/python-sandbox:latest
    resources:
      limits:
        memory: "256Mi"
        cpu: "500m"
      requests:
        memory: "128Mi"
        cpu: "100m"
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
```

**Benefits:**
- ✅ **Maximum Isolation**: Userspace kernel intercepts all syscalls
- ✅ **No Kernel Exposure**: Bugs in Python/C extensions can't exploit kernel
- ✅ **OCI Compatible**: Works with standard container orchestration

**Trade-offs:**
- ❌ **Performance Overhead**: 10-30% slowdown
- ❌ **Deployment Complexity**: Requires container runtime

#### 3.2 WebAssembly Sandboxing

```python
# Future: Run Python in WebAssembly sandbox

# Using Pyodide (Python compiled to WASM)
from pyodide import PyodideRuntime

class WASMSandbox:
    def __init__(self):
        self.runtime = PyodideRuntime()

    async def execute_procedure(self, code: str, args: dict):
        """Execute Python in WASM sandbox"""
        # WASM provides strong sandboxing by design
        # No filesystem, no network (unless explicitly provided)
        return await self.runtime.run_async(code, args)
```

**Benefits:**
- ✅ **Strong Sandboxing**: WASM design principles
- ✅ **Cross-Platform**: Works everywhere
- ✅ **No Process Overhead**: Runs in-process but isolated

**Limitations:**
- ⚠️ **Ecosystem Maturity**: Pyodide still maturing
- ⚠️ **Performance**: Slower than native Python
- ⚠️ **Library Support**: Not all Python packages work

---

## Autonomous Agents Architecture

### Agent Types

#### Type 1: Reactive Agents (Trigger-Based)

**Use Case:** Respond to graph changes

```python
# /home/user/memgraph/python/memgraph_rag/agents/reactive.py

from abc import ABC, abstractmethod
from typing import List

class ReactiveAgent(ABC):
    """Base class for reactive agents"""

    def __init__(self, name: str, graph: Graph):
        self.name = name
        self.graph = graph
        self.enabled = True

    @abstractmethod
    async def on_vertex_created(self, vertex: Vertex):
        """Called when vertex is created"""
        pass

    @abstractmethod
    async def on_vertex_updated(self, vertex: Vertex, changes: dict):
        """Called when vertex is updated"""
        pass

    @abstractmethod
    async def on_edge_created(self, edge: Edge):
        """Called when edge is created"""
        pass


class DataQualityAgent(ReactiveAgent):
    """Ensures data quality constraints"""

    async def on_vertex_created(self, vertex: Vertex):
        """Validate new vertices"""
        if 'User' in vertex.labels:
            # Check email format
            if 'email' in vertex.properties:
                if not self.is_valid_email(vertex.properties['email']):
                    await self.fix_email(vertex)

            # Check required fields
            required = ['name', 'email', 'created_at']
            missing = [f for f in required if f not in vertex.properties]
            if missing:
                await self.populate_defaults(vertex, missing)

    async def fix_email(self, vertex: Vertex):
        """Attempt to fix invalid email"""
        # Try to normalize
        # If can't fix, flag for review
        await self.graph.query("""
            MATCH (v:User {id: $id})
            SET v.needs_review = true,
                v.review_reason = 'Invalid email format'
        """, id=vertex.id)

    async def populate_defaults(self, vertex: Vertex, fields: List[str]):
        """Populate default values"""
        defaults = {
            'created_at': datetime.now(),
            'status': 'active',
        }

        updates = {f: defaults.get(f) for f in fields if f in defaults}

        await self.graph.query("""
            MATCH (v:User {id: $id})
            SET v += $updates
        """, id=vertex.id, updates=updates)


# Registration as Memgraph trigger
async def register_agent(graph: Graph, agent: ReactiveAgent):
    """Register agent as Memgraph trigger"""

    # Create trigger for vertex creation
    await graph.query("""
        CREATE TRIGGER {name}_vertex_create
        ON CREATE VERTEX
        EXECUTE PYTHON FUNCTION "{module}.on_vertex_created"
    """.format(name=agent.name, module=agent.__class__.__module__))

    # Similar for updates, deletes, etc.
```

#### Type 2: Proactive Agents (Goal-Seeking)

**Use Case:** Autonomous tasks to achieve goals

```python
# /home/user/memgraph/python/memgraph_rag/agents/proactive.py

import asyncio
from enum import Enum

class AgentStatus(Enum):
    IDLE = "idle"
    RUNNING = "running"
    PAUSED = "paused"
    FAILED = "failed"
    COMPLETED = "completed"

class ProactiveAgent(ABC):
    """Base class for goal-seeking agents"""

    def __init__(self, name: str, graph: Graph, goal: str):
        self.name = name
        self.graph = graph
        self.goal = goal
        self.status = AgentStatus.IDLE
        self.task = None

    async def start(self):
        """Start agent execution"""
        self.status = AgentStatus.RUNNING
        self.task = asyncio.create_task(self.run())

    async def stop(self):
        """Stop agent execution"""
        self.status = AgentStatus.PAUSED
        if self.task:
            self.task.cancel()

    @abstractmethod
    async def run(self):
        """Main agent loop"""
        pass

    @abstractmethod
    async def evaluate_progress(self) -> float:
        """Evaluate progress toward goal (0.0 to 1.0)"""
        pass


class KnowledgeEnrichmentAgent(ProactiveAgent):
    """
    Autonomously enriches knowledge graph with external data

    Goal: Ensure all entities have complete information
    """

    def __init__(self, graph: Graph):
        super().__init__(
            name="knowledge_enrichment",
            graph=graph,
            goal="Enrich all entities with complete information"
        )
        self.api_quota = 100  # API calls per hour

    async def run(self):
        """Main loop"""
        while self.status == AgentStatus.RUNNING:
            try:
                # Find entities needing enrichment
                incomplete = await self.find_incomplete_entities()

                if not incomplete:
                    await asyncio.sleep(60)  # Check every minute
                    continue

                # Enrich entities
                for entity in incomplete[:self.api_quota]:
                    await self.enrich_entity(entity)

                # Evaluate progress
                progress = await self.evaluate_progress()
                await self.log_progress(progress)

                # Sleep to respect API quotas
                await asyncio.sleep(3600)  # Run every hour

            except Exception as e:
                self.status = AgentStatus.FAILED
                await self.log_error(e)
                raise

    async def find_incomplete_entities(self) -> List[Vertex]:
        """Find entities missing information"""
        results = await self.graph.query("""
            MATCH (e:Entity)
            WHERE e.description IS NULL
               OR e.enriched_at IS NULL
               OR e.enriched_at < datetime() - duration('P7D')
            RETURN e
            LIMIT 100
        """)
        return [r['e'] for r in results]

    async def enrich_entity(self, entity: Vertex):
        """Enrich entity with external data"""
        # Query external API (Wikipedia, Wikidata, etc.)
        external_data = await self.query_external_api(entity.properties['name'])

        # Update entity
        await self.graph.query("""
            MATCH (e:Entity {id: $id})
            SET e.description = $description,
                e.external_links = $links,
                e.enriched_at = datetime(),
                e.enrichment_source = $source
        """,
        id=entity.id,
        description=external_data.get('description'),
        links=external_data.get('links'),
        source=external_data.get('source'))

        # Create related entities
        for related in external_data.get('related_entities', []):
            await self.create_or_link_entity(entity, related)

    async def evaluate_progress(self) -> float:
        """Calculate percentage of entities enriched"""
        result = await self.graph.query("""
            MATCH (e:Entity)
            WITH count(e) as total,
                 count(CASE WHEN e.description IS NOT NULL THEN 1 END) as enriched
            RETURN toFloat(enriched) / toFloat(total) as progress
        """)
        return result[0]['progress']


class EmbeddingMaintenanceAgent(ProactiveAgent):
    """
    Maintains embeddings for all documents and chunks

    Goal: All content has up-to-date embeddings
    """

    async def run(self):
        """Main loop"""
        while self.status == AgentStatus.RUNNING:
            # Find content without embeddings
            unembedded = await self.find_unembedded_content()

            # Batch embed
            if unembedded:
                await self.batch_embed(unembedded)

            # Find outdated embeddings (model changed)
            outdated = await self.find_outdated_embeddings()
            if outdated:
                await self.batch_embed(outdated)

            await asyncio.sleep(300)  # Run every 5 minutes

    async def find_unembedded_content(self) -> List[Vertex]:
        """Find chunks without embeddings"""
        results = await self.graph.query("""
            MATCH (c:Chunk)
            WHERE c.embedding IS NULL
            RETURN c
            LIMIT 1000
        """)
        return [r['c'] for r in results]

    async def batch_embed(self, chunks: List[Vertex], batch_size=100):
        """Embed chunks in batches"""
        from memgraph_rag.embeddings import get_embedder

        embedder = get_embedder()

        for i in range(0, len(chunks), batch_size):
            batch = chunks[i:i+batch_size]
            texts = [c.properties['content'] for c in batch]

            # Generate embeddings
            embeddings = await embedder.encode_batch(texts)

            # Update graph
            for chunk, embedding in zip(batch, embeddings):
                await self.graph.query("""
                    MATCH (c:Chunk {id: $id})
                    SET c.embedding = $embedding,
                        c.embedding_model = $model,
                        c.embedded_at = datetime()
                """,
                id=chunk.id,
                embedding=embedding.tolist(),
                model=embedder.model_name)
```

#### Type 3: Collaborative Agents (Multi-Agent)

**Use Case:** Complex tasks requiring multiple specialized agents

```python
# /home/user/memgraph/python/memgraph_rag/agents/collaborative.py

class AgentCoordinator:
    """Coordinates multiple agents working together"""

    def __init__(self, graph: Graph):
        self.graph = graph
        self.agents: Dict[str, ProactiveAgent] = {}
        self.message_queue = asyncio.Queue()

    async def register_agent(self, agent: ProactiveAgent):
        """Register an agent"""
        self.agents[agent.name] = agent
        await agent.start()

    async def send_message(self, from_agent: str, to_agent: str, message: dict):
        """Inter-agent communication"""
        await self.message_queue.put({
            'from': from_agent,
            'to': to_agent,
            'message': message,
            'timestamp': datetime.now()
        })

    async def process_messages(self):
        """Process inter-agent messages"""
        while True:
            msg = await self.message_queue.get()
            target_agent = self.agents.get(msg['to'])
            if target_agent:
                await target_agent.handle_message(msg['from'], msg['message'])


# Example: RAG Pipeline Agents

class IngestionAgent(ProactiveAgent):
    """Handles document ingestion"""
    async def run(self):
        # Monitor for new documents
        # Trigger chunking, embedding
        pass

class IndexingAgent(ProactiveAgent):
    """Maintains indices and optimizes retrieval"""
    async def run(self):
        # Monitor index health
        # Rebuild indices when needed
        # Optimize similarity computations
        pass

class QualityAgent(ProactiveAgent):
    """Monitors RAG quality"""
    async def run(self):
        # Analyze retrieval quality
        # Detect and fix issues
        # Suggest improvements
        pass

# Coordinator
async def setup_rag_agents(graph: Graph):
    coordinator = AgentCoordinator(graph)

    await coordinator.register_agent(IngestionAgent(graph))
    await coordinator.register_agent(IndexingAgent(graph))
    await coordinator.register_agent(QualityAgent(graph))

    # Start message processing
    await coordinator.process_messages()
```

### Agent Sandboxing

**Key Principle:** Agents have elevated privileges and run continuously, so sandboxing is critical

```python
# Agent security model

class SandboxedAgent(ProactiveAgent):
    """Agent with sandboxing constraints"""

    def __init__(self, name: str, graph: Graph, sandbox_config: SandboxConfig):
        super().__init__(name, graph, "")
        self.sandbox = self._create_sandbox(sandbox_config)

    def _create_sandbox(self, config: SandboxConfig) -> Sandbox:
        """Create sandbox for agent execution"""
        return Sandbox(
            # Resource limits
            max_memory=config.max_memory,
            max_cpu_time=config.max_cpu_time,

            # Network restrictions
            allowed_hosts=config.allowed_network_hosts,

            # Filesystem restrictions
            allowed_paths=["/tmp/agent_workspace"],
            read_only_paths=["/opt/models"],

            # Graph access restrictions
            allowed_labels=config.allowed_labels,
            allowed_operations=config.allowed_operations,  # READ, WRITE, DELETE

            # API quotas
            api_rate_limits=config.api_rate_limits,
        )

    async def run(self):
        """Run agent in sandbox"""
        async with self.sandbox:
            await super().run()
```

---

## Claude Code "Artifacts" Inspiration

### Artifact-Style Sandboxed Apps

**Concept:** Allow users to create interactive applications that query the graph

```python
# /home/user/memgraph/python/memgraph_rag/artifacts.py

class GraphArtifact(ABC):
    """
    Sandboxed interactive application with graph access

    Similar to Claude Code artifacts but backed by Memgraph
    """

    def __init__(self, artifact_id: str, graph: Graph):
        self.artifact_id = artifact_id
        self.graph = graph
        self.sandbox = self._create_sandbox()

    @abstractmethod
    async def render(self) -> dict:
        """Render artifact (returns HTML/JSON/etc.)"""
        pass

    @abstractmethod
    async def handle_interaction(self, event: dict) -> dict:
        """Handle user interaction"""
        pass

    def _create_sandbox(self) -> Sandbox:
        """Create execution sandbox"""
        return Sandbox(
            # Strict limits
            max_memory=64 * 1024 * 1024,  # 64 MB
            max_cpu_time=1000,  # 1 second

            # Read-only graph access
            graph_access=GraphAccess.READ_ONLY,

            # No network
            network_access=NetworkAccess.NONE,

            # Isolated filesystem
            filesystem=FilesystemAccess.ISOLATED,
        )


# Example: Interactive Knowledge Graph Explorer
class KnowledgeGraphExplorer(GraphArtifact):
    """Interactive graph visualization artifact"""

    async def render(self) -> dict:
        """Render graph visualization"""
        # Query graph
        results = await self.graph.query("""
            MATCH (n)-[r]->(m)
            RETURN n, r, m
            LIMIT 100
        """)

        # Convert to visualization format (D3.js, Cytoscape.js, etc.)
        nodes = []
        edges = []

        for result in results:
            nodes.append(self._vertex_to_node(result['n']))
            nodes.append(self._vertex_to_node(result['m']))
            edges.append(self._edge_to_link(result['r']))

        return {
            'type': 'graph_visualization',
            'data': {
                'nodes': nodes,
                'edges': edges
            },
            'layout': 'force_directed'
        }

    async def handle_interaction(self, event: dict) -> dict:
        """Handle user clicks on nodes"""
        if event['type'] == 'node_click':
            node_id = event['node_id']

            # Expand neighbors
            neighbors = await self.graph.query("""
                MATCH (n {id: $id})-[r]-(m)
                RETURN m, r
                LIMIT 20
            """, id=node_id)

            return {
                'action': 'add_nodes',
                'nodes': [self._vertex_to_node(r['m']) for r in neighbors],
                'edges': [self._edge_to_link(r['r']) for r in neighbors]
            }


# Example: RAG Query Interface
class RAGQueryInterface(GraphArtifact):
    """Interactive RAG query interface"""

    async def render(self) -> dict:
        """Render query interface"""
        return {
            'type': 'chat_interface',
            'title': 'Ask Questions About Your Knowledge Base',
            'placeholder': 'Enter your question...'
        }

    async def handle_interaction(self, event: dict) -> dict:
        """Handle user query"""
        if event['type'] == 'message':
            question = event['message']

            # Perform RAG query (sandboxed)
            response = await self._sandboxed_rag_query(question)

            return {
                'type': 'message',
                'content': response['text'],
                'sources': response['sources']
            }

    async def _sandboxed_rag_query(self, question: str) -> dict:
        """Execute RAG query in sandbox"""
        # This runs in the artifact sandbox with strict limits
        from memgraph_rag import MemgraphRAG

        rag = MemgraphRAG(self.graph, config={
            'max_retrieval': 5,
            'max_tokens': 1000,
        })

        response = await rag.query(question)
        return {
            'text': response.text,
            'sources': [c.source for c in response.contexts]
        }
```

---

## Monitoring and Observability

### Sandbox Metrics

```python
# Metrics to track

class SandboxMetrics:
    # Execution metrics
    sandbox_executions_total: Counter
    sandbox_execution_duration_seconds: Histogram
    sandbox_memory_peak_bytes: Histogram
    sandbox_cpu_time_seconds: Histogram

    # Error metrics
    sandbox_errors_total: Counter (by error_type)
    sandbox_timeouts_total: Counter
    sandbox_oom_kills_total: Counter

    # Security metrics
    sandbox_violations_total: Counter (by violation_type)
    sandbox_blocked_syscalls: Counter (by syscall)
    sandbox_blocked_network_attempts: Counter

    # Agent metrics
    agent_runs_total: Counter (by agent_name)
    agent_failures_total: Counter (by agent_name, failure_reason)
    agent_goal_progress: Gauge (by agent_name)
```

### Audit Logging

```python
# Comprehensive audit log

class SandboxAuditLog:
    async def log_execution(self, event: dict):
        """Log all sandbox executions"""
        await self.graph.query("""
            CREATE (e:SandboxExecution {
                id: $id,
                sandbox_id: $sandbox_id,
                procedure_name: $procedure_name,
                started_at: datetime($started_at),
                ended_at: datetime($ended_at),
                status: $status,
                cpu_time_ms: $cpu_time_ms,
                memory_peak_mb: $memory_peak_mb,
                exit_code: $exit_code,
                error_message: $error_message
            })
        """, **event)

    async def log_violation(self, violation: dict):
        """Log security violations"""
        await self.graph.query("""
            CREATE (v:SecurityViolation {
                id: $id,
                sandbox_id: $sandbox_id,
                violation_type: $violation_type,
                blocked_action: $blocked_action,
                timestamp: datetime($timestamp),
                severity: $severity
            })
        """, **violation)
```

---

## Recommendations

### For Production RAG Platform

**Recommended Approach: Hybrid Sandboxing**

1. **Tier 1: Trusted Code (In-Process)**
   - Built-in procedures
   - Vendor-provided modules
   - Thoroughly reviewed custom code
   - Minimal overhead, maximum performance

2. **Tier 2: Untrusted Code (Process Isolation)**
   - User-provided procedures
   - Custom RAG logic
   - Agent code
   - Seccomp + namespaces + resource limits

3. **Tier 3: Interactive Artifacts (Container Isolation)**
   - User-facing interactive apps
   - Maximum isolation via gVisor
   - Strict resource limits

### Implementation Phases

**Phase 1 (Months 1-2): Enhanced In-Process**
- [ ] Implement resource limiter (CPU, memory, FD)
- [ ] Add restricted Python environment
- [ ] Create procedure trust levels
- [ ] Basic monitoring and metrics

**Phase 2 (Months 3-4): Process Isolation**
- [ ] Implement sandbox process manager
- [ ] IPC protocol with protobuf
- [ ] Seccomp filter
- [ ] Process pooling

**Phase 3 (Months 5-6): Agents and Artifacts**
- [ ] Reactive agent framework
- [ ] Proactive agent framework
- [ ] Agent coordinator
- [ ] Artifact sandboxing

**Phase 4 (Months 7+): Advanced Isolation**
- [ ] Namespace isolation
- [ ] gVisor integration (optional)
- [ ] WASM exploration (future)

---

## Conclusion

Sandboxing is **critical** for a production RAG platform with Python runtime integration. The recommended hybrid approach provides:
- ✅ **Performance** where it matters (trusted code in-process)
- ✅ **Security** where it's needed (untrusted code isolated)
- ✅ **Flexibility** for different use cases (multiple tiers)
- ✅ **Scalability** via process pools and containers

Autonomous agents enable powerful RAG workflows but require careful security design. The proposed reactive/proactive/collaborative agent architecture provides flexibility while maintaining control through sandboxing and monitoring.

This design balances **innovation** (enabling powerful Python-based extensions) with **safety** (protecting the system from malicious or buggy code).
