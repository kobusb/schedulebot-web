# Plan Self-Review Round 2: Risk + Scope Creep

## A. Risk Assessment

### Technical Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| Slot computation correctness (timezone edge cases, DST transitions) | HIGH | MEDIUM | **must-fix**: Add spike task — build slot computation with comprehensive test cases for DST, multi-timezone, edge-of-day boundaries BEFORE building UI |
| sb_api endpoints don't exist yet — contract may change | HIGH | MEDIUM | Mitigated: OpenAPI spec in Sprint 0 establishes contract. MSW mocking decouples frontend dev. Risk is managed |
| Calendar API rate limits (Google/Microsoft) | MEDIUM | MEDIUM | sb_api concern. **should-fix**: Note that Schedule Bot BFF should handle 429/503 from sb_api gracefully (retry with backoff on slot queries) |
| shadcn/ui + Tailwind v4 compatibility | LOW | LOW | Both are actively maintained and designed to work together. Low risk |

### Dependency Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| sb_api development pace blocks frontend | HIGH | MEDIUM | Mitigated: MSW mocking decouples. Frontend can ship with mocks and integrate later. **should-fix**: Add explicit "mock → real" integration milestone to plan |
| iron-session library stability | LOW | LOW | Well-maintained, minimal dependency. Acceptable |
| date-fns-tz correctness | MEDIUM | LOW | Widely used, well-tested. **should-fix**: Pin IANA timezone database version for reproducible behavior |

### Knowledge Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| .ics file generation | LOW | LOW | Well-documented RFC 5545 standard. Libraries exist (ical-generator) |
| Accessibility for custom calendar/time components | MEDIUM | MEDIUM | **should-fix**: Reference WAI-ARIA date picker pattern in component design. shadcn/ui calendar component already follows this |

### Rollback Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Phase 1 ships but booking flow has bugs | MEDIUM | Feature flag: `ENABLE_PUBLIC_BOOKING=false` disables booking pages while dashboard remains accessible. **should-fix**: Add feature flag to Sprint 0 |
| sb_api integration breaks after switching from mocks | MEDIUM | MSW mocks serve as regression tests. Integration tests catch contract mismatches. Acceptable |

---

## B. Scope Creep Assessment

| Item | Verdict | Notes |
|------|---------|-------|
| Security design: detailed threat model | KEEP | Appropriate for a public-facing booking page handling email addresses |
| Idempotency keys | KEEP | Prevents double-booking — essential, not gold-plating |
| Multiple availability schedule profiles | DEFER | PRD mentions this but it's power-user feature. V1 needs ONE default schedule per user. Multiple profiles can be Phase 2+ | **should-fix** |
| Session sliding window (7 days) | SIMPLIFY | Fixed 7-day expiry is simpler than sliding window. Sliding window adds complexity for marginal UX benefit | **should-fix** |
| Honeypot field in booking form | KEEP | Trivial to implement, effective against basic bots |
| TanStack Query for dashboard | KEEP | Standard choice, not over-engineering |
| Zustand removed (was in original draft) | ALREADY CUT | Round 1 changed to React Context. Good |
| OpenAPI spec as Sprint 0 deliverable | KEEP | Essential for sb_api coordination |

---

## C. Fixes Applied

### Must-Fix

**1. Spike: slot computation + timezone tests**: Added to Sprint 0 — build and test the slot computation algorithm with comprehensive timezone edge cases (DST transitions, boundary times, multi-timezone) BEFORE building booking page UI. This de-risks the critical path.

### Should-Fix

**2. sb_api retry logic**: Added note to BFF design — handle 429/503 from sb_api with exponential backoff on slot queries.

**3. Mock → real integration milestone**: Added to Phase 1 — explicit "Replace MSW mocks with sb_api integration" task after sb_api endpoints are ready.

**4. Feature flag**: Added ENABLE_PUBLIC_BOOKING to Sprint 0 env vars. Allows disabling booking pages independently of dashboard.

**5. Simplify availability to single default schedule**: V1 supports ONE availability schedule per user. Multiple profiles deferred to Phase 2+. Removes schedule_id FK from meeting_types in v1 (all meeting types use user's default schedule).

**6. Session expiry**: Simplified to fixed 7-day expiry (no sliding window for v1).
