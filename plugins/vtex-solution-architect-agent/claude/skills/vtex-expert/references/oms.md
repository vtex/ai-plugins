# VTEX OMS — Platform Reference

## Order Lifecycle Overview

An order in VTEX moves through a fixed workflow. Each status represents a
state in the order's lifecycle, and transitions are triggered by system
events, API calls, or merchant actions in the Admin.

```
Order placed
  └─► payment-pending
        └─► payment-approved
              └─► ready-for-handling
                    └─► handling
                          └─► invoiced ──────────────────────► delivered (manual)
                                                              └─► canceled (before invoiced)
```

The `canceled` terminal state is reachable from most pre-invoiced states.
Once an order is invoiced, cancellation requires a return / reverse logistics
flow.

---

## Order Statuses (Workflow States)

| Status                  | Description                                                                                           |
| ----------------------- | ----------------------------------------------------------------------------------------------------- |
| `payment-pending`       | Order placed; awaiting payment confirmation                                                           |
| `payment-approved`      | Payment gateway confirmed; order enters fulfillment queue                                             |
| `ready-for-handling`    | Order is ready for the warehouse to start preparation (picking / packing)                             |
| `handling`              | Warehouse has started handling the order                                                              |
| `invoiced`              | Invoice (NF-e or equivalent) has been issued and sent to VTEX via the Invoice API                     |
| `delivered`             | Order marked as delivered (manual or via carrier webhook)                                             |
| `canceled`              | Order canceled; if payment was captured, a refund must be initiated separately                        |
| `window-to-cancel`      | A configured cancellation window is active (used to allow customer self-service cancellation)         |
| `on-order-completed`    | Terminal state for completed orders (post-delivery confirmation)                                      |

### Status transitions you can trigger via API

| Transition                    | API action                                         |
| ----------------------------- | -------------------------------------------------- |
| → `handling`                  | POST start-handling on the order                   |
| → `invoiced`                  | POST invoice to the order                          |
| → `canceled`                  | POST cancellation request                          |
| Update tracking               | POST tracking update to the invoice                |

---

## Invoicing an Order

Invoicing is the most critical OMS action — it advances the order to
`invoiced` and is required for the order to be considered fulfilled
from a fiscal standpoint.

### Invoice payload fields (key)

| Field             | Description                                                                |
| ----------------- | -------------------------------------------------------------------------- |
| `type`            | `Output` for a sale invoice, `Input` for a return invoice                  |
| `invoiceNumber`   | The fiscal document number (NF-e number in Brazil)                         |
| `invoiceValue`    | Total invoice value in cents                                               |
| `issuanceDate`    | Date the fiscal document was issued (ISO 8601)                             |
| `invoiceKey`      | Access key for the NF-e (Brazil) or equivalent fiscal ID                   |
| `trackingNumber`  | Optional carrier tracking number                                           |
| `trackingUrl`     | Optional URL for customer tracking page                                    |
| `courier`         | Carrier name                                                               |
| `items`           | Array of line items with quantity and price to be invoiced                 |

**Known behavior:** An order can be invoiced in multiple installments
(partial invoices). This is common for split fulfillment (items in the
same order shipped from different warehouses at different times). Each
partial invoice must cover a subset of items; the order reaches `invoiced`
when all items are covered.

---

## Tracking Updates

After invoicing, you can add or update tracking information without changing
the order status.

- POST a tracking update to the order's invoice endpoint.
- Include `trackingNumber`, `trackingUrl`, and `courier`.
- The tracking update triggers a customer notification email if transactional
  emails are configured.

For carriers that push delivery confirmation events, build a webhook receiver
that calls the VTEX tracking API when the carrier marks the package as
delivered.

---

## Order Cancellation

### Before invoiced

- POST a cancellation request to the order.
- If payment was captured (not just authorized), a refund must be issued
  via the payment provider — VTEX does not auto-refund; you must trigger
  the refund via the Payment Provider or the Payments API.
- Stock is released back to the warehouse.

### After invoiced

An invoiced order cannot be canceled via the standard flow. To handle
post-invoice returns:

1. Create a return request (via the Returns app or the Returns API).
2. Issue a return invoice (`type: Input`) for the returned items.
3. Process the refund via the payment provider.

**Known behavior:** The cancellation window (`window-to-cancel`) is a
configurable holding period (typically 30 minutes) during which the
customer can self-cancel the order from the My Orders page. Orders in
this window have not yet entered the fulfillment queue.

---

## OMS Hooks

VTEX OMS emits events that external systems can subscribe to. There are
two mechanisms: **Feed v3** (pull) and **Hook API** (push).

### Feed v3 — Pull-based event consumption

Feed v3 is a queue of order events. Your service polls the queue, processes
events, and commits (acknowledges) them.

