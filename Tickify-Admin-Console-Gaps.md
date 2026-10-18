# Tickify Admin Console — Feature Gaps & Scope

Oct 18, 2026 · @Tickify

## Overview

This doc lists every gap between the current production admin console and the planned scope, grouped by module in the order an organizer meets them: set up the event, sell it, manage attendees, issue passes, run the gate, then platform-level controls. The reference build is **tickify-admin-core**.

Duplicated scopes from the original notes (Sales Window Manager, Login Required, Invite-only) have been merged into one home each and cross-referenced. Conflicts and open questions are collected at the end.

## 1. Event setup

### 1.1 Occurrences — Gap

- Edit existing occurrences, including start and end date/time.
- Per-occurrence visibility toggle (show / hide on the event page).
- Occurrence scheduling and sales windows are managed centrally in the Sales Window Manager (Section 2).

### 1.2 Event settings — Gap

- **Login Required** toggle (event level) — On / Off. When On, buyers must sign in before purchasing; when Off, guest checkout is allowed. Distinct from the pre-registration list's own Login Required toggle (Section 4.5).
- **Status: Sold Out** — closes public sale of all remaining stock and shows sold-out text on the event page. Comps, Pass Generator issuance and early-access purchases can still continue after Sold Out.

### 1.3 Registration modes and layouts — Gap · Verify first

