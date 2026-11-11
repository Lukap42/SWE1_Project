# Restaurant Management System — REST API Endpoint Inventory

**Status:** Proposed design for team review (Checkpoint 3)  
**Purpose:** Shared contract for Message Schemas, Sequence Diagrams and later OpenAPI specification.  
**Base path (proposed):** `/api/v1`  
**Format:** JSON request/response bodies unless stated otherwise.

> This document proposes the API needed to implement the existing *User Stories-2.pdf* scope. Endpoint paths, HTTP status codes, payload fields and role rules below are **design decisions**, not an API already defined in Checkpoint 1. Confirm them as a team before drawing the sequence diagrams. The original MoSCoW prioritization is kept for traceability; double-booking prevention is treated as a **Must** invariant by the team's later decision.

## Shared conventions

| Concern | Proposed convention |
|---|---|
| IDs | Opaque UUID strings for `userId`, `dishId`, `orderId`, `reservationId`, `tableId`, `deliveryId`, `driverId`, `paymentId`, `reportId`; never derive permissions from knowing an ID. |
| Authentication | `Authorization: Bearer <accessToken>` for secured endpoints. Guests may read the menu, place an order and request a reservation. |
| Authorization | Backend validates identity, role and **ownership/assignment** of each resource. The user role alone is not sufficient to read another customer's order. |
| Date/time | ISO 8601 timestamps with timezone offset (for example `2026-11-20T18:00:00+01:00`). Reservation durations and restaurant timezone need one shared policy. |
| Money | `amountMinor` (integer in grosze) + `currency` (`PLN`), avoiding floating-point currency errors. |
| Pagination | List responses use `limit`, `cursor` query parameters and a `nextCursor` field when necessary. |
| HTTP errors | Common shape: `{ "error": { "code": "...", "message": "...", "details": {} } }`. |
| Request replay | `Idempotency-Key` header for payment-related order submission and other non-repeatable operations as needed. |
| HTTP statuses | `200` read/update, `201` created, `204` deleted/no content, `400` invalid input, `401` unauthenticated, `403` forbidden, `404` not found, `409` conflict, `422` business validation failure, `502/503` dependent service failure. Not all apply to every endpoint. |
| History & audit | Completed/cancelled records are retained for history and reporting; hard deletion is not a default business operation. |

**Actors:** `Guest/Customer`, `RestaurantManager`, `DeliveryDriver`, `RestaurantOwner`, `Admin` (proposed privileged role for granting roles).  
**Priorities:** **M** = Must Have, **S** = Should Have, **C** = Could Have. For shared support operations, the priority refers to the user story/process it enables.

## 1. Menu and dishes

| ID | Method | Endpoint | Purpose / required inputs | Successful response | Access | Priority / user story |
|---|---|---|---|---|---|---|
| MENU-01 | `GET` | `/menu` | Browse only currently available dishes; optional `category`, `limit`, `cursor` | `200` list of `dishId`, `name`, `priceAmountMinor`, `currency`, availability | Public | **M** · Browse menu |
| MENU-02 | `GET` | `/menu/{dishId}` | Read a specific available dish | `200` dish details | Public | **M** · Browse menu |
| MENU-03 | `POST` | `/menu/items` | Create dish: `name`, `priceAmountMinor`, `currency`, `available` | `201` dish with `dishId` | Manager | **S** · Update menu |
| MENU-04 | `PATCH` | `/menu/items/{dishId}` | Update price, name, or availability | `200` updated dish | Manager | **S** · Update menu |
| MENU-05 | `DELETE` | `/menu/items/{dishId}` | Remove item from current offer (prefer soft-delete/archival) | `204` | Manager | **S** · Update menu |

**Typical exceptions:** `400` invalid price/name, `404` item not found, `409` change conflicts with a concurrent update, `403` non-manager editing. `GET /menu/{dishId}` should not expose unavailable items to public users if the menu story excludes them.

## 2. Customer accounts and access management

