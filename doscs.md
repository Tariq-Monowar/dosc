# Stock system — detailed guide (shoes)

This document explains **how shoe stock is stored and updated** in this Django project, and **exactly which fields/tables** hold which data. It complements the short summary in the root [`README.md`](../README.md).

---

## 1. Big picture in one paragraph

- The **catalogue** describes the shoe model (`Product` + `Size` tables). It does **not** store “how many pairs we have”.
- Each **retailer (partner)** sells selected catalogue products as **`PartnerProduct`** (prices, SKU, colours, etc.).
- Sellable stock is stored **per partner, per product, per shoe size** in **`PartnerProductSize`** (`quantity` and optional `warehouse` on that row).
- If you use **multiple warehouses**, the app also keeps **`WarehouseInventory`**: one **`quantity`** per **(warehouse + partner product size)** combination.
- Every business event that should change stock should be reflected in **`StockMovement`** (audit trail). For **new** movements, Django can automatically update `WarehouseInventory` and `PartnerProductSize.quantity` — but **only for certain movement types** and only when `product_size`, `quantity > 0`, and `skip_inventory_update` is **not** passed (see section 5).

---

## 2. Layers of data (what “level” you are looking at)

```text
Product (global shoe model)
    └── PartnerProduct (this partner sells this product, with price etc.)
            └── PartnerProductSize (THIS is the stock line: one row per size, optional warehouse on row)
                    └── WarehouseInventory (optional “per warehouse bucket” for that same size line)
```

- **`Product`**: name, brand, description, gender, width, categories, links to size *charts* (`SizeTable`). **No stock quantity.**
- **`PartnerProduct`**: link to `Product` + `User` (partner), selling price, cost, SKU, season, colours (M2M). **`total_stock_quantity`** is a **Python property** that sums related `PartnerProductSize.quantity` rows — it is not a database column.
- **`PartnerProductSize`**: the row you point to from **cart**, **order**, and **stock movements** when you mean “this size of this partner’s offer”.

---

## 3. Tables and fields relevant to stock

### 3.1 `Product` (`Products` app)

**Purpose:** Catalogue definition of the shoe.

**Not used for stock counts.** Relevant for movements only as an optional extra link on `StockMovement.product` (helpful for reporting).

---

### 3.2 `PartnerProduct` (`Products` app)

**Purpose:** Partner-specific commercial data for one catalogue product.

| Field (idea)        | Database-ish name   | What it stores |
|--------------------|---------------------|----------------|
| Partner            | `partner_id` (FK → User) | Which retailer |
| Catalogue product  | `product_id` (FK → Product) | Which shoe model |
| Selling price      | `price`             | VK |
| Cost / purchase    | `cost_price`        | EK |
| SKU / trade code   | `sku`               | Your article code |
| Season, supplier, shoe type | `season`, `supplier`, `shoe_type` | Merchandising |
| Colours            | M2M → `Color`       | Which colour variants exist for this partner offer |
| Active / channels  | `is_active`, `local`, `online` | Visibility |

**Stock:** Indirect — sum of **`PartnerProductSize.quantity`** for this `PartnerProduct`.

---

### 3.3 `PartnerProductSize` (`Products` app) — **main stock line for shoes**

**Purpose:** One row = “this partner’s offer of this product in **one** `Size`”.

| Field | Typical meaning |
|-------|-----------------|
| `partner_product_id` | Which `PartnerProduct` (hence which partner + catalogue product) |
| `size_id` | Which `Size` row (EU/US value, insole range, etc.) |
| `warehouse_id` | Optional. If set, this row is tied to one `Warehouse`. Can be `NULL` for “unassigned” / global-quantity style usage (your process should stay consistent). |
| **`quantity`** | **Number of pairs** for this line. Updated by `StockMovement.save()` when rules in section 5 apply. |
| `min_stock`, `reorder_point` | Thresholds for alerts / reorder (business logic on top) |
| `ean_code` | Barcode for this **size** variant |

**Uniqueness (Django constraints):**

- Cannot duplicate the same `(partner_product, size, warehouse)` when warehouse is set.
- When `warehouse` is `NULL`, uniqueness is on `(partner_product, size)` only.

**Important:** `CartItem.size` and `Order.size` are **ForeignKeys to this model** — checkout is always tied to a **concrete stock line**.

---

### 3.4 `Warehouse` (`Others` app)

