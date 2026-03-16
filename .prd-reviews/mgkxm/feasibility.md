# PRD Review: Technical Feasibility

## Summary
The tech stack choices are sound and well-matched. The main feasibility risks are in the calendar/drag-drop UX complexity and the undefined API contract.

## Findings

### CRITICAL

1. **Calendar drag-drop at scale is hard**: Building a performant drag-and-drop calendar grid for 50+ employees across a week is one of the hardest frontend UX challenges. Existing libraries (react-big-calendar, FullCalendar) each have trade-offs:
   - **react-big-calendar**: React-native, good customization, but drag-drop for shift assignment (not just event moving) requires significant custom work
   - **FullCalendar**: Most mature, but React wrapper is a thin skin over imperative DOM — fights React's model
   - **Custom build**: Full control but 4-8 weeks of dedicated calendar work
   - **Recommendation**: Evaluate @dnd-kit + custom grid vs FullCalendar React wrapper. Prototype both before committing.

2. **No API contract = can't build the BFF**: The BFF pattern requires knowing the upstream API shape. Without sb_api docs, we'd be designing the BFF interface speculatively. Risk: significant rework when the real API is integrated.

### MAJOR

3. **React 19 + Next.js App Router maturity**: While both are production-ready, the Server Components model has sharp edges for highly interactive UIs like calendar drag-drop. All interactive calendar components will be Client Components, potentially negating some App Router benefits. Not a blocker, but the architecture sketch should acknowledge this.

4. **shadcn/ui doesn't include calendar/scheduling components**: shadcn/ui provides buttons, dialogs, forms — general UI. The scheduling-specific components (calendar grid, shift cards, timeline view, drag-drop zones) must be custom-built. This is the majority of the unique UI work.

5. **Zustand may be insufficient**: With schedule state (current view, selected date range, draft edits, undo history, conflict highlights), the client state is more complex than "minimal." Consider whether TanStack Query's cache + React context is enough, or if a more structured state solution is needed.

### MINOR

6. **Temporal API browser support**: The Temporal API is still Stage 3 and not shipped in any browser without polyfill. date-fns is the safer choice for now. Temporal adds ~40KB polyfill.

7. **Tailwind v4 is very new**: Released early 2025. Some ecosystem tools (IDE plugins, PostCSS configs) may have rough edges. v3 is battle-tested. Consider whether v4's benefits (CSS-first config, faster builds) outweigh the ecosystem risk.

8. **Print/PDF generation from Next.js**: If print capability is added, server-side PDF generation (puppeteer, @react-pdf/renderer) adds infrastructure complexity. Worth flagging for Phase 4.

## Verdict
**Stack is feasible but the calendar UX is the technical crux.** Prototype the drag-drop calendar grid early — it's the make-or-break component. The API contract gap is a process risk, not a technical one.
