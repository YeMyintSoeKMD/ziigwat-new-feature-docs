# My question:
```
i purchase Iphone 18 pro. its IMEI is 12345. and i sell it to someone.
when 1 yr later. that phone came back to my shop. my shop bought it back. then tell me how will be the processes of that
```

---

Yes. In this case, **the same `inventory_unit` should be reused**. Do not create a second device record for IMEI `12345`.

The key distinction is:

> **Sale Return** = reversing/returning a previous sale.  
> **Buyback** = your shop makes a brand-new purchase of a device that it previously sold.

So after one year, this should be treated as a **Buyback / Secondhand Purchase**, not as a sale return.

### Example lifecycle

Assume:

```text
Item:
iPhone 18 Pro

Inventory Unit:
ID = 101
IMEI = 12345
```

First, your shop originally purchases the phone:

```text
Supplier
   ↓
Purchase P-001
   ↓
iPhone 18 Pro
IMEI 12345
```

The system creates:

```text
inventory_unit #101

item_id = iPhone 18 Pro
IMEI = 12345
status = in_stock
location = Yangon
purchase_cost = 3,000,000
condition = new
```

Stock:

```text
iPhone 18 Pro stock +1
```

Movement:

```text
purchase_received

null → in_stock
```

Then you sell it:

```text
Order SO-001

iPhone 18 Pro
IMEI 12345
Price = 3,500,000
```

`order_details`:

```text
inventory_unit_id = 101
quantity = 1
final_price = 3,500,000
```

The device becomes:

```text
inventory_unit #101

status = sold
```

Stock:

```text
-1
```

Movement:

```text
sale

in_stock → sold
```

At this point, the device is no longer owned by your shop, but **the inventory unit record must remain forever** because it contains its history.

---

## One year later: shop buys the phone back

Suppose the customer comes back after one year and says:

> I want to sell this iPhone back to your shop.

The staff scans:

```text
IMEI 12345
```

The system should search **all serialized units**, not only current stock.

It finds:

```text
iPhone 18 Pro
IMEI 12345

Current status: Sold

Previously sold:
Order SO-001
1 year ago
```

This is actually very useful because the shop immediately knows the device's history.

The system should then offer something like:

```text
Device found in history

iPhone 18 Pro
IMEI: 12345
Status: Previously Sold

[ Buy Back Device ]
```

The shop negotiates:

```text
Buyback Price: 1,800,000
Condition: Used
Grade: B
Battery Health: 82%
Asking Price: 2,200,000
Location: Yangon
```

Then the shop creates a **new Buyback transaction**.

---

# I would add a Buyback module

I recommend adding:

```text
buybacks
    ↓
buyback_details
```

### `buybacks`

| Field | Purpose |
|---|---|
| id | Buyback transaction |
| shop_id | Shop |
| customer_id | Seller, nullable |
| location_id | Receiving location |
| buyback_no | BB-00001 |
| total_amount | Amount paid |
| status | completed / cancelled |
| bought_at | Date |
| created_by | Staff |
| note | Notes |

### `buyback_details`

| Field | Purpose |
|---|---|
| id | Detail |
| buyback_id | Buyback |
| item_id | Product |
| inventory_unit_id | Exact device |
| buyback_price | New acquisition cost |
| condition | Current condition |
| condition_grade | Current grade |
| asking_price | New intended selling price |
| note | Notes |

For the example:

```text
Buyback BB-001

Seller: Customer
Location: Yangon

iPhone 18 Pro
IMEI 12345

Buyback Price: 1,800,000
Condition: Used
Grade: B
Asking Price: 2,200,000
```

---

# Do not create another inventory unit

This is very important.

Do **not** do:

```text
inventory_unit #101
IMEI 12345
status = sold

inventory_unit #999
IMEI 12345
status = in_stock
```

That would create two physical identities for the same phone.

Instead:

```text
inventory_unit #101
IMEI 12345
```

remains the same device throughout its lifetime.

