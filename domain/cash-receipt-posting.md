# Cash Receipt (doc_type 2) — Domain Reference

> **Purpose:** Reverse-engineered business rules for customer cash receipts: settlement lines, invoice allocation, AR subledger (`in_inv_bal_det`), and GL posting. Intended for a clean Node.js/TypeScript service layer — **not** a copy of legacy triggers/procedures.
>
> **Sources:** Yii monolith (`wizard/protected/modules/receivable`), MySQL routines on `wizard` DB (`ac_post_receipt`, `api_create_receipt_*`, `in_inv_add_payments`, `in_inv_add_paid_amount`, `ac_check_total_detail`, `in_inv_add_doc_acc_bal`, `in_val_receipt_dp`).
>
> **Companions:** [Sales Invoice posting](./sales-invoice-posting.md) (creates AR and `in_inv_bal_det` installments); [Supplier payment posting](./payment-posting.md) (`doc_type` **3**, `ac_post_payment`).

---

## 1. Document identity and tables

| Concept | Value |
|--------|--------|
| Document type | **2** = Cash Receipt (client collection) |
| Header | `ac_receipt` (`aux`, `aux_type`, `downpay`, `posted`, `costcenter`, …) |
| Money in / settlement | `ac_receiptcash` (per line: `settlement`, `amount`, `in_out`, check/bank/card fields, `charges`) |
| GL application (AR) | `ac_receiptdetail` (`aux_id`, `account_id`, `amount`, `amount_dp`, `sub_acc`, …) |
| Invoice allocation | `ac_doc_det` (`doc_type = 2`, `doc_id = receipt`, `bal_doc_type` 10/11/…, `bal_doc_id`, `amount`, **`inv_det_id`**) |
| Open AR schedule | `in_inv_bal`, `in_inv_bal_det` (per sales invoice and installment) |
| GL lines | `ac_trans` (`doc_type = 2`, via `ac_trans_add`) |
| Client AR link | `g_aux_account` where **`acc_type = 1`** (typical receipt detail line) |

### Three layers (do not conflate)

| Layer | What it answers |
|-------|------------------|
| **`ac_receiptcash`** | How was money received? (cash, check, card, bank account) |
| **`ac_receiptdetail`** | Which **GL AR** account is credited (customer balance in the ledger)? |
| **`ac_doc_det`** | Which **open invoices** (subledger) does this receipt settle? |

**`ac_post_receipt`** posts only **cash + detail** to `ac_trans`. **`ac_doc_det`** drives **`in_inv_bal_det.paid`** via **`in_inv_add_payments`** (see §4) — no extra GL lines per allocated invoice.

### Save / validate / post pipeline (application)

From `Receipt.php` commit:

1. User enters **`ac_receiptcash`** and **`ac_receiptdetail`**, and optionally **`ac_doc_det`** (allocated documents tab).
2. `CALL g_get_aux_active(:aux, :aux_type, …)`
3. `CALL ac_check_total_detail(:receipt, 2)` — allocation cannot exceed AR detail (§5.2).
4. If `downpay = 1`: `CALL in_val_receipt_dp(:receipt)` — receipt total must cover linked orders (§6).
5. Post to GL:
   - `UPDATE ac_receipt SET posted = 0 WHERE receipt = :id`
   - `UPDATE ac_receipt SET posted = 1 WHERE receipt = :id`
   - Intended to invoke **`ac_post_receipt`** (logic in DB procedures; triggers may not appear in `information_schema` on all environments).

API / integration flow (`ReceiptController`, iSell): **`api_create_receipt_head`** → **`api_create_receipt_settlement`** → **`api_create_receipt_alloc_docs`** → **`api_create_receipt_posting`** → checks → posted flip.

---

## 2. AR subledger — `in_inv_bal` / `in_inv_bal_det`

### Created when a sales invoice gets payment terms

On invoice save/post (`in_inv_add_terms`, `in_inv_add_terms_manual`):

1. **`in_inv_terms`** — installments (due date, `pay_kind`, percentage).
2. **`in_inv_bal_det`** — one row per installment: `amount` = due, `paid = 0`, `costcenter`, `currency`, `aux`.
3. **`CALL in_inv_add_payments(doc_id, doc_type, aux, is_com)`** — rebuilds `paid` from existing settlements.

### Balance (generated column)