| ID | Method | Endpoint | Purpose / required inputs | Successful response | Access | Priority / user story |
|---|---|---|---|---|---|---|
| AUTH-01 | `POST` | `/auth/register` | Create customer account: `email`, `password`, basic profile fields | `201` `userId` and account profile | Public | **S** · Create account |
| AUTH-02 | `POST` | `/auth/login` | Exchange credentials for a session/access token | `200` token/session metadata | Public | **M supporting** · Access permissions |
| AUTH-03 | `POST` | `/auth/logout` | End/invalidate the active session or refresh token | `204` | Authenticated | **M supporting** · Access permissions |
| AUTH-04 | `GET` | `/users/me` | Read current user profile and role | `200` user profile | Authenticated | **S supporting** · Account management |
| AUTH-05 | `GET` | `/users` | List employee accounts, optionally filter by role | `200` users list | Admin / authorized Manager* | **M** · Access permissions |
| AUTH-06 | `PATCH` | `/users/{userId}/role` | Change employee role: `role` | `200` updated role | Admin / authorized Manager* | **M** · Access permissions |

**Typical exceptions:** `401` missing/invalid token, `403` insufficient privilege, `409` existing email, `422` unsupported role. Passwords are never returned. **\*** Who can grant roles and whether managers may promote other managers/owners must be agreed before finalizing the authorization matrix.

## 3. Orders and payment

| ID | Method | Endpoint | Purpose / required inputs | Successful response | Access | Priority / user story |
|---|---|---|---|---|---|---|
| ORD-01 | `POST` | `/orders` | Place order: `items[{dishId,quantity}]`, `deliveryAddress`, `contact`, `paymentMethodToken`; `Idempotency-Key` | `201` order ID, status, charged total and payment status **if payment is synchronous** | Guest / Customer | **M** · Order food |
| ORD-02 | `GET` | `/orders/{orderId}` | Get current order, items, status and estimated delivery time if known | `200` order details | Owner of order (guest via secure lookup token), Manager | **S** · Track order |
| ORD-03 | `GET` | `/users/me/orders` | Get customer's previous orders (with date, items, total) | `200` paginated order history | Authenticated Customer | **C** · Previous orders |
| ORD-04 | `GET` | `/orders` | View and filter current orders, e.g. `status`, `limit`, `cursor`; operational support for preparation/assignment | `200` paginated orders | Manager | **M supporting** · Delivery assignment |
| ORD-05 | `PATCH` | `/orders/{orderId}/status` | Advance fulfillment state, e.g. `PREPARING`, `READY`; only authorized transitions | `200` updated order | Manager / authorized staff* | **M supporting** · Order fulfillment |
| ORD-06 | `POST` | `/orders/{orderId}/cancel` | Cancel an order while cancellation is allowed; record reason | `200` updated order | Owner / Manager as policy allows* | **M supporting** · Abnormal paths |
| PAY-01 | `GET` | `/orders/{orderId}/payment` | Get payment state (never raw card details) | `200` payment ID/status | Order owner / Manager | **M supporting** · Payment validation |
| PAY-02 | `POST` | `/payments/webhooks` | Provider notifies backend of successful/failed payment; verify signature and deduplicate | `200`/`204` acknowledgement | Verified payment provider | **M supporting** · Payment processing |

**Payment design decision:** The previous user story requires validated payment **before order confirmation**. For robust external payment processing, prefer a `PENDING_PAYMENT` order and an eventual confirmation from the provider webhook rather than assuming `POST /orders` can always synchronously return `CONFIRMED`. If choosing asynchronous payment, `POST /orders` should return a pending state (`201` or `202`) and the sequence diagram must show subsequent confirmation. Document the chosen option consistently.

**Typical exceptions:** `400` bad address/items, `409` unavailable dish/invalid state, `422` payment declined, `502/503` payment provider unavailable, repeated request with the same idempotency key, guest order access without the lookup token.

## 4. Table reservations

| ID | Method | Endpoint | Purpose / required inputs | Successful response | Access | Priority / user story |
|---|---|---|---|---|---|---|
| RES-01 | `GET` | `/reservations/availability` | Check available slots: `date`, `time`, `amountOfPeople` | `200` availability/options | Public | **M** · Make reservation |
| RES-02 | `POST` | `/reservations` | Book: `date`, `time`, `amountOfPeople`, `contact`; optional selected slot/table if policy permits | `201` `reservationId`, `tableId`, `status=CONFIRMED` | Guest / Customer | **M** · Make reservation; **M invariant** no double-booking |
| RES-03 | `GET` | `/reservations` | View upcoming reservations with `from`, `to`, `status`, pagination | `200` list of reservations | Manager | **M** · View reservations |
| RES-04 | `GET` | `/reservations/{reservationId}` | Read reservation details | `200` reservation | Owner (or guest with secure lookup token), Manager | **M supporting** · Reservation management |
| RES-05 | `POST` | `/reservations/{reservationId}/cancel` | Cancel reservation if permitted | `200` updated reservation | Owner / Manager* | **M supporting** · Abnormal paths |
| RES-06 | `PATCH` | `/reservations/{reservationId}/status` | Mark completed/no-show as applicable | `200` updated reservation | Manager | **M supporting** · Lifecycle consistency |
| RES-07 | `GET` | `/reservations/trends` | Aggregate heavily booked periods/tables; query `from`, `to` | `200` counts/trends | Manager | **C** · Analyze reservation trends |

