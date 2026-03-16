# PRD Review Synthesis: Schedule Bot Web Frontend

## Overall Assessment

The draft PRD establishes solid directional intent for a modern scheduling web frontend. The tech stack choices (Next.js, Tailwind, shadcn/ui) are well-matched to the problem. However, the PRD has **5 critical gaps** that must be resolved before implementation can begin safely.

## Critical Issues (Must Resolve)

### 1. Auth is mislabeled as a non-goal
Auth is listed as "assumed provided by sb_api or an auth layer" but every user story requires knowing the user's role (Employee/Manager/Admin). This is a **hard prerequisite**, not a non-goal. We need clarity on: who provides auth, what's the session model, and how is RBAC enforced.

### 2. Tenant model is undefined
"Tenant" is used without definition. Is Schedule Bot an isolated sb_api instance, a tenant ID in a shared instance, or a multi-instance deployment? This affects API integration, data isolation, auth, and deployment architecture.

### 3. No API contract exists
The BFF pattern requires knowing the upstream API. Without sb_api documentation (endpoints, data model, capabilities), the frontend architecture is speculative. Risk: significant rework on integration.

### 4. Timezone handling is absent
Scheduling is inherently timezone-sensitive. No mention of how timezones are displayed, stored, or converted. DST transitions, multi-timezone teams, and shift display timezone all need specification.

### 5. Phase 1 has no standalone value
Auth + read-only viewing ships an empty app (no way to create schedules until Phase 2). Recommend including basic form-based schedule creation in Phase 1.

## Major Issues (Should Resolve)

### 6. Schedule terminology is ambiguous
"Schedule" means three things: a single shift, a weekly plan, and an employee's personal view. Define distinct terms (Shift, Schedule, Roster) to prevent implementation confusion.

### 7. Drag-drop calendar is the hardest part
Custom drag-drop scheduling grids are among the most complex frontend components. shadcn/ui doesn't provide this — it must be custom-built. Recommend prototyping early and deferring drag-drop to Phase 3 (use form-based creation in Phase 1-2).

### 8. Missing core features for scheduling tools
Not mentioned but expected: recurring schedules/templates, bulk operations, search/filter, print/PDF export, schedule lifecycle states (draft/published/archived). Templates and bulk operations are competitive differentiators.

### 9. Scale ranges are too broad
"Teams of 5-500" requires fundamentally different UX. Define MVP team size (5-50) and treat large-team support as a later enhancement.

### 10. Conflict detection rules undefined
PRD mentions conflict resolution but doesn't define what constitutes a conflict. Need: overtime thresholds, double-booking rules, minimum rest periods, cross-department policies.

### 11. No industry targeting
Scheduling needs vary dramatically by industry (healthcare shifts ≠ retail hours ≠ office schedules). Picking a primary target audience would sharpen every design decision.

### 12. sb_api team is an unacknowledged stakeholder
They control the API contract, auth, tenant model, and data model. Their input is a hard dependency for this project.

## Recommended Phase Reordering

**Current:**
1. Auth + Read-only viewing
2. Schedule creation (drag-drop)
3. Availability + conflicts
4. Dashboard + notifications

**Recommended:**
1. Auth + Schedule viewing + basic form-based creation (usable MVP)
2. Availability management + conflict detection
3. Drag-drop calendar UX (enhancement)
4. Dashboard, notifications, templates, rules configuration

## Questions for Human Clarification

### Blocking (cannot proceed without)
1. What is the sb_api API shape? (REST/GraphQL, endpoints, data model)
2. How is authentication handled? (OAuth2, session tokens, API keys?)
3. What does "tenant" mean in this context? (isolated instance, shared with tenant ID, other?)
4. What timezone model should we use? (user-local, org-default, UTC storage + local display?)

### Important (affects architecture)
5. What industry is the primary target? (healthcare, retail, office, general-purpose?)
6. What's the MVP team size? (5-50 recommended for v1)
7. Do schedules have lifecycle states? (draft → published → archived?)
8. What conflict rules should exist? (overtime, double-booking, rest periods?)

### Nice to have (can decide later)
9. Should shifts sync to external calendars? (Google Calendar, Outlook)
10. Is i18n needed from day one?
11. Is dark mode in scope?
12. Is this intended as a reference implementation for other tenants to copy?

## Review Dimension Summaries

| Dimension | Verdict | Key Finding |
|-----------|---------|-------------|
| Requirements | Incomplete | Auth labeled non-goal but is prerequisite; no acceptance criteria |
| Gaps | Significant | Timezone handling, schedule lifecycle, templates, bulk ops missing |
| Ambiguity | Blocking | "Tenant" and "schedule" have multiple meanings |
| Feasibility | Feasible with risk | Calendar drag-drop is the technical crux; prototype early |
| Scope | Ambitious | Phase reordering needed; MVP team size should be capped |
| Stakeholders | Critical gap | sb_api team must be consulted; industry targeting needed |

---
*Review performed by polecat/rust executing all 6 review dimensions inline (convoy dispatch unavailable).*