On both **`in_inv_bal`** and **`in_inv_bal_det`**:

```text
balance = amount - amount_ret - paid - amount_cn
```

### Reporting helpers

- **`in_inv_bal_amounts(doc_id, doc_type, type, aux, is_com)`** — e.g. type **4** = original document amount for UI on `ac_doc_det` (`AcDocDet::search`).

---

## 3. Linking a receipt to open sales invoices

### 3.1 Manual / UI

Receipt **Allocated documents** grid: `AcDocDet` with `doc_type = 2`, `doc_id = receipt`.  
Join: `inv_det_id` → `in_inv_bal_det.det_id` (optional pointer to a specific installment).

### 3.2 `api_create_receipt_alloc_docs`

Parameters: `(p_receipt, p_invoice, p_amount [, p_doc_type])`.

1. Resolve invoice header: sales **`doc_type` 10 or 11** from `in_sales.is_return` (or **`149`** / statement doc from `in_inv_bal_st` when not 10/11).
2. Client from `ac_receipt`; AR **`aux_id` / `account_id`** from `g_aux_account` (`acc_type = 1`).
3. **`inv_det_id`** = `MIN(det_id)` from `in_inv_bal_det` where `doc_type` / `doc_id` = invoice and **`is_com = 0`** (first installment row unless UI overrides).
4. **Insert `ac_doc_det`:**

   | Column | Value |
   |--------|--------|
   | `doc_type` | **2** |
   | `doc_id` | `p_receipt` |
   | `bal_doc_id` | invoice id |
   | `bal_doc_type` | 10 / 11 / … |
   | `amount` | `p_amount` |
   | `inv_det_id` | chosen installment |

### 3.3 `api_create_receipt_posting`

Builds **`ac_receiptdetail`** from summed **`ac_doc_det`** for the receipt (grouped by `aux_id`, currency, cost center, `account_id`).

Then reconciles to cash:

- `v_total_cash` = sum of `ac_receiptcash` (FX to receipt currency, signed by `in_out`).
- `v_total` = sum of `ac_receiptdetail`.
- If **`v_total_cash > v_total`**, **increase last `ac_receiptdetail.amount`** by the difference so AR credit = cash received.

Sets **`posted = 1`** on the header (API path); monolith UI may still use the posted flip to run **`ac_post_receipt`**.

### 3.4 Applying cash to installments — `in_inv_add_payments`

Called when invoice terms are rebuilt and when settlements change (DB hooks on `ac_doc_det` in production).

For invoice `(p_doc_id, p_doc_type, p_aux, p_is_com)`:

1. Reset `paid` / `amount_ret` on matching **`in_inv_bal_det`** rows; drop zero-amount rows.
2. Cursor over **`ac_doc_det`** where **`bal_doc_id` / `bal_doc_type`** = that invoice, `aux`, `is_ignore = 0`, ordered (returns 11/13 first when applicable).
3. Amount signed with **`in_inv_ac_doc_det_sign(doc_type, bal_doc_type, is_com)`** (receipt **2** → sales **10**: sign **+1**).
4. For each row: **`in_inv_add_paid_amount`** — walk installments **`ORDER BY thedate`**, increase **`paid`** until amount consumed.

**Note:** **`inv_det_id`** on `ac_doc_det` is primarily for UI / targeting; FIFO application is by **`in_inv_bal_det.thedate`** in **`in_inv_add_paid_amount`**.

### 3.5 Auto-settlement helpers

- **`in_settle_sales_invoice_func`** — creates receipt + optional `ac_doc_det` + `ac_receiptdetail` when `ac_auto_settlement <> 2`.
- UI: `ReceiptController::actionAutoSettleDocument` uses views `vw_settle_doc_client` / account-balance mode when `ac_auto_settlement = 2`.

---

## 4. Overpayment and unallocated advance

### 4.1 Cash greater than allocated invoices (customer credit on AR)

Allowed by design:

- **`ac_check_total_detail`**: **Σ `ac_doc_det.amount` ≤ Σ `ac_receiptdetail.amount`** (cannot over-allocate to invoices).
- **`api_create_receipt_posting`** / **`api_create_receipt_detail`**: bumps **`ac_receiptdetail`** so it matches **total cash**.

**Effect:** GL credits **full cash** to AR; only the allocated portion increases **`in_inv_bal_det.paid`**. The remainder is **unapplied customer credit** on the AR account until further `ac_doc_det` or auto-settle.

