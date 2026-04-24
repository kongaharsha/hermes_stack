---
type: concept
title: Airline Preferences
tags: [travel, preferences, reference]
---

Default choices for airline online check-in. Used by the airline check-in skill when it's picking a seat, deciding bag count, or answering the hazmat declaration.

## Defaults (applies to any airline)

- **Baggage:** Carry-on only
- **Seat:** Aisle, front third of cabin
- **Hazmat declaration:** Accept (no restricted items in my bag)
- **Survey popups:** Dismiss

## Per-airline overrides

Leave empty if the default applies. Fill in when a specific carrier needs something different (e.g., seat map layout, frequent route preference).

### United

- **Seat:** <e.g. Economy Plus if available, else any aisle row 7-15>
- **Baggage:** <inherit default, or override>
- **Meal:** <if long-haul>

### American

- **Seat:** <override>
- **Baggage:** <override>

### Delta

- **Seat:** <override>
- **Baggage:** <override>

### <add others as needed>

---

- <date>: created
