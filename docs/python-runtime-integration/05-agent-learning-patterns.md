# Agent Learning Patterns: Contextual Intelligence via Graph

**Date:** 2025-11-08
**Status:** Design Deep Dive
**Purpose:** Analyze agent learning patterns using Memgraph, focusing on the Learn From The Past Agent (LFTPAgent) pattern

---

## Executive Summary

**Use Case:** Memgraph provides contextual recommendations to primary agents (e.g., Claude Code) by observing interaction patterns, forming hypotheses, and continuously refining them based on outcomes.

**Example:** If a user repeatedly corrects Claude Code to "use `uv` instead of `python3` directly," a background LFTPAgent:
1. **Detects** the repeated pattern (graph pattern matching)
2. **Forms hypothesis** (link `python` tool → `uv` recommendation)
3. **Stores** with confidence score (evolving based on observations)
4. **Injects** recommendation via hook (when `python` tool about to be called)
5. **Observes** outcome (did user accept? did it work?)
6. **Refines** hypothesis (increase/decrease confidence, adjust conditions)

**Result:** Main agent (Claude Code) gets increasingly intelligent "for free" through contextual learning, without changing its core logic.

---

## The Learn From The Past Agent (LFTPAgent) Pattern

### Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│                    Primary Agent (Claude Code)                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐     │
│  │   Perceive   │→ │   Reason     │→ │   Act (Tools)    │     │
│  └──────┬───────┘  └──────────────┘  └────────┬─────────┘     │
│         │                                       │               │
│         │  1. User message                      │ 4. Tool call  │
└─────────┼───────────────────────────────────────┼───────────────┘
          │                                       │
          ↓                                       ↓
┌────────────────────────────────────────────────────────────────┐
│              Memgraph (Agent Intelligence Layer)               │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │           Interaction Graph                              │ │
│  │  User ──says──> Message ──triggers──> Tool ──produces──> │ │
│  │         ↑                    ↓                           │ │
│  │         └────corrects────────┘                           │ │
│  └──────────────────────────────────────────────────────────┘ │
│                         ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │           Pattern Detection (LFTPAgent)                  │ │
│  │  • Finds repeated sequences                              │ │
│  │  • Detects user corrections                              │ │
│  │  • Identifies preferences                                │ │
│  └──────────────────┬───────────────────────────────────────┘ │
│                     ↓                                          │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │         Hypothesis Graph                                 │ │
│  │  Tool ──should_use──> Recommendation {confidence: 0.85}  │ │
│  │        ──context──> Conditions                           │ │
│  └──────────────────┬───────────────────────────────────────┘ │
│                     │                                          │
│                     │ 5. Inject recommendation                │
│                     ↓                                          │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │         Hook System (Recommendation Injection)           │ │
│  │  • Pre-tool hooks                                        │ │
│  │  • Context-aware retrieval                               │ │
│  │  • Confidence-based suggestions                          │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
          ↓                                       ↑
          │ 6. Tool with recommendation           │ 7. Observe outcome
          ↓                                       ↑
┌─────────────────────────────────────────────────────────────────┐
│              LFTPAgent (Background Process)                     │
│  • Runs continuously                                            │
│  • Updates hypotheses based on outcomes                         │
│  • Prunes low-confidence patterns                               │
│  • Generalizes successful patterns                              │
└─────────────────────────────────────────────────────────────────┘
```

### Data Model: Interaction and Learning Graph

```cypher
// ============================================================
// INTERACTION GRAPH (Real-time capture)
// ============================================================

// User-Agent interaction
CREATE (u:User {id: "user_123", name: "Alice"})
CREATE (agent:Agent {id: "claude_code_1", type: "primary"})

// User sends message
CREATE (m1:Message {
    id: "msg_001",
    content: "Write a Python script to analyze CSV data",
    timestamp: datetime("2024-11-01T10:00:00")
})
CREATE (u)-[:SENT]->(m1)-[:TO]->(agent)

// Agent reasons about task
CREATE (reasoning:ReasoningStep {
    id: "reasoning_001",
    agent_id: "claude_code_1",
    thought: "Need to read CSV, use pandas for analysis",
    timestamp: datetime("2024-11-01T10:00:05")
})
CREATE (m1)-[:TRIGGERED]->(reasoning)

// Agent decides to use Python tool
CREATE (tool_call:ToolCall {
    id: "tool_001",
    tool_name: "python",
    command: "python3 -m pip install pandas",
    timestamp: datetime("2024-11-01T10:00:10")
})
CREATE (reasoning)-[:DECIDED_TO_USE]->(tool_call)

// Tool executes
CREATE (tool_result:ToolResult {
    id: "result_001",
    exit_code: 0,
    output: "Successfully installed pandas",
    timestamp: datetime("2024-11-01T10:00:15")
})
CREATE (tool_call)-[:PRODUCED]->(tool_result)

// User CORRECTS agent
CREATE (correction:UserCorrection {
    id: "correction_001",
    content: "Please don't use python3 directly, use uv instead",
    timestamp: datetime("2024-11-01T10:00:30"),
    correction_type: "tool_preference"
})
CREATE (u)-[:CORRECTED]->(tool_call)
CREATE (correction)-[:CORRECTS]->(tool_call)
CREATE (correction)-[:SUGGESTS_ALTERNATIVE {alternative: "uv"}]->(tool_call)

// Agent complies with correction
CREATE (tool_call_2:ToolCall {
    id: "tool_002",
    tool_name: "uv",
    command: "uv pip install pandas",
    timestamp: datetime("2024-11-01T10:01:00"),
    was_correction: true
})
CREATE (correction)-[:RESULTED_IN]->(tool_call_2)

// User is satisfied
CREATE (feedback:UserFeedback {
    id: "feedback_001",
    sentiment: "positive",
    timestamp: datetime("2024-11-01T10:01:30")
})
CREATE (tool_call_2)-[:RECEIVED_FEEDBACK]->(feedback)

// ============================================================
// PATTERN DETECTED (Similar event happens again)
// ============================================================

// Days later, similar situation
CREATE (m2:Message {
    id: "msg_050",
    content: "Install numpy package",
    timestamp: datetime("2024-11-05T14:00:00")
})
CREATE (u)-[:SENT]->(m2)-[:TO]->(agent)

