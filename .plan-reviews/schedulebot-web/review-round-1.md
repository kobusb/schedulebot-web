# Plan Self-Review Round 1: Completeness + Sequencing

## A. Completeness Review

### Infrastructure & Setup

| Item | Status | Notes |
|------|--------|-------|
| Build config (next.config.js) | MISSING | **should-fix**: Add to Sprint 0 — configure image domains, env vars, redirects |
| Environment variables | MISSING | **must-fix**: No env var inventory. Need: SB_API_KEY, SB_API_URL, NEXT_PUBLIC_APP_URL, SESSION_SECRET |
| CI pipeline | Present | Mentioned in Sprint 0: "lint, typecheck, test, deploy" |
| Deployment config | MISSING | **should-fix**: Add Vercel project config or Dockerfile to Sprint 0 |
| Error monitoring (Sentry or similar) | MISSING | **should-fix**: Add to Phase 3 (Notifications & Experience) |

### Data & Migrations

| Item | Status | Notes |
|------|--------|-------|
| Database schema defined | Present | Full SQL in data design section |
| Migration strategy | MISSING | **should-fix**: Note that migrations are sb_api's responsibility. Schedule Bot has no database of its own — all data is via sb_api REST |
| Session storage | MISSING | **must-fix**: BFF uses session cookies but where is session data stored? Options: encrypted cookie (stateless), Redis, or database. Need a decision |

### Testing

| Item | Status | Notes |
|------|--------|-------|
| Unit tests | Mentioned | Tech stack: "Vitest" but no specific test tasks in workstreams |
| E2E tests | Mentioned | Tech stack: "Playwright" but no specific test tasks |
| Test strategy | MISSING | **must-fix**: Add explicit testing tasks. At minimum: E2E test for booking flow (the critical path), unit tests for slot computation (the complex logic) |

### Error Handling

| Item | Status | Notes |
|------|--------|-------|
| Slot conflict (409) | Present | Idempotency section covers double-booking |
| sb_api unavailable | MISSING | **should-fix**: Add error boundary strategy — what does the booking page show if sb_api is down? |
| OAuth failure | MISSING | **should-fix**: What if calendar OAuth fails or token is revoked? Add reconnect flow |

### Documentation

| Item | Status | Notes |
|------|--------|-------|
| README | MISSING | **should-fix**: Add to Sprint 0 — project setup instructions, env var docs |
| OpenAPI spec | Present | In Sprint 0: "OpenAPI spec for sb_api scheduling endpoints" |

### Coarse-Grained Tasks

| Task | Issue | Suggestion |
|------|-------|-----------|
| "Auth flow (OAuth via sb_api, session cookies)" | Too broad for one task | **should-fix**: Break into: (1) OAuth redirect flow, (2) Callback handler + session creation, (3) Session middleware, (4) Logout |
| "Available slot computation in BFF" | Complex, critical | **should-fix**: Break into: (1) Fetch availability rules, (2) Fetch calendar busy times, (3) Slot generation algorithm, (4) Filter logic (buffers, notice, max/day) |

---

## B. Sequencing Review

### Current Phase Order
1. Sprint 0 (project setup)
2. Workstream 2 (host dashboard) + Workstream 3 (booking pages) + Workstream 4 (booking management)
3. Phase 2 (advanced settings)
4. Phase 3 (notifications & experience)
5. Phase 4 (v2 features)

### Sequencing Issues

**1. Workstream 3 depends on Workstream 2 but they're listed as parallel**
- FINDING: Booking pages need meeting types to exist. If a host hasn't created any meeting types, the booking page shows nothing. Workstream 2 (meeting type CRUD) must come before Workstream 3 (booking pages) — or at least meeting type CRUD must be done first.
- Severity: **should-fix** (the dependency map already shows this, but the workstream descriptions should make it explicit)
- Suggested: Add "(requires meeting type CRUD from Workstream 2)" note to Workstream 3

**2. Calendar connection is in Workstream 2 but calendar sync is in Workstream 4**
- FINDING: Calendar connection (OAuth) and calendar conflict checking are related but split across workstreams. Connection is needed before slot computation can use calendar data.
- Severity: **should-fix** (not a blocker — slot computation works without calendar data, just shows all available slots)
- Suggested: Note that slot computation initially works without calendar data, then calendar filtering is added when connection flow is complete

**3. Slot computation is the critical path — it's in Workstream 3 but depends on availability from Workstream 2**
- FINDING: The slot computation algorithm needs availability rules to exist. This is correctly shown in the dependency map but not emphasized as the critical path.
- Severity: **should-fix**: Mark slot computation as critical path. The minimum viable chain is: Sprint 0 → Auth → Availability Editor → Meeting Type CRUD → Slot Computation → Booking Form → Confirmation

**4. Phase 2 items could be Phase 1**
- FINDING: Buffer times, booking limits, and date overrides are in Phase 2 but the schema already supports them and the slot computation already accounts for them. They only need UI in the meeting type editor.
- Severity: **should-fix**: Consider moving buffer times and booking window config into Phase 1's meeting type editor (they're just form fields). Keep date overrides in Phase 2 (they need a separate calendar UI).

### Critical Path (corrected)

```
Sprint 0 → Auth Flow → Availability Editor → Meeting Type CRUD
                                                     ↓
                              Slot Computation ← Meeting Types exist
                                     ↓
                              Booking Form → Booking Confirmation
                                     ↓
                              Booking Management (list, cancel, reschedule)
```

Calendar connection can happen in parallel, enhancing slot computation when ready.

---

## C. Fixes Applied

### Must-Fix

**1. Environment variable inventory**: Added to Sprint 0 — SB_API_KEY, SB_API_URL, NEXT_PUBLIC_APP_URL, SESSION_SECRET, NEXT_PUBLIC_DEFAULT_TIMEZONE

**2. Session storage decision**: Added to design doc — use encrypted httpOnly cookies (iron-session or next-auth session strategy). Stateless, no external session store needed for v1 scale.

**3. Testing tasks**: Added explicit testing to workstreams:
- Workstream 3: E2E test for complete booking flow (Playwright)
- Slot computation: Unit tests with Vitest (this is the most complex pure logic)
- Workstream 4: E2E test for cancel/reschedule flow

### Should-Fix

**4. Deployment config**: Added Dockerfile to Sprint 0 deliverables

**5. Error monitoring**: Added Sentry/error monitoring setup to Phase 3

**6. sb_api error handling**: Added error boundary notes — booking page shows "temporarily unavailable" with retry if sb_api is down

**7. OAuth failure handling**: Added to Workstream 2 — reconnect flow if calendar token is revoked

**8. Critical path notation**: Added to dependency map

**9. Phase 1 scope adjustment**: Moved buffer time and booking window config from Phase 2 to Phase 1 meeting type editor (they're just form fields, schema already supports them)