**Purpose:** A location (shop, back room, DC) belonging to a **partner**.

| Field | Meaning |
|-------|---------|
| `partner_id` | Owner (partner user) |
| `name`, `abbreviation`, `address` | Identity / labels |
| `status` | `active` / `inactive` |
| `sequence` | Sort order in UIs |
| `product` (M2M) | Which `PartnerProduct` articles are *associated* with this site (merchandising / configuration). **Not the same as** live quantity — quantities are in `PartnerProductSize` + `WarehouseInventory`. |

---

### 3.5 `WarehouseInventory` (`StockMovements` app)

**Purpose:** Snapshot: **how many pairs** of one **`PartnerProductSize`** sit in **one warehouse**.

| Field | Meaning |
|-------|---------|
| `warehouse_id` | Where |
| `product_size_id` | Which `PartnerProductSize` (same “size line” as sales) |
| **`quantity`** | On-hand **at this warehouse only** |

**Unique together:** `(warehouse, product_size)` — one row per pair.

**How it changes:** Through `WarehouseInventory.update_stock()` and `transfer_stock()` when `StockMovement` triggers them (section 5).

**Note on transfers:** On `transfer`, the code **only** moves amounts between `WarehouseInventory` rows. It does **not** add the same delta to `PartnerProductSize.quantity` in the `transfer` branch — so the **partner-size total** and **sum of warehouse buckets** may need your business rules to stay aligned (many teams mirror totals on `PartnerProductSize` separately or avoid using both models inconsistently).

---

### 3.6 `StockMovement` (`StockMovements` app)

**Purpose:** **Immutable-style log** of stock events (who, when, type, qty, warehouses, optional links).

| Field group | Fields | Meaning |
|-------------|--------|---------|
| Type / ref | `type`, `reference_number` | Movement kind; reference auto-filled on first save (`GRN-2026-00123`-style prefixes) |
| Product resolution | `product`, `partner_product`, `product_size` | Links for reporting / trace.**Automatic stock maths uses `product_size` → `PartnerProductSize`.** |
| Quantity | `quantity` | Pieces moved (**must be > 0** for auto inventory update path) |
| Where | `location`, `destination_location` | Source warehouse; destination for transfers |
| Procurement | `manufacturer_order`, `order_item` | Link purchase batch or parsed line (`ManufacturerOrders`) |
| Sale extras | `selling_price`, `lieferscheinnummer` | Unit VK at sale time; delivery note id |
| Meta | `user`, `notes`, `created_at`, `updated_at` | Who triggered it / comments / timestamps |

**Special save behaviour:**

- **`skip_inventory_update=True`** in `save(...)`: saves the movement but **does not** call `_update_warehouse_inventory()` — use when another code path already adjusted stock (e.g. POS).

---

### 3.7 Sales touchpoints (`Others` app)

| Model | Stock-related data |
|-------|---------------------|
| `CartItem` | `partner_product_id`, **`size_id` → PartnerProductSize**, `color`, `quantity` |
| `Order` | Same idea: FK to **`PartnerProductSize`** as `size` (nullable in model but conceptually “which size line”), plus customer, partner, qty, etc. |

These tables **reserve or record** commerce; they **do not** replace `StockMovement` — normally you **decrease stock** via a `sale` or `picking` movement when fulfilment happens (unless your front-end batches this differently).

---

### 3.8 Manufacturer purchases (`ManufacturerOrders` app)

| Model | Role |
|-------|------|
| `ManufacturerOrderUpload` | One batch/upload (excel/manual/image pipeline) |
| `OrderItem` | Parsed supplier order lines (sizes, qty, pricing, arrival flags…) |

These describe **what you ordered**, not necessarily what is already in the shop.

**Bridging stock:** When stock arrives you create **`StockMovement`** with:

- `type='goods-receipt'`
- `location` = warehouse you put cartons in
- `product_size` = the `PartnerProductSize` that matches that SKU/EAN/size
- `quantity` = received pairs  

Optionally:

- `manufacturer_order` / `order_item` for auditing.

---

### 3.9 Non-shoe stock (same database, different model)

**`Others.Accessories`:** simple integer field **`stock`** on the accessory row itself — **not** wired through `PartnerProductSize` / `WarehouseInventory` unless you extend the code.

**`Clothing.ClothingVariant`:** field **`stock`** per clothing variant — separate from footwear stock.

