# GST Invoicing Microservice - Architecture & Flow

**Version:** 1.0
**Last Updated:** 2025-10-17
**Document Owner:** Product Team

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Database Schema](#database-schema)
3. [GST Calculation Logic](#gst-calculation-logic)
4. [User Flows](#user-flows)
5. [API Endpoints](#api-endpoints)
6. [Reconciliation Process](#reconciliation-process)

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         FRONTEND LAYER                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────┐    ┌─────────────────────────┐   │
│  │   Partner Portal        │    │   Admin Dashboard       │   │
│  │   (React/Vue/Next.js)   │    │   (React/Vue/Next.js)   │   │
│  │                         │    │                         │   │
│  │  - Invoice List         │    │  - Collect Invoices     │   │
│  │  - Generate Invoice     │    │  - Export Bank File     │   │
│  │  - E-sign (OTP)         │    │  - Upload Settlement    │   │
│  │  - View Status          │    │  - Reconciliation       │   │
│  └─────────────────────────┘    └─────────────────────────┘   │
│              │                              │                   │
└──────────────┼──────────────────────────────┼───────────────────┘
               │                              │
               │         HTTPS/REST           │
               │                              │
┌──────────────▼──────────────────────────────▼───────────────────┐
│                    GST INVOICING MICROSERVICE                   │
│                      (Node.js/Python/FastAPI)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    API LAYER                             │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │  Partner APIs:                                           │  │
│  │  • GET    /api/partner/:destination_id/invoices          │  │
│  │  • POST   /api/partner/:destination_id/invoice/generate  │  │
│  │  • POST   /api/partner/invoice/:invoice_id/request-otp   │  │
│  │  • POST   /api/partner/invoice/:invoice_id/verify-otp    │  │
│  │  • GET    /api/partner/invoice/:invoice_id/pdf           │  │
│  │                                                           │  │
│  │  Admin APIs:                                             │  │
│  │  • GET    /api/admin/invoices/submitted                  │  │
│  │  • POST   /api/admin/export-bank-file                    │  │
│  │  • POST   /api/admin/reconcile-payment                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   BUSINESS LOGIC                         │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │  • Invoice Service  - Calculate GST Base, Return Data    │  │
│  │  • OTP Service      - Generate, Send, Verify OTP         │  │
│  │  • Payment Service  - Collection, Reconciliation         │  │
│  │                                                           │  │
│  │  ✅ PDF Generation: Handled by Frontend HTML Template   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                         DATA LAYER                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐       │
│  │  PostgreSQL  │   │    Redis     │   │   AWS S3     │       │
│  │              │   │              │   │              │       │
│  │  • partners  │   │  • OTP Cache │   │  • Invoice   │       │
│  │  • payout_   │   │  • Sessions  │   │    PDFs      │       │
│  │    calc      │   │              │   │              │       │
│  │  • invoices  │   │              │   │              │       │
│  └──────────────┘   └──────────────┘   └──────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                    INTEGRATION LAYER                            │
├─────────────────────────────────────────────────────────────────┤
│  • Email Service (SendGrid/AWS SES) - OTP delivery             │
│  • SMS Service (Optional - Twilio)  - OTP delivery             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Database Schema

### **Existing Tables (Already in Your PostgreSQL)**

```sql
-- ✅ You already have these tables - NO CHANGES NEEDED

-- partners table (existing)
-- Stores partner master data
-- Key column: destination_id (UUID)

-- payout_calculation table (existing)
-- Stores payout calculation records from CSV
-- Key columns: destination_id, month, category, reason_for_payment, effective_payout_base
```

### **New Tables to Create**

### **1. Reason for Payment Mapping Table** (Configuration Only)

```sql
CREATE TABLE reason_payment_mapping (
    id SERIAL PRIMARY KEY,
    category VARCHAR(20) NOT NULL,  -- 'trail' or 'upfront'
    reason_for_payment VARCHAR(100) NOT NULL,

    -- GST Eligibility (ONLY reason_for_payment matters)
    is_eligible_for_gst BOOLEAN DEFAULT TRUE,

    -- Metadata for reference only
    payment_type VARCHAR(50),  -- 'COMMISSION' or 'OFFER' (informational only)
    description TEXT,
    is_active BOOLEAN DEFAULT TRUE,

    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),

    -- Constraints
    UNIQUE(category, reason_for_payment)
);

-- Create index
CREATE INDEX idx_category_reason ON reason_payment_mapping(category, reason_for_payment);

-- Insert mapping data based on your CSV
-- ✅ GST eligibility is based ONLY on reason_for_payment
-- ✅ TDS is NOT considered for GST calculation
INSERT INTO reason_payment_mapping
(category, reason_for_payment, is_eligible_for_gst, payment_type, description)
VALUES
('trail', 'recovery_payment', TRUE, 'COMMISSION', 'Trail commission on recovery'),
('trail', 'base_referral', TRUE, 'COMMISSION', 'Trail commission on base referral'),
('upfront', 'base_referral', TRUE, 'OFFER', 'Upfront payment - base referral'),
('upfront', 'dynamic', TRUE, 'OFFER', 'Upfront payment - dynamic'),
('upfront', 'negotiation_credit_base_referral', TRUE, 'OFFER', 'Negotiation credit base'),
('upfront', 'negotiation_partner_base_referral', TRUE, 'OFFER', 'Negotiation partner base'),
('upfront', 'negotiation_partner_marketing', TRUE, 'OFFER', 'Negotiation partner marketing');
```

### **2. GST Invoices Table** (Core Invoice Data)

```sql
CREATE TABLE invoices (
    invoice_id SERIAL PRIMARY KEY,

    -- Partner reference (links to your existing partners table)
    destination_id UUID NOT NULL,
    -- FOREIGN KEY (destination_id) REFERENCES partners(destination_id),

    -- Invoice details
    invoice_number VARCHAR(50) UNIQUE NOT NULL,
    invoice_date DATE NOT NULL,
    month VARCHAR(7) NOT NULL,  -- '2024-11'
    financial_year VARCHAR(10),  -- '2024-2025'

    -- GST calculation
    -- ✅ GST Base = Sum of eligible gross amounts (before TDS)
    gst_base_amount DECIMAL(15,2) NOT NULL,
    trail_amount DECIMAL(15,2) DEFAULT 0,
    upfront_amount DECIMAL(15,2) DEFAULT 0,

    -- E-signature tracking
    status VARCHAR(50) DEFAULT 'not_generated',
    -- Status values: 'not_generated', 'generated', 'submitted', 'in_bank_file', 'paid', 'failed'

    digitally_signed BOOLEAN DEFAULT FALSE,
    signed_at TIMESTAMP,
    signed_ip VARCHAR(50),

    -- Payment reconciliation (KEY: invoice_id is used for matching)
    utr_number VARCHAR(50),
    payment_date DATE,
    payment_status VARCHAR(50),  -- 'SUCCESS', 'FAILED', 'PENDING'
    payment_failure_reason TEXT,

    -- Bank file tracking
    bank_file_id VARCHAR(100),

    -- Audit
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),

    -- Constraints
    UNIQUE(destination_id, month)  -- One invoice per partner per month
);

-- Indexes for performance
CREATE INDEX idx_invoices_destination ON invoices(destination_id);
CREATE INDEX idx_invoices_month ON invoices(month);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_invoices_invoice_number ON invoices(invoice_number);
CREATE INDEX idx_invoices_utr ON invoices(utr_number);
CREATE INDEX idx_invoices_dest_month ON invoices(destination_id, month);

-- Trigger to auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_invoices_updated_at
    BEFORE UPDATE ON invoices
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

**Note:** Backend only stores minimal invoice metadata. Frontend generates PDF using HTML template with data from:
- `invoices` table (invoice_number, invoice_date, gst_base_amount)
- `partners` table (partner_name, gst_number, account_number, ifsc_code, state)

### **5. OTP Log Table** (For Audit)

```sql
CREATE TABLE otp_log (
    otp_id SERIAL PRIMARY KEY,

    -- References
    invoice_id INT REFERENCES invoices(invoice_id),
    destination_id UUID REFERENCES partners(destination_id),

    -- OTP details
    otp_hash TEXT NOT NULL,  -- bcrypt hash
    otp_sent_at TIMESTAMP DEFAULT NOW(),
    otp_expires_at TIMESTAMP NOT NULL,

    -- Verification
    verification_attempts INT DEFAULT 0,
    max_attempts INT DEFAULT 3,
    is_verified BOOLEAN DEFAULT FALSE,
    verified_at TIMESTAMP,

    -- Security
    ip_address VARCHAR(50),
    user_agent TEXT,

    -- Indexes
    INDEX idx_invoice (invoice_id),
    INDEX idx_expires (otp_expires_at)
);
```

### **6. Bank File Log Table**

```sql
CREATE TABLE bank_file_log (
    bank_file_id SERIAL PRIMARY KEY,

    -- File details
    file_name VARCHAR(255) NOT NULL,
    file_path TEXT,
    month VARCHAR(7),

    -- Statistics
    total_invoices INT,
    total_amount DECIMAL(15,2),

    -- Status
    status VARCHAR(50) DEFAULT 'generated',  -- 'generated', 'sent_to_bank', 'reconciled'

    -- Timestamps
    generated_at TIMESTAMP DEFAULT NOW(),
    sent_to_bank_at TIMESTAMP,

    -- Indexes
    INDEX idx_month (month),
    INDEX idx_status (status)
);
```

---

## GST Calculation Logic

### **Key Principle**

✅ **GST Eligibility**: Determined ONLY by `reason_for_payment` (category is for grouping only)
✅ **GST Base Amount**: Sum of gross payout amounts **BEFORE TDS deduction**
✅ **TDS is NOT considered** in GST calculation

---

### **Step 1: Fetch Eligible Payout Records**

```javascript
// Query your existing payout_calculation table
// Join with reason_payment_mapping to filter eligible records
async function getEligiblePayouts(destinationId, month) {
  const query = `
    SELECT
      pc.category,
      pc.reason_for_payment,
      pc.effective_payout_base,
      pc.financial_year
    FROM payout_calculation pc
    INNER JOIN reason_payment_mapping rpm
      ON pc.category = rpm.category
      AND pc.reason_for_payment = rpm.reason_for_payment
    WHERE pc.destination_id = $1
      AND pc.month = $2
      AND rpm.is_eligible_for_gst = TRUE
      AND rpm.is_active = TRUE
  `;

  return await db.query(query, [destinationId, month]);
}
```

### **Step 2: Aggregate by Category (Trail vs Upfront)**

```javascript
async function aggregatePayoutsByCategory(destinationId, month) {
  const eligiblePayouts = await getEligiblePayouts(destinationId, month);

  if (eligiblePayouts.length === 0) {
    throw new Error('No eligible payouts found for GST invoice');
  }

  let trail = {
    gross: 0,      // Sum of gross amounts BEFORE TDS
    count: 0       // Number of trail payout records
  };

  let upfront = {
    gross: 0,      // Sum of gross amounts BEFORE TDS
    count: 0       // Number of upfront payout records
  };

  let financialYear = eligiblePayouts[0].financial_year;

  for (const payout of eligiblePayouts) {
    // ✅ Use effective_payout_base as gross amount (before TDS)
    const grossAmount = parseFloat(payout.effective_payout_base);

    if (payout.category === 'trail') {
      trail.gross += grossAmount;
      trail.count++;
    } else if (payout.category === 'upfront') {
      upfront.gross += grossAmount;
      upfront.count++;
    }
  }

  return {
    trail: trail,
    upfront: upfront,
    financialYear: financialYear
  };
}
```

### **Step 3: Calculate GST Base Amount**

```javascript
function calculateGSTBase(trail, upfront) {
  // ✅ GST Base = Direct sum of gross amounts (before TDS)
  // ✅ NO TDS calculation involved

  const trailBase = trail.gross;
  const upfrontBase = upfront.gross;
  const totalGSTBase = trailBase + upfrontBase;

  return {
    trailBase: trailBase,
    upfrontBase: upfrontBase,
    totalGSTBase: totalGSTBase
  };
}
```

### **Step 4: Calculate GST @ 18%**

```javascript
async function calculateGST(gstBase, destinationId) {
  const GST_RATE = 0.18;
  const gstAmount = gstBase * GST_RATE;

  // Fetch partner state from existing partners table
  const partner = await db.query(
    'SELECT state FROM partners WHERE destination_id = $1',
    [destinationId]
  );

  const companyState = 'Maharashtra';  // Your company's state
  const isSameState = partner[0].state === companyState;

  return {
    gstAmount: gstAmount,
    cgst: isSameState ? gstAmount / 2 : 0,  // 9%
    sgst: isSameState ? gstAmount / 2 : 0,  // 9%
    igst: isSameState ? 0 : gstAmount        // 18%
  };
}
```

### **Complete Calculation Function**

```javascript
async function calculateInvoiceAmounts(destinationId, month) {
  // Step 1 & 2: Aggregate eligible payouts by category
  const { trail, upfront, financialYear } = await aggregatePayoutsByCategory(
    destinationId,
    month
  );

  // Step 3: Calculate GST base (sum of gross amounts)
  const gstBase = calculateGSTBase(trail, upfront);

  // Step 4: Calculate GST @ 18%
  const gst = await calculateGST(gstBase.totalGSTBase, destinationId);

  return {
    trail: {
      gross: trail.gross,
      count: trail.count
    },
    upfront: {
      gross: upfront.gross,
      count: upfront.count
    },
    gstBase: gstBase.totalGSTBase,
    gst: gst,
    totalInvoiceAmount: gstBase.totalGSTBase + gst.gstAmount,
    financialYear: financialYear
  };
}
```

### **Example Calculation**

```
Partner: Rajendra Nalbalwar (destination_id: 75c6b6c6-4ff2-415b-b793-7ab2343e2c21)
Month: 2024-11

Step 1: Query eligible payouts from your database
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SELECT * FROM payout_calculation pc
INNER JOIN reason_payment_mapping rpm
  ON pc.category = rpm.category
  AND pc.reason_for_payment = rpm.reason_for_payment
WHERE pc.destination_id = '75c6b6c6-4ff2-415b-b793-7ab2343e2c21'
  AND pc.month = '2024-11'
  AND rpm.is_eligible_for_gst = TRUE;

Results:
  trail/recovery_payment:  effective_payout_base = ₹50,000
  trail/base_referral:     effective_payout_base = ₹30,000
  upfront/base_referral:   effective_payout_base = ₹40,000
  upfront/dynamic:         effective_payout_base = ₹10,000


Step 2: Aggregate by category
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Trail:
  - Gross = ₹50,000 + ₹30,000 = ₹80,000
  - Count = 2 records

Upfront:
  - Gross = ₹40,000 + ₹10,000 = ₹50,000
  - Count = 2 records


Step 3: Calculate GST Base
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ GST Base = Trail Gross + Upfront Gross
✅ GST Base = ₹80,000 + ₹50,000 = ₹1,30,000

(No TDS deduction involved)


Step 4: Calculate GST @ 18%
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Assuming partner is in same state (Maharashtra):

GST Amount = ₹1,30,000 × 18% = ₹23,400
CGST (9%) = ₹11,700
SGST (9%) = ₹11,700
IGST = ₹0


Final Invoice Breakdown:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Trail Payout:        ₹80,000
Upfront Payout:      ₹50,000
────────────────────────────
GST Base Amount:     ₹1,30,000
GST @ 18%:           ₹23,400
────────────────────────────
Total Invoice:       ₹1,53,400
```

---

## Frontend PDF Generation

### **Backend Provides Raw Data Only**

Backend API returns only the data needed for PDF generation:

```json
{
  "invoiceId": 12345,
  "invoiceNumber": "INV/NOV24/001",
  "invoiceDate": "2024-12-01",
  "month": "2024-11",

  // Partner details from partners table
  "partnerName": "Rajendra Nalbalwar",
  "gstNumber": "29AAPFA1234A1Z5",
  "accountNumber": "1234567890",
  "ifscCode": "HDFC0001234",
  "state": "Maharashtra",

  // Calculated amounts
  "gstBaseAmount": 130000.00,
  "trailAmount": 80000.00,
  "upfrontAmount": 50000.00
}
```

### **Frontend Handles:**

1. **CGST/SGST vs IGST Logic** - Based on partner state
2. **PDF Formatting** - Using HTML template
3. **Styling** - Invoice design and layout
4. **PDF Generation** - Convert HTML to PDF

### **Data Required for Your HTML Template:**

```javascript
// API call to get invoice data
GET /api/partner/:destination_id/invoice/:month/data

// Response contains all 7 required fields:
{
  partnerName: "Rajendra Nalbalwar",        // ✅ From partners table
  gstNumber: "29AAPFA1234A1Z5",             // ✅ From partners table
  invoiceDate: "2024-12-01",                // ✅ From invoices table
  invoiceNumber: "INV/NOV24/001",           // ✅ From invoices table
  gstBaseAmount: 130000.00,                 // ✅ Calculated from payout_calculation
  accountNumber: "1234567890",              // ✅ From partners table
  ifscCode: "HDFC0001234",                  // ✅ From partners table

  // Additional data for CGST/IGST logic
  state: "Maharashtra",                     // ✅ From partners table
  trailAmount: 80000.00,                    // ✅ Optional: for breakdown
  upfrontAmount: 50000.00                   // ✅ Optional: for breakdown
}
```

### **Frontend Flow:**

```javascript
// Step 1: Fetch invoice data
const response = await fetch(`/api/partner/${destinationId}/invoice/${month}/data`);
const data = await response.json();

// Step 2: Your HTML template applies CGST/IGST logic
const companyState = "Maharashtra";
const isSameState = data.state === companyState;

const gstAmount = data.gstBaseAmount * 0.18;
const cgst = isSameState ? gstAmount / 2 : 0;
const sgst = isSameState ? gstAmount / 2 : 0;
const igst = isSameState ? 0 : gstAmount;

// Step 3: Generate PDF using your existing HTML code
const pdf = generatePDF({
  ...data,
  cgst,
  sgst,
  igst,
  totalAmount: data.gstBaseAmount + gstAmount
});

// Step 4: Display PDF to partner
displayPDF(pdf);
```

---

## User Flows

### **FLOW 1: Partner Invoice Generation & E-Signature**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PARTNER FLOW                                 │
└─────────────────────────────────────────────────────────────────┘

Step 1: Login & View Invoices
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Partner logs in with destination_id

┌─────────────────────────────────────────┐
│  Partner Dashboard - Rajendra Nalbalwar │
├─────────────────────────────────────────┤
│  Month      GST Base    GST      Action │
│  2024-11    ₹1,30,000   ₹23,400  [Generate Invoice]│
│  2024-10    ₹95,000     ₹17,100  ✅ Submitted      │
│  2024-09    ₹88,000     ₹15,840  💰 Paid           │
│             UTR: HDFC2411250012345                 │
└─────────────────────────────────────────┘


Step 2: Generate Invoice - Popup Opens
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Partner clicks "Generate Invoice" → Popup opens

┌─────────────────────────────────────────┐
│  Generate Invoice - November 2024       │
├─────────────────────────────────────────┤
│  Partner Details:                       │
│  Name: Rajendra Nalbalwar               │
│  GST: 29AAPFA1234A1Z5                   │
│  Email: rajendra@email.com              │
│                                         │
│  Invoice Number (editable):             │
│  [INV/NOV24/001                    ]    │
│                                         │
│  Breakdown:                             │
│  Trail Payout:        ₹80,000           │
│  Upfront Payout:      ₹50,000           │
│  ────────────────────────────           │
│  GST Base:            ₹1,30,000         │
│  GST @ 18%:           ₹23,400           │
│  ────────────────────────────           │
│  Total:               ₹1,53,400         │
│                                         │
│  [Cancel]  [Generate Invoice]           │
└─────────────────────────────────────────┘


Step 3: Invoice Generated - PDF Preview
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
System generates PDF and displays in popup

┌─────────────────────────────────────────┐
│  Invoice Generated ✅                   │
├─────────────────────────────────────────┤
│  Invoice: INV/NOV24/001                 │
│  Date: 01-Dec-2024                      │
│                                         │
│  [PDF Preview - Scrollable]             │
│  ┌─────────────────────────────────┐   │
│  │ INVOICE                         │   │
│  │ INV/NOV24/001                   │   │
│  │                                 │   │
│  │ Bill To:                        │   │
│  │ Rajendra Nalbalwar              │   │
│  │ GST: 29AAPFA1234A1Z5            │   │
│  │                                 │   │
│  │ Particulars:                    │   │
│  │ Trail Commission    ₹80,000     │   │
│  │ Upfront Payment     ₹50,000     │   │
│  │                     ────────    │   │
│  │ Taxable Amount:     ₹1,30,000   │   │
│  │ CGST @ 9%:          ₹11,700     │   │
│  │ SGST @ 9%:          ₹11,700     │   │
│  │                     ────────    │   │
│  │ Total:              ₹1,53,400   │   │
│  └─────────────────────────────────┘   │
│                                         │
│  [Download PDF]  [E-Sign Invoice]       │
└─────────────────────────────────────────┘


Step 4: E-Signature - Request OTP
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Partner clicks "E-Sign Invoice"

System sends OTP to partner's registered email

📧 Email:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
To: rajendra@email.com
Subject: OTP for Invoice Signature

Your OTP: 483921

Use this OTP to digitally sign Invoice INV/NOV24/001
for November 2024 (GST Amount: ₹23,400)

Valid for 10 minutes. Do not share this OTP.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────┐
│  Enter OTP to Sign Invoice              │
├─────────────────────────────────────────┤
│  OTP sent to raj***@email.com           │
│                                         │
│  ┌───┬───┬───┬───┬───┬───┐            │
│  │ 4 │ 8 │ 3 │ 9 │ 2 │ 1 │            │
│  └───┴───┴───┴───┴───┴───┘            │
│                                         │
│  Expires in: 09:45                      │
│  Attempts remaining: 3                  │
│                                         │
│  [Verify & Sign]  [Resend OTP]          │
└─────────────────────────────────────────┘


Step 5: OTP Verified - Invoice Submitted
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Partner enters correct OTP

┌─────────────────────────────────────────┐
│  ✅ Invoice Signed Successfully!        │
├─────────────────────────────────────────┤
│  Your invoice INV/NOV24/001 has been    │
│  submitted for payment processing.      │
│                                         │
│  GST Payment: ₹23,400                   │
│  Status: Submitted                      │
│  Signed at: 01-Dec-2024, 3:42 PM        │
│                                         │
│  Payment will be processed in the       │
│  next batch cycle.                      │
│                                         │
│  [Download Signed Invoice]              │
│  [Back to Dashboard]                    │
└─────────────────────────────────────────┘

Partner Dashboard Updates:
┌─────────────────────────────────────────┐
│  Month      GST Amount   Status         │
│  2024-11    ₹23,400      ✅ Submitted   │
│  2024-10    ₹17,100      💰 Paid        │
└─────────────────────────────────────────┘
```

---

### **FLOW 2: Admin Collection & Bank File Export**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ADMIN FLOW - COLLECTION                      │
└─────────────────────────────────────────────────────────────────┘

Step 1: View Submitted Invoices
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌──────────────────────────────────────────────────────────────┐
│  Admin Dashboard - Submitted Invoices (Not Paid)            │
├──────────────────────────────────────────────────────────────┤
│  Filters: Month [2024-11 ▼]  Status [Submitted ▼]          │
│                                                              │
│  ☑ Select All (150 invoices)                                │
│                                                              │
│  ☑ Invoice ID  Invoice#      Partner         GST Amount     │
│  ☑ inv_12345   INV/NOV24/001 Rajendra N.     ₹23,400       │
│  ☑ inv_12346   INV/NOV24/002 Sunita S.       ₹18,500       │
│  ☑ inv_12347   INV/NOV24/003 Suvendu C.      ₹15,200       │
│  ☑ inv_12348   INV/NOV24/004 Ronak J.        ₹32,100       │
│  ...                                                         │
│                                                              │
│  Total Selected: 150 invoices                               │
│  Total Amount: ₹24,50,000                                   │
│                                                              │
│  [Export Bank File] ← Button                                 │
└──────────────────────────────────────────────────────────────┘


Step 2: Generate & Download Bank File
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────┐
│  Export Bank File                       │
├─────────────────────────────────────────┤
│  Month: November 2024                   │
│  Total Invoices: 150                    │
│  Total Amount: ₹24,50,000               │
│                                         │
│  File Format:                           │
│  • Invoice ID (for reconciliation)      │
│  • Destination ID                       │
│  • Partner Name                         │
│  • Account Number                       │
│  • IFSC Code                            │
│  • GST Amount                           │
│  • Invoice Number                       │
│                                         │
│  [Cancel]  [Generate Bank File]         │
└─────────────────────────────────────────┘

Generated File: GST_Payment_Nov2024.csv
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
invoice_id,destination_id,partner_name,account_number,ifsc_code,gst_amount,invoice_number
inv_12345,75c6b6c6-4ff2-415b-b793-7ab2343e2c21,Rajendra Nalbalwar,1234567890,HDFC0001234,23400.00,INV/NOV24/001
inv_12346,564ff61c-3c3e-40eb-abd8-518c900433aa,Sunita Sahu,9876543210,ICIC0005678,18500.00,INV/NOV24/002
inv_12347,ffc8d3bb-80ff-4e30-a403-2ef357779e85,Suvendu Chakraborty,5555666677,SBIN0001234,15200.00,INV/NOV24/003
...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────┐
│  ✅ Bank File Generated                 │
├─────────────────────────────────────────┤
│  File: GST_Payment_Nov2024.csv          │
│  Invoices: 150                          │
│  Total Amount: ₹24,50,000               │
│                                         │
│  [Download File] 📥                     │
│                                         │
│  Next Step:                             │
│  Upload this file to your bank portal   │
│  for processing GST payments.           │
└─────────────────────────────────────────┘

Admin downloads file → Uploads to bank portal → Bank processes payments
```

---

### **FLOW 3: Payment Reconciliation**

```
┌─────────────────────────────────────────────────────────────────┐
│              ADMIN FLOW - RECONCILIATION                        │
└─────────────────────────────────────────────────────────────────┘

Step 1: Bank Returns Settlement File
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Bank processes payments and returns settlement file

Bank Settlement File: Settlement_Nov2024.csv
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
invoice_id,utr_number,amount,payment_date,status,remarks
inv_12345,HDFC2412050012345,23400.00,2024-12-08,SUCCESS,
inv_12346,HDFC2412050012346,18500.00,2024-12-08,SUCCESS,
inv_12347,HDFC2412050012347,15200.00,2024-12-08,FAILED,Invalid account number
inv_12348,HDFC2412050012348,32100.00,2024-12-08,SUCCESS,
...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


Step 2: Upload Settlement File for Reconciliation
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────┐
│  Reconcile Payments                     │
├─────────────────────────────────────────┤
│  Upload bank settlement file:           │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ 📄 Drag & drop CSV file here    │   │
│  │    or click to browse           │   │
│  └─────────────────────────────────┘   │
│                                         │
│  [Cancel]  [Upload & Reconcile]         │
└─────────────────────────────────────────┘

Admin uploads Settlement_Nov2024.csv


Step 3: Auto-Reconciliation Process
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

System Process:
1. Parse CSV file
2. For each row, match by invoice_id (KEY IDENTIFIER)
3. Update invoice table:
   - utr_number
   - payment_date
   - payment_status (SUCCESS/FAILED)
   - payment_failure_reason (if FAILED)
4. Generate reconciliation report


Step 4: Reconciliation Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌──────────────────────────────────────────────────────────────┐
│  Reconciliation Complete ✅                                  │
├──────────────────────────────────────────────────────────────┤
│  File: Settlement_Nov2024.csv                               │
│  Processed: 150 invoices                                     │
│                                                              │
│  Summary:                                                    │
│  ✅ Successful Payments: 148 (₹24,34,800)                  │
│  ❌ Failed Payments: 2 (₹15,200)                           │
│  ⚠️ Unmatched Records: 0                                    │
│                                                              │
│  Failed Invoices:                                           │
│  ┌────────────────────────────────────────────────────┐    │
│  │ inv_12347  Suvendu C.  ₹15,200  Invalid account   │    │
│  │ inv_12385  Amit K.     ₹12,300  Insufficient bal. │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  [Download Report]  [Export Failed Invoices]                │
│  [Retry Failed Payments]                                     │
└──────────────────────────────────────────────────────────────┘


Step 5: Partner Sees Updated Status
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Partner Dashboard Auto-Updates:

┌─────────────────────────────────────────┐
│  Partner Dashboard - Rajendra Nalbalwar │
├─────────────────────────────────────────┤
│  Month   GST Amount  Status             │
│  2024-11 ₹23,400     💰 Paid            │
│          UTR: HDFC2412050012345         │
│          Date: 08-Dec-2024              │
│          [View Details] [Download]      │
│                                         │
│  2024-10 ₹18,500     💰 Paid            │
│  2024-09 ₹15,200     ✅ Submitted       │
└─────────────────────────────────────────┘

Click "View Details":
┌─────────────────────────────────────────┐
│  Payment Details - Nov 2024             │
├─────────────────────────────────────────┤
│  Invoice: INV/NOV24/001                 │
│  Status: Paid ✅                        │
│                                         │
│  GST Amount: ₹23,400                    │
│  UTR Number: HDFC2412050012345          │
│  Payment Date: 08-Dec-2024              │
│                                         │
│  Timeline:                              │
│  ✅ Invoice Generated: 01-Dec-2024      │
│  ✅ Digitally Signed: 01-Dec-2024       │
│  ✅ Payment Processed: 08-Dec-2024      │
│                                         │
│  [Download Invoice] [Download Receipt]  │
└─────────────────────────────────────────┘
```

---

## API Endpoints

### **Partner APIs**

#### **1. Get Partner Invoices**

```http
GET /api/partner/:destination_id/invoices
Query Params: ?financial_year=2024-2025&status=all

Response 200:
{
  "destinationId": "75c6b6c6-4ff2-415b-b793-7ab2343e2c21",
  "partnerName": "Rajendra Nalbalwar",
  "invoices": [
    {
      "invoiceId": "inv_12345",
      "invoiceNumber": "INV/NOV24/001",
      "month": "2024-11",
      "invoiceDate": "2024-12-01",
      "gstBaseAmount": 130000.00,
      "gstAmount": 23400.00,
      "status": "submitted",
      "signedAt": "2024-12-01T15:42:30Z",
      "invoicePdfUrl": "https://s3.../INV-NOV24-001.pdf",
      "signedPdfUrl": "https://s3.../INV-NOV24-001-signed.pdf",
      "utrNumber": null,
      "paymentDate": null
    },
    {
      "invoiceId": "inv_12234",
      "invoiceNumber": "INV/OCT24/125",
      "month": "2024-10",
      "gstBaseAmount": 95000.00,
      "gstAmount": 17100.00,
      "status": "paid",
      "utrNumber": "HDFC2411250012345",
      "paymentDate": "2024-11-25"
    }
  ],
  "summary": {
    "totalInvoices": 8,
    "pendingGeneration": 1,
    "submitted": 2,
    "paid": 5,
    "ytdGstAmount": 185000.00
  }
}
```

#### **2. Get Invoice Data for PDF Generation**

**Purpose:** Frontend calls this to get raw data for PDF generation using HTML template

```http
GET /api/partner/:destination_id/invoice/:month/data
Query Params: ?invoiceNumber=INV/NOV24/001  // Optional custom invoice number

Response 200:
{
  "invoiceId": 12345,
  "invoiceNumber": "INV/NOV24/001",
  "invoiceDate": "2024-12-01",
  "month": "2024-11",
  "financialYear": "2024-2025",

  // ✅ All 7 required fields for your HTML template:
  "partnerName": "Rajendra Nalbalwar",           // 1. From partners table
  "gstNumber": "29AAPFA1234A1Z5",                // 2. From partners table
  "accountNumber": "1234567890",                 // 3. From partners table
  "ifscCode": "HDFC0001234",                     // 4. From partners table
  "gstBaseAmount": 130000.00,                    // 5. Calculated from payout_calculation

  // Additional data for frontend logic:
  "state": "Maharashtra",                        // For CGST/IGST logic
  "address": "123 Main St, Mumbai",
  "email": "rajendra@email.com",
  "trailAmount": 80000.00,                       // Optional: for breakdown
  "upfrontAmount": 50000.00,                     // Optional: for breakdown

  // Status
  "status": "generated",
  "digitallySigned": false
}

// ✅ Frontend then:
// 1. Receives this JSON
// 2. Applies CGST/IGST logic based on state
// 3. Renders HTML template with data
// 4. Converts HTML to PDF using your existing code
```

#### **3. Request OTP for E-Signature**

```http
POST /api/partner/invoice/:invoice_id/request-otp
Content-Type: application/json

Request:
{
  "destinationId": "75c6b6c6-4ff2-415b-b793-7ab2343e2c21"
}

Response 200:
{
  "otpId": "otp_67890",
  "invoiceId": "inv_12345",
  "message": "OTP sent successfully",
  "otpSentTo": "raj***@email.com",
  "expiresAt": "2024-12-01T15:55:00Z",
  "expiresIn": 600,  // seconds
  "attemptsRemaining": 3
}
```

#### **4. Verify OTP & Sign Invoice**

```http
POST /api/partner/invoice/:invoice_id/verify-otp
Content-Type: application/json

Request:
{
  "destinationId": "75c6b6c6-4ff2-415b-b793-7ab2343e2c21",
  "otpId": "otp_67890",
  "otpCode": "483921"
}

Response 200 (Success):
{
  "success": true,
  "invoiceId": "inv_12345",
  "invoiceNumber": "INV/NOV24/001",
  "status": "submitted",
  "signedAt": "2024-12-01T15:42:30Z",
  "signedIp": "103.45.67.89",
  "signedPdfUrl": "https://s3.../INV-NOV24-001-signed.pdf",
  "message": "Invoice signed successfully. Payment will be processed in the next batch."
}

Response 400 (Invalid OTP):
{
  "success": false,
  "error": "Invalid OTP",
  "attemptsRemaining": 2,
  "message": "Incorrect OTP. You have 2 attempts remaining."
}

Response 400 (OTP Expired):
{
  "success": false,
  "error": "OTP expired",
  "message": "OTP has expired. Please request a new OTP."
}

Response 429 (Max Attempts):
{
  "success": false,
  "error": "Max attempts exceeded",
  "message": "You have exceeded maximum OTP attempts. Please contact support."
}
```

#### **5. Download Invoice PDF**

```http
GET /api/partner/invoice/:invoice_id/pdf?type=signed

Query Params:
- type: "original" or "signed" (default: "signed")

Response 200:
Content-Type: application/pdf
[PDF Binary Data]

or Redirect to S3 URL:
Status: 302 Found
Location: https://s3.../INV-NOV24-001-signed.pdf
```

---

### **Admin APIs**

#### **1. Get Submitted Invoices (Not Paid)**

```http
GET /api/admin/invoices/submitted
Query Params: ?month=2024-11&status=submitted&unpaid=true

Response 200:
{
  "filters": {
    "month": "2024-11",
    "status": "submitted",
    "unpaid": true
  },
  "summary": {
    "totalInvoices": 150,
    "totalAmount": 2450000.00
  },
  "invoices": [
    {
      "invoiceId": "inv_12345",
      "invoiceNumber": "INV/NOV24/001",
      "destinationId": "75c6b6c6-4ff2-415b-b793-7ab2343e2c21",
      "partnerName": "Rajendra Nalbalwar",
      "partnerGst": "29AAPFA1234A1Z5",
      "month": "2024-11",
      "gstBaseAmount": 130000.00,
      "gstAmount": 23400.00,
      "status": "submitted",
      "signedAt": "2024-12-01T15:42:30Z",
      "accountNumber": "1234567890",
      "ifscCode": "HDFC0001234",
      "isPaid": false
    },
    // ... more invoices
  ]
}
```

#### **2. Export Bank File**

```http
POST /api/admin/export-bank-file
Content-Type: application/json

Request:
{
  "month": "2024-11",
  "invoiceIds": ["inv_12345", "inv_12346", ...],  // Optional, exports all if not provided
  "format": "csv"  // Default: csv
}

Response 200:
{
  "bankFileId": "bf_789",
  "fileName": "GST_Payment_Nov2024.csv",
  "downloadUrl": "https://s3.../bank-files/GST_Payment_Nov2024.csv",
  "summary": {
    "totalInvoices": 150,
    "totalAmount": 2450000.00,
    "generatedAt": "2024-12-05T10:30:00Z"
  },
  "columns": [
    "invoice_id",
    "destination_id",
    "partner_name",
    "account_number",
    "ifsc_code",
    "gst_amount",
    "invoice_number"
  ]
}

CSV File Format:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
invoice_id,destination_id,partner_name,account_number,ifsc_code,gst_amount,invoice_number
inv_12345,75c6b6c6-4ff2-415b-b793-7ab2343e2c21,Rajendra Nalbalwar,1234567890,HDFC0001234,23400.00,INV/NOV24/001
...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### **3. Reconcile Payment (Upload Settlement File)**

```http
POST /api/admin/reconcile-payment
Content-Type: multipart/form-data

Request:
{
  "settlementFile": [CSV File],
  "bankFileId": "bf_789",  // Optional
  "month": "2024-11"
}

Expected CSV Format from Bank:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
invoice_id,utr_number,amount,payment_date,status,remarks
inv_12345,HDFC2412050012345,23400.00,2024-12-08,SUCCESS,
inv_12346,HDFC2412050012346,18500.00,2024-12-08,SUCCESS,
inv_12347,HDFC2412050012347,15200.00,2024-12-08,FAILED,Invalid account
...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Response 200:
{
  "reconciliationId": "rec_456",
  "bankFileId": "bf_789",
  "month": "2024-11",
  "processedAt": "2024-12-10T11:00:00Z",
  "summary": {
    "totalRecords": 150,
    "successful": 148,
    "failed": 2,
    "unmatched": 0,
    "successfulAmount": 2434800.00,
    "failedAmount": 15200.00
  },
  "successfulPayments": [
    {
      "invoiceId": "inv_12345",
      "invoiceNumber": "INV/NOV24/001",
      "destinationId": "75c6b6c6-4ff2-415b-b793-7ab2343e2c21",
      "utrNumber": "HDFC2412050012345",
      "amount": 23400.00,
      "paymentDate": "2024-12-08",
      "status": "paid"
    }
    // ... more
  ],
  "failedPayments": [
    {
      "invoiceId": "inv_12347",
      "invoiceNumber": "INV/NOV24/003",
      "destinationId": "ffc8d3bb-80ff-4e30-a403-2ef357779e85",
      "amount": 15200.00,
      "status": "failed",
      "failureReason": "Invalid account number"
    }
    // ... more
  ],
  "unmatchedRecords": []
}
```

#### **4. Get Reconciliation Report**

```http
GET /api/admin/reconciliation/:reconciliation_id

Response 200:
{
  "reconciliationId": "rec_456",
  "bankFileId": "bf_789",
  "month": "2024-11",
  "processedAt": "2024-12-10T11:00:00Z",
  "summary": {
    "totalRecords": 150,
    "successful": 148,
    "failed": 2,
    "successfulAmount": 2434800.00,
    "failedAmount": 15200.00
  },
  "downloadUrls": {
    "fullReport": "https://s3.../reports/reconciliation_rec_456.pdf",
    "failedInvoicesCsv": "https://s3.../reports/failed_invoices_rec_456.csv"
  }
}
```

---

## Reconciliation Process

### **Key Identifier: invoice_id**

The reconciliation process uses `invoice_id` as the **primary key identifier** to match bank settlement records with internal invoice records.

### **Reconciliation Flow**

```
┌─────────────────────────────────────────────────────────────────┐
│                    RECONCILIATION PROCESS                       │
└─────────────────────────────────────────────────────────────────┘

Step 1: Admin Exports Bank File
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
System generates bank file with invoice_id as key column

invoice_id,destination_id,partner_name,account_number,ifsc_code,gst_amount,invoice_number
inv_12345,75c6b6c6...,Rajendra Nalbalwar,1234567890,HDFC0001234,23400.00,INV/NOV24/001

↓ Admin uploads to bank portal


Step 2: Bank Processes Payments
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Bank processes payments and returns settlement file with invoice_id

invoice_id,utr_number,amount,payment_date,status,remarks
inv_12345,HDFC2412050012345,23400.00,2024-12-08,SUCCESS,
inv_12346,HDFC2412050012346,18500.00,2024-12-08,SUCCESS,
inv_12347,HDFC2412050012347,15200.00,2024-12-08,FAILED,Invalid account

↓ Admin uploads settlement file to system


Step 3: Auto-Matching by invoice_id
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
System matches each settlement record by invoice_id

FOR EACH row in settlement file:
  1. Extract invoice_id from row
  2. FIND invoice in database WHERE invoice_id = row.invoice_id
  3. IF found:
       - UPDATE invoice SET:
           utr_number = row.utr_number
           payment_date = row.payment_date
           payment_status = row.status
           payment_failure_reason = row.remarks (if FAILED)
           status = 'paid' (if SUCCESS) OR 'failed' (if FAILED)
           updated_at = NOW()
  4. IF not found:
       - ADD to unmatched_records list

↓ Generate reconciliation report


Step 4: Update Partner Dashboard
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Partners see real-time updated payment status

Invoice Status Changes:
- submitted → paid (if SUCCESS)
- submitted → failed (if FAILED)

Partners can view:
- UTR Number
- Payment Date
- Payment Status
```

### **Reconciliation Logic (Pseudocode)**

```javascript
async function reconcilePayments(settlementFile, bankFileId) {
  const settlements = parseCSV(settlementFile);

  const results = {
    successful: [],
    failed: [],
    unmatched: []
  };

  for (const row of settlements) {
    const { invoice_id, utr_number, amount, payment_date, status, remarks } = row;

    // Find invoice by invoice_id (KEY IDENTIFIER)
    const invoice = await db.query(
      'SELECT * FROM invoices WHERE invoice_id = $1',
      [invoice_id]
    );

    if (!invoice) {
      // Invoice not found - unmatched record
      results.unmatched.push({
        invoice_id,
        utr_number,
        amount,
        reason: 'Invoice ID not found in system'
      });
      continue;
    }

    // Validate amount matches
    if (parseFloat(amount) !== parseFloat(invoice.gst_amount)) {
      results.unmatched.push({
        invoice_id,
        utr_number,
        amount,
        expectedAmount: invoice.gst_amount,
        reason: 'Amount mismatch'
      });
      continue;
    }

    // Update invoice with settlement details
    const updateQuery = `
      UPDATE invoices
      SET
        utr_number = $1,
        payment_date = $2,
        payment_status = $3,
        payment_failure_reason = $4,
        status = CASE
          WHEN $3 = 'SUCCESS' THEN 'paid'
          WHEN $3 = 'FAILED' THEN 'failed'
          ELSE status
        END,
        updated_at = NOW()
      WHERE invoice_id = $5
      RETURNING *
    `;

    const updatedInvoice = await db.query(updateQuery, [
      utr_number,
      payment_date,
      status,
      status === 'FAILED' ? remarks : null,
      invoice_id
    ]);

    // Categorize result
    if (status === 'SUCCESS') {
      results.successful.push({
        invoice_id,
        invoice_number: invoice.invoice_number,
        destination_id: invoice.destination_id,
        utr_number,
        amount,
        payment_date
      });
    } else {
      results.failed.push({
        invoice_id,
        invoice_number: invoice.invoice_number,
        destination_id: invoice.destination_id,
        amount,
        failure_reason: remarks
      });
    }
  }

  // Save reconciliation record
  const reconciliation = await saveReconciliation({
    bank_file_id: bankFileId,
    processed_at: new Date(),
    summary: {
      total: settlements.length,
      successful: results.successful.length,
      failed: results.failed.length,
      unmatched: results.unmatched.length
    }
  });

  return {
    reconciliation_id: reconciliation.id,
    summary: reconciliation.summary,
    details: results
  };
}
```

### **Edge Cases Handled**

1. **Invoice ID Not Found**: Settlement record has invoice_id that doesn't exist in system
   - Action: Flag as unmatched, admin review required

2. **Amount Mismatch**: Settlement amount ≠ invoice GST amount
   - Action: Flag as unmatched, admin review required

3. **Duplicate UTR**: Same UTR appears multiple times
   - Action: First occurrence processed, subsequent flagged for review

4. **Failed Payment**: Bank returns FAILED status
   - Action: Update invoice status to 'failed', record failure reason

5. **Partial Settlements**: Settlement amount < invoice amount
   - Action: Flag as unmatched, manual reconciliation required

---

## Technology Stack Recommendations

### **Backend**
- **Language**: Node.js (Express) or Python (FastAPI)
- **Database**: PostgreSQL (ACID compliance)
- **Cache**: Redis (OTP storage, sessions)
- **Storage**: AWS S3 or Google Cloud Storage (PDFs)
- **Queue**: Bull (Node.js) or Celery (Python) for async jobs

### **Frontend**
- **Framework**: React.js or Next.js
- **UI Library**: Material-UI or Tailwind CSS
- **State Management**: React Context API or Redux
- **PDF Viewer**: react-pdf or PDF.js

### **Integrations**
- **Email**: SendGrid or AWS SES
- **SMS**: Twilio or MSG91 (optional)
- **PDF Generation**: Puppeteer (Node.js) or WeasyPrint (Python)
- **OTP**: bcrypt for hashing

### **DevOps**
- **Hosting**: AWS, GCP, or Azure
- **CI/CD**: GitHub Actions or GitLab CI
- **Monitoring**: Datadog or New Relic

---

## Security Considerations

1. **Authentication**: JWT-based with refresh tokens
2. **Authorization**: Role-based access (Partner vs Admin)
3. **OTP Security**: bcrypt hashing, 10-min expiry, max 3 attempts
4. **Data Encryption**:
   - At rest: AES-256
   - In transit: TLS 1.3
5. **SQL Injection**: Use parameterized queries
6. **XSS Prevention**: Input sanitization, CSP headers
7. **Rate Limiting**: Max 100 requests/min per user
8. **Audit Trail**: Log all sensitive operations with timestamps and IP addresses

---

## Compliance

1. **GST Compliance**: Invoice format per CGST Act
2. **Data Retention**: 7 years minimum
3. **Digital Signature**: OTP-based consent with timestamp and IP logging
4. **Audit Trail**: Immutable logs for all transactions
5. **Privacy**: Mask sensitive data (account numbers) in logs

---

## Next Steps

1. **Database Setup**: Create PostgreSQL database with schema
2. **Backend Implementation**: Build REST APIs
3. **PDF Template Design**: Create invoice PDF template
4. **Frontend Development**: Build partner portal and admin dashboard
5. **Integration**: Connect email service for OTP delivery
6. **Testing**: Unit tests, integration tests, UAT
7. **Deployment**: Set up CI/CD pipeline and deploy to production

---

**End of Document**
