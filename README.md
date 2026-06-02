# Batch Payments for Agents (BPA)

**An open specification for how an autonomous AI agent pays many recipients in a single, governed, verifiable batch — via x402 or MPP, on Base, Ethereum, and Solana.**

BPA defines the request format, lifecycle, idempotency and partial-failure rules, settlement confirmation, governance controls (spend caps, allowlists, approval, audit), fees, and transport bindings for agent-initiated batch disbursement. It is framework-neutral: any agent on any stack can construct a batch payout that is idempotent, partially-recoverable, settlement-verifiable, and governed before execution.

- **Status:** 1.0 — Final
- **License:** [CC-BY-4.0](#license)
- **Reference implementation:** Spraay Gateway — `gateway.spraay.app`
- **Spec:** [`1.0/BATCH-PAYMENTS-FOR-AGENTS-1.0.md`](1.0/BATCH-PAYMENTS-FOR-AGENTS-1.0.md)
- **Machine-readable request schema (JSON Schema 2020-12):** [`1.0/disbursement-request.schema.json`](1.0/disbursement-request.schema.json)

## Why BPA exists

Card and bank rails penalize exactly the payments agents need to make: amounts too small to survive fixed fees, payments with no human at checkout, and one-to-many disbursement to global or unbanked recipients. Agents increasingly need to pay crowds (data labeling), devices (DePIN), players (gaming), and data providers (per-call marketplaces). No framework-neutral standard existed for *how an agent safely pays many recipients at once* — BPA is that standard.

## Core settlement chains

Settlement is normatively defined for the three chains where agent payment activity concentrates (Base, Solana) plus institutional reach (Ethereum).

| Chain | CAIP-2 | Mechanism | Contract |
|---|---|---|---|
| Base | `eip155:8453` | `SprayContract` (atomic) | [`0x1646452F98E36A3c9Cfc3eDD8868221E207B5eEC`](https://basescan.org/address/0x1646452F98E36A3c9Cfc3eDD8868221E207B5eEC#code) |
| Ethereum | `eip155:1` | `SprayContract` (atomic) | [`0x15E7aEDa45094DD2E9E746FcA1C726cAd7aE58b3`](https://etherscan.io/address/0x15E7aEDa45094DD2E9E746FcA1C726cAd7aE58b3#code) |
| Solana | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` | native, non-custodial | settles via native transaction |

Both EVM contracts are source-verified (exact match). On-chain protocol fee is 30 bps (0.30%), with a 200-recipient cap per call. Additional non-normative deployments are listed in §12 of the spec.

## At a glance

- **Non-custodial** — the Facilitator builds an unsigned transaction; the agent's own signature authorizes the spend. The gateway never holds funds.
- **Atomic** on all three core rails (all-or-nothing).
- **Two fees, both disclosed** — a per-call access fee (x402/MPP) and the on-chain protocol fee.
- **Governed before execution** — spend caps, allowlists, approval gating, and tamper-evident audit, all fail-closed.

## Using it

- **Reference Facilitator:** `gateway.spraay.app` — discovery at `/.well-known/x402.json`, `/.well-known/mpp.json`, `/openapi.json`, `/llms.txt`.
- **MCP (Smithery):** `@plagtech/spraay-x402-mcp` — tools `spraay_batch_execute`, `spraay_batch_estimate`.
- **Validate a request** against [`1.0/disbursement-request.schema.json`](1.0/disbursement-request.schema.json) before sending.

## Conformance

An implementation is BPA-1.0 conformant if it meets the checklist in §14 of the spec. The reference implementation's conformance statement is in §16. Third-party Facilitators are encouraged to publish a self-attestation of the same form.

## Contributing / proposing changes

BPA uses semantic versioning with version-pinned, immutable published URLs (`/bpa/1.0`). Open an issue or PR stating the section affected, the motivation, and whether the change is clarifying, additive, or breaking. See §17 of the spec for the governance model.

## License

This specification and its JSON Schema are released under **Creative Commons Attribution 4.0 International (CC-BY-4.0)**. Implementations are unrestricted and require no permission.