CREATE (tool_call_3:ToolCall {
    id: "tool_050",
    tool_name: "python",
    command: "python3 -m pip install numpy",
    timestamp: datetime("2024-11-05T14:00:10")
})
CREATE (m2)-[:TRIGGERED]->(tool_call_3)

// User corrects AGAIN (pattern detected!)
CREATE (correction_2:UserCorrection {
    id: "correction_002",
    content: "Use uv instead",
    timestamp: datetime("2024-11-05T14:00:20"),
    correction_type: "tool_preference"
})
CREATE (u)-[:CORRECTED]->(tool_call_3)
CREATE (correction_2)-[:CORRECTS]->(tool_call_3)

// ============================================================
// HYPOTHESIS GRAPH (Learned patterns)
// ============================================================

// LFTPAgent creates hypothesis after detecting pattern
CREATE (hypothesis:Hypothesis {
    id: "hyp_001",
    rule: "When using python/pip, suggest uv instead",
    confidence: 0.75,  // Initial confidence based on 2 observations
    created_at: datetime("2024-11-05T14:05:00"),
    created_by: "lftp_agent",
    observations: 2,
    successes: 2,
    failures: 0,
    last_updated: datetime("2024-11-05T14:05:00")
})

// Hypothesis links to tool pattern
CREATE (tool_pattern:ToolPattern {
    id: "pattern_001",
    tool_name: "python",
    command_pattern: ".*pip install.*",
    context: "package_installation"
})
CREATE (hypothesis)-[:DETECTS]->(tool_pattern)

// Hypothesis recommends alternative
CREATE (recommendation:Recommendation {
    id: "rec_001",
    type: "tool_substitution",
    message: "Consider using `uv` instead of `python3 -m pip` for faster package installation",
    alternative_tool: "uv",
    command_template: "uv pip install {package}",
    priority: "medium",
    show_once: false
})
CREATE (hypothesis)-[:RECOMMENDS]->(recommendation)

// Link to user preference (personalization)
CREATE (u)-[:HAS_PREFERENCE]->(recommendation)

// Conditions for when to apply recommendation
CREATE (condition:RecommendationCondition {
    id: "cond_001",
    type: "tool_about_to_be_called",
    tool_name: "python",
    command_contains: ["pip", "install"]
})
CREATE (recommendation)-[:APPLIES_WHEN]->(condition)

// Track hypothesis evolution
CREATE (hypothesis)-[:BASED_ON_OBSERVATION]->(correction)
CREATE (hypothesis)-[:BASED_ON_OBSERVATION]->(correction_2)

// ============================================================
// HYPOTHESIS REFINEMENT (Continuous learning)
// ============================================================

// Third instance: Agent now gets recommendation proactively
CREATE (m3:Message {
    id: "msg_100",
    content: "Install requests library",
    timestamp: datetime("2024-11-10T09:00:00")
})

CREATE (reasoning_2:ReasoningStep {
    id: "reasoning_100",
    thought: "Need to install requests package",
    timestamp: datetime("2024-11-10T09:00:05")
})
CREATE (m3)-[:TRIGGERED]->(reasoning_2)

// Before tool call, recommendation is INJECTED
CREATE (rec_injection:RecommendationInjection {
    id: "injection_001",
    recommendation_id: "rec_001",
    hypothesis_id: "hyp_001",
    confidence: 0.75,
    injected_at: datetime("2024-11-10T09:00:06"),
    accepted: true  // Agent used the recommendation
})
CREATE (reasoning_2)-[:RECEIVED_RECOMMENDATION]->(rec_injection)
CREATE (rec_injection)-[:INJECTED]->(recommendation)

// Agent ACCEPTS recommendation (uses uv directly)
CREATE (tool_call_4:ToolCall {
    id: "tool_100",
    tool_name: "uv",
    command: "uv pip install requests",
    timestamp: datetime("2024-11-10T09:00:10"),
    used_recommendation: true,
    recommendation_id: "rec_001"
})
CREATE (reasoning_2)-[:DECIDED_TO_USE]->(tool_call_4)
CREATE (rec_injection)-[:RESULTED_IN]->(tool_call_4)

// User does NOT correct (success!)
CREATE (feedback_2:UserFeedback {
    id: "feedback_100",
    sentiment: "neutral",  // No correction = implicit approval
    timestamp: datetime("2024-11-10T09:00:30")
})
CREATE (tool_call_4)-[:RECEIVED_FEEDBACK]->(feedback_2)

// LFTPAgent updates hypothesis (increase confidence)
MATCH (h:Hypothesis {id: "hyp_001"})
SET h.confidence = 0.85,  // Increased from 0.75
    h.observations = 3,
    h.successes = 3,
    h.last_updated = datetime("2024-11-10T09:01:00")

// ============================================================
// GENERALIZATION (Learn broader patterns)
// ============================================================

// LFTPAgent notices this is part of broader pattern
CREATE (general_hyp:Hypothesis {
    id: "hyp_002",
    rule: "User prefers uv for all Python package management",
    confidence: 0.70,
    created_at: datetime("2024-11-10T09:05:00"),
    observations: 3,
    generalized_from: "hyp_001"
})
CREATE (general_hyp)-[:GENERALIZES]->(hypothesis)

// Apply to broader tool patterns
CREATE (broad_pattern:ToolPattern {
    tool_name: "python",
    command_pattern: ".*(pip|poetry|conda).*",  // Any package manager
    context: "any_python_package_management"
})
CREATE (general_hyp)-[:DETECTS]->(broad_pattern)

// ============================================================
// CONTEXTUAL CONDITIONS (When NOT to apply)
// ============================================================

// LFTPAgent learns exceptions
CREATE (exception:RecommendationException {
    id: "exception_001",
    condition: "In CI/CD environment, respect existing setup",
    context_pattern: ".*CI=true.*|.*GITHUB_ACTIONS.*",
    reason: "System package manager may be required"
})
CREATE (recommendation)-[:HAS_EXCEPTION]->(exception)

