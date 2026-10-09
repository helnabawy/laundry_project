# Product

<!-- impeccable:product-schema 1 -->

## Platform

ios

Flutter (Dart SDK ^3.13.2) shipping to iPhone first; Android builds from the same
codebase and inherits the iOS-first design language. iPad is not a stated target.
Confirmed by the user, not inferred from the repo (which contains both `ios/` and
`android/` runners).

## Users

Two authenticated roles share one app, separated by role-based routing after OTP login:

- **Customer** — a UAE resident who needs laundry collected from home and returned
  clean. Reaches the app on their own phone, usually one-handed, often deciding a
  pickup slot in a spare minute. Their job: get items collected, know where the
  order stands, pay the invoice, receive items back.
- **Driver** — a courier working a shift, using the app outdoors, in a vehicle, in
  sunlight, frequently with one hand while carrying bags. Their job: see today's
  pickups and deliveries, navigate to the address, reach the customer, confirm a
  pickup or delivery, collect cash when owed, and report a failure when it happens.

Staff/facility operators are explicitly **not** app users: `staffMustUsePortal`
routes them to a web Admin Portal that does not exist yet.

Primary language is **English (LTR)**, with Arabic (RTL) as a fully supported
locale chosen on first run and switchable from Account. Both locales ship complete
ARB catalogs today; no string may be hard-coded.

## Product Purpose

A laundry pickup-and-delivery service, booked either of two ways: the customer
picks priced products from a catalogue and pays a fixed total up front, or the
customer just says what category needs cleaning and the facility counts and
prices the items after pickup. Either way, a driver collects the items and a
driver returns them in a delivery window. Success is an order that moves from
booked to delivered without the customer having to ask anyone where it is, and
a driver who can complete a stop without leaving the app.

## Positioning

The app supports **two coexisting order paths**, chosen by the customer per order,
not a global mode:

- **Shop flow** — the customer browses a flat, priced product catalogue (e.g.
  "T-shirt — AED 2"), builds a cart with quantities, and sees the exact total
  — including any VIP surcharge and cash-on-delivery fee — before confirming.
  The price is fixed at checkout; nothing changes it afterwards. `Invoice` is
  built the moment the order is created, `LaundryOrder.lines` is empty, and
  the order's status skips the facility inspection/payment wait entirely.
- **Wizard flow** (the original path) — the customer picks a category and
  sub-service; price is set **after** physical inspection at the facility, never
  estimated at booking. The wizard deliberately shows no price, and `Invoice`
  only exists once the laundry has counted the items post-pickup.
  `LaundryOrder.lines` is never empty for this path.

Both paths share the same `OrderStatus` enum, `CreateOrder` use case, and tracking
UI — `order.lines.isEmpty` is the one signal presentation code uses to tell them
apart (see Order lifecycle below). Two service tiers (Standard and VIP) exist in
both flows and differ by turnaround hours, which in turn filter the delivery
slots offered; in the shop flow VIP is also a visible cart-level surcharge.

## Operating Context

- **Auth:** phone (UAE format `5X XXX XXXX`) + 4-digit OTP, then profile completion
  with a first pickup address. Session expiry on 401 sends the user back to login
  with an explanation.
- **Choosing a laundry:** several laundries can run on the platform, each with its
  own catalogue, prices, service levels, time slots and drivers. A customer picks
  one after sign-in, and the choice is skipped when only one laundry exists. Home
  shows "Ordering from …" with a Change action. Switching empties the cart, after
  confirming, because prices differ between laundries. A reorder switches back to
  the laundry that handled the original order. Drivers never choose: they belong
  to one laundry.
- **Staying current:** order status changes arrive as push notifications (Firebase).
  A notification refreshes the screens showing that order, and tapping it opens the
  order, or the task for a driver. While the shop is open, a laundry's price edits
  refresh it silently. Every list also supports pull-to-refresh, and the app
  refetches when it returns to the foreground after 30 seconds or more.
- **Customer flow — shop path:** Home ("Shop our products" / the Products tab) →
  flat product grid with category filter chips → cart (quantities, running
  subtotal) → 2-step checkout (VIP toggle + payment method, with a live,
  already-confirmed total; then pickup & delivery slots + address) →
  confirmation (itemized total shown immediately) → tracking timeline → delivered.
  Reordering a past shop order repopulates the cart from that order's invoice
  and re-runs checkout.
- **Customer flow — wizard path:** Home ("Quick order") → 4-step order wizard
  (what to clean → service type → service level → pickup & delivery slots) →
  confirmation → tracking timeline → invoice (issued after pickup) → payment
  (bank card via external browser, or cash on delivery) → delivered.
- **Condition report (client decision, wizard path only):** stains or damage
  found while sorting are *always* reported to the customer after sorting and
  before processing. They appear first on the invoice (plus a notice on
  tracking), and the customer must tick "I've reviewed the condition report"
  before choosing a payment method — the step that starts processing. When
  nothing is found, the invoice says so. The shop path has no sorting step to
  report from — price and payment are both already settled at checkout.
- **Cash-on-delivery fee (shop path):** paying cash on delivery adds a flat
  handling fee (`PaymentMethod.codFee`, currently AED 5) on top of the cart
  total — shown live in checkout next to the "Pay on delivery" option and as
  its own line on the confirmation, tracking invoice, and receipt. Paying by
  card carries no fee. The wizard path doesn't have this fee today; its
  payment method is chosen after the facility has already priced the order.
