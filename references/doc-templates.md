# Test Document Templates

All test documents are written in `tests/doc/`.

## Directory Structure

```
tests/doc/
├── scenarios/
│   ├── unit-scenarios.md
│   ├── api-scenarios.md
│   ├── integration-scenarios.md
│   ├── e2e-scenarios.md
│   ├── security-scenarios.md
│   ├── accessibility-scenarios.md
│   ├── performance-scenarios.md
│   ├── load-stress-scenarios.md
│   ├── smoke-scenarios.md
│   └── chaos-scenarios.md
└── testcases/
    ├── unit-testcases.md
    ├── api-testcases.md
    ├── integration-testcases.md
    ├── e2e-testcases.md
    ├── security-testcases.md
    ├── accessibility-testcases.md
    ├── performance-testcases.md
    ├── load-stress-testcases.md
    ├── smoke-testcases.md
    └── chaos-testcases.md
```

## Scenario Document Format

Use the template at [assets/templates/scenario-template.md](../assets/templates/scenario-template.md).

Key fields per scenario:
- **ID**: `SC-{DOMAIN}-{NNN}` (e.g., SC-UNIT-001)
- **Title**: Brief description
- **Objective**: What this scenario verifies
- **Preconditions**: Required state before execution
- **Steps**: High-level test flow
- **Expected Result**: Success criteria
- **Priority**: Critical / High / Medium / Low

## Test Case Document Format

Use the template at [assets/templates/testcase-template.md](../assets/templates/testcase-template.md).

Key fields per test case:
- **ID**: `TC-{DOMAIN}-{NNN}` (e.g., TC-UNIT-001)
- **Scenario**: Parent scenario ID
- **Title**: Specific test description
- **Preconditions**: Setup requirements
- **Input**: Test data
- **Steps**: Detailed execution steps
- **Expected Output**: Exact expected result
- **Actual Output**: (filled during execution)
- **Status**: Pass / Fail / Skip
- **Priority**: Critical / High / Medium / Low

## Writing Rules

1. One scenario covers one logical behavior
2. Each scenario has 2-5 test cases (happy path + edge cases)
3. Use concrete values in examples, not placeholders
4. Mark security/accessibility test cases with relevant standards (OWASP/WCAG)
5. Include boundary values in every domain's test cases
