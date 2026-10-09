# Sales Invoice (doc_type 10) — Domain Reference

> **Purpose:** Reverse-engineered business rules for creating, totaling, VAT aggregation, and GL posting of sales invoices. Intended for a clean Node.js/TypeScript service layer — **not** a copy of Sybase triggers/procedures.
>
> **Sources:** Yii monolith (`wizard/protected/modules/sales`, `account`, `global`), JS (`wizard/js/sales.js`), MySQL routines on `wizard` DB (`in_compute_doc_vat`, `in_val_sales`, `in_post_sales_invoice`, `in_post_sales_detail`, `in_post_sales_vat`, `in_post_sales_acc`).

---

## 1. Document identity and tables

| Concept | Value |
|--------|--------|
| Document type | **10** = Sales Invoice; **11** = Return Sales Invoice (posting reverses D/C) |
| Header | `in_sales` |
| Lines | `in_salesdetail` |
| VAT aggregate (reporting / FX) | `in_doc_vat` (rebuilt on save) |
| GL lines | `ac_trans` (via `ac_trans_add`) |
| Client AR link | `g_aux_account` where `acc_type = 1` |
| Collections / allocation | [Cash Receipt posting](./cash-receipt-posting.md) (`ac_doc_det` → `in_inv_bal_det`); supplier side: [Payment posting](./payment-posting.md) |

### Save / validate / post pipeline (application)

1. **Line & header amounts** — UI + `SalesHelper::rows()` + `sales.setSummary()` / discount routines in `sales.js`.
2. **On commit (before post)** — from `Sales.php`:
   - `CALL in_compute_doc_vat(10, :invoice)`
   - `CALL in_val_sales(:invoice)`
3. **Post to GL** — pattern used throughout app:
   - `UPDATE in_sales SET posted = 0 WHERE invoice = :id`
   - `UPDATE in_sales SET posted = 1 WHERE invoice = :id`
   - Intended to invoke **`in_post_sales_invoice`** (procedure exists in DB; MySQL schema may not expose legacy triggers — posting logic is in procedures).

---

## 2. Line-level mathematics

### Symbols (per `in_salesdetail` line)

| Field | Meaning |
|-------|---------|
| `amount` | Extended price before line discount |
| `disc_pc` | Line discount % |
| `discount` | Line discount amount |
| `gross_amount` | Subtotal after line discount |
| `discount_p` | Allocated **document-level** discount |
| `vat_pc` | VAT rate % on line |
| `vat` | VAT amount |
| `net_amount` | Line total incl. VAT |

### Formulas (canonical — `SalesHelper.php` + `sales.js`)

1. **`amount`** ≈ `unitprice × qty` (or `× packages` when priced by pack); rounded to currency precision. Free lines (`is_free = 1`) → `0`.
2. **Line discount:**  
   `discount = amount × disc_pc / 100`  
   then **`Library::truncate(discount, currency_precision)`** (server).
3. **`gross_amount = amount - discount`** (rounded).
4. **`discount_p`** — initially `0`; set when user applies header `doc_discount` / `doc_discount_pc` (proportional allocation by `gross_amount`; last line absorbs rounding remainder).
5. **VAT** (when VAT applies and not export):  
   `vat = (gross_amount - discount_p) × vat_pc / 100`  
   VAT stored with **4 decimal** precision (`round` / `localTruncate(4)`). Export / no VAT → `vat = 0`, `vat_pc = 0`.
6. **Line net:**  
   `net_amount = amount - discount - discount_p + vat`  
   equivalently: `gross_amount - discount_p + vat`.

### Document-level discount allocation (`sales.js`)

- `doc_discount = doc_discount_pc / 100 × doc_amount` (or manual `doc_discount`).
- Per line (except last):  
  `discount_p = truncate((gross_amount / doc_amount) × doc_discount)`.
- Last line: remainder so Σ `discount_p` = `doc_discount`.
- After allocation, **recompute** `vat` on `(gross_amount - discount_p)` and refresh header via `sales.setSummary()`.

