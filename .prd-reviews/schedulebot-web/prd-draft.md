# PRD: Schedule Bot — Modern Web Frontend

## Problem Statement

The sb_api scheduling API exists but lacks a user-facing web interface. Users currently have no visual way to interact with scheduling functionality — creating, viewing, editing, and managing schedules. Without a frontend, the API's value is limited to developers who can integrate directly.

**Who:** End-users who need to manage schedules (employees, managers, administrators). The app will serve as one tenant of the sb_api system, with other apps also acting as tenants.

**Why now:** A clean, modern web frontend establishes the primary user experience for Schedule Bot and serves as the reference tenant implementation for sb_api.

## Goals

- Deliver a production-ready web frontend for sb_api scheduling operations
- Provide an intuitive, modern UX for schedule creation, viewing, editing, and management
- Establish Schedule Bot brand identity with a modern green-based color palette
- Build a reference single-tenant implementation that demonstrates sb_api integration patterns
- Achieve fast page loads and responsive design across desktop and mobile
- Follow accessibility best practices (WCAG 2.1 AA minimum)

## Non-Goals

- Multi-tenant architecture within this app (each tenant is a separate app)
- Native mobile apps (responsive web is sufficient for v1)
- Admin/super-admin panel for managing multiple tenants
- Real-time collaboration features (e.g., live co-editing of schedules)
- Offline support / PWA capabilities (defer to v2)
- Payment/billing integration
- Building a custom auth system (sb_api provides API key auth + OAuth2 flows for Google/Microsoft; Schedule Bot integrates these via BFF pattern)

## User Stories / Scenarios

### Actors
- **Employee**: Views their schedule, requests changes, marks availability
- **Manager**: Creates/edits schedules, assigns shifts, approves requests
- **Administrator**: Configures scheduling rules, manages teams/departments

### Core Stories

1. **As a manager**, I want to create a weekly schedule for my team so that everyone knows their shifts
2. **As an employee**, I want to view my upcoming schedule so I know when I'm working
3. **As an employee**, I want to request time off or shift swaps so I can manage my availability
4. **As a manager**, I want to see an overview of all team schedules so I can identify coverage gaps
5. **As an administrator**, I want to configure scheduling rules (max hours, required coverage) so schedules are compliant
6. **As a manager**, I want to receive notifications about schedule conflicts or requests so I can respond promptly
7. **As an employee**, I want to set my recurring availability preferences so managers can plan around them

### Key Scenarios

- **Schedule creation flow**: Manager selects date range → views team roster → drag-drops shifts onto a calendar grid → publishes schedule → team gets notified
- **Availability management**: Employee opens availability page → marks available/unavailable time blocks → submits → manager sees updated availability when creating schedules
- **Conflict resolution**: System detects double-booking or overtime → highlights conflicts → manager adjusts or overrides with reason

## Constraints

### Technical
- **Framework**: Next.js (App Router) with React 19+
- **Styling**: Tailwind CSS v4
- **API integration**: Must consume sb_api REST endpoints as a tenant (Go stdlib HTTP, JSON, `/api/v1/` prefix)
- **Deployment**: Should support Vercel, Docker, or similar modern hosting
- **Browser support**: Last 2 versions of Chrome, Firefox, Safari, Edge
- **Performance**: LCP < 2.5s, FID < 100ms, CLS < 0.1 (Core Web Vitals)

### Business
- Single-tenant: this app is Schedule Bot only; other tenants are separate apps
- Brand identity must feel professional yet approachable
- Must work for teams of 5–500 people

### Timeline
- Not specified — quality over speed

## Brand & Design Direction

### Brand Colors — Modern Green Palette

| Role | Color | Hex | Usage |
|------|-------|-----|-------|
| Primary | Emerald Green | `#10B981` | CTAs, active states, primary actions |
| Primary Dark | Deep Emerald | `#059669` | Hover states, emphasis |
| Primary Light | Mint | `#D1FAE5` | Backgrounds, highlights, success states |
| Secondary | Slate | `#475569` | Text, secondary UI elements |
| Accent | Teal | `#14B8A6` | Links, secondary actions, data viz |
| Background | Off-White | `#F8FAFC` | Page background |
| Surface | White | `#FFFFFF` | Cards, modals, elevated surfaces |
| Error | Rose | `#F43F5E` | Errors, destructive actions |
| Warning | Amber | `#F59E0B` | Warnings, conflicts |

### Design Principles
1. **Calendar-first**: The schedule view is the hero — prioritize calendar/grid UX
2. **Scannable**: Dense information presented clearly — color coding, visual hierarchy
3. **Responsive**: Mobile-first but optimized for desktop manager workflows
4. **Minimal chrome**: Let content breathe, reduce visual noise
5. **Accessible**: High contrast ratios, keyboard navigation, screen reader support

### Design Suggestions

**Layout approach**: Sidebar navigation (collapsible on mobile) + main content area. Top bar for user context, notifications, and quick actions.

**Calendar views**:
- Day / Week / Month toggle (like Google Calendar but purpose-built for shifts)
- Drag-and-drop shift assignment on the calendar grid
- Color-coded by role, department, or shift type

**Alternative consideration — Dashboard-first vs Calendar-first**:
A dashboard landing page with KPIs (coverage %, upcoming conflicts, pending requests) might serve managers better than jumping straight to the calendar. Recommend: dashboard as default for managers, calendar/schedule view as default for employees. Role-based default views.

