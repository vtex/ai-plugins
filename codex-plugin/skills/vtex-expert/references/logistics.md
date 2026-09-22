# VTEX Logistics — Platform Reference

## Core Concepts

VTEX logistics operates on four fundamental entities that form the shipping
calculation chain: **Warehouses → Docks → Carriers → SLAs**.

```
Warehouse (stock source)
  └─► Dock (dispatch point)
        └─► Carrier (transport)
              └─► SLA (delivery promise shown to customer)
```

Every shipping option shown at checkout is an SLA composed by traversing
this chain. If any link is broken (e.g., no carrier associated with a dock,
no SLA covering the destination ZIP), no shipping option appears.

---

## Warehouses

A **Warehouse** is a stock location. It holds inventory and defines which
docks it feeds.

Key configuration fields:

| Field                  | Description                                                                 |
| ---------------------- | --------------------------------------------------------------------------- |
| Name                   | Internal identifier                                                         |
| Docks                  | One or more docks this warehouse ships from (with a transit time per dock)  |
| Business days          | Which days the warehouse operates (affects SLA calculation)                 |
| Working hours          | Operating hours for same-day order cut-off                                  |
| Overhead time          | Extra handling time added to every shipment from this warehouse             |

A warehouse can be associated with multiple docks. The **transit time to
dock** (warehouse → dock leg) is added to the carrier transit time to
compute the final delivery time.

---

## Docks

A **Dock** is a dispatch point — the place shipments leave from. Docks
connect warehouses to carriers.

Key configuration fields:

| Field              | Description                                                                     |
| ------------------ | ------------------------------------------------------------------------------- |
| Name               | Internal identifier                                                             |
| Carriers           | Carriers that can pick up from this dock                                        |
| ZIP code / address | Used for distance-based carrier rate calculation                                |
| Processing time    | Extra time added at the dock before the carrier picks up                        |
| Business days      | Dock operating calendar                                                         |

A dock can serve multiple carriers and multiple warehouses can feed the
same dock.

---

## Carriers

A **Carrier** represents a transport service — either a native VTEX carrier
(White Label), an external carrier integrated via API, or a carrier configured
with a freight table.

### Carrier types

| Type                        | Description                                                                |
| --------------------------- | -------------------------------------------------------------------------- |
| White Label Carrier         | Managed entirely in VTEX; freight table defines rates by ZIP range         |
| External Carrier (via API)  | Carrier system is queried at checkout for live rates                       |
| Pickup Point Carrier        | Routes orders to a physical pickup point instead of a delivery address     |

### White Label Carrier — freight table

The freight table maps ZIP code ranges to price and transit time. Each row
defines:

- ZIP start / ZIP end (or a regex pattern for flexibility)
- Price (flat rate for the range)
- Transit time (business days)
- Weight limit and cubic weight factor
- Minimum / maximum order value (optional filters)

Rows are evaluated top-to-bottom; the first matching row wins. If no row
matches the destination ZIP, the carrier is not offered at checkout.

**Known quirk:** ZIP patterns use a VTEX-specific regex-like syntax, not
standard regex. Test patterns in the Admin freight table simulator before
publishing.

---

## SLAs (Service Level Agreements)

An SLA is the delivery promise shown to the customer. VTEX computes SLA
options by combining all valid warehouse → dock → carrier paths for the
customer's ZIP code and cart.

The delivery time shown to the customer is:

```
SLA delivery time = warehouse overhead
                  + warehouse → dock transit time
                  + dock processing time
                  + carrier transit time
                  (business days only, filtered by each entity's calendar)
```

SLA configuration is done at the carrier level. Each carrier defines one
or more SLAs with:

| Field            | Description                                                        |
| ---------------- | ------------------------------------------------------------------ |
| Name             | Label shown to customer (e.g., "Standard", "Express")             |
| Delivery type    | Delivery or Pickup                                                 |
| Delivery window  | Fixed time range (for scheduled delivery)                          |
| Shipping estimate | Business-day estimate shown at checkout (if not using scheduled)  |

---

## Shipping Strategies

A **Shipping Strategy** (called **Shipping Policy** in the Admin) defines
which carriers are eligible for an order, under which conditions. It is
associated with a dock and filters which carrier SLAs are offered.

You can have multiple shipping strategies active simultaneously. VTEX
evaluates all valid paths and presents the resulting SLAs to the customer.

### Common shipping strategy patterns

| Pattern                            | How to configure                                                                                   |
| ---------------------------------- | -------------------------------------------------------------------------------------------------- |
| Standard ground shipping           | White Label Carrier with freight table; assign to dock; no delivery window                         |
| Express / same-day                 | Separate carrier with shorter transit time; business hours filter on warehouse                     |
| Scheduled delivery                 | Enable delivery windows on the carrier SLA (see Scheduled Delivery section)                        |
| Ship-from-store                    | Create a warehouse per store; assign each to a dock with a local carrier                           |
| Pickup in store                    | Create a Pickup Point; assign a carrier of type Pickup Point; link to dock                         |
| Multi-carrier (cheapest or fastest)| Create multiple carriers on the same dock; customer chooses at checkout                            |