### Header rollup (`sales.setSummary`)

| UI / header field | Rule |
|-------------------|------|
| `doc_amount` (`in_sales.amount`) | Σ line `gross_amount` |
| `doc_vat` (`in_sales.vat`) | Σ line `vat` (rounded to currency precision) |
| `doc_discount` (`in_sales.discount`) | Document discount |
| **`doc_net_amount`** | `doc_amount + doc_vat - doc_discount` (+ optional sales taxes; retention may alter base per `in_sales_retention_method`) |

Payment terms / credit facility on save often use:  
`total = amount - discount + vat` (see `Sales.php`).

### Validation (`in_val_sales`)

- `truncate(in_sales.amount - in_sales.discount)` ≈ Σ `(gross_amount - discount_p)` within currency tolerance.
- `truncate(in_sales.vat)` ≈ Σ line `vat` within tolerance.
- Optional line checks: `amount` / `gross_amount` vs `qty × unitprice` (rule-dependent).

---

## 3. VAT aggregation (`in_compute_doc_vat` for doc_type 10)

On call for invoice `p_doc_id`:

1. `DELETE FROM in_doc_vat WHERE doc_type IN (10,11) AND doc_id = p_doc_id`.
2. **Insert** rows grouped by `vat_cat`, `vat_pc`, currency from lines:

   - **Taxable base per bucket:** `sum(gross_amount - discount_p)`
   - **VAT per bucket:** `sum(vat)` (rounded with document currency precision)
   - FX columns `amount_1`, `vat_1`, etc. via `g_get_exch_rate` or header `rate_1` / `rate_2` and `in_vat_is_comp_rate(aux, currency, date)`.

3. If `pending_del <> 2`, may call `in_document_to_deliver(p_doc_id, p_doc_type)`.
4. May call `in_compute_doc_tax(...)` for additional tax buckets.

**Implication for rewrite:** GL VAT amounts should match **sum of line `vat`**; `in_doc_vat` is the official VAT register slice by category + FX.

---

## 4. Chart of accounts — Sales revenue

### 4.1 Choose posting rule set (`post_acc`)

Function: **`in_post_sales_acc(p_post_acc, p_product_id, p_vat_cat, p_aux, p_acc_type)`**  
Called from **`in_post_sales_detail`** with export flag as `p_acc_type` 0 or 1 for revenue lines; `2` = discount; `3` = COGS.

| Priority | Option | Resolution |
|----------|--------|------------|
| Default | `g_post_sales_client_cat = 0` AND `g_post_sales_by_sales_cat = 0` | `wz_get_option_number('g_post_sales')` → `g_post_acc.post_acc` |
| Client category | `g_post_sales_client_cat = 1` | `g_aux` → `g_aux_cat.post_acc` for invoice `aux` |
| Sales category | `g_post_sales_by_sales_cat = 1` | `in_sales_cat.post_acc` for header `sales_cat` |

Example `g_post_acc` names (`post_type = 1`): Sales (1001), Sales category (1006), Client Category 1 (1007), Sales2 (1008).

### 4.2 Map product + VAT + client → account (`g_post_fam`)

Inside `in_post_sales_acc`:

1. Load **`g_post_acc`**: `post_fam`, `fam_level`, `is_discount`, `is_exempted`.
2. **Family** from **`in_product`** (`fam2`…`fam9`) using `fam_level`, unless `post_fam = 0` → `family_name = 1`.
3. **`post_type`** for `g_post_fam` row:
   - `1` normal
   - `2` if rule `is_discount = 1` and `p_acc_type = 2`
   - `3` if rule `is_exempted = 1` and `g_aux.apply_vat = 0` for client
4. Lookup **`g_post_fam`**: `(post_acc, post_type, vat_cat, family_name)` → `account_id`, `service_acc`, `export_acc`, `cog_acc`.
5. **Return account** by `p_acc_type` and product:
   - `p_acc_type = 1` → `export_acc`
   - `p_acc_type = 3` → `cog_acc`
   - `in_product.prod_type = 2` (service) → `service_acc`
   - else → `account_id` (default **sales revenue**)

