# contracts

[English](README.md) · [Türkçe](README.tr.md)

Multi-ERP parçaları arasındaki her sınırın tek doğru kaynağı.

| Yol | İçerik |
|---|---|
| `openapi/common.yaml` | Ortak şemalar: sayfalama, para, adres, Problem Details |
| `openapi/customers.yaml` | customer-service REST API'si |
| `openapi/products.yaml` | product-service REST API'si |
| `openapi/invoices.yaml` | invoice-service REST API'si |
| `openapi/reports.yaml` | report-service REST API'si |
| `openapi/documents.yaml` | document-service: çizim, şablonlar, e-posta |
| `events/events.schema.json` | Domain olay zarfı ve içerikleri |

## Kurallar

- Para ve miktarlar metin olarak gönderilir (`"750.00"`) ve `Decimal128` olarak saklanır. Asla float kullanılmaz.
- Hatalar `application/problem+json` (RFC 7807) biçimindedir.
- Liste uçları `{ items, meta }` döner; `page` (1'den başlar) ve `size` (en çok 100) alır.
- Her servis `GET /health` (canlılık) ve `GET /ready` (hazır olma) uçlarını sunar.
- Her `/api/*` isteği çalışma alanını `X-Tenant-Id` başlığında taşır; bunu edge, alan çerezinden doldurur (bkz. tenant-service). Başlıksız istekler `401` alır; servisler tüm veriyi buna göre ayırır ve servisten servise çağrılarda iletir.

## Domain olayları (RabbitMQ)

Olaylar kalıcı topic exchange `erp.events`'e, publisher confirm ile kalıcı JSON mesajlar olarak yayınlanır. Routing key olayın `type` değeridir; AMQP `message_id`, `type` ve `app_id` özellikleri zarfın `id`, `type` ve `source` alanlarını, `tenant-id` başlığı da `tenantId`'yi tekrarlar. Tüketiciler bir olayı yalnızca kendi `tenantId`'si içinde uygular (`tenantId`'si olmayan olaylar `main` alanına aittir).

| Routing key | Yayınlayan | Tüketenler |
|---|---|---|
| `customer.created` / `customer.updated` / `customer.deleted` | customer-service | report-service |
| `product.created` / `product.updated` / `product.deleted` | product-service | report-service |
| `product.stock-changed` | product-service | report-service |
| `invoice.approved` | invoice-service | product-service, report-service |
| `invoice.cancelled` | invoice-service | product-service, report-service |
| `invoice.sent` | invoice-service | — (`{ id, number, customerId, to, at }`) |

Her tüketici, açılışta bağlarıyla birlikte tanımladığı kalıcı bir quorum kuyruğuna sahiptir. Mesajlar işlendikten sonra onaylanır (ack). Başarısız bir mesaj bir süre sonra yeniden teslim edilir; 10 teslimden sonra (`x-delivery-limit`) `erp.dead` exchange'i üzerinden `<kuyruk>.dead` kuyruğuna düşer ve kuyruğu tıkamak yerine incelenmeyi bekler. Teslimat "en az bir kez" olduğu için işleyiciler idempotent olmalıdır.

| Kuyruk | Bağlar | İdempotentlik |
|---|---|---|
| `product-service.invoices` | `invoice.approved`, `invoice.cancelled` | her ürün son uygulanan 200 olay kimliğini tutar; stok `$inc` ve kimlik `$push` tek atomik güncellemede yapılır |
| `report-service` | `customer.#`, `product.#`, `invoice.#` | projeksiyonlar varlık kimliğine göre upsert'tür; yeniden teslimler aynı duruma varır |

Bir kuyruk yalnızca tanımlandıktan sonra yayınlanan olayları alır. report-service ayrıca açılışta müşteri ve ürün projeksiyonlarını servislerin API'lerinden yeniden kurar; böylece sıfırdan açılan bir rapor veritabanı eski olaylara bağlı kalmaz.

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

## Tarayıcı olayları (micro frontend'ler)

Parçalar birbirini hiçbir zaman import etmez; `window` üzerindeki `CustomEvent`'lerle konuşurlar.

| Olay | İçerik | Yayınlayan | Karşılayan |
|---|---|---|---|
| `erp:toast` | `{ type: "success" \| "error" \| "info", message: string }` | herhangi biri | mf-shell |
| `erp:navigate` | `{ path: string }` | herhangi biri | storefront |
| `erp:customer-changed` | `{ id: string }` | mf-customers | mf-invoices, mf-dashboard |
| `erp:product-changed` | `{ id: string }` | mf-products | mf-invoices, mf-dashboard |
| `erp:invoice-changed` | `{ id: string, status: string }` | mf-invoices | mf-dashboard |

```js
window.dispatchEvent(new CustomEvent("erp:toast", { detail: { type: "success", message: "Kaydedildi" } }));
```