**Important:** Preventing double booking cannot rely only on `GET /reservations/availability`; `POST /reservations` must atomically recheck and reserve the table/slot. Concurrent conflicting bookings yield one success and one `409 Conflict`.

**Typical exceptions:** `400` past/invalid date, invalid contact, `409` no suitable table, `404` unknown reservation, `403` no ownership, `503` notification failure (reservation result must remain unambiguous).

## 5. Delivery management

| ID | Method | Endpoint | Purpose / required inputs | Successful response | Access | Priority / user story |
|---|---|---|---|---|---|---|
| DEL-01 | `GET` | `/deliveries/unassigned` | List delivery-ready orders awaiting driver | `200` list | Manager | **M** · Assign delivery |
| DEL-02 | `GET` | `/drivers/available` | List available drivers with current planned-delivery counts | `200` drivers and workload | Manager | **M** · Assign delivery |
| DEL-03 | `POST` | `/deliveries/{deliveryId}/assignments` | Assign available `driverId` to unassigned delivery | `201` assignment/updated delivery | Manager | **M** · Assign delivery |
| DEL-04 | `GET` | `/drivers/me/deliveries` | Driver sees their active assigned deliveries | `200` list with addresses and statuses | Delivery Driver | **M** · View assigned deliveries |
| DEL-05 | `GET` | `/deliveries/{deliveryId}` | View delivery details | `200` delivery | Assigned Driver, Manager | **M supporting** · Driver operations |
| DEL-06 | `PATCH` | `/deliveries/{deliveryId}/status` | Driver updates status: `PICKED_UP`, `IN_TRANSIT`, `DELIVERED`, `FAILED` (allowed transitions only) | `200` delivery/status | Assigned Driver; Manager for exception recovery* | **S** · Mark delivered |
| DEL-07 | `POST` | `/deliveries/{deliveryId}/reassign` | Reassign driver after unavailability; requires explicit business rules | `200` updated assignment | Manager | **M supporting** · Abnormal paths |
| DEL-08 | `GET` | `/deliveries/{deliveryId}/route` | Suggested route/address or route provider reference | `200` suggested route | Assigned Driver, Manager | **C** · View estimated route |

**Typical exceptions:** `409` delivery already assigned/driver unavailable, `403` driver not assigned, `422` invalid state transition, `503` routing provider unavailable. Drivers must never be able to change another driver's delivery.

## 6. Statistics and finances

| ID | Method | Endpoint | Purpose / required inputs | Successful response | Access | Priority / user story |
|---|---|---|---|---|---|---|
| REP-01 | `GET` | `/reports/orders/popular-dishes` | Dish sales counts for a date range (`from`, `to`) | `200` dish IDs/names and order counts | Manager | **S** · View order statistics |
| REP-02 | `GET` | `/reports/financial/monthly` | Financial earnings and costs by category for `month=YYYY-MM` | `200` revenues, cost categories and totals | Owner | **M** · View financial reports |
| REP-03 | `GET` | `/reports/financial/comparison` | Compare `months` (`YYYY-MM`, repeated) | `200` per-month totals and differences | Owner | **S** · Compare months |
| REP-04 | `GET` | `/reports/financial/export` | Export selected monthly report, `month`, `format=csv|pdf` | `200` downloadable CSV/PDF | Owner | **C** · Export financial reports |

**Typical exceptions:** `400` bad date/month, `403` unauthorized user, `404` or empty report when no data (agree one behavior), `503` reporting service unavailable. Financial results need a documented source of costs; costs are not described in detail in the original user stories/class documentation.

## 7. External interfaces (not public client REST endpoints)

