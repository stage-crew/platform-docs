# Tickify V3 → V4 Registration, Checkout and Data Model Gaps: Implementation Guide

## Purpose

V4 must support sixteen features covering registration, checkout and event categorization, and must move its ticket tier and event data onto the target V4 types. This guide covers eighteen gaps: Gaps 1–16 specify each feature as config plus the expected behaviour; Gaps 17–18 map the current V4 data model to the target types.

For each feature gap you get: what it does, which live events need it, the config to set (where there is any), the expected V4 behaviour, and a working reference where one exists. For Gaps 17 and 18 you get the current shape, the target shape, a field-by-field mapping and migration notes.

**Scope.** Gaps 1–7 are config in sample/mock data on the frontend (tickify-web); no backend work. Gap 12 is a frontend layout change. Gaps 17 and 18 are type and mapper changes on the frontend. Gaps 8–11, 13 and 14 (cart and order hold timeout, abandoned-cart reminders, checkout, confirmation, payment return, account orders) cannot be done frontend-only: order creation, holds, expiry, seat release, coupon validation, payment status, order filtering and scheduled emails must run on the server. Their behaviour is specified here the same way, but each needs backend work before it can be QA'd end to end. Gap 15 requires importing V3 event categories and event-category assignments, exposing them through the events API, and connecting the homepage category slider and events-page filters to that data. Gap 16 requires event-level photo settings, tier-level overrides, conditional upload fields for individual registration, and server-side upload validation and storage.

