# Developer Experience & API Design (SPECULATION)

**Date:** 2025-11-08
**Status:** Speculative - Best Practices Based
**Purpose:** SDK design, onboarding flow, and migration path for developers

---

## ⚠️ Design Philosophy

This API design is based on **best practices** from successful developer tools:
- **Anthropic Claude API**: Clean, intuitive, well-documented
- **LangChain**: Flexible, extensible, Python-first
- **Supabase**: Fast time-to-value, great DX
- **Stripe**: Excellent docs, clear error messages

**Goal:** Developer goes from `pip install` → first agent recommendation in **< 30 minutes**.

---

## Quick Start: The "Hello World" Experience

### Install (< 1 minute)

```bash
pip install memgraph-agents
```

### Connect (< 2 minutes)

```python
from memgraph_agents import MemgraphAgents

# Option 1: Cloud (easiest)
mg = MemgraphAgents(
    cloud_url="https://api.memgraph.cloud",
    api_key="mg_live_..."  # from dashboard
)

# Option 2: Self-hosted
mg = MemgraphAgents(
    host="localhost",
    port=7687,
    username="memgraph",
    password="memgraph"
)
```

### First Agent in 3 Lines (< 5 minutes)

```python
# Enable LFTPAgent (Learn From The Past)
mg.agents.enable("lftp")

# Observe an interaction
mg.agents.observe({
    "user_id": "dev_123",
    "tool": "python",
    "args": ["script.py"],
    "context": {"os": "macos", "project": "data_pipeline"}
})

# Observe a correction
mg.agents.observe({
    "user_id": "dev_123",
    "correction": {
        "from_tool": "python",
        "to_tool": "uv",
        "reason": "user prefers uv"
    }
})

# After 2-3 corrections, get recommendations
recs = mg.agents.recommend({
    "user_id": "dev_123",
    "tool_about_to_use": "python"
})

print(recs)
# [
#   {
#     "recommendation": "use 'uv' instead of 'python'",
#     "confidence": 0.85,
#     "reasoning": "User corrected this 3 times in past week"
#   }
# ]
```

**Time to value: < 10 minutes** (including reading docs)

---

## Python SDK Design

### Core API: `MemgraphAgents`

```python
class MemgraphAgents:
    """
    Main entry point for Memgraph Agent Intelligence platform
    """

    def __init__(
        self,
        cloud_url: Optional[str] = None,  # Cloud: https://api.memgraph.cloud
        api_key: Optional[str] = None,    # Cloud API key
        host: str = "localhost",           # Self-hosted
        port: int = 7687,
        username: str = "memgraph",
        password: str = "memgraph",
        **kwargs
    ):
        """
        Connect to Memgraph instance

        Args:
            cloud_url: Cloud URL (mutually exclusive with host/port)
            api_key: Cloud API key
            host: Self-hosted hostname
            port: Self-hosted port
            username: Username for auth
            password: Password for auth
        """
        pass

    # Agent management
    @property
    def agents(self) -> AgentManager:
        """Access to agent management"""
        return self._agent_manager

    # Direct graph access (for advanced users)
    @property
    def graph(self) -> GraphClient:
        """Direct Cypher query access"""
        return self._graph_client

    # Observability
    @property
    def metrics(self) -> MetricsClient:
        """Usage metrics and analytics"""
        return self._metrics_client
```

### Agent Management API

