# Design Document: Schedule Bot Web Frontend

## 1. API Design — Interface & Developer Ergonomics

### BFF Layer (Next.js API Routes → sb_api)

The BFF is the only component that talks to sb_api directly. It holds the API key server-side and translates between frontend needs and sb_api's REST interface.

#### BFF Route Structure

```
app/api/
├── auth/
│   ├── login/route.ts          # Initiate OAuth flow via sb_api
│   ├── callback/route.ts       # Handle OAuth callback, set session cookie
│   └── logout/route.ts         # Clear session
├── me/route.ts                 # GET current user profile
├── availability/
│   ├── route.ts                # GET/PUT weekly availability schedule
│   └── overrides/route.ts      # GET/POST/DELETE date overrides
├── meeting-types/
│   ├── route.ts                # GET list, POST create
│   └── [id]/route.ts           # GET/PUT/DELETE single meeting type
├── meeting-types/[id]/questions/
│   └── route.ts                # GET/POST/PUT/DELETE custom questions
├── bookings/
│   ├── route.ts                # GET list (with filters: upcoming, past, cancelled)
│   └── [id]/
│       ├── route.ts            # GET/DELETE (cancel)
│       └── reschedule/route.ts # POST reschedule
├── calendars/
│   ├── route.ts                # GET connected calendars
│   └── connect/route.ts        # POST initiate calendar OAuth
└── public/                     # No auth required — invitee-facing
    ├── [user-slug]/route.ts    # GET host profile + active meeting types
    ├── [user-slug]/[meeting-type-slug]/
    │   ├── route.ts            # GET meeting type details
    │   └── slots/route.ts      # GET available time slots for date range
    └── bookings/route.ts       # POST create booking (invitee submits)
```

#### Key API Patterns

**Session management**: OAuth callback sets an httpOnly secure cookie with a session token. BFF validates session on each request and injects `X-User-ID` header to sb_api.

**Slot computation**: The `GET /api/public/[user-slug]/[meeting-type-slug]/slots` endpoint is the most complex. It must:
1. Fetch host's availability rules from sb_api
2. Fetch host's calendar events (busy times) from sb_api
3. Fetch existing bookings from sb_api
4. Compute available slots considering: availability rules, calendar conflicts, existing bookings, buffer times, minimum notice, booking window, max bookings/day
5. Return slots in the invitee's requested timezone

**Decision**: Slot computation lives in the **BFF**, not sb_api. Rationale:
- sb_api provides raw data (availability rules, calendar events, bookings)
- Schedule Bot owns the slot computation algorithm (different tenants may compute differently)
- Keeps sb_api simple and tenant-agnostic

**Caching strategy**: Available slots are volatile (a new booking invalidates them). Use short TTL (30s) or no cache on slot endpoints. Meeting type details and host profiles can be cached longer (5m).

#### sb_api Endpoints Needed (co-development)

These endpoints must be built in sb_api to support the frontend:

```
# Availability
GET    /api/v1/users/{id}/availability          # Weekly schedule
PUT    /api/v1/users/{id}/availability          # Set weekly schedule
GET    /api/v1/users/{id}/availability/overrides # Date overrides
POST   /api/v1/users/{id}/availability/overrides # Create override
DELETE /api/v1/users/{id}/availability/overrides/{id}

# Meeting Types
GET    /api/v1/meeting-types                     # List for tenant
POST   /api/v1/meeting-types                     # Create
GET    /api/v1/meeting-types/{id}                # Get by ID or slug
PUT    /api/v1/meeting-types/{id}                # Update
DELETE /api/v1/meeting-types/{id}                # Soft delete

# Meeting Type Questions
GET    /api/v1/meeting-types/{id}/questions
POST   /api/v1/meeting-types/{id}/questions
PUT    /api/v1/meeting-types/{id}/questions/{id}
DELETE /api/v1/meeting-types/{id}/questions/{id}

# Bookings
GET    /api/v1/bookings                          # List (filterable by status, date range)
POST   /api/v1/bookings                          # Create (invitee books)
GET    /api/v1/bookings/{id}                     # Get details
DELETE /api/v1/bookings/{id}                     # Cancel
POST   /api/v1/bookings/{id}/reschedule          # Reschedule

# Calendar Events (for conflict checking)
GET    /api/v1/users/{id}/calendar-events?start=&end=  # Busy times from connected calendars
```

