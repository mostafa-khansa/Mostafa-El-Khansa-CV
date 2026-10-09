# Master Domain Guide: The Architecture of B2B Accounting & ERP Engines

This guide provides a comprehensive, end-to-end blueprint of how enterprise-grade accounting and ERP software works from a pure business and domain perspective. It synthesizes foundational double-entry principles with the operational mechanics of commercial documents, subledger aging schedules, multi-tier tax compliance, financial statement generation, and volatile multi-currency exchange management (such as USD/LBP parallel rate structures).

---

## Pillar 1: The Foundations of the General Ledger & Chart of Accounts

At the heart of every accounting system is the **General Ledger (GL)**, governed by the immutable equation of double-entry bookkeeping:
$$\text{Total Debits} = \text{Total Credits}$$

### 1.1 The Chart of Accounts (COA) Structure
The Chart of Accounts is the master catalog classifying every financial event in an enterprise. Accounts are structured hierarchically and numerically into five primary financial classes:
1. **1000 series (Assets):** Resources owned by the company (Cash, Bank accounts, Accounts Receivable, Inventory, Equipment). *Normal balance: Debit.*
2. **2000 series (Liabilities):** Obligations owed to external parties (Accounts Payable, Accrued Expenses, Output VAT Payable). *Normal balance: Credit.*
3. **3000 series (Equity):** Residual interest in assets after deducting liabilities (Retained Earnings, Share Capital). *Normal balance: Credit.*
4. **4000 series (Revenue / Income):** Inflows from selling goods or services. *Normal balance: Credit.*
5. **5000 series (Expenses / Costs):** Outflows or costs incurred to generate revenue (Cost of Goods Sold, Salaries, Rent, Realized Exchange Losses). *Normal balance: Debit.*

### 1.2 The Immutable Journal Table (`ac_trans`)
Every financial transaction in the system ultimately resolves into rows inside the general transaction ledger table (`ac_trans`). A posted document cannot be arbitrarily deleted; instead, it posts balanced lines where:
$$\sum \text{Debits} = \sum \text{Credits}$$
Each row records the account ID, foreign currency amount, parallel base currency amounts, exchange rates, and analytical dimensions (such as cost centers or branches).

---

## Pillar 2: Commercial Operations & Subledger Schedules

Operational documents (Sales, Purchases, Receipts, and Payments) do not interact with the general ledger alone. They bridge the operational world with financial control through **Subledgers** and **Aging Schedules**.

### 2.1 Sales Invoices (Doc Type 10)
* **Business Intent:** You sell goods or services to a customer on credit or cash.
* **Subledger Impact (`in_inv_bal_det`):** Creates open installment schedules tracked under **Accounts Receivable (AR)**, mapping invoice items, gross amounts, and due dates.
* **General Ledger Impact (`ac_trans`):**
  * **Debit:** Accounts Receivable (Total invoice value including tax)
  * **Credit:** Sales Revenue (Net taxable base)
  * **Credit:** Output VAT Liability (Tax collected)
  * *Perpetual Inventory (if applicable):* Debit COGS, Credit Inventory.

### 2.2 Purchase Bills (Doc Type 12)
* **Business Intent:** You purchase inventory, raw materials, or operating expenses from a supplier.
* **Subledger Impact (`in_inv_bal_det`):** Creates open payable installments tracked under **Accounts Payable (AP)**.
* **General Ledger Impact (`ac_trans`):**
  * **Debit:** Inventory / Expense / Asset account (Net taxable base)
  * **Debit:** Input VAT Recoverable (Recoverable tax asset)
  * **Credit:** Accounts Payable (Total bill value including tax)

### 2.3 Cash Receipts (Doc Type 2)
* **Business Intent:** A customer settles their open invoices via cash, check, or bank wire.
* **Subledger Impact (`ac_doc_det`):** Allocates the payment against open sales installments in FIFO order, updating the `paid` column in `in_inv_bal_det`.
* **General Ledger Impact (`ac_trans`):**
  * **Debit:** Bank / Cash settlement account
  * **Credit:** Accounts Receivable (Clearing customer debt)

### 2.4 Supplier Payments (Doc Type 3)
* **Business Intent:** You pay an open supplier bill via bank transfer, check, or cash.
* **Subledger Impact (`ac_doc_det`):** Allocates the payment against open purchase installments, updating `in_inv_bal_det.paid`.
* **General Ledger Impact (`ac_trans`):**
  * **Debit:** Accounts Payable (Clearing supplier debt)
  * **Credit:** Bank / Cash settlement account