```python
class AgentManager:
    """
    Manage and configure agents
    """

    def enable(
        self,
        agent_type: str,  # "lftp", "performance", "quality", etc.
        config: Optional[Dict] = None
    ) -> Agent:
        """
        Enable an agent

        Args:
            agent_type: Type of agent ("lftp", "performance", "quality", etc.)
            config: Optional configuration

        Returns:
            Agent instance

        Example:
            >>> mg.agents.enable("lftp", config={
            ...     "min_observations": 2,  # Minimum corrections before recommendation
            ...     "confidence_threshold": 0.7,
            ...     "learning_window_days": 30
            ... })
        """
        pass

    def disable(self, agent_type: str):
        """Disable an agent"""
        pass

    def list(self) -> List[Agent]:
        """List all enabled agents"""
        pass

    def get(self, agent_type: str) -> Agent:
        """Get specific agent instance"""
        pass

    def observe(
        self,
        event: Dict[str, Any],
        user_id: Optional[str] = None,
        context: Optional[Dict] = None
    ):
        """
        Observe an event (interaction, correction, etc.)

        This is the main way to feed data to agents.

        Args:
            event: Event data (format depends on event type)
            user_id: Optional user ID (for per-user learning)
            context: Optional context (OS, project, environment, etc.)

        Example - Tool usage:
            >>> mg.agents.observe({
            ...     "type": "tool_call",
            ...     "tool": "grep",
            ...     "args": ["pattern", "file.txt"],
            ...     "duration_ms": 45,
            ...     "success": True
            ... }, user_id="dev_123")

        Example - User correction:
            >>> mg.agents.observe({
            ...     "type": "correction",
            ...     "from_tool": "grep",
            ...     "to_tool": "ripgrep",
            ...     "reason": "faster"
            ... }, user_id="dev_123")

        Example - Error:
            >>> mg.agents.observe({
            ...     "type": "error",
            ...     "error_type": "TypeError",
            ...     "message": "...",
            ...     "stacktrace": "..."
            ... }, user_id="dev_123")
        """
        pass

    def recommend(
        self,
        context: Dict[str, Any],
        user_id: Optional[str] = None,
        min_confidence: float = 0.5
    ) -> List[Recommendation]:
        """
        Get recommendations from all enabled agents

        Args:
            context: Current context (tool about to use, query, etc.)
            user_id: Optional user ID
            min_confidence: Minimum confidence threshold (0.0-1.0)

        Returns:
            List of recommendations sorted by confidence

        Example:
            >>> recs = mg.agents.recommend({
            ...     "tool_about_to_use": "python",
            ...     "file_type": "script"
            ... }, user_id="dev_123", min_confidence=0.7)
            >>>
            >>> for rec in recs:
            ...     print(f"{rec.recommendation} (confidence: {rec.confidence})")
        """
        pass
```

### Agent-Specific APIs

```python
class Agent:
    """Base agent class"""

    @property
    def type(self) -> str:
        """Agent type (e.g., 'lftp', 'performance')"""
        pass

    @property
    def config(self) -> Dict[str, Any]:
        """Current configuration"""
        pass

    def update_config(self, config: Dict[str, Any]):
        """Update agent configuration"""
        pass

    def get_insights(self, user_id: Optional[str] = None) -> List[Insight]:
        """
        Get current insights learned by this agent

        Example:
            >>> agent = mg.agents.get("lftp")
            >>> insights = agent.get_insights(user_id="dev_123")
            >>> for insight in insights:
            ...     print(f"Pattern: {insight.pattern}, Confidence: {insight.confidence}")
        """
        pass

    def clear_insights(self, user_id: Optional[str] = None):
        """
        Clear learned insights (useful for testing or reset)
        """
        pass


class LFTPAgent(Agent):
    """
    Learn From The Past Agent
    Specific methods for LFTP agent
    """

    def get_patterns(self, user_id: str) -> List[Pattern]:
        """
        Get detected patterns for a user

        Returns:
            List of patterns with observation counts and confidence

        Example:
            >>> patterns = lftp.get_patterns("dev_123")
            >>> for p in patterns:
            ...     print(f"{p.from_tool} → {p.to_tool}: {p.observations} times")
        """
        pass

    def get_hypotheses(self, user_id: str) -> List[Hypothesis]:
        """Get active hypotheses being tested"""
        pass

    def force_learn(
        self,
        user_id: str,
        from_tool: str,
        to_tool: str,
        confidence: float = 1.0
    ):
        """
        Manually inject a learned preference (skip observation period)

        Useful for:
        - Importing preferences from another system
        - Admin overrides
        - Testing

        Example:
            >>> lftp.force_learn(
            ...     user_id="dev_123",
            ...     from_tool="python",
            ...     to_tool="uv",
            ...     confidence=0.9
            ... )
        """
        pass


class PerformanceAgent(Agent):
    """Performance optimization agent"""

    def get_slow_queries(
        self,
        threshold_ms: int = 100,
        limit: int = 10
    ) -> List[SlowQuery]:
        """Get slowest queries detected"""
        pass

    def suggest_indices(self) -> List[IndexSuggestion]:
        """Get index recommendations based on query patterns"""
        pass
```

### Recommendation Object

