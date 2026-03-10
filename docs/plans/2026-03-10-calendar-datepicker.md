# Calendar Datepicker & Unified Delivery Method Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace date pills on screen-4 with an inline calendar on screen-3, and collapse the two screen-4 variants into one adaptive screen that shows exactly one delivery method based on selected date + zone.

**Architecture:** All changes are in `/tmp/widget-prototype/index.html`. The calendar renders inline below the date row on screen-3 when clicked. `selectedDate` (a Date object) drives which delivery method shows on screen-4. Screen-4 content is JS-rendered on navigation.

**Tech Stack:** Vanilla HTML/CSS/JS, single-file prototype.

---

### Task 1 — `selectedDate` state and helpers

**Files:**
- Modify: `/tmp/widget-prototype/index.html` (script block, near `let deliveryMode`)

**Step 1: Add state + helpers after `let enteredAddress = '';`**

```javascript
let selectedDate = null; // Date object, null until screen-3 is first visited

function isToday(d) {
  const t = new Date();
  return d.getFullYear() === t.getFullYear() &&
         d.getMonth()    === t.getMonth()    &&
         d.getDate()     === t.getDate();
}

function getDeliveryMethod() {
  if (!selectedDate) return null;
  return (isToday(selectedDate) && deliveryMode === 'local') ? 'local' : 'nationwide';
}
```

**Step 2: Update `navigateTo()` — initialise `selectedDate` on screen-3, reset on screen-address**

Inside `navigateTo(destId)`, replace the existing `if (destId === 'screen-3') updateScreen3Date();` with:

```javascript
if (destId === 'screen-3') {
  if (!selectedDate) {
    selectedDate = new Date();
    if (deliveryMode === 'nationwide') selectedDate.setDate(selectedDate.getDate() + 1);
  }
  calOpen = false;
  document.getElementById('calendar').style.display = 'none';
  updateScreen3Date();
}
if (destId === 'screen-address') selectedDate = null;
```

Note: `calOpen` and the calendar element are added in Task 3 — that's fine, these lines run after Task 3 is merged.

**Step 3: Update `updateScreen3Date()` to use `selectedDate`**

Replace the function body:

```javascript
function updateScreen3Date() {
  const d = selectedDate || new Date();
  document.getElementById('screen3-date').textContent = formatDateShort(d);
}
```

**Step 4: Verify**

Open `file:///tmp/widget-prototype/index.html`, enter address, reach screen-3.
Date row should show today (local zone) or tomorrow (nationwide zone).

**Step 5: Commit**

```bash
cd /tmp/widget-prototype
git add index.html
git commit -m "feat: selectedDate state and delivery method helpers"
```

---

### Task 2 — Calendar CSS

**Files:**
- Modify: `/tmp/widget-prototype/index.html` (style block, before `</style>`)

**Step 1: Append calendar styles**

```css
/* ─── Calendar Component ─── */
.calendar {
  background: var(--color-white);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-card);
  padding: 12px;
  margin-bottom: 16px;
}
.calendar-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 10px;
}
.calendar-month-label {
  font-size: 15px;
  font-weight: 700;
  color: var(--color-text);
}
.cal-nav {
  width: 28px;
  height: 28px;
  border: 1px solid var(--color-border);
  border-radius: 6px;
  background: var(--color-white);
  font-size: 16px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--color-primary);
  font-family: var(--font-sans);
  line-height: 1;
}
.cal-nav:hover { background: var(--color-bg); }
.cal-nav:disabled { color: var(--color-border); cursor: default; background: none; }
.calendar-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 2px;
}
.cal-weekday {
  font-size: 11px;
  font-weight: 600;
  color: var(--color-muted);
  text-align: center;
  padding: 4px 0;
}
.cal-day {
  aspect-ratio: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 13px;
  border-radius: 6px;
  cursor: pointer;
  color: var(--color-text);
}
.cal-day--empty { cursor: default; }
.cal-day:hover:not(.cal-day--disabled):not(.cal-day--today-blocked):not(.cal-day--selected) {
  background: var(--color-bg);
}
.cal-day--disabled {
  color: var(--color-border);
  cursor: not-allowed;
}
.cal-day--today-blocked {
  color: var(--color-muted);
  cursor: not-allowed;
  text-decoration: line-through;
}
.cal-day--today {
  background: var(--color-selected-bg);
  font-weight: 700;
  color: var(--color-primary);
}
.cal-day--selected {
  background: var(--color-primary) !important;
  color: white !important;
  font-weight: 700;
}
```

