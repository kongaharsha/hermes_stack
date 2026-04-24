---
name: laundry-booking
description: Book laundry / dry-cleaning pickup via local browser automation.
---

# Laundry Booking

Books a laundry or dry-cleaning pickup using a locally-controlled Chrome instance.

## Workflow

1. **Confirm Intent:** Before triggering any automation, state the proposed action (including proposed pickup/drop-off dates) and confirm with the user.
2. **Setup Browser:** Always use **headed Chrome** (connect to port 9222/headed mode) until requested otherwise (currently planned for the next 4 sessions).
3. **Data Verification:** Read ~/brain/concepts/service-preferences.md.
4. **Calendar/Scheduling:**
   - First, ask the user if they want the *fastest available* slots for both pickup and drop-off, or if they want to review all available options.
   - For all dates and times, always **confirm the selected windows with the user** before final checkout.
5. **Flow Execution:**
   - Navigate to provider site.
   - Phone auth/2FA as required (user intervention required for codes).
   - Wizard steps: Select Pickup Date (check for accordions) -> Time -> Drop-off Date -> Time -> Confirm.
   - Always use snapshot -i after navigation to refresh UI state/refs.
6. **Task Completion:** Finalize order and log details in ~/brain/logs/services.md.

## Implementation Insights (Riverside Laundromat)
- UI: Multi-step wizard.
- Pitfalls: Persistent sign-in loops (if activation code fails, refresh/re-navigate), stale elements (refresh refs constantly), multi-page transitions (Confirm button does not auto-finish).
- If automated interaction stalls, stop and prompt for manual intervention.

## Resilience Patterns
- If automated interaction fails or times out, do NOT retry indefinitely. Stop and inform the user.
- Prefer explicit confirmation for pickup/drop-off windows.