```python
@dataclass
class Recommendation:
    """
    A recommendation from an agent
    """
    agent_type: str              # Which agent produced this
    recommendation: str          # Human-readable recommendation
    confidence: float            # 0.0 - 1.0
    reasoning: str               # Why this recommendation?
    actions: Optional[List[Action]]  # Structured actions (for automation)
    metadata: Dict[str, Any]     # Additional context

    def to_dict(self) -> Dict:
        """Convert to dictionary"""
        pass

    def __str__(self) -> str:
        """Human-readable format"""
        return f"[{self.agent_type}] {self.recommendation} (confidence: {self.confidence:.2f})\nReasoning: {self.reasoning}"


@dataclass
class Action:
    """
    A structured action that can be automated
    """
    type: str                    # "replace_tool", "add_flag", "change_config", etc.
    parameters: Dict[str, Any]   # Action-specific params

    # Example:
    # Action(
    #     type="replace_tool",
    #     parameters={"from": "python", "to": "uv", "args_unchanged": True}
    # )
```

---

## Advanced: Hooks System (For Integration with Tools like Claude Code)

```python
class HookManager:
    """
    Register hooks to inject recommendations at the right moments
    """

    def register_pre_tool_hook(
        self,
        callback: Callable[[ToolContext], Optional[Recommendation]]
    ):
        """
        Register a hook that fires BEFORE a tool is called

        Use this to inject recommendations just-in-time.

        Args:
            callback: Function that receives ToolContext, returns Recommendation or None

        Example:
            >>> def pre_tool_hook(ctx: ToolContext):
            ...     recs = mg.agents.recommend({
            ...         "tool_about_to_use": ctx.tool,
            ...         "args": ctx.args
            ...     }, user_id=ctx.user_id)
            ...
            ...     if recs:
            ...         return recs[0]  # Return top recommendation
            ...     return None
            >>>
            >>> mg.hooks.register_pre_tool_hook(pre_tool_hook)
        """
        pass

    def register_post_tool_hook(
        self,
        callback: Callable[[ToolResult], None]
    ):
        """
        Register a hook that fires AFTER a tool completes

        Use this to observe outcomes and feed back to agents.

        Example:
            >>> def post_tool_hook(result: ToolResult):
            ...     mg.agents.observe({
            ...         "type": "tool_result",
            ...         "tool": result.tool,
            ...         "success": result.exit_code == 0,
            ...         "duration_ms": result.duration_ms
            ...     }, user_id=result.user_id)
            >>>
            >>> mg.hooks.register_post_tool_hook(post_tool_hook)
        """
        pass

    def unregister_hook(self, hook_id: str):
        """Remove a registered hook"""
        pass

    @property
    def hooks(self) -> HookManager:
        """Access to hook system"""
        return self._hook_manager
```

---

## Observability & Debugging

### Logging and Tracing

```python
# Enable detailed logging
import logging
logging.getLogger("memgraph_agents").setLevel(logging.DEBUG)

# Trace a specific interaction
with mg.trace(trace_id="debug_session_1"):
    mg.agents.observe({...})
    recs = mg.agents.recommend({...})

# Get trace details
trace = mg.get_trace("debug_session_1")
print(trace.events)  # All events in chronological order
print(trace.graph_queries)  # Cypher queries executed
print(trace.duration_ms)  # Total time
```

### Metrics Dashboard

```python
# Get usage metrics
metrics = mg.metrics.get_summary(
    start_date="2024-01-01",
    end_date="2024-01-31"
)

print(f"Observations: {metrics.total_observations}")
print(f"Recommendations: {metrics.total_recommendations}")
print(f"Acceptance rate: {metrics.recommendation_acceptance_rate:.1%}")
print(f"Avg confidence: {metrics.avg_confidence:.2f}")

# Per-agent breakdown
for agent_type, stats in metrics.by_agent.items():
    print(f"{agent_type}: {stats.recommendations} recs, {stats.acceptance_rate:.1%} accepted")
```

### Debugging Recommendations

```python
# Get detailed explanation of WHY a recommendation was made
rec = recs[0]
explanation = mg.agents.explain(rec)

print(explanation.pattern_matched)  # Which pattern triggered this
print(explanation.observations)  # Historical observations that led to this
print(explanation.confidence_breakdown)  # How confidence was calculated
print(explanation.graph_path)  # Cypher query and results

# Visualize (if in Jupyter)
explanation.visualize()  # Shows graph structure
```

---

## Error Handling

### Clear Error Messages

```python
try:
    mg.agents.enable("nonexistent_agent")
except AgentNotFoundError as e:
    print(e)
    # AgentNotFoundError: Agent type 'nonexistent_agent' not found.
    # Available agents: lftp, performance, quality, security, cost
    #
    # Did you mean 'lftp'? (Levenshtein distance: 2)
    #
    # See docs: https://docs.memgraph.com/agents/types

try:
    mg.agents.observe({"invalid": "event"})
except ValidationError as e:
    print(e)
    # ValidationError: Event missing required field 'type'.
    # Expected one of: tool_call, correction, error, query
    #
    # Example:
    #   mg.agents.observe({
    #       "type": "tool_call",
    #       "tool": "grep",
    #       ...
    #   })
    #
    # See docs: https://docs.memgraph.com/agents/api#observe
```

