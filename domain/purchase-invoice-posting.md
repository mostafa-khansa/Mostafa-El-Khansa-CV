# Purchase Bill / Invoice (doc_type 12) — Domain Reference

> **Purpose:** Reverse-engineered business rules for creating, totaling, VAT aggregation, and GL posting of purchase bills (goods, assets, expenses). Intended for a clean Node.js/TypeScript service layer — **not** a copy of Sybase triggers/procedures.
>
> **Sources:** Yii monolith (`wizard/protected/modules/sales`, `account`, `product`), JS (`wizard/js/doc.js`, `custom.js`), MySQL routines on `wizard` DB (`in_compute_doc_vat`, `in_val_pur`, `in_post_pur_invoice`, `in_post_pur_detail`, `in_post_pur_vat`, `in_post_pur_acc`, `in_post_expense_acc`, `in_post_asset_acc`).
>
> **Companions:** [Sales Invoice posting](./sales-invoice-posting.md) (doc_type 10/11); [Supplier payment posting](./payment-posting.md) (settlements against AP / doc_type 12/13). Purchases are the mirror entry to sales (debit cost/inventory + input VAT, credit supplier).

---

## 1. Document identity and tables

| Concept | Value |
|--------|--------|
| Document type | **12** = Purchase Invoice / Bill; **13** = Return Purchase (posting reverses D/C) |
| Header | `in_purchases` |
| Lines | `in_purdetail` |
| VAT aggregate (reporting / FX) | `in_doc_vat` (rebuilt on save; `doc_type` 12 or 13) |
| GL lines | `ac_trans` (via `ac_trans_add`) |
| Supplier AP link | `g_aux_account` by header `pur_category` (see §6.2) |

### Header `pur_category` (document kind)

| `pur_category` | UI / module | Posting rule option (`wz_get_option_number`) |
|----------------|-------------|-----------------------------------------------|
| **1** | Goods / stock purchase | `g_post_pur` |
| **2** | Fixed-asset purchase | `g_post_asset` |
| **3** | Expense bill | `g_post_exp` |

Other header flags used at post time: `exp_goods`, `is_exp_provision`, `is_deferred`, `acc_type` (override supplier account type), `in_lc` (letter of credit), `rate_1` / `rate_2`, `division` / line `p_division`.

### Line `in_product.category` (what each line posts to)

Resolved inside **`in_post_pur_detail`** (independent of header `pur_category` for mixed lines):

| `in_product.category` | Account resolver |
|----------------------|------------------|
| **1** | Stock: `in_post_pur_acc` (inventory or purchases account — §4.1) |
| **2** | Asset: `in_post_asset_acc(..., post_type 5)` |
| **3** (else) | Expense: `in_post_expense_acc(..., acc_type 0)` |

### Save / validate / post pipeline (application)

Typical goods purchase commit — from `Purchase.php`:

1. **Line & header amounts** — UI grids + shared doc discount / VAT routines (`doc.js`, purchase panels).
2. **On commit (before post)**:
   - `CALL in_recover_charges_invoice(:doc_no)` (when used)
   - `CALL in_compute_doc_vat(12, :doc_no)` (or **13** for returns)
   - `CALL in_val_pur(:doc_no, 1)` (second arg `1` = validate sub-items when applicable)
3. **Post to GL** — same flip pattern as sales:
   - `UPDATE in_purchases SET posted = 0 WHERE invoice = :id`
   - `UPDATE in_purchases SET posted = 1 WHERE invoice = :id`
   - Intended to invoke **`in_post_pur_invoice`** (logic lives in DB procedures; this MySQL instance may not expose legacy triggers — `information_schema.TRIGGERS` can be empty while routines remain authoritative).

API wrapper `wapi_pur_posted` recomputes VAT and sets `posted = 1` but does not replace full GL rules; production posting still centers on **`in_post_pur_invoice`**.

---

## 2. Line-level mathematics

