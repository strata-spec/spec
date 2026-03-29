# Database: E-commerce Platform

## What this database is for

Transactional e-commerce database for a direct-to-consumer retail operation. Tracks customer
accounts, product catalogue, inventory levels, orders, and fulfilment events. Primary consumers
are the analytics team (revenue dashboards, cohort analysis), operations (fulfilment monitoring),
and the customer support team (order lookup). The database is the system of record for all
post-checkout activity.

The database is write-heavy on `orders` and `order_items` and read-heavy on `products`. Historical
orders are never deleted. The `customers` table uses a soft-delete pattern (`deleted_at`) for
GDPR right-to-erasure requests — always filter `deleted_at IS NULL` for active customers.

## Key business concepts

- **Order**: A single checkout event by a customer. One row per order. An order has a lifecycle:
  `pending → confirmed → shipped → delivered` (or `cancelled` / `refunded` at any stage). Use
  `orders.status` to filter by lifecycle stage.
- **Order Item**: A single SKU within an order, with a quantity and the unit price at the time
  of purchase. Line-level revenue and discount information is stored here, not on `orders`.
- **Customer**: A registered account holder. Customers may place many orders over time. The
  `customer_since` column records when the account was first created, not the date of first
  purchase.
- **Product**: A sellable item with a name, category, and base price. Products are not deleted;
  discontinued items have `status = 'discontinued'`. The `products` table is the catalogue — it
  does not track inventory.
- **Inventory**: Stock-on-hand for a given product at a given warehouse. One row per
  product-warehouse combination. Inventory is updated in real time by the warehouse management
  system. Use `inventory.quantity_on_hand` for current stock; do not derive stock from
  `order_items`.
- **Revenue**: Defined as the sum of `order_items.unit_price * order_items.quantity` for orders
  with `status NOT IN ('cancelled', 'refunded')`. Revenue is always reported in USD.
- **Gross Margin**: Revenue minus cost of goods sold. COGS is stored in
  `order_items.unit_cost` (populated from the supplier invoice at the time of shipment).
  COGS may be NULL for items whose cost has not yet been confirmed.

## Known gotchas

- `orders.total_amount` is a denormalised convenience field. For accurate revenue figures,
  always aggregate from `order_items.unit_price * order_items.quantity`. The two will disagree
  when promotions are applied at checkout (a known data quality issue being addressed in Q3).
- `products.price` is the current catalogue price and changes over time. The price at which
  an item was actually sold is in `order_items.unit_price`, which is captured at order creation
  and never changes.
- `customers.deleted_at IS NULL` must always be appended for GDPR compliance. Deleted customers
  retain their `orders` rows but have their `email`, `name`, and `phone` columns nulled.
- `order_items.unit_cost` is NULL for approximately 4% of items where the supplier invoice
  has not yet been matched. Exclude NULL costs from any margin analysis and document the
  exclusion in the query.
- The `status` column on `orders` uses lowercase string literals. All filter values must be
  lowercase (e.g., `status = 'shipped'`, not `status = 'Shipped'`).
- `inventory.quantity_on_hand` can be negative. This is a legitimate warehouse condition
  (backorders accepted before stock arrives). A negative value does not indicate corrupt data.
- `orders.customer_id` is NULL for guest checkouts. Guest orders account for roughly 22% of
  all orders. Do not exclude NULL customer_id rows from revenue queries unless the question
  specifically asks about registered customers.

## Sensitive columns

- `customers.email` — personal data, do not expose in aggregated outputs
- `customers.phone` — personal data
- `customers.name` — personal data, use `customer_id` for grouping

## Do not infer metrics from

- `ecommerce.schema_migrations` — database migration metadata
- `ecommerce.audit_events` — internal system audit trail, not a business table