### 4.2 `in_inv_add_doc_acc_bal` (receipt as open document)

For **`doc_type = 2`** on the receipt itself: if header **`in_inv_bal`** is fully open (`amount = balance`), rebuilds **`in_inv_bal_det`** on the **receipt** for  
`(receipt detail total − Σ ac_doc_det allocations)` per currency — tracks **unapplied amount** on the receipt document in the subledger.

### 4.3 Paying more than invoice open balance

**`in_inv_add_paid_amount`**: after FIFO application, if **`v_rem ≠ 0`**, inserts **`in_inv_bal_det`** with **`amount = 0`**, **`paid = v_rem`** → negative **`balance`** on that bucket (invoice-level **credit / prepayment** in the subledger).

### 4.4 Negative receipt detail

**`ac_post_receipt`**: negative **`ac_receiptdetail.amount`** posts as **Debit** AR (refund). Blocked when **`downpay = 1`**.

---

## 5. Validation

### `ac_check_total_detail(p_doc_id, 2)`

Fails if, for any `(aux_id, currency, costcenter)` group:

**Σ `ac_doc_det.amount` > Σ `ac_receiptdetail.amount`**

Message: allocated documents cannot exceed the receipt detail amount for that client account.

Also validates **sub-account** when `g_aux.is_sub_acc = 1`.

### `in_val_receipt_dp(p_receipt)` (down payment)

Receipt total (FX to detail currency) must be **≥** sum of **`ac_receiptorders.amount`**.

---

## 6. GL posting — `ac_post_receipt`

**`doc_type` on `ac_trans` = 2**.  
Unpost: `p_posted = 0` → `DELETE FROM ac_trans WHERE doc_id = p_receipt AND doc_type = 2`.

### 6.1 Credit — client AR (`ac_receiptdetail`)

Grouped cursor: `account_id`, `aux_id`, `currency`, `costcenter`, `p_division`, `sub_acc`.

| Line amount | D/C | Notes |
|-------------|-----|--------|
| **> 0** | **Credit** (`v_dc_det = 'C'`) | Reduces AR |
| **< 0** | **Debit** (amount sign-flipped) | Refund; not allowed if `downpay = 1` |

FX: `rate_1` / `rate_2` on detail or `g_get_exch_rate`.

**Down-payment VAT** (`downpay = 1`, client `apply_vat = 1`, `use_dp = 2`):

On each positive detail line, extract VAT from gross using max `vat_pc` from **`in_vat_det`**:

| Account | D/C | Amount |
|---------|-----|--------|
| `ac_account_type` **191** (+ `g_get_aux_vat_acc`) | **Debit** | VAT portion |
| `ac_account_type` **190** (+ aux VAT) | **Credit** | Same VAT |

If `g_get_aux_vat_acc` fails, posting aborts (`LEAVE SWL_return`).

### 6.2 Debit / credit — settlements (`ac_receiptcash`)

Per line: account from **`g_settlement_kind`** for `settlement`; description from **`ac_post_receipt_description`**.

| `in_out` | Cash line D/C | Running total |
|----------|---------------|---------------|
| **1** (money in) | **Debit** | Adds to payment total |
| **0** (money out) | **Credit** | Subtracts |

**Amount:** usually **`amount - charges`** on the bank line unless **`g_bank_account.post_charges = 1`** (then full amount on bank + opposite charge line).

**Bank charges** (`charges > 0`):

- Debit **`g_bank_account.charges_acc`** for charges.
- Charge FX reduces effective detail total used in balancing.

**Pending checks** (`pend_rec_acc_client = 1`, settlement `kind = 2`):

- May post to client **`acc_type = 230`** with auxiliary instead of bank until configured.

### 6.3 Balancing

**`ac_balance_doc`**: compares cash side vs detail side (`v_amount_tot_p` vs `v_amount_tot_d`, including FX).  
Single-currency: **`ac_balance_trans_cur`** adjusts `amount_1` / `amount_2` on a line; then **`ac_is_doc_balanced`**.

### 6.4 Minimal double-entry (single currency, no charges, no DP VAT)

| # | Account | D/C | Amount |
|---|---------|-----|--------|
| 1 | Bank / cash (`g_settlement_kind`) | **D** | Cash received |
| 2 | Client AR (`ac_receiptdetail`, `acc_type 1`) | **C** | Same |

