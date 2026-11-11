# Dana’s Bakery Order System — Technical Specification

## Purpose

Build a deterministic order-processing state machine for Dana’s Bakery with LangGraph’s `StateGraph`. The system makes no LLM calls. Given the same input and menu/stock configuration, it follows the same graph path and returns the same result.

## Contract

### Input and graph state

The graph state contains these fields:

| Field | Type | Meaning |
|---|---|---|
| `item` | `str` | Requested menu item; normalized to lowercase during validation. |
| `quantity` | `int` | Number of units; must be a positive integer. |
| `day` | `str` | Order day, weekday name, or ISO date used for the Friday discount exception. |
| `city` | `str` | Delivery destination. An empty value means pickup. |
| `status` | `str` | Exactly `accepted` or `rejected` in every result. |
| `delivery` | `str` | Exactly `pickup` or `delivery`; the fee is internal pricing data. |
| `total` | `float` | Final payable total in NIS, including any delivery fee. |
| `message` | `str` | Customer-facing outcome and any substitution or delivery explanation. |
| `steps` | `list[str]` | Ordered diary of every graph node visited, including repeated visits. |
| `current_item` | `str` | Menu item currently being checked and, on acceptance, fulfilled. |
| `attempt_count` | `int` | Number of replacement candidates tried; never exceeds two. |

Every result contains exactly these six public keys: `status`, `item`, `delivery`, `total`, `message`, and `steps`. `status` is always `accepted` or `rejected`; unfulfillable orders and validation failures are `rejected`. `delivery` is always `pickup` or `delivery`. `item` is the fulfilled item when accepted and the requested item when rejected. Delivery cost is calculated internally and included in `total`, never placed in the public `delivery` field.

### Menu and inventory

| Item | Unit price (NIS) | Stock (units) | Availability |
|---|---:|---:|---|
| Challah | 18 | 10 | Available |
| Sourdough | 25 | 4 | Available |
| Rugelach | 30 | 3 | Available |
| Babka | 45 | 0 | Out of stock |
| Croissants | — | 0 | Discontinued / unavailable |

Menu matching is case-insensitive. Croissants are a recognized menu item but cannot be fulfilled. Unknown names fail validation.

### Validation and inventory

1. `validate_order` normalizes the requested item and checks that it is a recognized menu item and that quantity is a positive integer. Invalid requests go to `reject_order` and return `rejected`.
2. `check_stock` checks that the full requested quantity is available for `current_item`. Partial fulfillment is never allowed.
3. If the item is available in sufficient quantity, proceed to pricing.
4. If unavailable or short on stock, `handle_substitution` follows the configured chain `babka → rugelach → sourdough → challah`. A candidate is tried only if it exists in the chain and the replacement-attempt limit has not been reached. Each candidate increments `attempt_count` and returns to `check_stock`.
5. At most two replacement candidates are tried. If neither has enough stock, or no next candidate exists, `reject_order` returns `rejected`. The order is never partially fulfilled. Substitution routing uses internal route/availability fields so the public `status` remains within its two allowed values throughout processing. Under this bound, an order for Babka can try Rugelach and then Sourdough; it cannot proceed to Challah after those two attempts.

### Delivery and pricing

- Jerusalem: public `delivery` is `delivery`; the internal delivery fee is 20 NIS.
- Empty or whitespace-only city: public `delivery` is `pickup`, with no delivery fee.
- Any non-empty city other than Jerusalem (case-insensitive): automatically convert to pickup, set public `delivery` to `pickup`, set the internal fee to zero, and explain the conversion in `message`.
- Compute baked-goods subtotal from the fulfilled item’s unit price times the full quantity.
- Apply Avi’s 15% discount to the baked-goods subtotal only when the subtotal is strictly greater than 150 NIS and the order day is not Friday.
- Never discount delivery. Total is discounted baked-goods subtotal plus the full delivery fee, rounded to two decimal places.
- Friday is recognized from a weekday name or an ISO `YYYY-MM-DD` date.

### Graph and observability

`StateGraph` has the required nodes: `validate_order`, `check_stock`, `handle_substitution`, `calculate_pricing`, `finalize_response`, and `reject_order`. Conditional edges route valid input to inventory checks; available inventory to pricing; stockouts through bounded substitution; and invalid or unfulfillable requests to rejection. `steps` is appended to on entry to every node, including repeated `check_stock` visits, so the execution diary reflects the actual path. The graph is compiled once and can be visualized with `display(Image(graph.get_graph().draw_mermaid_png()))`.

### Operational notes

Menu prices, stock, substitution rules, and the replacement-attempt limit are constants that should be moved to validated configuration if the system becomes operational. This specification’s inventory is an in-memory snapshot; production order acceptance must make stock reservation atomic with order persistence to prevent overselling. The flow is deterministic and side-effect-free as written; it does not contact a payment, delivery, or ordering service.

## Revision history

### Contract and assertion alignment

- Restricted every result status to exactly `accepted` or `rejected`; orders that cannot be fulfilled and failed substitutions now route to `reject_order` with status `rejected`.
- Changed the public delivery field to exactly `pickup` or `delivery`; delivery fees remain internal pricing values and are included in the total without being exposed through that field.
- Ensured every output contains all six required keys: status, item, delivery, total, message, and steps.
- Kept substitution matching and customer messages normalized to lowercase, enforcing at most two replacement attempts before rejection.
- Replaced brittle assertions with checks for the allowed status and delivery values, numeric totals, complete result keys, and case-insensitive message context.
