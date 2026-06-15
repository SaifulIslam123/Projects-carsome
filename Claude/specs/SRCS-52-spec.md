# SRCS-52 — [MOB|CA] Standardize Appointment Selection Window to 7 Days for Create & Update Appointment Flows

> **Mode:** Change Request
> **Change Type:** Mixed (Logic + UI)
> **Priority:** <!-- No data found — fill in manually -->
> **Labels:** <!-- No data found — fill in manually -->
> **Jira:** https://carsome.atlassian.net/browse/SRCS-52

---

## Summary of Change

The appointment date selection behaviour is currently inconsistent between the Create Appointment and Reschedule/Update Appointment flows. The business rule requires that users can only select an inspection date within the next **7 calendar days** from today (Today through Today + 7, inclusive), regardless of flow or appointment type (Inbound/Outbound).

This change enforces that 7-day cap client-side across all flows and updates the `DateList` widget to visually disable and block out-of-range dates.

---

## Current Behaviour

> What the feature does **right now**, based on the existing code.

1. **`AppointmentDateTimeViewModel.getDate()`** calls `_service.getInspectionDates()` which hits `/legacy/v4/inspection-location/$locationId/slot-date/` (inbound) or `/legacy/v4/inspection-location/$locationId/slot-date/?type=sa` (outbound). The full list of dates returned by the backend is stored as-is in `_inboundInspectionDates` / `_outboundInspectionDates` with **no client-side date range filtering**.

2. **`DateList`** (`lib/ui/view/appointment/appointment_datetime/date_list.dart`) renders every date in the list as a horizontally-scrollable tile. It does **not** check `item.disabled` when rendering — the only visual state change is `selected` (blue) vs `unselected` (grey). All tiles are fully tappable regardless of `disabled` value.

3. **`AppointmentDateTimeViewModel.selectDate()`** has no guard against picking a `disabled` date — if `onTap` fires, the date is selected unconditionally.

4. Both the **Create Appointment** flow (`DateTimeView` in `lib/ui/view/price_check/book_appointment/date_time.dart`) and the **Reschedule Appointment** flow (`RescheduleAppointmentView` in `lib/ui/view/appointment/appointment_reschedule/reschedule_appointment_view.dart`) call the same `getDate()` + `showDateTimeSheet()` pipeline through the shared `AppointmentDateTimeViewModel`, so they render dates identically — there is currently **no difference** in date range enforcement between these two flows at the client layer.

5. At submission time (`BookAppointmentRP.bookAppointment()` and `RescheduleAppointmentViewModel.rescheduleAppointment()`), the selected date is sent directly to the API with **no client-side date range validation**.

---

## Requested Change

> What Jira is asking us to modify, add, or remove.

- Restrict the selectable inspection dates to **Today through Today + 7 calendar days (inclusive)** across all three booking flows: Create Appointment, Edit Appointment, and Reschedule Appointment.
- Dates returned by the API that fall outside this window must be rendered as **visually disabled** in the `DateList` widget and must **not be selectable**.
- The same 7-day rule applies to both **Inbound** and **Outbound** appointment types.
- Existing appointments booked outside the window remain viewable, but reschedule is still restricted to the 7-day window.

---

## What Stays the Same

> Parts of the existing implementation that must **not** be touched.

- The `getInspectionDates` and `getInspectionTime` API calls and their URL construction in `url.dart` remain unchanged.
- The `AppointmentDateTimeViewModel` public interface (state fields, notifiers, `getDate` signature) remains unchanged.
- The `showDateTimeSheet` → `InboundBody` / `OutboundBody` rendering pipeline and their time-slot (AM/PM chip) logic remain unchanged.
- `RescheduleAppointmentViewModel.rescheduleAppointment()` and `BookAppointmentRP.bookAppointment()` API call bodies remain unchanged.
- The `InspectionDate` model shape (`date`, `disabled`) remains unchanged — no new fields required.
- The location selection flow and `AppointmentLocationViewModel` are untouched.
- The `checkSelectedDate()` auto-selection logic in `AppointmentDateTimeViewModel` must continue to auto-select the **first non-disabled** date (already handled if filtering is done before `checkSelectedDate` is called).