// User working in Docker? Different recommendation
CREATE (context_condition:ContextCondition {
    id: "context_001",
    type: "environment",
    key: "container",
    value: "docker",
    alternative_recommendation: "Use apt/yum for system packages in containers"
})
CREATE (recommendation)-[:CONTEXT_AFFECTS]->(context_condition)
```

### LFTPAgent Implementation

```python
# /home/user/memgraph/python/memgraph_agent/learning/lftp_agent.py

import asyncio
from typing import List, Dict, Optional
from datetime import datetime, timedelta
from memgraph import Graph

class LFTPAgent:
    """
    Learn From The Past Agent

    Observes interaction patterns, forms hypotheses, and provides
    contextual recommendations to primary agents.
    """

    def __init__(self, graph: Graph, user_id: str):
        self.graph = graph
        self.user_id = user_id
        self.running = False

        # Configuration
        self.min_observations = 2  # Minimum pattern occurrences
        self.confidence_threshold = 0.7  # Minimum confidence to recommend
        self.observation_window = timedelta(days=30)  # Look back period
        self.update_interval = 60  # Check for new patterns every 60s

    async def start(self):
        """Start background learning loop"""
        self.running = True
        await asyncio.gather(
            self.pattern_detection_loop(),
            self.hypothesis_refinement_loop(),
            self.hypothesis_cleanup_loop()
        )

    async def stop(self):
        """Stop background learning"""
        self.running = False

    # ========================================================================
    # PATTERN DETECTION
    # ========================================================================

    async def pattern_detection_loop(self):
        """Continuously detect new patterns"""
        while self.running:
            # Detect tool preference patterns
            await self.detect_tool_preferences()

            # Detect workflow patterns
            await self.detect_workflow_patterns()

            # Detect anti-patterns (things user doesn't like)
            await self.detect_anti_patterns()

            # Detect context-dependent preferences
            await self.detect_contextual_preferences()

            await asyncio.sleep(self.update_interval)

    async def detect_tool_preferences(self):
        """
        Detect when user repeatedly corrects tool usage

        Pattern: User corrects tool A → suggests tool B (multiple times)
        """
        results = await self.graph.query("""
            // Find user corrections suggesting alternatives
            MATCH (u:User {id: $user_id})-[:CORRECTED]->(tool_call:ToolCall)
            MATCH (correction:UserCorrection)-[:CORRECTS]->(tool_call)
            MATCH (correction)-[:SUGGESTS_ALTERNATIVE]->(tool_call)
            WHERE correction.timestamp > datetime() - duration($window)

            // Group by tool and alternative
            WITH tool_call.tool_name as original_tool,
                 correction.alternative as suggested_tool,
                 collect(correction) as corrections,
                 count(correction) as correction_count
            WHERE correction_count >= $min_observations

            // Check if hypothesis already exists
            OPTIONAL MATCH (existing:Hypothesis)-[:DETECTS]->(tp:ToolPattern {tool_name: original_tool})
            WHERE existing.created_by = 'lftp_agent'
              AND tp.tool_name = original_tool

            RETURN original_tool,
                   suggested_tool,
                   correction_count,
                   corrections,
                   existing IS NOT NULL as hypothesis_exists,
                   existing.id as existing_hypothesis_id
        """,
        user_id=self.user_id,
        window=self.observation_window.total_seconds(),
        min_observations=self.min_observations)

        for result in results:
            if not result['hypothesis_exists']:
                # Create new hypothesis
                await self.create_tool_preference_hypothesis(
                    original_tool=result['original_tool'],
                    suggested_tool=result['suggested_tool'],
                    corrections=result['corrections'],
                    observation_count=result['correction_count']
                )
            else:
                # Update existing hypothesis
                await self.update_hypothesis_confidence(
                    hypothesis_id=result['existing_hypothesis_id'],
                    new_observations=result['correction_count']
                )

    async def detect_workflow_patterns(self):
        """
        Detect repeated workflow sequences

        Pattern: User frequently does A → B → C
        """
        results = await self.graph.query("""
            // Find sequences of user actions
            MATCH path = (u:User {id: $user_id})-[:SENT]->(m1:Message)
                        -[:TRIGGERED*3..5]->(action)
            WHERE m1.timestamp > datetime() - duration($window)

            // Extract action sequences
            WITH [node in nodes(path) | node.type + ':' + coalesce(node.name, node.tool_name, '')] as sequence

            // Count sequence occurrences
            WITH sequence, count(*) as occurrence_count
            WHERE occurrence_count >= $min_observations

            RETURN sequence, occurrence_count
        """,
        user_id=self.user_id,
        window=self.observation_window.total_seconds(),
        min_observations=self.min_observations)

        for result in results:
            await self.create_workflow_hypothesis(
                sequence=result['sequence'],
                occurrence_count=result['occurrence_count']
            )

    async def detect_anti_patterns(self):
        """
        Detect things user DOESN'T want

        Pattern: Agent does X → User stops/corrects immediately (repeatedly)
        """
        results = await self.graph.query("""
            // Find agent actions that get immediately stopped
            MATCH (agent:Agent)-[:DECIDED_TO_USE]->(action)
            MATCH (u:User {id: $user_id})-[:STOPPED]->(action)
            WHERE action.timestamp > datetime() - duration($window)
              AND duration.between(action.timestamp, u.stopped_at) < duration('PT30S')

            // Group by action type
            WITH action.type as action_type,
                 action.details as action_details,
                 count(*) as stop_count
            WHERE stop_count >= $min_observations

            RETURN action_type, action_details, stop_count
        """,
        user_id=self.user_id,
        window=self.observation_window.total_seconds(),
        min_observations=self.min_observations)

        for result in results:
            await self.create_anti_pattern_hypothesis(
                action_type=result['action_type'],
                details=result['action_details'],
                stop_count=result['stop_count']
            )

    async def detect_contextual_preferences(self):
        """
        Detect context-dependent preferences

        Pattern: In context X, user prefers Y; in context Z, prefers W
        """
        results = await self.graph.query("""
            // Find corrections grouped by context
            MATCH (u:User {id: $user_id})-[:CORRECTED]->(action)
            MATCH (correction:UserCorrection)-[:CORRECTS]->(action)
            MATCH (context:Context)<-[:IN_CONTEXT]-(action)
            WHERE correction.timestamp > datetime() - duration($window)

            WITH context.type as context_type,
                 context.details as context_details,
                 correction.preferred_action as preferred,
                 count(*) as pref_count
            WHERE pref_count >= $min_observations

            RETURN context_type, context_details, preferred, pref_count
        """,
        user_id=self.user_id,
        window=self.observation_window.total_seconds(),
        min_observations=self.min_observations)

        for result in results:
            await self.create_contextual_hypothesis(
                context_type=result['context_type'],
                context_details=result['context_details'],
                preferred_action=result['preferred'],
                observation_count=result['pref_count']
            )

    # ========================================================================
    # HYPOTHESIS CREATION
    # ========================================================================

    async def create_tool_preference_hypothesis(self,
                                               original_tool: str,
                                               suggested_tool: str,
                                               corrections: List[Dict],
                                               observation_count: int):
        """Create hypothesis for tool preference"""

        # Calculate initial confidence based on observations
        initial_confidence = min(0.5 + (observation_count * 0.1), 0.9)

        # Extract command patterns from corrections
        command_patterns = self._extract_command_patterns(corrections)

        await self.graph.query("""
            // Create hypothesis
            CREATE (h:Hypothesis {
                id: randomUUID(),
                rule: $rule,
                confidence: $confidence,
                created_at: datetime(),
                created_by: 'lftp_agent',
                user_id: $user_id,
                observations: $observations,
                successes: $observations,
                failures: 0,
                last_updated: datetime()
            })

            // Link to tool pattern
            CREATE (tp:ToolPattern {
                id: randomUUID(),
                tool_name: $original_tool,
                command_pattern: $command_pattern,
                context: 'general'
            })
            CREATE (h)-[:DETECTS]->(tp)

            // Create recommendation
            CREATE (rec:Recommendation {
                id: randomUUID(),
                type: 'tool_substitution',
                message: $message,
                alternative_tool: $suggested_tool,
                command_template: $command_template,
                priority: 'medium',
                show_once: false
            })
            CREATE (h)-[:RECOMMENDS]->(rec)

            // Link to user
            MATCH (u:User {id: $user_id})
            CREATE (u)-[:HAS_PREFERENCE]->(rec)

            // Condition for when to apply
            CREATE (cond:RecommendationCondition {
                type: 'tool_about_to_be_called',
                tool_name: $original_tool
            })
            CREATE (rec)-[:APPLIES_WHEN]->(cond)

            // Link to observations
            UNWIND $correction_ids as correction_id
            MATCH (correction:UserCorrection {id: correction_id})
            CREATE (h)-[:BASED_ON_OBSERVATION]->(correction)

            RETURN h.id as hypothesis_id
        """,
        rule=f"When using {original_tool}, suggest {suggested_tool} instead",
        confidence=initial_confidence,
        user_id=self.user_id,
        observations=observation_count,
        original_tool=original_tool,
        command_pattern=command_patterns,
        message=f"Consider using `{suggested_tool}` instead of `{original_tool}` based on your preferences",
        suggested_tool=suggested_tool,
        command_template=self._create_command_template(suggested_tool),
        correction_ids=[c['id'] for c in corrections])

        print(f"✨ Created hypothesis: {original_tool} → {suggested_tool} (confidence: {initial_confidence:.2f})")

    # ========================================================================
    # HYPOTHESIS REFINEMENT
    # ========================================================================

    async def hypothesis_refinement_loop(self):
        """Continuously refine hypotheses based on outcomes"""
        while self.running:
            await self.refine_all_hypotheses()
            await asyncio.sleep(self.update_interval)

    async def refine_all_hypotheses(self):
        """Update confidence scores based on recent outcomes"""
        results = await self.graph.query("""
            // Find recent recommendation injections and their outcomes
            MATCH (h:Hypothesis {user_id: $user_id, created_by: 'lftp_agent'})
            MATCH (h)-[:RECOMMENDS]->(rec:Recommendation)
            OPTIONAL MATCH (rec)<-[:INJECTED]-(inj:RecommendationInjection)
            WHERE inj.injected_at > h.last_updated

            // Check outcomes
            OPTIONAL MATCH (inj)-[:RESULTED_IN]->(tool_call:ToolCall)
            OPTIONAL MATCH (tool_call)-[:RECEIVED_FEEDBACK]->(feedback)
            OPTIONAL MATCH (u:User)-[:CORRECTED]->(tool_call)

            WITH h,
                 count(inj) as injection_count,
                 sum(CASE WHEN inj.accepted = true THEN 1 ELSE 0 END) as acceptance_count,
                 sum(CASE WHEN feedback.sentiment = 'positive' THEN 1 ELSE 0 END) as positive_feedback,
                 sum(CASE WHEN u IS NOT NULL THEN 1 ELSE 0 END) as correction_count
            WHERE injection_count > 0

            RETURN h.id as hypothesis_id,
                   h.confidence as current_confidence,
                   h.observations as current_observations,
                   h.successes as current_successes,
                   injection_count,
                   acceptance_count,
                   positive_feedback,
                   correction_count
        """, user_id=self.user_id)

        for result in results:
            # Calculate new confidence
            new_confidence = self._calculate_confidence(
                current_confidence=result['current_confidence'],
                current_successes=result['current_successes'],
                current_observations=result['current_observations'],
                new_acceptances=result['acceptance_count'],
                new_corrections=result['correction_count'],
                new_positive=result['positive_feedback'],
                new_injections=result['injection_count']
            )

            # Update hypothesis
            await self.graph.query("""
                MATCH (h:Hypothesis {id: $hypothesis_id})
                SET h.confidence = $new_confidence,
                    h.observations = h.observations + $new_observations,
                    h.successes = h.successes + $new_successes,
                    h.failures = h.failures + $new_failures,
                    h.last_updated = datetime()
            """,
            hypothesis_id=result['hypothesis_id'],
            new_confidence=new_confidence,
            new_observations=result['injection_count'],
            new_successes=result['acceptance_count'] + result['positive_feedback'],
            new_failures=result['correction_count'])

            print(f"📈 Updated hypothesis {result['hypothesis_id']}: confidence {result['current_confidence']:.2f} → {new_confidence:.2f}")

    def _calculate_confidence(self,
                             current_confidence: float,
                             current_successes: int,
                             current_observations: int,
                             new_acceptances: int,
                             new_corrections: int,
                             new_positive: int,
                             new_injections: int) -> float:
        """
        Bayesian-style confidence update

        Increase confidence: Agent accepted recommendation, user didn't correct
        Decrease confidence: Agent accepted but user corrected
        """
        total_observations = current_observations + new_injections
        total_successes = current_successes + new_acceptances + new_positive - new_corrections

        # Success rate
        success_rate = total_successes / total_observations if total_observations > 0 else 0

        # Adjust confidence (exponential moving average)
        alpha = 0.3  # Learning rate
        new_confidence = (1 - alpha) * current_confidence + alpha * success_rate

        # Clamp between 0.1 and 0.95
        return max(0.1, min(0.95, new_confidence))

    # ========================================================================
    # HYPOTHESIS CLEANUP
    # ========================================================================

    async def hypothesis_cleanup_loop(self):
        """Remove low-confidence or stale hypotheses"""
        while self.running:
            await self.cleanup_hypotheses()
            await asyncio.sleep(3600)  # Run every hour

    async def cleanup_hypotheses(self):
        """Remove hypotheses that are no longer relevant"""
        # Remove low-confidence hypotheses
        await self.graph.query("""
            MATCH (h:Hypothesis {user_id: $user_id, created_by: 'lftp_agent'})
            WHERE h.confidence < 0.3
               OR (h.observations > 10 AND h.successes < 3)

            // Archive before deleting
            CREATE (archived:ArchivedHypothesis)
            SET archived = properties(h)
            SET archived.archived_at = datetime()

            // Delete hypothesis and relationships
            DETACH DELETE h
        """, user_id=self.user_id)

        # Remove stale hypotheses (not updated in 60 days)
        await self.graph.query("""
            MATCH (h:Hypothesis {user_id: $user_id, created_by: 'lftp_agent'})
            WHERE h.last_updated < datetime() - duration('P60D')

            CREATE (archived:ArchivedHypothesis)
            SET archived = properties(h)
            SET archived.archived_at = datetime()

            DETACH DELETE h
        """, user_id=self.user_id)

    # ========================================================================
    # RECOMMENDATION RETRIEVAL (for hook injection)
    # ========================================================================

    async def get_recommendations_for_context(self,
                                             context: Dict) -> List[Dict]:
        """
        Retrieve relevant recommendations based on current context

        Called by hook system when primary agent about to perform action
        """
        results = await self.graph.query("""
            // Match hypotheses with sufficient confidence
            MATCH (h:Hypothesis {user_id: $user_id, created_by: 'lftp_agent'})
            WHERE h.confidence >= $confidence_threshold

            MATCH (h)-[:RECOMMENDS]->(rec:Recommendation)
            MATCH (rec)-[:APPLIES_WHEN]->(cond:RecommendationCondition)

            // Check if condition matches current context
            WHERE cond.type = $context_type
              AND ($tool_name IS NULL OR cond.tool_name = $tool_name)

            // Check for exceptions
            OPTIONAL MATCH (rec)-[:HAS_EXCEPTION]->(exc:RecommendationException)
            WHERE $context_string =~ exc.context_pattern

            // Exclude if exception applies
            WITH rec, h, cond, exc
            WHERE exc IS NULL

            RETURN rec.id as recommendation_id,
                   rec.message as message,
                   rec.alternative_tool as alternative_tool,
                   rec.command_template as command_template,
                   rec.priority as priority,
                   h.confidence as confidence,
                   h.observations as observations
            ORDER BY h.confidence DESC, rec.priority DESC
            LIMIT 3
        """,
        user_id=self.user_id,
        confidence_threshold=self.confidence_threshold,
        context_type=context.get('type'),
        tool_name=context.get('tool_name'),
        context_string=str(context))

        return [dict(r) for r in results]

    # ========================================================================
    # GENERALIZATION
    # ========================================================================

    async def generalize_patterns(self):
        """
        Look for opportunities to generalize specific hypotheses

        Example: "use uv for pip" + "use uv for poetry" → "use uv for all package mgmt"
        """
        results = await self.graph.query("""
            // Find related hypotheses with similar recommendations
            MATCH (h1:Hypothesis {user_id: $user_id})-[:RECOMMENDS]->(rec1:Recommendation)
            MATCH (h2:Hypothesis {user_id: $user_id})-[:RECOMMENDS]->(rec2:Recommendation)
            WHERE h1.id < h2.id  // Avoid duplicates
              AND rec1.alternative_tool = rec2.alternative_tool
              AND h1.confidence > 0.7 AND h2.confidence > 0.7

            MATCH (h1)-[:DETECTS]->(tp1:ToolPattern)
            MATCH (h2)-[:DETECTS]->(tp2:ToolPattern)
            WHERE tp1.tool_name = tp2.tool_name
              AND tp1.context <> tp2.context

            // Check if generalization already exists
            OPTIONAL MATCH (general:Hypothesis)-[:GENERALIZES]->(h1)
            WHERE (general)-[:GENERALIZES]->(h2)

            RETURN h1, h2, tp1, tp2, rec1,
                   general IS NULL as should_generalize
        """, user_id=self.user_id)

        for result in results:
            if result['should_generalize']:
                await self.create_generalized_hypothesis(
                    specific_hypotheses=[result['h1'], result['h2']],
                    tool_patterns=[result['tp1'], result['tp2']],
                    recommendation=result['rec1']
                )

    # ========================================================================
    # HELPER METHODS
    # ========================================================================

    def _extract_command_patterns(self, corrections: List[Dict]) -> str:
        """Extract regex pattern from command examples"""
        # Analyze commands to create pattern
        # For now, simple approach
        return ".*"  # Match all (can be refined)

    def _create_command_template(self, tool: str) -> str:
        """Create command template for tool"""
        templates = {
            "uv": "uv pip install {package}",
            "rye": "rye add {package}",
            # ... more tools
        }
        return templates.get(tool, f"{tool} {{args}}")
