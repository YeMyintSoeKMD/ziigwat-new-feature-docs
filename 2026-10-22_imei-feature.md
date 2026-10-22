The **minimal database change set** for the finalized serialized inventory + IMEI + buyback design.

## 1. Existing tables — new columns to add

### `items`

| Column | Type | Nullable | Default | Purpose |
|---|---|---:|---|---|
| `inventory_tracking` | varchar | No | `quantity` | `quantity` or `serialized` |

Existing items should automatically remain:

```text
inventory_tracking = quantity
```

---

### `order_details`

| Column | Type | Nullable | Purpose |
|---|---|---:|---|
| `inventory_unit_id` | bigint FK | Yes | Exact physical device sold |
| `cost_price_snapshot` | decimal(15,4) | Yes | Historical acquisition cost at time of sale |

For normal products:

```text
inventory_unit_id = null
```

For serialized products:

```text
inventory_unit_id = required
quantity = 1
```

---

### `sale_return_details`

| Column | Type | Nullable | Purpose |
|---|---|---:|---|
| `inventory_unit_id` | bigint FK | Yes | Exact physical device returned |

---

### `purchase_return_details`

| Column | Type | Nullable | Purpose |
|---|---|---:|---|
| `inventory_unit_id` | bigint FK | Yes | Exact serialized unit returned to supplier |

---

### `stock_transfers`

| Column | Type | Nullable | Purpose |
|---|---|---:|---|
| `inventory_unit_id` | bigint FK | Yes | Exact physical unit being transferred |

Normal quantity transfers leave this `null`.

---

### `adjustment_details`

| Column | Type | Nullable | Purpose |
|---|---|---:|---|
| `inventory_unit_id` | bigint FK | Yes | Exact device being adjusted/lost/damaged |

---

# 2. New tables

## 1. `item_identifier_requirements`

Defines which identifiers a serialized item requires.

| Column | Type | Nullable | Purpose |
|---|---|---:|---|
| `id` | bigint PK | No | Primary key |
| `shop_id` | bigint FK | No | Shop |
| `item_id` | bigint FK | No | Item |
| `identifier_type` | varchar | No | `imei`, `serial`, `eid`, `device_no`, `custom` |
| `slot` | tinyint | Yes | IMEI 1, IMEI 2, etc. |
| `label` | varchar | No | Display label |
| `is_required` | boolean | No | Required when receiving |
| `is_unique` | boolean | No | Must be unique |
| `validation_rule` | varchar | Yes | `imei`, `text`, etc. |
| `sort_order` | int | No | Display order |
| `is_active` | boolean | No | Enabled/disabled |
| `created_at` | datetime | No | |
| `updated_at` | datetime | No | |

Recommended unique:

```text
item_id + identifier_type + slot
```

---

# 2. `inventory_units`

One row = one physical serialized device.

| Column | Type | Nullable | Purpose |
|---|---|---:|---|
| `id` | bigint PK | No | Physical unit ID |
| `uuid` | uuid | No | Public-safe identifier |
| `shop_id` | bigint FK | No | Shop |
| `item_id` | bigint FK | No | Item / variant |
| `location_id` | bigint FK | No | Current location |
| `original_purchase_detail_id` | bigint FK | Yes | Original first purchase |
| `status` | varchar | No | Current status |
| `condition` | varchar | Yes | `new`, `used`, `refurbished` |
| `condition_grade` | varchar | Yes | A/B/C etc. |
| `purchase_cost` | decimal(15,4) | Yes | **Current/latest acquisition cost** |
| `asking_price` | decimal(15,4) | Yes | Current device asking price |
| `minimum_sale_price` | decimal(15,4) | Yes | Optional minimum price |
| `warranty_until` | date | Yes | Current warranty |
| `received_at` | datetime | Yes | Latest stock-entry date |
| `last_sold_at` | datetime | Yes | Latest sale |
| `note` | text | Yes | Notes |
| `created_at` | datetime | No | |
| `updated_at` | datetime | No | |
| `deleted_at` | datetime | Yes | Soft delete if required |

Recommended statuses:

```text
in_stock
reserved
sold
quarantined
damaged
under_repair
lost
returned_to_supplier
disposed
```

---

# 3. `inventory_unit_identifiers`

Stores IMEI, Serial, EID, etc.

| Column | Type | Nullable | Purpose |
|---|---|---:|---|
| `id` | bigint PK | No | |
| `shop_id` | bigint FK | No | Shop |
| `inventory_unit_id` | bigint FK | No | Physical unit |
| `identifier_type` | varchar | No | `imei`, `serial`, etc. |
| `slot` | tinyint | Yes | IMEI1 / IMEI2 |
| `value` | varchar | No | Original display value |
| `normalized_value` | varchar | No | Search/uniqueness value |
| `is_primary` | boolean | No | Primary identifier |
| `created_at` | datetime | No | |
| `updated_at` | datetime | No | |

Recommended unique:

```text
inventory_unit_id + identifier_type + slot
```

For unique identifier types such as IMEI, enforce:

```text
shop_id + identifier_type + normalized_value
```

---

# 4. `inventory_unit_attributes`

Current flexible attributes of a device.

| Column | Type | Nullable | Purpose |
|---|---|---:|---|
| `id` | bigint PK | No | |
| `inventory_unit_id` | bigint FK | No | Physical unit |
| `attribute_key` | varchar | No | `battery_health`, etc. |
| `value` | text | Yes | Current value |
| `value_type` | varchar | No | text/number/boolean/date/etc. |
| `created_at` | datetime | No | |
| `updated_at` | datetime | No | |