Purchase lines use the **same field model and formulas as sales** (`amount`, `discount`, `gross_amount`, `discount_p`, `vat`, `net_amount`). See [Sales Invoice §2](./sales-invoice-posting.md#2-line-level-mathematics) for symbols and UI allocation rules.

### Validation (`in_val_pur`)

Header vs detail (currency tolerance by precision):

- `truncate(in_purchases.amount - in_purchases.discount)` ≈ Σ `(gross_amount - discount_p)`
- `truncate(in_purchases.vat)` ≈ Σ `vat` from **`in_doc_vat`** where `doc_type IN (12, 13)` and `doc_id = invoice`

Optional line-level price checks (when `wz_rules_cust.det_id = 3` rule is off) mirror sales: `amount`, `gross_amount`, `net_amount`, and `vat` vs qty × unitprice and rates.

Payment / credit checks on save often use:  
`total = amount - discount + vat` (same pattern as `Sales.php`).

---

## 3. VAT aggregation (`in_compute_doc_vat` for doc_type 12 / 13)

On call for purchase `p_doc_id`:

1. Load header: currency precision, `thedate`, `rate_1`, `rate_2`, `pending_del`, taxable net, `aux`, `pay_mode`.
2. `DELETE FROM in_doc_vat WHERE doc_type = p_doc_type AND doc_id = p_doc_id`.
3. **Insert** rows grouped by `vat_cat`, `vat_pc`, currency from `in_purdetail`:

   - **Taxable base per bucket:** `sum(gross_amount - discount_p)`
   - **VAT per bucket:** `sum(vat)` rounded to document currency precision
   - **FX:** `amount_1`, `amount_2`, `vat_1`, `vat_2` via `g_get_exch_rate` when `rate_1 = 0`, else `sum(line) × rate_1/2`

4. If `pending_del <> 2`, may call `in_document_to_deliver(p_doc_id, p_doc_type)`.
5. Calls `in_compute_doc_tax(...)` for additional tax buckets (same doc types as sales PO chain).

**Implication for rewrite:** GL input VAT should align with **`in_doc_vat`** buckets; supplier AP line uses **`sum(vat_1)` / `sum(vat_2)`** from `in_doc_vat` for base-currency amounts on the credit (see §6.2).

---

## 4. Chart of accounts — purchases, inventory, expense, assets

### 4.1 Stock lines — `in_post_pur_acc`

Function: **`in_post_pur_acc(p_post_acc, p_product_id, p_vat_cat, p_aux, p_acc_type)`**  
Called from **`in_post_pur_detail`** (and from sales COGS credit via `acc_type = 2` on rule **1005**).

| `p_acc_type` | Returns |
|--------------|---------|
| **0** | Normal purchase / stock account (`account_id`, or `local_acc` / `import_acc` when `is_loc_imp`) |
| **1** | Discount account (when `g_post_acc.is_discount = 1`) |
| **2** | **Perpetual inventory** — `inv_acc` from `g_post_fam` |
| **3–6** | Adjustment gain/loss, COGS, closing (used on other doc types) |

**Perpetual vs periodic** (`use_perp_inventory` option):

| `use_perp_inventory` | `post_acc` passed | `p_acc_type` | Typical GL role |
|---------------------|-------------------|--------------|-----------------|
| **0** | `g_post_pur` | **0** | Purchases / stock clearing (`account_id`) |
| **1** | **1005** (“Inventory” rule) | **2** | Balance-sheet **inventory** (`inv_acc`) |

Family resolution matches sales: `g_post_acc` → `fam_level` → `in_product.fam2…fam9` → `g_post_fam` row `(post_acc, post_type, vat_cat, family_name)`.  
`post_type` **2** if discount posting and `p_acc_type = 1`; **3** if exempt supplier (`g_aux.apply_vat = 0`) and rule `is_exempted`.

### 4.2 Expense lines — `in_post_expense_acc`

Posting rule: **`g_post_exp`**.  
`p_acc_type`: **0** expense, **1** discount, **2** provision, **3** expense-on-production.

Lookup: `g_post_fam` (when `post_fam = 1`) or **`g_post_product`** by product.

### 4.3 Asset lines — `in_post_asset_acc`

Posting rule: **`g_post_asset`**.  
Purchase posting uses **`p_post_type = 5`** → asset purchase account (`pur_acc`, or local/import split).

### 4.4 Admin UI

Posting Account screens maintain matrices for purchases (`post_type = 2` in `g_post_acc`), assets, and expenses — views such as `vw_posting_accounts_*` and tabs in `wizard/protected/modules/account/views/postingaccount/`.

Separate discount options (by doc family):

| Option | Meaning |
|--------|---------|
| `post_discount_p` | Post purchase discounts to separate accounts (goods) |
| `post_discount_a` | Asset purchases |
| `post_discount_e` | Expense bills |

Runtime behavior is driven by **`g_post_acc.is_discount`** on the active rule (`g_post_pur`, `g_post_asset`, or `g_post_exp`).

---

## 5. Chart of accounts — VAT input

Procedure: **`in_post_pur_vat`**

Master: **`in_vat`** per `vat_cat` on each line.

| `post_vat_aux` (`wz_get_option_number`) | VAT account used |
|----------------------------------------|------------------|
| **0** | `in_vat.pur_vat_acc` |
| **1** | `in_vat.pur_vat` (may resolve **`g_get_aux_vat_acc`** for supplier-specific VAT account + auxiliary) |

Amounts: from **`in_doc_vat`** grouped by resolved account (and `aux_id` when present).  
**Normal bill:** **Debit** input VAT (`p_dc = 'D'` from `in_post_pur_invoice`).  
**Return (13):** Debit/Credit swapped.

### Reverse-charge / special VAT (`g_is_apply_vat(p_date, 1) = 2`)

For each VAT bucket:

1. Post VAT line with `p_dc` (debit on normal bill).
2. Post **mirror** line with opposite D/C on same VAT account.
3. If **not** deferred: additional **debit** to a representative **inventory/expense/asset** account for **total VAT** (amounts from `in_doc_vat`) — VAT effectively absorbed into cost base.
4. If **deferred**: extra line on deferred account from `in_deferred`.

Sales output VAT (`in_post_sales_vat`) credits liability and uses `sales_vat*` columns; purchases debit recoverable VAT and use **`pur_vat*`**.

---

## 6. GL posting — `in_post_pur_invoice`

**Normal bill (`p_is_return = 0`):** `v_dc_1 = 'C'`, `v_dc_2 = 'D'`.  
**Return (`p_is_return = 1`):** `v_doc_type = 13`; D/C swapped.

Unpost: `p_posted = 0` → `DELETE FROM ac_trans WHERE doc_id = p_invoice AND doc_type = v_doc_type`.

### 6.1 Cost lines — `in_post_pur_detail`

Resolve `v_post_acc` from `pur_category` (§1).  
`v_post_disc = g_post_acc.is_discount` for that rule.

**Debit** cost accounts (grouped by account, cost center, division, optional line description for assets/expenses):

| `v_post_disc` | Debit amount per group |
|---------------|-------------------------|
| **0** | Σ `(gross_amount - discount_p)` |
| **1** | Σ `amount` |

**Separate discount lines** (when `v_post_disc = 1`): **Credit** discount account (`acc_type 1` on pur / asset / expense resolver) for Σ `(discount + discount_p)`.

**Special ordering:** expense provision + perpetual + `pur_category = 3` may use higher `order_no` (5000) on detail lines before supplier credit.

### 6.2 Accounts payable (supplier)

When `p_net_amount + p_vat + p_tax_1…p_tax_4 > 0`:

- **Credit** supplier: `g_aux_account` for `aux = supplier` and:
  - `pur_category = 1` → **`acc_type = 2`**
  - `pur_category = 2` → **`acc_type = 9`**
  - `pur_category = 3` → **`acc_type = 8`**
  - Or explicit `p_acc_type` when provided
- **Document currency amount:** `p_net_amount + p_vat + p_tax_1 + p_tax_2 + p_tax_3 + p_tax_4`
- **Base currencies:** net/taxes from header rates; **VAT leg of FX** uses `sum(vat_1)`, `sum(vat_2)` from `in_doc_vat` (doc_type 12/13)
- `value_date` may come from `in_get_min_date_inv_terms`

### 6.3 VAT input

If `p_vat > 0`: **`in_post_pur_vat`** with `p_dc = v_dc_2` (**Debit** on normal bill).  
If `g_get_aux_vat_acc` fails when `post_vat_aux = 1`, procedure may exit without posting VAT lines (`p_is_return` OUT = 0).

### 6.4 Optional postings (same procedure)

| Condition | Behavior |
|-----------|----------|
| `p_is_deferred = 1` | `in_post_pur_deferred` — **debit** `in_deferred.account_id` instead of `in_post_pur_detail` (except LC path) |
| `p_in_lc` + `pur_category = 3` + not return | **Debit** LC account (`g_aux_account`, `acc_type = 53`); skip normal detail |
| `p_tax_1` … `p_tax_4` | **Debit** `in_tax.pur_account_id` by tax type |
| `exp_goods = 1` + perpetual + `is_exp_provision` | After supplier line: `in_post_pur_detail_perp_exp_goods` — paired D/C on expense account (capitalization step) |
| `is_exp_provision = 1` + `pur_category = 1` | `in_post_pur_provision_exp` — **debit** expense, **credit** provision (`in_provision_exp`) |
| `use_dp_py <> 0` + not return | `in_post_pur_dp` — down-payment netting vs supplier |
| Rounding | Adjust `ac_trans` `amount_1` / `amount_2` on supplier or last VAT line; then `ac_is_doc_balanced` |

### 6.5 Minimal double-entry (typical goods bill)

No separate discount posting, no taxes, no deferred/LC/provision/exp_goods, standard VAT:

| # | Account | D/C | Amount (doc ccy) |
|---|---------|-----|------------------|
| 1 | Inventory or purchases (`in_post_pur_acc` / rule 1005 — §4.1) | **D** | Σ `(gross_amount - discount_p)` |
| 2 | Input VAT (`in_vat` per §5) | **D** | Σ line `vat` / `in_doc_vat` |
| 3 | Supplier AP (`g_aux_account`, `acc_type` per §6.2) | **C** | `(amount - discount) + vat` |

If **`g_post_acc.is_discount = 1`**: row 1 uses Σ `amount`; discounts post separately as **credits** on discount accounts.

### 6.6 Mirror vs sales (mental model)

| | Sales (10) | Purchase (12) |
|--|------------|---------------|
| Third party | Client | Supplier |
| Balance sheet receivable/payable | **Debit** AR | **Credit** AP |
| P&L / stock | **Credit** revenue | **Debit** expense / inventory / asset |
| VAT | **Credit** output VAT | **Debit** input VAT |

---

## 7. TypeScript rewrite — suggested modules

```
purchase-invoice/
  pricing/
    computeLineAmounts(line, currencyPrecision)   // shared with sales
    allocateDocumentDiscount(lines, docDiscount)
    computeHeaderTotals(lines, docDiscount)
  vat/
    computeLineVat(taxableBase, vatPc, supplierVatFlags)
    buildDocVatBuckets(lines, header)             // mirrors in_doc_vat 12/13
  validation/
    validateHeaderVsDetails(header, lines)        // mirrors in_val_pur
  posting/
    resolvePostAcc(purCategory)                   // g_post_pur | g_post_asset | g_post_exp
    resolveStockAccount(perpetual, line, aux)
    resolveExpenseAccount(line, aux)
    resolveAssetAccount(line, aux)
    resolveVatAccount(vatCat, supplier, postVatAux)
    resolveSupplierApAccount(purCategory, accTypeOverride)
    buildJournalEntries(purchase, lines, options) // mirrors in_post_pur_*
```

### Invariants to preserve

1. **Two discount layers:** line (`discount`) vs document (`discount_p` + header `discount`).
2. **Cost posting base:** **`gross_amount - discount_p`** unless discount-is-separate mode uses **`amount`** for main lines.
3. **Product category** drives account resolver even when header `pur_category` differs.
4. **Perpetual inventory** forces posting rule **1005** + `inv_acc`, not `g_post_pur` main account.
5. **FX:** supplier credit uses **`in_doc_vat.vat_1/vat_2`** for VAT portion of base currencies; detail lines use header `rate_1`/`rate_2` or `g_get_exch_rate`.
6. **Options:** `use_perp_inventory`, `g_post_pur` / `g_post_asset` / `g_post_exp`, `post_vat_aux`, `post_discount_p|a|e`, `use_dp_py`, deferred, `exp_goods`, `is_exp_provision`, reverse VAT mode.

---

## 8. Key code references (monolith)

| Area | Path |
|------|------|
| Save + VAT + validate + post flip | `wizard/protected/modules/sales/models/Purchase.php` (~1323–1348) |
| Returns / other flows | Same file (`in_compute_doc_vat(13, …)`, `in_val_pur`) |
| Purchase type / grids | `wizard/protected/modules/sales/controllers/PurchaseController.php`, `ExpensesController.php`, `PurchaseAssetsController.php` |
| `pur_category` / `exp_goods` UI | `wizard/js/custom.js` (`pur_category_exp_goods`) |
| Posting rule UI | `wizard/protected/modules/account/views/postAccount/`, `postingaccount/` |
| Deferred expense events | `wizard/protected/modules/global/controllers/InDeferredController.php` (`in_post_pur_acc`, `in_post_expense_acc`) |
| VAT master | `wizard/protected/modules/global/models/Vat.php` (`in_vat` — `pur_vat`, `pur_vat_acc`, closing columns) |

### Database routines (inspect via `information_schema.ROUTINES`)

- `in_compute_doc_vat` (branches 12, 13)
- `in_val_pur`
- `in_post_pur_invoice`
- `in_post_pur_detail`
- `in_post_pur_detail_perp_exp_goods`
- `in_post_pur_vat`
- `in_post_pur_acc`
- `in_post_expense_acc`
- `in_post_asset_acc`
- `in_post_pur_deferred`
- `in_post_pur_provision_exp`
- `in_post_pur_dp`
- `in_post_pur_description`

---

## 9. Related document types (VAT proc only)

`in_compute_doc_vat` also handles purchase orders (**17**, **18**), sales, offers, etc. This document focuses on **12 / 13** posted purchase bills.

---

*Last consolidated from codebase + MySQL routine definitions. Update this file when posting options or procedures change.*