### 4.3 Admin UI matrix (configuration)

Posting Account screens maintain **`wz_post_acc`** exposed as views e.g.:

- `vw_posting_accounts_sales_company`
- `vw_posting_accounts_sales_client_cat` / `_det`
- `vw_posting_accounts_sales_sales_cat`
- `vw_posting_accounts_sales_product`
- `vw_posting_accounts_sales_family` / `_family_name`

Columns include `account_id`, `account_id_serv`, `account_id_disc`, keyed by `doc_type = 10`, `post_acc_type`, `priority_id`, `vat_cat`, etc. Runtime posting in procedures uses **`g_post_fam`** after resolving `post_acc` as above.

### 4.4 Separate posting options (wz options)

For doc_type 10 UI (`postingaccount/_separateoptions.php`):

| Option | Meaning |
|--------|---------|
| `post_discount_s` | Post sales discounts to separate accounts |
| `post_client_nt_s` | Client exempted handling |
| `post_service_s` | Service products to service revenue accounts |

Related: **`g_post_acc.is_discount`** drives whether revenue is credited on **`amount`** vs **`gross_amount - discount_p`** and whether discounts post via separate lines (see §5).

---

## 5. Chart of accounts — VAT output

Procedure: **`in_post_sales_vat`**

Master: **`in_vat`** per `vat_cat` on each line.

| `post_vat_aux` (`wz_get_option_number`) | VAT account used |
|----------------------------------------|------------------|
| **0** | `in_vat.sales_vat_acc` |
| **1** | If `tax_refund` → `tax_refund_acc`; else if `self_supply` → `self_acc`; else if `prod_type = 2` → `sales_vat_serv`; else → `sales_vat` |

Amount: **Credit** (normal invoice) Σ line `vat` grouped by resolved account. FX on `ac_trans` uses `in_vat_is_comp_rate` and header rates.

If `post_vat_aux = 1` and not refund/self-supply, **`g_get_aux_vat_acc`** may set auxiliary on VAT line.

Other `in_vat` columns (purchases, closing, etc.) are used by other document types — not primary sales invoice output.

---

## 6. GL posting — `in_post_sales_invoice`

**Normal invoice (`p_is_return = 0`):** `v_dc_1 = 'D'`, `v_dc_2 = 'C'`.  
**Return (`p_is_return = 1`):** D/C swapped; `doc_type` 11.

### 6.1 Revenue & discounts — `in_post_sales_detail`

Resolve `v_post_acc` (§4.1).  
`v_post_disc = g_post_acc.is_discount` for that rule.

**Credit sales revenue** (grouped by account, cost center, division):

| `v_post_disc` | Credit amount per group |
|---------------|-------------------------|
| 0 | Σ `(gross_amount - discount_p)` |
| 1 | Σ `amount` |

**Separate discount lines** (when `v_post_disc = 1`): **Debit** discount account (`in_post_sales_acc(..., 2)`) for Σ `(discount + discount_p)`.

**Perpetual inventory** (`use_perp_inventory = 1`):

- **Debit** COGS — `in_post_sales_acc(..., 3)`
- **Credit** inventory — `in_post_pur_acc(1005, ..., 2)` at `in_avg_cost` / `in_avg_cost_lotno` × qty

### 6.2 Accounts receivable

If `p_net_amount + p_vat + taxes > 0` and not deferred:

- **Debit** client: `g_aux_account` where `aux = customer` and **`acc_type = 1`**
- Amount: **`p_net_amount + p_tax_1..p_tax_4 + p_vat`** (document + FX columns)
- `p_net_amount` = taxable net before VAT on header (aligned with `amount - discount` validation)

### 6.3 VAT output

**Credit** VAT account(s) from §5 for Σ line `vat` (via `in_post_sales_vat`).

### 6.4 Optional postings (same procedure)