Recommended unique:

```text
inventory_unit_id + attribute_key
```

Example:

```text
battery_health = 82
box_included = true
screen_condition = minor scratches
```

---

# 5. `inventory_unit_movements`

Records **what happened to the physical unit**.

| Column | Type | Nullable | Purpose |
|---|---|---:|---|
| `id` | bigint PK | No | |
| `shop_id` | bigint FK | No | Shop |
| `inventory_unit_id` | bigint FK | No | Device |
| `stock_transaction_id` | bigint FK | Yes | Related aggregate stock transaction |
| `referenceable_type` | varchar | Yes | Purchase/Order/Buyback/etc. |
| `referenceable_id` | bigint | Yes | Source document ID |
| `from_location_id` | bigint FK | Yes | Previous location |
| `to_location_id` | bigint FK | Yes | Destination |
| `movement_type` | varchar | No | Event |
| `from_status` | varchar | Yes | Previous status |
| `to_status` | varchar | No | New status |
| `performed_by` | bigint FK | Yes | User |
| `occurred_at` | datetime | No | Event time |
| `note` | text | Yes | |
| `created_at` | datetime | No | |
| `updated_at` | datetime | No | |

Examples:

```text
purchase_received
sale
sale_return
buyback_received
purchase_return
transfer
inspection
repair_out
repair_return
loss
disposal
```

---

# 6. `inventory_unit_state_history`

Stores historical condition/grade/cost/pricing snapshots.

| Column | Type | Nullable | Purpose |
|---|---|---:|---|
| `id` | bigint PK | No | |
| `shop_id` | bigint FK | No | Shop |
| `inventory_unit_id` | bigint FK | No | Physical device |
| `referenceable_type` | varchar | Yes | Purchase/Buyback/Inspection/etc. |
| `referenceable_id` | bigint | Yes | Related source |
| `event_type` | varchar | No | Snapshot reason |
| `location_id` | bigint FK | Yes | Location at that time |
| `status` | varchar | Yes | Status snapshot |
| `condition` | varchar | Yes | Condition snapshot |
| `condition_grade` | varchar | Yes | Grade snapshot |
| `purchase_cost` | decimal(15,4) | Yes | Cost at that time |
| `asking_price` | decimal(15,4) | Yes | Asking price at that time |
| `minimum_sale_price` | decimal(15,4) | Yes | Price floor at that time |
| `warranty_until` | date | Yes | Warranty at that time |
| `attributes_snapshot` | json | Yes | Battery/condition/etc. snapshot |
| `performed_by` | bigint FK | Yes | User |
| `occurred_at` | datetime | No | Snapshot date |
| `note` | text | Yes | |
| `created_at` | datetime | No | |
| `updated_at` | datetime | No | |

This table answers:

```text
What was this phone's condition/value at that time?
```

---

# 7. `buybacks`

Header for secondhand/customer buyback transaction.

| Column | Type | Nullable | Purpose |
|---|---|---:|---|
| `id` | bigint PK | No | |
| `shop_id` | bigint FK | No | Shop |
| `seller_customer_id` | bigint FK | Yes | Seller/customer |
| `location_id` | bigint FK | No | Receiving location |
| `buyback_no` | varchar | No | Buyback number |
| `total_amount` | decimal(15,4) | No | Total amount paid |
| `payment_status` | varchar | Yes | Payment status |
| `status` | varchar | No | draft/completed/cancelled |
| `bought_at` | datetime | No | Buyback time |
| `created_by` | bigint FK | Yes | Staff |
| `note` | text | Yes | |
| `created_at` | datetime | No | |
| `updated_at` | datetime | No | |

Recommended unique:

```text
shop_id + buyback_no
```

---

# 8. `buyback_details`

Exact serialized devices purchased back.

| Column | Type | Nullable | Purpose |
|---|---|---:|---|
| `id` | bigint PK | No | |
| `buyback_id` | bigint FK | No | Buyback header |
| `item_id` | bigint FK | No | Item |
| `inventory_unit_id` | bigint FK | No | Existing/new physical device |
| `buyback_price` | decimal(15,4) | No | Amount paid for device |
| `condition` | varchar | Yes | Condition at buyback |
| `condition_grade` | varchar | Yes | Grade at buyback |
| `asking_price` | decimal(15,4) | Yes | Planned resale price |
| `minimum_sale_price` | decimal(15,4) | Yes | Optional floor |
| `attributes_snapshot` | json | Yes | Battery/etc. at buyback |
| `note` | text | Yes | |
| `created_at` | datetime | No | |
| `updated_at` | datetime | No | |

For an IMEI already known to ZiiGwat:

```text
inventory_unit_id = existing device
```

Do **not** create another `inventory_unit`.

---

## Final change summary

**Existing tables modified:**

```text
items
    + inventory_tracking

order_details
    + inventory_unit_id
    + cost_price_snapshot

sale_return_details
    + inventory_unit_id

purchase_return_details
    + inventory_unit_id

stock_transfers
    + inventory_unit_id

adjustment_details
    + inventory_unit_id
```

**New tables:**

```text
item_identifier_requirements

inventory_units

inventory_unit_identifiers

inventory_unit_attributes

inventory_unit_movements

inventory_unit_state_history

buybacks

buyback_details
```
