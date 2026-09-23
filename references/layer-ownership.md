# Layer Ownership Rules (Dedup)

각 테스트 항목은 정확히 하나의 소유 계층(owning layer)을 가진다. 같은 항목을 여러 계층에서 테스트하지 않는다. Phase A-3 Dedup Check와 Phase B/C 테스트 배치 시 이 파일을 Read한다.

Each test item has exactly ONE owning layer. Never test the same item in multiple layers.

| Verification Item | Owner Layer | Forbidden In |
|-------------------|-------------|--------------|
| Pure function logic (input→output) | **unit** | api, integration |
| Zod schema validation | **unit** | api |
| Constants/config values | **unit** | any other |
| Component rendering | **unit** | e2e |
| API endpoint success (200/201) | **api** | integration |
| API input validation (400) | **api** | security |
| Auth check (401) | **api** (per-feature file) | bulk security file |
| Admin check (403) | **api** (per-feature file) | bulk security file |
| Service-to-service with real DB | **integration** | unit (with full mock) |
| Browser-based user flow | **e2e** | api (using request only) |
| OWASP attack vectors | **security** | api |
| WCAG accessibility | **accessibility** | e2e |
| Response time / load | **performance** | any other |

**File rules:**
- 1 endpoint (or tightly related group) = 1 file
- No aggregation files (e.g., `management.test.ts` bundling separate endpoints)
- No temp/verify files — add regression TC to the canonical file with bug reference
- E2E files MUST use browser (`page`) — `request`-only tests go to api layer
