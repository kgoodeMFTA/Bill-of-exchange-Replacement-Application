# API Contracts — CheckChain REST Gateway

**Base URL:** `https://api.checkchain.example.com/api` (production)  
**Dev URL:** `http://localhost:3001/api`  
**Auth:** Bearer JWT in `Authorization` header  
**Content-Type:** `application/json`

---

## Table of Contents

1. [Authentication — /api/auth](#1-authentication)
2. [Agreements — /api/agreements](#2-agreements)
3. [Attestations — /api/attestations](#3-attestations)
4. [Disputes — /api/disputes](#4-disputes)
5. [Tokens — /api/tokens](#5-tokens-bank-only)
6. [Receipts — /api/receipts](#6-receipts)
7. [AI Endpoints — /api/ai](#7-ai-endpoints)
8. [Error Schema](#8-error-schema)

---

## 1. Authentication

### POST /api/auth/login

Exchange credentials for a JWT.

**Request:**
```json
{
  "email": "user@payerorg.example.com",
  "password": "••••••••",
  "orgMSP": "PayerOrgMSP"
}
```

**Response 200:**
```json
{
  "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 3600,
  "user": {
    "userId": "usr_01HX8K2R3N5P6Q7S8T9",
    "email": "user@payerorg.example.com",
    "orgMSP": "PayerOrgMSP",
    "role": "PAYER",
    "kycVerified": true,
    "displayName": "Alex Chen"
  }
}
```

**Response 401:**
```json
{ "error": "INVALID_CREDENTIALS", "message": "Email or password incorrect." }
```

---

### POST /api/auth/refresh

**Request:** `{ "refreshToken": "..." }`  
**Response 200:** `{ "token": "...", "expiresIn": 3600 }`

---

### GET /api/auth/me

Returns current user profile.

**Response 200:**
```json
{
  "userId": "usr_01HX8K2R3N5P6Q7S8T9",
  "email": "user@payerorg.example.com",
  "orgMSP": "PayerOrgMSP",
  "role": "PAYER",
  "kycVerified": true,
  "walletBalance": "12500.00",
  "currency": "USD"
}
```

---

## 2. Agreements

### POST /api/agreements

Create a new payment agreement.

**Request:**
```json
{
  "payeeAddress": "contractor@payeeorg.example.com",
  "payeeMSP": "PayeeOrgMSP",
  "amount": "5000.00",
  "currency": "USD",
  "releaseMode": "AUTO",
  "conditions": [
    {
      "conditionId": "COND-001",
      "type": "DELIVERABLE",
      "description": "Final inspection report signed by licensed inspector",
      "requiredAttestors": ["PayeeOrgMSP"]
    },
    {
      "conditionId": "COND-002",
      "type": "DATE",
      "description": "Completion by 2026-02-28",
      "deadline": "2026-02-28T23:59:59Z"
    }
  ],
  "privateNotes": "Unit 4B bathroom renovation.",
  "expiresAt": "2026-03-15T10:00:00Z"
}
```

**Response 201:**
```json
{
  "agreementId": "AGT-2026-0001",
  "status": "DRAFT",
  "txId": "7a3f2b1c9d8e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b",
  "blockNumber": 23,
  "createdAt": "2026-01-15T10:00:00Z",
  "message": "Agreement created. Share the agreement ID with your payee to allow them to review."
}
```

**Errors:** `400 VALIDATION_ERROR`, `402 INSUFFICIENT_BALANCE`, `409 AGREEMENT_EXISTS`

---

### GET /api/agreements

List agreements for the authenticated user.

**Query params:** `status` (filter), `page` (default 1), `limit` (default 20), `role` (`PAYER|PAYEE`)

**Response 200:**
```json
{
  "agreements": [
    {
      "agreementId": "AGT-2026-0001",
      "payerAddress": "user@payerorg.example.com",
      "payeeAddress": "contractor@payeeorg.example.com",
      "amount": "5000.00",
      "currency": "USD",
      "status": "FUNDED",
      "releaseMode": "AUTO",
      "conditionCount": 2,
      "satisfiedConditionCount": 1,
      "createdAt": "2026-01-15T10:00:00Z",
      "updatedAt": "2026-01-15T14:32:00Z",
      "expiresAt": "2026-03-15T10:00:00Z"
    }
  ],
  "total": 1,
  "page": 1,
  "limit": 20
}
```

---

### GET /api/agreements/:id

Get agreement detail. Returns public fields + private terms if caller is a party.

**Response 200:**
```json
{
  "agreementId": "AGT-2026-0001",
  "payerMSP": "PayerOrgMSP",
  "payerAddress": "user@payerorg.example.com",
  "payerDisplayName": "Alex Chen",
  "payeeMSP": "PayeeOrgMSP",
  "payeeAddress": "contractor@payeeorg.example.com",
  "payeeDisplayName": "Jordan Lee",
  "amount": "5000.00",
  "currency": "USD",
  "status": "FUNDED",
  "releaseMode": "AUTO",
  "escrowAmount": "5000.00",
  "createdAt": "2026-01-15T10:00:00Z",
  "updatedAt": "2026-01-15T14:32:00Z",
  "expiresAt": "2026-03-15T10:00:00Z",
  "conditions": [
    {
      "conditionId": "COND-001",
      "type": "DELIVERABLE",
      "description": "Final inspection report signed by licensed inspector",
      "requiredAttestors": ["PayeeOrgMSP"],
      "satisfied": false
    },
    {
      "conditionId": "COND-002",
      "type": "DATE",
      "description": "Completion by 2026-02-28",
      "deadline": "2026-02-28T23:59:59Z",
      "satisfied": false
    }
  ],
  "attestations": [],
  "disputeId": null,
  "releaseTxId": null,
  "releaseBlockNumber": null,
  "privateNotes": "Unit 4B bathroom renovation."
}
```

---

### POST /api/agreements/:id/fund

Fund a `DRAFT` agreement with CashTokens.

**Request:**
```json
{ "amount": "5000.00" }
```

**Response 200:**
```json
{
  "agreementId": "AGT-2026-0001",
  "status": "FUNDED",
  "txId": "9b2e4d6f8a0c2e4f6a8c0e2f4a6c8e0f2a4c6e8f0a2c4e6f8a0c2e4f6a8c0e2f",
  "blockNumber": 31,
  "escrowAmount": "5000.00",
  "message": "Funds locked in escrow. Payee has been notified."
}
```

---

### POST /api/agreements/:id/cancel

Cancel an agreement (draft: payer only; funded: mutual consent required).

**Request:**
```json
{ "reason": "Project scope changed, agreement no longer needed." }
```

**Response 200:**
```json
{
  "agreementId": "AGT-2026-0001",
  "status": "CANCELED",
  "canceledAt": "2026-01-16T08:00:00Z",
  "refundedAmount": "5000.00",
  "message": "Agreement canceled. Funds returned to payer wallet."
}
```

---

### POST /api/agreements/:id/release

Manually release funds (MANUAL mode, payer approves).

**Request:** `{}` (no body needed)

**Response 200:**
```json
{
  "agreementId": "AGT-2026-0001",
  "status": "RELEASED",
  "txId": "c4e6f8a0c2e4f6a8c0e2f4a6c8e0f2a4c6e8f0a2c4e6f8a0c2e4f6a8c0e2f4a6",
  "blockNumber": 78,
  "releasedAmount": "5000.00",
  "releasedAt": "2026-01-20T15:00:00Z"
}
```

---

## 3. Attestations

### POST /api/attestations

Submit proof of condition fulfillment. Accepts `multipart/form-data`.

**Request (multipart):**
```
agreementId: AGT-2026-0001
conditionId: COND-001
evidenceFile: <binary file upload>
attestationNote: "Inspection completed by John Smith, License #12345"
```

**Response 201:**
```json
{
  "attestationId": "ATT-2026-0001-COND-001",
  "agreementId": "AGT-2026-0001",
  "conditionId": "COND-001",
  "evidenceHash": "a4f2e8c1d9b3f7e2a5c8d4e9f1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0",
  "evidenceURI": "s3://checkchain-evidence/AGT-2026-0001/inspection-report-20260120.pdf",
  "submittedAt": "2026-01-20T09:15:00Z",
  "txId": "7a3f2b1c9d8e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b",
  "blockNumber": 47,
  "agreementNewStatus": "RELEASED",
  "message": "Attestation recorded. All conditions satisfied — funds released automatically."
}
```

---

### GET /api/attestations/:agreementId

List all attestations for an agreement.

**Response 200:**
```json
{
  "attestations": [
    {
      "attestationId": "ATT-2026-0001-COND-001",
      "conditionId": "COND-001",
      "attestorAddress": "contractor@payeeorg.example.com",
      "evidenceHash": "a4f2e8c1d9b3f7e2a5c8d4e9f1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0",
      "evidenceURI": "s3://checkchain-evidence/AGT-2026-0001/inspection-report-20260120.pdf",
      "submittedAt": "2026-01-20T09:15:00Z",
      "txId": "7a3f2b1c9d8e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b",
      "blockNumber": 47
    }
  ]
}
```

---

## 4. Disputes

### POST /api/disputes

Raise a dispute on a funded agreement.

**Request:**
```json
{
  "agreementId": "AGT-2026-0001",
  "reason": "The submitted inspection report does not cover bathroom scope items 3 and 4 as specified in the agreement."
}
```

**Response 201:**
```json
{
  "disputeId": "DISP-2026-0001",
  "agreementId": "AGT-2026-0001",
  "status": "OPEN",
  "raisedAt": "2026-01-22T11:00:00Z",
  "txId": "2f4a6c8e0f2a4c6e8f0a2c4e6f8a0c2e4f6a8c0e2f4a6c8e0f2a4c6e8f0a2c4e",
  "message": "Dispute raised. Arbitrator has been notified."
}
```

---

### GET /api/disputes/:id

Get dispute detail.

**Response 200:**
```json
{
  "disputeId": "DISP-2026-0001",
  "agreementId": "AGT-2026-0001",
  "status": "OPEN",
  "raisedByMSP": "PayerOrgMSP",
  "raisedByAddress": "user@payerorg.example.com",
  "reason": "The submitted inspection report does not cover bathroom scope items 3 and 4.",
  "raisedAt": "2026-01-22T11:00:00Z",
  "arbitratorMSP": "ArbitratorOrgMSP",
  "decision": null,
  "resolution": null,
  "resolvedAt": null,
  "attestations": [ { "..." : "..." } ]
}
```

---

### POST /api/disputes/:id/resolve

Resolve a dispute (Arbitrator role only).

**Request:**
```json
{
  "decision": "SPLIT",
  "resolution": "The inspection report covers 3 of 4 scope items. Payee receives 75% of funds for completed work.",
  "splitRatio": "75:25"
}
```

**Response 200:**
```json
{
  "disputeId": "DISP-2026-0001",
  "status": "RESOLVED",
  "decision": "SPLIT",
  "agreementStatus": "SETTLED",
  "resolvedAt": "2026-01-25T14:00:00Z",
  "txId": "e8f0a2c4e6f8a0c2e4f6a8c0e2f4a6c8e0f2a4c6e8f0a2c4e6f8a0c2e4f6a8c0"
}
```

---

## 5. Tokens (Bank Only)

### POST /api/tokens/mint

Mint CashTokens for a user. Requires `SettlementBankOrgMSP` JWT.

**Request:**
```json
{
  "ownerMSP": "PayerOrgMSP",
  "ownerAddress": "user@payerorg.example.com",
  "amount": "10000.00",
  "currency": "USD",
  "referenceId": "DEPOSIT-2026-0042"
}
```

**Response 201:**
```json
{
  "txId": "b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2",
  "ownerAddress": "user@payerorg.example.com",
  "mintedAmount": "10000.00",
  "newBalance": "22500.00",
  "totalSupply": "87500.00"
}
```

---

### POST /api/tokens/burn

Burn CashTokens after settlement. Requires `SettlementBankOrgMSP` JWT.

**Request:**
```json
{
  "ownerMSP": "PayeeOrgMSP",
  "ownerAddress": "contractor@payeeorg.example.com",
  "amount": "5000.00",
  "referenceId": "SETTLEMENT-2026-0018"
}
```

**Response 200:**
```json
{
  "txId": "c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4",
  "burnedAmount": "5000.00",
  "newBalance": "0.00",
  "totalSupply": "82500.00"
}
```

---

### GET /api/tokens/balance

Get authenticated user's token balance.

**Response 200:**
```json
{
  "ownerAddress": "user@payerorg.example.com",
  "balance": "12500.00",
  "currency": "USD",
  "kycVerified": true
}
```

---

### GET /api/tokens/total-supply

Get total supply. Bank role only.

**Response 200:**
```json
{ "totalSupply": "82500.00", "currency": "USD" }
```

---

## 6. Receipts

### GET /api/receipts/:txId

Get a digital cashier's check receipt for a release transaction.

**Response 200:**
```json
{
  "receiptId": "RCP-AGT-2026-0001",
  "agreementId": "AGT-2026-0001",
  "txId": "c4e6f8a0c2e4f6a8c0e2f4a6c8e0f2a4c6e8f0a2c4e6f8a0c2e4f6a8c0e2f4a6",
  "blockNumber": 78,
  "channelId": "agreements-channel",
  "payer": {
    "displayName": "Alex Chen",
    "address": "user@payerorg.example.com",
    "orgMSP": "PayerOrgMSP"
  },
  "payee": {
    "displayName": "Jordan Lee",
    "address": "contractor@payeeorg.example.com",
    "orgMSP": "PayeeOrgMSP"
  },
  "amount": "5000.00",
  "amountWords": "Five Thousand and 00/100",
  "currency": "USD",
  "releasedAt": "2026-01-20T15:00:00Z",
  "createdAt": "2026-01-15T10:00:00Z",
  "verificationUrl": "https://verify.checkchain.example.com/receipt/c4e6f8a0c2e4f6a8c0e2f4a6c8e0f2a4",
  "chaincodeName": "checkchain-cc",
  "networkId": "checkchain-mainnet",
  "settlementBankMSP": "SettlementBankOrgMSP",
  "signature": "MEQCIBx...",
  "conditions": [
    {
      "conditionId": "COND-001",
      "description": "Final inspection report signed by licensed inspector",
      "evidenceHash": "a4f2e8c1d9b3f7e2a5c8d4e9f1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0",
      "satisfiedAt": "2026-01-20T09:15:00Z"
    }
  ]
}
```

---

## 7. AI Endpoints

### POST /api/ai/draft-conditions

Generate structured release conditions from plain-English requirements.

**Request:**
```json
{
  "requirementsText": "I need to pay my contractor $5,000 for bathroom renovation. They need to finish by end of February, and I need a signed inspection report and before/after photos before I release payment.",
  "agreementContext": {
    "amount": "5000.00",
    "currency": "USD",
    "payeeType": "contractor"
  }
}
```

**Response 200:**
```json
{
  "conditions": [
    {
      "conditionId": "COND-001",
      "type": "DELIVERABLE",
      "description": "Signed inspection report from a licensed inspector confirming renovation completion",
      "requiredAttestors": ["PayeeOrgMSP"],
      "aiGenerated": true
    },
    {
      "conditionId": "COND-002",
      "type": "DELIVERABLE",
      "description": "Before and after photographic documentation of the bathroom renovation",
      "requiredAttestors": ["PayeeOrgMSP"],
      "aiGenerated": true
    },
    {
      "conditionId": "COND-003",
      "type": "DATE",
      "description": "All work completed by February 28, 2026",
      "deadline": "2026-02-28T23:59:59Z",
      "aiGenerated": true
    }
  ],
  "summary": "3 conditions generated: 2 deliverables (inspection report, photos) and 1 date deadline. All conditions must be met before funds release.",
  "disclaimer": "AI-generated conditions are suggestions only. Review carefully and consult legal counsel for high-value agreements. CheckChain does not provide legal advice.",
  "model": "gpt-4o",
  "generatedAt": "2026-01-15T09:55:00Z"
}
```

---

### POST /api/ai/explain-agreement

Explain an agreement in plain English.

**Request:**
```json
{ "agreementId": "AGT-2026-0001" }
```

**Response 200:**
```json
{
  "summary": "This is a $5,000 payment agreement between Alex Chen (payer) and Jordan Lee (contractor). Alex has locked $5,000 in escrow. Jordan will receive the funds automatically once they submit a signed inspection report and before/after renovation photos — both due by February 28, 2026. If Alex believes the conditions haven't been met, they can raise a dispute within 30 days of attestation submission.",
  "keyPoints": [
    "Funds are locked — Alex cannot access them without Jordan's consent or arbitration.",
    "Payment releases automatically when both deliverables are submitted.",
    "Agreement expires March 15, 2026 — after that, funds return to Alex.",
    "Disputes go to the Arbitrator org for resolution."
  ],
  "riskFlags": [],
  "disclaimer": "This explanation is AI-generated and does not constitute legal advice.",
  "model": "gpt-4o"
}
```

---

### POST /api/ai/risk-score

Score the risk of an agreement before funding.

**Request:**
```json
{
  "agreementId": "AGT-2026-0001",
  "counterpartyAddress": "contractor@payeeorg.example.com"
}
```

**Response 200:**
```json
{
  "overallScore": 28,
  "riskLevel": "LOW",
  "breakdown": {
    "counterpartyHistory": {
      "score": 15,
      "label": "LOW",
      "notes": "Counterparty has 12 prior completed agreements, 0 disputes."
    },
    "conditionClarity": {
      "score": 20,
      "label": "LOW",
      "notes": "Conditions are specific and measurable. Inspection report provides objective evidence."
    },
    "amountExposure": {
      "score": 30,
      "label": "MEDIUM",
      "notes": "$5,000 is within normal range for contractor agreements. No unusual concentration risk."
    },
    "timelineRisk": {
      "score": 25,
      "label": "LOW",
      "notes": "44-day completion window is reasonable for bathroom renovation scope."
    }
  },
  "recommendations": [
    "Consider adding a milestone condition (e.g., rough work inspection at 50% completion).",
    "Ensure inspection report requirement specifies the inspector's license number."
  ],
  "disclaimer": "Risk scores are AI-generated estimates based on available data and do not constitute financial advice. Past performance does not guarantee future results.",
  "model": "gpt-4o"
}
```

---

## 8. Error Schema

All error responses follow this structure:

```json
{
  "error": "ERROR_CODE",
  "message": "Human-readable description.",
  "details": { "field": "conditions[0].type", "issue": "must be DELIVERABLE | DATE | MILESTONE | MULTIPARTY" },
  "requestId": "req_01HX8K2R3N5P6Q7S8T9",
  "timestamp": "2026-01-15T10:00:00Z"
}
```

| HTTP Status | Error Code | Meaning |
|-------------|-----------|---------|
| 400 | `VALIDATION_ERROR` | Request body fails schema validation |
| 401 | `UNAUTHORIZED` | Missing or invalid JWT |
| 403 | `FORBIDDEN` | Authenticated but insufficient role/org |
| 404 | `NOT_FOUND` | Resource does not exist |
| 409 | `CONFLICT` | Resource already exists or state conflict |
| 402 | `INSUFFICIENT_BALANCE` | CashToken balance too low |
| 422 | `FABRIC_ERROR` | Chaincode rejected transaction |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Unexpected server error |
| 503 | `FABRIC_UNAVAILABLE` | Fabric network unreachable |
