# VTEX Promotions & Pricing — Platform Reference

## Core Entities

| Entity            | Description                                                                        |
| ----------------- | ---------------------------------------------------------------------------------- |
| Price table       | A named set of prices for SKUs; applied per customer segment or trade policy       |
| Customer cluster  | A named group of customers; used to trigger promotions, price tables, and content  |
| Promotion         | A rule that applies a discount or benefit under defined conditions                 |
| Coupon            | An optional code that activates or gates a promotion                               |

---

## Price Tables

A **Price Table** is a complete alternative price list for all SKUs. Each
SKU can have a different price in each price table.

### How price tables are applied

1. A customer is identified (logged in or identified via cookie/cluster).
2. VTEX checks if the customer belongs to a cluster or trade policy that
   has an associated price table.
3. If matched, the price table prices override the base price table at
   checkout.

### Price table configuration

- Price tables are created in **Admin → Pricing → Price Tables**.
- Prices are set per SKU either via the Admin, the Pricing API, or a
  spreadsheet import.
- A price table can define either an absolute price or a percentage markup
  / markdown on the base price.

### Key behaviors

| Behavior                                   | Detail                                                                     |
| ------------------------------------------ | -------------------------------------------------------------------------- |
| Base price table                           | The default price table — every account has one                           |
| Multiple price tables                      | A customer can match multiple tables; the most specific match wins         |
| Price table + promotion                    | A promotion discount is calculated on top of the price table price        |
| Trade policy binding                       | A price table can be bound directly to a trade policy (B2B or marketplace)|
| Price table via cluster                    | A cluster-based price table requires the customer to be identified         |

**Known behavior:** If a customer matches a cluster-bound price table but
is anonymous, the base price table applies. Price tables based on clusters
require authentication or a cluster assignment mechanism (e.g., cookie set
by a VTEX IO app or via the Identity API).

---

## Customer Clusters

A **Customer Cluster** is a named tag that can be assigned to customers.
Promotions and price tables reference clusters to target specific customer
groups (e.g., VIP customers, wholesale buyers, loyalty tier).

### How clusters are assigned

- Manually via the Admin (CRM section) on a per-customer basis.
- Programmatically via the **Customer API** or **MasterData API** by
  writing to the `CL` entity's `clusterSegments` field.
- Automatically via a VTEX IO app that evaluates rules (loyalty points,
  purchase history, etc.) and writes the cluster tag.

### Cluster behaviors

- A customer can belong to multiple clusters simultaneously.
- Cluster membership is evaluated at session start and cached for the
  session duration.
- Promotions targeting a cluster only apply when the customer is identified
  (logged in). Anonymous customers do not have clusters.

---

## Promotion Types

### 1. Regular Promotion (Percentage or Nominal Discount)

The most common type. Applies a percentage or fixed amount discount to
eligible items.

**Conditions available:**

- Minimum order value
- Minimum quantity of items
- Specific SKUs, categories, brands, or collections
- Customer cluster
- Payment method (e.g., 10% off with Visa)
- First purchase only
- UTM source / campaign
- Coupon code

**Effect options:**

- Percentage discount on item price
- Nominal discount on item price
- Maximum price per item

### 2. Combo Promotion (Buy X Get Y)

Triggers when the customer has items from two defined groups in the cart.

- **Group A** — the trigger items (customer must buy N items from group A)
- **Group B** — the discounted items (discount applied to group B items)

Example: Buy 2 from Group A (shirts), get 1 from Group B (belts) at 50% off.

**Known limitation:** The combo engine compares quantities across the two
groups. Complex multi-item combos (buy 3 of X and 2 of Y) can hit engine
evaluation limits — test edge cases in staging before publishing.

### 3. Buy Together Promotion

Triggers when specific SKUs appear together in the cart. Applies a discount
to the set.

Use for complementary product bundles (camera + memory card + bag at a
bundle price).

### 4. Gift Promotion

Adds a free item (gift) to the cart when conditions are met. The gift SKU
must exist in the catalog.

**Configuration:**

- Define the condition (minimum order value, specific SKU in cart, etc.)
- Define the gift SKU and quantity
- Optionally limit total gifts per order