```

### Hook System Integration

```python
# /home/user/memgraph/python/memgraph_agent/hooks.py

from typing import Dict, List, Optional, Callable, Any
from memgraph_agent.learning.lftp_agent import LFTPAgent

class AgentHookSystem:
    """
    Hook system for injecting recommendations into agent execution
    """

    def __init__(self, graph: Graph, user_id: str):
        self.graph = graph
        self.user_id = user_id
        self.lftp_agent = LFTPAgent(graph, user_id)

        # Hook registry
        self.pre_tool_hooks: List[Callable] = []
        self.post_tool_hooks: List[Callable] = []
        self.pre_reasoning_hooks: List[Callable] = []

        # Register LFTPAgent hooks
        self.register_pre_tool_hook(self.inject_tool_recommendations)

    async def start(self):
        """Start LFTPAgent background process"""
        await self.lftp_agent.start()

    # ========================================================================
    # HOOK REGISTRATION
    # ========================================================================

    def register_pre_tool_hook(self, hook: Callable):
        """Register hook called before tool execution"""
        self.pre_tool_hooks.append(hook)

    def register_post_tool_hook(self, hook: Callable):
        """Register hook called after tool execution"""
        self.post_tool_hooks.append(hook)

    # ========================================================================
    # HOOK EXECUTION
    # ========================================================================

    async def execute_pre_tool_hooks(self,
                                    tool_name: str,
                                    tool_args: Dict,
                                    context: Dict) -> Dict:
        """
        Execute all pre-tool hooks

        Returns modified tool_args and any recommendations
        """
        result = {
            'tool_name': tool_name,
            'tool_args': tool_args,
            'recommendations': [],
            'modifications': []
        }

        for hook in self.pre_tool_hooks:
            hook_result = await hook(tool_name, tool_args, context)

            # Accumulate recommendations
            if 'recommendations' in hook_result:
                result['recommendations'].extend(hook_result['recommendations'])

            # Apply modifications
            if 'modified_tool_name' in hook_result:
                result['tool_name'] = hook_result['modified_tool_name']
                result['modifications'].append(f"tool: {tool_name} → {hook_result['modified_tool_name']}")

            if 'modified_tool_args' in hook_result:
                result['tool_args'] = hook_result['modified_tool_args']
                result['modifications'].append("args modified")

        return result

    async def execute_post_tool_hooks(self,
                                     tool_name: str,
                                     tool_result: Any,
                                     context: Dict):
        """Execute all post-tool hooks"""
        for hook in self.post_tool_hooks:
            await hook(tool_name, tool_result, context)

    # ========================================================================
    # LFTP RECOMMENDATION INJECTION
    # ========================================================================

    async def inject_tool_recommendations(self,
                                         tool_name: str,
                                         tool_args: Dict,
                                         context: Dict) -> Dict:
        """
        Hook: Inject recommendations from LFTPAgent before tool execution
        """
        # Get relevant recommendations
        recommendations = await self.lftp_agent.get_recommendations_for_context({
            'type': 'tool_about_to_be_called',
            'tool_name': tool_name,
            'tool_args': tool_args,
            **context
        })

        result = {
            'recommendations': []
        }

        for rec in recommendations:
            # Log injection
            injection_id = await self._log_recommendation_injection(
                recommendation_id=rec['recommendation_id'],
                tool_name=tool_name,
                context=context
            )

            # Format recommendation for agent
            formatted_rec = {
                'id': rec['recommendation_id'],
                'injection_id': injection_id,
                'message': rec['message'],
                'confidence': rec['confidence'],
                'observations': rec['observations'],
                'priority': rec['priority'],
                'alternative_tool': rec.get('alternative_tool'),
                'command_template': rec.get('command_template')
            }

            result['recommendations'].append(formatted_rec)

            # Auto-apply high-confidence recommendations?
            if rec['confidence'] > 0.9 and rec.get('alternative_tool'):
                result['modified_tool_name'] = rec['alternative_tool']
                result['auto_applied'] = True

                # Update injection log
                await self._mark_injection_accepted(injection_id)

        return result

    async def _log_recommendation_injection(self,
                                           recommendation_id: str,
                                           tool_name: str,
                                           context: Dict) -> str:
        """Log that recommendation was injected"""
        result = await self.graph.query("""
            MATCH (rec:Recommendation {id: $rec_id})

            CREATE (inj:RecommendationInjection {
                id: randomUUID(),
                recommendation_id: $rec_id,
                tool_name: $tool_name,
                context: $context,
                injected_at: datetime(),
                accepted: false
            })
            CREATE (inj)-[:INJECTED]->(rec)

            RETURN inj.id as injection_id
        """,
        rec_id=recommendation_id,
        tool_name=tool_name,
        context=context)

        return result[0]['injection_id']

    async def _mark_injection_accepted(self, injection_id: str):
        """Mark injection as accepted (agent used recommendation)"""
        await self.graph.query("""
            MATCH (inj:RecommendationInjection {id: $injection_id})
            SET inj.accepted = true,
                inj.accepted_at = datetime()
        """, injection_id=injection_id)