**Migration rule (applies to every gap).** V3 does not carry the full config. When migrating V3 → V4, the V3 data fills only the fields it actually has. Every V4-only option must still exist in the output: keep its current V4 value, or give it its documented default. Never drop a V4 field because V3 has no source for it.

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
| 10 | Checkout page (review, coupons, platform fee, additional services) | Platform + event (`platformFeeEnabled`, `additionalServices`) + tier (`platformFee`) | All event types | [V4 checkout](https://abir-web-development.up.railway.app/checkout/watch-party-order-2027-001) |
| 11 | Order confirmation page | Platform | All event types | [V4 confirmation](https://abir-web-development.up.railway.app/checkout/art-walk-order-2027-002/success) |
| 12 | Event card: price and Book now | Platform | All event types | — |
| 13 | Payment return page (cancelled, failed, pending, other issues) | Platform | All event types | — |
| 14 | Account orders page: paid orders only | Platform | All event types | — |
| 15 | Homepage category slider and event categorization | Platform categories + event assignments | All event types | [V4 events](https://abir-web-development.up.railway.app/events) · [Music](https://abir-web-development.up.railway.app/events?category=Music) · [Sports](https://abir-web-development.up.railway.app/events?category=Sports) |
| 16 | Photo required for individual registration | Event default + tier override | Ticket categories with `registrationType: "individual"` | — |
| 17 | Ticket tier data model (`TicketCategory`) | Tier | All event types | — |
| 18 | Event detail, settings and media data model | Event | All event types | — |

## Core concepts

Every setting lives either on the event (applies to all tiers) or on a ticket tier (applies to that tier only). Event config is split between the top level of the event record and its `settings` object. Tier-level config adds to or overrides event-level config. Gaps 11–14 are platform behaviour with no per-event setting; Gap 10 reads the platform fee and additional-services config listed below.

| Level | Where it lives | Examples |
| --- | --- | --- |
| Event detail | Top level of the event record (`EventDetail`, Gap 18) | `registrationFields`, `registrationNote`, `clusters`, `addons`, `content.bannerMedia`, `content.thumbnailMedia` |
| Event settings | The event's `settings` object (`EventSettings`, Gap 18) | `registrationMode`, `registrationLayout`, `showClusterFilter`, `clusterSelection`, `onPageRegistration`, `collectIndividualInformation`, `teamRegistration`, `orderHold`, `abandonedCartReminder`, `platformFeeEnabled`, `additionalServices`, `photoRequired`, `photoFieldLabel`, `visibility` |
| Ticket tier | Each item in the event's tiers list (`TicketCategory`, Gap 17) | `cluster`, `separateRegistrationPage`, `teamRegistration`, `registrationFields`, `allowedCouponCodes`, `cardImageUrl`, `registrationType`, `platformFee`, `photoRequired` |

**Config snippets.** Each snippet shows only the fields relevant to its gap. A complete tier must still satisfy the `TicketCategory` type in Gap 17 (for example `slug`, `remaining` and the required `platformFee`). Prices are in major units (`price: 1600`, not minor units).

**Field scope.** Every registration field has a required `scope`:

- `"order"` — asked once per order (for example organisation name, dietary notes).
- `"attendee"` — asked once for each ticket or person (for example student ID, membership number).

**Field shape.** A registration field (`RegistrationField`, Gap 18) takes `id`, `label`, optional `placeholder`, optional `helpText`, `required` (default false), `scope`, optional `type` (text by default; also `textarea`, `select`) and `options` when `type` is `select`.

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

The current V4 tier fields `teamMinSize` and `teamMaxSize` move into this `teamRegistration` override (Gap 17).

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

**Related config.** The target settings add `clusterSelection` (`"filter" | "required"`) next to `showClusterFilter`; the behaviour of `"required"` is not specified in this guide, so confirm it against the event details doc. The event-level `clusters` list (`EventCluster`: id, name, colour, icon and `ticketTierIds`) stays as in current V4 (Gap 18).

**Open question:** tiers can be grouped both by the tier's `cluster` name and by `EventCluster.ticketTierIds`. Confirm which one drives the filter, and what happens when they disagree.

## Gap 3 — Event-level additional fields

Add extra questions that every buyer answers, whichever tier they pick.

**Config — event detail (top level).**

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

- `/events/<event>/register` lists the tiers; each links to `/events/<event>/register/<tier>` (whether `<tier>` is the tier `id` or `slug` is open; see the check in Gap 6).
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

Race categories can also carry a `raceRegistration` override and a `bibSeries` for bib numbering (both in Gap 17), and the event can set `settings.raceRegistration` (Gap 18). Bib assignment behaviour is not covered by this guide.

**Expected behaviour — the buyer's path**

1. The event page shows the categories but no inline registration form.
2. The category select page lists tiers as cards (using `cardImageUrl`), filterable by cluster.
3. The tier page opens the wizard, which collects each runner's details and the tier's fields.
4. The registration form is submitted at the end of the wizard, and the buyer goes straight to checkout (Gap 10).

**Check:** the reference tier URL ends in `half-marathon-memberelite` while the tier `id` is `half-marathon-member-elite`. Tiers keep a `slug` field in the target type (Gap 17); confirm whether the route uses `slug` or `id`.

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
- The unpaid order created when the registration form is submitted (see Purchase flow) appears in the cart immediately.
- The cart shows one order card per event. Each card opens that order's checkout page (`/checkout/<order-id>`) so the buyer can resume.
- Because registration comes first, every order has the buyer's email, signed in or not.
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

Checkout is where the buyer reviews the order, redeems coupons, chooses additional services, sees the platform fee and pays. It is not only a payment step.

- **Reference:** [V4 checkout](https://abir-web-development.up.railway.app/checkout/watch-party-order-2027-001)
- **Platform fee and additional services spec:** [event details doc](https://github.com/stage-crew/platform-docs/blob/main/tickify-event-details-and-registration.md#additional-services-and-the-platform-fee)

**Config.** The page itself has none. It reads:

| Input | Where | Used for |
| --- | --- | --- |
| `allowedCouponCodes` | Each tier (Gap 7) | Coupon validation |
| `orderHold` | Event settings (Gap 8) | Countdown |
| `platformFee` | Each tier (required, Gap 17) | Platform fee line |
| `platformFeeEnabled` | Event settings (required) | Turns the platform fee line on |
| `additionalServices.refundGuarantee.charge` | Event settings | Refund guarantee price |
| `additionalServices.whatsappTickets.charge` | Event settings | WhatsApp ticket delivery price |

**Expected behaviour**

- The page lives at `/checkout/<order-id>` and is reached from the registration form, from an order card in the cart, from the reminder link (Gap 9), or from "Try again" on the payment return page (Gap 13).
- **Order review:** shows the event, tiers, quantities (and seats, for seated events) and the details entered on the registration form.
- **Coupons:** a coupon field validated on the server against each tier's `allowedCouponCodes` (Gap 7). The price summary updates when a code is applied or removed.
- **Platform fee:** when `settings.platformFeeEnabled` is on, the price summary shows a platform fee line calculated from the `platformFee` of each tier in the order.
- **Additional services:** the refund guarantee and WhatsApp ticket delivery are offered only when configured in `settings.additionalServices`, each with its own `charge`. Selected services appear in the price summary. If WhatsApp delivery is selected, a WhatsApp number is required (prefilled if the registration form collected a phone number).
- **Price summary:** the order total includes tickets, coupon discounts, the platform fee and selected services, with each line shown separately.
- **State:** totals, the platform fee and the services total are derived in selectors and never stored. Only the buyer's choices (which services are selected) live in the registration store.
- **Hold timer:** when the order hold is enabled, the countdown is shown here. Nothing on this page resets it.
- **Before payment:** availability of regular tiers is re-checked. If any tier in the order has sold out, the buyer is told and payment is not started.
- **Payment succeeds:** the buyer goes to the confirmation page (Gap 11) and the order leaves the cart.
- **Payment cancelled, failed, pending or any other issue:** the buyer goes to the payment return page (Gap 13), which shows what happened and what to do next. The order stays in the cart, subject to the hold timer.
- **Free orders** (total of 0, including after a 100% coupon): no payment step. Confirming the order goes straight to the confirmation page.

**Open questions — to confirm from the platform fee spec before implementing**

- The platform fee formula: per ticket, per order or a percentage, and how it is rounded.
- Whether lines with `admits: false` (donations) attract a platform fee.
- How bundles are charged: once per bundle or per included ticket.
- Whether a platform fee applies to free tiers, or to orders brought to 0 by a coupon. This decides whether such an order is still free and skips payment.
- Whether each service's `charge` is per ticket or per order, and a fixed fee or a percentage; whether services are opt-in or pre-selected; and the refund guarantee's terms.
- VAT: `vatBps` is dropped from the tier (Gap 17), so confirm that no VAT is expected on the platform fee or the services.

**Open questions — product**

- Who sets each service's `charge`: Tickify or the organizer?
- How do the event-level `addons` (`EventAddon[]`, Gap 18) relate to `settings.additionalServices`? Confirm whether add-ons also appear on checkout.
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
- Opening the URL for an unpaid order sends the buyer to that order's checkout page instead, or to the payment return page (Gap 13) if a payment is still pending.
- Reloading or reopening the page shows the same confirmation. It never creates a second order or a second charge.
- From this point the order appears on the buyer's account orders page (Gap 14).

## Gap 12 — Event card: price and Book now

On event cards (wherever events are listed), show **"Starts from <price>"** on the left and the **"Book now"** button on the right.

**Config.** None. The price and sold-out state come from the event's tiers.

**Expected behaviour**

- Left side: "Starts from" followed by the lowest tier price. Every tier counts, including tiers that are sold out or not yet on sale.
- If the API supplies `minPrice` on the event (Gap 18), it must follow the same rule.
- Free events (every tier priced at 0) show "Free" instead of "Starts from <price>".
- Right side: the "Book now" button.
- Sold-out events (every tier at `remaining: 0`): the button label changes to "Sold out" and the button is disabled.
- The same layout applies on every event card, wherever event cards appear.

**Open questions**

- An event with both free and paid tiers has a lowest price of 0. Should its card show "Free" (which may read as the whole event being free) or "Starts from Free"?
- The target tier also has `soldOut`, `markSoldOut` and `status` (Gap 17). Confirm whether a tier marked sold out while stock remains counts as sold out for the card.

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

## Gap 15 — Homepage category slider and event categorization

V3 already supports event categories, and one event can belong to multiple categories. Pull the existing categories and each event's category assignments from V3 into V4, and use that data for both the homepage category section and the events-page category filters.

- **References:** [V4 events](https://abir-web-development.up.railway.app/events) · [Music filter](https://abir-web-development.up.railway.app/events?category=Music) · [Sports filter](https://abir-web-development.up.railway.app/events?category=Sports)

**Data — V3 categories and event assignments.**

- Import the existing category list from V3 rather than creating a separate hardcoded list in V4.
- Preserve each event's existing category assignments, including events assigned to more than one category.
- Keep categories as a shared platform list, with multiple category assignments supported per event.
- Use the same category data and event assignments for the homepage slider and the events-page filters.

**Expected behaviour — homepage category section**

- Keep the homepage category slider and populate it with the categories pulled from V3.
- Clicking a category opens the events page with that category selected through the `category` query parameter.
- For example, clicking **Music** opens `/events?category=Music`; clicking **Sports** opens `/events?category=Sports`.
- Build category links from the actual category values and URL-encode them where necessary.

**Expected behaviour — events page**

- `/events` keeps its category filters, populated from the same V3 category list.
- Opening `/events?category=<category>` directly selects that category and shows only events assigned to it.
- An event assigned to multiple categories appears in the results for each of those categories, once per result list.
- Changing the category filter updates the URL and the displayed events; refreshing a filtered URL retains the selected category.
- Clearing the category filter returns to the unfiltered events listing.
- If the selected category has no matching events, show a clear empty state.

**Open question:** the target `EventDetail` (Gap 18) has no categories field. Confirm where the imported assignments live: a new field, or an existing one such as `discoveryTags`.

## Gap 16 — Photo required for individual registration

V3 includes a **Photo Required** toggle in Event Settings, with a customizable field label such as **Upload your NID** or **Upload your Photo/Selfie**. In V3, this feature applies only when Individual Registration Mode is enabled.

Add this feature to V4, applicable only when the ticket category's `registrationType` is `"individual"` (`registrationType` replaces the current V4 tier field `kind`; see Gap 17). The event-level setting provides the default, and each ticket tier can override that default to enable or disable photo collection.

**Config — event default and tier override.** The following field names are proposed for V4 and are added to the types in Gaps 17 and 18; map them to the existing V3 settings during migration.

```ts
// Event Settings
photoRequired: true,
photoFieldLabel: "Upload your Photo/Selfie",

// Ticket category / tier
{
  id: "vip",
  name: "VIP",
  registrationType: "individual",
  photoRequired: true,
},
{
  id: "general",
  name: "General",
  registrationType: "individual",
  photoRequired: false,
},
```

On the ticket category, `photoRequired` is an optional boolean: omit it to inherit the event default, set it to `true` to require an upload, or set it to `false` to hide the upload field. An explicit `false` must override an event default of `true`.

**Expected behaviour**

- Retain the event-level **Photo Required** toggle and customizable field label from V3, including each event's existing values.
- Add a tier-level control with three choices: **Use event default**, **Enabled**, and **Disabled**.
- Show and require the upload field only when the category has `registrationType: "individual"` and its effective `photoRequired` setting is enabled.
- For every other registration type, hide the upload field and do not require a photo, regardless of the event default or tier override.
- When enabled, use the event's configured label and collect an upload for each attendee registered under that ticket category.
- When disabled, hide the upload field entirely; do not show it as an optional field.
- In the example above, VIP attendees must upload a photo, while General attendees do not see the upload field, even though both categories use individual registration.
- Apply the rule independently for each tier in an order. A photo requirement on one tier must not apply to attendees from another tier where photo collection is disabled.
- Validate the effective requirement on the server and associate each uploaded file with the correct attendee before proceeding to checkout.

## Gap 17 — Ticket tier data model (`TicketCategory`)

Bring the current V4 tier shape in line with the target `TicketCategory` type, and add the mapper from one to the other. The tier snippets in Gaps 1–16 already use the target names.

- **Type reference:** [Ticket category config — types](https://github.com/stage-crew/platform-docs/blob/main/ticket-category-config.md#types)

### 17.1 Current V4 tier shape

```json
{
  "id": "abaf9809-4464-59e4-8c17-85dd0dddfd4d",
  "slug": "general-15km-1481",
  "name": "General - 15KM",
  "description": "",
  "badge": null,
  "altPriceText": null,
  "kind": "GENERAL",
  "currency": "BDT",
  "faceMinor": 120000,
  "vatBps": 0,
  "minPerOrder": 1,
  "maxPerOrder": 300,
  "teamMinSize": null,
  "teamMaxSize": null,
  "isHighlighted": false,
  "registrationFields": []
}
```

### 17.2 Target shape

```ts
type TicketCategory = {
  id: string;
  slug: string;                 // kept from current V4 (not in the reference type)
  name: string;
  description: string;
  altPriceText?: string;        // kept from current V4 (not in the reference type)
  minPerOrder?: number;         // kept from current V4 (not in the reference type)
  isHighlighted?: boolean;      // kept from current V4 (not in the reference type)
  photoRequired?: boolean;      // proposed in Gap 16 (not in the reference type)
  price: number;
  originalPrice?: number;
  remaining: number;
  maxPerOrder: number;
  badge?: string;
  soldOut?: boolean;
  platformFee: number;
  status?: TicketCategoryStatus;
  allowedCouponCodes?: string[];
  bundleItems?: Array<{ categoryId: string; quantity: number }>;
  bundleSize?: number;
  cluster?: string;
  separateRegistrationPage?: boolean;
  registrationFields?: RegistrationField[];
  admits?: boolean;
  cardImageUrl?: string;
  donation?: DonationConfig;
  isDonation?: boolean;
  raceRegistration?: RaceRegistrationOverride;
  teamRegistration?: TeamRegistrationConfig;
  tournamentRegistration?: TournamentRegistrationOverride;
  bibSeries?: RaceBibSeries;    // added to TicketCategory
  allowDirect?: boolean;
  apiId?: number;
  clusterInfo?: TicketCluster | null;
  currencyCode?: string;
  currencySymbol?: string;
  isBundle?: boolean;
  markSoldOut?: boolean;
  priority?: number;
  quantity?: number;
  registrationType?: RegistrationType;
  soldCount?: number;
  tags?: string[];
  volumeDiscountTiers?: VolumeDiscountTier[];
};

type RegistrationType = "general" | "team" | "group" | "individual" | string;

export type RaceBibSeries = {
  /** Last number in the range, inclusive. Absent or `null` is unlimited. */
  end?: number | null;
  prefix?: string;
  /** Zero-pad width. Absent or `0` means no padding. */
  padding?: number;
  start: number;
};
```

`slug`, `altPriceText`, `minPerOrder` and `isHighlighted` are kept from the current V4 tier as extensions of the reference type, and `photoRequired` is added for Gap 16. `vatBps` is dropped. `cluster` and `clusterInfo` stay as in the reference type (event-level clusters are covered in Gap 18).

### 17.3 Field-by-field mapping

| Current V4 tier field | Target `TicketCategory` field | Action |
| --- | --- | --- |
| `id` | `id` | Same. |
| `slug` | `slug` | Keep; carried over unchanged. |
| `name` | `name` | Same. |
| `description` | `description` | Same (`""` stays valid). |
| `badge` (`null`) | `badge?: string` | `null` → omit the key. |
| `altPriceText` (`null`) | `altPriceText?: string` | Keep. `null` → omit the key. |
| `kind` (`"GENERAL"`) | `registrationType?: RegistrationType` | Values are lowercase, so `"GENERAL"` → `"general"`. |
| `currency` | `currencyCode` | Rename. `currencySymbol` is also needed and the tier does not carry it. |
| `faceMinor` (`120000`) | `price` | Minor units → major units (`120000` → `1200`). |
| `vatBps` | none | Drop. |
| `minPerOrder` | `minPerOrder?: number` | Keep, next to `maxPerOrder`. |
| `maxPerOrder` | `maxPerOrder` | Same. |
| `teamMinSize` / `teamMaxSize` | `teamRegistration.minSize` / `.maxSize` | Move into the `teamRegistration` override. Create the object only when at least one is non-null. |
| `isHighlighted` | `isHighlighted?: boolean` | Keep. |
| `registrationFields` | `registrationFields?` | Same name. Each field must match `RegistrationField`, with a `scope`. |

### 17.4 Target fields with no current V4 source

These do not exist on the current tier. Add them from the API or with their documented defaults:

- **Inventory and status:** `remaining`, `soldOut`, `markSoldOut`, `soldCount`, `quantity`, `status` (`TicketCategoryStatus`).
- **Fees and pricing:** `platformFee` (required), `originalPrice`, `allowedCouponCodes`, `volumeDiscountTiers`.
- **Bundles and clusters:** `bundleItems`, `bundleSize`, `isBundle`, `cluster`, `clusterInfo`.
- **Routing and display:** `separateRegistrationPage`, `allowDirect`, `cardImageUrl`, `priority`, `tags`.
- **Donation and admission:** `admits`, `isDonation`, `donation`.
- **Mode overrides:** `raceRegistration`, `teamRegistration`, `tournamentRegistration`.
- **Race bibs:** `bibSeries` (`start`, optional `end`, `prefix`, `padding`). It is optional and V3 has no source, so omit it when there is no V4 value.
- **Photo upload:** `photoRequired` (Gap 16). Omit it to inherit the event default.
- **API linkage:** `apiId`.

### 17.5 Migration notes

- V3 fills `id`, `name`, `description`, `price`, `maxPerOrder` and any other field it has. `slug`, `altPriceText`, `minPerOrder` and `isHighlighted` keep their V4 values when V3 has no source. Every other field follows the migration rule.
- `platformFee` is required and has no V3 or current tier source, so it needs an explicit default. It feeds the platform fee line on checkout (Gap 10).
- Status precedence: an authored `to_be_announced` is read before the stock count, and `on_sale` can never override zero stock.

**Open question:** confirm the minor → major divisor for BDT. It applies to `faceMinor` here and to the donation `*Minor` settings in Gap 18.

## Gap 18 — Event detail, settings and media data model

Bring the current V4 `EventDetail`, `EventSettings` and `EventContent` in line with the target types, migrate V3 media into the new media shape, and carry each event's V3 visibility into V4.

- **Reference:** [Tickify event details and registration](https://github.com/stage-crew/platform-docs/blob/main/tickify-event-details-and-registration.md) · [Visibility](https://github.com/stage-crew/platform-docs/blob/main/tickify-event-details-and-registration.md#visibility)

### 18.1 Structural gaps

| Area | Current V4 | Target | Gap |
| --- | --- | --- | --- |
| Identity | `id: string`, `slug`, `title`, `subtitle` | `slug`, `id?: number`, `title`, `eyebrow`, `summary`, optional `occurrenceId` | `id` becomes optional and numeric. `subtitle` has no direct target. `eyebrow` and `summary` are new (nullable, see 18.2). |
| Dates | `startsAt`, `endsAt`, `doorsOpenAt`, `timezone` | `starts`, `ends`, `doorsOpen` (`ISODateTime \| string`), `timezone` | Renames. |
| Status and flags | `status`, `isFeatured`, `sellable` at event level | `settings.status`, `settings.isFeatured`; no `sellable` | Move into `settings`. `EventStatus` widens to the browse statuses plus `draft`, `ongoing`, `completed`, `sold_out`. |
| Images | `imageCardUrl`, `imageBannerUrl` | `content.thumbnailMedia`, `content.bannerMedia` (`EventMedia`) | Replace single URLs with `{ type: "image" \| "video"; src }[]` (see 18.5). |
| Organisation | `organization { id, slug, name }` | `organizer` (description, email, socials, `logo_url`, `slug`, numeric `id`) | Rename and widen. |
| Venue | `venue { id, name, city, timezone }` | `venue { address, googleMapEmbedding, id: number, latitude, longitude, name } \| null`, plus event-level `location` and `address` strings | No `city` or `timezone` on the target venue. Needs address, map embed and coordinates. |
| Price | none | `priceRange?`, `minPrice?` | New. `minPrice` must follow the Gap 12 rule. |
| SEO | `seoTitle`, `seoDescription` | `content.metaDescription` | `seoDescription` → `metaDescription`. `seoTitle` has no target. |
| Content body | `content.description`, `acknowledgements`, `ticketPolicy` | `content.descriptionHtml`, `content.policyHtml` | Rename. `acknowledgements` has no target. |
| Content extras | `rulebookUrl`, `promoVideoDesktopUrl`, `promoVideoMobileUrl`, `promoBannerUrl` | Video items inside `bannerMedia` / `thumbnailMedia` | Promo media are kept (see 18.5). `rulebookUrl` has no target. |
| Content labels | none | `buttonLabel`, `creativesLabel`, `comingSoonText`, `soldOutText`, `modified` | New, optional. |
| FAQ | `content.faq: unknown \| null` | `faqs: { question; answer }[]` (top level, required) | Typed and moved. |
| Gallery, sponsors, performers | `content.gallery`, `.sponsors`, `.performers` (`unknown`) | Top-level `gallery`, `sponsors`, `creatives`, plus new `partners` | Typed and moved. `performers` → `creatives`. |
| Occurrences | `occurrences[]` | `occurrenceId?` only | Occurrence detail leaves `EventDetail`. |
| Clusters | `clusters: EventCluster[]` (id, name, colorHex, iconName, `ticketTierIds`) | Unchanged | No change. `showClusterFilter` and `clusterSelection` are still added to settings (Gap 2). |
| Registration fields | `registrationFields: unknown[]` | `registrationFields?: RegistrationField[]`, plus required `registrationNote` | Typed. |
| Payment | `paymentRails` | `paymentRails` | Same. |

### 18.2 New top-level `EventDetail` fields

- **Required:** `eyebrow`, `summary`, `location`, `address`, `highlights: string[]`, `schedule: EventScheduleItem[]`, `registrationNote`, `faqs`.
- **Optional:** `addons` (`EventAddon[]`), `imageMapData`, `discoveryTags`, `creatives`, `partners`, `sponsors`, `gallery`, `reviewSummary`, `marketingTrackers`, `created`, `modified`.

**Fields V3 cannot fill.** `eyebrow`, `summary`, `registrationNote`, `highlights` and `schedule` stay `null` when V3 has no source. The target types declare them as non-nullable (`string`, `string[]`, `EventScheduleItem[]`), so widen them to `T | null`, and make the templates handle `null` (no eyebrow, no highlights block, no schedule section).

### 18.3 `EventSettings` mapping

**Renames and retypes**

| Current | Target |
| --- | --- |
| `pageTemplate` | `eventPageTemplate` (narrowed to `EventDetailTemplate`) |
| `registrationLayout`, `registrationMode` | Same names; drop the `string` fallback so they match the `RegistrationLayout` / `RegistrationMode` unions |
| `collectIndividualInfo` | `collectIndividualInformation` |
| `showRemainingPasses` | `showRemainingTickets` |
| `showRemainingPercent` | `showAvailabilityAsPercentage` |
| `showPhaseTimer` / `showSaleCountdown` | `showSalesTimer`, with `salesPhaseEndsAt` and `salesTimerLabel` |
| `eligibility`, `eligibilityMessage` | `restrictedToAccessList`, `accessRestrictionMessage` (semantics to confirm) |
| `currency: string` | `currency: { code, symbol }` |
| `donationsEnabled`, `donationPresetsMinor`, `donationAllowAnyAmount`, `donationMinAmountMinor`, `donationMaxAmountMinor` | `donation?: DonationConfig` (`suggestedAmounts`, `allowCustomAmount`, `minAmount`, `maxAmount`). Presence of `donation` is the flag. Minor → major units (divisor in Gap 17). |
| `onPageRegistration`, `registrationOpensAt`, `registrationClosesAt` | Same |

**Current settings with no target (drop or re-home)**

`ticketingMode`, `multiCategoryRegistration`, `requireAttendeeDetails`, `loginRequired` (only `preRegistration.loginRequired` remains), `refundsEnabled`, `refundCutoffAt`, `eticketDeliveryEnabled`, `physicalDeliveryEnabled`. The last four overlap with `additionalServices` (`refundGuarantee`, `whatsappTickets`).

**New target settings**

- **Required:** `allowAccessRequests`, `autoPublishGuestMoments`, `enableGuestMoments`, `guestMomentsRequireCheckIn`, `enableInteractiveSeatMap`, `eventType`, `isFeatured`, `isRegistrationOpen`, `maxTicketsPerOrder`, `reservedSeating`, `restrictedToAccessList`, `showClusterFilter`, `clusterSelection` (`"filter" | "required"`), `salesTimerLabel`, `showSalesTimer`, `salesPhaseEndsAt`, `platformFeeEnabled`, `priceLabel`, `additionalServices`, `preRegistration`, `theme`, `applicationRegistration`, `raceRegistration`, `teamRegistration`, `tournamentRegistration`, `visibility` (see 18.6).

The reference types declare the last ten of these as optional (for example `visibility?: EventVisibility`). In V4 they are required, so drop the `?` from each. V3 has no source for most of them, so each needs a documented default during migration.

**Settings proposed in this guide (not yet in the target type).** Add these to `EventSettings` as optional fields: `orderHold` (Gap 8), `abandonedCartReminder` (Gap 9), `photoRequired` and `photoFieldLabel` (Gap 16).

### 18.4 New supporting types

`DecimalString`, `MarketingTrackerType`, `EventDetailTemplate`, `RegistrationField`, `AttachmentSpec`, `RaceWave`, `RaceAgeCategory`, `RaceRegistrationConfig`, `ApplicationStatus`, `ApplicationRegistrationConfig`, `DonationDonor`, `DonationCampaign`, `DonationConfig`, `PreRegistrationCta`, `PreRegistrationChannel`, `EarlyAccessConfig`, `PreRegistrationConfig`, `PreRegistrationViewer`, `EventCurrency`, `EventMediaItem`, `EventMedia`, `EventThemeFont`, `EventTheme`, `AdditionalServices`, `EventAddon`, `EventGalleryItem`, `EventCreative`, `EventPartner`, `EventSponsor`, `EventReviewSummary`, `EventMarketingTracker`, `EventImageMapArea`, `EventImageMapData`, `EventScheduleItem`.

`EventCluster` is not in this list: it stays as it is in current V4.

### 18.5 Media migration (V3 → V4)

- Output shape is `EventMedia` = `{ type: "image" | "video"; src: string }[]`.
- **Banner:** populate `content.bannerMedia` from V3 `home_slider_banner`, not from `banner`.
- **Thumbnail:** populate `content.thumbnailMedia` from V3 `thumbnail`.
- **V4-only media** (promo videos and extra banner items already in V4) are kept, per the migration rule.

### 18.6 Visibility migration (V3 → V4)

In V3, an event is visible on the site when `eventSettings.display` is `true`. V4 controls this with `settings.visibility`. The reference type marks it optional, with an absent value meaning public; in V4 it is required.

```ts
export type EventVisibility = "public" | "unlisted" | "private";

// EventSettings (required in V4; see 18.3)
visibility: EventVisibility;
```

**Mapping**

| V3 `eventSettings.display` | V4 `settings.visibility` |
| --- | --- |
| `true` | `"public"` |
| `false` | `"private"` |

- Every event visible in V3 stays visible in V4, and every hidden V3 event stays hidden.
- Every migrated event gets an explicit `visibility` value, because the field is required.
- V3 `display` is a real source, so its mapped value replaces any existing V4 `visibility` during the merge.
- The migration never produces `"unlisted"`; V3 has no equivalent state.

**Open questions**

- Settings with no target (listed in 18.3): drop each one, or re-home it? In particular, do the refund and delivery flags map to `additionalServices`?
- Event fields with no target (`subtitle`, `seoTitle`, `acknowledgements`, `rulebookUrl`): drop or re-home?
- Does `restrictedToAccessList` carry the same meaning as the current `eligibility` setting?
- Where do kept V4-only media go: into `bannerMedia` or `thumbnailMedia`, and in what order relative to the V3 items?
- What should a V3 event with no `display` value become: `"public"` or `"private"`?

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
- [ ] Tier route segment (`id` or `slug`) confirmed and consistent
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
- [ ] Platform fee line appears only when `platformFeeEnabled` is on, calculated from each tier's `platformFee` per the confirmed formula
- [ ] Only services configured in `settings.additionalServices` are offered, each priced by its own `charge`
- [ ] Selected services appear as separate lines; WhatsApp delivery requires a number
- [ ] Order total includes tickets, discounts, platform fee and selected services
- [ ] Totals, fee and services total are derived in selectors; only the selected services are stored
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
- [ ] Lowest price includes sold-out and not-yet-on-sale tiers, whether computed from tiers or read from `minPrice`
- [ ] Free events show "Free"
- [ ] Sold-out events show a disabled "Sold out" button

**Homepage categories and event categorization**

- [ ] Category list and event-category assignments are imported from V3
- [ ] One event can retain multiple categories and appears under each assigned category
- [ ] Homepage category slider uses the imported categories
- [ ] Clicking a homepage category opens `/events?category=<category>` with the correct filter selected
- [ ] Events-page category filters use the same category list as the homepage slider
- [ ] Music and Sports reference URLs show only events assigned to the selected category, without duplicates
- [ ] Direct links and page refreshes retain the selected category
- [ ] Changing or clearing the filter updates both the URL and the results
- [ ] A category with no matching events shows an empty state

**Photo required for individual registration**

- [ ] V3 event-level Photo Required values and custom field labels are retained in V4
- [ ] Upload field appears only for categories with `registrationType: "individual"`
- [ ] Tier with no override inherits the event default
- [ ] Tier enabled override requires an upload even when the event default is disabled
- [ ] Tier disabled override hides the upload field even when the event default is enabled
- [ ] Custom labels such as "Upload your NID" and "Upload your Photo/Selfie" appear correctly
- [ ] Required upload is collected for each attendee in an enabled individual-registration tier
- [ ] VIP can require uploads while General hides the field in the same event and order
- [ ] Non-individual categories never show or require an upload, even with photo settings enabled
- [ ] Server validates the requirement and saves each upload against the correct attendee

**Ticket tier data model**

- [ ] Tier type updated to `TicketCategory`, including `slug`, `altPriceText`, `minPerOrder`, `isHighlighted`, `photoRequired` and `bibSeries` (`RaceBibSeries`)
- [ ] Mapper converts `kind` to lowercase `registrationType`, `currency` to `currencyCode` (plus `currencySymbol`) and `faceMinor` to `price` in major units, and drops `vatBps`
- [ ] `null` `badge` and `altPriceText` are omitted
- [ ] `teamMinSize` / `teamMaxSize` move into `teamRegistration`, created only when one is non-null
- [ ] Every tier has a `platformFee`, with an explicit default where there is no source
- [ ] Every registration field matches `RegistrationField` and has a `scope`
- [ ] V3 → V4 merge keeps every V4-only tier option

**Event detail, settings and media**

- [ ] `EventDetail`, `EventSettings`, `EventContent` and the new supporting types updated
- [ ] Settings renames applied; donation settings moved into `donation` in major units
- [ ] Required settings, including `platformFeeEnabled`, `priceLabel`, `additionalServices`, `preRegistration`, `theme`, `applicationRegistration`, `raceRegistration`, `teamRegistration`, `tournamentRegistration` and `visibility`, are present on every event, with defaults where V3 has no source
- [ ] `orderHold`, `abandonedCartReminder`, `photoRequired` and `photoFieldLabel` added to `EventSettings`
- [ ] `eyebrow`, `summary`, `registrationNote`, `highlights` and `schedule` accept `null`, and templates hide their sections when `null`
- [ ] `bannerMedia` comes from V3 `home_slider_banner` (not `banner`); `thumbnailMedia` comes from V3 `thumbnail`
- [ ] V4-only media and settings are kept in the merge
- [ ] V3 `display: true` events have `visibility: "public"` and V3 `display: false` events have `visibility: "private"`; visible V3 events are still visible in V4
- [ ] `clusters` and `EventCluster` unchanged
- [ ] Open questions in Gaps 17 and 18 resolved

## References

- Event settings and registration modes: [Tickify event details and registration](https://github.com/stage-crew/platform-docs/blob/main/tickify-event-details-and-registration.md#tickify-event-details-and-registration)
- Event visibility: [Visibility](https://github.com/stage-crew/platform-docs/blob/main/tickify-event-details-and-registration.md#visibility)
- Platform fee and additional services: [Additional services and the platform fee](https://github.com/stage-crew/platform-docs/blob/main/tickify-event-details-and-registration.md#additional-services-and-the-platform-fee)
- Tier options and types: [Ticket category config](https://github.com/stage-crew/platform-docs/blob/main/ticket-category-config.md#ticketcategory-config) · [Types](https://github.com/stage-crew/platform-docs/blob/main/ticket-category-config.md#types)

| Event | V3 | V4 |
| --- | --- | --- |
| Econo Carnival Bangladesh S01 | [tickify.live](https://tickify.live/event/econo-carnival-bangladesh-season-01/) | [bickfoundation](https://tickify.bickfoundation.org/o/tickify/econo-carnival-bangladesh-season-01) |
| CA Bangladesh Accounting Day Run 2026 | [tickify.live](https://tickify.live/event/ca-bangladesh-accounting-day-run-2026/) | [bickfoundation](https://tickify.bickfoundation.org/o/icab/ca-bangladesh-accounting-day-run-2026) |

Reference builds: [creator-tech-expo](https://abir-web-development.up.railway.app/events/creator-tech-expo), [food-carnival](https://abir-web-development.up.railway.app/events/food-carnival), [dhaka-marathon](https://abir-web-development.up.railway.app/events/dhaka-marathon).

Checkout references: [cart](https://abir-web-development.up.railway.app/cart), [checkout](https://abir-web-development.up.railway.app/checkout/watch-party-order-2027-001), [confirmation](https://abir-web-development.up.railway.app/checkout/art-walk-order-2027-002/success). Payment return page and account orders page: no reference yet.
