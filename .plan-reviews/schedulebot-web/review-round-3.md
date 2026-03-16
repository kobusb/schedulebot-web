# Plan Self-Review Round 3: Testability + Coherence

## A. Testability Review

### Phase 1 Workstream Acceptance Criteria

| Workstream | Task | Testable? | Criteria |
|-----------|------|-----------|----------|
| WS1: Sprint 0 | Project scaffolding | Yes | `npm run dev` starts without errors |
| WS1 | Tailwind + shadcn/ui | Yes | Component renders with correct styles |
| WS1 | Auth flow | Yes | OAuth redirect → callback → session cookie set → protected route accessible |
| WS1 | MSW mocking | Yes | BFF routes return mock data in development |
| WS1 | Slot computation spike | Yes | Unit tests pass for all timezone edge cases |
| WS2 | Dashboard shell | Yes | Sidebar renders, navigation works, responsive breakpoints correct |
| WS2 | Availability editor | Yes | Can set Mon-Fri 9-5, save, reload, see saved state |
| WS2 | Meeting type CRUD | Yes | Create → appears in list → edit → changes persist → delete → removed |
| WS2 | Calendar connection | Yes | OAuth flow completes, calendar shows as connected |
| WS3 | Booking page loads | Yes | `/{slug}/{type}` returns 200, shows meeting type info + time slots |
| WS3 | Slot display | Yes | Available slots shown in invitee's timezone, unavailable slots hidden |
| WS3 | Booking submission | Yes | Submit form → 201 response → confirmation page displays |
| WS3 | E2E booking flow | Yes | Playwright test: navigate → select date → select time → fill form → confirm |
| WS4 | Bookings list | Yes | List shows bookings, tabs filter correctly |
| WS4 | Cancel booking | Yes | Cancel → status changes → invitee notified (check sb_api call) |
| WS4 | Reschedule | Yes | New time selected → old booking cancelled → new booking created |
| WS5 | sb_api integration | Yes | MSW disabled, real API calls succeed, E2E tests pass |

### Missing Tests

| Gap | Severity | Suggestion |
|-----|----------|-----------|
| No test for timezone auto-detection | should-fix | Unit test: mock `Intl.DateTimeFormat` → verify correct timezone displayed |
| No test for error boundaries | should-fix | Integration test: mock sb_api 500 → verify "temporarily unavailable" shown |
| No test for session expiry | should-fix | Integration test: expired cookie → redirect to login |

All identified — these are should-fix additions to the test suite, not blockers.

---

## B. Coherence Review

### Naming Consistency

| Concept | Consistent? | Notes |
|---------|------------|-------|
| Meeting Types | Yes | Used consistently throughout. "Event Types" only appears in parenthetical Calendly comparison |
| Bookings | Yes | Consistent. "Events" only in Calendly comparison |
| Availability | Yes | "Availability schedules", "availability rules", "availability overrides" — clear hierarchy |
| Host / Invitee | Yes | Consistent actor names throughout |
| BFF | Yes | "BFF" used consistently for Next.js API routes |
| Slot computation | Yes | Same term in API design, scale design, and implementation plan |

### Architecture Coherence

All pieces fit together:
- PRD defines what → Design doc defines how → Implementation plan defines when
- BFF proxies all sb_api calls (no direct browser-to-sb_api communication)
- Auth flows through BFF session → sb_api API key + user headers
- Public routes (booking pages) use no auth, BFF handles public sb_api calls
- Schema matches API endpoints matches UI components

### Integration Points Between Phases

| From | To | Glue | Status |
|------|----|------|--------|
| Phase 1 (mock) → Phase 1 (real) | Workstream 5 | MSW → real sb_api | COVERED |
| Phase 1 → Phase 2 | Meeting type editor expansion | Same component, more fields | COVERED (form fields already in schema) |
| Phase 2 → Phase 3 | Email config | Dashboard UI → sb_api template API | COVERED (integration design has template customization) |
| Phase 3 → Phase 4 | Team features | New entity types, new meeting type variants | Acknowledged as v2, clean separation |

### Completeness Delta (after 5 prior rounds)

After 5 review rounds, the plan is comprehensive. One minor gap remains:

- ISSUE: No explicit task for **SEO metadata** on booking pages (title, description, OG tags). Booking pages are public and may be shared on social media.
  Severity: should-fix
  Fix: Add to Workstream 3 — dynamic OG tags and meta description per meeting type

---

## C. Fixes Applied

**1. SEO metadata**: Added to Workstream 3 — dynamic `<title>`, meta description, and OG tags per booking page (meeting type name, host name, duration).

**2. Test suggestions noted**: Timezone auto-detection, error boundary, and session expiry tests documented as should-adds to the test suite. Not blocking.

---

## Plan Review Summary (6 rounds complete)

**PRD Alignment (3 rounds):**
- Round 1 (requirements + goals): 8 fixes
- Round 2 (constraints + non-goals): 1 fix
- Round 3 (user-stories + open-questions): 4 fixes

**Plan Self-Review (3 rounds):**
- Round 4 (completeness + sequencing): 9 fixes
- Round 5 (risk + scope-creep): 6 fixes
- Round 6 (testability + coherence): 1 fix

**Total: 29 fixes applied across 6 review rounds.**

Plan is ready for bead creation.