### Client-Side Data Layer

```typescript
// lib/api/client.ts — typed fetch wrapper
// All dashboard data fetching goes through TanStack Query hooks

// Example hook structure:
// hooks/use-meeting-types.ts
// hooks/use-availability.ts
// hooks/use-bookings.ts
// hooks/use-available-slots.ts (public, no auth)
```

**TanStack Query configuration**:
- `staleTime: 60_000` for dashboard data (1 minute)
- `staleTime: 0` for booking slots (always refetch)
- Optimistic updates for meeting type edits, availability changes
- `queryKey` conventions: `['meeting-types']`, `['bookings', { status }]`, `['slots', userSlug, meetingTypeSlug, dateRange]`

---

## 2. Data Design — Models, Storage, Migrations

### sb_api Database Schema (proposed additions)

```sql
-- Availability: weekly recurring rules
CREATE TABLE availability_schedules (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name        VARCHAR(100) NOT NULL DEFAULT 'Default',
    timezone    VARCHAR(50) NOT NULL,  -- IANA timezone (e.g., America/New_York)
    is_default  BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, name)
);

-- Weekly rules: which hours on which days
CREATE TABLE availability_rules (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_id UUID NOT NULL REFERENCES availability_schedules(id) ON DELETE CASCADE,
    day_of_week SMALLINT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6), -- 0=Sunday
    start_time  TIME NOT NULL,
    end_time    TIME NOT NULL,
    CHECK (end_time > start_time)
);
CREATE INDEX idx_availability_rules_schedule ON availability_rules(schedule_id);

-- Date-specific overrides
CREATE TABLE availability_overrides (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    date        DATE NOT NULL,
    start_time  TIME,       -- NULL = unavailable entire day
    end_time    TIME,       -- NULL = unavailable entire day
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, date),
    CHECK (
        (start_time IS NULL AND end_time IS NULL) OR
        (start_time IS NOT NULL AND end_time IS NOT NULL AND end_time > start_time)
    )
);

-- Meeting types
CREATE TABLE meeting_types (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name                VARCHAR(200) NOT NULL,
    slug                VARCHAR(100) NOT NULL,
    description         TEXT,
    duration_minutes    SMALLINT NOT NULL CHECK (duration_minutes > 0),
    location_type       VARCHAR(20) NOT NULL DEFAULT 'video',  -- video, phone, in_person, custom
    location_value      TEXT,  -- video link, phone number, address, or custom text
    color               VARCHAR(7),  -- hex color for calendar display
    booking_window_days SMALLINT NOT NULL DEFAULT 60,
    min_notice_hours    SMALLINT NOT NULL DEFAULT 4,
    buffer_before_mins  SMALLINT NOT NULL DEFAULT 0,
    buffer_after_mins   SMALLINT NOT NULL DEFAULT 0,
    max_per_day         SMALLINT,  -- NULL = unlimited
    is_active           BOOLEAN NOT NULL DEFAULT true,
    schedule_id         UUID REFERENCES availability_schedules(id),  -- NULL = use default
    confirmation_enabled BOOLEAN NOT NULL DEFAULT true,
    reminder_enabled    BOOLEAN NOT NULL DEFAULT true,
    reminder_hours_before SMALLINT NOT NULL DEFAULT 24,
    allow_cancellation  BOOLEAN NOT NULL DEFAULT true,
    allow_reschedule    BOOLEAN NOT NULL DEFAULT true,
    cancellation_notice_hours SMALLINT NOT NULL DEFAULT 0,  -- 0 = anytime before meeting
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, slug)
);

-- Custom intake questions per meeting type
CREATE TABLE meeting_type_questions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_type_id UUID NOT NULL REFERENCES meeting_types(id) ON DELETE CASCADE,
    label           VARCHAR(500) NOT NULL,
    type            VARCHAR(20) NOT NULL DEFAULT 'text',  -- text, textarea, select, checkbox
    options         JSONB,  -- for select: ["Option A", "Option B"]
    is_required     BOOLEAN NOT NULL DEFAULT false,
    sort_order      SMALLINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_questions_meeting_type ON meeting_type_questions(meeting_type_id, sort_order);

-- Bookings
CREATE TABLE bookings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_type_id UUID NOT NULL REFERENCES meeting_types(id),
    host_user_id    UUID NOT NULL REFERENCES users(id),
    invitee_name    VARCHAR(200) NOT NULL,
    invitee_email   VARCHAR(320) NOT NULL,
    invitee_timezone VARCHAR(50) NOT NULL,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    location_value  TEXT,  -- resolved location (actual video link, etc.)
    status          VARCHAR(20) NOT NULL DEFAULT 'confirmed',  -- confirmed, cancelled, rescheduled
    cancel_reason   TEXT,
    calendar_event_id VARCHAR(500),  -- external calendar event ID for sync
    reschedule_token VARCHAR(64) UNIQUE,  -- for invitee self-service reschedule
    cancel_token     VARCHAR(64) UNIQUE,   -- for invitee self-service cancel
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bookings_host_time ON bookings(host_user_id, start_time) WHERE status = 'confirmed';
CREATE INDEX idx_bookings_meeting_type ON bookings(meeting_type_id, start_time);

-- Answers to custom questions
CREATE TABLE booking_answers (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id  UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
    question_id UUID NOT NULL REFERENCES meeting_type_questions(id),
    value       TEXT NOT NULL,
    UNIQUE(booking_id, question_id)
);
```

