# PRD Alignment Round 1: Requirements + Goals

## A. Requirements Coverage

### Problem Statement Requirements
- COVERED: "Calendly-style appointment scheduling frontend" → Design doc covers full booking flow, meeting types, availability
- COVERED: "hosts set availability, create meeting types, share links" → Workstream 2 (dashboard) + Workstream 3 (booking pages)
- COVERED: "invitees can self-book" → Booking pages design, public API routes, no-auth booking

### Explicit Requirements from PRD

| # | Requirement | Status | Plan Reference | Notes |
|---|------------|--------|----------------|-------|
| 1 | Meeting Types with name, description, slug, duration | COVERED | Data design: meeting_types table, UX: MeetingTypeForm |  |
| 2 | Duration options (15/30/45/60/custom) | COVERED | Schema: duration_minutes with CHECK > 0 |  |
| 3 | Location type (in-person, phone, video, custom) | COVERED | Schema: location_type + location_value |  |
| 4 | Booking window (how far ahead) | COVERED | Schema: booking_window_days |  |
| 5 | Minimum scheduling notice | COVERED | Schema: min_notice_hours, slot computation step 5c |  |
| 6 | Buffer time before/after | COVERED | Schema: buffer_before_mins, buffer_after_mins |  |
| 7 | Max bookings per day | COVERED | Schema: max_per_day |  |
| 8 | Custom intake questions | COVERED | Schema: meeting_type_questions, booking_answers |  |
| 9 | Active/inactive toggle | COVERED | Schema: is_active |  |
| 10 | Color coding | COVERED | Schema: color field |  |
| 11 | One-on-One meeting type | COVERED | Phase 1 scope |  |
| 12 | Group/Round-Robin/Collective | COVERED | Phase 4 (v2), acknowledged as non-v1 |  |
| 13 | Weekly recurring availability | COVERED | Schema: availability_rules |  |
| 14 | Multiple schedule profiles | COVERED | Schema: availability_schedules with name |  |
| 15 | Date-specific overrides | COVERED | Schema: availability_overrides |  |
| 16 | Calendar conflict checking | COVERED | Slot computation step 4, integration flow |  |
| 17 | Timezone-aware display | COVERED | date-fns-tz, timezone auto-detection, invitee_timezone |  |
| 18 | Booking status (confirmed/cancelled/rescheduled) | COVERED | Schema: bookings.status |  |
| 19 | Invitee responses to custom questions | COVERED | Schema: booking_answers |  |
| 20 | Calendar event sync | COVERED | Integration design: calendar push flow |  |
| 21 | Cancellation/reschedule by invitee | COVERED | Self-service tokens, security design |  |
| 22 | Personal scheduling page | COVERED | `/{user-slug}` route |  |
| 23 | Direct meeting type link | COVERED | `/{user-slug}/{meeting-type-slug}` route |  |
| 24 | Embeddable widget | COVERED | Phase 4 (v2), per scope table |  |
| 25 | Confirmation/reminder settings | GAP | Schema has no fields for this. sb_api handles email but frontend needs config UI. | **must-fix** |
| 26 | Cancellation/reschedule policy | GAP | PRD mentions "cancellation/reschedule policy" on bookings but no schema or UI design for configuring policies per meeting type. | **should-fix** |
| 27 | Booking page progressive enhancement | GAP | PRD says "must work without JavaScript for basic functionality." Plan mentions SSR but doesn't address no-JS fallback. | **should-fix** |

### User Story Coverage

| # | Story | Status | Notes |
|---|-------|--------|-------|
| 1 | Host connect calendar | COVERED | Calendar connection flow in integration design |
| 2 | Host set weekly availability | COVERED | AvailabilityEditor component |
| 3 | Host create meeting types | COVERED | MeetingTypeForm component |
| 4 | Host share scheduling link | COVERED | Copy link button in dashboard |
| 5 | Host see upcoming bookings | COVERED | BookingsList component |
| 6 | Host cancel/reschedule | COVERED | Booking management workstream |
| 7 | Host add buffer time | COVERED | Meeting type settings |
| 8 | Host date overrides | COVERED | OverrideCalendar component |
| 9 | Host custom intake questions | COVERED | Phase 2 scope |
| 10 | Invitee timezone display | COVERED | TimezoneSelector, auto-detect |
| 11 | Invitee book without account | COVERED | Public routes, no auth |
| 12 | Invitee confirmation + calendar invite | PARTIAL | Confirmation page exists but "Add to Calendar" link generation not detailed in plan. | **should-fix** |
| 13 | Invitee cancel/reschedule via email link | COVERED | Self-service tokens |
| 14 | Team round-robin | COVERED | Phase 4 (v2) |
| 15 | Team collective | COVERED | Phase 4 (v2) |

