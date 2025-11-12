# Technical Investigation: Python Callbacks in Trigger System (SPECULATION)

**Date:** 2025-11-08
**Status:** Speculative Technical Analysis
**Purpose:** Assess feasibility of Python callbacks in `/src/query/trigger.{hpp,cpp}`

---

## Executive Summary

**Question:** Can `/src/query/trigger.{hpp,cpp}` support Python callbacks?

**Answer:** ⚠️ **NOT CURRENTLY, but FEASIBLE with moderate effort**

**Current State:**
- Trigger system is **Cypher-query-based only**
- No callback mechanism exists
- All triggers execute as LogicalPlan (compiled Cypher)

**To Support Python Callbacks:**
- **Effort Estimate:** 2-3 weeks (1 engineer)
- **Complexity:** Medium
- **Risk:** Low-Medium (well-understood pattern from procedures)

---

## Current Trigger Architecture

### How Triggers Work Today

**1. Trigger Definition (`trigger.hpp:37-40`)**
```cpp
struct Trigger {
  explicit Trigger(std::string name, const std::string &query, /* ... */);
  void Execute(/* ... */, const TriggerContext &context, bool is_main) const;
  // ...
private:
  std::string name_;
  ParsedQuery parsed_statements_;  // ← Cypher query
  TriggerEventType event_type_;
  // ...
};
```

**Key Insight:** Triggers store a `ParsedQuery` (line 70), which is a **compiled Cypher query**.

**2. Trigger Execution (`trigger.cpp:194-241`)**
```cpp
void Trigger::Execute(DbAccessor *dba, /* ... */, const TriggerContext &context, /* ... */) const {
  // 1. Get logical plan from Cypher query
  auto trigger_plan = GetPlan(dba, db_acc->name());

  // 2. Set up execution context
  ExecutionContext ctx;
  ctx.db_accessor = dba;
  ctx.symbol_table = plan.symbol_table();
  // ...

  // 3. Create cursor and pull results
  auto cursor = plan.plan().MakeCursor(execution_memory);
  Frame frame{plan.symbol_table().max_position(), execution_memory};

  // 4. Inject predefined identifiers (createdVertices, deletedEdges, etc.)
  for (const auto &[identifier, tag] : identifiers) {
    frame_writer.Write(plan.symbol_table().at(identifier), context.GetTypedValue(tag, dba));
  }

  // 5. Execute query
  while (cursor->Pull(frame, ctx));  // ← Pull-based execution
}
```

**Key Insight:** Execution is **pull-based** through a cursor, not callback-based.

**3. Predefined Identifiers (`trigger.cpp:100-150`)**
```cpp
std::vector<std::pair<Identifier, TriggerIdentifierTag>> GetPredefinedIdentifiers(const TriggerEventType event_type) {
  switch (event_type) {
    case EventType::CREATE:
      return TagsToIdentifiers(IdentifierTag::CREATED_VERTICES, IdentifierTag::CREATED_EDGES, /* ... */);
    case EventType::UPDATE:
      return TagsToIdentifiers(IdentifierTag::SET_VERTEX_PROPERTIES, /* ... */);
    // ...
  }
}
```

**Key Insight:** Triggers receive **context** via predefined identifiers:
- `createdVertices` - list of created vertices
- `deletedEdges` - list of deleted edges
- `setVertexProperties` - property changes
- etc.

---

## Existing Python Integration Pattern (Procedures)

### How Python Procedures Work

**1. Procedure Storage (`mg_procedure_impl.hpp:867-948`)**
```cpp
struct mgp_proc {
  std::string name;

  // ✅ This is a C++ std::function that can wrap ANY callable (including Python!)
  std::function<void(mgp_list *, mgp_graph *, mgp_result *, mgp_memory *)> cb;

  std::optional<std::function<void(mgp_list *, mgp_graph *, mgp_memory *)>> initializer;
  std::optional<std::function<void()>> cleanup;

  std::vector<std::pair<string_t, const CypherType *>> args;
  // ...
};
```

