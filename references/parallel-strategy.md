# Parallel Execution Strategy

## Agent Grouping

### Phase B & C: 3-Agent Parallel Groups

```
Agent 1: Unit + API + Smoke
Agent 2: Integration + E2E
Agent 3: Security + Accessibility + Performance + Load/Stress + Chaos
```

**Grouping rationale**:
- Agent 1: Fast, isolated tests — quick turnaround
- Agent 2: Flow-based tests — share integration context
- Agent 3: Non-functional tests — share quality/security context

### Phase D: 3-Agent Verification

Same grouping as Phase B/C for consistency.

## Agent Tool Usage Pattern

Launch all agents in a **single message** with multiple Agent tool calls:

```
Agent tool call 1: {
  description: "Generate unit/api/smoke tests",
  prompt: "[Phase A context] + [domain-specific instructions]",
  subagent_type: "general-purpose"
}

Agent tool call 2: {
  description: "Generate integration/e2e tests",
  prompt: "[Phase A context] + [domain-specific instructions]",
  subagent_type: "general-purpose"
}

Agent tool call 3: {
  description: "Generate security/a11y/perf/load/chaos tests",
  prompt: "[Phase A context] + [domain-specific instructions]",
  subagent_type: "general-purpose"
}
```

## Context Sharing

Each agent must receive from Phase A:
1. Project stack (language, framework, test tools)
2. Source files to test (paths + key functions/classes)
3. Existing test structure (to avoid duplication)
4. Test framework configuration
5. Quality criteria targets

## File Path Isolation

Agents write to non-overlapping paths — no merge conflicts:

| Agent | Document Path | Code Path |
|-------|--------------|-----------|
| 1 | `tests/doc/*/unit-*`, `api-*`, `smoke-*` | `tests/unit/`, `tests/api/`, `tests/smoke/` |
| 2 | `tests/doc/*/integration-*`, `e2e-*` | `tests/integration/`, `tests/e2e/` |
| 3 | `tests/doc/*/security-*`, `accessibility-*`, `performance-*`, `load-stress-*`, `chaos-*` | `tests/security/`, `tests/a11y/`, `tests/perf/`, `tests/load/`, `tests/chaos/` |

## Selective Execution

When user requests specific domains only:
- Reduce to minimum agents needed
- Single domain → no parallelism, direct execution
- 2-3 domains → 1-2 agents based on grouping
- Maintain same file path conventions regardless

## Error Handling

- If an agent fails, report the failure and continue with others
- Collect partial results from successful agents
- Report failed domains in the Preparation Report
- Allow user to retry failed domains
