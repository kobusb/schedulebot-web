# PRD: Schedule Bot — Modern Web Frontend

## Problem Statement

sb_api provides a Calendly-style appointment scheduling backend, but lacks a user-facing web interface. Schedule Bot is the primary frontend tenant — a booking and availability management tool where hosts set their availability, create meeting types, and share scheduling links so invitees can self-book appointments.

**Who:** Professionals and teams who need to let others book time with them — consultants, sales teams, recruiters, support teams, freelancers, and anyone who currently plays email tag to schedule meetings.

**Why now:** The sb_api infrastructure (tenants, users, OAuth calendar integration) is in place. Schedule Bot establishes the primary user experience and serves as the reference tenant implementation.

## Goals

- Deliver a production-ready Calendly-style scheduling frontend for sb_api
- Provide intuitive booking pages where invitees can self-schedule appointments
- Enable hosts to manage availability, meeting types, and booking settings
- Integrate with Google and Microsoft calendars (via sb_api's existing OAuth)
- Establish Schedule Bot brand identity with a modern green palette
- Build a reference single-tenant implementation demonstrating sb_api patterns
- Achieve fast page loads and responsive design across desktop and mobile
- Follow accessibility best practices (WCAG 2.1 AA minimum)

## Non-Goals

- Multi-tenant architecture within this app (each tenant is a separate app)
- Native mobile apps (responsive web is sufficient for v1)
- Building a custom auth system (sb_api provides API key auth + OAuth2 for Google/Microsoft)
- Payment collection for paid meetings (defer to v2)
- CRM integrations (Salesforce, HubSpot — defer to v2)
- SMS/WhatsApp notifications (defer to v2)
- Offline support / PWA capabilities (defer to v2)
- Video conferencing hosting (integrate with Zoom/Meet/Teams links, don't build one)

## Domain Model

### Core Concepts (Calendly-aligned)

**Meeting Types** (Calendly calls these "Event Types")
The templates that define what can be booked:
- Name, description, slug (for booking URL)
- Duration (15m, 30m, 45m, 60m, custom)
- Location type: in-person, phone, video (Zoom/Meet/Teams link), custom
- Booking window: how far in advance can someone book? (e.g., 1-60 days out)
- Minimum scheduling notice: how soon before a slot can someone book? (e.g., 4 hours)
- Buffer time: padding before/after meetings (e.g., 15m between back-to-back)
- Max bookings per day (optional cap)
- Custom intake questions (text, dropdown, checkbox — collected from invitee at booking)
- Confirmation/reminder settings
- Active/inactive toggle
- Color coding for calendar display

**Meeting Type Variants:**
- **One-on-One**: Single host, single invitee (default)
- **Group**: One host, multiple invitees book the same slot (e.g., webinar, class)
- **Round-Robin**: Multiple hosts, system assigns based on availability/fairness
- **Collective**: Multiple hosts must ALL be available (team interview, panel meeting)

**Availability**
When a host can accept bookings:
- Weekly recurring schedule (e.g., Mon-Fri 9am-5pm)
- Multiple schedule profiles (e.g., "Weekday hours", "Weekend office hours")
- Date-specific overrides (e.g., "Dec 25 — unavailable", "March 10 — 10am-2pm only")
- Calendar conflict checking: automatically block times with existing calendar events
- Timezone-aware: host sets their timezone, invitee sees slots in their own timezone

**Bookings** (Calendly calls these "Events")
Actual scheduled appointments:
- Meeting type reference
- Host(s) and invitee(s)
- Start time, end time, timezone
- Location details (video link, address, phone number)
- Status: confirmed, cancelled, rescheduled
- Invitee responses to custom intake questions
- Calendar event IDs (synced to host's connected calendar)
- Cancellation/reschedule policy and reason

**Scheduling Pages**
Public-facing booking interfaces:
- Personal scheduling page: `schedulebot.app/{user-slug}` — lists all active meeting types
- Direct meeting type link: `schedulebot.app/{user-slug}/{meeting-type-slug}` — jumps to time selection
- Embeddable widget (inline, popup, or popup text)

**Workflows** (v2 — but design for extensibility)
Automated actions triggered by booking events:
- Confirmation email to invitee
- Reminder email/notification before meeting
- Follow-up email after meeting
- Custom webhook triggers

### Data Model Relationship

```
Tenant (sb_api)
└── User (host)
    ├── Availability Schedules
    │   ├── Weekly Rules (recurring)
    │   └── Date Overrides (specific dates)
    ├── Meeting Types
    │   ├── Custom Questions
    │   └── Settings (buffer, notice, window, max/day)
    ├── Bookings
    │   ├── Invitee(s)
    │   ├── Answers to custom questions
    │   └── Calendar sync status
    └── Calendar Connections (sb_api — already exists)
        ├── Google Calendar
        └── Microsoft Calendar
```

## User Stories / Scenarios

### Actors
- **Host**: Sets availability, creates meeting types, manages bookings
- **Invitee**: Books time with a host via a public scheduling page (no account required)
- **Team Admin**: Manages team members, round-robin pools, collective meeting types

### Core Stories — Host

1. **As a host**, I want to connect my Google/Microsoft calendar so Schedule Bot knows when I'm busy
2. **As a host**, I want to set my weekly availability hours so invitees can only book during those times
3. **As a host**, I want to create meeting types (e.g., "30 min intro call", "60 min consultation") so invitees can pick the right meeting
4. **As a host**, I want to share my scheduling link so people can book time without email back-and-forth
5. **As a host**, I want to see all my upcoming bookings in one place so I can prepare
6. **As a host**, I want to cancel or reschedule a booking and have the invitee notified automatically
7. **As a host**, I want to add buffer time between meetings so I'm not booked back-to-back
8. **As a host**, I want to set date-specific overrides (holidays, special hours) on my availability
9. **As a host**, I want to add custom intake questions to a meeting type so I get context before the meeting

### Core Stories — Invitee

10. **As an invitee**, I want to see available time slots in my own timezone so I don't have to convert times
11. **As an invitee**, I want to book a meeting in a few clicks without creating an account
12. **As an invitee**, I want to receive a confirmation with calendar invite and video link
13. **As an invitee**, I want to cancel or reschedule my booking via a link in the confirmation email

### Core Stories — Team

14. **As a team admin**, I want to set up round-robin meeting types so leads are distributed fairly across my team
15. **As a team admin**, I want to create collective meeting types so all required people must be available

### Key Scenarios

- **Booking flow**: Invitee clicks scheduling link → selects meeting type → sees available time slots (filtered by host availability + calendar conflicts) → picks a slot → fills in name, email, custom questions → confirms → both parties get calendar invite
- **Availability management**: Host opens settings → sets Mon-Fri 9-5 as base hours → adds override for Dec 25 (unavailable) → connects Google Calendar → busy times automatically blocked
- **Reschedule flow**: Invitee clicks reschedule link → sees new available slots → picks new time → old booking cancelled, new one created → both parties notified

## Constraints

### Technical
- **Framework**: Next.js (App Router) with React 19+
- **Styling**: Tailwind CSS v4
- **API integration**: Must consume sb_api REST endpoints as a tenant (Go stdlib HTTP, JSON, `/api/v1/` prefix)
- **Deployment**: Should support Vercel, Docker, or similar modern hosting
- **Browser support**: Last 2 versions of Chrome, Firefox, Safari, Edge
- **Performance**: LCP < 2.5s, FID < 100ms, CLS < 0.1 (Core Web Vitals)
- **Booking pages must be fast**: These are public-facing — invitees judge the product by booking page speed

### Business
- Single-tenant: Schedule Bot is one tenant of sb_api
- Brand identity must feel professional yet approachable
- Booking pages must work without JavaScript for basic functionality (progressive enhancement)
- Invitees must NOT need an account to book

### Timeline
- Not specified — quality over speed

## sb_api Integration (from codebase analysis)

### Existing Infrastructure
- **REST API** at `/api/v1/`, Go stdlib `net/http`
- **Multi-tenant** via API keys (`Authorization: Bearer sk_live_...`)
- **Users**: UUID, tenant_id, email, name, timezone, slug
- **Calendar OAuth**: Google and Microsoft, encrypted token storage, sync status tracking
- **Pagination**: offset-based, default 20, max 100

### Needs to be Built in sb_api (co-development)
The following entities do NOT yet exist in sb_api and must be designed:
- Availability schedules (weekly rules + date overrides)
- Meeting types (with settings, custom questions)
- Bookings (with invitee info, status, calendar sync)
- Team/round-robin/collective configurations (v2)
- Notification/workflow triggers (v2)

### Auth Model for Frontend
1. Schedule Bot backend (BFF) holds the sb_api API key (server-side only)
2. Hosts authenticate via OAuth (Google/Microsoft) through sb_api's OAuth endpoints
3. User identity passed via `X-User-ID`/`X-User-Slug` headers from BFF to sb_api
4. Invitees do NOT authenticate — booking pages are public

## Brand & Design Direction

### Brand Colors — Modern Green Palette

| Role | Color | Hex | Usage |
|------|-------|-----|-------|
| Primary | Emerald Green | `#10B981` | CTAs, active states, primary actions |
| Primary Dark | Deep Emerald | `#059669` | Hover states, emphasis |
| Primary Light | Mint | `#D1FAE5` | Backgrounds, highlights, success states |
| Secondary | Slate | `#475569` | Text, secondary UI elements |
| Accent | Teal | `#14B8A6` | Links, secondary actions, time slot highlights |
| Background | Off-White | `#F8FAFC` | Page background |
| Surface | White | `#FFFFFF` | Cards, modals, elevated surfaces |
| Error | Rose | `#F43F5E` | Errors, destructive actions, cancelled bookings |
| Warning | Amber | `#F59E0B` | Warnings, scheduling conflicts |

### Design Principles
1. **Booking-first**: The invitee booking experience is the hero — it must be frictionless
2. **Clean and trustworthy**: Invitees are landing on a stranger's scheduling page — it must feel safe and professional
3. **Scannable time slots**: Available times presented clearly with timezone context
4. **Responsive**: Booking pages must be excellent on mobile (many invitees book from phones)
5. **Accessible**: High contrast ratios, keyboard navigation, screen reader support

### Design Suggestions

**Two distinct UX surfaces:**

1. **Host Dashboard** (authenticated, complex):
   - Sidebar navigation: Bookings, Meeting Types, Availability, Integrations, Settings
   - Calendar view of upcoming bookings
   - Quick-copy scheduling links
   - Meeting type management cards

2. **Booking Pages** (public, simple):
   - Minimal, focused UI — meeting type info + time slot picker
   - Two-step: select time → enter details → confirm
   - Host's photo/name/branding for trust
   - Timezone auto-detection with manual override
   - Mobile-optimized time slot grid

**Component library**: shadcn/ui (Radix + Tailwind) for host dashboard. Booking pages should be lightweight — possibly Server Components with minimal client JS.

## Clarifications from Human Review

**Q: Does the proposed entity structure (Meeting Types, Availability, Bookings) match your vision?**
A: Yes, confirmed.

**Q: Team support (round-robin, collective) — need roles in sb_api now?**
A: Pushed to v2. No role field needed in sb_api for v1.

**Q: Custom domain support for booking URLs?**
A: Pushed to v2. V1 uses default domain `/{slug}/{meeting-type}`.

**Q: Video conferencing links — auto-generate or paste-in?**
A: Auto-generate (Zoom/Meet/Teams). This will be provided by a future sb_api version. V1 of Schedule Bot should design for this but may use paste-in as interim.

**Q: Notification delivery — sb_api or Schedule Bot?**
A: sb_api sends emails. sb_api should provide template customization so tenant apps can customize email content/branding.

**Q: Calendar sync — auto-push bookings to host's calendar?**
A: Yes, auto-push. Bookings should automatically create calendar events on the host's connected calendar.

**Q: i18n from day one?**
A: Pushed to v2, unless deferring would complicate things later. (Note: recommend structuring string extraction from day one to avoid painful retrofit — use next-intl or similar with English-only initially.)

**Q: Embed widget priority?**
A: v2 or v3. Nice to have but not critical for launch.

## v1 vs v2+ Scope Summary

| Feature | v1 | v2+ |
|---------|----|----|
| One-on-one meeting types | Yes | |
| Availability (weekly + overrides) | Yes | |
| Public booking pages | Yes | |
| Calendar sync (auto-push) | Yes | |
| Booking management (view/cancel/reschedule) | Yes | |
| Email notifications (via sb_api) | Yes | |
| Custom intake questions | Yes | |
| Buffer times, booking limits | Yes | |
| Team features (round-robin, collective) | | v2 |
| Auto-generate video links (sb_api) | | v2 |
| Custom domains | | v2 |
| i18n | | v2 |
| Embed widget | | v2-v3 |
| CRM integrations | | v2+ |
| Payment collection | | v2+ |
| Workflows/automations | | v2+ |
| SMS/WhatsApp notifications | | v2+ |

## Rough Approach

### Architecture
```
Next.js App Router
├── app/
│   ├── (auth)/              # Host login via OAuth
│   ├── (dashboard)/         # Host management UI
│   │   ├── bookings/        # Booking list + calendar view
│   │   ├── meeting-types/   # Create/edit meeting types
│   │   ├── availability/    # Set weekly hours + overrides
│   │   ├── integrations/    # Calendar connections
│   │   └── settings/        # Profile, branding, preferences
│   ├── [user-slug]/         # Public scheduling pages
│   │   ├── page.tsx         # List of meeting types
│   │   └── [meeting-type]/  # Time slot picker + booking form
│   └── api/                 # BFF routes proxying to sb_api
├── components/
│   ├── ui/                  # shadcn/ui base components
│   ├── booking/             # Booking page components (lightweight)
│   ├── dashboard/           # Host dashboard components
│   └── layout/              # Shell, sidebar, navigation
├── lib/
│   ├── api/                 # sb_api client SDK/hooks
│   ├── auth/                # OAuth utilities
│   ├── availability/        # Availability calculation logic
│   └── timezone/            # Timezone utilities
└── styles/
    └── globals.css          # Tailwind config, custom properties
```

### Key Technical Decisions
1. **Next.js App Router** with Server Components for booking pages (fast, SEO-friendly), Client Components for dashboard interactivity
2. **shadcn/ui** for host dashboard components
3. **Booking pages as Server Components**: Minimal JS for fastest possible load. Time slot selection can hydrate on interaction.
4. **TanStack Query** for server state management in dashboard
5. **BFF pattern**: Next.js API routes proxy to sb_api — keeps API key server-side
6. **date-fns-tz** for timezone-aware date handling (critical for scheduling across timezones)
7. **Timezone auto-detection** via `Intl.DateTimeFormat().resolvedOptions().timeZone`

### Phasing (revised)
- **Phase 1**: Host onboarding + availability + basic one-on-one meeting types + public booking pages
- **Phase 2**: Booking management (view, cancel, reschedule) + calendar sync + notifications
- **Phase 3**: Custom intake questions, buffer times, booking limits, date overrides
- **Phase 4**: Team features (round-robin, collective), embed widget, workflows