---

## Before / After Comparison

### UI / Screen Flow

| Before | After |
|--------|-------|
| All API-returned dates shown; every tile tappable | Dates > Today+7 shown with disabled style; taps blocked |
| No visual indicator for out-of-range dates | Greyed-out tile (reduced opacity or `ThemeColor.gray200` bg) with no `onTap` |

### Logic / Behaviour

| Before | After |
|--------|-------|
| `getDate()` stores API dates as-is | `getDate()` filters list: only keep `date >= today && date <= today + 7` |
| `selectDate()` accepts any date | `selectDate()` no-ops if the matching `InspectionDate.disabled == true` |
| `DateList._buildDateItem` ignores `item.disabled` | `_buildDateItem` reads `item.disabled` → blocks `onTap`, applies grey style |

### API / Data Shape _(if applicable)_

| Before | After |
|--------|-------|
| No change — dates come from backend as-is | No change — same API call; filter is applied client-side after fetch |

---

## Affected Files

| File | Type | Change Description |
|------|------|--------------------|
| `lib/core/viewmodel/appointment_datetime_view_model.dart` | ViewModel | Filter dates to 7-day window in `getDate()`; guard `selectDate()` against disabled dates |
| `lib/ui/view/appointment/appointment_datetime/date_list.dart` | Widget | Render disabled visual state for `item.disabled == true`; block `onTap` for disabled tiles |

> **Note:** `lib/core/model/inspection_date.dart` already has the `disabled` field — no model changes needed. All other files in the confirmed list are read-only for this change.

---

## Implementation Plan

> Step-by-step what to change, in which file, and why.

### Step 1 — `appointment_datetime_view_model.dart`: filter in `getDate()`

Inside the `try` block of `getDate()`, after fetching `inspectionDateResponse`, apply a client-side cap before storing the list.

Add a helper at the top of the class (or as a file-level private function):

```dart
List<InspectionDate> _applySevenDayWindow(List<InspectionDate> dates) {
  final DateTime today = DateTime(
    DateTime.now().year,
    DateTime.now().month,
    DateTime.now().day,
  );
  final DateTime maxDate = today.add(const Duration(days: 7));
  return dates.map((d) {
    if (d.date == null) return d;
    final DateTime dateOnly = DateTime(d.date!.year, d.date!.month, d.date!.day);
    if (dateOnly.isBefore(today) || dateOnly.isAfter(maxDate)) {
      return InspectionDate(date: d.date, disabled: true);
    }
    return d;
  }).toList();
}
```

Then in `getDate()`, wrap the stored lists:

```dart
// outbound path
_outboundInspectionDates = _applySevenDayWindow(
  inspectionDateResponse?.inspectionDates ?? [],
);

// inbound path
_inboundInspectionDates = _applySevenDayWindow(
  inspectionDateResponse.inspectionDates ?? [],
);
```

> **Why:** The backend may return dates beyond 7 days for some locations or for the reschedule endpoint. This client-side filter is the single source of truth for the allowed window, making Create, Edit, and Reschedule consistent regardless of backend response.

### Step 2 — `appointment_datetime_view_model.dart`: guard `selectDate()`

In `selectDate()`, before calling `getInspectionTime`, check whether the tapped date is disabled in the stored list:

```dart
Future selectDate(DateTime date, AppointmentType appointmentType, int? locationId) async {
  final List<InspectionDate> list = appointmentType == AppointmentType.outbound
      ? _outboundInspectionDates
      : _inboundInspectionDates;

  final InspectionDate? match = list.firstWhereOrNull((d) => d.date == date);
  if (match?.disabled == true) return; // guard: ignore tap on disabled date

  // ... existing logic unchanged below
```

> **Why:** Even if `DateList` blocks taps (Step 3), this adds a defence-in-depth guard so the ViewModel never processes an out-of-window date regardless of how the tap reaches it.

