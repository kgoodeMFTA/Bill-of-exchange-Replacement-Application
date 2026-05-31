# Product Requirements Document — CheckChain v1

**Product:** CheckChain — AI-Powered Digital Check Replacement on Hyperledger Fabric  
**Version:** 1.0  
**Status:** Draft  
**Date:** 2026-01-15  
**Author:** Product Team  

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Personas](#2-personas)
3. [User Stories & Acceptance Criteria](#3-user-stories--acceptance-criteria)
4. [Feature Scope](#4-feature-scope)
5. [Success Metrics](#5-success-metrics)
6. [Regulatory Landscape](#6-regulatory-landscape)
7. [Phased Roadmap](#7-phased-roadmap)

---

## 1. Problem Statement

Paper checks remain the dominant payment instrument for contractors, landlords, and small businesses handling conditional payments — situations where funds must be released upon proof of completion, not immediately. The existing workarounds are:

- **Personal checks:** No enforcement; payee can deposit before conditions met.
- **Escrow services:** Expensive (1–3% of transaction), slow, require attorneys or title companies.
- **Wire transfers:** Irreversible, no condition enforcement, no receipt standardization.
- **Stablecoins / DeFi:** Regulatory exposure, permissionless blockchains inappropriate for business-to-business use, counterparty trust issues.

**CheckChain** replaces paper checks and informal escrow with cryptographically enforced payment agreements on a permissioned Hyperledger Fabric network. Funds (represented as tokenized deposit claims against a licensed Settlement Bank) are locked at agreement creation and released only when encoded conditions are satisfied — verified by the payee, a notary oracle, or multi-party attestation.

---

## 2. Personas

### 2.1 Contractor / Payee (Primary)

**Profile:** Independent contractor, general contractor, or service provider. Revenue $80K–$500K/year. Manages 5–30 active payment agreements simultaneously. Pain: clients withhold payment after work is delivered; no cryptographic proof of delivery.

**Goals:**
- Submit digital proof of work (photos, signed documents, file hashes) that triggers automatic payment.
- Get a printable, legally credible receipt equivalent to a cashier's check.
- See real-time status of all outstanding agreements.

**Tech comfort:** Moderate. Uses QuickBooks, DocuSign. Not a crypto native.

### 2.2 Landlord / Payer (Primary)

**Profile:** Individual landlord or small property management firm. Manages 2–50 units. Pain: security deposit disputes, repair contractor payment disputes, written evidence hard to produce.

**Goals:**
- Create a payment agreement with release conditions (e.g., inspection passed, repairs completed).
- Know funds are locked and cannot be redirected until conditions are met or agreement canceled.
- Cancel agreement and recover funds if contractor fails.

**Tech comfort:** Low-moderate. Uses Zillow, basic banking apps.

### 2.3 Small Business / Both Sides (Primary)

**Profile:** SMB with 1–50 employees. Uses payments for vendor contracts, milestone-based project payments, subcontractor releases. Pain: accounts payable timing, dispute resolution costs, audit trail gaps.

**Goals:**
- Structured milestone-based payment with auditable trail.
- AI-assisted generation of clear release conditions from plain-English scope of work.
- Integration potential with existing accounting workflows.

### 2.4 Settlement Bank Org (Infrastructure)

**Profile:** Licensed bank or credit union participating in the Fabric network as the Settlement Bank Organization. Responsible for minting/burning tokenized deposit claims (CashTokens) that represent USD held in an omnibus account.

**Goals:**
- Maintain 1:1 backing of all outstanding CashTokens with actual USD deposits.
- Comply with BSA/AML/KYC requirements for all participants.
- Burn tokens upon fund release and initiate ACH/wire settlement to payee's external bank.

### 2.5 Arbitrator Org (Support)

**Profile:** Designated neutral party (e.g., industry association, legal firm, title company) enrolled in the Fabric network. Resolves disputed agreements.

**Goals:**
- Review on-chain attestation evidence and off-chain supporting documents.
- Issue a resolution decision recorded on-chain.
- Never hold or control funds.

---

## 3. User Stories & Acceptance Criteria

### Epic 1: Agreement Creation

| # | User Story | Acceptance Criteria |
|---|-----------|-------------------|
| US-001 | As a payer, I want to create a payment agreement with release conditions so that the payee is paid only upon verified fulfillment. | Agreement created with `DRAFT` status; conditions JSON validated against schema; agreement ID (e.g., AGT-2026-0001) assigned; payer and payee MSP IDs recorded. |
| US-002 | As a payer, I want to use AI to generate release conditions from a plain-English description so that I don't need legal expertise. | Clause drafter returns structured conditions JSON; payer can edit before submission; AI output labeled as "AI-suggested, not legal advice." |
| US-003 | As any party, I want to see a plain-English explanation of any agreement so that I understand what I'm agreeing to. | AI explainer returns summary under 200 words; no technical jargon; links to original terms. |

### Epic 2: Funding

| # | User Story | Acceptance Criteria |
|---|-----------|-------------------|
| US-010 | As a payer, I want to fund an agreement by locking CashTokens so that the payee knows funds are committed. | Agreement transitions from `DRAFT` → `FUNDED`; payer token balance decremented; escrow account on-chain incremented; chaincode event emitted. |
| US-011 | As a payee, I want to see that an agreement is funded before I begin work. | Agreement detail page shows `FUNDED` status with transaction hash and block number. |

### Epic 3: Attestation

| # | User Story | Acceptance Criteria |
|---|-----------|-------------------|
| US-020 | As a payee, I want to submit proof of condition fulfillment so that funds are released. | SHA-256 hash of evidence file recorded on-chain; attestation record linked to agreement; agreement transitions to `PENDING_RELEASE` or `RELEASED` based on release mode. |
| US-021 | As a notary/oracle org, I want to co-sign an attestation so that multi-party conditions are enforced. | Multi-party attestation requires all designated signers; chaincode validates all required MSP IDs have attested. |

### Epic 4: Release, Dispute, Cancel

| # | User Story | Acceptance Criteria |
|---|-----------|-------------------|
| US-030 | As a payee (auto-release), I want funds released automatically when conditions are met so that I don't have to wait for payer approval. | Chaincode evaluates conditions; if all satisfied and `releaseMode=AUTO`, tokens transferred to payee; agreement moves to `RELEASED`. |
| US-031 | As a payer, I want to dispute a release so that I can contest an invalid attestation. | Dispute record created; agreement moves to `DISPUTED`; arbitrator org notified via event; funds frozen until resolution. |
| US-032 | As a payer, I want to cancel an unfunded agreement at any time so that I can withdraw before committing funds. | Agreement in `DRAFT` canceled with `CANCELED` status; no funds involved; both parties notified. |
| US-033 | As either party, I want to cancel a funded agreement by mutual signature so that funds return to the payer. | Requires both payer and payee to submit cancel signatures; on second signature, tokens returned to payer; agreement moved to `CANCELED`. |

### Epic 5: Receipts

| # | User Story | Acceptance Criteria |
|---|-----------|-------------------|
| US-040 | As a payee, I want a cryptographically signed digital cashier's check receipt so that I have legal-quality proof of payment. | Receipt includes payer, payee, amount in words and numerals, agreement ID, chaincode tx hash, block number, timestamps, QR code; printable HTML/CSS format; PDF-exportable. |

### Epic 6: AI Features

| # | User Story | Acceptance Criteria |
|---|-----------|-------------------|
| US-050 | As a payer, I want AI risk scoring of my counterparty and proposed conditions so that I can assess deal risk before funding. | Risk score 1–100 with breakdown (counterparty history, condition clarity, amount, timeline); prominently labeled "AI-generated, not financial advice." |
| US-051 | As an arbitrator, I want an AI summary of both sides' attestations so that I can make a faster, informed decision. | Dispute assistant summarizes attestations from both sides; suggests resolution path; cites specific evidence hashes; includes confidence level. |

---

## 4. Feature Scope

### In Scope (v1)

- Agreement lifecycle: Create, Fund, Attest, Release, Dispute, Cancel
- Condition types: deliverable-based (file hash), date-based (block timestamp), milestone-based (sequential attestations), multi-party attestation
- CashToken: USD-denominated fungible token minted by Settlement Bank Org
- Receipt generation: Digital Cashier's Check (HTML/CSS printable + PDF)
- AI: Clause drafter, agreement explainer, risk scorer, dispute assistant, receipt narrator
- REST API gateway with Fabric SDK integration
- Web portal: payer + payee flows
- Private data collections for sensitive agreement terms
- KYC stub (org-level enrollment via Fabric CA)

### Out of Scope (v1)

- Cross-network / cross-channel payments
- Native mobile apps (iOS/Android)
- Direct bank account ACH pull (v1 requires manual deposit to Settlement Bank to receive tokens)
- Recurring/subscription payment agreements
- Secondary market transfer of agreements
- Stablecoin or public blockchain integration
- International currencies (v1 is USD-only)
- Tax form generation (1099-K, etc.)
- Payroll / employment payments (regulatory category separate from commercial B2B)

---

## 5. Success Metrics

| Metric | Target (6-month post-launch) |
|--------|------------------------------|
| Agreements created / month | 500 |
| Agreement completion rate (RELEASED / total non-canceled) | ≥ 85% |
| Average time to release after attestation (auto mode) | < 5 minutes |
| Dispute rate | < 3% of funded agreements |
| AI clause drafter adoption | ≥ 60% of new agreements |
| Receipt download / agreement | ≥ 90% |
| Net Promoter Score (contractor persona) | ≥ 45 |

---

## 6. Regulatory Landscape

### 6.1 The Token Model: Tokenized Deposit Claim

**CheckChain v1 uses a "tokenized deposit claim" model, not a stablecoin or cryptocurrency.**

The CashToken is a programmable representation of a USD deposit held in an FDIC-insured omnibus account at the Settlement Bank Org. It is:

- **Not a cryptocurrency.** It has no independent value, no mining, and no public blockchain.
- **Not a stablecoin.** It is a claim against a specific institution's deposit, not an algorithmic or collateralized peg.
- **Not a security.** It conveys no ownership, dividend rights, or speculative interest.
- **Functionally analogous to a cashier's check.** The bank holds the funds; the token is a record of the bank's obligation.

This characterization directly shapes the regulatory analysis below. Legal counsel and compliance review are required before any production deployment.

---

### 6.2 Money Transmitter Licensing (FinCEN / State)

**FinCEN MSB Registration**

Under 31 U.S.C. § 5330 and 31 C.F.R. § 1010.100(ff), a "money transmitter" includes any person that transfers funds on behalf of the public. The Settlement Bank Org, if a licensed depository institution, may be exempt from MSB registration under the bank exemption (31 C.F.R. § 1010.100(ff)(8)(ii)). The platform operator (non-bank) may need to register as an MSB and file a FinCEN Form 107.

**State Money Transmitter Licenses**

48 states + DC require separate state MTL. Notable requirements:

| State | Surety Bond | Net Worth | Notes |
|-------|------------|-----------|-------|
| New York | $500K–$5M | $1M min | BitLicense not applicable (no crypto) |
| California | Variable | $500K min | DBO oversight |
| Texas | $300K–$2M | Varies | TDF |
| Florida | $100K+ | $100K min | OFR |

**Practical Path (v1 MVP):** Partner exclusively with a licensed bank as the Settlement Bank Org. The bank handles all money transmission under its existing charter. The platform provides the technology layer only.

---

### 6.3 UCC Article 3/4 & Article 4A

- **UCC Article 3 (Negotiable Instruments):** Paper checks are governed by Article 3. A digital token that mimics a check's function may not qualify as a "negotiable instrument" under Art. 3 because it lacks a paper writing with a handwritten signature. CheckChain receipts are not negotiable instruments; they are contractual payment records.
- **UCC Article 4 (Bank Deposits and Collections):** Applies to the bank collection process. When the Settlement Bank releases funds via ACH after token burn, Article 4 governs the underlying bank transaction.
- **UCC Article 4A (Funds Transfers):** Governs wholesale electronic funds transfers. If CheckChain's settlement leg uses Fedwire or a bank-to-bank transfer, Article 4A applies to that leg. Importantly, Article 4A provides a comprehensive liability framework that may preempt common law claims.

---

### 6.4 Regulation E (Consumer Electronic Fund Transfers)

Reg E (12 C.F.R. Part 1005) applies when:
- The account is held by or for a consumer (natural person), AND
- The fund transfer is electronic.

**Applicability:** If any payer or payee is a consumer (vs. a business entity), Reg E error resolution procedures (60-day dispute window, 10-day provisional credit), disclosure requirements, and liability limits apply. **v1 should restrict use to business-to-business agreements** and clearly disclaim consumer use in the Terms of Service.

---

### 6.5 BSA/AML/KYC

The Bank Secrecy Act (31 U.S.C. § 5311 et seq.) requires covered entities to maintain AML programs, file Suspicious Activity Reports (SARs), and comply with Customer Identification Program (CIP) rules (31 C.F.R. § 1020.220).

**v1 Approach:**
- Settlement Bank Org runs full KYC/CIP on all participants enrolling for token wallets.
- CheckChain platform stores org-level enrollment status; individual user KYC is delegated to the bank.
- CashToken minting is gated on KYC-verified status recorded in the Fabric CA enrollment certificate.
- Transaction monitoring thresholds: any single agreement > $3,000 triggers enhanced due diligence review by the Settlement Bank; SAR filing for suspicious activity > $5,000 as required.
- CTR filing for cash transactions > $10,000 (relevant when depositing physical cash to fund tokens).

---

### 6.6 FDIC Pass-Through Insurance

The omnibus deposit account at the Settlement Bank qualifies for FDIC pass-through insurance to individual depositors (per 12 C.F.R. § 330.5) **if:**
- The account is properly titled as a custodial account.
- The bank maintains records identifying each beneficial owner's share.
- The beneficial owners are eligible for FDIC coverage (U.S. persons, not excluded categories).

Per-depositor limit: $250,000. Settlement Bank must implement a sub-ledger identifying each participant's share of the omnibus account. This is a banking operations requirement, not a chaincode requirement.

---

### 6.7 FinTech Regulatory Disclaimer

> **CheckChain is a technology platform. It does not provide banking, legal, financial, or money transmission services. The Settlement Bank Org is the regulated entity responsible for deposit-taking and fund transmission. Before deploying in production, obtain independent legal review of licensing requirements in each jurisdiction where the platform will be offered.**

---

## 7. Phased Roadmap

### Phase 1 — Foundation (Months 1–6)
- Core agreement lifecycle (Create, Fund, Attest, Release, Cancel)
- CashToken (USD only)
- Payer + Payee portals
- AI clause drafter + explainer
- Digital Cashier's Check receipt
- Pilot with 3 partner contractors + 2 landlord clients
- Settlement Bank Org: single partner bank in sandbox

### Phase 2 — Dispute & Risk (Months 7–10)
- Dispute workflow with Arbitrator Org
- AI dispute assistant
- Risk scoring v1 (condition clarity + counterparty history)
- Webhook dispatcher for accounting integrations (QuickBooks, Xero)
- Additional condition types: oracle-signed data feeds (e.g., weather, inspection services)

### Phase 3 — Scale & Compliance (Months 11–18)
- Multi-currency support (CAD, EUR via additional Settlement Bank Orgs)
- ACH pull integration (payer funds agreement directly from bank account)
- Mobile-responsive progressive web app
- Full AML transaction monitoring integration (Sardine or Alloy)
- State-by-state compliance review and MTL applications
- SOC 2 Type II audit preparation

### Phase 4 — Ecosystem (Months 19–24)
- Public API for third-party integrations
- Marketplace for arbitration services
- NFT-based proof-of-payment certificates (optional, jurisdiction-permitting)
- Cross-channel agreement linking (multi-step project financing)
