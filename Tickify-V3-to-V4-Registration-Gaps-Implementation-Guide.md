# Tickify V3 → V4 Registration and Checkout Gaps: Implementation Guide

## Purpose

V4 must support fourteen registration and checkout features; this guide specifies each one as config plus the expected behaviour.

For each gap you get: what it does, which live events need it, the config to set (where there is any), the expected V4 behaviour, and a working reference where one exists.

**Scope.** Gaps 1–7 are config in sample/mock data on the frontend (tickify-web); no backend work. Gap 12 is a frontend layout change. Gaps 8–11, 13 and 14 (cart and order hold timeout, abandoned-cart reminders, checkout, confirmation, payment return, account orders) cannot be done frontend-only: order creation, holds, expiry, seat release, coupon validation, payment status, order filtering and scheduled emails must run on the server. Their behaviour is specified here the same way, but each needs backend work before it can be QA'd end to end.

How to use it: pick a gap, apply the config to the listed event in V4, then compare against the V3 page and the V4 reference. Tick it off in the QA checklist at the end.

## Gap summary

| # | Feature | Config level | Live event that needs it | V4 reference |
| --- | --- | --- | --- | --- |
| 1 | Team registration mode | Event + tier | [Econo Carnival Bangladesh S01](https://tickify.bickfoundation.org/o/tickify/econo-carnival-bangladesh-season-01) | [creator-tech-expo](https://abir-web-development.up.railway.app/events/creator-tech-expo) |
| 2 | Ticket tier clusters | Event + tier | Any event with grouped tiers | [food-carnival](https://abir-web-development.up.railway.app/events/food-carnival) |
| 3 | Event-level additional fields | Event | Any event | [food-carnival](https://abir-web-development.up.railway.app/events/food-carnival) |
| 4 | Tier-level additional fields | Tier | Any event | [food-carnival](https://abir-web-development.up.railway.app/events/food-carnival) |
| 5 | Separate registration page per tier | Tier | [CA Bangladesh Accounting Day Run 2026](https://tickify.bickfoundation.org/o/icab/ca-bangladesh-accounting-day-run-2026) | [dhaka-marathon](https://abir-web-development.up.railway.app/events/dhaka-marathon) |
| 6 | Race / marathon wizard | Event + tier | [CA Bangladesh Accounting Day Run 2026](https://tickify.bickfoundation.org/o/icab/ca-bangladesh-accounting-day-run-2026) | [dhaka-marathon](https://abir-web-development.up.railway.app/events/dhaka-marathon) |
| 7 | Tier-level coupon assignment | Tier | Any event with coupons | — |
| 8 | Cart and order hold timeout | Event | All event types | [V4 cart](https://abir-web-development.up.railway.app/cart) |
| 9 | Abandoned-cart reminder email | Event | Events with order hold timeout disabled | — |
| 10 | Checkout page (review, coupons, add-on services) | Platform (add-ons TBD) | All event types | [V4 checkout](https://abir-web-development.up.railway.app/checkout/watch-party-order-2027-001) |
| 11 | Order confirmation page | Platform | All event types | [V4 confirmation](https://abir-web-development.up.railway.app/checkout/art-walk-order-2027-002/success) |
| 12 | Event card: price and Book now | Platform | All event types | — |
| 13 | Payment return page (cancelled, failed, pending, other issues) | Platform | All event types | — |
| 14 | Account orders page: paid orders only | Platform | All event types | — |

## Core concepts

Every setting lives either on the event (applies to all tiers) or on a ticket tier (applies to that tier only). Tier-level config adds to or overrides event-level config. Gaps 10–14 are platform behaviour with no per-event setting (except the open question on add-on services in Gap 10).

| Level | Where it lives | Examples |
| --- | --- | --- |
| Event | Event settings object | `registrationMode`, `registrationLayout`, `showClusterFilter`, `onPageRegistration`, `collectIndividualInformation`, `teamRegistration`, `registrationFields`, `orderHold`, `abandonedCartReminder` |
| Ticket tier | Each item in the event's tiers list | `cluster`, `separateRegistrationPage`, `teamRegistration`, `registrationFields`, `allowedCouponCodes`, `cardImageUrl` |

**Field scope.** Every registration field has a `scope`:

- `"order"` — asked once per order (for example organisation name, dietary notes).
- `"attendee"` — asked once for each ticket or person (for example student ID, membership number).

**Field shape.** A registration field takes `id`, `label`, optional `placeholder`, optional `helpText`, `required` (default false), `scope`, optional `type` (text by default; also `textarea`, `select`) and `options` when `type` is `select`.

**Purchase flow.** Every event, of every type, follows the same path:

**Registration form → Checkout page → Payment → Confirmation page (or Payment return page)**

1. The buyer completes the registration form: event and tier fields, team fields (Gap 1), or every runner's details in the race wizard (Gap 6).
2. Submitting the form creates an unpaid order and takes the buyer straight to that order's checkout page (`/checkout/<order-id>`). There is no separate "add to cart" step.
3. On the checkout page the buyer reviews the order, redeems coupons, chooses additional services and pays (Gap 10).
4. After a successful payment the buyer lands on the confirmation page (Gap 11). If the payment is cancelled, fails, is still pending or hits any other issue, the buyer lands on the payment return page (Gap 13) instead.

If the buyer leaves before paying, the unpaid order stays in their cart (Gap 8) so they can return and complete payment later, within the limits of the order hold setting. Free orders (total of 0) skip the payment step.

**Where orders live.** Unpaid orders are in the cart (Gap 8). Paid and confirmed orders are in the buyer's account at `/profile/orders` (Gap 14). An order is never in both places.

**Regular vs seated tickets.** A regular (general admission) ticket is a quantity from a tier. A seated ticket is a specific seat on the seat map. This distinction drives Gaps 8–9: only seated tickets are ever reserved by an unpaid order.

## Gap 1 — Team registration mode

Switch Econo Carnival Bangladesh Season 01 to team registration, so one buyer registers a whole team with team-level and member-level details.

- **Event to change:** [V4](https://tickify.bickfoundation.org/o/tickify/econo-carnival-bangladesh-season-01) · [V3 (current behaviour)](https://tickify.live/event/econo-carnival-bangladesh-season-01/)
- **Reference:** [creator-tech-expo](https://abir-web-development.up.railway.app/events/creator-tech-expo)

**Step 1 — Event settings.** Set team mode, the sidebar layout, and the default team size and fields for all tiers.

```ts
registrationMode: "team",
registrationLayout: "sidebar",
showClusterFilter: true,
teamRegistration: {
  minSize: 1,
  maxSize: 10,
  teamFields: creatorTeamFields,     // asked once per team
  memberFields: creatorMemberFields, // asked for each member
},
```

**Step 2 — Tier override (optional).** A tier can set its own team size and fields, which replace the event defaults for that tier.

```ts
{
  id: "team-pro",
  name: "Team Pro",
  description: "Everything in Standard plus office hours and reserved seating.",
  price: 1600,
  remaining: 35,
  maxPerOrder: 8,
  badge: "For product teams",
  separateRegistrationPage: true,
  teamRegistration: {
    minSize: 2,
    maxSize: 8,
    teamFields: [
      {
        id: "organization-name",
        label: "Organization or Institution",
        placeholder: "Organization name",
        helpText: "This appears on the team booking and group check-in list.",
        required: true,
        scope: "order",
      },
    ],
  },
}
```

**Expected behaviour**

- The buyer cannot submit a team smaller than `minSize` or larger than `maxSize`.
- Team fields are shown once; member fields repeat for each member.
- Keep `maxPerOrder` consistent with `teamRegistration.maxSize` (both 8 in the example).
- Submitting the team registration takes the buyer to checkout (Gap 10).

**Open question:** the contents of `creatorTeamFields` and `creatorMemberFields` for Econo Carnival are not specified yet.

## Gap 2 — Ticket tier clusters

Group tiers under a named cluster (for example a session or category) and let buyers filter the tier list by cluster.

- **Reference:** [food-carnival](https://abir-web-development.up.railway.app/events/food-carnival)

**Step 1 — Event settings.** Turn on the filter.

```ts
showClusterFilter: true,
```

**Step 2 — Tier config.** Give each tier a `cluster` name. Tiers with the same name are grouped together.

```ts
{
  id: "lunch-entry",
  name: "Lunch Entry",
  description: "Entry to the daytime session.",
  price: 350,
  remaining: 74,
  maxPerOrder: 8,
  cluster: "Lunch Session",
}
```

**Expected behaviour**

- A filter shows one option per distinct `cluster` value; choosing one hides tiers from other clusters.
- Cluster names must match exactly (case and spacing) to group together.

## Gap 3 — Event-level additional fields

Add extra questions that every buyer answers, whichever tier they pick.

**Config — event details.**

```ts
registrationFields: [
  {
    id: "dietary-notes",
    label: "Dietary requirements",
    placeholder: "Optional allergies or dietary notes",
    helpText: "Share allergies or requirements the event team should know about.",
    scope: "order",
    type: "textarea",
  },
],
```

**Expected behaviour**

- The field appears on the registration form for every tier.
- With `scope: "order"` it is asked once per order; no `required` means it is optional.

## Gap 4 — Tier-level additional fields

Add questions that only buyers of one specific tier answer.

**Config — on the tier.**

```ts
{
  id: "sunset-tasting",
  name: "Tasting Pass",
  description: "Sunset entry plus five tasting tokens.",
  price: 950,
  remaining: 22,
  maxPerOrder: 6,
  cluster: "Sunset Session",
  registrationFields: [
    {
      id: "tasting-track",
      label: "Preferred tasting track",
      placeholder: "Choose a track",
      required: true,
      scope: "order",
      type: "select",
      options: ["Local classics", "Grill and smoke", "Dessert trail"],
    },
  ],
}
```

**Expected behaviour**

- The field appears only when this tier is in the order, alongside any event-level fields.
- `type: "select"` renders a dropdown from `options`; `required: true` blocks the registration form from being submitted until it is answered.

## Gap 5 — Separate registration page per tier

Give each tier its own registration page, reached from a category-select page, instead of registering on the event page.

- **Event to change:** [V4](https://tickify.bickfoundation.org/o/icab/ca-bangladesh-accounting-day-run-2026) · [V3 (current behaviour)](https://tickify.live/event/ca-bangladesh-accounting-day-run-2026/)
- **Reference:** [dhaka-marathon](https://abir-web-development.up.railway.app/events/dhaka-marathon) · [category select page](https://abir-web-development.up.railway.app/events/dhaka-marathon/register)

**Config — on each tier.** Set `separateRegistrationPage: true`. Attendee-scope fields collect per-person details on that page.

```ts
{
  id: "student-walker",
  name: "Student Walker",
  description: "Discounted place with a valid student ID.",
  price: 300,
  remaining: 5,
  maxPerOrder: 3,
  cluster: "Student admission",
  separateRegistrationPage: true,
  allowedCouponCodes: [],
  registrationFields: [
    {
      id: "student-id",
      label: "Student ID",
      placeholder: "Enter student ID",
      required: true,
      scope: "attendee",
    },
  ],
}
```

**Expected behaviour**

- `/events/<event>/register` lists the tiers; each links to `/events/<event>/register/<tier-id>`.
- Each tier page shows only that tier's fields plus event-level fields.
- With `scope: "attendee"`, Student ID is asked once per ticket (up to 3 here).
- Submitting a tier page takes the buyer to checkout (Gap 10).

## Gap 6 — Race / marathon wizard registration

For marathon events, combine separate tier pages (Gap 5) with race mode: a step-by-step wizard that collects each runner's details, with no registration on the event page itself.

- **Event to change:** [V4](https://tickify.bickfoundation.org/o/icab/ca-bangladesh-accounting-day-run-2026) · [V3 (current behaviour)](https://tickify.live/event/ca-bangladesh-accounting-day-run-2026/)
- **Reference:** [dhaka-marathon](https://abir-web-development.up.railway.app/events/dhaka-marathon) → [category select](https://abir-web-development.up.railway.app/events/dhaka-marathon/register) → [a tier page](https://abir-web-development.up.railway.app/events/dhaka-marathon/register/half-marathon-memberelite)

**Step 1 — Event settings.**

```ts
registrationMode: "race",
registrationLayout: "wizard",
onPageRegistration: false,          // no registration form on the event page
collectIndividualInformation: true, // details for every runner
showClusterFilter: true,
```

**Step 2 — Tier config.** Each race category is a tier with its own page, cluster (distance) and runner fields.

```ts
{
  id: "half-marathon-member-elite",
  name: "Half Marathon — Member Elite",
  description: "Members with a qualifying time, seeded at the front.",
  price: 2900,
  remaining: 26,
  maxPerOrder: 1,
  cluster: "Half Marathon",
  separateRegistrationPage: true,
  cardImageUrl: "https://images.unsplash.com/photo-1552674605-db6ffd4facb5?auto=format&fit=crop&w=600&q=80",
  registrationFields: [
    {
      id: "icab-membership",
      label: "ICAB membership number",
      required: true,
      scope: "attendee",
    },
    {
      id: "previous-finish",
      label: "Previous finish time at this distance",
      helpText: "Used for seeding. For example 1:42.",
      placeholder: "H:MM",
      required: true,
      scope: "attendee",
    },
  ],
}
```

**Expected behaviour — the buyer's path**

1. The event page shows the categories but no inline registration form.
2. The category select page lists tiers as cards (using `cardImageUrl`), filterable by cluster.
3. The tier page opens the wizard, which collects each runner's details and the tier's fields.
4. The registration form is submitted at the end of the wizard, and the buyer goes straight to checkout (Gap 10).

**Check:** the reference tier URL ends in `half-marathon-memberelite` while the tier `id` is `half-marathon-member-elite`. Confirm whether the route slug is derived from `id` or set separately.

## Gap 7 — Tier-level coupon assignment

Restrict which coupon codes can be used on each tier. Buyers enter coupons on the checkout page (Gap 10).

**Config — on the tier.** List the allowed codes in `allowedCouponCodes`.

```ts
{
  id: "student-walker",
  name: "Student Walker",
  description: "Discounted place with a valid student ID.",
  price: 300,
  remaining: 5,
  maxPerOrder: 3,
  cluster: "Student admission",
  separateRegistrationPage: true,
  allowedCouponCodes: [], // e.g. ["STUDENT10"]
}
```

**Expected behaviour**

- A code outside the tier's list is rejected for that tier.
- In an order with more than one tier, a code applies only to the tiers that allow it, and is rejected if no tier in the order allows it.
- Codes are validated on the server when applied at checkout, not only in the browser.

**Open question:** what an empty list means — no coupons allowed on this tier, or all event coupons allowed. Confirm against the [ticket category config doc](https://github.com/stage-crew/platform-docs/blob/main/ticket-category-config.md#ticketcategory-config).

## Gap 8 — Cart and order hold timeout

The cart holds a buyer's unpaid orders so they can come back and complete payment later. V4 needs it for every event type. The cart is always on; there is no setting to turn it off.

The only event-level control is the **Order Hold Timeout** toggle in Event Settings, because not every event needs an order timeout. When it is enabled, the organizer sets the hold duration in minutes.

- **Reference:** [V4 cart](https://abir-web-development.up.railway.app/cart)

**Config — event settings.**

```ts
orderHold: {
  enabled: true,
  durationMinutes: 10, // default 10; no min or max; only used when enabled
},
```

**Expected behaviour — cart (always on)**

- A **Cart** button sits in the site navbar on every page, for signed-in and guest buyers, and opens the cart page.
- Submitting the registration form creates an unpaid order. The order appears in the cart immediately, and the buyer is taken straight to its checkout page (see Purchase flow). There is no "add to cart" step.
- The cart shows one order card per event. Each card opens that order's checkout page (`/checkout/<order-id>`) so the buyer can resume.
- Because registration comes first, every order has the buyer's email, signed in or not.
- Paid orders leave the cart and appear on the buyer's account orders page (Gap 14).
- The cart page is always reachable, including when it is empty; an empty cart shows an empty state rather than an error or redirect.
- **Regular tickets are not reserved by an unpaid order.** Availability is only taken at payment. If a regular tier sells out while an order for it is unpaid, checkout must re-check availability and tell the buyer before payment is attempted, not after.
- **Seated tickets are reserved** when the order is created, for as long as the hold setting below allows.

**Expected behaviour — Order Hold Timeout enabled**

- The organizer sets the hold duration in minutes. The default is 10, and there is no minimum or maximum.
- The timer starts when the order is created (registration form submitted). Each order has its own timer. Returning to checkout, applying coupons, choosing add-on services or a failed payment attempt does not reset it. It is enforced on the server, not only in the browser.
- Each order card in the cart shows a countdown of the time left to complete payment before the order expires. The checkout page shows the same countdown.
- When the time runs out, the unpaid order expires and leaves the cart.
- For seated events, the reserved seats are released and become selectable by other buyers immediately.
- A payment already in progress when the timer runs out must not be lost: either extend the hold while the payment gateway session is open, or reject and refund. Decide which. If the order is rejected, the buyer sees that on the payment return page (Gap 13).

**Expected behaviour — Order Hold Timeout disabled**

- The unpaid order stays in the cart for as long as the event is live. Its order card shows no countdown.
- For seated events, the selected seats stay reserved for that buyer for the same period.
- Regular tickets are still not reserved.
- When the event ends (or goes off sale), open orders expire.
- The abandoned-cart reminder (Gap 9) is available for the event.

**Risk — seated events with the hold disabled.** Abandoned orders keep seats reserved until the event ends. On a seated event, every abandoned order removes those seats from sale; the seat map can look sold out while the actual sell-through is far lower. Recommendation: allow `enabled: false` only on non-seated events, or require the hold to be enabled when the event has a seat map. Pending sign-off.

**Open questions**

- When a hold expires, is the buyer's registration data discarded with the order, or kept so they can re-register without re-typing it? This matters most for race and team registrations with many attendees.
- With no minimum, what happens if an organizer sets `durationMinutes` to 0 or leaves it blank while the hold is enabled?
- If a buyer registers again for the same event before paying, does the new registration merge into the existing order (one card per event) or create a second order card?
- How does a guest buyer see their cart on another device or after clearing their browser — only through the reminder link (Gap 9), or by email lookup?
- Can a buyer remove an unpaid order from the cart themselves, releasing any reserved seats?

## Gap 9 — Abandoned-cart reminder email

Let organizers send promotional reminder emails, a set number of days before the event, to buyers who registered but did not pay. Each email contains a direct link to that buyer's unpaid order so they can resume checkout. The email template and its promotional content are managed on the admin side and are out of scope here.

**Availability.** The option appears in Event Settings **only when Order Hold Timeout (Gap 8) is disabled**. When the hold is enabled it is hidden and forced off.

**Config — event settings.**

```ts
abandonedCartReminder: {
  enabled: true,        // only settable when orderHold.enabled === false
  daysBeforeEvent: 3,   // send N days before the event starts
},
```

**Expected behaviour**

- On the day set by `daysBeforeEvent`, a reminder goes to every buyer with an unpaid order for this event.
- Each email contains a direct link to the buyer's own unpaid order. The link opens that order's checkout page (Gap 10), with the registration details and tickets (and seats, for seated events) already in place.
- The link works without signing in and uses an unguessable token. Do not use the plain order ID: the reference IDs (`watch-party-order-2027-001`) are sequential, so a link of that form would let anyone reach other buyers' orders and registration details by changing the number.
- If the order can no longer be paid (event off sale, or every tier in it sold out), the link shows a clear message instead of an error. If the order has since been paid, the link opens its confirmation page (Gap 11).
- Not sent to buyers who have since paid for this event, or when every tier in their order is sold out.
- One reminder per unpaid order.

**Open questions**

- Orders abandoned after the send date never get a reminder. Is that acceptable, or should a later order be picked up on the next daily run until the event starts?
- One reminder only, or several (for example `daysBeforeEvent: [7, 1]`)?
- Should organizers also be able to send a reminder on demand ("Send now") from the dashboard, in addition to the scheduled one?

## Gap 10 — Checkout page

After submitting the registration form, the buyer goes straight to the checkout page for that order. Checkout is where the buyer reviews the order, redeems coupons, chooses additional services and pays. It is not only a payment step.

- **Reference:** [V4 checkout](https://abir-web-development.up.railway.app/checkout/watch-party-order-2027-001)

**Config.** None for the page itself. Coupon rules come from each tier's `allowedCouponCodes` (Gap 7), and the countdown comes from the event's `orderHold` (Gap 8). Add-on service config is an open question below.

**Expected behaviour**

- The page lives at `/checkout/<order-id>` and is reached from the registration form, from an order card in the cart, from the reminder link (Gap 9), or from "Try again" on the payment return page (Gap 13).
- **Order review:** shows the event, tiers, quantities (and seats, for seated events) and the details entered on the registration form.
- **Coupons:** a coupon field validated on the server against each tier's `allowedCouponCodes` (Gap 7). The price summary updates when a code is applied or removed.
- **Additional services:** the buyer can choose optional services, such as ticket delivery via WhatsApp and a refund guarantee. Selected services appear in the price summary. If WhatsApp delivery is selected, a WhatsApp number is required (prefilled if the registration form collected a phone number).
- **Hold timer:** when the order hold is enabled, the countdown is shown here. Nothing on this page resets it.
- **Before payment:** availability of regular tiers is re-checked. If any tier in the order has sold out, the buyer is told and payment is not started.
- **Payment succeeds:** the buyer goes to the confirmation page (Gap 11) and the order leaves the cart.
- **Payment cancelled, failed, pending or any other issue:** the buyer goes to the payment return page (Gap 13), which shows what happened and what to do next. The order stays in the cart, subject to the hold timer.
- **Free orders** (total of 0, including after a 100% coupon): no payment step. Confirming the order goes straight to the confirmation page.

**Open questions**

- Are additional services offered on every event, or does the organizer choose per event? Who sets their price (Tickify or the organizer)?
- Refund guarantee: is it priced per order or per ticket, as a fixed fee or a percentage, and what are its terms?
- Do free orders still show the checkout page (for example to offer WhatsApp delivery), or go straight from registration to confirmation?
- Can the buyer edit registration details from checkout, or must they go back to the registration form?

## Gap 11 — Order confirmation page

V4 currently has no order confirmation page. Add one, shown after a successful payment, or after registration completes for a free order.

- **Template:** [V4 confirmation](https://abir-web-development.up.railway.app/checkout/art-walk-order-2027-002/success)

**Config.** None.

**Expected behaviour**

- The page lives at `/checkout/<order-id>/success`. Layout and content follow the V4 template.
- It confirms the order ID, event and tickets, and how the tickets will be delivered (including WhatsApp, if chosen at checkout).
- It is shown only when the server has confirmed the order as paid (or as a completed free registration). The payment gateway's redirect alone is not enough; the page reads the order status from the server.
- It is used for successful payments only. Cancelled, failed and pending payments, and any other issue, go to the payment return page (Gap 13).
- Opening the URL for an unpaid order sends the buyer to that order's checkout page instead, or to the payment return page if a payment is still pending.
- Reloading or reopening the page shows the same confirmation. It never creates a second order or a second charge.
- From this point the order appears on the buyer's account orders page (Gap 14).

## Gap 12 — Event card: price and Book now

On event cards (wherever events are listed), show **"Starts from <price>"** on the left and the **"Book now"** button on the right.

**Config.** None. The price and sold-out state come from the event's tiers.

**Expected behaviour**

- Left side: "Starts from" followed by the lowest tier price. Every tier counts, including tiers that are sold out or not yet on sale.
- Free events (every tier priced at 0) show "Free" instead of "Starts from <price>".
- Right side: the "Book now" button.
- Sold-out events (every tier at `remaining: 0`): the button label changes to "Sold out" and the button is disabled.
- The same layout applies on every event card, wherever event cards appear.

**Open question:** an event with both free and paid tiers has a lowest price of 0. Should its card show "Free" (which may read as the whole event being free) or "Starts from Free"?

## Gap 13 — Payment return page

When a payment does not succeed, the buyer needs to see what happened and what to do next. Add a payment return page for every outcome other than a successful payment: cancelled, failed, pending, or any other issue. Successful payments go to the confirmation page (Gap 11), never to this page.

- **Reference:** none yet. Proposed route: `/checkout/<order-id>/return` (to confirm).

**Config.** None.

**Expected behaviour**

- When the buyer comes back from the payment gateway, the outcome is read from the server's order status (confirmed with the gateway), not from the redirect URL or its parameters. Paid goes to the confirmation page (Gap 11); every other outcome goes to this page.
- The page shows the order ID, the event, the order amount, and a plain message for the outcome:

| Outcome | What the buyer is told | Next step offered |
| --- | --- | --- |
| Cancelled | The payment was cancelled and no money was taken. | Try again, or go to cart |
| Failed or declined | The payment did not go through, with the reason if the gateway gives one. | Try again, or go to cart |
| Pending | The payment is still being confirmed; do not pay again. | None while pending; the page updates when the status changes |
| Order expired during payment | The order hold ran out, and how any amount taken will be refunded (only if Gap 8 decides "reject and refund"). | Back to the event to register again |
| Tickets sold out during payment | The tickets are no longer available, and how any amount taken will be refunded. | Back to the event |
| Any other issue | Something went wrong and the order is not confirmed. | Try again, or contact support |

- "Try again" reopens the same order's checkout page (Gap 10) with the registration details, coupons and add-on services kept, so nothing has to be re-entered. It is shown only while the order can still be paid.
- When the order hold is enabled, the countdown is shown on this page too. A cancelled or failed payment does not reset the timer.
- After a cancelled, failed or pending outcome, the order stays in the cart (Gap 8) unless it has expired. None of these orders appear on the account orders page (Gap 14).
- While a payment is pending, "Try again" is not offered and the order's card in the cart shows the pending status instead of opening checkout, so the buyer cannot be charged twice for the same order. When the status resolves, the page moves to the confirmation page (paid) or to the failed state.
- Reloading or reopening the page shows the current status. It never starts a new payment.
- Like the checkout and confirmation pages, it must not expose an order to anyone who changes the order ID in the URL.

**Open questions**

- Which payment gateways V4 uses, and how each gateway's return codes map to the outcomes above.
- How long can a payment stay pending before it is treated as failed, and does a pending payment extend the order hold? This ties into the Gap 8 decision on payments in progress at expiry.
- What refund wording and timelines should the page show, and who provides the text?
- Confirm the route for this page.

## Gap 14 — Account orders page: paid orders only

The orders page in the buyer's account (`/profile/orders`) lists only paid and confirmed orders. Orders in any other state are not shown there. Unpaid orders are found in the cart (Gap 8) instead.

- **Page:** `/profile/orders`

**Config.** None.

**Expected behaviour**

- Listed: orders the server has confirmed as paid, plus confirmed free registrations (total of 0), which have nothing to pay.
- Not listed: unpaid orders (pending payment), payments still pending confirmation, and cancelled, failed or expired orders.
- An order appears as soon as it is confirmed, at the same point the confirmation page (Gap 11) is shown.
- The filter is applied on the server: the orders API returns only confirmed orders, rather than the page hiding the others in the browser.
- When the buyer has no confirmed orders, the page shows an empty state rather than listing unpaid ones.

**Open questions**

- Refunded orders (for example under the refund guarantee): do they stay on the page with a "Refunded" label, or disappear?
- Orders placed as a guest with the same email: do they appear here once the buyer signs in?

## Migration and QA checklist

**Econo Carnival Bangladesh Season 01**

- [ ] Event set to `registrationMode: "team"` with sidebar layout
- [ ] Team size limits enforced (min and max)
- [ ] Team fields asked once; member fields repeat per member
- [ ] Tier-level team overrides replace event defaults
- [ ] Behaviour matches V3 and the creator-tech-expo reference

**CA Bangladesh Accounting Day Run 2026**

- [ ] Event set to race mode, wizard layout, `onPageRegistration: false`
- [ ] Every tier has `separateRegistrationPage: true`
- [ ] Category select page lists all tiers with images and cluster filter
- [ ] Each tier page opens the wizard and collects per-runner fields
- [ ] Behaviour matches V3 and the dhaka-marathon reference

**All events — registration**

- [ ] Cluster filter groups tiers correctly
- [ ] Event-level fields show on every tier
- [ ] Tier-level fields show only for their tier
- [ ] Required fields block registration form submission; `order` vs `attendee` scope asks the right number of times
- [ ] Coupons outside a tier's `allowedCouponCodes` are rejected

**Purchase flow and checkout**

- [ ] Every event follows Registration form → Checkout → Payment → Confirmation (or Payment return page)
- [ ] Submitting the registration form (including team and runner details) creates an unpaid order and opens its checkout page
- [ ] Checkout shows the order review, coupon field, additional services and price summary
- [ ] Coupons are validated on the server; in a mixed order they apply only to tiers that allow them
- [ ] Selected add-on services appear in the price summary; WhatsApp delivery requires a number
- [ ] A regular tier that sells out while its order is unpaid is caught at checkout, before payment
- [ ] Cancelled, failed or pending payment lands on the payment return page; the order stays in the cart
- [ ] Free orders skip payment and reach the confirmation page

**Cart and hold timeout**

- [ ] Cart button is in the navbar on every page, signed in or not
- [ ] Cart shows one order card per event, each opening its checkout page; empty cart shows an empty state
- [ ] Unpaid regular tickets do not reduce `remaining`
- [ ] Seats are reserved when the order is created; other buyers cannot select them
- [ ] Hold defaults to 10 minutes; timer starts when the order is created and is not reset by revisiting checkout, coupons, add-ons or a failed payment
- [ ] Hold enabled: countdown shown on each order card and on checkout; order expires at the set minutes; seats released immediately
- [ ] Hold enabled: payment in progress at expiry is handled per the decided rule
- [ ] Hold disabled: no countdown; order and seat reservations persist until the event ends, then expire
- [ ] Paid orders leave the cart and appear on `/profile/orders`

**Abandoned-cart reminders**

- [ ] Reminder option is hidden when the hold is enabled
- [ ] Reminder sent `daysBeforeEvent` days before the event to buyers with unpaid orders only
- [ ] Each email links directly to the buyer's own unpaid order and opens its checkout without sign-in
- [ ] The link uses an unguessable token; changing it does not reveal another buyer's order
- [ ] Link to an off-sale or sold-out order shows a clear message; link to a paid order opens its confirmation page
- [ ] No reminder after payment or when every tier in the order is sold out

**Confirmation page**

- [ ] Confirmation page shown only after successful payment or free registration, matching the V4 template
- [ ] Page reads order status from the server; an unpaid order's success URL redirects to its checkout, or to the payment return page if a payment is pending
- [ ] Reloading the page does not create a second order or charge

**Payment return page**

- [ ] Every non-successful gateway return lands on the payment return page; successful ones land on the confirmation page
- [ ] Outcome comes from the server's order status, not the redirect URL
- [ ] Cancelled, failed, pending, expired, sold-out and other issues each show their own message and next step
- [ ] "Try again" reopens the same order's checkout with registration details, coupons and add-ons kept; hidden when the order can no longer be paid
- [ ] Pending: no "Try again", and the cart card does not open checkout; the page moves to confirmation or the failed state when the status resolves
- [ ] Hold countdown shown and not reset by a cancelled or failed payment
- [ ] Reloading the page never starts a new payment
- [ ] Changing the order ID in the URL does not reveal another buyer's order

**Account orders page**

- [ ] `/profile/orders` lists only paid and confirmed orders (including confirmed free registrations)
- [ ] Pending payment, payment pending confirmation, cancelled, failed and expired orders do not appear
- [ ] A newly paid order appears straight after payment
- [ ] Filtering is done on the server; the orders API does not return unconfirmed orders
- [ ] A buyer with no confirmed orders sees an empty state

**Event cards**

- [ ] "Starts from <price>" on the left and "Book now" on the right on every event card
- [ ] Lowest price includes sold-out and not-yet-on-sale tiers
- [ ] Free events show "Free"
- [ ] Sold-out events show a disabled "Sold out" button

## References

Full list of event settings and registration modes: [Tickify event details and registration](https://github.com/stage-crew/platform-docs/blob/main/tickify-event-details-and-registration.md#tickify-event-details-and-registration). Full tier options: [Ticket category config](https://github.com/stage-crew/platform-docs/blob/main/ticket-category-config.md#ticketcategory-config).

| Event | V3 | V4 |
| --- | --- | --- |
| Econo Carnival Bangladesh S01 | [tickify.live](https://tickify.live/event/econo-carnival-bangladesh-season-01/) | [bickfoundation](https://tickify.bickfoundation.org/o/tickify/econo-carnival-bangladesh-season-01) |
| CA Bangladesh Accounting Day Run 2026 | [tickify.live](https://tickify.live/event/ca-bangladesh-accounting-day-run-2026/) | [bickfoundation](https://tickify.bickfoundation.org/o/icab/ca-bangladesh-accounting-day-run-2026) |

Reference builds: [creator-tech-expo](https://abir-web-development.up.railway.app/events/creator-tech-expo), [food-carnival](https://abir-web-development.up.railway.app/events/food-carnival), [dhaka-marathon](https://abir-web-development.up.railway.app/events/dhaka-marathon).

Checkout references: [cart](https://abir-web-development.up.railway.app/cart), [checkout](https://abir-web-development.up.railway.app/checkout/watch-party-order-2027-001), [confirmation](https://abir-web-development.up.railway.app/checkout/art-walk-order-2027-002/success). Payment return page and account orders page: no reference yet.
