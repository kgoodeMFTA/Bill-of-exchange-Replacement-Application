# System Architecture — CheckChain

**Version:** 1.0  
**Date:** 2026-01-15

---

## Table of Contents

1. [Component Overview](#1-component-overview)
2. [Hyperledger Fabric Network Design](#2-hyperledger-fabric-network-design)
3. [Component Architecture Diagram](#3-component-architecture-diagram)
4. [Data Flow Diagrams](#4-data-flow-diagrams)
5. [Technology Stack](#5-technology-stack)
6. [Security Architecture](#6-security-architecture)
7. [Deployment Architecture](#7-deployment-architecture)

---

## 1. Component Overview

CheckChain is composed of five layers:

| Layer | Technology | Responsibility |
|-------|-----------|---------------|
| **Blockchain** | Hyperledger Fabric 2.5 | Immutable agreement ledger, token accounting, access control |
| **Chaincode** | Go + fabric-contract-api-go | AgreementContract, CashTokenContract business logic |
| **Backend Gateway** | Node.js 20 / TypeScript / Express | REST API, Fabric SDK gateway, event listener, identity management |
| **Frontend** | Next.js 14 / Tailwind / shadcn/ui | Payer + Payee portals, AI tools, receipt viewer |
| **AI Service** | LLM Gateway (OpenAI / Anthropic / Mock) | Clause generation, explainability, risk scoring, dispute assistance |

Off-chain supporting services:

| Service | Technology | Responsibility |
|---------|-----------|---------------|
| **Database** | PostgreSQL 16 | User accounts, agreement cache, evidence metadata, audit log |
| **Object Storage** | AWS S3 (or MinIO for dev) | Evidence files; only SHA-256 hashes go on-chain |
| **Identity / CA** | Hyperledger Fabric CA | Org-level enrollment, MSP certificate management |
| **KYC Service** | Stub (v1); Alloy/Sardine (v2) | Identity verification gating token wallet activation |

---

## 2. Hyperledger Fabric Network Design

### 2.1 Organizations

| Org | MSP ID | Role | Peers |
|-----|--------|------|-------|
| PayerOrg | PayerOrgMSP | Represents payer-side participants | 2 peers (peer0, peer1) |
| PayeeOrg | PayeeOrgMSP | Represents payee-side participants | 2 peers (peer0, peer1) |
| SettlementBankOrg | SettlementBankOrgMSP | Licensed bank; mints/burns CashTokens | 2 peers (peer0, peer1) |
| ArbitratorOrg | ArbitratorOrgMSP | Neutral dispute resolution | 2 peers (peer0, peer1) |

> **MVP Note:** The dev network scaffold in `fabric-network/` starts with PayerOrg + SettlementBankOrg only. PayeeOrg and ArbitratorOrg are added by modifying `configtx.yaml` following the documented procedure.

### 2.2 Ordering Service

- **Type:** RAFT (crash fault tolerant)
- **Orderer nodes:** 3 orderer nodes (OrdererOrg) for production; 1 orderer in dev network
- **Block parameters:** `BatchTimeout: 2s`, `MaxMessageCount: 500`, `AbsoluteMaxBytes: 99MB`

### 2.3 Channel Design

```
Channel: agreements-channel
  ├── Members: PayerOrg, PayeeOrg, SettlementBankOrg, ArbitratorOrg
  ├── Chaincode: checkchain-cc (AgreementContract + CashTokenContract)
  └── Private Data Collections:
        ├── agreementTermsPDC       — sensitive contract clauses (PayerOrg + PayeeOrg only)
        ├── tokenBalancesPDC        — individual wallet balances (SettlementBankOrg only)
        └── kycStatusPDC            — KYC verification records (SettlementBankOrg + read by all for gating)
```

### 2.4 Private Data Collections

| Collection | Members | Purpose | Block-to-Live |
|-----------|---------|---------|---------------|
| `agreementTermsPDC` | PayerOrg, PayeeOrg | Full condition text, private notes, legal attachments | 0 (infinite) |
| `tokenBalancesPDC` | SettlementBankOrg | Individual CashToken balances (privacy) | 0 |
| `kycStatusPDC` | SettlementBankOrg (write), all (read via function) | KYC approval status per participant | 0 |

### 2.5 Endorsement Policies

| Chaincode Function | Endorsement Policy |
|-------------------|--------------------|
| `CreateAgreement` | `AND(PayerOrgMSP.peer, PayeeOrgMSP.peer)` |
| `FundAgreement` | `AND(PayerOrgMSP.peer, SettlementBankOrgMSP.peer)` |
| `SubmitAttestation` | `OR(PayeeOrgMSP.peer, ArbitratorOrgMSP.peer)` |
| `ReleaseFunds` | `AND(SettlementBankOrgMSP.peer, OR(PayeeOrgMSP.peer, ArbitratorOrgMSP.peer))` |
| `RaiseDispute` | `OR(PayerOrgMSP.peer, PayeeOrgMSP.peer)` |
| `ResolveDispute` | `AND(ArbitratorOrgMSP.peer, SettlementBankOrgMSP.peer)` |
| `Mint` / `Burn` | `AND(SettlementBankOrgMSP.peer)` (bank-only) |
| `CancelAgreement` | `AND(PayerOrgMSP.peer, PayeeOrgMSP.peer)` |

---

## 3. Component Architecture Diagram

```mermaid
graph TB
    subgraph "Client Layer"
        FE["Frontend\nNext.js 14\n/dashboard /agreements /receipts"]
    end

    subgraph "API Gateway Layer"
        API["Backend API Gateway\nNode.js / TypeScript / Express\nPort 3001"]
        AIS["AI Service\nLLM Gateway\n(OpenAI | Anthropic | Mock)"]
    end

    subgraph "Data Layer"
        PG["PostgreSQL\nUsers, Cache, Audit"]
        S3["S3 / MinIO\nEvidence Files"]
    end

    subgraph "Fabric Network — agreements-channel"
        subgraph "PayerOrg"
            PP0["peer0.payerorg"]
            PP1["peer1.payerorg"]
        end
        subgraph "PayeeOrg"
            PYP0["peer0.payeeorg"]
            PYP1["peer1.payeeorg"]
        end
        subgraph "SettlementBankOrg"
            SBP0["peer0.settlementbankorg"]
            SBP1["peer1.settlementbankorg"]
        end
        subgraph "ArbitratorOrg"
            ARP0["peer0.arbitratororg"]
            ARP1["peer1.arbitratororg"]
        end
        ORD["RAFT Orderer\n3 nodes"]
        CC["checkchain-cc\nAgreementContract\nCashTokenContract"]
    end

    subgraph "Identity"
        CA["Fabric CA\nPer-Org MSP"]
        KYC["KYC Service\n(stub v1)"]
    end

    FE -->|HTTPS REST| API
    API -->|fabric-network SDK| PP0
    API -->|fabric-network SDK| SBP0
    API -->|Prisma ORM| PG
    API -->|S3 SDK| S3
    API -->|HTTP| AIS
    PP0 <-->|gossip| PP1
    PP0 <-->|endorsement| PYP0
    PP0 <-->|endorsement| SBP0
    SBP0 <-->|endorsement| ARP0
    PP0 -->|submit tx| ORD
    ORD -->|deliver blocks| PP0
    ORD -->|deliver blocks| PYP0
    ORD -->|deliver blocks| SBP0
    ORD -->|deliver blocks| ARP0
    CC -.->|installed on| PP0
    CC -.->|installed on| PYP0
    CC -.->|installed on| SBP0
    CC -.->|installed on| ARP0
    CA -->|enroll| PP0
    CA -->|enroll| SBP0
    KYC -->|KYC status| API
```

---

## 4. Data Flow Diagrams

### 4.1 Create Agreement Flow

```mermaid
sequenceDiagram
    actor Payer
    participant FE as Frontend
    participant API as Backend Gateway
    participant AI as AI Service
    participant CC as Chaincode
    participant PG as PostgreSQL

    Payer->>FE: Fill agreement wizard (parties, amount, conditions)
    FE->>AI: POST /api/ai/draft-conditions (plain-English requirements)
    AI-->>FE: conditions JSON (suggested)
    Payer->>FE: Review + confirm conditions
    FE->>API: POST /api/agreements
    API->>API: Validate request, check JWT
    API->>CC: CreateAgreement(agreementId, payerMSP, payeeMSP, amount, currency, conditions, expiration)
    Note over CC: Endorsement: PayerOrg + PayeeOrg peers
    CC->>CC: Validate args, write Agreement{status:DRAFT} to world state
    CC-->>API: TxID, blockNumber
    API->>PG: Insert agreement_cache row
    API-->>FE: 201 { agreementId, txId, status: "DRAFT" }
    FE-->>Payer: Agreement created confirmation
```

### 4.2 Fund Agreement Flow

```mermaid
sequenceDiagram
    actor Payer
    participant FE as Frontend
    participant API as Backend Gateway
    participant CC as Chaincode
    participant SB as SettlementBankOrg Peer

    Payer->>FE: Click "Fund Agreement"
    FE->>API: POST /api/agreements/:id/fund { amount }
    API->>SB: Verify payer CashToken balance >= amount
    API->>CC: FundAgreement(agreementId, amount)
    Note over CC: Endorsement: PayerOrg + SettlementBankOrg
    CC->>CC: Transfer tokens: payer wallet → escrow account
    CC->>CC: Update Agreement{status:FUNDED, escrowAmount}
    CC->>CC: Emit AgreementFunded event
    CC-->>API: TxID, blockNumber
    API-->>FE: 200 { status: "FUNDED", txId, blockNumber }
    FE-->>Payer: Agreement funded — payee notified
```

### 4.3 Attest and Release Flow

```mermaid
sequenceDiagram
    actor Payee
    participant FE as Frontend
    participant API as Backend Gateway
    participant S3 as Object Storage
    participant CC as Chaincode

    Payee->>FE: Upload evidence file(s)
    FE->>API: POST /api/attestations (multipart)
    API->>API: Compute SHA-256 of evidence file
    API->>S3: Store evidence file
    API->>CC: SubmitAttestation(agreementId, conditionId, evidenceHash, attestorMSP)
    Note over CC: Endorsement: PayeeOrg or ArbitratorOrg peer
    CC->>CC: Record Attestation{hash, timestamp, attestorMSP}
    CC->>CC: Evaluate: all conditions satisfied?
    alt All conditions met (AUTO release mode)
        CC->>CC: Transfer tokens: escrow → payee wallet
        CC->>CC: Update Agreement{status:RELEASED}
        CC->>CC: Emit AgreementReleased event
    else Requires payer approval (MANUAL mode)
        CC->>CC: Update Agreement{status:PENDING_RELEASE}
        CC->>CC: Emit PendingRelease event
    end
    CC-->>API: TxID, blockNumber, newStatus
    API-->>FE: 200 { status, txId }
    FE-->>Payee: Attestation submitted — status updated
```

### 4.4 Dispute Flow

```mermaid
sequenceDiagram
    actor Payer
    actor Arbitrator
    participant FE as Frontend
    participant API as Backend Gateway
    participant CC as Chaincode

    Payer->>FE: Click "Dispute" on funded agreement
    FE->>API: POST /api/disputes { agreementId, reason }
    API->>CC: RaiseDispute(agreementId, reason, disputerMSP)
    CC->>CC: Update Agreement{status:DISPUTED}
    CC->>CC: Create DisputeCase record
    CC->>CC: Emit DisputeRaised event (ArbitratorOrg listener)
    CC-->>API: TxID, disputeId
    API-->>FE: 200 { disputeId, status: "DISPUTED" }
    Note over Arbitrator: Reviews on-chain evidence + off-chain docs
    Arbitrator->>API: POST /api/disputes/:id/resolve { decision, resolution }
    API->>CC: ResolveDispute(disputeId, decision, arbitratorMSP)
    Note over CC: Endorsement: ArbitratorOrg + SettlementBankOrg
    alt Decision: RELEASE_TO_PAYEE
        CC->>CC: Transfer tokens: escrow → payee
        CC->>CC: Update Agreement{status:RELEASED}
    else Decision: RETURN_TO_PAYER
        CC->>CC: Transfer tokens: escrow → payer
        CC->>CC: Update Agreement{status:CANCELED}
    else Decision: SPLIT
        CC->>CC: Transfer proportional tokens to each party
        CC->>CC: Update Agreement{status:SETTLED}
    end
    CC-->>API: TxID
```

---

## 5. Technology Stack

| Component | Technology | Version | Rationale |
|-----------|-----------|---------|-----------|
| Blockchain | Hyperledger Fabric | 2.5 LTS | Permissioned ledger; MSP-based access control; private data collections |
| Chaincode | Go + fabric-contract-api-go | 1.22 / 0.1.0 | Type-safe contract API; strong performance; standard Fabric choice |
| Ordering | RAFT | — | CFT; no external ZooKeeper dependency; production-ready in Fabric 2.x |
| Backend runtime | Node.js | 20 LTS | Fabric Node SDK (fabric-network) native support |
| Backend framework | Express | 4.x | Lightweight; middleware ecosystem; well-understood |
| Backend language | TypeScript | 5.x | Type safety across API + SDK layer |
| ORM | Prisma | 5.x | Type-safe queries; schema-first; excellent migration tooling |
| Database | PostgreSQL | 16 | ACID compliance; JSON support for agreement metadata |
| Frontend | Next.js | 14 (App Router) | Server components; excellent DX; Vercel-deployable |
| UI library | shadcn/ui + Tailwind | — | Accessible, unstyled components; full control over design |
| Forms | react-hook-form + zod | — | Schema-driven validation; performance |
| AI gateway | OpenAI API / Anthropic | gpt-4o / claude-3-5 | Clause generation, risk scoring, explanations |
| Object storage | AWS S3 / MinIO | — | Evidence file storage; only hashes on-chain |
| Container | Docker + Docker Compose | — | Reproducible dev environment |

---

## 6. Security Architecture

### 6.1 Identity & Authentication

- **On-chain identity:** Fabric CA issues x.509 certificates per org. Each API call to Fabric uses an enrolled identity from the org's wallet. MSP ID extracted from certificate in `ctx.GetClientIdentity().GetMSPID()`.
- **API authentication:** JWT tokens (RS256) issued by backend auth service. Each token encodes `orgMSP`, `userId`, `role`.
- **KYC gating:** Token wallet activation requires KYC approval status in `kycStatusPDC`. Chaincode reads this PDC before any `Mint` or `FundAgreement` call.

### 6.2 Data Privacy

- Sensitive agreement terms stored in `agreementTermsPDC` — only PayerOrg + PayeeOrg peers hold the data. SettlementBankOrg and ArbitratorOrg see only the public Agreement record.
- Evidence files stored in S3 with server-side encryption (AES-256). On-chain: SHA-256 hash only.
- Wallet balances in `tokenBalancesPDC` — SettlementBankOrg only. Other orgs query balances via authorized chaincode function that returns only the caller's own balance.

### 6.3 Threat Model Notes

| Threat | Mitigation |
|--------|-----------|
| Fake attestation submission | SHA-256 hash of evidence stored on-chain; off-chain file must match |
| Unauthorized fund release | Endorsement policy requires SettlementBankOrg + PayeeOrg/ArbitratorOrg |
| Replay attacks | Fabric's per-transaction nonce + proposal response mechanism prevents replay |
| Malicious org peer | RAFT ordering requires majority orderer nodes; single org cannot unilaterally order blocks |
| API injection | Parameterized Prisma queries; chaincode arg validation; zod input validation at API layer |

---

## 7. Deployment Architecture

### 7.1 Development (MOCK_FABRIC=true)

```
localhost
├── Backend (port 3001) — MOCK_FABRIC=true returns simulated responses
├── Frontend (port 3000) — Next.js dev server
├── PostgreSQL (port 5432) — Docker
└── MinIO (port 9000) — Docker (S3-compatible)
```

### 7.2 Development (Real Fabric Network)

```
localhost
├── Backend (port 3001) — MOCK_FABRIC=false, connects to Fabric
├── Frontend (port 3000)
├── PostgreSQL (port 5432) — Docker
├── MinIO (port 9000) — Docker
└── Fabric Network (Docker Compose)
    ├── orderer.example.com (port 7050)
    ├── peer0.payerorg.example.com (port 7051)
    ├── peer0.settlementbankorg.example.com (port 9051)
    ├── ca.payerorg.example.com (port 7054)
    └── ca.settlementbankorg.example.com (port 8054)
```

### 7.3 Production

```
Cloud (AWS / GCP)
├── EKS / GKE — Backend API (3 replicas, HPA)
├── Vercel / CloudFront — Frontend (CDN-distributed)
├── RDS PostgreSQL (Multi-AZ)
├── S3 (evidence storage)
├── Fabric Network (dedicated VMs or Kubernetes)
│   ├── Managed via Hyperledger Bevel or IBM Blockchain Platform
│   └── Each org operates their own infrastructure
└── AI Gateway — OpenAI / Anthropic API (behind rate limiter)
```