# ============================================================================
# INTEGRATION WITH PRIMARY AGENT (Claude Code example)
# ============================================================================

class ClaudeCodeWithLearning:
    """
    Claude Code agent enhanced with LFTPAgent learning
    """

    def __init__(self, user_id: str):
        self.user_id = user_id
        self.graph = Graph.connect("bolt://localhost:7687")
        self.hook_system = AgentHookSystem(self.graph, user_id)

    async def start(self):
        """Start Claude Code with learning enabled"""
        await self.hook_system.start()
        print("✨ Claude Code with contextual learning enabled!")

    async def execute_tool(self, tool_name: str, tool_args: Dict, context: Dict) -> Any:
        """
        Execute tool with hook system

        1. Pre-tool hooks inject recommendations
        2. Tool execution (possibly modified)
        3. Post-tool hooks observe outcome
        """
        # PRE-TOOL HOOKS
        hook_result = await self.hook_system.execute_pre_tool_hooks(
            tool_name, tool_args, context
        )

        # Show recommendations to agent (or auto-apply)
        if hook_result['recommendations']:
            print(f"\n💡 Recommendations based on your past preferences:")
            for rec in hook_result['recommendations']:
                print(f"   • {rec['message']} (confidence: {rec['confidence']:.0%})")

            if hook_result.get('auto_applied'):
                print(f"   ✓ Auto-applied high-confidence recommendation")

        # Execute tool (possibly modified)
        actual_tool = hook_result['tool_name']
        actual_args = hook_result['tool_args']

        result = await self._execute_tool_internal(actual_tool, actual_args)

        # POST-TOOL HOOKS
        await self.hook_system.execute_post_tool_hooks(
            actual_tool, result, context
        )

        return result

    async def _execute_tool_internal(self, tool_name: str, args: Dict) -> Any:
        """Actual tool execution (implementation specific)"""
        # ... tool execution logic ...
        pass
