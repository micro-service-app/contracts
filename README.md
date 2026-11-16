# Contracts

[English](README.md) · [Türkçe](README.tr.md)

Single source of truth for every boundary between Multi-ERP parts.

| Path | Content |
|---|---|
| `openapi/common.yaml` | Shared schemas: pagination, money, address, Problem Details |
| `openapi/customers.yaml` | customer-service REST API |
| `openapi/products.yaml` | product-service REST API |
| `openapi/invoices.yaml` | invoice-service REST API |
| `openapi/reports.yaml` | report-service REST API |
| `openapi/documents.yaml` | document-service: rendering, templates, e-mail |
| `events/events.schema.json` | Domain event envelope and payloads |

## Conventions

- Money and quantities are sent as strings (`"750.00"`) and stored as `Decimal128`. Never use floats.
- Errors use `application/problem+json` (RFC 7807).
- List endpoints return `{ items, meta }` and accept `page` (1-based) and `size` (max 100).
- Every service exposes `GET /health` (liveness) and `GET /ready` (readiness).
- Every `/api/*` request carries the workspace in `X-Tenant-Id`, set by the edge from the workspace cookie (see tenant-service). Requests without it get `401`; services scope all data by it and forward it on service-to-service calls.

## Domain events (RabbitMQ)

Events are published to the durable topic exchange `erp.events` as persistent JSON messages, with publisher confirms. The routing key equals the event `type`; the AMQP `message_id`, `type` and `app_id` properties repeat the envelope's `id`, `type` and `source`, and the `tenant-id` header repeats `tenantId`. Consumers apply an event only inside its `tenantId` (events without it belong to `main`).

| Routing key | Publisher | Consumers |
|---|---|---|
| `customer.created` / `customer.updated` / `customer.deleted` | customer-service | report-service |
| `product.created` / `product.updated` / `product.deleted` | product-service | report-service |
| `product.stock-changed` | product-service | report-service |
| `invoice.approved` | invoice-service | product-service, report-service |
| `invoice.cancelled` | invoice-service | product-service, report-service |
| `invoice.sent` | invoice-service | — (`{ id, number, customerId, to, at }`) |

Every consumer owns a durable quorum queue that it declares, with its bindings, on startup. Messages are acknowledged after they are handled. A failed message is redelivered after a delay; after 10 deliveries (`x-delivery-limit`) it is dead-lettered through the `erp.dead` exchange into `<queue>.dead`, where it waits for inspection instead of blocking the queue. Delivery is at least once, so handlers must be idempotent.

| Queue | Bindings | Idempotency |
|---|---|---|
| `product-service.invoices` | `invoice.approved`, `invoice.cancelled` | each product keeps the last 200 applied event ids; the stock `$inc` and the id `$push` happen in one atomic update |
| `report-service` | `customer.#`, `product.#`, `invoice.#` | projections are upserts keyed by entity id; redeliveries converge to the same state |

A queue only receives events published after it was declared. report-service also rebuilds its customer and product projections from the services' APIs on startup, so a fresh report database does not depend on old events.

```json
{
  "id": "6f1c1f3e-8a77-4e43-9f43-2a0b6a7d9c11",
  "type": "invoice.approved",
  "tenantId": "main",
  "occurredAt": "2026-11-14T10:00:00Z",
  "source": "invoice-service",
  "data": {
    "id": "6735a1...",
    "number": "ERP2026000000001",
    "customerId": "6735a0...",
    "customerName": "Örnek Gıda A.Ş.",
    "grandTotal": "1800.00",
    "at": "2026-11-14T10:00:00Z",
    "lines": [{ "productId": "6735a2...", "quantity": "2" }]
  }
}
```

## Browser events (micro frontends)

Fragments never import each other. They talk through `window` `CustomEvent`s.

| Event | Detail | Emitted by | Handled by |
|---|---|---|---|
| `erp:toast` | `{ type: "success" \| "error" \| "info", message: string }` | any | mf-shell |
| `erp:navigate` | `{ path: string }` | any | storefront |
| `erp:customer-changed` | `{ id: string }` | mf-customers | mf-invoices, mf-dashboard |
| `erp:product-changed` | `{ id: string }` | mf-products | mf-invoices, mf-dashboard |
| `erp:invoice-changed` | `{ id: string, status: string }` | mf-invoices | mf-dashboard |

```js
window.dispatchEvent(new CustomEvent("erp:toast", { detail: { type: "success", message: "Kaydedildi" } }));
```