**How it works:**

1. Configure a filter on the feed (which order statuses to receive events for).
2. GET events from the queue endpoint (returns up to 10 events per call).
3. Process each event.
4. POST a commit with the event handles to acknowledge.
5. Unacknowledged events reappear after the visibility timeout.

**Key behaviors:**

- Events are ordered within a single order but not across orders.
- An event for the same status change can appear multiple times if not
  committed (at-least-once delivery).
- Design your consumer to be idempotent.
- Each VTEX account can have up to one Feed v3 configuration per app key.

### Hook API — Push-based event notification

The Hook API sends an HTTP POST to your endpoint whenever an order reaches
a configured status. This is simpler to consume than Feed v3 for low-volume
use cases.

**Configuration:**

```json
{
  "filter": {
    "status": ["payment-approved", "invoiced", "canceled"]
  },
  "hook": {
    "headers": {
      "Authorization": "Bearer <your-token>"
    },
    "url": "https://your-endpoint.example.com/vtex-orders"
  }
}
```

**Key behaviors:**

- VTEX retries failed webhook deliveries (non-2xx response) up to 5 times
  with exponential backoff.
- Hook payload includes the `orderId`, `status`, `lastChange`, and `currentState`.
  Your endpoint must call the Orders API to get full order details.
- One hook configuration per account. If you need multiple consumers, Fan-out
  from a single endpoint.

### Feed v3 vs Hook API

| Factor                        | Feed v3                           | Hook API                               |
| ----------------------------- | --------------------------------- | -------------------------------------- |
| Delivery model                | Pull (poll)                       | Push (webhook)                         |
| Acknowledgement               | Required (commit)                 | Not required (2xx response = consumed) |
| Retry                         | Visibility timeout re-delivery    | VTEX retries on non-2xx                |
| Ordering guarantee            | Within an order                   | None                                   |
| Multiple consumers            | One queue per app key             | Fan-out from single endpoint           |
| Best for                      | High-volume, reliable processing  | Simple integrations, low volume        |

---

## Order Notification (invoiceNotification)

The `invoiceNotification` endpoint notifies VTEX that a fiscal document
has been issued outside VTEX (e.g., by an ERP). This is distinct from
posting an invoice directly — it updates VTEX's record with the invoice
data generated externally.

Use this pattern when:
- An ERP or fiscal middleware generates the NF-e and sends the key to VTEX.
- The fulfillment system (WMS) triggers the notification after packing.

---

## Multi-Origin Orders (Split Fulfillment)

When an order contains items from multiple warehouses, VTEX creates a single
order with multiple **shipments**. Each shipment has its own status and
invoice. The order's overall status reflects the aggregate of all shipments.

- Each shipment must be invoiced independently.
- Tracking is managed per shipment.
- Cancellation of a subset of items creates a partial cancellation and
  requires a partial invoice adjustment.

---

## OMS API Reference

Key endpoints (verify contracts with `vtex-developer` MCP):

| Operation                       | Method | Path                                                                |
| ------------------------------- | ------ | ------------------------------------------------------------------- |
| Get order                       | GET    | `/api/oms/pvt/orders/{orderId}`                                     |
| List orders                     | GET    | `/api/oms/pvt/orders`                                               |
| Start handling                  | POST   | `/api/oms/pvt/orders/{orderId}/start-handling`                      |
| Send invoice                    | POST   | `/api/oms/pvt/orders/{orderId}/invoice`                             |
| Update tracking                 | PUT    | `/api/oms/pvt/orders/{orderId}/invoice/{invoiceNumber}`             |
| Cancel order                    | POST   | `/api/oms/pvt/orders/{orderId}/cancel`                              |
| Get Feed v3 events              | GET    | `/api/orders/feed`                                                  |
| Commit Feed v3 events           | POST   | `/api/orders/feed`                                                  |
| Configure Hook                  | POST   | `/api/orders/hook/config`                                           |
| Get Hook configuration          | GET    | `/api/orders/hook/config`                                           |

---

## Common OMS Mistakes

| Mistake                                                        | Consequence                                                              |
| -------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Invoicing with `invoiceValue` ≠ order total                    | Order may be flagged for manual review; fiscal discrepancy               |
| Not committing Feed v3 events                                  | Events reappear; your consumer processes them multiple times             |
| Assuming Hook delivery is exactly-once                         | Retries can cause duplicate processing; implement idempotency keys       |
| Canceling after invoiced via the standard cancel endpoint      | Returns a 400 error; must use the returns flow instead                   |
| Not issuing a return invoice after a return request            | Order stays in invoiced state; confusing for the customer and finance    |
| Polling the Orders API instead of using Feed v3 or Hook        | Rate limits; missed events during polling gaps                           |
