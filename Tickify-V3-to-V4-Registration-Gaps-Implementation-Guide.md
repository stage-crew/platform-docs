# Tickify V3 → V4 Registration Gaps: Implementation Guide

## Purpose

V4 must support nine registration and checkout features; this guide specifies each one as config plus the expected behaviour.

For each gap you get: what it does, which live events need it, the config to set, the expected V4 behaviour, and a working reference event where one exists.

**Scope.** Gaps 1–7 are config in sample/mock data on the frontend (tickify-web); no backend work. Gaps 8–9 (cart with order hold timeout, abandoned-cart reminders) cannot be done frontend-only: holds, expiry, seat release and scheduled emails must run on the server. Their config is specified here the same way, but each needs backend work before it can be QA'd end to end.

How to use it: pick a gap, apply the config to the listed event in V4, then compare against the V3 page and the V4 reference event. Tick it off in the QA checklist at the end.

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

## Core concepts

Every setting lives either on the event (applies to all tiers) or on a ticket tier (applies to that tier only). Tier-level config adds to or overrides event-level config.

| Level | Where it lives | Examples |
| --- | --- | --- |
| Event | Event settings object | `registrationMode`, `registrationLayout`, `showClusterFilter`, `onPageRegistration`, `collectIndividualInformation`, `teamRegistration`, `registrationFields`, `orderHold`, `abandonedCartReminder` |
| Ticket tier | Each item in the event's tiers list | `cluster`, `separateRegistrationPage`, `teamRegistration`, `registrationFields`, `allowedCouponCodes`, `cardImageUrl` |

**Field scope.** Every registration field has a `scope`:

- `"order"` — asked once per order (for example organisation name, dietary notes).
- `"attendee"` — asked once for each ticket or person (for example student ID, membership number).

**Field shape.** A registration field takes `id`, `label`, optional `placeholder`, optional `helpText`, `required` (default false), `scope`, optional `type` (text by default; also `textarea`, `select`) and `options` when `type` is `select`.

**Regular vs seated tickets.** A regular (general admission) ticket is a quantity from a tier. A seated ticket is a specific seat on the seat map. This distinction drives Gaps 8–9: only seated tickets are ever reserved by a cart.

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

## Gap 6 — Race / marathon wizard registration

For marathon events, combine separate tier pages (Gap 5) with race mode: a step-by-step wizard that collects each runner's details, with no registration on the event page itself.