**Component library approach**:
- Use Radix UI primitives (headless, accessible) + Tailwind for styling
- Alternatively: shadcn/ui (built on Radix + Tailwind) — provides pre-built, customizable components that match our stack perfectly
- Recommendation: **shadcn/ui** — fastest path to polished UI while maintaining full customization control

## sb_api Integration (from codebase analysis)

### Confirmed API Shape
- **REST API** (Go stdlib `net/http`, no GraphQL) at `/api/v1/`
- **Auth**: API key via `Authorization: Bearer sk_live_...` or `X-API-Key` header
- **User context**: `X-User-ID` (UUID) or `X-User-Slug` headers on user-scoped endpoints
- **Pagination**: offset-based, default 20, max 100

### Existing Endpoints
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/tenants` | None | Create tenant (returns API key) |
| GET | `/api/v1/tenant` | API key | Get current tenant |
| GET | `/api/v1/users` | API key | List users (paginated) |
| POST | `/api/v1/users` | API key | Create user |
| GET | `/api/v1/users/{id\|slug}` | API key | Get user by UUID or slug |
| PUT | `/api/v1/users/{id}` | API key | Update user |
| POST | `/api/v1/oauth/{google\|microsoft}` | API key + user | Start OAuth flow |
| GET | `/api/v1/oauth/{google\|microsoft}` | API key + user | OAuth callback |
| GET | `/health` | None | Health check |

### Existing Data Model
- **Tenants**: UUID, name, slug (unique), created_at
- **Users**: UUID, tenant_id, email, name, timezone (default UTC), slug, created_at
- **API Keys**: key_hash (SHA256), key_prefix, per-tenant
- **Calendar Connections**: user_id, provider (google/microsoft), encrypted tokens, read/write calendar IDs, sync status
- **OAuth Apps**: user_id, provider, encrypted client credentials

### Critical Finding: No Scheduling Entities Yet
**sb_api has no shift, schedule, team, department, or availability entities.** The API provides tenant/user/auth infrastructure and calendar OAuth, but the scheduling domain model must be designed and built alongside the frontend. This is a co-development effort, not just a frontend for an existing API.

### Tenant Model (Confirmed)
Schedule Bot is one tenant of a shared sb_api instance. Each tenant gets an isolated API key. All data is scoped by `tenant_id`. No cross-tenant data leakage — all queries enforce tenant isolation.

### Auth Model for Frontend
The BFF pattern works well here:
1. Schedule Bot backend holds the sb_api API key (server-side only)
2. Frontend authenticates users via OAuth (Google/Microsoft) through sb_api's OAuth endpoints
3. User identity passed via `X-User-ID`/`X-User-Slug` headers from BFF to sb_api
4. User roles/permissions need to be designed (not yet in sb_api)

### Timezone
Users have a `timezone` field (default UTC). Display in user-local timezone, store in UTC.

## Remaining Open Questions

1. **Scheduling data model**: What entities should sb_api add? (shifts, schedules, teams, departments, availability, rules?) This is a co-design effort.
2. **User roles/permissions**: sb_api has Users but no role field. Who defines Employee vs Manager vs Admin? sb_api or Schedule Bot?
3. **Real-time needs**: WebSocket/SSE for live schedule updates, or polling sufficient?
4. **Notification system**: In-app only, or also email/push? Where does notification delivery live?
5. **Conflict detection rules**: Overtime thresholds, double-booking, minimum rest periods — defined in sb_api or frontend?
6. **Calendar sync direction**: sb_api has calendar OAuth. Does it push shifts TO calendars, or is that Schedule Bot's job?
7. **Industry target**: Healthcare, retail, office, or general-purpose? Affects data model design.
8. **MVP team size**: Recommend 5-50 for v1. Confirm?
9. **i18n**: Needed from day one?
10. **Schedule lifecycle**: Draft → published → archived? Editable after publish?

## Rough Approach

### Architecture
```
Next.js App Router
├── app/
│   ├── (auth)/          # Login/auth pages
│   ├── (dashboard)/     # Manager dashboard
│   ├── schedule/        # Calendar/schedule views
│   ├── availability/    # Employee availability management
│   ├── team/            # Team/roster management
│   ├── settings/        # App settings, rules config
│   └── api/             # API routes (BFF pattern for sb_api)
├── components/
│   ├── ui/              # shadcn/ui base components
│   ├── schedule/        # Schedule-specific components
│   └── layout/          # Shell, sidebar, navigation
├── lib/
│   ├── api/             # sb_api client SDK/hooks
│   ├── auth/            # Auth utilities
│   └── utils/           # Shared utilities
└── styles/
    └── globals.css      # Tailwind config, custom properties
```

### Key Technical Decisions
1. **Next.js App Router** with Server Components for data fetching, Client Components for interactivity
2. **shadcn/ui** for component primitives — accessible, customizable, Tailwind-native
3. **React Query (TanStack Query)** for server state management and caching
4. **Zustand** for minimal client state (UI state, user preferences)
5. **BFF pattern**: Next.js API routes proxy to sb_api — keeps API keys server-side, enables response transformation
6. **date-fns** or **Temporal API** for date/time handling (scheduling is date-heavy)

### Phasing (suggested)
- **Phase 1**: Auth + Schedule viewing (read-only calendar, employee view)
- **Phase 2**: Schedule creation + editing (manager workflow, drag-drop)
- **Phase 3**: Availability management + conflict detection
- **Phase 4**: Dashboard, notifications, rules configuration