**Known behavior:** If the gift SKU is out of stock, the promotion still
triggers but the gift item cannot be added. Configure a fallback or monitor
gift SKU stock separately.

### 5. Progressive Discount (Tiered Discount)

Applies increasing discounts as the quantity of eligible items grows.

Example: 5% off for 3 items, 10% off for 5 items, 15% off for 10+ items.

Useful for wholesale or volume incentives.

### 6. Campaign Promotion

Time-limited promotions tied to a campaign entity. Campaigns group related
promotions and can be activated / deactivated together.

Use campaigns to manage Black Friday or seasonal event promotions as a unit
without editing each promotion individually.

---

## Promotion Stacking

By default, VTEX allows multiple promotions to apply to the same cart.
Understanding stacking order is critical.

### Stacking rules

| Rule                              | Behavior                                                                             |
| --------------------------------- | ------------------------------------------------------------------------------------ |
| Promotions stack by default       | All eligible promotions apply unless `canAccumulateWithOtherPromotions` is false     |
| Exclusive promotions              | If `isExclusive = true`, only this promotion applies (others are suppressed)          |
| Promotion priority                | When multiple promotions compete for the same item, priority field determines order   |
| Percentage stacking               | Applied sequentially, not additively: 10% + 10% ≠ 20%; it's 10% then 10% on remainder |
| Coupon + promotion                | A promotion can require a coupon, or a coupon can activate an otherwise inactive promo|

### Common stacking mistake

Setting two promotions as exclusive on the same SKU without considering which
takes precedence. The promotion with the lower priority number wins exclusivity.
Use the promotion simulator in **Admin → Promotions → Simulator** to verify
stacking behavior before publishing.

---

## Coupons

A **Coupon** is an alphanumeric code entered by the customer at checkout.
It can:

- Activate a promotion that requires a coupon (the promotion is otherwise inactive)
- Be a condition on an otherwise unconditionally active promotion
- Be generated in bulk (for single-use or multi-use codes)

### Coupon configuration

| Field              | Description                                                              |
| ------------------ | ------------------------------------------------------------------------ |
| Code               | The alphanumeric string the customer enters                              |
| Max use count      | Total times the coupon can be used (blank = unlimited)                   |
| Max use per client | How many times one customer can use this coupon                          |
| Expiry date        | When the coupon stops working                                            |

**Known behavior:** Coupon codes are case-insensitive in VTEX checkout.
"SUMMER10" and "summer10" are treated as the same code.

---

## Promotions & Bindings

Promotions are account-wide by default — they apply across all storefronts
bound to the account. There is no native per-binding promotion scope.

To differentiate promotions by storefront:
- Use UTM parameters to gate promotions (each storefront sets its own UTMs)
- Use customer clusters (assign clusters per storefront context if customers
  are identified)
- Use trade policies if each storefront is a separate trade policy

---

## Pricing API Reference

Key endpoints (verify contracts with `vtex-developer` MCP):

| Operation                         | Method | Path                                                                    |
| --------------------------------- | ------ | ----------------------------------------------------------------------- |
| Get price for SKU                 | GET    | `/api/pricing/prices/{skuId}`                                           |
| Create / update price             | PUT    | `/api/pricing/prices/{skuId}`                                           |
| Get price table                   | GET    | `/api/pricing/tables/{priceTableId}`                                    |
| List price tables                 | GET    | `/api/pricing/tables`                                                   |
| Get price in table for SKU        | GET    | `/api/pricing/prices/{skuId}/computed/{priceTableId}`                   |

## Promotions API Reference

| Operation                         | Method | Path                                                                    |
| --------------------------------- | ------ | ----------------------------------------------------------------------- |
| List promotions                   | GET    | `/api/rnb/pvt/benefits/calculatorconfiguration`                         |
| Get promotion by ID               | GET    | `/api/rnb/pvt/calculatorconfiguration/{idCalculatorConfiguration}`      |
| Create promotion                  | POST   | `/api/rnb/pvt/calculatorconfiguration`                                  |
| Update promotion                  | POST   | `/api/rnb/pvt/calculatorconfiguration/{idCalculatorConfiguration}`      |
| List coupons                      | GET    | `/api/rnb/pvt/coupon`                                                   |
| Create coupon                     | POST   | `/api/rnb/pvt/coupon`                                                   |