---

## Scheduled Delivery

Scheduled delivery lets customers choose a specific delivery date and time
window at checkout. It is configured at the **carrier SLA** level.

### How to enable scheduled delivery

1. Go to **Admin → Shipping → Carriers → [your carrier] → Edit**.
2. Open the SLA configuration for the relevant delivery type.
3. Enable **Delivery windows**.
4. Define the available time windows (e.g., 08:00–12:00, 14:00–18:00).
5. Set the **lead time** — the minimum notice required before a window can
   be selected (e.g., 24h). Orders placed after the cut-off for a window
   cannot select that window.
6. Optionally enable **Delivery Capacity** to cap the number of orders per
   window (see Delivery Capacity section).

### Delivery window fields

| Field                | Description                                                               |
| -------------------- | ------------------------------------------------------------------------- |
| Start time           | Window open time (e.g., 08:00)                                            |
| End time             | Window close time (e.g., 12:00)                                           |
| Days of week         | Which days this window is available                                       |
| Lead time            | Minimum notice before window opens (in hours or business days)            |
| Price                | Surcharge for this window (can be 0)                                      |

**Known behavior:** Delivery windows are defined in local time based on the
account's timezone. Verify the account timezone in **Admin → Account Management**
if windows appear offset for customers in different time zones.

---

## Delivery Capacity

Delivery Capacity limits how many deliveries can be assigned to a given
scheduled delivery window. This prevents over-booking for logistics-constrained
time slots.

### How to configure Delivery Capacity

1. Navigate to **Admin → Shipping → Delivery Capacity**.
2. Select the carrier and SLA.
3. For each time window, set the **maximum number of orders** (capacity).
4. Optionally block specific dates (holidays, maintenance periods).

### How capacity is consumed

- When a customer selects a delivery window at checkout, the capacity counter
  decrements in real time.
- Once a window reaches its limit, it is no longer offered to new customers.
- Capacity is released if the order is cancelled before it advances past the
  "Ready for Handling" status.

### Capacity management considerations

| Topic                     | Detail                                                                             |
| ------------------------- | ---------------------------------------------------------------------------------- |
| Reservation timing        | Capacity is reserved at order placement, not at checkout page load                 |
| Cart abandonment          | A window selected but not purchased does not consume capacity                      |
| Capacity reset            | Capacity resets per calendar period; configure weekly or daily as needed           |
| API management            | Capacity can be read and updated via the Logistics API for dynamic management      |

**Known limitation:** There is no native alerting when a window reaches
capacity. Build an external monitor via the Logistics API if real-time
capacity visibility is required.

---

## Pickup Points

A **Pickup Point** is a physical location (store, locker, partner location)
where customers collect their orders.

### Configuration

1. Go to **Admin → Shipping → Pickup Points → Add**.
2. Set name, address, and geographic coordinates.
3. Define business hours for each day of the week.
4. Associate the pickup point with a carrier of type **Pickup Point**.
5. Assign that carrier to a dock.

### Pickup flow

- Customer selects "Pick up" at checkout and chooses a pickup point.
- The order is routed to the pickup point's associated warehouse → dock.
- Fulfillment team prepares the order and marks it ready for pickup.
- Customer receives a ready-for-pickup notification (if configured).
- Status advances to "Delivered" when the customer collects.

**Known behavior:** The distance radius for filtering pickup points shown to
the customer is configured at the carrier level, not per pickup point.

---

## Multi-Origin Shipping (Split Shipment)

When items in a cart are sourced from different warehouses, VTEX can split
the order into multiple shipments.

- Split shipment is enabled by default if items have different origins.
- Each shipment follows its own warehouse → dock → carrier → SLA chain.
- The customer may see multiple delivery options and dates, one per shipment.
- Use the **Shipping Strategy** to control whether split shipment is allowed
  or whether VTEX should always try to consolidate from a single origin.

---

## Logistics API Reference

Key endpoints for programmatic logistics management (verify with
`vtex-developer` MCP for current contracts):

| Operation                           | Method | Path                                                                |
| ----------------------------------- | ------ | ------------------------------------------------------------------- |
| List warehouses                     | GET    | `/api/logistics/pvt/configuration/warehouses`                       |
| Get warehouse                       | GET    | `/api/logistics/pvt/configuration/warehouses/{warehouseId}`         |
| List docks                          | GET    | `/api/logistics/pvt/configuration/docks`                            |
| List carriers                       | GET    | `/api/logistics/pvt/configuration/carriers`                         |
| Get SLA by carrier                  | GET    | `/api/logistics/pvt/configuration/carriers/{carrierId}`             |
| Simulate shipping                   | POST   | `/api/logistics/pvt/shipping/simulate`                              |
| Get delivery capacity               | GET    | `/api/logistics/pvt/configuration/capacity`                         |
| Update delivery capacity            | POST   | `/api/logistics/pvt/configuration/capacity`                         |
| List pickup points                  | GET    | `/api/logistics/pvt/configuration/pickuppoints`                     |