The donation feature may not exist in the new console yet. It is required because a live v3 event already uses donations ([Here I'm](https://tickify.live/event/here-i-m/)), so it blocks migrating that event.

| Area | Add | Notes |
| --- | --- | --- |
| Registration Mode | Donation | Amount-based registration, no fixed ticket price |
| Registration Mode | Race | For marathon / running events |
| Registration Layout | Amount | Amount chips, custom amount input, optional campaign progress bar. Donation mode only |

## 2. Sales Window Manager — Gap

One screen to see and control when each occurrence is visible and when its sales open and close. Today these controls are scattered and hard to find. This section merges the Sales Window Manager items from Event Settings and from Pre-registration.

**Per occurrence, show and edit:**

- Which event and occurrence the schedule applies to (clearly labelled).
- Visibility and status (incl. Sold Out).
- Pre-registration list opening and closing date/time (Section 4.4).
- Early Access start date/time (Section 4.7).
- General registration opening and closing date/time.

**Rules:**

- General registration opening is the shared boundary: early access ends and public sales start at that moment.
- Validate that Early Access start is before general sales opening.
- If no list closing date is set, the list closes when general sales open.

## 3. Ticket Categories form — Gap

Seven category-level features are planned; production has partial or no support for each.

| Feature | Planned | Production today |
| --- | --- | --- |
| Open-amount pricing | Fixed price, Free and Open amount, with minimum and optional suggested amount | Only Fixed and Free; Open amount and its fields missing |
| Visibility options | Public, Unlisted, Private, Invite only | Only Public and Private |
| Invite-only purchasing | Restrict purchase to approved emails, regardless of the event's access mode | Toggle missing (see conflict note in Section 11) |
| Individual-information override | Event setting / Always / Never for collecting per-attendee info | Missing |
| Bundles / Combo ("Composed of") | Add component lines; contents included at the category's price | Whole section and Add line action missing |
| Support-line purchasing | Accept an amount without consuming quantity or issuing a ticket | Toggle exists; buyer-facing flow not built |
| Behaviour summary | Explains what the config reads as, collects, creates and admits | Only "Reads as" and "Admits" shown |

## 4. Pre-registration / Notify Me & Early Access — Gap

Lets organizers collect interest before sales open, notify the list when sales start, and optionally let the list buy early. Joining the list never reserves or guarantees a ticket, and the UI must say so.

### 4.1 Enable

- "Announce before the sale" toggle.
- Replaces the buy button with **Pre-register** or **Notify me** until general registration opens.
- Switches automatically to the normal purchase flow when general registration opens.

### 4.2 Call to action and messaging

- Choose Pre-register or Notify me.
- Optional custom button label.
- Custom confirmation message shown after joining.

### 4.3 Registration details and capacity

- Collect name, email, phone.
- Custom pre-registration questions, kept at registration/order level.
- Optional list capacity, displayed on the event page and enforced on submit.

### 4.4 List opening and closing

- Optional open and close date/time.
- No open date: accept registrations as soon as the event page is live.
- No close date: close the list when ticket sales open.
- Status shown as Scheduled, Open, Closed or Full.

### 4.5 Login requirement

- Separate Login Required toggle for joining the list (independent of the event-level toggle in 1.2).

### 4.6 Sale notifications

- Channels: Email, SMS, WhatsApp, Push.
- Notify registrants when ticket sales open.
- Only channels connected to a working sending service can be enabled.

### 4.7 Early access

- Toggle early purchasing for the list.
- Early Access Starts date/time, before general registration opens; ends automatically when general registration opens.
- Eligibility: everyone on the list, or selected registrants granted access on the Pre-registrations page.
- Checkout validates eligibility; non-eligible buyers are blocked until public sale.

### 4.8 Pre-registrations management page

- List and search registrants; view contact details, registration time, custom answers.
- Filter by early-access eligibility.
- Grant or revoke early access, individually or in bulk.
- Export registration data.
- Track notification delivery and early-access grants.

### 4.9 Scheduling

All dates in this section are managed in the Sales Window Manager (Section 2).

## 5. Attendees page — Gap

Reference: tickify-admin-core. Offset passes are excluded everywhere on this page: list, totals, check-in counts, exports and bulk selections.

### 5.1 Summary cards

| Card | Shows |
| --- | --- |
| Records | Total records, with Paid / Complimentary / On-spot breakdown |
| Admissions | People admitted, counting tickets that admit more than one person |
| Checked in | Checked-in count and percentage |
| Source | Staff-issued vs customer-purchased |

Record count and admission count must be visibly distinct.

### 5.2 Search and filters

- Free-text search.
- Status tabs: All, Checked in, Not checked in.
- Dropdowns: Category (ticket categories), Issuance (Paid, Complimentary, On-spot), Source (Customer, Staff/manual).
- Search, tabs and dropdowns combine.

### 5.3 Table columns

Select checkbox · Attendee ID · Name + email · Ticket category · Issuance · Source · Order reference · Check-in status · Gate · Registration date. Issuance, source and check-in use badges. Clicking a row opens the detail panel.

### 5.4 Selection and bulk actions

- Select individual rows or the entire filtered set; show the count and whether it covers all filtered results; clear selection.
- Actions: Resend tickets · Download tickets (ZIP) · Export selected · Check in · Undo check-in · Add to segment.
- Apply only to eligible records; report completed, skipped and failed counts.

### 5.5 Attendee detail panel (slide-over)

| Block | Contents |
| --- | --- |
| Identity | Name, email, attendee ID, issuance, source, check-in status |
| Ticket | Category/type, people admitted, group, seat/table or GA, barcode, source description, ticket status |
| Order | Order reference, purchase date, payment method (masked), amount paid, discount and promo code, fees |
| Scan history | Outcome, gate, date/time, staff member, scanner/device ID; re-entries |
| Registration answers | Responses to the event's questions |
| Activity log | Changes and actions on the record |

### 5.6 Individual actions

Resend to recorded email · Send to another email · Download ticket · Check in · Edit details · Open cancellation/refund workflow.

### 5.7 Exports

- **Export CSV** (main) exports the full filtered result, including the current search.
- **Export data** (bulk) exports the selected records only.
- Export scope must always match what the user sees selected and filtered.

## 6. Pass Generator — Gap · Verify first

Most planned controls are already on screen, so verify each one works before treating it as missing or adding a duplicate. Reference: tickify-admin-core; seat allocation is a new requirement.

### 6.1 Confirmed production gaps

- [ ] Sequential bulk generation mode (only Single and Sheet upload exist).
- [ ] Issuance-type selection (fixed to Complimentary today).
- [ ] QR sheet layout options: QR with details / QR only.
- [ ] Live A4 sheet preview.

### 6.2 Full requirements

| Area | Requirement |
| --- | --- |
| Generation | Single, sequential bulk, and sheet upload with column mapping |
| Pass configuration | Batch purpose; access level General / VIP / Crew / Press / Vendor |
| Issuance | Paid, Comp, On-spot, Offset. Paid passes create orders and flow into settlement reports. Offset per Section 7 |
| Output | Standard tickets, printable A4 QR sheets, or both. Default: QR sheets, no email |
| Delivery | Do not send / Send now / Schedule |
| Batch management | Statuses Draft, Generated, Printed, Partially void, Void. Void a batch or a single pass. Reprint issues a new code and invalidates the old one. Download ZIP |
| Tracking | Send-run history with delivered/failed counts and retry; download and print audit logs |
| Permissions | Separate grants for generate/print, comp, and void/reprint. Offset limited to platform Super Admins |

### 6.3 Seat allocation — New requirement

For stadium and other seated events (e.g. the SAFF Championship):

- Select zone, block and available seats when generating.
- Check availability so a seat can never be allocated twice, including against online sales.
- Voiding a pass frees its seat; reprinting keeps it.

## 7. Offset Passes — New requirement

An offset pass is a non-chargeable pass issued from an event's Pass Generator by platform Super Admins only. It has no price or refundable value and is tracked apart from regular tickets.

| Rule | Requirement |
| --- | --- |
| Eligibility | Non-seated events only. Hide the Offset issuance type on seated events |
| Inventory | Never consumes or changes a category's sellable quantity. Example: General limit 100, 98 sold, 5 offsets issued → 2 still on sale |
| Capacity | No cap on offset passes. They sit in the separate offset attendee record, not in event capacity. Expected volume is 10–20 per event |
| Records | Separate offset attendee record with the buyer's details, outside regular attendee lists |
| Visibility | Super Admins only. Excluded from organizer-facing lists, reports and calculations |
| Financials | No chargeable or refundable value; no revenue |
| Entry | Gate scanner accepts valid offset passes and rejects voided or replaced codes |
| Scan counting | Offset scans are never counted in check-in totals, gate counts or scan history. They are tracked only in the offset attendee record |
| Access | Issue, view and manage limited to Super Admins; cannot be granted to organization roles |

**Offset Passes page (Super Admin):** view passes across all events; search and filter; void, reprint, resend and re-download; open the related batch in the Pass Generator.

## 8. Scanner App — Gap

- When enrolling a phone, allow a device name (e.g. the volunteer's name).
- Show that name wherever a scanner/device ID appears, including the attendee scan history (Section 5.5).

## 9. Homepage merchandising — Gap

- **Homepage Banners:** add an event-status filter to banner management.
- **Homepage Event Ordering:** add manual sort order, like Homepage Banners, to control the order of events in the frontend's *On Sale Now* section.

## 10. Fees & Commissions — Gap

- Pagination on the Organizers, Event overrides and Category overrides lists.
- Add organizer action.
- Create new event override and category override actions.

## 11. Decisions and open questions

Four decisions are made and already reflected in the sections above; eight questions remain open and block build on the features they touch.

### 11.1 Decided

| Topic | Decision | Applied in |
| --- | --- | --- |
| Issuance after Sold Out | Comps, Pass Generator issuance and early-access purchases continue after Sold Out | 1.2 |
| Offset passes vs capacity | No cap on offset passes; they live in the separate offset attendee record; expected 10–20 per event | 7 |
| Offset passes at the gate | Offset scans are never counted; tracked only in the offset attendee record | 5, 7 |
| Paid issuance in Pass Generator | Paid passes create orders and flow into settlement reports | 6.2 |

### 11.2 Still open

- [ ] **Invite-only is specified twice in Ticket Categories** — as a Visibility option and as a separate Invite-only purchasing toggle. Pick one model: visibility controls who *sees* it, the toggle controls who can *buy* it, or merge them.
- [ ] **Two allowlists** — category Invite-only (approved emails) and pre-registration early-access eligibility both gate purchase by person. Do they share one allowlist or stay separate?
- [ ] **Two Login Required toggles** (event and pre-registration list). Which wins when they disagree?
- [ ] **Support-line purchasing vs Donation mode** — both accept money without issuing a ticket. Is support-line replaced by Donation mode, or does it serve a different case?
- [ ] **Sold Out mechanics** — manual or automatic? Per event or per occurrence? Reversible?
- [ ] **Seat allocation concurrency** — how do generator allocation and live online seat selection lock seats against each other?
- [ ] **Notification channels** — which of SMS, WhatsApp and Push have a working sending service today?
- [ ] **Donation feature** — does it already exist in the new console?

## 12. Suggested sequencing

Proposed order, by risk rather than by module. Live-event blockers come first; dependent features ship together.

| Phase | Scope | Why this order |
| --- | --- | --- |
| 1 | Seat allocation (6.3); Donation mode (1.3) | Seat allocation is needed for SAFF. Donation blocks migrating a live v3 event |
| 2 | Occurrence editing (1.1); Sold Out and event Login Required (1.2); Pass Generator gaps (6.1, 6.2) | Day-to-day operational controls organizers hit on every event |
| 3 | Sales Window Manager (2) with Pre-registration and Early Access (4) | Same scheduling model; building them apart means rebuilding the date logic twice |
| 4 | Attendees page (5); Offset Passes (7) | Offset exclusion rules shape the attendee list, totals and exports, so design them together |
| 5 | Ticket Categories (3) | Needs the open decisions in 11.2 first |
| 6 | Scanner device names (8); Homepage (9); Fees & Commissions (10) | Small, independent; good fill-in work |
