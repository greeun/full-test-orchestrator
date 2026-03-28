# Test Domains Guide

## 1. Unit

**Focus**: Single function/module in isolation
**Mock**: External dependencies (DB, API, filesystem)
**Patterns**:
- Arrange-Act-Assert
- One assertion per logical concept
- Cover: happy path, error path, boundary values, null/undefined

**Scenarios to consider**:
- Input validation (empty, null, negative, overflow)
- Return value correctness
- Error/exception throwing
- State mutations
- Edge cases (empty array, single element, max length)

## 2. API

**Focus**: HTTP endpoint request/response contract
**Verify**:
- Status codes (200, 201, 400, 401, 403, 404, 500)
- Response body schema
- Headers (Content-Type, CORS, auth)
- Query params / path params / body parsing
- Error response format

**Scenarios to consider**:
- CRUD operations
- Authentication required vs public
- Invalid payload / missing fields
- Rate limiting behavior
- Pagination

## 3. Integration

**Focus**: Module-to-module interaction
**Verify**:
- Database read/write roundtrip
- External service communication
- Message queue produce/consume
- Cache hit/miss behavior
- Transaction rollback

**Scenarios to consider**:
- Service A calls Service B with valid data
- Service B is unavailable — fallback behavior
- Data consistency across modules
- Concurrent access patterns

## 4. E2E

**Focus**: Full user journey through the application
**Tools**: Playwright, Cypress, Selenium
**Verify**:
- Critical user flows (login → action → result)
- Navigation and routing
- Form submission and validation
- Data persistence across pages

**Scenarios to consider**:
- Happy path user journey
- Error recovery (network failure, invalid input)
- Multi-step workflows
- Cross-browser behavior (if applicable)

## 5. Security

**Focus**: OWASP Top 10 vulnerability coverage
**Verify**:
- XSS (reflected, stored, DOM-based)
- SQL/NoSQL injection
- CSRF protection
- Authentication bypass
- Authorization escalation
- Sensitive data exposure
- Security headers
- Input sanitization

**Scenarios to consider**:
- Malicious input in all user-facing fields
- JWT/token manipulation
- Path traversal
- Rate limiting / brute force protection
- CORS misconfiguration

## 6. Accessibility

**Focus**: WCAG 2.1 AA compliance
**Tools**: axe-core, pa11y, Lighthouse
**Verify**:
- Color contrast ratios (≥ 4.5:1 normal, ≥ 3:1 large text)
- Keyboard navigation (Tab, Enter, Escape, Arrow keys)
- Screen reader compatibility (ARIA labels, roles, landmarks)
- Focus management
- Alt text for images

**Scenarios to consider**:
- Full keyboard-only navigation
- Screen reader announcement order
- Dynamic content updates (live regions)
- Form error announcement
- Modal/dialog focus trap

## 7. Performance

**Focus**: User experience speed metrics
**Metrics**:
- LCP (Largest Contentful Paint) < 2.5s
- FID (First Input Delay) < 100ms
- CLS (Cumulative Layout Shift) < 0.1
- TTFB (Time to First Byte) < 200ms
- API response time < 200ms

**Scenarios to consider**:
- Initial page load
- Subsequent navigation
- Large data rendering
- Image/asset loading
- Bundle size impact

## 8. Load/Stress

**Focus**: System capacity and breaking point
**Tools**: k6, JMeter, Artillery, Locust
**Metrics**:
- Max concurrent users
- Requests per second (TPS)
- Error rate under load
- Response time degradation curve
- Recovery time after overload

**Scenarios to consider**:
- Ramp-up: gradual increase to target load
- Spike: sudden traffic burst
- Soak: sustained load over time
- Stress: beyond expected capacity
- Resource exhaustion (memory, CPU, connections)

## 9. Smoke

**Focus**: Post-deploy critical path verification
**Characteristics**:
- Fast (< 2 minutes total)
- Covers only critical paths
- Binary pass/fail
- Run after every deployment

**Scenarios to consider**:
- Application starts and responds to health check
- Login flow works
- Core CRUD operation succeeds
- Database connectivity confirmed
- External service connectivity confirmed

## 10. Chaos

**Focus**: Fault injection and system resilience
**Tools**: Chaos Monkey, Litmus, Gremlin, tc (traffic control)
**Inject**:
- Network latency / packet loss
- Service crash / restart
- Database connection failure
- Disk full / memory pressure
- DNS failure

**Scenarios to consider**:
- Downstream service timeout — circuit breaker activates
- Database failover — app reconnects
- Partial network partition — graceful degradation
- Memory pressure — no OOM crash
- Recovery after fault removal