**Step 2: Commit**

```bash
git add index.html
git commit -m "feat: calendar CSS"
```

---

### Task 3 — Calendar HTML on screen-3

**Files:**
- Modify: `/tmp/widget-prototype/index.html` (screen-3 section)

**Step 1: Make date-row a toggle trigger and add calendar container below it**

Replace the existing date-row + its surrounding step-row in screen-3 `screen-content` with:

```html
<div class="step-row">
  <span class="step-badge">3</span>
  <span class="step-label">Date</span>
</div>
<div class="date-row" id="date-row" style="cursor:pointer; margin-bottom:0">
  <span id="screen3-date">—</span>
  <span>📅</span>
</div>

<!-- Inline calendar — hidden until date-row is clicked -->
<div class="calendar" id="calendar" style="display:none; margin-top:8px">
  <div class="calendar-header">
    <button class="cal-nav" id="cal-prev">‹</button>
    <span class="calendar-month-label" id="cal-month-year"></span>
    <button class="cal-nav" id="cal-next">›</button>
  </div>
  <div class="calendar-grid" id="cal-grid">
    <!-- populated by renderCalendar() -->
  </div>
</div>
```

**Step 2: Verify**

screen-3 looks identical to before (calendar div is invisible). date-row cursor is a pointer.

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: calendar HTML structure in screen-3"
```

---

### Task 4 — Calendar JS (render + open/close + selection)

**Files:**
- Modify: `/tmp/widget-prototype/index.html` (script block)

**Step 1: Add calendar view state variables (near top of script)**

```javascript
let calYear  = new Date().getFullYear();
let calMonth = new Date().getMonth(); // 0-indexed
let calOpen  = false;
```

**Step 2: Add `renderCalendar()` function**

```javascript
const MONTH_NAMES = ['January','February','March','April','May','June',
  'July','August','September','October','November','December'];

function renderCalendar() {
  document.getElementById('cal-month-year').textContent =
    `${MONTH_NAMES[calMonth]} ${calYear}`;

  const today = new Date(); today.setHours(0,0,0,0);
  const prevBtn = document.getElementById('cal-prev');
  const thisMonth = new Date(calYear, calMonth, 1);
  const currentMonth = new Date(); currentMonth.setDate(1); currentMonth.setHours(0,0,0,0);
  prevBtn.disabled = thisMonth <= currentMonth;

  const firstDay  = new Date(calYear, calMonth, 1).getDay();
  const offset    = firstDay === 0 ? 6 : firstDay - 1; // Mon-first grid
  const daysInMonth = new Date(calYear, calMonth + 1, 0).getDate();

  const WEEKDAYS = ['Mo','Tu','We','Th','Fr','Sa','Su'];
  let html = WEEKDAYS.map(d => `<div class="cal-weekday">${d}</div>`).join('');

  for (let i = 0; i < offset; i++) {
    html += `<div class="cal-day cal-day--empty"></div>`;
  }

  for (let day = 1; day <= daysInMonth; day++) {
    const d = new Date(calYear, calMonth, day); d.setHours(0,0,0,0);

    const isPast    = d < today;
    const isTodayCell = d.getTime() === today.getTime();
    const isBlockedToday = isTodayCell && deliveryMode === 'nationwide';
    const isSelected = selectedDate &&
      d.getTime() === new Date(new Date(selectedDate).setHours(0,0,0,0)).getTime();

    let cls = 'cal-day';
    if      (isPast)          cls += ' cal-day--disabled';
    else if (isBlockedToday)  cls += ' cal-day--today-blocked';
    else if (isSelected)      cls += ' cal-day--selected';
    else if (isTodayCell)     cls += ' cal-day--today';

    const disabled = isPast || isBlockedToday;
    html += `<div class="${cls}"${disabled ? '' : ` data-date="${d.toISOString()}"`}>${day}</div>`;
  }

  document.getElementById('cal-grid').innerHTML = html;
}
```

**Step 3: Add calendar events to the main click handler (insert BEFORE the `#address-bar` check)**