```

---

## Architectural Implications

### Why This Pattern Favors Specific Architectures

**1. Agent-First Microservices (STRONGLY FAVORED)**

**Why:**
- LFTPAgent runs as **independent background service**
- Doesn't block main agent execution
- Can be scaled independently
- Crash in learning doesn't crash primary agent

**Architecture:**
```
┌──────────────────┐     ┌──────────────────┐
│  Claude Code     │────▶│  Hook Gateway    │
│  (Primary Agent) │◀────│  (Fast, sync)    │
└──────────────────┘     └────────┬─────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────┐
│        Memgraph (Shared Knowledge)           │
│  • Interaction graph                         │
│  • Hypothesis graph                          │
│  • Real-time queries (< 10ms)                │
└────────────┬─────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────┐
│      LFTPAgent (Background Service)          │
│  • Pattern detection (async, 60s interval)   │
│  • Hypothesis refinement                     │
│  • Generalization                            │
│  • Cleanup                                   │
└──────────────────────────────────────────────┘
```

**2. Enhanced MAGE Model (LESS SUITABLE)**

**Why:**
- Background learning needs to run continuously
- Pattern detection is compute-intensive
- Would compete with primary agent for resources
- Harder to isolate failures

**Could work with:**
- Dedicated LFTPAgent procedure that runs on schedule
- But less elegant than microservices

**3. Hybrid Model (GOOD COMPROMISE)**

**Primary agent in-process, LFTPAgent as service**
- Best of both worlds
- Hook injection very fast (in-process)
- Learning decoupled (service)

### Key Architectural Requirements This Pattern Reveals

#### 1. **Real-Time Graph Queries (Low Latency)**

Hook injection must be **< 10ms** to not slow primary agent:
```python
# This must be FAST
recommendations = await graph.query("""
    MATCH (h:Hypothesis {user_id: $user_id})
    WHERE h.confidence > 0.7
    MATCH (h)-[:RECOMMENDS]->(rec)
    RETURN rec
    LIMIT 3
""")
```

**Requirements:**
- Indexed lookups on `user_id`, `confidence`
- Query plan caching
- Connection pooling
- Possibly in-memory cache for top recommendations

#### 2. **Pattern Matching Over Temporal Sequences**

Graph excels at this vs SQL:
```cypher
// Find: User corrected tool A → tool B (repeatedly)
MATCH (u:User)-[:CORRECTED]->(t1:ToolCall)
MATCH (correction)-[:SUGGESTS_ALTERNATIVE]->(t1)
WHERE correction.timestamp > datetime() - duration('P30D')
WITH t1.tool_name as from_tool,
     correction.alternative as to_tool,
     count(*) as occurrences
