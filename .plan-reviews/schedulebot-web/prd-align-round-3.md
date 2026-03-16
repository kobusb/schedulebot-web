# PRD Alignment Round 3: User Stories + Open Questions

## A. User Stories Coverage

### Host Stories — End-to-End Journey Trace

**Story 1: Connect calendar**
- COVERED: Integration design → Calendar connect flow (steps 1-7). Workstream 2: Calendar connection flow. BFF route: `POST /api/calendars/connect`

**Story 2: Set weekly availability**
- COVERED: UX design → AvailabilityEditor. Workstream 2. BFF route: `GET/PUT /api/availability`. Schema: availability_schedules + availability_rules

**Story 3: Create meeting types**
- COVERED: UX design → MeetingTypeForm. Workstream 2. BFF routes: meeting-types CRUD. Schema: meeting_types table

**Story 4: Share scheduling link**
- COVERED: Workstream 2: "Copy scheduling link to clipboard." UX: Quick-copy in dashboard
- PARTIAL: The plan doesn't describe what the scheduling link looks like or where it's prominently displayed. **should-fix**: Add link format to UX design (`{domain}/{user-slug}`) and ensure it's visible on dashboard home + meeting type cards

**Story 5: See upcoming bookings**
- COVERED: UX design → BookingsList (Upcoming tab). Workstream 4: Bookings list

**Story 6: Cancel/reschedule booking (host-initiated)**
- COVERED: Workstream 4: "Cancel booking (host-initiated, with email notification)." BFF route: `DELETE /api/bookings/{id}`, `POST /api/bookings/{id}/reschedule`
- GAP: Host-initiated reschedule UI not detailed. Cancel is covered but reschedule needs a "pick new time" flow for the host. **should-fix**: Add host reschedule flow to Workstream 4 UX (host picks new time → invitee notified)

**Story 7: Buffer time**
- COVERED: Schema: buffer_before_mins, buffer_after_mins. Meeting type settings. Slot computation filters buffer violations

**Story 8: Date-specific overrides**
- COVERED: UX design → OverrideCalendar. Schema: availability_overrides. Phase 2 scope

**Story 9: Custom intake questions**
- COVERED: Schema: meeting_type_questions + booking_answers. Phase 2 scope

### Invitee Stories — End-to-End Journey Trace

**Story 10: Timezone display**
- COVERED: TimezoneSelector, auto-detection via `Intl.DateTimeFormat`, manual override. "Times shown in {timezone}" display

**Story 11: Book without account**
- COVERED: Public routes, no auth. Booking form collects name + email only. Explicit constraint: "Invitees must NOT need an account"

**Story 12: Confirmation with calendar invite + video link**
- COVERED (after round 1 fix): Booking confirmation page with "Add to Calendar" links (.ics, Google Calendar, Outlook). Video link from meeting type location_value
- GAP: Confirmation EMAIL with calendar invite attachment not detailed in plan. The confirmation page shows links, but the email from sb_api should also include an .ics attachment. **should-fix**: Note in integration design that sb_api confirmation email must include .ics attachment

**Story 13: Cancel/reschedule via email link**
- COVERED: Security design → self-service tokens (reschedule_token, cancel_token). BFF public routes for token-based cancel/reschedule

### Team Stories (v2)

**Story 14-15: Round-robin, collective**
- COVERED: Phase 4 (v2), correctly deferred per scope table

### Key Scenarios

**Booking flow**: COVERED — UX design steps 1-3 trace the complete invitee journey from link click to confirmation

**Availability management**: COVERED — AvailabilityEditor (weekly) + OverrideCalendar (dates) + Calendar connection

**Reschedule flow**: COVERED — Self-service tokens. Invitee sees new slots, picks new time, both notified

---

## B. Open Questions Resolution

### Original PRD Open Questions (now in "Remaining Open Questions" section — all resolved in human review)

All original questions were answered in the "Clarifications from Human Review" section. Checking each is reflected in plan:

| Question | Status | Plan Reference |
|----------|--------|----------------|
| Data model confirmed | RESOLVED | Schema in data design matches confirmed model |
| Team support v2 | RESOLVED | Phase 4 correctly defers team features |
| Custom domains v2 | RESOLVED | Phase 4 includes custom domains |
| Video links auto-generate (future sb_api) | RESOLVED | Plan notes paste-in for v1, sb_api auto-generate in Phase 4 |
| sb_api sends email with template customization | RESOLVED | Integration design: email notification flow + template customization section |
| Calendar auto-push confirmed | RESOLVED | Integration design: booking → calendar push flow |
| i18n v2 | RESOLVED | Phase 4 includes i18n framework. Plan notes next-intl |
| Embed widget v2-v3 | RESOLVED | Phase 4 includes embed widget |

### Questions from Human Review — Deferred Items

| Deferred Item | Status | Notes |
|--------------|--------|-------|
| i18n "unless deferring complicates later" | DEFERRED-OK | PRD recommends "structuring string extraction from day one." Plan doesn't explicitly include i18n-ready string extraction in Phase 1. **should-fix**: Add note to Workstream 1 to use string constants (not inline strings) for all user-facing text, making future i18n extraction easier |

---

## C. Fixes Applied

### Should-Fix

**1. Scheduling link visibility**: Added note to UX design that scheduling link format (`{domain}/{user-slug}`) is prominently displayed on dashboard home page and on each meeting type card.

**2. Host-initiated reschedule UI**: Added to Workstream 4 — host reschedule flow: host clicks "Reschedule" on booking → picks new time from availability → invitee notified with new time.

**3. Confirmation email .ics attachment**: Added note to integration design that sb_api confirmation email must include .ics calendar invite attachment (not just the web confirmation page).

**4. i18n-ready string extraction**: Added to Workstream 1 (Sprint 0) — use string constants for all user-facing text to enable future i18n extraction without refactoring.

---

## PRD Alignment Summary (3 rounds)

- **Round 1 (requirements + goals)**: 8 fixes applied (2 must-fix, 6 should-fix). Key: notification settings, a11y audit, brand deliverables
- **Round 2 (constraints + non-goals)**: 1 fix applied. Plan was well-constrained. Email verification default set to OFF
- **Round 3 (user-stories + open-questions)**: 4 fixes applied (all should-fix). Key: scheduling link visibility, host reschedule UI, .ics in emails, i18n-ready strings

**All PRD requirements, goals, constraints, non-goals, user stories, and open questions are now addressed in the plan.**