---

## Migration Path for Existing Memgraph Users

### Scenario 1: Existing Memgraph + Custom Cypher Queries

**Before (raw Cypher):**
```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver("bolt://localhost:7687")

with driver.session() as session:
    result = session.run("""
        MATCH (u:User)-[:USES]->(t:Tool)
        WHERE u.id = $user_id
        RETURN t.name, count(*) as usage_count
        ORDER BY usage_count DESC
        LIMIT 5
    """, user_id="dev_123")

    for record in result:
        print(f"{record['t.name']}: {record['usage_count']}")
```

**After (with agents):**
```python
from memgraph_agents import MemgraphAgents

mg = MemgraphAgents(host="localhost", port=7687)

# Option 1: Use agent intelligence
mg.agents.enable("lftp")
mg.agents.observe({
    "type": "tool_call",
    "tool": "grep",
    ...
})
recs = mg.agents.recommend(...)

# Option 2: Still use raw Cypher (backward compatible!)
with mg.graph.session() as session:
    result = session.run("""
        MATCH (u:User)-[:USES]->(t:Tool)
        WHERE u.id = $user_id
        RETURN t.name, count(*) as usage_count
        ORDER BY usage_count DESC
        LIMIT 5
    """, user_id="dev_123")

    for record in result:
        print(f"{record['t.name']}: {record['usage_count']}")
```

**Migration: Gradual adoption**
1. Install `memgraph-agents` (compatible with existing setup)
2. Keep existing Cypher queries working
3. Gradually add agent observations
4. Start getting recommendations alongside existing logic
5. Eventually replace custom queries with agent API

### Scenario 2: Using Memgraph MAGE Procedures

**Before:**
```python
# Using MAGE PageRank
result = session.run("""
    CALL pagerank.get() YIELD node, rank
    WHERE node:Tool
    RETURN node.name, rank
    ORDER BY rank DESC
""")
```

**After (MAGE + Agents):**
```python
# Agents automatically use MAGE algorithms under the hood!
# No need to call PageRank manually

agent = mg.agents.get("lftp")
insights = agent.get_insights()  # Uses PageRank internally for tool importance

# Or still call MAGE directly if you want:
result = mg.graph.run("""
    CALL pagerank.get() YIELD node, rank
    WHERE node:Tool
    RETURN node.name, rank
    ORDER BY rank DESC
""")
```

---

## Configuration & Deployment

### Environment Variables

```bash
# Cloud deployment
export MEMGRAPH_CLOUD_URL="https://api.memgraph.cloud"
export MEMGRAPH_API_KEY="mg_live_..."

# Self-hosted
export MEMGRAPH_HOST="localhost"
export MEMGRAPH_PORT="7687"
export MEMGRAPH_USERNAME="memgraph"
export MEMGRAPH_PASSWORD="memgraph"

# Agent configuration
export MEMGRAPH_AGENTS_ENABLED="lftp,performance,quality"
export MEMGRAPH_LFTP_MIN_OBSERVATIONS="2"
export MEMGRAPH_LFTP_CONFIDENCE_THRESHOLD="0.7"
```

### Configuration File

```yaml
# memgraph_agents.yaml
connection:
  host: localhost
  port: 7687
  username: memgraph
  password: memgraph

agents:
  lftp:
    enabled: true
    min_observations: 2
    confidence_threshold: 0.7
    learning_window_days: 30

  performance:
    enabled: true
    slow_query_threshold_ms: 100

  quality:
    enabled: false  # Disabled for now

logging:
  level: INFO
  format: json

metrics:
  enabled: true
  export_interval_seconds: 60
```

```python
# Load from config file
mg = MemgraphAgents.from_config("memgraph_agents.yaml")
```

---

## Documentation Structure

### Recommended Docs Organization