### Step 3 — `date_list.dart`: disabled visual state + tap block

In `_buildDateItem`, update the `InkWell` and inner `Container` to check `item.disabled`:

```dart
InkWell _buildDateItem(InspectionDate item) {
  final bool isDisabled = item.disabled == true;
  final bool isSelected = item.date == selectedDate && !isDisabled;

  return InkWell(
    onTap: isDisabled ? null : () => onSelect?.call(item.date),
    child: Opacity(
      opacity: isDisabled ? 0.35 : 1.0,
      child: Container(
        // ... existing sizing/margin unchanged
        decoration: BoxDecoration(
          borderRadius: BorderRadius.circular(4),
          shape: BoxShape.rectangle,
          color: isSelected ? ThemeColor.blue300 : ThemeColor.gray50,
        ),
        child: Column(
          // ... existing day/date/month text unchanged
        ),
      ),
    ),
  );
}
```

> **Why:** `DateList` is shared between `InboundBody` and `OutboundBody`, so this single change covers all appointment types. Using `Opacity(0.35)` is consistent with how other disabled UI elements look in the app.

### Step 4 — Verify `checkSelectedDate()` still works

`checkSelectedDate()` auto-selects a date from the filtered list. After Step 1, the list only contains dates within the window (with out-of-window ones marked `disabled: true`). `checkSelectedDate()` does `_inboundInspectionDates[0].date` — confirm the first element in the filtered list is a valid (non-disabled) date. If the API returns today as the first date (most common case), this is fine. If **all** returned dates are disabled (edge case: e.g., outbound location with no available slots in the window), `checkIfCanEnableButton` will not enable the CTA — the existing empty-slot screen (`EmptySlot`) handles this gracefully.

No code change needed here, but QA should verify the empty-slot state for a location with no availability in the next 7 days.

---

## Acceptance Criteria

| Scenario | Acceptance Criteria |
|----------|---------------------|
| Create Appointment — Display Available Dates | Given a user is creating a new appointment, when the appointment calendar is opened, then the user should be able to select dates from Today up to Today + 7 calendar days only. |
| Create Appointment — Disable Dates Beyond 7 Days | Given a user is creating a new appointment, when the user views dates beyond Today + 7 calendar days, then those dates should be displayed as disabled and should not be selectable. |
| Edit Appointment — Display Available Dates | Given a user is editing an existing appointment, when the appointment calendar is opened, then the user should be able to select dates from Today up to Today + 7 calendar days only. |
| Edit Appointment — Disable Dates Beyond 7 Days | Given a user is editing an existing appointment, when the user views dates beyond Today + 7 calendar days, then those dates should be disabled and not selectable. |
| Reschedule Appointment — Display Available Dates | Given a user is rescheduling an existing appointment, when the appointment calendar is opened, then the user should be able to select dates from Today up to Today + 7 calendar days only. |
| Reschedule Appointment — Disable Dates Beyond 7 Days | Given a user is rescheduling an existing appointment, when the user views dates beyond Today + 7 calendar days, then those dates should be disabled and not selectable. |
| Consistent Date Selection Logic Across Flows | Given a user accesses the Create, Edit, or Reschedule Appointment flow, when the appointment calendar is displayed, then the same 7-day appointment selection window should be applied across all flows. |
| Existing Appointment Outside New Window | Given an appointment already exists outside the new 7-day booking window, when the user views the appointment details, then the appointment should remain viewable, and the user should only be allowed to reschedule to a date within the next 7 calendar days. |
| Prevent Invalid Date Selection | Given a date outside the 7-day booking window is selected through any unexpected client-side behavior, when the user attempts to proceed with the appointment booking or update, then the system should prevent the action and display an appropriate validation message. |
| Boundary Validation — Day 7 | Given a user is selecting an appointment date, when the selected date falls exactly on Today + 7 calendar days, then the date should remain selectable. |
| Boundary Validation — Day 8 | Given a user is selecting an appointment date, when the selected date falls on Today + 8 calendar days or later, then the date should be disabled and not selectable. |
| Appointment Type Consistency | Given a user is creating or updating an appointment, when the appointment type is Inbound or Outbound, then the same 7-day appointment selection window should be enforced consistently. |