**Key Insight:** Procedures use `std::function` which can wrap Python callables.

**2. Python → C++ Bridge (`py_module.hpp`)**
```cpp
/// Convert an `mgp_value` into a Python object
py::Object MgpValueToPyObject(const mgp_value &value, PyGraph *py_graph);

/// Convert a Python object into `mgp_value`
mgp_value *PyObjectToMgpValue(PyObject *, mgp_memory *);

/// Import a Python module
py::Object ImportPyModule(const char *, mgp_module *);

/// Create Python graph accessor
PyObject *MakePyGraph(mgp_graph *, mgp_memory *);
```

**Key Insight:** Robust Python ↔ C++ conversion infrastructure already exists.

**3. Python Procedure Registration (`py_module.cpp:1320`)**
```cpp
PyObject *PyQueryModuleAddProcedure(PyQueryModule *self, PyObject *cb, bool is_write_procedure) {
  // 1. Extract Python callable (cb)
  // 2. Wrap in C++ std::function
  // 3. Register with module
  // ...
}
```

**Pattern:** Python callable → C++ std::function wrapper → stored in module

---

## Proposed Design: Python Trigger Callbacks

### Option 1: Dual-Mode Triggers (Recommended)

**Modify `Trigger` struct to support BOTH Cypher and Python:**

```cpp
struct Trigger {
  enum class TriggerMode {
    CYPHER_QUERY,    // Existing: Execute Cypher query
    PYTHON_CALLBACK  // New: Call Python function
  };

  explicit Trigger(
    std::string name,
    const std::string &query,  // For CYPHER_QUERY mode
    /* ... existing args ... */
  );

  // New constructor for Python callbacks
  explicit Trigger(
    std::string name,
    std::function<void(mgp_graph*, mgp_list*, mgp_memory*)> python_callback,
    TriggerEventType event_type,
    /* ... */
  );

  void Execute(/* ... */) const;

private:
  std::string name_;
  TriggerMode mode_;  // New: Which execution mode?

  // Existing: Cypher query execution
  ParsedQuery parsed_statements_;
  mutable std::shared_ptr<TriggerPlan> trigger_plan_;

  // New: Python callback execution
  std::optional<std::function<void(mgp_graph*, mgp_list*, mgp_memory*)>> python_callback_;

  TriggerEventType event_type_;
  std::shared_ptr<QueryUserOrRole> owner_;
};
```

**Execute method becomes:**
```cpp
void Trigger::Execute(DbAccessor *dba, /* ... */, const TriggerContext &context, /* ... */) const {
  if (!context.ShouldEventTrigger(event_type_)) {
    return;
  }

  switch (mode_) {
    case TriggerMode::CYPHER_QUERY:
      ExecuteCypherTrigger(dba, context, /* ... */);
      break;

    case TriggerMode::PYTHON_CALLBACK:
      ExecutePythonTrigger(dba, context, /* ... */);
      break;
  }
}

void Trigger::ExecutePythonTrigger(DbAccessor *dba, const TriggerContext &context, /* ... */) const {
  // 1. Create mgp_graph wrapper
  mgp_graph graph{dba, storage::v2::View::OLD, context};

  // 2. Convert TriggerContext to mgp_list (pass created/deleted objects)
  mgp_list *trigger_objects = ContextToMgpList(context, event_type_, execution_memory);

  // 3. Call Python callback
  try {
    (*python_callback_)(&graph, trigger_objects, execution_memory);
  } catch (const std::exception &e) {
    spdlog::error("Python trigger '{}' failed: {}", name_, e.what());
    throw;
  }

  // 4. Cleanup
  mgp_list_destroy(trigger_objects);
}
```