These interactions should also appear in relevant sequence diagrams. Exact paths depend on external providers and are **not** assumed to be the same as the restaurant's own API.

| Interaction | Direction | Data involved | Trigger | Important failure |
|---|---|---|---|---|
| Payment charge/authorization | Backend → Payment Provider | provider payment token, total minor units, currency, order reference/idempotency key | Order submission | Declined, timeout, unknown outcome |
| Payment webhook | Payment Provider → Backend (`PAY-02`) | payment reference, payment status, signature | Provider completion | Duplicate/invalid signature |
| Confirmation message | Backend → Notification Service | notification type, destination, order/reservation reference | Successful order/reservation | Provider unavailable; retry without duplicate business object |
| Route calculation | Backend → Routing Provider | origin, destination(s), mode | Driver route request (Could) | Routing unavailable |
| Data persistence | Backend → Database | data model entities and transaction/locking rules | Reads and writes | Transaction rollback, conflict |

## 8. Traceability to User Stories

| User story in Checkpoint 1 | Priority | Endpoint IDs |
|---|---|---|
| Customer — Browse menu | M | MENU-01, MENU-02 |
| Customer — Order food | M | ORD-01, PAY-02 (and payment provider call) |
| Customer — Make reservation | M | RES-01, RES-02 |
| Customer — Create account | S | AUTH-01, AUTH-02, AUTH-04 |
| Customer — Track order | S | ORD-02 |
| Customer — Previous orders | C | ORD-03 |
| Manager — View upcoming reservations | M | RES-03, RES-04 |
| Manager — Assign orders to drivers | M | ORD-04, DEL-01, DEL-02, DEL-03 |
| Manager — Different role permissions | M | AUTH-02, AUTH-05, AUTH-06 + authorization of all protected endpoints |
| Manager — Order statistics | S | REP-01 |
| Manager — Prevent double booking | **M by later team decision** (originally S) | RES-01, RES-02 (atomic booking) |
| Manager — Update menu and prices | S | MENU-03, MENU-04, MENU-05 |
| Manager — Reservation trends | C | RES-07 |
| Driver — View assigned deliveries | M | DEL-04, DEL-05 |
| Driver — Mark order delivered | S | DEL-06 |
| Driver — View estimated route | C | DEL-08 |
| Owner — View financial reports | M | REP-02 |
| Owner — Compare months | S | REP-03 |
| Owner — Export reports | C | REP-04 |

**Won't Have (intentionally excluded):** customer ratings of dishes/deliveries, predictive sales analysis, in-app driver-customer messaging, owner notifications on every new order.

## 9. Decisions the team must lock before creating message schemas

1. **Payment flow:** synchronous confirmed order vs. pending order + provider webhook; define retries, refund/cancellation and what happens if payment succeeds but the response is lost.
2. **Guest access:** secure order/reservation lookup token and how it is delivered; alternatively require login for access to tracking/details (but this changes the implied guest flow).
3. **Role policy:** who is allowed to grant or change roles (Manager vs. Admin), whether a manager can promote themselves, session revocation after role change.
4. **Reservation rules:** reservation duration, timezone, table allocation policy, overlapping-window definition, cancellation/no-show rules.
5. **Order vs. delivery states:** one agreed set of status names, ownership of transitions, and how `DELIVERED` maps to `Order.COMPLETED`.
6. **Payment and delivery service boundary:** decide which are separate communicating agents; define external interfaces in diagrams where applicable.
7. **Financial reports:** define origin of expense data, date boundary, handling of refunds and cancelled orders.
8. **Notifications and errors:** whether a confirmation is in-app, email or SMS; failed notifications must not silently change a committed reservation/order.
9. **Endpoint versioning:** confirm `/api/v1`, field naming (`amountOfPeople`), JSON format and the shared error structure.

## 10. Recommended repository layout

```text
SWE1_Project/
  api/
    REST_API_ENDPOINTS.md   # this file: endpoint inventory and team decisions
    openapi.yaml            # later: formal message schemas/OpenAPI
  Sequence Diagrams/
    ...                     # editable .puml or .drawio sources
  Reverse Sides/
    ...
```

**Workflow:** Agree this endpoint inventory → define requests/responses in `openapi.yaml` → draw sequence diagrams referring to endpoint IDs and method+path → review them together for consistency.
