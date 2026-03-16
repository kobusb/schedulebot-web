# PRD Alignment Round 2: Constraints + Non-Goals

## A. Constraints Compliance

### Technical Constraints

| Constraint | Status | Notes |
|-----------|--------|-------|
| Next.js (App Router) with React 19+ | RESPECTED | Plan explicitly uses Next.js 15 App Router throughout |
| Tailwind CSS v4 | RESPECTED | In Sprint 0 setup + tech stack summary |
| sb_api REST endpoints (Go stdlib, JSON, /api/v1/) | RESPECTED | BFF proxies to sb_api REST. All endpoint designs match REST convention |
| Deployment: Vercel, Docker, or similar | RESPECTED | Tech stack: "Vercel (primary), Docker (alt)" |
| Browser support: last 2 versions | RESPECTED | No exotic browser APIs used. shadcn/ui handles compatibility |
| Core Web Vitals (LCP < 2.5s, FID < 100ms, CLS < 0.1) | RESPECTED | Scale design sets even tighter targets for booking pages (LCP < 1.5s) |
| Booking pages must be fast | RESPECTED | Server Components, minimal JS, performance targets |

### Business Constraints

| Constraint | Status | Notes |
|-----------|--------|-------|
| Single-tenant (Schedule Bot only) | RESPECTED | No multi-tenant logic in plan |
| Professional yet approachable brand | RESPECTED | Color palette defined, design tokens in Sprint 0. Brand deliverables added in round 1 |
| Booking pages work without JS | RESPECTED | Progressive enhancement added in round 1. Form POST fallback for date selection |
| Invitees must NOT need an account | RESPECTED | Public routes, no auth on booking pages. Invitee data collected only in booking form |

### Timeline Constraint

| Constraint | Status | Notes |
|-----------|--------|-------|
| Quality over speed | RESPECTED | Plan has 4 phases, no artificial deadlines. Accessibility audit included |

**No constraint violations found.**

---

## B. Non-Goals Enforcement

### PRD Non-Goals Checklist

| Non-Goal | Status | Plan Sections Checked |
|----------|--------|----------------------|
| Multi-tenant architecture | CLEAN | Plan is single-tenant throughout. BFF holds one API key. No tenant switching UI |
| Native mobile apps | CLEAN | Only responsive web. No React Native, Capacitor, or native mentions |
| Building custom auth system | CLEAN | Plan uses sb_api's OAuth endpoints. BFF session management is integration, not a custom auth system |
| Payment collection for paid meetings | CLEAN | No payment mentions anywhere in plan |
| CRM integrations | CLEAN | No CRM mentions in any phase |
| SMS/WhatsApp notifications | CLEAN | Email only via sb_api. No SMS/messaging mentions |
| Offline support / PWA | CLEAN | No service workers, offline caching, or PWA manifest |
| Video conferencing hosting | CLEAN | Plan uses paste-in links for v1, sb_api auto-generate for v2. No video infrastructure |

### Scope Creep Check

| Plan Section | Status | Notes |
|-------------|--------|-------|
| Phase 1: Core Booking Flow | CLEAN | Essential MVP features only |
| Phase 2: Polish & Advanced Settings | CLEAN | Buffer times, booking limits, overrides — all in v1 scope table |
| Phase 3: Notifications & Experience | CLEAN | Email config via sb_api, error handling, a11y — all appropriate |
| Phase 4: Team & Embed (v2) | CLEAN | Correctly labeled v2, matches scope table |
| Security design: CAPTCHA | BORDERLINE | PRD doesn't mention CAPTCHA. However, rate limiting + abuse prevention is a reasonable security measure for public booking pages. **Recommendation**: Keep, but mark as "if abuse detected" (already conditional in plan) | **should-fix: clarify** |
| Security design: Email verification | BORDERLINE | "confirm booking" link is an anti-spam measure, not a user story. PRD says "book in a few clicks" — adding email verification adds friction. **Recommendation**: Keep optional per meeting type, but default to OFF to match PRD's frictionless booking intent | **should-fix** |

---

## C. Fixes Applied

### Should-Fix

**1. Email verification default**: Changed from ambiguous to explicitly optional, default OFF. PRD emphasizes frictionless booking — requiring email verification contradicts this unless spam is a problem.

**2. CAPTCHA clarification**: Already conditional ("if abuse detected") — no change needed, just confirmed intent.

**No must-fix items.** The plan is well-constrained.