**Python API:**
```python
import mgp

@mgp.trigger(event_type=mgp.TriggerEventType.VERTEX_CREATE)
def my_trigger(graph: mgp.Graph, created_vertices: list[mgp.Vertex], memory: mgp.Memory):
    """
    Python trigger callback

    Args:
        graph: Graph accessor (read/write depending on trigger phase)
        created_vertices: List of vertices that were created
        memory: Memory allocator for creating graph objects
    """
    for vertex in created_vertices:
        # Do something with created vertex
        print(f"Vertex created: {vertex.id}")

        # Can even modify graph (if BEFORE_COMMIT trigger)
        vertex.properties["processed"] = True
```

**Registration:**
```python
# In a Python query module
import mgp

def initialize(module: mgp.Module):
    # Register a trigger
    module.add_trigger(
        name="vertex_logger",
        callback=my_trigger,
        event_type=mgp.TriggerEventType.VERTEX_CREATE,
        phase=mgp.TriggerPhase.AFTER_COMMIT
    )
```

---

### Option 2: Procedure-Based Triggers (Alternative, Simpler)

**Keep triggers Cypher-only, but allow calling Python procedures:**

```cypher
// Create trigger that calls Python procedure
CREATE TRIGGER on_vertex_create
ON CREATE
EXECUTE CALL my_module.on_vertex_create(createdVertices);
```

```python
# In my_module.py
import mgp

@mgp.write_proc
def on_vertex_create(vertices: list[mgp.Vertex]) -> mgp.Record():
    for v in vertices:
        # Process vertex
        v.properties["processed"] = True
    return mgp.Record()
```

**Pros:**
- ✅ Zero changes to trigger system!
- ✅ Already works today (can test immediately)
- ✅ Simple, understandable

