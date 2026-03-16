# PRD Review: Missing Requirements

## Summary
Several significant capability areas are absent from the PRD that users of a scheduling tool would expect.

## Findings

### CRITICAL

1. **No timezone handling strategy**: Scheduling apps are inherently timezone-sensitive. Teams may span timezones. Shifts displayed in whose timezone? How are timezone transitions (DST) handled? This is a correctness issue, not a nice-to-have.

2. **No state management for schedule lifecycle**: Schedules have states (draft → published → archived). The PRD describes creation and viewing but not the lifecycle. Can a published schedule be edited? What happens to notifications when a published schedule changes?

3. **No conflict detection rules defined**: The PRD mentions "conflict resolution" in scenarios but doesn't define what constitutes a conflict. Overtime thresholds? Double-booking across departments? Minimum rest periods between shifts?

### MAJOR

4. **No onboarding/empty state design**: What does a brand new team see? No schedules, no employees configured. The empty state IS the first impression and needs design attention.

5. **No search/filter requirements**: For a 500-person team, finding a specific employee's schedule or filtering by department/role is essential. Not mentioned.

6. **No print/share capability**: Managers in retail, healthcare, and food service frequently print weekly schedules and post them physically. PDF generation or print-optimized views are expected.

7. **No recurring schedule/template support**: Most teams have repeating patterns (e.g., 2-week rotation). Templates save massive time. This is a core differentiator for scheduling tools.

8. **No bulk operations**: Assigning shifts one by one for 50+ people is unusable. Need bulk assign, bulk edit, copy-week-to-week.

### MINOR

9. **No keyboard shortcut strategy**: For power users creating complex schedules, keyboard navigation and shortcuts dramatically improve efficiency.

10. **No dark mode consideration**: Modern web apps increasingly support dark mode. Not required for v1 but worth acknowledging as a non-goal if intentional.

11. **No loading/skeleton state design**: Calendar views with many entries need thoughtful loading states to avoid layout shift.

12. **No mobile-specific interaction patterns**: "Responsive" is mentioned but drag-drop on mobile is fundamentally different from desktop. Need touch-specific interaction design for schedule editing.

## Verdict
**Multiple expected capabilities are missing.** Timezone handling and schedule lifecycle are the most critical gaps — both affect data correctness. Template/recurring schedules and bulk operations are the most impactful UX gaps.