```
docs.memgraph.com/agents/
├── getting-started/
│   ├── quickstart.md          # 5-minute tutorial
│   ├── installation.md        # pip install + setup
│   ├── concepts.md            # Agents, observations, recommendations
│   └── your-first-agent.md    # End-to-end example
│
├── agents/
│   ├── lftp.md                # Learn From The Past Agent
│   ├── performance.md         # Performance optimization
│   ├── quality.md             # Quality assurance
│   ├── security.md            # Security anomaly detection
│   └── cost.md                # Cost optimization
│
├── api-reference/
│   ├── client.md              # MemgraphAgents class
│   ├── agent-manager.md       # AgentManager API
│   ├── hooks.md               # Hook system
│   ├── metrics.md             # Metrics and observability
│   └── errors.md              # Error handling
│
├── guides/
│   ├── integration-langchain.md
│   ├── integration-crewai.md
│   ├── integration-claude-code.md
│   ├── migration-from-raw-cypher.md
│   ├── debugging.md
│   ├── production-deployment.md
│   └── security-best-practices.md
│
├── examples/
│   ├── claude-code-integration/
│   ├── langchain-agent/
│   ├── notebook-assistant/
│   └── ci-cd-optimization/
│
└── advanced/
    ├── horizontal-learning.md  # Cross-customer intelligence
    ├── custom-agents.md        # Build your own agent
    ├── performance-tuning.md
    └── architecture.md         # How it works internally
```

---

## CLI Tool (Optional but Recommended)

```bash
# Install CLI
pip install memgraph-agents[cli]

# Check connection
memgraph-agents status
# ✅ Connected to Memgraph (localhost:7687)
# ✅ 3 agents enabled (lftp, performance, quality)
# ✅ 1,247 observations in last 7 days
# ✅ 89 recommendations given

# Enable an agent
memgraph-agents enable lftp --min-observations 2

# View insights
memgraph-agents insights --agent lftp --user dev_123
# Pattern: python → uv (confidence: 0.87, observations: 5)
# Pattern: grep → ripgrep (confidence: 0.72, observations: 3)

# Clear insights (for testing)
memgraph-agents clear --agent lftp --user dev_123

# Export data
memgraph-agents export --format json --output insights.json

# Logs
memgraph-agents logs --follow
```

---

## TypeScript SDK (For Frontend/Node.js)

```typescript
import { MemgraphAgents } from '@memgraph/agents';

const mg = new MemgraphAgents({
  cloudUrl: 'https://api.memgraph.cloud',
  apiKey: 'mg_live_...'
});

// Enable agent
await mg.agents.enable('lftp');

// Observe
await mg.agents.observe({
  type: 'tool_call',
  tool: 'grep',
  args: ['pattern', 'file.txt']
}, { userId: 'dev_123' });

// Recommend
const recs = await mg.agents.recommend({
  toolAboutToUse: 'python'
}, { userId: 'dev_123' });

console.log(recs[0].recommendation);
```

---

## Key DX Principles

### 1. **Fast Time to Value**
- Install → first recommendation in < 30 minutes
- Default configs that "just work"
- No complex setup required

### 2. **Progressive Disclosure**
- Simple API for basic use (`observe` + `recommend`)
- Advanced features available but not required (hooks, custom agents)
- Raw Cypher access for power users

### 3. **Excellent Error Messages**
- Clear explanation of what went wrong
- Suggested fixes
- Links to relevant docs
- Did-you-mean suggestions

### 4. **Observable & Debuggable**
- Tracing for every interaction
- Explain WHY a recommendation was made
- Metrics dashboard to monitor agent performance

### 5. **Migration-Friendly**
- Works alongside existing Memgraph usage
- Backward compatible with raw Cypher
- Gradual adoption path

---

## Open Questions (Need User Testing)

1. **API naming:**
   - `observe()` vs `track()` vs `log_event()`?
   - `recommend()` vs `get_recommendations()` vs `suggest()`?

2. **Hook system complexity:**
   - Is callback-based API intuitive?
   - Or should it be decorator-based?

3. **Configuration format:**
   - YAML vs TOML vs Python dict?
   - Env vars vs config file?

4. **Recommendation format:**
   - Human-readable string vs structured actions vs both?

5. **Onboarding:**
   - Is 30 minutes realistic or too ambitious?
   - What's the biggest stumbling block?

**Validation:** Build prototype, test with 5 developers, iterate based on feedback.

---

## Next Steps

1. ✅ **Build minimal SDK** (core `observe` + `recommend` API) - Week 1-2
2. ✅ **Write getting started docs** - Week 2
3. ✅ **User test with 3 developers** - Week 3
4. ✅ **Iterate based on feedback** - Week 4
5. ✅ **Launch beta SDK** - Month 2

**Success Metric:** Developer can complete getting started tutorial in < 30 minutes without external help.