---

## Pillar 3: Multi-Tier VAT Compliance & Precision Rules

Value Added Tax (VAT) requires absolute mathematical rigor, multi-rate tracking, and strict reconciliation between tax registers and the general ledger.

### 3.1 Line-Level Precision and Rounding
* **Calculation:** Line tax is computed on the taxable base (gross line value minus allocated document discounts):
  $$\text{VAT} = (\text{Gross Amount} - \text{Discount}) \times \frac{\text{VAT Rate}}{100}$$
* **4-Decimal Precision:** To prevent cumulative rounding drift across multi-line invoices, individual line VAT amounts are computed and stored with 4 decimal places before rolling up to document totals. Final document headers and GL entries round to standard currency precision.

### 3.2 The VAT Register (`in_doc_vat`)
When an invoice or bill is saved, the system recalculates and populates an aggregate table (`in_doc_vat`), grouping transactions by VAT Category and Rate Percentage. This stores:
* Taxable Base Amounts
* Tax Amounts
This register acts as the primary audit trail for government tax filings.

### 3.3 Special VAT Scenarios
* **Exports and Exemptions:** Zero-rated or exempt transactions force $\text{VAT Rate} = 0\%$ and route revenue to specialized export accounts.
* **Down Payments & Prepayments:** When down payments carry tax implications, specialized provisional tax accounts (e.g., unearned/advance VAT accounts) ensure compliance prior to final invoice issuance.

---

## Pillar 4: Multi-Currency & Parallel Exchange Rate Management

In multi-currency environments—particularly economies dealing with dual or volatile rates (such as official vs. market rates or USD/LBP conversions)—managing exchange rate drift is critical to keeping the trial balance intact.

### 4.1 Dual-Amount Storage Architecture
Transactions maintain parallel valuation streams:
* **Transaction Currency Amount:** The native currency of the deal (e.g., USD).
* **Base Currency Amounts (`amount_1`, `amount_2`):** Converted local or reporting currency values using dedicated exchange rate factors (`rate_1`, `rate_2`) captured at the exact date of the transaction via exchange rate lookups (`g_get_exch_rate`).

### 4.2 Realized Exchange Gains and Losses
When an invoice issued at Rate $A$ is settled by a payment recorded at Rate $B$, a variance occurs between the recorded AR credit and the cash debit in base currency terms. 
* To resolve this, the settlement engine automatically generates a balancing line routed to a **Realized Exchange Gain / Loss** income or expense account.
* This ensures that $\sum \text{Debits} = \sum \text{Credits}$ is maintained down to the exact subledger allocation level.

---

## Pillar 5: Financial Statements & Reporting Engines

Financial statements are not complex business logic engines; rather, they are structured analytical queries operating directly on the immutable ledger (`ac_trans`) and subledger tables.

### 5.1 The Trial Balance
* **Definition:** A diagnostic report listing all active COA accounts with their cumulative Debit and Credit sums over a given period.
* **Validation Rule:** Total System Debits must equal Total System Credits. If the trial balance is out of balance, ledger corruption or incomplete postings have occurred.

### 5.2 The Income Statement (Profit & Loss)
* **Definition:** Measures financial performance over a specific period by aggregating all **Revenue (4000 series)** and **Expense (5000 series)** accounts.
* **Formula:** 
  $$\text{Net Income} = \sum \text{Revenues} - \sum \text{Expenses}$$

### 5.3 The Balance Sheet
* **Definition:** A snapshot of financial position at a specific point in time, aggregating **Assets (1000 series)**, **Liabilities (2000 series)**, and **Equity (3000 series)**.
* **Formula:**
  $$\text{Assets} = \text{Liabilities} + \text{Equity}$$
  *(Note: Net Income from the Income Statement flows directly into Retained Earnings under Equity at period-end closure).*

---

## Summary Checklist for Domain Implementation
1. **Enforce Double-Entry Integrity:** Never allow a document to post unless its generated GL lines balance perfectly.
2. **Isolate Subledgers from GL:** Maintain AR/AP subledger schedules (`in_inv_bal_det`) for aging and customer/supplier tracking independently of aggregate GL control accounts.
3. **Maintain Tax Register Sync:** Ensure every invoice line item updates the `in_doc_vat` bucket table to match posted GL tax accounts.
4. **Capture FX Variances Automatically:** Compute settlement exchange differentials on the fly during payment allocation to prevent balance sheet drift.