---

## Regression Risk

| Risk | Likelihood | Area to Test |
|------|------------|--------------|
| `DateList` disabled style overrides selected style | Medium | Tap a date, check blue highlight still appears; disabled dates must never show blue |
| `checkSelectedDate()` crashes on empty list | Low | Open date picker for a location with all dates outside the 7-day window |
| Outbound date list behaves differently from Inbound | Low | Test both Inbound (inspection centre) and Outbound (home pickup) date pickers |
| Reschedule flow no longer pre-fills current date | Low | Open reschedule, confirm the existing appointment date is pre-filled in the location/date fields (this is done in `RescheduleAppointmentView.onModelReady`, unaffected by this change) |
| `_applySevenDayWindow` uses device local time vs server time | Medium | Test around midnight / timezone boundaries |

---

## UX & Design

- Figma: <!-- paste link here -->
- Design notes: No Figma reference provided in the ticket. Disabled date tile should follow existing disabled-component styling (reduced opacity or `ThemeColor.gray200` background, non-interactive). Confirm with design team if a strikethrough or "X" marker is needed for disabled tiles.

---

## Edge Cases & Error Handling

- **Boundary — Day 7 inclusive:** `today.add(Duration(days: 7))` is used as `maxDate`. Dates where `dateOnly == maxDate` pass the filter and remain enabled.
- **Boundary — Day 8 and beyond:** Dates where `dateOnly.isAfter(maxDate)` are marked `disabled: true` and shown greyed-out.
- **Past dates:** Dates before today (e.g., today - 1) are also marked disabled in `_applySevenDayWindow`, guarding against any backend quirks.
- **Empty window:** If the backend returns no dates within the 7-day window (e.g., all slots fully booked), the existing `EmptySlot` widget is displayed — no new empty-state handling needed.
- **`DateTime` comparison:** Use date-only `DateTime(y, m, d)` comparisons to avoid time-of-day edge cases. `DateTime.now()` includes the current time, which would make "today" appear as a past date once the time passes — stripping to midnight prevents this.
- **Appointment type:** `_applySevenDayWindow` is called for both the outbound and inbound branches of `getDate()`, so both types are covered.

---

## Dependencies

- No backend changes required (confirmed: client-side filter is sufficient).
- No dependency on other in-flight tickets.

---

## Testing Notes

- [ ] Open Create Appointment → date picker → verify Day 8+ dates are greyed and non-tappable
- [ ] Open Create Appointment → date picker → verify Day 7 (Today + 7) is tappable
- [ ] Open Reschedule Appointment → date picker → verify same 7-day cap applies
- [ ] Open Edit Appointment (price_check update flow) → date picker → verify same 7-day cap applies
- [ ] Select a valid date (Days 1–7) → complete booking → verify no regression in submission
- [ ] Test with Inbound location and with Outbound location
- [ ] Test on both Android and iOS
- [ ] Open a date picker where the backend returns >7 days — verify extra dates are disabled
- [ ] Simulate an appointment that already exists outside the 7-day window → view detail → open reschedule → verify only 7-day window is offered

---

## Open Questions

- [ ] Should "Today" be based on device local time or server time? (Currently using `DateTime.now()` = device local time.)
Answer: device local time
- [ ] Is the 7-day window also enforced on the backend for the reschedule API, or is client-side the only gate?
Answer: only client-side only
- [ ] Should disabled dates beyond Day 7 be hidden entirely (removed from list) or shown as disabled tiles? 
(This spec assumes shown-as-disabled for transparency; confirm with design/product.)
Answer: shown as disabled tiles like the existing code ui implementation
- [ ] Is there a specific validation/error message string to display if an out-of-range date somehow reaches the ViewModel? Or should it silently no-op?
Answer: yes show this message as string "Appointment date should be with 7 days from today."
---

*Generated by `/spec --cr` on 2026-06-11*