- **Invoice inquiries (client decision):** there is no direct contact line.
  "Question about this invoice?" opens common answers first, then an
  electronic assistant; a team member is the last step, offered only after
  the assistant has not helped, and only if the customer chooses it (the
  transcript goes with the hand-off).
- **Driver flow:** today's tasks split into Pickup and Delivery tabs → task detail
  with map, call/message, and navigation hand-off → confirm pickup → hand over
  to laundry; or confirm delivery, collecting cash first when the invoice is
  unpaid, with an optional proof-of-delivery photo. Either stop can be reported
  as failed with a reason. The driver's own screens don't distinguish shop vs.
  wizard orders — a pickup and a delivery look the same either way.
- **Order lifecycle:** one `OrderStatus` enum serves both paths, but each path
  only ever visits a subset of it:
  - **Wizard path:** `pending → driverAssigned → pickedUp → atFacility →
    awaitingPayment → processing → outForDelivery → delivered` — the facility
    inspects, prices, and waits for payment before processing starts.
  - **Shop path:** `pending → driverAssigned → pickedUp → processing →
    outForDelivery → delivered` — price and payment are already settled at
    checkout, so `atFacility`/`awaitingPayment` are skipped, but the items are
    still actually washed, so `processing` stays on the timeline and the
    4-stage tracking strip (vs. the wizard path's 5 stages).
  - Both paths share the terminal states `pickupFailed`, `deliveryFailed`,
    `cancelled`. `order.lines.isEmpty` is how code tells a shop order from a
    wizard order (see `order_mock_data_source.dart`'s `confirmPickup`, which
    branches on `row['invoice'] == null` for the same reason at the data layer).
- **Backend:** currently a mock data layer (`core/mock/mock_database.dart`) behind
  repository interfaces, with `dio`-backed remote data sources already written.
  The facility-side transitions that a real Admin Portal and Auto-Dispatch would
  drive are simulated so the app demos end to end. For the shop path, the mock
  is also the pricing authority: it resolves every `productId` against the live
  catalogue and computes the total server-side — a client-supplied total is
  never trusted.

## Capabilities and Constraints

- **Architecture:** clean architecture — `domain` (entities, repositories,
  use cases) / `data` (models, data sources, repository impls) / `presentation`
  (cubits, pages, widgets), wired with `get_it`; state via `flutter_bloc` cubits;
  routing via `go_router` with a refresh listenable driven by `SessionCubit`.
- **Dependencies in play:** `dio`, `equatable`, `flutter_secure_storage`,
  `shared_preferences`, `intl`, `url_launcher`, `go_router`, `flutter_bloc`,
  `get_it`. No image, animation, or design-system package is present today.
- **Localization:** `flutter_localizations` + generated `AppLocalizations` from
  `lib/l10n/app_en.arb` / `app_ar.arb`. Every user-facing string is a key.
- **Currency:** AED, formatted through the `currencyAed` key.
- **Assets:** the project has **no** `assets/` directory, no bundled fonts, and no
  imagery. The brand mark is drawn in code (`CustomPainter`). Any new asset or font
  must be added deliberately and declared in `pubspec.yaml`.
- **Maps:** no map SDK is integrated — `map_placeholder.dart` is a stand-in and
  navigation hands off to the system Maps app via `url_launcher`.
- **Not built yet:** auto-dispatch, real payments
  (card payment opens an external browser), address geocoding.

## Brand Commitments

None binding. The user has explicitly released the incumbent identity — the ink /
teal / gold palette in `core/theme/app_colors.dart`, the painted rhombus-with-drop
logo in `core/widgets/app_logo.dart`, and the current component shapes — for
replacement. The product name in-app is the localized `appName` ("Laundry" /
"غسيل") with tagline "Wash, iron & delivery to your door"; no legal name, logo
file, or trademark asset has been supplied.

## Evidence on Hand

- Real, complete bilingual copy for every screen (`lib/l10n/app_en.arb`,
  `lib/l10n/app_ar.arb`) — use it rather than inventing labels.
- Realistic seeded content in `lib/core/mock/mock_database.dart` (service
  categories, sub-services, tiers, slots, orders, invoices, driver tasks) and
  in `catalog_mock_data_source.dart` (a 15-item bilingual shop product
  catalogue spanning the same categories, AED-priced).
- **No** real customer testimonials, ratings, photography, partner logos, press,
  pricing tables, or usage statistics exist. None may be fabricated in UI.

## Product Principles

1. **Price is honest about when it's known.** The shop path shows a real,
   confirmed total before the order is placed, because it genuinely knows the
   price — the customer chose priced products. The wizard path never estimates,
   implies, or previews a total before the facility issues the invoice, because
   it genuinely doesn't know yet. Neither path shows a number it can't stand
   behind; which one that is depends on how the order was placed, not on UI
   polish.
2. **State is always answerable.** At any moment the customer can see where the
   order is and what happens next without contacting support.
3. **The driver's screen is a working surface.** Outdoor legibility, one-handed
   reach, and unmistakable confirm actions outrank expression on every driver screen.
4. **Both locales are first-class.** Nothing may look like a translation
   afterthought in Arabic; layout is direction-agnostic by construction.
5. **Irreversible actions are guarded.** Confirming pickup, confirming delivery,
   collecting cash, and reporting a failure each state their consequence before
   they commit.

## Accessibility & Inclusion

No formal standard has been mandated by the user. Product-specific needs that are
nonetheless binding: iOS Dynamic Type must not break layouts (both roles include
older customers and drivers working in poor light); every tappable control meets
44×44 pt; Dark Mode is a first-class appearance; RTL mirroring must be complete;
and driver confirm/fail actions must be distinguishable without relying on color alone.