```javascript
// Open/close calendar
if (e.target.closest('#date-row')) {
  calOpen = !calOpen;
  const cal = document.getElementById('calendar');
  if (calOpen) {
    const ref = selectedDate || new Date();
    calYear  = ref.getFullYear();
    calMonth = ref.getMonth();
    renderCalendar();
    cal.style.display = 'block';
  } else {
    cal.style.display = 'none';
  }
  return;
}

// Prev month
if (e.target.closest('#cal-prev')) {
  e.stopPropagation();
  calMonth--;
  if (calMonth < 0) { calMonth = 11; calYear--; }
  renderCalendar();
  return;
}

// Next month
if (e.target.closest('#cal-next')) {
  e.stopPropagation();
  calMonth++;
  if (calMonth > 11) { calMonth = 0; calYear++; }
  renderCalendar();
  return;
}

// Day selection
const dayEl = e.target.closest('.cal-day[data-date]');
if (dayEl) {
  e.stopPropagation();
  selectedDate = new Date(dayEl.dataset.date);
  updateScreen3Date();
  document.getElementById('calendar').style.display = 'none';
  calOpen = false;
  return;
}
```

**Step 4: Verify**

1. Navigate to screen-3 — date row shows correct date
2. Click date row — calendar opens showing current month
3. Today's cell: highlighted in primary (local zone) or struck-through (nationwide zone)
4. Click a future date — calendar closes, date row updates
5. Prev arrow disabled when on current month; clicking next goes forward
6. Back to screen-address — navigating to screen-3 again resets date

**Step 5: Commit**

```bash
git add index.html
git commit -m "feat: calendar render, open/close, and date selection"
```

---

### Task 5 — Unified screen-4

**Files:**
- Modify: `/tmp/widget-prototype/index.html`

**Step 1: Replace `screen-4-local` and `screen-4-nationwide` with a single `screen-4`**

Delete both `<section id="screen-4-local">` and `<section id="screen-4-nationwide">` blocks. Add in their place:

```html
<!-- SCREEN 4: Delivery details (content rendered by renderScreen4()) -->
<section class="screen" id="screen-4">
  <div class="widget-header">
    <a class="header-back" data-go="screen-3">‹ Back to Cart</a>
    <div class="header-actions">
      <div class="header-cart">🛍<span class="cart-badge">1</span></div>
      <div class="header-lang">🇬🇧 ∨</div>
      <span class="header-close">✕</span>
    </div>
  </div>
  <div class="checkout-progress">
    <div class="progress-step active">
      <span class="progress-badge">1</span> Delivery details
    </div>
    <div class="progress-divider"></div>
    <div class="progress-step inactive">
      <span class="progress-badge">2</span> Payment method
    </div>
    <div class="progress-divider"></div>
    <div class="progress-step inactive">
      <span class="progress-badge">3</span> Confirmation
    </div>
  </div>
  <div class="screen-content" id="screen-4-content">
    <!-- populated by renderScreen4() on Add to Cart -->
  </div>
  <div class="widget-footer">Powered by <strong>sous</strong></div>
</section>
```

**Step 2: Add `renderScreen4()` function**