WHERE occurrences >= 2
RETURN from_tool, to_tool, occurrences
```

**Why graphs win:**
- No complex self-joins
- Natural temporal relationships
- Efficient path traversal

#### 3. **Hypothesis Evolution (Versioning)**

Hypotheses evolve over time:
```cypher
CREATE (h_v2:Hypothesis {version: 2})
SET h_v2 = properties(h_v1)
SET h_v2.confidence = 0.85  // Updated
CREATE (h_v2)-[:EVOLVED_FROM]->(h_v1)
```

**Enables:**
- A/B testing hypotheses
- Rollback if confidence drops
- Analyze what changed

#### 4. **Context-Aware Retrieval**

Recommendations depend on context:
```cypher
MATCH (rec:Recommendation)-[:APPLIES_WHEN]->(cond)
WHERE cond.tool_name = $current_tool
  AND NOT EXISTS {
    MATCH (rec)-[:HAS_EXCEPTION]->(exc)
    WHERE $env_vars =~ exc.context_pattern
  }
RETURN rec
```

**Requires:**
- Flexible condition matching
- Exception handling
- Context propagation

#### 5. **Bayesian Confidence Updates**

Confidence evolves with observations:
```python
new_confidence = (1 - alpha) * old_confidence + alpha * success_rate
```

**Tracked in graph:**
- Observations count
- Success/failure counts
- Temporal decay (recent observations weighted more)

---

## Broader Applicability of This Pattern

### Pattern: Background Intelligence Agents

**General Form:**
1. **Observe** primary agent interactions (via graph)
2. **Detect** patterns (graph queries)
3. **Form** hypotheses (create nodes)
4. **Test** hypotheses (observe outcomes)
5. **Inject** recommendations (via hooks)
6. **Refine** based on feedback (update confidence)

### Other Applications

#### 1. **Performance Optimization Agent**

```cypher
// Detect: Query Q repeatedly slow
MATCH (agent)-[:EXECUTED]->(query:Query)
WHERE query.latency_ms > 1000
WITH query.pattern as slow_pattern, count(*) as occurrences
WHERE occurrences > 10

// Hypothesis: Add index on frequently accessed property
CREATE (h:Hypothesis {
    rule: "Add index on property X for query pattern Y",
    confidence: 0.7
})

// Inject recommendation
CREATE (rec:Recommendation {
    message: "Consider adding index on property X (used in 85% of slow queries)",
    action: "CREATE INDEX ON :Label(property)"
})
```

#### 2. **Security Anomaly Agent**

```cypher
// Detect: Unusual access patterns
MATCH (agent)-[:ACCESSED]->(resource:Resource)
WHERE agent.id = $agent_id
  AND resource.sensitivity = 'high'
WITH agent, count(DISTINCT resource) as unique_accesses,
     collect(resource.type) as access_types
WHERE unique_accesses > usual_pattern + 3 * stddev