**`ac_doc_det`** does not add GL lines; it updates **`in_inv_bal_det.paid`** only.

### 6.5 Mental model vs sales invoice

| | Sales invoice (10) | Cash receipt (2) |
|--|-------------------|------------------|
| AR | **Debit** (customer owes) | **Credit** (customer pays) |
| Bank | — | **Debit** (cash in) |
| Invoice open balance | `in_inv_bal_det` created on invoice | `paid` increased via `ac_doc_det` |

---

## 7. TypeScript rewrite — suggested modules

```
cash-receipt/
  settlement/
    buildCashLines(acReceiptcash[])           // in_out, charges, settlement kind
    reconcileDetailToCash(detail, cash, date) // api_create_receipt_posting rule
  allocation/
    linkToInvoice(receipt, invoice, amount)   // ac_doc_det + inv_det_id
    applyPaymentsToSchedule(invoice, allocs)  // in_inv_add_paid_amount FIFO
  validation/
    checkAllocationVsDetail(receipt)          // ac_check_total_detail
    validateDownPaymentOrders(receipt)        // in_val_receipt_dp
  posting/
    buildJournalEntries(receipt, cash, detail, options)  // ac_post_receipt
    resolveSettlementAccount(settlement, options)
```

### Invariants to preserve

1. **Σ allocate ≤ Σ AR detail**; **Σ AR detail can equal Σ cash** even if allocate is lower.
2. **GL** = cash lines vs AR lines only; **subledger** = `ac_doc_det` → `in_inv_bal_det.paid`.
3. Installment application: **FIFO by `thedate`** unless product explicitly implements `inv_det_id`-only application.
4. Options: `ac_auto_settlement`, `pend_rec_acc_client`, `use_dp`, `post_charges` on bank accounts, `ac_tolerance_per` for FX imbalance.

---

## 8. Key code references (monolith)

| Area | Path |
|------|------|
| Commit + post flip | `wizard/protected/modules/receivable/models/Receipt.php` (~454–488) |
| Trip / batch API receipts | `wizard/protected/modules/receivable/controllers/ReceiptController.php` (`actionCreateReceiptsFromTrips`) |
| Allocation grid | `wizard/protected/modules/receivable/views/receipt/tabs/_docdet.php` |
| `AcDocDet` model | `wizard/protected/modules/receivable/models/AcDocDet.php` |
| iSell allocation | `wizard/protected/modules/iSellIntegration/controllers/ISellIntegrationController.php` |
| Auto-settle UI | `wizard/protected/modules/receivable/views/common/views/_settle_balance_detail.php` |

### Database routines

| Routine | Role |
|---------|------|
| `ac_post_receipt` | GL posting |
| `ac_post_receipt_description` | `ac_trans` descriptions |
| `ac_balance_doc` / `ac_balance_trans_cur` | FX / single-currency balance |
| `ac_check_total_detail` | Allocate vs detail |
| `api_create_receipt_head` | New `ac_receipt` |
| `api_create_receipt_settlement` | `ac_receiptcash` line |
| `api_create_receipt_alloc_docs` | `ac_doc_det` + `inv_det_id` |
| `api_create_receipt_posting` | `ac_receiptdetail` from allocations + cash top-up |
| `api_create_receipt_detail` | Alternate detail insert + top-up |
| `in_inv_add_payments` | Rebuild `paid` from `ac_doc_det` |
| `in_inv_add_paid_amount` | FIFO installment application |
| `in_inv_add_doc_acc_bal` | Unapplied amount on receipt document |
| `in_inv_ac_doc_det_sign` | Sign of settlement vs open doc |
| `in_val_receipt_dp` | Down-payment vs orders |
| `in_settle_sales_invoice_func` | One-shot receipt + allocate |

---

## 9. Related document types

| `doc_type` | Document | Mirror of receipt? |
|------------|----------|-------------------|
| **3** | Supplier payment | Yes — see [Payment posting](./payment-posting.md) |
| **4** / **5** | Debit / credit note | Allocate via `ac_doc_det`; different posting procs |
| **25** / **26** | Client bills | `ac_check_total_detail` cases 25/26 |

This document focuses on **2** client cash receipts against **sales invoices (10/11)**.

---

*Last consolidated from codebase + MySQL routine definitions. Update this file when posting options or procedures change.*
