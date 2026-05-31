# UX Flows — CheckChain

**Version:** 1.0  
**Date:** 2026-01-15  
**Framework:** Next.js 14 (App Router) + shadcn/ui + Tailwind

---

## Table of Contents

1. [Information Architecture](#1-information-architecture)
2. [Dashboard](#2-dashboard)
3. [New Agreement Wizard](#3-new-agreement-wizard)
4. [Agreement Detail](#4-agreement-detail)
5. [Attestation Submission](#5-attestation-submission)
6. [Dispute Workflow](#6-dispute-workflow)
7. [Receipt View (Digital Cashier's Check)](#7-receipt-view-digital-cashiers-check)
8. [AI: Draft Conditions](#8-ai-draft-conditions)
9. [Settings / KYC](#9-settings--kyc)
10. [Component Library Notes](#10-component-library-notes)

---

## 1. Information Architecture

```
/
├── /dashboard                         — Home: agreement summary, wallet balance, activity
├── /agreements
│   ├── /agreements/new                — Multi-step wizard
│   └── /agreements/[id]              — Agreement detail + actions
├── /attestations
│   └── /attestations/[id]            — (redirects to /agreements/[id]#attest)
├── /disputes
│   └── /disputes/[id]               — Dispute detail (for arbitrators)
├── /receipts
│   └── /receipts/[txid]             — Printable Digital Cashier's Check
├── /ai
│   └── /ai/draft-conditions          — Standalone AI clause tool
├── /settings
│   ├── /settings/profile
│   ├── /settings/kyc
│   └── /settings/wallet
└── /auth
    ├── /auth/login
    └── /auth/logout
```

---

## 2. Dashboard

**Route:** `/dashboard`  
**Auth:** Required

### Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  CheckChain Logo    [Dashboard] [Agreements] [AI Tools]   [User ▼] │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Welcome back, Alex Chen                    [+ New Agreement]       │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │ Wallet       │  │ Active       │  │ Released     │             │
│  │ $12,500.00   │  │ Agreements   │  │ This Month   │             │
│  │ USD          │  │     4        │  │  $15,200.00  │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
│                                                                     │
│  ACTIVE AGREEMENTS                              [View All →]        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ AGT-2026-0001  Jordan Lee   $5,000  ● FUNDED    1/2 conds  │   │
│  │ AGT-2026-0002  M. Rivera    $2,200  ● DRAFT     —          │   │
│  │ AGT-2026-0003  K. Patel     $8,500  ● RELEASED  2/2 conds  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  RECENT ACTIVITY                                                    │
│  • Jan 20 — Attestation submitted on AGT-2026-0001 by Jordan Lee   │
│  • Jan 15 — Agreement AGT-2026-0001 funded ($5,000)                │
│  • Jan 12 — AGT-2026-0003 released — Receipt available             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### States

- **Loading:** Skeleton cards (3 stat cards + table rows)
- **Empty (new user):** "You have no agreements yet. Create your first agreement to get started." + CTA button
- **KYC not verified:** Banner: "Complete identity verification to fund agreements." + link to /settings/kyc

---

## 3. New Agreement Wizard

**Route:** `/agreements/new`  
**Steps:** 5 (Parties → Amount → Conditions → Review → Fund)

### Step 1: Parties

```
┌─────────────────────────────────────────────────────────────────────┐
│  New Agreement                            Step 1 of 5: Parties      │
│  ●─────○─────○─────○─────○                                          │
│                                                                     │
│  YOU ARE THE: ● Payer   ○ Payee                                    │
│                                                                     │
│  PAYEE                                                              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Email address of payee                                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Organization (MSP)  [PayeeOrg ▼]                            │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ✓ Payee found: Jordan Lee (contractor@payeeorg.example.com)        │
│    KYC Status: Verified                                             │
│                                                                     │
│                                    [Cancel]  [Next: Amount →]       │
└─────────────────────────────────────────────────────────────────────┘
```

**Validation:** Payee email must resolve to a verified user in the system. Shows "User not found" error inline.

---

### Step 2: Amount & Terms

```
┌─────────────────────────────────────────────────────────────────────┐
│  New Agreement                     Step 2 of 5: Amount & Terms      │
│  ●─────●─────○─────○─────○                                          │
│                                                                     │
│  AMOUNT                                                             │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ $  5,000.00                                    USD           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  Your wallet: $12,500.00 available                                  │
│                                                                     │
│  RELEASE MODE                                                       │
│  ● AUTO — funds release automatically when all conditions met       │
│  ○ MANUAL — you approve release after conditions are submitted      │
│                                                                     │
│  EXPIRATION DATE                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ March 15, 2026                                              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  PRIVATE NOTES (visible to both parties only)                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Unit 4B bathroom renovation per quote #Q-2026-042            │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│                          [← Back]  [Next: Conditions →]            │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Step 3: Conditions

```
┌─────────────────────────────────────────────────────────────────────┐
│  New Agreement                         Step 3 of 5: Conditions      │
│  ●─────●─────●─────○─────○                                          │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ ✨ AI Clause Drafter                                        │  │
│  │ Describe your requirements in plain English:                │  │
│  │ ┌────────────────────────────────────────────────────────┐  │  │
│  │ │ Pay $5,000 when contractor delivers signed inspection   │  │  │
│  │ │ report and before/after photos by end of February...   │  │  │
│  │ └────────────────────────────────────────────────────────┘  │  │
│  │                              [Generate Conditions →]        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  CONDITIONS (2)                                   [+ Add Manually]  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ COND-001  📄 DELIVERABLE                           [✕ Remove]│  │
│  │ Signed inspection report from licensed inspector             │  │
│  │ Attestors: PayeeOrgMSP                                       │  │
│  ├──────────────────────────────────────────────────────────────┤  │
│  │ COND-002  📅 DATE                                  [✕ Remove]│  │
│  │ Work completed by February 28, 2026                          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ⚠️ AI Disclaimer: These are suggestions. Review before proceeding. │
│                                                                     │
│                           [← Back]  [Next: Review →]               │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Step 4: Review

```
┌─────────────────────────────────────────────────────────────────────┐
│  New Agreement                              Step 4 of 5: Review     │
│  ●─────●─────●─────●─────○                                          │
│                                                                     │
│  REVIEW YOUR AGREEMENT                                              │
│                                                                     │
│  Payer:         Alex Chen (PayerOrgMSP)                             │
│  Payee:         Jordan Lee (PayeeOrgMSP)                            │
│  Amount:        $5,000.00 USD                                       │
│  Release Mode:  AUTO                                                │
│  Expires:       March 15, 2026                                      │
│                                                                     │
│  CONDITIONS                                                         │
│  1. [DELIVERABLE] Signed inspection report from licensed inspector  │
│  2. [DATE] Work completed by February 28, 2026                      │
│                                                                     │
│  Private Notes: Unit 4B bathroom renovation                         │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ ✨ AI Risk Score                                            │  │
│  │ Overall Risk: LOW (28/100)                                  │  │
│  │ Counterparty: 12 completed, 0 disputes — Low risk           │  │
│  │ Conditions: Specific & measurable — Low risk                │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ☐ I have reviewed all terms and agree to create this agreement.    │
│                                                                     │
│                        [← Back]  [Create Agreement →]               │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Step 5: Fund

```
┌─────────────────────────────────────────────────────────────────────┐
│  Agreement Created! ✓                       Step 5 of 5: Fund       │
│  ●─────●─────●─────●─────●                                          │
│                                                                     │
│  Agreement ID: AGT-2026-0001                                        │
│  TX Hash: 7a3f2b1c...f9a0b                                          │
│  Block: #23                                                         │
│                                                                     │
│  FUND THIS AGREEMENT                                                │
│  Lock $5,000.00 in escrow to notify your payee that funds           │
│  are committed. Funds are secured until conditions are met          │
│  or the agreement is canceled by mutual consent.                    │
│                                                                     │
│  Your wallet: $12,500.00 → $7,500.00 after funding                 │
│                                                                     │
│  [Fund Now — $5,000.00]   [Fund Later — Go to Dashboard]           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Post-fund state:** Shows transaction confirmation with tx hash, block number, and link to Agreement Detail.

---

## 4. Agreement Detail

**Route:** `/agreements/[id]`

```
┌─────────────────────────────────────────────────────────────────────┐
│  Agreement AGT-2026-0001                           ● FUNDED         │
│                                                                     │
│  Payer: Alex Chen         Payee: Jordan Lee                         │
│  Amount: $5,000.00 USD    Release: AUTO                             │
│  Created: Jan 15, 2026    Expires: Mar 15, 2026                     │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ ✨ AI Summary                                               │  │
│  │ Jordan needs to submit a signed inspection report and        │  │
│  │ photos by Feb 28. Payment releases automatically.           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  CONDITIONS                                                         │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ ○ COND-001  Signed inspection report    [PENDING]          │    │
│  │   Required attestor: PayeeOrgMSP                           │    │
│  ├────────────────────────────────────────────────────────────┤    │
│  │ ○ COND-002  Completed by Feb 28, 2026  [PENDING]          │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ATTESTATIONS (0)                                                   │
│  No attestations submitted yet.                                     │
│                                                                     │
│  BLOCKCHAIN RECORD                                                  │
│  Funded TX: 9b2e4d6f...0e2f  Block: #31  Jan 15, 2026 2:32 PM      │
│                                                                     │
│  ACTIONS (shown by role)                                            │
│  [Submit Attestation]   [Raise Dispute]   [Cancel Agreement]        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Role-based action visibility:**

| Action | Visible to Payer | Visible to Payee | Status Required |
|--------|-----------------|-----------------|-----------------|
| Submit Attestation | No | Yes | FUNDED, PENDING_RELEASE |
| Approve Release | Yes | No | PENDING_RELEASE |
| Raise Dispute | Yes | Yes | FUNDED, PENDING_RELEASE |
| Cancel Agreement | Yes (draft), both (funded) | Yes (funded) | DRAFT or FUNDED |
| View Receipt | Yes | Yes | RELEASED, SETTLED |

---

## 5. Attestation Submission

**Route:** Modal on `/agreements/[id]` (or full page for mobile)

```
┌───────────────────────────────────────────────────────────┐
│  Submit Attestation                              [✕ Close] │
│                                                           │
│  CONDITION                                                │
│  COND-001: Signed inspection report from licensed          │
│  inspector                                                │
│                                                           │
│  EVIDENCE FILE                                            │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  Drag & drop your file here, or click to browse    │ │
│  │  Accepted: PDF, JPG, PNG, DOCX (max 25MB)          │ │
│  └─────────────────────────────────────────────────────┘ │
│  inspection-report-unit4b.pdf (2.1 MB)  ✓               │
│  SHA-256: a4f2e8c1...a9b0                                │
│                                                           │
│  ATTESTATION NOTE                                         │
│  ┌─────────────────────────────────────────────────────┐ │
│  │ Inspection by John Smith, License #12345, Jan 20   │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
│  The SHA-256 hash of your file will be recorded on the   │
│  Hyperledger Fabric ledger. The file itself is stored     │
│  securely off-chain.                                      │
│                                                           │
│                    [Cancel]  [Submit Attestation]         │
└───────────────────────────────────────────────────────────┘
```

**Post-submission:**
- If AUTO mode + all conditions met: Toast "🎉 All conditions met! Funds released automatically." + redirect to Receipt.
- If MANUAL mode: Toast "Attestation submitted. Waiting for payer approval."

---

## 6. Dispute Workflow

### Raise Dispute (Modal)

```
┌───────────────────────────────────────────────────────────┐
│  Raise Dispute on AGT-2026-0001               [✕ Close]   │
│                                                           │
│  ⚠️ Once raised, this agreement will be frozen and        │
│  reviewed by the Arbitrator organization.                 │
│                                                           │
│  REASON FOR DISPUTE                                       │
│  ┌─────────────────────────────────────────────────────┐ │
│  │ The submitted inspection report does not cover       │ │
│  │ bathroom scope items 3 and 4 specified in the        │ │
│  │ agreement conditions.                               │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
│  [Cancel]  [Submit Dispute — Funds will be frozen]        │
└───────────────────────────────────────────────────────────┘
```

### Dispute Detail (Arbitrator View)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Dispute DISP-2026-0001                            ● OPEN           │
│  Agreement: AGT-2026-0001  Amount: $5,000.00                        │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ ✨ AI Dispute Analysis                                      │  │
│  │ [Summary of both sides' positions]                          │  │
│  │ Suggested Resolution: SPLIT 75:25 (Medium confidence)       │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  PAYER'S POSITION: "Inspection report incomplete..."               │
│  PAYEE'S ATTESTATIONS:                                              │
│  • COND-001: inspection-report.pdf  Hash: a4f2...a9b0  (Jan 20)    │
│                                                                     │
│  RESOLUTION                                                         │
│  Decision:  [RELEASE_TO_PAYEE ▼]                                   │
│             [RETURN_TO_PAYER  ]                                     │
│             [SPLIT            ]                                     │
│  Split Ratio: [75] : [25]  (payee : payer)                         │
│  Resolution Notes:                                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Report covers 3 of 4 scope items. 75% payment is awarded.   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│                              [Submit Resolution]                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Receipt View (Digital Cashier's Check)

**Route:** `/receipts/[txid]`  
**Special:** Print-optimized CSS. "Print" button triggers `window.print()`.

```
┌─────────────────────────────────────────────────────────────────────┐
│  [Print Receipt]  [Download PDF]                   (screen header)  │
└─────────────────────────────────────────────────────────────────────┘

┌══════════════════════════════════════════════════════════════════════╗
║                    CHECKCHAIN SETTLEMENT NETWORK                     ║
║              ─────────────────────────────────────────               ║
║  DIGITAL CASHIER'S CHECK                              No. AGT-2026-0001  ║
║                                                                      ║
║  Date: January 20, 2026                    Amount: **$5,000.00**     ║
║                                                                      ║
║  PAY TO THE ORDER OF: Jordan Lee                                     ║
║               contractor@payeeorg.example.com (PayeeOrgMSP)          ║
║                                                                      ║
║  ─────────────────────────────────────────────────────────────────   ║
║  FIVE THOUSAND AND 00/100 DOLLARS                                    ║
║  ─────────────────────────────────────────────────────────────────   ║
║                                                                      ║
║  Remitter: Alex Chen                                                 ║
║            user@payerorg.example.com (PayerOrgMSP)                  ║
║                                                                      ║
║  Memo: Unit 4B bathroom renovation                                   ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────┐  ┌─────────┐  ║
║  │ CONDITIONS SATISFIED                             │  │ QR CODE │  ║
║  │ ✓ COND-001: Inspection report   Jan 20, 2026   │  │ [█████] │  ║
║  │   Evidence: a4f2e8c1...a9b0                     │  │ [█   █] │  ║
║  │ ✓ COND-002: Deadline (Feb 28)   Jan 20, 2026   │  │ [█████] │  ║
║  └──────────────────────────────────────────────────┘  └─────────┘  ║
║                                                                      ║
║  BLOCKCHAIN VERIFICATION                                             ║
║  Network:    CheckChain Fabric — agreements-channel                  ║
║  Chaincode:  checkchain-cc                                           ║
║  TX Hash:    c4e6f8a0c2e4f6a8c0e2f4a6c8e0f2a4c6e8f0a2c4e6f8a0c2e4  ║
║  Block:      #78                                                     ║
║  Released:   2026-01-20T15:00:00Z                                    ║
║                                                                      ║
║  Verify at: https://verify.checkchain.example.com/receipt/c4e6f8a0  ║
║                                                                      ║
║  ─────────────────────────────────────────────────────────────────   ║
║  Settlement Bank: SettlementBankOrgMSP                               ║
║  This document is cryptographic proof of payment and condition       ║
║  fulfillment recorded on a permissioned Hyperledger Fabric ledger.   ║
╚══════════════════════════════════════════════════════════════════════╝
```

**CSS Print rules:**
- Hide navigation, header, print/download buttons
- Full-page receipt card at 8.5×11" or A4
- Border: `2px solid #1a1a2e` with inner accent line
- Font: Courier New for check fields, sans-serif for labels
- Amount in bold large type
- QR code generated client-side via `qrcode.react`

---

## 8. AI: Draft Conditions

**Route:** `/ai/draft-conditions`  
**Purpose:** Standalone tool — paste requirements, get conditions JSON.

```
┌─────────────────────────────────────────────────────────────────────┐
│  ✨ AI Clause Drafter                                               │
│                                                                     │
│  Describe your payment requirements in plain English:               │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ I need to pay my plumber $2,800 for fixing the burst pipe.   │  │
│  │ They need to complete the repair, get it pressure-tested,    │  │
│  │ and submit a final receipt from the parts supplier.          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Amount: [$2,800.00]  Currency: [USD]  Payee type: [Contractor ▼]  │
│                                                                     │
│                              [Generate Conditions]                  │
│                                                                     │
│  ─────────────────── GENERATED CONDITIONS ───────────────────────  │
│                                                                     │
│  Release Mode: MANUAL                                               │
│                                                                     │
│  COND-001 [DELIVERABLE]                                             │
│  Photographic evidence of completed pipe repair with visible        │
│  connections and surrounding area                                   │
│                                                                     │
│  COND-002 [DELIVERABLE]                                             │
│  Pressure test report confirming no leaks at ≥ X PSI               │
│  (clarification: specify required pressure rating)                  │
│                                                                     │
│  COND-003 [DELIVERABLE]                                             │
│  Parts supplier receipt showing materials used                      │
│                                                                     │
│  ⚠️ AI-generated. Review carefully. Not legal advice.              │
│                                                                     │
│  [Use These Conditions →] (pre-fills new agreement wizard)          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. Settings / KYC

**Route:** `/settings/kyc`

```
┌─────────────────────────────────────────────────────────────────────┐
│  Identity Verification                                              │
│                                                                     │
│  Status: ● NOT VERIFIED                                             │
│                                                                     │
│  To fund agreements, you must complete identity verification        │
│  through our Settlement Bank partner. This is required by law.      │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Full Legal Name   [                                        ]  │  │
│  │ Date of Birth     [                                        ]  │  │
│  │ Government ID     [Upload front] [Upload back]              │  │
│  │ Business Name     [                       ] (if applicable) │  │
│  │ EIN / SSN (last 4) [                                       ] │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  [Submit for Verification]                                          │
│                                                                     │
│  Verification is processed by our Settlement Bank partner.          │
│  Typically completes in 1 business day.                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 10. Component Library Notes

| Component | Source | Usage |
|-----------|--------|-------|
| `Button` | shadcn/ui | Primary, secondary, destructive variants |
| `Card` | shadcn/ui | Agreement list items, stat cards |
| `Dialog` | shadcn/ui | Attestation submission, dispute modal |
| `Form` | shadcn/ui + react-hook-form | All forms with zod validation |
| `Badge` | shadcn/ui | Status chips (FUNDED, RELEASED, DISPUTED) |
| `Tabs` | shadcn/ui | Agreement detail sections |
| `Progress` | shadcn/ui | Condition satisfaction progress bar |
| `Skeleton` | shadcn/ui | Loading states |
| `Toast` | shadcn/ui (Sonner) | Success/error notifications |
| `QRCode` | `qrcode.react` | Receipt verification QR |
| `FileUpload` | Custom (react-dropzone) | Evidence file upload |

**Status badge colors:**
- `DRAFT` — gray
- `FUNDED` — blue
- `PENDING_RELEASE` — amber
- `RELEASED` — green
- `DISPUTED` — red
- `CANCELED` — gray (muted)
- `SETTLED` — purple