Only its state changes.

---

# When bought back

I would initially change:

```text
sold
   ↓
buyback_received
   ↓
quarantined
```

Why `quarantined` first?

Because a phone coming back after one year should probably be inspected before becoming sellable.

So after buyback:

```text
inventory_unit #101

status = quarantined
location = Yangon

purchase_cost = 1,800,000
condition = used
condition_grade = B

asking_price = 2,200,000
```

Stock increases:

```text
stocks.balance_quantity +1
```

because the shop physically owns the phone again.

But:

```text
Available to Sell = 0
```

until inspection is completed.

Remember our rule:

```text
On Hand Stock ≠ Available Stock
```

`quarantined` can count as on-hand stock, but it isn't available to sell.

---

# After inspection

Staff checks the phone.

If everything is fine:

```text
quarantined
     ↓
inspection_passed
     ↓
in_stock
```

Now:

```text
On Hand = 1
Available = 1
```

No aggregate stock increase happens at this point because the phone was already physically received during the buyback.

Only its status changes.

If there is a problem:

```text
quarantined
     ↓
damaged
```

or:

```text
quarantined
     ↓
under_repair
```

---

# Device history becomes very useful

After this process, searching IMEI `12345` could show:

```text
iPhone 18 Pro
IMEI 12345

CURRENT
──────────────────────────

Status:
In Stock

Condition:
Used - Grade B

Location:
Yangon

Current Cost:
1,800,000

Asking Price:
2,200,000


HISTORY
──────────────────────────

2026
Purchased from Supplier
Cost: 3,000,000

2026
Received into Yangon
Status: In Stock

2026
Sold
Order: SO-001
Customer: Mg Mg
Price: 3,500,000

2027
Bought Back
Buyback: BB-001
Seller: Mg Mg
Price: 1,800,000

2027
Received for Inspection
Status: Quarantined

2027
Inspection Passed
Status: In Stock
```

This is exactly where your `inventory_unit_movements` becomes powerful.

---

# Then you sell it again

Suppose you later resell it for:

```text
2,100,000
```

Create another order detail:

```text
Order SO-500

inventory_unit_id = 101
quantity = 1

cost_price_snapshot = 1,800,000
final_price = 2,100,000
```

Then:

```text
in_stock → sold
```

Stock:

```text
-1
```

Movement:

```text
sale
```

Now the same device has **two sales in its history**:

```text
IMEI 12345

Purchase #1
Supplier → Shop
3,000,000

Sale #1
Shop → Customer A
3,500,000

Buyback
Customer A → Shop
1,800,000

Sale #2
Shop → Customer B
2,100,000
```

That is completely valid.

---

## One change I would make to our previous database design

This scenario shows that this field alone:

```text
inventory_units.purchase_detail_id
```

is not enough to represent the full acquisition history.

A physical phone can enter your inventory multiple times:

```text
Supplier Purchase
        ↓
Sale
        ↓
Customer Buyback
        ↓
Sale
        ↓
Buyback again
        ↓
Sale again
```

I would either rename it:

```text
original_purchase_detail_id
```

to mean only the **first/original purchase**, or eventually remove the dependency on that field for acquisition history.

The actual complete history should come from:

```text
inventory_unit_movements
```

and the corresponding business documents:

```text
purchases
orders
buybacks
sale_returns
purchase_returns
transfers
adjustments
```

So one phone can have this lifecycle:

```text
                        IMEI 12345
                            │
                            ▼
                    Inventory Unit #101
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
           Identity                    History
        IMEI 12345               purchase_received
                                 sale
                                 buyback_received
                                 inspection
                                 sale
                                 buyback_received
                                 ...
```

The **identity never changes**. The ownership, condition, location, cost, price and status can change many times.

For a phone-shop POS like ZiiGwat, I would definitely add **Buyback / Secondhand Purchase** to the technical requirements, because this will be a common real-world workflow once IMEI tracking is supported.