| Condition | Behavior |
|-----------|----------|
| `p_retention > 0` | Credit AR (retention portion); Debit retention (`acc_type = 52` or default from `ac_account_type`) |
| `p_tax_1` … `p_tax_4` | Credit `in_tax.account_id` by tax type |
| `post_com = 1` | `in_post_sales_commission` |
| `use_dp <> 0` | `in_post_sales_dp` (down payment) |
| `p_is_deferred = 1` | `in_post_sales_deferred` instead of standard revenue path |
| Rounding | Adjust `ac_trans` amounts in base currencies; `ac_is_doc_balanced` |

### 6.5 Minimal double-entry (typical credit sale)

No separate discount posting, no perpetual inventory, no retention/taxes:

| # | Account | D/C | Amount (doc ccy) |
|---|---------|-----|------------------|
| 1 | Client AR (`g_aux_account`, `acc_type=1`) | **D** | `(amount - discount) + vat` |
| 2 | Sales revenue (`g_post_fam` via `in_post_sales_acc`) | **C** | Σ `(gross_amount - discount_p)` |
| 3 | VAT output (`in_vat` per §5) | **C** | Σ `vat` |

If **`g_post_acc.is_discount = 1`**: line 2 uses Σ `amount`; discounts post separately on discount accounts (debit).

---

## 7. TypeScript rewrite — suggested modules

```
sales-invoice/
  pricing/
    computeLineAmounts(line, currencyPrecision)
    allocateDocumentDiscount(lines, docDiscount)
    computeHeaderTotals(lines, docDiscount, taxes?)
  vat/
    computeLineVat(taxableBase, vatPc, isExport)
    buildDocVatBuckets(lines, header)  // mirrors in_doc_vat
  validation/
    validateHeaderVsDetails(header, lines)  // mirrors in_val_sales tolerances
  posting/
    resolvePostAcc(header)  // g_post_sales / client cat / sales cat
    resolveRevenueAccount(postAcc, line, aux, accType)
    resolveVatAccount(vatCat, header, product)
    buildJournalEntries(invoice, lines, options)  // mirrors in_post_sales_*
```

### Invariants to preserve

1. **Two discount layers:** line (`discount`) vs document (`discount_p` + header `discount`).
2. **VAT base** for lines, `in_doc_vat`, and revenue posting: **`gross_amount - discount_p`** (unless discount-is-separate mode uses `amount` for revenue).
3. **Rounding:** currency precision on money; VAT often 4 dp then header VAT rounded to currency.
4. **FX:** parallel amounts `amount_1`, `amount_2` on `ac_trans` — follow `rate_1`/`rate_2` or `g_get_exch_rate` rules.
5. **Options are behavioral:** `g_post_sales*`, `post_vat_aux`, `post_discount_s`, `use_perp_inventory`, `post_com`, `use_dp`, deferred flag.

---

## 8. Key code references (monolith)

| Area | Path |
|------|------|
| Save + VAT + validate | `wizard/protected/modules/sales/models/Sales.php` (~1209, ~1483) |
| Server line math | `wizard/protected/modules/sales/components/SalesHelper.php` (~1000–1060) |
| Client totals / doc discount | `wizard/js/sales.js` (`setSummary`, ~3780–4600) |
| VAT category model | `wizard/protected/modules/global/models/Vat.php` (`in_vat`) |
| Posting account UI | `wizard/protected/modules/account/views/postingaccount/` |
| Deferred revenue uses same acc resolver | `wizard/protected/modules/global/controllers/InDeferredController.php` (`in_post_sales_acc`) |

### Database routines (inspect via `information_schema.ROUTINES`)

- `in_compute_doc_vat`
- `in_val_sales`
- `in_post_sales_invoice`
- `in_post_sales_detail`
- `in_post_sales_vat`
- `in_post_sales_acc`

---

## 9. Related document types (VAT proc only)

`in_compute_doc_vat` also handles 11, 12, 13, 14, 15, 16, 17, 18, 192, etc. This document focuses on **10 / 11** sales invoices.

---

*Last consolidated from codebase + MySQL routine definitions. Update this file when posting options or procedures change.*
