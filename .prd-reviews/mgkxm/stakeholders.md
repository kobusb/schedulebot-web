# PRD Review: Stakeholder Analysis

## Summary
The PRD identifies three user roles well but misses several stakeholders who will influence requirements and adoption.

## Findings

### CRITICAL

1. **sb_api team is an unacknowledged stakeholder**: Schedule Bot is a tenant of sb_api. The sb_api team controls the API contract, auth model, tenant model, and data model. Their roadmap and priorities directly constrain Schedule Bot. They should be consulted on:
   - API shape and capabilities
   - Auth/tenant model decisions
   - Rate limits and data access patterns
   - Breaking change notification process

2. **Other tenant app teams**: If Schedule Bot is the "reference implementation," other tenant teams are stakeholders who will pattern-match against this codebase. Their needs (documentation quality, abstraction patterns, tenant setup guide) aren't captured.

### MAJOR

3. **Payroll/HR integration consumers**: Schedule data frequently feeds into payroll systems. Even if integration is out of scope, the data model and export formats should accommodate this downstream need. Payroll teams should be consulted on required fields (hours worked, overtime classification, department codes).

4. **Compliance/Legal**: Scheduling apps in many industries must comply with labor laws (predictive scheduling laws, rest period requirements, overtime rules). The PRD mentions "scheduling rules" but doesn't acknowledge the legal compliance angle. In healthcare, retail, and food service, this is non-negotiable.

5. **IT/Operations**: Who deploys and maintains this? Monitoring, logging, incident response, backup strategy, security patching. DevOps requirements are absent.

### MINOR

6. **End-user representatives**: The PRD defines three roles (Employee, Manager, Admin) but doesn't mention user research or representative users. Scheduling needs vary dramatically by industry:
   - Healthcare: 12-hour shifts, on-call rotations, credential matching
   - Retail: variable hours, split shifts, seasonal scaling
   - Office: mostly fixed schedules, meeting room booking crossover
   Which industry is the primary target?

7. **Support/helpdesk team**: Who handles user issues? Is there a support contact, help documentation, or in-app help? Support team needs visibility into the product.

8. **Accessibility reviewers**: WCAG 2.1 AA is stated as a goal. Who validates compliance? Automated tools catch ~30% of issues — manual review by accessibility experts is needed.

## Conflicting Needs

| Stakeholder A | Stakeholder B | Conflict |
|---------------|---------------|----------|
| Manager (wants flexibility) | Compliance (wants constraints) | Scheduling rules may limit manager freedom |
| Employee (wants simplicity) | Manager (wants power features) | Feature density vs. clean employee UX |
| sb_api team (API stability) | Schedule Bot (feature velocity) | New frontend features may require API changes |
| Reference tenant consumers (clean code) | Schedule Bot users (ship fast) | Polish vs. speed |

## Verdict
**The sb_api team is the most critical missing stakeholder.** Their decisions on API shape, auth, and tenant model are hard dependencies. Industry targeting (healthcare vs. retail vs. office) would dramatically sharpen the user stories and conflict detection rules.
