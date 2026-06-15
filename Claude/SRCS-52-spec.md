# SRCS-52 — [MOB|CA] Standardize Appointment Selection Window to 7 Days for Create & Update Appointment Flows

> **Branch:** `feature/SRCS-52-standardize-appointment-selection-7-days`
> **Priority:** `High`
> **Labels:** <!-- No data found — fill in manually -->
> **Jira:** https://carsome.atlassian.net/browse/SRCS-52

---

## Overview

Currently, the appointment date selection behavior is inconsistent between the Create Appointment and Update/Reschedule Appointment flows. To provide a consistent user experience and align with business rules, the appointment selection lead time should be standardized across all appointment booking journeys.

The change applies to the following screens in the CARagent App:

| Module | Screen | Action |
| --- | --- | --- |
| CARagent App | Create Appointment | Date Selection |
| CARagent App | Reschedule Appointment | Date Selection |

---

## Goals

Ensure users can only select appointment dates within the next **7 calendar days** from the current date, regardless of whether they are creating a new appointment or rescheduling an existing one. This enforces a consistent date selection rule across all appointment booking journeys and aligns the app with the business rule on appointment lead time.

---

## Functional Requirements

- The appointment date calendar must restrict selectable dates to **Today** through **Today + 7 calendar days** (inclusive) across all flows: Create, Edit, and Reschedule.
- Dates beyond Today + 7 calendar days must be displayed as **disabled** and must not be selectable.
- The 7-day window must apply consistently regardless of appointment type (Inbound or Outbound).
- An existing appointment that falls outside the new 7-day window must remain viewable, but the user may only reschedule it to a date within the next 7 calendar days.
- If a date outside the allowed window is selected through unexpected client-side behavior, the system must block the action and display an appropriate validation message.

---

## Acceptance Criteria

| Scenario | Acceptance Criteria |
| --- | --- |
| Create Appointment — Display Available Dates | Given a user is creating a new appointment, when the appointment calendar is opened, then the user should be able to select dates from Today up to Today + 7 calendar days only. |
| Create Appointment — Disable Dates Beyond 7 Days | Given a user is creating a new appointment, when the user views dates beyond Today + 7 calendar days, then those dates should be displayed as disabled and should not be selectable. |
| Edit Appointment — Display Available Dates | Given a user is editing an existing appointment, when the appointment calendar is opened, then the user should be able to select dates from Today up to Today + 7 calendar days only. |
| Edit Appointment — Disable Dates Beyond 7 Days | Given a user is editing an existing appointment, when the user views dates beyond Today + 7 calendar days, then those dates should be displayed as disabled and should not be selectable. |
| Reschedule Appointment — Display Available Dates | Given a user is rescheduling an existing appointment, when the appointment calendar is opened, then the user should be able to select dates from Today up to Today + 7 calendar days only. |
| Reschedule Appointment — Disable Dates Beyond 7 Days | Given a user is rescheduling an existing appointment, when the user views dates beyond Today + 7 calendar days, then those dates should be displayed as disabled and should not be selectable. |
| Consistent Date Selection Logic Across Flows | Given a user accesses the Create, Edit, or Reschedule Appointment flow, when the appointment calendar is displayed, then the same 7-day appointment selection window should be applied across all flows. |
| Existing Appointment Outside New Window | Given an appointment already exists outside the new 7-day booking window, when the user views the appointment details, then the appointment should remain viewable, and the user should only be allowed to reschedule to a date within the next 7 calendar days. |
| Prevent Invalid Date Selection | Given a date outside the 7-day booking window is selected through any unexpected client-side behavior, when the user attempts to proceed with the booking or update, then the system should prevent the action and display an appropriate validation message. |
| Boundary Validation — Day 7 | Given a user is selecting an appointment date, when the selected date falls exactly on Today + 7 calendar days, then the date should remain selectable. |
| Boundary Validation — Day 8 | Given a user is selecting an appointment date, when the selected date falls on Today + 8 calendar days or later, then the date should be disabled and not selectable. |
| Appointment Type Consistency | Given a user is creating or updating an appointment, when the appointment type is Inbound or Outbound, then the same 7-day appointment selection window should be enforced consistently. |

---

## Affected Screens / Components

<!-- List all screens, components, or modules that will be added or modified -->

| Screen / Component | Change Type | Notes |
|--------------------|-------------|-------|
| Create Appointment — Date Selection | Modify | Restrict calendar to Today + 7 days |
| Edit Appointment — Date Selection | Modify | Restrict calendar to Today + 7 days |
| Reschedule Appointment — Date Selection | Modify | Restrict calendar to Today + 7 days |

---


## Edge Cases & Error Handling

- **Boundary — Day 7 inclusive:** Today + 7 must remain selectable; only Day 8 onwards is disabled.
- **Boundary — Day 8 and beyond:** Must be visually disabled in the calendar and blocked from selection.
- **Existing appointment outside window:** The appointment remains viewable, but reschedule is still restricted to the 7-day window.
- **Unexpected client-side date selection:** System must validate the selected date before proceeding and display an appropriate error message if the date is outside the allowed window.
- **Appointment type (Inbound/Outbound):** The same 7-day rule applies regardless of type — no type-specific exceptions.

---

## Out of Scope

<!-- Explicitly list what is NOT included in this ticket -->

- Backend-side enforcement of the 7-day window (unless confirmed with BE team)
- Changes to appointment viewing or cancellation flows
- Any changes to the time slot selection logic

---

## Testing Notes

<!-- What should QA verify? Any specific devices, OS versions, or scenarios to test -->

- Verify calendar disables Day 8+ on Create, Edit, and Reschedule flows
- Verify Day 7 (Today + 7) remains selectable
- Verify existing appointments outside the 7-day window are still viewable but cannot be rescheduled outside the window
- Verify validation message appears if an out-of-range date is forced through
- Test on both Android and iOS
- Test with both Inbound and Outbound appointment types

---

## Open Questions

<!-- Unresolved questions that need answers before or during development -->

- [ ] Is the 7-day window enforced on the backend as well, or is this a frontend-only change?
- [ ] Should "Today" be based on device local time or server time?
- [ ] Is there a specific error/validation message string to use, or should it follow the existing validation pattern?

---

*Generated by `/spec` on 2026-06-10*