---

## 4. All movement `type` values (choices)

Defined on `StockMovement`. Labels are German UI strings in code.

| `type` value | Label (German) |
|--------------|----------------|
| `order-placed` | Bestellt |
| `goods-receipt` | Wareneingang |
| `sale` | Verkauf |
| `transfer` | Umlagerung |
| `picking` | Kommissionierung |
| `production` | Produktion (Verbrauch) |
| `production-return` | Fertigmeldung (zurück ins Lager) |
| `customer-return` | Rückgabe Kunde |
| `supplier-return` | Retoure Lieferant |
| `return` | Rückbuchung |
| `scrap` | Ausschuss / Defekt |
| `inventory-adj` | Inventurkorrektur |

---

## 5. Automatic stock update rules (exact behaviour)

 Preconditions for **automatic** `_update_warehouse_inventory()`:

1. The `StockMovement` row is **new** (`is_new`).
2. `product_size` is set (**FK to `PartnerProductSize`**).
3. **`quantity > 0`**.
4. `skip_inventory_update` was **not** passed to `save()`.

Then:

| `type` | Updates `WarehouseInventory`? | Updates `PartnerProductSize.quantity`? |
|--------|---------------------------------|----------------------------------------|
| `order-placed` | No | No |
| `goods-receipt`, `return`, `customer-return` | Yes, **add** `quantity` at **`location`** | Yes, **`+ quantity`** on that `PartnerProductSize` |
| `sale`, `picking`, `production`, `scrap` | Yes, **subtract** at **`location`** | Yes, **`- quantity`** |
| **`transfer`** | Yes: **move** between `location` and `destination_location` | **No change** in this code path |

**Movement types that exist on the model but are not handled in `_update_warehouse_inventory`:**  
examples include **`supplier-return`**, **`production-return`**, and **`inventory-adj`**. Saving them still creates a history row, but **automatic** changes to `WarehouseInventory` / `PartnerProductSize.quantity` do not run — extend the logic or update stock another way if needed.

If `WarehouseInventory` would become **negative**, `update_stock` raises **`ValueError`** (protection against overselling in that warehouse bucket).

---

## 6. Example scenarios (mental data flow)

### 6.1 Receive 24 pairs EU 42 into “Back store Zürich”

1. Locate or create **`PartnerProductSize`** for that partner-product + EU 42.
2. Create **`StockMovement`**:  
   `type=goods-receipt`, `quantity=24`, `location=zurich_warehouse`, `product_size=` that row.  
3. Result: **`PartnerProductSize.quantity` += 24**; **`WarehouseInventory`** row for `(zurich, that product_size)` **+= 24** (creates row if missing).

### 6.2 Customer buys 1 pair — same size row

1. **`Order`** (or **`CartItem`**) references that **`PartnerProductSize`**.
2. On fulfilment create **`StockMovement`**:  
   `type=sale`, `quantity=1`, `location`=shipping warehouse, `product_size`=same row, optional **`selling_price`**.  
3. Result: inventory **decrease** by 1 in both **`WarehouseInventory`** and **`PartnerProductSize.quantity`** (assuming preconditions).

### 6.3 Move 6 pairs between two shops

 **`StockMovement`**:  
`type=transfer`, `quantity=6`, `location`=source WH, **`destination_location`**=destination WH, `product_size`=same stock line.

- **`WarehouseInventory`**: −6 source, +6 destination.  
- **`PartnerProductSize.quantity`**: unchanged by this snippet (section 5).

Plan your workflows so totals stay truthful for reporting.

---

## 7. Where to look in code

| Topic | File |
|-------|------|
| Size-level quantity + uniqueness | [`Products/models.py`](../Products/models.py) (`PartnerProductSize`) |
| Warehouse master | [`Others/models.py`](../Others/models.py) (`Warehouse`) |
| Movements + auto update | [`StockMovements/models.py`](../StockMovements/models.py) |
| Cart / sales FK to size row | [`Others/models.py`](../Others/models.py) (`CartItem`, `Order`) |
| Purchases lines | [`ManufacturerOrders/models.py`](../ManufacturerOrders/models.py) |

---

## 8. Prisma mirror

For a relational diagram and table names usable outside Django, see [`prisma/schema.prisma`](../prisma/schema.prisma). Django remains authoritative; regenerate or compare if you migrate the DB.
