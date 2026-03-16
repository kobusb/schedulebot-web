# PRD Review: Ambiguity Analysis

## Summary
The PRD contains several terms and concepts that are used inconsistently or without definition, creating risk of misaligned implementation.

## Findings

### CRITICAL

1. **"Tenant" is ambiguous**: The PRD says "this frontend will be one tenant" and "other apps that act as a tenant." Does this mean:
   - (a) Schedule Bot has its own isolated sb_api instance?
   - (b) Schedule Bot shares an sb_api instance with other apps, separated by tenant ID?
   - (c) Each deployment of Schedule Bot IS a tenant (multi-instance single-tenant)?
   This affects architecture (shared vs isolated data), auth, and API integration.

2. **"Schedule" means multiple things**: Sometimes it's a single shift assignment, sometimes it's a weekly plan for a team, sometimes it's an employee's personal view of their shifts. Need distinct terms: Shift (single time block), Schedule (collection of shifts for a period), Roster (team assignment view).

### MAJOR

3. **"Modern" is subjective**: Used 3 times without definition. Modern could mean: latest framework versions, contemporary design patterns, specific visual aesthetic, or cutting-edge features. Needs grounding in specific attributes.

4. **REST or GraphQL — which?**: Constraint says "REST/GraphQL endpoints" — these are architecturally different choices. The BFF pattern works differently for each. Need to decide or design for both.

5. **"Best practices" — whose?**: The prompt says "follow best practices" but doesn't specify whose. Next.js best practices? React best practices? UX best practices for scheduling apps? Industry-specific compliance best practices?

6. **Manager vs Administrator boundary is unclear**: Both roles can "configure" things. What's the privilege boundary? Can a manager create scheduling rules, or only an admin? Can an admin also create schedules?

### MINOR

7. **"Professional yet approachable" brand tone**: These can conflict. Professional implies formal, restrained. Approachable implies warm, casual. Need examples or references of what this looks like.

8. **Phase ordering rationale not explained**: Why is auth + read-only first? If managers can't create schedules, what are employees viewing? Might need seed data or a demo mode for Phase 1 to be useful.

9. **"Reference tenant implementation"**: This implies other tenants will copy this pattern. That's a different bar than "build a scheduling app" — it means the code should be exemplary and well-documented. Not mentioned in goals.

## Verdict
**The tenant model and schedule terminology ambiguity are blocking.** These affect every architectural decision downstream. The REST/GraphQL choice also needs resolution before implementation begins.
