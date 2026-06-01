# CheckChain

Programmable payments to replace the paper check.

A working, full-stack prototype of a smart-contract-based payment platform built on **Hyperledger Fabric**. CheckChain creates conditional payment agreements (escrow-as-code), releases funds when verifiable conditions are met, and issues cryptographically signed "Digital Cashier's Check" receipts. Designed for contractors, landlords, escrow agents, and B2B AP teams.

> Built by [Kerry Goode](https://github.com/KgoodeMFTA) — M.S. FinTech & Analytics, Wake Forest University. This project is a direct extension of capstone research on blockchain integration for programmable consumer banking.

---

## What it does

- **Conditional payment agreements** — Payer + Payee define release conditions: deliverable hashes, dates, milestones, multi-party attestations
- **Tokenized deposit escrow** — Funds are held as a tokenized deposit claim issued by a permissioned Settlement Bank organization (NOT a stablecoin, NOT crypto)
- **Multi-party attestation** — Payee or oracle/notary submits proof of fulfillment; chaincode verifies and releases
- **Dispute resolution** — Configurable Arbitrator org with RELEASE / RETURN / SPLIT resolution paths
- **Digital Cashier's Check** — printable, QR-verified receipt showing payer, payee, amount in words, agreement ID, chaincode tx hash, and block number
- **AI features** — clause generation from plain English, agreement explainer, counterparty risk scoring, dispute assistant

## Tech stack

| Layer       | Technology                                              |
| ----------- | ------------------------------------------------------- |
| Ledger      | Hyperledger Fabric 2.x (permissioned)                   |
| Chaincode   | Go (contract-api-go), with private data collections     |
| Backend     | Node.js, TypeScript, Express, fabric-network SDK, Prisma |
| Frontend    | Next.js 14, TypeScript, Tailwind, shadcn/ui             |
| Storage     | Postgres (off-chain cache), S3 (evidence files; hashes on-chain) |
| AI          | LLM gateway: OpenAI / Anthropic / Mock                  |
| Auth        | JWT (MVP)                                               |
| Test        | go test (chaincode), jest (backend), 13 backend tests   |
| Container   | Docker, docker-compose                                  |

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) and [`docs/CHAINCODE_SPEC.md`](docs/CHAINCODE_SPEC.md).

## Quickstart (no Fabric required)

```bash
# Backend (terminal 1)
cd backend
cp .env.example .env       # defaults: MOCK_FABRIC=true, AI_PROVIDER=mock
npm install
npm run db:generate
npm run dev
# → http://localhost:3001

# Frontend (terminal 2)
cd frontend
npm install
echo "NEXT_PUBLIC_API_URL=http://localhost:3001/api" > .env.local
npm run dev
# → http://localhost:3000

# Demo login: user@payerorg.example.com / demo123
```

In `MOCK_FABRIC=true` mode, the backend simulates chaincode responses so the entire UX runs without a live Fabric network. The same backend calls go to real chaincode when you point it at a network.

## With a real Fabric network

Easiest path uses fabric-samples test-network:

```bash
# In a separate directory
git clone https://github.com/hyperledger/fabric-samples.git
cd fabric-samples/test-network
./network.sh up createChannel -c agreements-channel -s couchdb -ca
./network.sh deployCC -ccn checkchain-cc \
  -ccp /path/to/check-replacement-app/chaincode \
  -ccl go \
  -cccg /path/to/check-replacement-app/chaincode/collections_config.json

# Then in backend/.env:
MOCK_FABRIC=false
FABRIC_PEER_ENDPOINT=grpcs://localhost:7051
```

A custom 2-org dev network compose is provided under [`fabric-network/`](fabric-network/) for advanced setups.

## Run tests

```bash
cd backend && npm test
cd chaincode && go test ./...
```

## Project structure

```
.
├── chaincode/              Go chaincode for Hyperledger Fabric
│   ├── agreement_contract.go    Lifecycle: create, fund, attest, release, dispute, resolve, cancel
│   ├── cash_token_contract.go   Bank-restricted mint/burn/transfer of tokenized deposits
│   ├── collections_config.json  Private data collections (terms, balances, KYC)
│   └── *_test.go
├── fabric-network/         Dev compose, channel artifacts, deploy scripts
├── backend/                Express + fabric-network SDK + Prisma
│   ├── src/
│   │   ├── routes/         agreements, attestations, disputes, tokens, ai, receipts, auth
│   │   ├── fabric/         gateway.ts (with MOCK_FABRIC mode), identity.ts
│   │   ├── services/       ai_service, kyc_service
│   │   └── middleware/
│   └── tests/              jest
├── frontend/               Next.js 14 + TypeScript + Tailwind
│   └── src/app/            dashboard, agreements/new, agreements/[id], receipts/[txid], ai/draft-conditions
└── docs/                   PRD, ARCHITECTURE, CHAINCODE_SPEC, API_CONTRACTS, AI_PROMPTS, DATA_MODEL, UX_FLOWS
```

## Documentation

- **[PRD](docs/PRD.md)** — Personas, user stories, regulatory analysis, roadmap
- **[Architecture](docs/ARCHITECTURE.md)** — Fabric topology, off-chain components, data flows
- **[Chaincode Spec](docs/CHAINCODE_SPEC.md)** — Asset shapes, state machines, function signatures, endorsement policies
- **[API Contracts](docs/API_CONTRACTS.md)** — REST endpoints with JSON examples
- **[AI Prompts](docs/AI_PROMPTS.md)** — Clause drafter, explainer, risk scorer, dispute assistant
- **[UX Flows](docs/UX_FLOWS.md)** — Screen-by-screen wireframes including the Digital Cashier's Check

## Regulatory framing

CheckChain's in-app dollar is a **tokenized deposit claim** issued by a permissioned Settlement Bank organization on the Fabric network. It is intentionally **not** a stablecoin and **not** a virtual currency under NYDFS BitLicense. The model aligns with FDIC's 2026 NPR and the GENIUS Act treatment of tokenized deposits as governed identically to standard FDIC-insured deposits under the FDI Act.

Operating considerations covered in [`docs/PRD.md`](docs/PRD.md):

- FinCEN MSB registration (BSA, CIP, CDD, SAR thresholds)
- State money transmitter licensing (51-jurisdiction roadmap via agent-of-payee and bank-partner-exemption strategies)
- UCC Article 3 (negotiable instruments), Article 4A (wholesale wires)
- Regulation E (consumer EFTs — applicable where any consumer participates)
- OCC interpretive letters #1170, #1172, #1174
- FDIC pass-through deposit insurance via custodial structure

## Roadmap

- v1.0 — current scope (agreement lifecycle + tokenized escrow + AI clause drafter + cashier's check receipt)
- v1.5 — Real bank-partner integration (BaaS — Column / Lead Bank / Cross River), live ACH/RTP rails
- v2.0 — Invoice financing on locked agreements, factoring marketplace, lien waiver automation
- v3.0 — Multi-network BaaS offering for partner banks

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Security reports: [SECURITY.md](SECURITY.md).

## License

[MIT](LICENSE) — see file for full text.

---

## Disclaimer

This is a portfolio / capstone project. It is not licensed, registered, or audited for production money transmission, banking, or escrow services in any U.S. or international jurisdiction. Nothing in this repository constitutes legal, financial, or regulatory advice. No part of this code should be used to move real funds without independent legal and regulatory review, FinCEN registration where applicable, state money transmitter licensing or exemption analysis, and a chartered banking partner.