// Create alert
CREATE (alert:SecurityAlert {
    message: "Agent accessing unusually high number of sensitive resources",
    confidence: 0.9,
    recommended_action: "Verify agent authorization"
})
```

#### 3. **Cost Optimization Agent**

```cypher
// Detect: Expensive operations
MATCH (agent)-[:USED]->(tool:Tool)-[:INCURRED_COST]->(cost:Cost)
WHERE cost.amount > 1.0
WITH tool.name as expensive_tool,
     sum(cost.amount) as total_cost,
     count(*) as usage_count

// Hypothesis: Cheaper alternative exists
CREATE (h:Hypothesis {
    rule: "Use cheaper tool B instead of expensive tool A",
    estimated_savings: total_cost * 0.7
})

CREATE (rec:Recommendation {
    message: "Tool A costs $X per use. Tool B provides similar results for $Y (70% savings)",
    priority: "high"
})
```

#### 4. **Code Quality Agent**

```cypher
// Detect: Frequent test failures after code changes
MATCH (agent)-[:WROTE]->(code:CodeChange)
MATCH (code)-[:CAUSED]->(test:TestFailure)
WITH code.file_pattern as problematic_pattern,
     count(test) as failure_count

// Hypothesis: Specific file patterns are error-prone
CREATE (h:Hypothesis {
    rule: "Extra caution needed when modifying files matching pattern X",
    confidence: 0.8
})

CREATE (rec:Recommendation {
    message: "Files matching this pattern have 65% test failure rate. Suggest: 1) Write tests first, 2) Request code review",
    type: "preventive"
})
```

#### 5. **Workflow Optimization Agent**

```cypher
// Detect: User frequently does A, then B, then C
MATCH path = (u:User)-[:DID]->(a1:Action)
            -[:THEN]->(a2:Action)
            -[:THEN]->(a3:Action)
WHERE path.occurred_count > 5

// Hypothesis: Workflow can be automated
CREATE (h:Hypothesis {
    rule: "Automate sequence: A → B → C",
    confidence: 0.85
})

CREATE (rec:Recommendation {
    message: "You've done this sequence 12 times. Create a custom command to automate?",
    type: "automation_suggestion",
    estimated_time_savings: "~5 minutes per occurrence"
})
```

---

## Design Principles for Background Intelligence Agents

### 1. **Non-Intrusive**
- Must not slow down primary agent
- Recommendations are suggestions, not commands
- Can be disabled/ignored without breaking functionality

### 2. **Transparent**
- Show confidence scores
- Explain reasoning ("based on 5 similar situations")
- Allow user to see/edit hypotheses

### 3. **Privacy-Preserving**
- Learning is per-user (in their graph instance)
- Can aggregate anonymously if opted-in
- No data leaves user's workspace

### 4. **Continuous Improvement**
- Confidence evolves with more data
- Old hypotheses pruned automatically
- Generalizes patterns when appropriate

### 5. **Fail-Safe**
- If learning agent crashes, primary agent unaffected
- Bad recommendations don't break workflows
- User can override any suggestion

---

## Implementation Roadmap for LFTPAgent Pattern

### Phase 1: Foundation (Weeks 1-2)
- [ ] Interaction graph capture (user messages, tool calls, corrections)
- [ ] Basic pattern detection (repeated corrections)
- [ ] Simple hypothesis creation (tool preferences)
- [ ] Hook system for recommendation injection

### Phase 2: Learning (Weeks 3-4)
- [ ] Confidence calculation and updates
- [ ] Hypothesis refinement based on outcomes
- [ ] Context-aware recommendation retrieval
- [ ] Logging and observability

### Phase 3: Intelligence (Weeks 5-6)
- [ ] Pattern generalization
- [ ] Workflow sequence detection
- [ ] Anti-pattern identification
- [ ] Contextual condition matching

### Phase 4: Production (Weeks 7-8)
- [ ] Performance optimization (query caching, indexing)
- [ ] Hypothesis cleanup and archiving
- [ ] A/B testing framework
- [ ] Privacy controls and opt-out

### Phase 5: Ecosystem (Weeks 9-10)
- [ ] Claude Code integration (reference implementation)
- [ ] LangGraph integration
- [ ] CrewAI integration
- [ ] Documentation and examples

---

## Metrics for LFTPAgent Success

### Learning Effectiveness
- **Pattern Detection Rate**: Patterns detected / total interactions
- **Hypothesis Accuracy**: Accepted recommendations / total recommendations
- **Confidence Calibration**: Actual success rate vs predicted confidence
- **Generalization Success**: Generalized hypotheses that remain valid

### User Value
- **Time Saved**: Fewer corrections needed over time
- **Workflow Efficiency**: Automated repetitive tasks
- **Error Reduction**: Fewer mistakes caught by recommendations
- **User Satisfaction**: Explicit feedback on recommendations

### System Performance
- **Hook Latency**: Time to inject recommendations (target: < 10ms)
- **Background CPU**: LFTPAgent resource usage (target: < 5%)
- **Graph Query Performance**: Pattern detection queries (target: < 100ms)
- **Hypothesis Count**: Active hypotheses per user (target: 10-50)

---

## Conclusion

The **Learn From The Past Agent (LFTPAgent)** pattern demonstrates why **agent-first architecture** and **graph-native storage** are a perfect match:

1. **Graphs naturally represent agent interactions**
   - Tool calls, corrections, preferences as edges
   - Temporal sequences without complex joins
   - Pattern matching via graph traversal

2. **Background learning enhances primary agents**
   - Non-intrusive contextual intelligence
   - Continuous improvement over time
   - Personalized to each user's preferences

3. **This pattern strongly favors microservices architecture**
   - LFTPAgent as independent service
   - Decoupled from primary agent
   - Scales independently

4. **Broadly applicable beyond tool preferences**
   - Performance optimization
   - Security anomaly detection
   - Cost optimization
   - Code quality improvements
   - Workflow automation

**Strategic Implication:** Memgraph positioned as **"The Intelligence Layer for AI Agents"** - not just data storage, but active learning and recommendation system that makes all agents smarter over time.

This is a **unique competitive advantage** that neither Neon (relational model) nor Databricks (batch analytics) can easily replicate. Graph structure + real-time queries + pattern matching = perfect fit for agent learning systems.