### Key Data Design Decisions

1. **All times stored as TIMESTAMPTZ** (UTC in PostgreSQL). Display conversion happens in the frontend.
2. **Availability rules are time-of-day in the host's timezone**. The `availability_schedules.timezone` field anchors the interpretation. This is important because "9am-5pm" must respect DST transitions.
3. **Bookings store absolute UTC times**. No ambiguity.
4. **Invitee self-service tokens**: `reschedule_token` and `cancel_token` are random 64-char strings embedded in confirmation email links. No auth required — the token IS the auth.
5. **Soft delete for meeting types**: `is_active = false` rather than DELETE, because bookings reference them.
6. **Multiple availability schedules per user**: Supports "Weekday hours" vs "Consulting hours" — a meeting type can be linked to a specific schedule.

---

## 3. UX Design — User Experience & Interface

### Information Architecture

```
Host Dashboard (authenticated)
├── Home                    # Quick stats, upcoming bookings, scheduling link
├── Bookings                # List view with tabs: Upcoming, Past, Cancelled
│   └── Booking Detail      # Full details, cancel/reschedule actions
├── Meeting Types           # Card grid of all meeting types
│   ├── New Meeting Type    # Creation wizard
│   └── Edit Meeting Type   # Settings, questions, availability link
├── Availability            # Weekly schedule editor + overrides calendar
├── Integrations            # Calendar connections (Google, Microsoft)
└── Settings                # Profile, branding, preferences

Booking Pages (public)
├── /{user-slug}            # Host profile + meeting type list
└── /{user-slug}/{type-slug}
    ├── Step 1: Select Date # Calendar month view, available dates highlighted
    ├── Step 2: Select Time # Time slot list for selected date
    └── Step 3: Confirm     # Name, email, custom questions, submit
```

### Booking Page UX (the critical path)

This is the most important UX surface — it's what every invitee sees.

**Step 1 — Date Selection:**
- Left panel: meeting type info (name, duration, description, host photo/name)
- Right panel: calendar month view
- Dates with available slots are selectable (bold/colored)
- Dates without slots are grayed out
- Timezone selector at top: auto-detected, manually overridable