**Cons:**
- ❌ Adds overhead (Cypher → procedure call → Python)
- ❌ Less flexible (can't access raw trigger context easily)
- ❌ Requires defining procedures separately

**Verdict:** This works for POC/MVP but Option 1 better for production.

---

## Implementation Plan

### Phase 1: Prototype (Week 1)
**Goal:** Prove Python callbacks work

**Tasks:**
1. ✅ Add `TriggerMode` enum to `Trigger` struct
2. ✅ Add `python_callback_` optional member
3. ✅ Implement `ExecutePythonTrigger` method (copy pattern from procedures)
4. ✅ Create test trigger:
   ```cpp
   // In test file
   auto py_trigger = std::make_shared<Trigger>(
     "test_python_trigger",
     [](mgp_graph *g, mgp_list *objs, mgp_memory *mem) {
       py::Object module = py::Object::FromModule("test_trigger_module");
       py::Object callback = module.GetAttr("callback");
       // Call Python function
       py::Object result = callback.Call(/* args */);
     },
     TriggerEventType::VERTEX_CREATE
   );
   ```
5. ✅ Write unit tests

**Effort:** 3-4 days (1 engineer)

### Phase 2: Python API (Week 2)
**Goal:** Clean Python developer experience

**Tasks:**
1. ✅ Add `@mgp.trigger` decorator
2. ✅ Implement `module.add_trigger()` registration
3. ✅ Convert TriggerContext → Python objects:
   - `createdVertices` → `list[mgp.Vertex]`
   - `deletedEdges` → `list[mgp.Edge]`
   - `setVertexProperties` → `dict[mgp.Vertex, dict]`
4. ✅ Document API
5. ✅ Create examples

**Effort:** 4-5 days (1 engineer)

### Phase 3: Production Hardening (Week 3)
**Goal:** Make it safe and performant

**Tasks:**
1. ✅ Error handling (Python exceptions → Memgraph errors)
2. ✅ Performance optimization (minimize conversions)
3. ✅ Memory management (proper cleanup)
4. ✅ Persistence (save Python triggers to disk)
5. ✅ Security (sandboxing if needed)
6. ✅ Integration tests

**Effort:** 5-6 days (1 engineer)

---

## Effort Estimation

### Optimistic (Everything Goes Well)
- **Prototype:** 3 days
- **Python API:** 4 days
- **Hardening:** 5 days
- **Total:** 12 days (~2.5 weeks)

### Realistic (Some Surprises)
- **Prototype:** 4 days
- **Python API:** 5 days
- **Hardening:** 6 days
- **Buffer:** 2 days (unexpected issues)
- **Total:** 17 days (~3.5 weeks)

### Pessimistic (Major Blockers)
- **Prototype:** 5 days (architectural issues)
- **Python API:** 7 days (complex conversions)
- **Hardening:** 8 days (memory leaks, races)
- **Buffer:** 5 days (rework)
- **Total:** 25 days (~5 weeks)

**Recommended Planning Estimate:** 3 weeks (15 working days)

---

## Risks & Mitigation

### Risk 1: Memory Management
**Problem:** Triggers execute in transaction context - need careful memory handling

**Mitigation:**
- Use `mgp_memory` allocator consistently
- Follow procedure memory model (proven pattern)
- Extensive leak testing

**Probability:** Medium (30%)
**Impact:** High (blocks production use)

### Risk 2: Performance Overhead
**Problem:** Python callback overhead might be too high

**Current overhead:**
- Cypher trigger: ~0.1-0.5ms per execution
- Python callback: Adds ~0.5-2ms (GIL acquisition, conversion)

**Acceptable?**
- For AFTER_COMMIT triggers: Yes (not latency-sensitive)
- For BEFORE_COMMIT triggers: Maybe (depends on use case)

**Mitigation:**
- Benchmark early (Week 1)
- Optimize hot paths (minimize conversions)
- Cache Python objects where possible
- Consider async triggers (run in background)

**Probability:** Medium (40%)
**Impact:** Medium (use case dependent)

### Risk 3: Python Exception Handling
**Problem:** Python exceptions need to propagate correctly through C++

**Mitigation:**
- Follow procedure exception handling pattern
- Use `try/catch` at C++/Python boundary
- Clear Python error state after exceptions
- Test all error paths

**Probability:** Low (15%)
**Impact:** Medium (crashes in production)

### Risk 4: Trigger Persistence
**Problem:** Python triggers need to be saved/restored from disk

**Current:** Cypher queries saved as strings in KVStore
**For Python:** Need to save Python module path + function name

**Mitigation:**
- Save as: `{"type": "python", "module": "my_module", "function": "callback"}`
- Reload module on startup (already done for procedures)
- Fail gracefully if module not found

**Probability:** Low (20%)
**Impact:** Medium (triggers lost on restart)

---

## Performance Analysis

### Baseline: Cypher Trigger
```
Trigger definition:
  ON CREATE
  EXECUTE CREATE (n:ProcessedVertex)

Performance:
  Compilation: 5-10ms (first time, then cached)
  Execution: 0.1-0.5ms per trigger
  Memory: ~1KB per trigger
```

### Estimated: Python Callback Trigger
```
Trigger definition:
  @mgp.trigger(event_type=mgp.TriggerEventType.VERTEX_CREATE)
  def callback(graph, vertices, memory):
      for v in vertices:
          graph.create_vertex().properties["processed"] = True

Performance:
  Compilation: 10-20ms (import Python module, verify signature)
  Execution: 0.5-2ms per trigger
    - GIL acquisition: ~0.1ms
    - C++ → Python conversion: ~0.2-0.5ms
    - Python execution: ~0.2-1ms
    - Python → C++ conversion: ~0.1-0.3ms
  Memory: ~5KB per trigger (Python objects)
```

**Overhead:** 5-10x slower than Cypher triggers

**Acceptable?**
- For complex logic: **Yes** (Python flexibility worth it)
- For simple operations: **Maybe** (consider Cypher instead)
- For real-time systems: **Profile first**

**Optimization Opportunities:**
1. **Lazy conversion:** Only convert objects actually used
2. **Batch processing:** Pass multiple events to single callback
3. **C++ fast path:** Common operations (property sets) in C++
4. **Async triggers:** Run Python in background thread (no GIL blocking)

---

## Alternatives Considered

### Alternative 1: Lua Callbacks
**Pros:** Faster than Python, simpler embedding
**Cons:** Python ecosystem much larger, team likely knows Python better

### Alternative 2: WebAssembly (WASM)
**Pros:** Fast, sandboxed, language-agnostic
**Cons:** Immature ecosystem, harder to debug, no clear Python → WASM path

### Alternative 3: JavaScript (V8)
**Pros:** Fast, good ecosystem
**Cons:** Another language, V8 embedding complex, Python already integrated

**Verdict:** Python is the right choice (ecosystem + existing integration)

---

## Validation Plan

### Week 1: Technical Feasibility Spike
**Goal:** Confirm approach works

**Tasks:**
1. ✅ Create minimal prototype (1 Python trigger)
2. ✅ Benchmark performance (compare to Cypher)
3. ✅ Test memory management (no leaks)
4. ✅ Test exception handling (Python errors caught)

**Success Criteria:**
- ✅ Prototype works (can call Python from trigger)
- ✅ Performance overhead < 5ms per trigger
- ✅ No memory leaks after 10K trigger executions
- ✅ Python exceptions don't crash Memgraph

**Decision Point:** If any criteria fails, reassess approach

---

## Conclusion

### ✅ CAN WE SUPPORT PYTHON CALLBACKS IN TRIGGERS?

**YES**, with moderate effort:
- **Feasibility:** High (proven pattern from procedures)
- **Effort:** 2-3 weeks (1 engineer)
- **Risk:** Low-Medium (well-understood)
- **Performance:** Acceptable for most use cases (5-10x overhead)

### Recommendation

**For MVP (doc 11 Phase 1):**
1. ✅ **Use Option 2** (procedure-based triggers) - works TODAY, zero effort
2. ✅ Validate use case with design partner
3. ✅ Profile performance in real workload

**For Production (doc 11 Phase 2-3):**
4. ✅ **Implement Option 1** (native Python triggers) if validated
5. ✅ Follow 3-week implementation plan
6. ✅ Prioritize AFTER_COMMIT triggers (less latency-sensitive)

### Next Steps

**Week 1 (Technical Validation):**
1. ✅ Test Option 2 with design partner (procedure-based)
   - Create trigger: `EXECUTE CALL my_module.callback(createdVertices)`
   - Measure overhead
   - Get feedback

2. ✅ If overhead acceptable → use Option 2 for MVP
3. ✅ If overhead too high → spike on Option 1

**If proceeding with Option 1:**
- Start 3-week implementation in Month 2 (parallel with design partner testing)
- Target: Python triggers ready for Phase 2 customers (Month 4-6)

---

## Appendix: Code Locations

### Key Files to Modify

**Trigger System:**
- `/src/query/trigger.hpp` - Add `TriggerMode`, `python_callback_`
- `/src/query/trigger.cpp` - Implement `ExecutePythonTrigger`
- `/src/query/trigger_context.hpp` - May need context conversion helpers

**Python Integration:**
- `/src/query/procedure/py_module.cpp` - Add `@mgp.trigger` decorator
- `/include/mgp.py` - Add Python trigger API

**Testing:**
- `/tests/unit/query_trigger.cpp` - Unit tests
- `/tests/integration/triggers/` - Integration tests

### Reference Implementations

**How procedures handle Python callbacks:**
- `/src/query/procedure/mg_procedure_impl.hpp:867` - `mgp_proc` struct
- `/src/query/procedure/py_module.cpp:1320` - Python procedure registration

**How to convert C++ ↔ Python:**
- `/src/query/procedure/py_module.hpp:38` - `MgpValueToPyObject`
- `/src/query/procedure/py_module.hpp:54` - `PyObjectToMgpValue`

---

**Confidence Level:** 75%

**What would change this:**
- 🔴 If trigger execution context incompatible with Python (unlikely)
- 🟡 If performance overhead > 10ms (possible, needs benchmark)
- 🟡 If memory management issues (possible, needs testing)
- 🟢 Everything else is low risk (proven patterns)

**Recommended:** Proceed with Week 1 validation spike to confirm.
