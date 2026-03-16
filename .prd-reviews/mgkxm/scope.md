# PRD Review: Scope Analysis

## Summary
The PRD scope is ambitious for a v1. The phasing helps, but Phase 2 alone (drag-drop schedule creation) is a major engineering effort. MVP definition needs tightening.

## Findings

### CRITICAL

1. **Phase 1 has no standalone value**: Auth + read-only calendar viewing requires schedules to already exist. But Phase 2 (creation) comes later. Options:
   - (a) Allow API-created schedules to be viewed (Phase 1 is a viewer for data created elsewhere)
   - (b) Include basic schedule creation in Phase 1 (form-based, not drag-drop)
   - (c) Provide demo/seed data
   Without one of these, Phase 1 ships an empty app.

2. **Drag-drop schedule creation should be Phase 3, not Phase 2**: Form-based schedule creation (select employee, select time, save) is much simpler and provides core value. Drag-drop is a UX enhancement that can layer on top. Suggested reordering:
   - Phase 1: Auth + Schedule viewing + basic form-based creation
   - Phase 2: Availability management + conflict detection
   - Phase 3: Drag-drop calendar UX (enhancement over forms)
   - Phase 4: Dashboard, notifications, rules, templates

### MAJOR

3. **500-person team support is a Phase 4+ concern**: The UX for 500-person teams requires virtualization, search, filters, department views — none of which are in Phase 1-2. Define the MVP team size (e.g., 5-50 people) and add large-team support later.

4. **WCAG 2.1 AA from day one is costly with drag-drop**: Accessible drag-and-drop is notoriously difficult. If we defer drag-drop to Phase 3, accessibility is much simpler for Phase 1-2.

5. **Three distinct role-based experiences**: Employee, Manager, and Administrator views are essentially three mini-apps. For MVP, consider starting with Manager + Employee only. Administrator features (rules configuration) can defer.

### MINOR

6. **Brand identity work is parallel, not blocking**: Color palette, typography, and design tokens can be established in Sprint 0 while architecture is set up. Don't let design exploration block engineering start.

7. **"Reference tenant implementation" adds scope**: If the goal is a reference implementation, we need documentation, clean abstractions, and example patterns. This should be a separate phase or non-goal for v1.

## Scope Creep Risks

| Risk | Trigger | Mitigation |
|------|---------|------------|
| Calendar library rabbit hole | Evaluating 5 libraries, building custom | Time-box to 3 days, pick best-available |
| Design perfection | Iterating on pixel-perfect calendar views | Ship functional first, polish in v2 |
| API speculation | Building BFF for imagined API shape | Mock API with MSW, adapt when real API arrives |
| Feature parity with competitors | "But When I Work has..." | Stick to defined user stories, file enhancement beads |

## Verdict
**Scope is achievable if phases are reordered and MVP team size is capped.** The biggest risk is Phase 1 having no standalone value and Phase 2 being too ambitious. Recommend form-based creation in Phase 1, drag-drop as a later enhancement.
