# Supplier Payment (doc_type 3) — Domain Reference

> **Purpose:** Reverse-engineered business rules for supplier payments: settlement lines, purchase bill allocation, AP subledger (`in_inv_bal_det`), and GL posting. Intended for a clean Node.js/TypeScript service layer — **not** a copy of legacy triggers/procedures.
>
> **Sources:** Yii monolith (`wizard/protected/modules/receivable`, `sales`), MySQL routines on `wizard` DB (`ac_post_payment`, `api_create_payment_*`, `in_inv_add_payments`, `in_inv_add_paid_amount`, `ac_check_total_detail`, `in_inv_add_doc_acc_bal`, `in_val_payment_dp`, `in_settle_pur_invoice_func`).
>
> **Companions:** [Purchase bill posting](./purchase-invoice-posting.md) (creates AP and `in_inv_bal_det` installments); [Cash receipt posting](./cash-receipt-posting.md) (client-side mirror with `doc_type` **2** and `ac_post_receipt`).

---

## 1. Document identity and tables

| Concept | Value |
|--------|--------|
| Document type | **3** = Supplier Payment |
| Header | `ac_payment` (`aux`, `aux_type`, `downpay`, `posted`, `costcenter`, `payment_req`, …) |
| Money out / settlement | `ac_paymentcash` (per line: `settlement`, `amount`, **`out_in`**, check/bank/card fields, `charges`) |
| GL application (AP) | `ac_paymentdetail` (`aux_id`, `account_id`, `amount`, `amount_dp`, `rate_1`, `rate_2`, `sub_acc`, …) |
| Bill allocation | `ac_doc_det` (`doc_type = 3`, `doc_id = payment`, `bal_doc_type` **12** / **13** / …, `bal_doc_id`, `amount`, **`inv_det_id`**) |
| Open AP schedule | `in_inv_bal`, `in_inv_bal_det` (per purchase bill and installment) |
| GL lines | `ac_trans` (`doc_type = 3`, via `ac_trans_add`) |
| Supplier AP link | `g_aux_account` — typical detail lines use **`acc_type = 2`** (goods), **8** (expense), or **9** (asset), matching [purchase invoice AP](./purchase-invoice-posting.md#62-accounts-payable-supplier) |

Settlement kinds for payments are scoped by **`g_settlement_kind.doc_type = 3`** (see `ReceivableHelper.php`).

### Three layers (do not conflate)

| Layer | What it answers |
|-------|------------------|
| **`ac_paymentcash`** | How did money leave? (bank transfer, check, cash, card, …) |
| **`ac_paymentdetail`** | Which **GL AP** account is debited (supplier balance in the ledger)? |
| **`ac_doc_det`** | Which **open purchase bills** (subledger) does this payment settle? |

**`ac_post_payment`** posts only **cash + detail** to `ac_trans`. **`ac_doc_det`** drives **`in_inv_bal_det.paid`** via **`in_inv_add_payments`** (see §4) — no extra GL lines per allocated bill.

### Save / validate / post pipeline (application)

From `Payment.php` commit:

1. User enters **`ac_paymentcash`** and **`ac_paymentdetail`**, and optionally **`ac_doc_det`** (allocated documents tab).
2. `CALL g_get_aux_active(:aux, :aux_type, …)`
3. `CALL ac_check_total_detail(:payment, 3)` — allocation cannot exceed AP detail (§5).
4. Post to GL:
   - `UPDATE ac_payment SET posted = 0 WHERE payment = :id`
   - `UPDATE ac_payment SET posted = 1 WHERE payment = :id`
   - Intended to invoke **`ac_post_payment`** (logic in DB procedures; triggers may not appear in `information_schema` on all environments).
5. If `downpay = 1`: `CALL in_val_payment_dp(:payment)` — payment detail total must cover linked purchase orders (§5.2).

API / integration routines exist as **`api_create_payment_head`**, **`api_create_payment_settlement`**, **`api_create_payment_alloc_docs`**, **`api_create_payment_posting`** — see §3.3 for differences vs the receipt API.

---

## 2. AP subledger — `in_inv_bal` / `in_inv_bal_det`

### Created when a purchase bill gets payment terms

On purchase save (`in_inv_add_terms_manual` from `Purchase.php` for doc_type **12** / **13**):

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

## 3. Linking a payment to open purchase bills

### 3.1 Manual / UI

Payment **Allocated documents** grid: `AcDocDet` with `doc_type = 3`, `doc_id = payment`.  
Typical targets: **`bal_doc_type` 12** (purchase bill), **13** (return purchase).  
Join: `inv_det_id` → `in_inv_bal_det.det_id` (optional pointer to a specific installment).

### 3.2 `api_create_payment_alloc_docs`

Parameters: `(p_payment, p_invoice, p_amount [, p_doc_type])`.

**Note:** The shipped procedure body in some environments still references sales return (`bal_doc_type` 11) and client `acc_type = 1`; production UI and manual `ac_doc_det` inserts use **purchase** types **12 / 13** and supplier AP `aux_id`. Prefer monolith / explicit inserts for purchase allocation when integrating.

For a correct purchase allocation row:

| Column | Value |
|--------|--------|
| `doc_type` | **3** |
| `doc_id` | `p_payment` |
| `bal_doc_id` | `in_purchases.invoice` |
| `bal_doc_type` | **12** or **13** |
| `amount` | allocated amount |
| `aux` / `aux_id` | supplier from `ac_payment` + matching `g_aux_account` AP row |
| `inv_det_id` | optional; often `MIN(det_id)` from `in_inv_bal_det` for that bill |

### 3.3 `api_create_payment_posting`

Builds **`ac_paymentdetail`** from summed **`ac_doc_det`** for the payment (grouped by `aux_id`, currency, cost center, `account_id` via join on `g_aux_account`).

Unlike **`api_create_receipt_posting`**, this routine does **not** reconcile detail to total cash — it only inserts detail from allocations and sets **`posted = 1`**. The UI must keep **Σ `ac_paymentdetail` = Σ settlement cash** (enforced at GL post via **`ac_balance_doc`** / **`ac_is_doc_balanced`**).

### 3.4 Applying cash to installments — `in_inv_add_payments`

Shared with receipts; called when bill terms are rebuilt and when settlements change (DB hooks on `ac_doc_det` in production).

For bill `(p_doc_id, p_doc_type, p_aux, p_is_com)`:

1. Reset `paid` / `amount_ret` on matching **`in_inv_bal_det`** rows; drop zero-amount rows.
2. Cursor over **`ac_doc_det`** where **`bal_doc_id` / `bal_doc_type`** = that bill, `aux`, `is_ignore = 0`, ordered (return settlement doc types **11** / **13** sort first when applicable).
3. Amount signed with **`in_inv_ac_doc_det_sign(doc_type, bal_doc_type, is_com)`**:
   - Payment **3** → purchase bill **12**: sign **+1** (increase `paid`).
   - Payment **3** → purchase return **13**: sign **-1**.
   - Payment **3** → sales invoice **10** (unusual cross-link): sign **-1** per function rules.
4. For each row: **`in_inv_add_paid_amount`** — walk installments **`ORDER BY thedate`**, adjust **`paid`** until amount consumed.

**Note:** **`inv_det_id`** on `ac_doc_det` is primarily for UI / targeting; FIFO application is by **`in_inv_bal_det.thedate`** in **`in_inv_add_paid_amount`**.

### 3.5 Auto-settlement helpers

- **`in_settle_pur_invoice_func`** — creates payment + optional `ac_doc_det` + detail from a purchase bill (`Purchase.php`).
- UI: `PaymentController` / shared settle views use **`vw_settle_doc_supplier`** when **`ac_auto_settlement = 2`** (account-balance mode); otherwise document-level settle grids.

---

## 4. Overpayment and unallocated advance

### 4.1 Cash greater than allocated bills (supplier credit on AP)

Allowed by design:

- **`ac_check_total_detail`**: **Σ `ac_doc_det.amount` ≤ Σ `ac_paymentdetail.amount`** (cannot over-allocate to bills).
- User / integration must align **Σ detail** with **Σ cash** before post (no automatic detail top-up in `api_create_payment_posting`).

**Effect:** GL debits **full AP detail**; only the allocated portion increases **`in_inv_bal_det.paid`** on bills. The remainder is **unapplied supplier prepayment** on the AP account until further allocation or auto-settle.

### 4.2 `in_inv_add_doc_acc_bal` (payment as open document)

For **`doc_type = 3`**: if header **`in_inv_bal`** is fully open (`amount = balance`), rebuilds **`in_inv_bal_det`** on the **payment** for  
`(payment detail total − Σ ac_doc_det allocations)` per currency — tracks **unapplied amount** on the payment document in the subledger.

### 4.3 Paying more than bill open balance

**`in_inv_add_paid_amount`**: after FIFO application, if **`v_rem ≠ 0`**, inserts **`in_inv_bal_det`** with **`amount = 0`**, **`paid = v_rem`** → negative **`balance`** on that bucket (bill-level **credit / prepayment** in the subledger).

### 4.4 Negative payment detail

**`ac_post_payment`**: negative **`ac_paymentdetail.amount`** posts as **Credit** AP (reversal / refund from supplier). Blocked when **`downpay = 1`**.

---

## 5. Validation

### `ac_check_total_detail(p_doc_id, 3)`

Fails if, for any `(aux_id, currency, costcenter)` group:

**Σ `ac_doc_det.amount` > Σ `ac_paymentdetail.amount`**

Message: allocated documents cannot exceed the payment detail amount for that supplier account.

Also validates **sub-account** when `g_aux.is_sub_acc = 1` (detail must have `sub_acc`).

### `in_val_payment_dp(p_payment)` (down payment)

Payment detail total (FX to first detail currency via `g_get_exch_rate`) must be **≥** sum of **`ac_paymentorders.amount`**.

### `ac_paymentcash_valid_checkno`

Validates check numbers on payment cash lines (called from cash line save hooks where configured).

---

## 6. GL posting — `ac_post_payment`

**`doc_type` on `ac_trans` = 3**.  
Unpost: `p_posted = 0` → `DELETE FROM ac_trans WHERE doc_id = p_payment AND doc_type = 3`.

Posting order: **AP detail lines first**, then **settlement cash**, then **`ac_balance_doc`** / **`ac_is_doc_balanced`**.

HR-linked payments (`hr_loan` / `hr_advance` referencing the payment) use alternate **`g_get_exch_rate`** parameters on detail and cash FX.

### 6.1 Debit — supplier AP (`ac_paymentdetail`)

Cursor: `account_id`, `aux_id`, `currency`, `costcenter`, `amount`, `rate_1`, `rate_2`, `p_division`, `sub_acc`.

| Line amount | D/C | Notes |
|-------------|-----|--------|
| **> 0** | **Debit** (`v_dc_det = 'D'`) | Reduces AP |
| **< 0** | **Credit** (amount sign-flipped) | Reversal; not allowed if `downpay = 1` |

FX: if `rate_1 = 0`, `amount_1` / `amount_2` via **`g_get_exch_rate`** (or HR rate mode); else `amount × rate_1` / `rate_2` rounded to base currency precision.

**Down-payment VAT** (`downpay = 1`, supplier `apply_vat = 1`, **`use_dp_py = 2`**):

On each positive detail line, extract VAT from gross using max `vat_pc` from **`in_vat_det`**:

| Account | D/C | Amount |
|---------|-----|--------|
| `ac_account_type` **193** (+ `g_get_aux_vat_acc`) | **Credit** | VAT portion |
| `ac_account_type` **192** (+ aux VAT) | **Debit** | Same VAT |

If `g_get_aux_vat_acc` fails, posting aborts (`LEAVE SWL_return`).

### 6.2 Credit / debit — settlements (`ac_paymentcash`)

Per line: account from **`g_settlement_kind`** for `settlement` (column **`out_in`** on cash); description from **`ac_post_payment_description`**.

Settlement `kind` is remapped for descriptions (e.g. check kind **2** → internal kind **6**).

| `out_in` | Cash line D/C | Running total |
|----------|---------------|---------------|
| **1** (money out) | **Credit** | Adds to payment-out total |
| **0** (money in) | **Debit** | Subtracts (reversal leg) |

**Amount:** if **`g_bank_account.post_charges = 0`**, bank line uses **`amount + charges`** in one credit; if **`post_charges = 1`**, bank credit is **`amount`** and charges post as a separate credit on the bank account.

**Bank charges** (`charges > 0`):

- **Debit** **`g_bank_account.charges_acc`** for charges (`v_dc_2` opposite to main bank line when `out_in = 1`).
- Requires `charges_acc` configured or posting signals *Define the bank charges account*.

**Pending checks** (`pay_pend_payable = 1`, check settlement / kind **6** after remap):

- Replace bank GL account with **`g_bank_account.payable_acc`** (pending payable / checks in transit) until cleared — mirror of client **`pend_rec_acc_client`** on receipts.

Check fields (`check_no`, `maturity_date`, `bank_id`) flow to `ac_trans` on bank lines.

### 6.3 Balancing

**`ac_balance_doc`**: compares cash side vs detail side (`v_amount_tot_d − v_amount_tot_p` and FX legs).  
Single-currency: **`ac_balance_trans_cur`** adjusts `amount_1` / `amount_2` on a line; then **`ac_is_doc_balanced`**.

### 6.4 Minimal double-entry (single currency, no charges, no DP VAT, `out_in = 1`)

| # | Account | D/C | Amount |
|---|---------|-----|--------|
| 1 | Supplier AP (`ac_paymentdetail`) | **D** | Payment to supplier |
| 2 | Bank / cash (`g_settlement_kind` → `ac_paymentcash.account_id`) | **C** | Same |

**`ac_doc_det`** does not add GL lines; it updates **`in_inv_bal_det.paid`** only.

### 6.5 Mental model vs purchase bill

| | Purchase bill (12) | Supplier payment (3) |
|--|-------------------|------------------------|
| AP | **Credit** (supplier owed) | **Debit** (pay supplier) |
| Bank | — | **Credit** (cash out) |
| Bill open balance | `in_inv_bal_det` created on bill | `paid` increased via `ac_doc_det` |

### 6.6 Mirror vs cash receipt

| | Cash receipt (2) | Supplier payment (3) |
|--|------------------|----------------------|
| Subledger docs | Sales **10** / **11** | Purchase **12** / **13** |
| Balance sheet | **Credit** AR (`acc_type` 1) | **Debit** AP (2 / 8 / 9) |
| Bank | **Debit** (cash in, `in_out = 1`) | **Credit** (cash out, `out_in = 1`) |
| Pending instrument | `pend_rec_acc_client` | `pay_pend_payable` |
| DP VAT accounts | **191** D / **190** C | **193** C / **192** D |
| DP option | `use_dp` | `use_dp_py` |

---

## 7. TypeScript rewrite — suggested modules

```
supplier-payment/
  settlement/
    buildCashLines(acPaymentcash[])            // out_in, charges, settlement kind (doc_type 3)
    reconcileDetailToCash(detail, cash, date)  // UI responsibility; optional service helper
  allocation/
    linkToBill(payment, invoice, amount)       // ac_doc_det + inv_det_id, bal_doc_type 12/13
    applyPaymentsToSchedule(bill, allocs)        // in_inv_add_paid_amount FIFO
  validation/
    checkAllocationVsDetail(payment)           // ac_check_total_detail case 3
    validateDownPaymentOrders(payment)         // in_val_payment_dp
  posting/
    buildJournalEntries(payment, cash, detail, options)  // ac_post_payment
    resolveSettlementAccount(settlement, bankId, options)   // pay_pend_payable, charges
```

### Invariants to preserve

1. **Σ allocate ≤ Σ AP detail**; **Σ AP detail must match Σ cash** at post (detail may exceed allocate).
2. **GL** = cash lines vs AP lines only; **subledger** = `ac_doc_det` → `in_inv_bal_det.paid`.
3. Installment application: **FIFO by `thedate`** unless product explicitly implements `inv_det_id`-only application.
4. Options: `ac_auto_settlement`, `pay_pend_payable`, `use_dp_py`, `post_charges` on bank accounts, `ac_tolerance_per` for FX imbalance.

---

## 8. Key code references (monolith)

| Area | Path |
|------|------|
| Commit + post flip | `wizard/protected/modules/receivable/models/Payment.php` (~435–499) |
| Payment UI / auto-settle | `wizard/protected/modules/receivable/controllers/PaymentController.php` |
| Down payment (PO link) | `wizard/protected/modules/receivable/controllers/DownpaymentController.php` |
| Allocation grid | `wizard/protected/modules/receivable/views/payment/tabs/_docdet.php` |
| `AcDocDet` model | `wizard/protected/modules/receivable/models/AcDocDet.php` |
| Settlement kinds (doc_type 3) | `wizard/protected/modules/receivable/components/ReceivableHelper.php` |
| Auto-settle from purchase | `wizard/protected/modules/sales/models/Purchase.php` (`in_settle_pur_invoice_func`) |
| Open bills grid | `wizard/protected/modules/receivable/models/SettleDocSupplier.php` (`vw_settle_doc_supplier`) |
| Shared settle UI | `wizard/protected/modules/receivable/views/common/views/_settle_balance_detail.php` |

### Database routines

| Routine | Role |
|---------|------|
| `ac_post_payment` | GL posting |
| `ac_post_payment_description` | `ac_trans` descriptions |
| `ac_balance_doc` / `ac_balance_trans_cur` | FX / single-currency balance |
| `ac_check_total_detail` | Allocate vs detail (case **3**) |
| `api_create_payment_head` | New `ac_payment` |
| `api_create_payment_settlement` | `ac_paymentcash` line |
| `api_create_payment_alloc_docs` | `ac_doc_det` (verify purchase vs sales targets) |
| `api_create_payment_posting` | `ac_paymentdetail` from allocations |
| `wapi_payment` / `wapi_paymentcash` / `wapi_paymentdetail` / `wapi_payment_docs` | API wrappers |
| `in_inv_add_payments` | Rebuild `paid` from `ac_doc_det` |
| `in_inv_add_paid_amount` | FIFO installment application |
| `in_inv_add_doc_acc_bal` | Unapplied amount on payment document |
| `in_inv_ac_doc_det_sign` | Sign of settlement vs open doc |
| `in_val_payment_dp` | Down-payment vs `ac_paymentorders` |
| `in_settle_pur_invoice_func` | One-shot payment + allocate from bill |
| `ac_paymentcash_valid_checkno` | Check number validation |

---

## 9. Related document types

| `doc_type` | Document | Allocate via `ac_doc_det`? |
|------------|----------|----------------------------|
| **2** | Cash receipt | Yes — mirror on AR / sales |
| **4** / **5** | Debit / credit note (supplier) | Yes — `ac_check_total_detail` cases 4/5 |
| **12** / **13** | Purchase bill / return | **Target** of payment allocation |
| **25** / **26** | Client bills | `ac_check_total_detail` cases 25/26 (not supplier payment focus) |

This document focuses on **3** supplier payments against **purchase bills (12/13)**.

---

*Last consolidated from codebase + MySQL routine definitions. Update this file when posting options or procedures change.*
