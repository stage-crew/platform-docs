# Tickify V3 → V4 Registration Gaps: Implementation Guide

## Purpose

V4 must support seven registration features that V3 events already rely on; this guide specifies each one as config plus the expected behaviour.

For each gap you get: what it does, which live events need it, the config to set, the expected V4 behaviour, and a working reference event. All config lives in sample/mock data on the frontend (tickify-web); there is no backend work in scope.

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
| 8 | Pre-registration and notify me | Event | Any event announced before sale | Stage Laughs, Winter Charity Gala (sample data) |

## Core concepts

Every setting lives either on the event (applies to all tiers) or on a ticket tier (applies to that tier only). Tier-level config adds to or overrides event-level config.

| Level | Where it lives | Examples |
| --- | --- | --- |
| Event | Event settings object | `registrationMode`, `registrationLayout`, `showClusterFilter`, `onPageRegistration`, `collectIndividualInformation`, `teamRegistration`, `registrationFields` |
| Ticket tier | Each item in the event's tiers list | `cluster`, `separateRegistrationPage`, `teamRegistration`, `registrationFields`, `allowedCouponCodes`, `cardImageUrl` |

**Field scope.** Every registration field has a `scope`:

- `"order"` — asked once per order (for example organisation name, dietary notes).
- `"attendee"` — asked once for each ticket or person (for example student ID, membership number).

**Field shape.** A registration field takes `id`, `label`, optional `placeholder`, optional `helpText`, `required` (default false), `scope`, optional `type` (text by default; also `textarea`, `select`) and `options` when `type` is `select`.

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

## Gap 8 — Pre-registration and notify me

Let an event be announced before it sells: the buy button is replaced by a Pre-register or Notify me button until tickets open, and pre-registrants can optionally buy early during an early-access window.

- **Reference events (tickify-web sample data):** Stage Laughs: Live Comedy (notify me, minimal), Winter Charity Gala (pre-register with early access), Aarong FIFA World Cup 2026 Watch Party (window already closed, sale open).
- **Spec:** [Pre-registration and notify me](https://github.com/stage-crew/platform-docs/blob/main/tickify-event-details-and-registration.md#pre-registration-and-notify-me)

**It is not a registration mode.** It is an overlay on any mode, set in `settings.preRegistration`. Adding the object turns it on; removing it turns it off. There is no separate boolean.

**Config — event settings.**

```ts
preRegistration: {
  cta: "pre_register" | "notify_me",
  ctaLabel?: string,             // overrides the label derived from cta
  confirmationMessage?: string,
  fields?: RegistrationField[],  // every field must be scope: "order"
  loginRequired?: boolean,
  capacity?: number | null,      // display only; enforced server-side
  opensAt?: ISODateTime | null,
  closesAt?: ISODateTime | null, // defaults to registrationOpensAt
  earlyAccess?: { startsAt: ISODateTime, eligibility?: "all" | "selected" },
  notifications?: ("email" | "sms" | "whatsapp" | "push")[],
}
```

**Timeline**

1. `opensAt` — the list opens; the event shows state `pre_registration` and the Pre-register / Notify me button.
2. `earlyAccess.startsAt` — state becomes `early_access`; eligible viewers see the buy button.
3. `registrationOpensAt` — general sale opens; the pre-registration module removes itself.

The early-access window is derived from `earlyAccess.startsAt` and `registrationOpensAt`; never store it separately. `closesAt` closes the list, not the sale, and is unrelated to `registrationClosesAt`.

**Who gets early access**

- The viewer must be on the list (`viewer.status === "registered"`).
- If `eligibility` is `"selected"`, the viewer also needs `earlyAccessGranted`.
- With no viewer (not signed in, or the prerendered page), the state stays `pre_registration`.

**Pre-registration form**

- Shows an input only for name, email or phone the viewer's profile does not already have; if all three are known, it shows a confirm step instead.
- Extra `fields` must not repeat name, email or phone, and must not use `scope: "attendee"`. Either mistake throws in development.
- The list shows as full only when the viewer data says `listFull`; `capacity` is just a displayed number.

**Rules to keep consistent**

- `isRegistrationOpen` and `registrationOpensAt` must agree. A warning fires in development if the boolean is true before the sale date, or false after it.
- Only the pre-registration button reads the clock and viewer, on the client. Do not pass `now` from the page, or the state freezes at build time.
- For an event that only needs to say "entries open later", use `isRegistrationOpen: false` with `comingSoonText` instead of this config.

**Known gap:** a direct tier URL (`/register/<tier-id>`) is prerendered without a viewer, so it blocks eligible early-access buyers. Early access on tier pages stays incomplete until that route can resolve a viewer.

**Open question:** which live V4 events need pre-registration, and whether they use Pre-register or Notify me.

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

**Pre-registration events**

- [ ] Pre-register / Notify me button replaces the buy button before the sale
- [ ] Early access shows the buy button only to eligible viewers
- [ ] Module disappears once general sale opens
- [ ] Form asks only for missing name, email or phone
- [ ] `isRegistrationOpen` and `registrationOpensAt` agree

## References

Full list of event settings and registration modes: [Tickify event details and registration](https://github.com/stage-crew/platform-docs/blob/main/tickify-event-details-and-registration.md#tickify-event-details-and-registration). Full tier options: [Ticket category config](https://github.com/stage-crew/platform-docs/blob/main/ticket-category-config.md#ticketcategory-config).

| Event | V3 | V4 |
| --- | --- | --- |
| Econo Carnival Bangladesh S01 | [tickify.live](https://tickify.live/event/econo-carnival-bangladesh-season-01/) | [bickfoundation](https://tickify.bickfoundation.org/o/tickify/econo-carnival-bangladesh-season-01) |
| CA Bangladesh Accounting Day Run 2026 | [tickify.live](https://tickify.live/event/ca-bangladesh-accounting-day-run-2026/) | [bickfoundation](https://tickify.bickfoundation.org/o/icab/ca-bangladesh-accounting-day-run-2026) |

Reference builds: [creator-tech-expo](https://abir-web-development.up.railway.app/events/creator-tech-expo), [food-carnival](https://abir-web-development.up.railway.app/events/food-carnival), [dhaka-marathon](https://abir-web-development.up.railway.app/events/dhaka-marathon).