- **Event to change:** [V4](https://tickify.bickfoundation.org/o/icab/ca-bangladesh-accounting-day-run-2026) · [V3 (current behaviour)](https://tickify.live/event/ca-bangladesh-accounting-day-run-2026/)
- **Reference:** [dhaka-marathon](https://abir-web-development.up.railway.app/events/dhaka-marathon) → [category select](https://abir-web-development.up.railway.app/events/dhaka-marathon/register) → [a tier page](https://abir-web-development.up.railway.app/events/dhaka-marathon/register/half-marathon-memberelite)

**Step 1 — Event settings.**

```ts
registrationMode: "race",
registrationLayout: "wizard",
onPageRegistration: false,        // no registration form on the event page
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
4. The registration form is submitted at the end of the wizard.

**Check:** the reference tier URL ends in `half-marathon-memberelite` while the tier `id` is `half-marathon-member-elite`. Confirm whether the route slug is derived from `id` or set separately.

## Gap 7 — Tier-level coupon assignment

Restrict which coupon codes can be used on each tier.

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

**Open question:** what an empty list means — no coupons allowed on this tier, or all event coupons allowed. Confirm against the [ticket category config doc](https://github.com/stage-crew/platform-docs/blob/main/ticket-category-config.md#ticketcategory-config).

## Gap 8 — Cart and order hold timeout

V4 needs a cart for every event type. The cart is always on; there is no setting to turn it off. The only event-level control is the **Order Hold Timeout** toggle, which sets how long an unpaid order stays in the cart.

- **Reference:** [V4 cart](https://abir-web-development.up.railway.app/cart)

**Config — event settings.**

```ts
orderHold: {
  enabled: true,
  durationMinutes: 10, // default 10; no min or max; only used when enabled
},
```

**Expected behaviour — cart (always on)**

- Every event, of every type, checks out through the cart.
- The registration form is always filled in before tickets are added to the cart. This includes event and tier fields, team fields (Gap 1) and every runner's details in the race wizard (Gap 6). The cart holds completed registrations, and checkout only takes payment.
- Because registration comes first, every cart has the buyer's email, signed in or not.
- The cart page is always reachable, including when it is empty; an empty cart shows an empty state rather than an error or redirect.
- **Regular tickets are not reserved by adding them to the cart.** Availability is only taken at payment. If a regular tier sells out while it sits in a cart, checkout must re-check availability and tell the buyer before payment is attempted, not after.
- **Seated tickets are reserved** when added to the cart, for as long as the hold setting below allows.

**Expected behaviour — Order Hold Timeout enabled**

- The organizer sets the hold duration in minutes. The default is 10, and there is no minimum or maximum.
- The timer starts when the first item is added to the cart. Adding more items does not reset it. It is enforced on the server, not only in the browser.
- The buyer sees a visible countdown in the cart and checkout.
- When the time runs out, the unpaid order expires and the cart is emptied.
- For seated events, the reserved seats are released and become selectable by other buyers immediately.
- A payment already in progress when the timer runs out must not be lost: either extend the hold while the payment gateway session is open, or reject and refund. Decide which.

**Expected behaviour — Order Hold Timeout disabled**

- The unpaid order stays in the cart for as long as the event is live.
- For seated events, the selected seats stay reserved for that buyer for the same period.
- Regular tickets are still not reserved.
- When the event ends (or goes off sale), open carts expire.

**Risk — seated events with the hold disabled.** Abandoned carts keep seats reserved until the event ends. On a seated event, every abandoned cart removes those seats from sale; the seat map can look sold out while the actual sell-through is far lower. Recommendation: allow `enabled: false` only on non-seated events, or require the hold to be enabled when the event has a seat map. Pending sign-off.

**Open questions**

- When a hold expires, is the buyer's registration data discarded with the cart, or kept so they can re-add without re-typing it? This matters most for race and team registrations with many attendees.
- With no minimum, what happens if an organizer sets `durationMinutes` to 0 or leaves it blank while the hold is enabled?

## Gap 9 — Abandoned-cart reminder email

Let organizers send a reminder email, a set number of days before the event, to buyers who added tickets to the cart but did not pay. The email template itself is managed on the admin side and is out of scope here.

**Availability.** The option appears in Event Settings **only when Order Hold Timeout (Gap 8) is disabled**. When the hold is enabled it is hidden and forced off.

**Config — event settings.**

```ts
abandonedCartReminder: {
  enabled: true,        // only settable when orderHold.enabled === false
  daysBeforeEvent: 3,   // send N days before the event starts
},
```

**Expected behaviour**

- On the day set by `daysBeforeEvent`, a reminder goes to every buyer with an unpaid, non-empty cart for this event.
- Not sent to buyers who have since paid for this event, or when every item in their cart is sold out.
- One reminder per cart.

**Open questions**

- Carts abandoned after the send date never get a reminder. Is that acceptable, or should a later cart be picked up on the next daily run until the event starts?
- One reminder only, or several (for example `daysBeforeEvent: [7, 1]`)?

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

**All events**

- [ ] Cluster filter groups tiers correctly
- [ ] Event-level fields show on every tier
- [ ] Tier-level fields show only for their tier
- [ ] Required fields block registration form submission; `order` vs `attendee` scope asks the right number of times
- [ ] Coupons outside a tier's `allowedCouponCodes` are rejected

**Cart, hold timeout and reminders**

- [ ] Every event checks out through the cart; the cart page loads when empty
- [ ] Registration form (including team and runner details) must be completed before an item can be added to the cart
- [ ] Hold defaults to 10 minutes; timer runs from the first item and does not reset when more are added
- [ ] Adding regular tickets to the cart does not reduce `remaining`
- [ ] A regular tier that sells out while in a cart is caught at checkout, before payment
- [ ] Adding seats to the cart reserves them; other buyers cannot select them
- [ ] Hold enabled: countdown shown; order expires at the set minutes; seats released immediately
- [ ] Hold enabled: payment in progress at expiry is handled per the decided rule
- [ ] Hold disabled: cart and seat reservations persist until the event ends, then expire
- [ ] Reminder option is hidden when the hold is enabled
- [ ] Reminder sent `daysBeforeEvent` days before the event to unpaid, non-empty carts only
- [ ] No reminder after payment or when the cart is sold out

## References

Full list of event settings and registration modes: [Tickify event details and registration](https://github.com/stage-crew/platform-docs/blob/main/tickify-event-details-and-registration.md#tickify-event-details-and-registration). Full tier options: [Ticket category config](https://github.com/stage-crew/platform-docs/blob/main/ticket-category-config.md#ticketcategory-config).

| Event | V3 | V4 |
| --- | --- | --- |
| Econo Carnival Bangladesh S01 | [tickify.live](https://tickify.live/event/econo-carnival-bangladesh-season-01/) | [bickfoundation](https://tickify.bickfoundation.org/o/tickify/econo-carnival-bangladesh-season-01) |
| CA Bangladesh Accounting Day Run 2026 | [tickify.live](https://tickify.live/event/ca-bangladesh-accounting-day-run-2026/) | [bickfoundation](https://tickify.bickfoundation.org/o/icab/ca-bangladesh-accounting-day-run-2026) |

Reference builds: [creator-tech-expo](https://abir-web-development.up.railway.app/events/creator-tech-expo), [food-carnival](https://abir-web-development.up.railway.app/events/food-carnival), [dhaka-marathon](https://abir-web-development.up.railway.app/events/dhaka-marathon).