**Step 2 — Time Selection:**
- Same left panel with meeting info
- Right panel switches to time slot list for the selected date
- Slots shown as buttons: "9:00 AM", "9:30 AM", "10:00 AM"...
- Clicking a slot shows a "Confirm" button next to it (inline confirmation, like Calendly)
- Shows timezone context: "Times shown in America/New_York"

**Step 3 — Booking Form:**
- Name (required)
- Email (required)
- Custom intake questions (if configured)
- "Schedule Event" primary CTA
- After submission: confirmation page with details + "Add to Calendar" link

**Mobile considerations:**
- Stack left/right panels vertically (meeting info on top, calendar/slots below)
- Time slots as full-width buttons for easy tapping
- Sticky "Back" button to return to date selection

### Host Dashboard UX

**Availability Editor:**
- Weekly grid: 7 columns (days), rows for hours
- Drag to create availability blocks (similar to Google Calendar's working hours)
- Toggle days on/off
- Add multiple blocks per day (e.g., 9-12 and 2-5, skip lunch)
- Override calendar: month view where you can click dates to set overrides

**Meeting Type Editor:**
- Wizard-style creation: Name → Duration → Location → Settings → Questions
- Live preview of how the booking page will look
- Copy link button always visible

**Bookings List:**
- Tab bar: Upcoming | Past | Cancelled
- Each booking shows: invitee name/email, meeting type, date/time, status
- Click to expand: full details, intake question answers, cancel/reschedule actions
- Calendar view toggle (day/week/month) as alternative to list

### Component Architecture

```
components/
├── ui/                          # shadcn/ui primitives
│   ├── button, input, dialog, select, tabs, calendar...
├── booking/                     # Public booking page components
│   ├── MeetingTypeInfo.tsx      # Left panel: host info + meeting details
│   ├── DatePicker.tsx           # Calendar month with available date highlighting
│   ├── TimeSlotPicker.tsx       # List of available time slots
│   ├── BookingForm.tsx          # Name, email, custom questions
│   ├── BookingConfirmation.tsx  # Success page with details
│   └── TimezoneSelector.tsx     # Dropdown with auto-detect
├── dashboard/
│   ├── AvailabilityEditor.tsx   # Weekly schedule drag editor
│   ├── OverrideCalendar.tsx     # Date override management
│   ├── MeetingTypeCard.tsx      # Card in the meeting type grid
│   ├── MeetingTypeForm.tsx      # Create/edit form
│   ├── BookingsList.tsx         # Filterable booking list
│   ├── BookingDetail.tsx        # Single booking expanded view
│   └── QuickStats.tsx           # Home page metrics
└── layout/
    ├── DashboardShell.tsx       # Sidebar + top bar + main content
    ├── Sidebar.tsx              # Navigation links
    └── PublicLayout.tsx         # Minimal layout for booking pages
```

---

## 4. Scale Design — Performance & Bottlenecks

### Performance Targets

| Surface | Metric | Target | Strategy |
|---------|--------|--------|----------|
| Booking pages | LCP | < 1.5s | Server Components, minimal JS |
| Booking pages | FID | < 50ms | Hydrate only interactive elements |
| Booking pages | TTFB | < 200ms | Edge caching for static parts |
| Slot endpoint | Response time | < 500ms | Efficient availability computation |
| Dashboard | LCP | < 2.5s | Standard SPA patterns, code splitting |

### Availability Slot Computation (the hot path)

This is the most CPU-intensive operation. For each slot request:

```
Input: host_id, meeting_type_id, date_range (typically 1 month), invitee_timezone
Output: list of available time slots

Algorithm:
1. Fetch availability rules for the host (cacheable, ~1 query)
2. Fetch date overrides in range (~1 query)
3. Fetch existing confirmed bookings in range (~1 query)
4. Fetch calendar busy times from sb_api (~1 API call, may hit Google/Microsoft)
5. For each day in range:
   a. Get applicable availability windows (rules + overrides)
   b. Generate candidate slots (every 15/30/60m depending on duration)
   c. Filter out: calendar conflicts, existing bookings, buffer violations,
      past slots, minimum notice violations, max-per-day exceeded
6. Return filtered slots in invitee's timezone
```

**Optimization strategy:**
- Steps 1-2 can be cached (availability changes infrequently)
- Step 3-4 must be fresh (bookings change constantly)
- Calendar busy times are the bottleneck — sb_api should cache synced events
- Compute slots on the server (BFF) to avoid sending raw availability to the client
- Return slots for 1 week at a time, load more as invitee navigates

### Scaling Considerations

**Low scale (v1 target: hundreds of hosts, thousands of bookings/month):**
- Single Next.js deployment on Vercel handles this easily
- PostgreSQL handles the query load without optimization
- No need for Redis, queues, or worker processes

**Future scale considerations (design for, don't build yet):**
- Calendar sync can become a background job (sb_api concern)
- Slot computation could be cached in Redis with invalidation on booking/availability change
- Booking creation should be idempotent (prevent double-booking from double-click)

### Idempotency & Race Conditions

**Double-booking prevention**: When an invitee submits a booking:
1. BFF sends `POST /api/v1/bookings` to sb_api
2. sb_api must check for conflicts within a transaction (SELECT FOR UPDATE on the time range)
3. If conflict detected, return 409 Conflict
4. Frontend shows "This slot was just booked — please select another time"

**Idempotency key**: Include a client-generated UUID in the booking request. sb_api should deduplicate by this key to prevent double-submissions.

---

## 5. Security Design — Threat Model & Attack Surface

### Attack Surface

| Surface | Exposure | Risk |
|---------|----------|------|
| Booking pages | Public internet | Spam bookings, scraping, DoS |
| BFF API routes | Public internet (behind auth) | Session hijacking, CSRF |
| sb_api key | Server-side only | Key leakage = full tenant access |
| OAuth tokens | Encrypted in sb_api DB | Token theft = calendar access |
| Invitee tokens | In email links | Token brute-force = booking manipulation |

### Threat Mitigations

**1. sb_api Key Protection**
- Key NEVER sent to the browser. Only the BFF server-side code uses it.
- Store in environment variable (`SB_API_KEY`), not in code.
- Next.js server-only modules (`server-only` package) prevent accidental client import.

**2. Session Security**
- httpOnly, Secure, SameSite=Lax cookies for session tokens
- Session tokens are opaque (random), not JWTs (no client-side decoding needed)
- Session expiry: 7 days with sliding window
- CSRF protection via SameSite cookie + Origin header validation

**3. Booking Page Abuse Prevention**
- Rate limiting on booking creation: max 5 bookings per IP per hour (implemented in BFF middleware)
- Rate limiting on slot queries: max 60 requests per IP per minute
- CAPTCHA (hCaptcha or Turnstile) on booking form if abuse detected
- Email verification: optional "confirm booking" link (configurable per meeting type, default OFF — PRD requires frictionless booking)
- Honeypot field in booking form for basic bot detection

**4. Invitee Self-Service Tokens**
- 64 characters, cryptographically random
- Single-use: cancel token consumed on use
- Expiry: reschedule/cancel tokens expire when the booking's start_time passes
- Not guessable: 64 chars of `[a-zA-Z0-9]` = ~384 bits of entropy

**5. Input Validation**
- All user input validated on both client (UX) and server (security)
- Email format validation
- Slug validation (alphanumeric + hyphens, 3-100 chars, reserved word blocklist — match sb_api's existing rules)
- Meeting type settings: sane ranges (duration 5-480 min, buffer 0-120 min, window 1-365 days)
- Custom question answers: max length 5000 chars, sanitize HTML

**6. Content Security**
- CSP headers: restrict script sources, no inline scripts in booking pages
- X-Frame-Options: DENY (until embed widget in v2)
- No user-generated HTML rendering (markdown at most, sanitized)

### Data Privacy
- Invitee email/name stored only in bookings table (not a user account)
- No tracking pixels or third-party analytics on booking pages (v1)
- Booking data retention: follow host's configured policy (default: indefinite, configurable deletion)

---

## 6. Integration Design — System Fit

### Architecture Context

```
┌─────────────────────────────────────────────┐
│                  Invitee                     │
│              (web browser)                   │
└──────────────────┬──────────────────────────┘
                   │ HTTPS
                   ▼
┌─────────────────────────────────────────────┐
│           Schedule Bot (Next.js)             │
│  ┌──────────────┐  ┌─────────────────────┐  │
│  │ Booking Pages │  │  Host Dashboard     │  │
│  │ (SSR/public) │  │  (auth'd SPA)       │  │
│  └──────┬───────┘  └────────┬────────────┘  │
│         │                   │               │
│  ┌──────▼───────────────────▼────────────┐  │
│  │         BFF API Routes                 │  │
│  │  - Session management                  │  │
│  │  - Slot computation                    │  │
│  │  - Rate limiting                       │  │
│  │  - sb_api proxy                        │  │
│  └──────────────────┬────────────────────┘  │
└─────────────────────┼───────────────────────┘
                      │ REST (API key auth)
                      ▼
┌─────────────────────────────────────────────┐
│              sb_api (Go)                     │
│  - Tenants, Users, API Keys                 │
│  - Availability, Meeting Types, Bookings    │
│  - Calendar OAuth + Sync                    │
│  - Email notifications (templates)          │
│  ├──────────────┐  ┌─────────────────────┐  │
│  │  PostgreSQL   │  │  Google/Microsoft   │  │
│  │  (data store) │  │  Calendar APIs      │  │
│  └──────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────┘
```

### Calendar Integration Flow

**Connect calendar (host onboarding):**
1. Host clicks "Connect Google Calendar" in Schedule Bot dashboard
2. BFF calls `POST /api/v1/oauth/google` on sb_api (with user context)
3. sb_api returns OAuth authorization URL
4. Host redirected to Google consent screen
5. Google redirects to sb_api's callback URL
6. sb_api stores encrypted tokens, returns success
7. BFF redirects host back to dashboard with success message

**Calendar conflict checking (during slot computation):**
1. Invitee requests available slots
2. BFF calls `GET /api/v1/users/{id}/calendar-events?start=...&end=...`
3. sb_api fetches events from connected calendars (Google/Microsoft APIs)
4. sb_api returns busy time ranges
5. BFF excludes these from available slots

**Booking → Calendar push:**
1. Invitee confirms booking
2. BFF calls `POST /api/v1/bookings`
3. sb_api creates booking record
4. sb_api creates calendar event on host's connected calendar via Google/Microsoft API
5. sb_api returns booking with `calendar_event_id`
6. On cancellation/reschedule, sb_api updates/deletes the calendar event

### Email Notification Flow

All email is sent by sb_api (confirmed by human review):

1. Booking created → sb_api sends confirmation to invitee + notification to host
2. Booking cancelled → sb_api sends cancellation notice to both parties
3. Booking rescheduled → sb_api sends reschedule notice to both parties
4. Reminder (configurable) → sb_api sends reminder N hours before meeting

**Template customization**: sb_api provides base email templates. Schedule Bot can customize via API:
- Brand colors, logo URL
- Custom text for confirmation/reminder/cancellation
- Reply-to address

### Co-Development Coordination with sb_api

Since scheduling entities don't exist in sb_api yet, development must be coordinated:

**Phase 1 approach — Mock API, then integrate:**
1. Define the sb_api endpoint contracts (request/response schemas) in an OpenAPI spec
2. Build Schedule Bot BFF against mocked responses (MSW or static fixtures)
3. sb_api team implements endpoints to match the agreed contract
4. Replace mocks with real sb_api calls

**Contract-first development**: Write OpenAPI YAML for all proposed endpoints before either side starts coding. This becomes the handshake document.

---

## Implementation Plan

### Phase 1: Core Booking Flow (MVP)

**Goal**: A host can create a meeting type, set availability, and share a link. An invitee can book a time slot.

#### Workstream 1: Project Setup (Sprint 0)
- Next.js App Router project scaffolding
- Tailwind CSS v4 + shadcn/ui setup
- Brand design tokens (colors, typography selection [Inter or similar], spacing)
- Logo placeholder, favicon, OG image template for booking page social previews
- BFF pattern setup (API route structure, sb_api client)
- Auth flow (OAuth via sb_api, session cookies)
- CI/CD pipeline (lint, typecheck, test, deploy)
- Mock sb_api responses (MSW) for development
- OpenAPI spec for sb_api scheduling endpoints (contract-first)

#### Workstream 2: Host Dashboard Foundation
- Dashboard shell (sidebar, top bar, responsive layout)
- Auth-protected routes with session validation
- Availability editor (weekly schedule UI)
- Meeting type CRUD (create, edit, list, delete)
- Meeting type notification settings (confirmation, reminder timing)
- Meeting type cancellation/reschedule policy settings
- Calendar connection flow (Google/Microsoft OAuth through sb_api)
- Copy scheduling link to clipboard

#### Workstream 3: Public Booking Pages
- `/{user-slug}` — host profile with meeting type list
- `/{user-slug}/{meeting-type-slug}` — date/time selection
- Available slot computation in BFF
- Booking form (name, email, submit)
- Booking confirmation page with "Add to Calendar" links (.ics download, Google Calendar, Outlook web)
- Timezone auto-detection + manual override
- Mobile-responsive booking experience
- SSR for fast load + SEO
- Progressive enhancement: date selection works via form POST fallback (no-JS)

#### Workstream 4: Booking Management
- Bookings list (upcoming, past, cancelled tabs)
- Booking detail view
- Cancel booking (host-initiated, with email notification)
- Cancel/reschedule via invitee self-service tokens in email links
- Calendar auto-push (booking creates calendar event via sb_api)

### Phase 2: Polish & Advanced Settings
- Custom intake questions on meeting types
- Buffer times (before/after meetings)
- Booking limits (max per day)
- Date-specific availability overrides
- Booking window configuration
- Minimum scheduling notice
- Meeting type colors for calendar display

### Phase 3: Notifications & Experience
- Email notification configuration (sb_api template customization)
- Reminder emails (configurable timing)
- Host notification preferences
- Rate limiting and abuse prevention on booking pages
- Error handling and edge cases (expired slots, conflicts, etc.)
- Loading states, empty states, onboarding flow
- Accessibility audit: axe-core in CI, manual keyboard nav testing for DatePicker/TimeSlotPicker/AvailabilityEditor, screen reader testing for booking flow

### Phase 4: Team & Embed (v2)
- Team features: round-robin, collective meeting types
- User roles in sb_api (admin/member)
- Embed widget (inline, popup)
- i18n framework (next-intl, English-only initially)
- Custom domains
- Auto-generated video conferencing links (via sb_api)

### Dependency Map

```
Project Setup ─────┬──→ Host Dashboard ──→ Meeting Type CRUD ──┐
                   │                                           │
                   ├──→ Auth Flow ─────────────────────────────┤
                   │                                           │
                   └──→ Mock API ──→ Public Booking Pages ─────┤
                                                               │
                        Calendar Connection ──→ Calendar Sync ──┤
                                                               │
                                       Slot Computation ───────┤
                                                               │
                                              Booking Form ────┤
                                                               ▼
                                                    Booking Management
                                                    (view, cancel, reschedule)
```

### Tech Stack Summary

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Framework | Next.js 15 (App Router) | Server Components for booking pages, BFF pattern |
| UI Components | shadcn/ui | Accessible, Tailwind-native, customizable |
| Styling | Tailwind CSS v4 | Design token system, fast iteration |
| Server State | TanStack Query v5 | Caching, optimistic updates, query invalidation |
| Client State | React Context (minimal) | Only for UI state like timezone selection |
| Date/Time | date-fns + date-fns-tz | Lightweight, timezone-aware, tree-shakeable |
| Form Handling | React Hook Form + Zod | Validation, type safety |
| API Mocking | MSW (Mock Service Worker) | Development against sb_api contract |
| Testing | Vitest + Playwright | Unit + E2E |
| Deployment | Vercel (primary), Docker (alt) | Edge functions, preview deploys |
