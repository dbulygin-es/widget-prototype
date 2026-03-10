# Calendar Datepicker & Unified Delivery Method — Design

**Date:** 2026-03-10

## Problem

The prototype showed both Local and Nationwide delivery as radio options on screen-4, forcing the user to choose a delivery method explicitly. In reality the method is determined by the selected date (same-day = local, future = nationwide) and the address eligibility (zone). The old date pills on screen-4 were also a weak UI — they didn't communicate the zone constraint visually.

## Solution

1. Move date selection to screen-3 via a full calendar component.
2. Derive the delivery method from `selectedDate` + `deliveryMode` (zone).
3. Replace the two screen-4 variants with one adaptive screen-4.

---

## Calendar Component (screen-3, inline)

Clicking the date row expands an inline calendar below it. No modal/overlay.

### Anatomy
```
  March  2026       ‹  ›
  Mo Tu We Th Fr Sa Su
               1   2
   3  4  5  6  7   8   9
  10 11 12 13 14  15  16
```

### Cell States
| State | Visual |
|---|---|
| Past date | Muted, disabled, no cursor |
| Today — local zone | Pre-selected, primary color |
| Today — nationwide zone | Grayed out, strikethrough, cursor: not-allowed |
| Future date — selectable | Normal, hover highlight |
| Selected date | Filled primary background, white text |

### Behaviour
- Prev/next month arrows; past months not reachable
- Selecting a date collapses the calendar and updates the date row
- Default selection: today (local zone) or tomorrow (nationwide zone)

---

## Delivery Method Logic

| Zone (demo panel) | Date selected | Screen-4 shows |
|---|---|---|
| Local | Today | Local delivery + time slots |
| Local | Future date | Nationwide delivery (no amber notice) |
| Nationwide | Today | Impossible — today is disabled |
| Nationwide | Future date | Nationwide delivery + amber notice |

Amber notice ("Local delivery not available at your address") appears **only** when `deliveryMode === 'nationwide'`, not when a local-zone user voluntarily picks a future date.

---

## Screen-4 Unified

Single `screen-4` replaces `screen-4-local` and `screen-4-nationwide`.

**Local path** (`selectedDate === today && deliveryMode === 'local'`):
- Local delivery selected (€3.50), expanded with time slot radio rows
- Nationwide delivery unselected below
- Pickup unselected below

**Nationwide path** (any other combination):
- Amber pill shown only if `deliveryMode === 'nationwide'`
- Nationwide delivery selected (€5.95), estimated delivery date shown
- Pickup unselected below

Date pills removed from screen-4 — date lives entirely on screen-3.

---

## Files Changed

- `/tmp/widget-prototype/index.html` — all changes in this single file