### Scenario Coverage

| Scenario | Status | Notes |
|----------|--------|-------|
| Booking flow | COVERED | Detailed in UX design step-by-step |
| Availability management | COVERED | AvailabilityEditor + OverrideCalendar |
| Reschedule flow | COVERED | Self-service tokens + reschedule endpoint |

---

## B. Goals Alignment

| # | Goal | Status | How Plan Achieves It | Notes |
|---|------|--------|---------------------|-------|
| 1 | Production-ready Calendly-style frontend | ALIGNED | Full-stack design with BFF, auth, booking flow, error handling |  |
| 2 | Intuitive booking pages for invitee self-scheduling | ALIGNED | 3-step booking flow (date → time → form), timezone auto-detect, mobile-responsive |  |
| 3 | Enable hosts to manage availability, meeting types, settings | ALIGNED | Dashboard with AvailabilityEditor, MeetingTypeForm, settings pages |  |
| 4 | Integrate with Google/Microsoft calendars | ALIGNED | Calendar connection flow, conflict checking, auto-push bookings |  |
| 5 | Schedule Bot brand identity with green palette | PARTIAL | Color palette defined in PRD, design tokens in Sprint 0 setup. But plan doesn't mention typography, logo, or visual design deliverables. | **should-fix** |
| 6 | Reference single-tenant implementation | PARTIAL | Code is well-structured, but plan doesn't include documentation as a deliverable. If this should be a reference, it needs docs/README/patterns guide. | **should-fix** — OR remove "reference implementation" from goals |
| 7 | Fast page loads, responsive design | ALIGNED | Server Components for booking pages, performance targets defined, mobile UX designed |  |
| 8 | WCAG 2.1 AA accessibility | PARTIAL | shadcn/ui (Radix) provides accessible primitives. But no accessibility testing task in the plan. Custom components (DatePicker, TimeSlotPicker) need manual a11y verification. | **must-fix** |

---

## C. Fixes Applied to Plan

### Must-Fix

**1. Add confirmation/reminder notification settings to schema and UI**

Added to meeting_types schema:
- `confirmation_enabled BOOLEAN DEFAULT true`
- `reminder_enabled BOOLEAN DEFAULT true`
- `reminder_hours_before SMALLINT DEFAULT 24`

Added to Phase 1 Workstream 2: notification settings in meeting type editor.

**2. Add accessibility testing to implementation plan**

Added to Phase 3:
- Axe-core automated accessibility scanning in CI
- Manual keyboard navigation testing for custom components (DatePicker, TimeSlotPicker, AvailabilityEditor)
- Screen reader testing for booking flow

### Should-Fix

**3. Add cancellation/reschedule policy config to meeting types**

Added to meeting_types schema:
- `allow_cancellation BOOLEAN DEFAULT true`
- `allow_reschedule BOOLEAN DEFAULT true`
- `cancellation_notice_hours SMALLINT DEFAULT 0` (0 = anytime before meeting)

**4. Clarify progressive enhancement for booking pages**

Added note to UX design: booking pages use Server Components for initial render. Date selection works via form submission fallback (no-JS). Time slot selection requires JS (acceptable — invitees booking meetings virtually certainly have JS enabled). Form submission works via standard HTML form POST.

**5. Add "Add to Calendar" link generation to booking confirmation**

Added to Workstream 4: generate `.ics` file download link and Google Calendar / Outlook web links on confirmation page.

**6. Add brand design deliverables to Sprint 0**

Added to Workstream 1 (Sprint 0): logo placeholder, typography selection (Inter or similar), favicon, OG image template for booking page social previews.

**7. Clarify reference implementation scope**

Downgraded from goal — this is a bonus outcome, not a v1 deliverable. Good code structure serves as implicit reference. Explicit documentation for other tenants deferred to post-launch.

**8. Add accessibility testing task**

(Covered in must-fix #2 above)