```javascript
function renderScreen4() {
  const isLocal        = getDeliveryMethod() === 'local';
  const isNationwideZone = deliveryMode === 'nationwide';
  const dispatchDate   = selectedDate ? formatDateShort(selectedDate) : '—';
  let html = '';

  if (!isLocal && isNationwideZone) {
    html += `<span class="notice-pill amber">Local delivery not available at your address</span>`;
  }

  if (isLocal) {
    html += `
      <div class="radio-row selected">
        <div class="radio-row-left">
          <div class="radio-circle checked"></div>
          <span class="radio-row-label">Local delivery</span>
        </div>
        <span style="font-size:13px;font-weight:600;color:var(--color-primary)">€3.50</span>
      </div>
      <div class="delivery-option-expanded">
        <p class="section-heading">Select delivery time</p>
        <div class="radio-row selected">
          <div class="radio-row-left">
            <div class="radio-circle checked"></div>
            <span class="radio-row-label">17:00 – 18:00</span>
          </div>
        </div>
        <div class="radio-row">
          <div class="radio-row-left">
            <div class="radio-circle"></div>
            <span class="radio-row-label">18:00 – 19:00</span>
          </div>
        </div>
        <div class="radio-row">
          <div class="radio-row-left">
            <div class="radio-circle"></div>
            <span class="radio-row-label">19:00 – 20:00</span>
          </div>
        </div>
      </div>
      <div class="radio-row" style="margin-top:8px">
        <div class="radio-row-left">
          <div class="radio-circle"></div>
          <span class="radio-row-label">Nationwide delivery</span>
        </div>
        <span>🚚</span>
      </div>`;
  } else {
    html += `
      <div class="radio-row selected">
        <div class="radio-row-left">
          <div class="radio-circle checked"></div>
          <div>
            <div class="radio-row-label">Nationwide delivery</div>
            <div class="radio-row-sublabel">Ships ${dispatchDate} · €5.95</div>
          </div>
        </div>
        <span>🚚</span>
      </div>`;
  }

  html += `
    <div class="radio-row">
      <div class="radio-row-left">
        <div class="radio-circle"></div>
        <span class="radio-row-label">Pickup options</span>
      </div>
      <span>🏪</span>
    </div>`;

  document.getElementById('screen-4-content').innerHTML = html;
}
```

**Step 3: Update Add to Cart handler**

Replace the existing `#btn-add-to-cart` block with:

```javascript
if (e.target.closest('#btn-add-to-cart')) {
  renderScreen4();
  navigateTo('screen-4');
  return;
}
```

**Step 4: Remove `buildDatePillsHTML()` function** — no longer needed.

**Step 5: Verify**

| Scenario | Expected on screen-4 |
|---|---|
| Local zone + today selected | Local delivery + time slots, no amber pill |
| Local zone + future date | Nationwide "Ships Tue 11 Mar · €5.95", no amber pill |
| Nationwide zone + any date | Nationwide + amber pill |

**Step 6: Commit**

```bash
git add index.html
git commit -m "feat: unified screen-4 with adaptive delivery method"
```

---

### Task 6 — Demo controls reset + push

**Step 1: Update demo-controls change handler**

Replace the existing handler body:

```javascript
deliveryMode = this.value;
selectedDate = null; // will re-initialise on next screen-3 visit

// If currently on screen-3, re-initialise immediately
if (document.getElementById('screen-3').classList.contains('active')) {
  selectedDate = new Date();
  if (deliveryMode === 'nationwide') selectedDate.setDate(selectedDate.getDate() + 1);
  updateScreen3Date();
  if (calOpen) renderCalendar(); // refresh open calendar if visible
}
```

**Step 2: Final walkthrough verification**

1. Load, enter address, Continue
2. screen-3: date row shows today (local) → click opens calendar, today cell highlighted
3. Toggle demo → Nationwide: today cell strikes through, date row updates to tomorrow
4. Select a future date → Add to Cart → screen-4 shows correct method + amber pill (nationwide zone)
5. Click address bar → returns to screen-address with input pre-filled
6. Re-enter address, reach screen-3 again → selectedDate reset, shows earliest available date

**Step 3: Push**

```bash
cd /tmp/widget-prototype
git push origin main
```
