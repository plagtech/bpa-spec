# Batch Payments for Agents (BPA) — Specification v1.0

**Status:** 1.0 — Final (settlement contracts verified on BaseScan and Etherscan, exact match)
**Editors:** Spraay Protocol
**Reference implementation:** Spraay Gateway (`gateway.spraay.app`)
**License:** CC-BY-4.0 (see §18)

---

## Abstract

This document specifies **Batch Payments for Agents (BPA)**: a framework-neutral standard for how an autonomous software agent disburses funds to many recipients in a single, governed, verifiable operation. BPA defines the request format, lifecycle, idempotency and partial-failure semantics, settlement confirmation, multi-chain behavior, and the security/governance controls an agent payment MUST operate behind.

BPA is transport-agnostic. It binds to `x402` and to the Machine Payments Protocol (MPP), and exposes a canonical tool shape for MCP-based agents. The on-chain reference implementation is the Spraay batch contract on Base.

---

## 1. Terminology

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as described in RFC 2119.

- **Payer** — the agent (or the wallet it controls) initiating a disbursement.
- **Recipient** — an address receiving funds in a batch.
- **Batch** — an ordered set of (recipient, amount, asset) tuples settled under one request.
- **Disbursement Request** — the signed or authorized instruction that defines a Batch.
- **Settlement** — the on-chain (or rail-native) finalization of transfers in a Batch.
- **Facilitator** — the service that validates the request, enforces policy, and submits settlement (the Spraay Gateway, in the reference implementation).
- **Policy** — the deterministic ruleset (caps, allowlists, approvals) evaluated before settlement.

---

## 2. Motivation

Card networks and bank rails penalize exactly the payments agents need to make: amounts too small to survive fixed fees, payments with no human at checkout, and one-to-many disbursements to global or unbanked recipients. Agents increasingly need to pay crowds (labeling/annotation), devices (DePIN/IoT), players (gaming economies), and data providers (per-call marketplaces).

No framework-neutral contract exists for *how an agent pays many recipients at once safely*. BPA defines that contract so that any agent, on any framework, can construct a disbursement that is **idempotent, partially-recoverable, settlement-verifiable, and governed before execution** — not after.

---

## 3. Scope

In scope: request format, lifecycle, idempotency, partial failure, settlement proof, multi-chain semantics, governance controls, fee disclosure, transport bindings.

Out of scope: KYC/AML obligations of the Payer, tax treatment, recipient onboarding, and model-layer prompt safety. BPA assumes prompt-level safety is **not** a control surface; all enforcement is deterministic and occurs before settlement.

---

## 4. Disbursement Request

A BPA Disbursement Request **MUST** be expressible as the following object. Field names are normative. A machine-readable JSON Schema (2020-12) is published alongside this spec (see §15) and is the authoritative validation artifact.

```json
{
  "bpa_version": "1.0",
  "idempotency_key": "string (REQUIRED, unique per logical batch)",
  "chain": "string (CAIP-2 chain id, e.g. 'eip155:8453')",
  "asset": "string (token contract address, or native symbol)",
  "trigger": {
    "type": "immediate | on_approval | on_condition",
    "condition_ref": "string (OPTIONAL; required when type=on_condition)"
  },
  "recipients": [
    { "address": "string", "amount": "string (base units)", "ref": "string (OPTIONAL line id)" }
  ],
  "policy_ref": "string (OPTIONAL; id of the policy to evaluate)",
  "metadata": { "purpose": "string", "external_id": "string" }
}
```

- `idempotency_key` **MUST** be present. A Facilitator **MUST** treat two requests with the same key as the same logical batch (see §5).
- `amount` **MUST** be expressed in the asset's base units (integer string) to avoid float drift.
- `recipients` **MUST** contain at least one entry. A Facilitator **MUST** disclose its maximum batch size. The Spraay Base contract enforces a hard on-chain cap of **200 recipients** per call (`MAX_RECIPIENTS`); batches larger than the cap **MUST** be chunked by the Facilitator into multiple settlements under one logical `idempotency_key`.
- `trigger.type` of `on_approval` or `on_condition` defines a **task-gated** batch (see §8); funds **MUST NOT** settle until the condition resolves true.

---

## 5. Idempotency

Batch disbursement is financially irreversible, so replay safety is mandatory.

- A Facilitator **MUST** persist the `idempotency_key` and the resulting settlement reference.
- A repeated request with a matching `idempotency_key` **MUST** return the original result and **MUST NOT** initiate a second settlement.
- A repeated key with a *different* recipient set or amount **MUST** be rejected with a conflict error and **MUST NOT** settle.
- Idempotency records **SHOULD** be retained no less than the chain's deepest practical reorg window.

---

## 6. Lifecycle & States

```
RECEIVED → VALIDATED → POLICY_EVAL → { DENIED | APPROVED }
APPROVED → (trigger satisfied?) → SETTLING → { SETTLED | PARTIALLY_SETTLED | FAILED }
```

- `POLICY_EVAL` **MUST** be deterministic and **MUST** fail closed: if policy cannot be evaluated, the result is `DENIED`.
- A Batch in `APPROVED` with a non-`immediate` trigger **MUST** remain unsettled until the trigger resolves true.
- Every state transition **MUST** produce an audit record (§10).

---

## 7. Partial Failure Semantics

A Facilitator **MUST** declare its atomicity mode, one of:

- **ATOMIC** — all transfers settle or none do (single-transaction batch).
- **BEST_EFFORT** — each recipient settles independently; failures are isolated.

For `BEST_EFFORT`, the result **MUST** enumerate per-recipient status:

```json
{
  "batch_status": "PARTIALLY_SETTLED",
  "results": [
    { "ref": "w-001", "address": "0x..", "status": "settled", "tx": "0x.." },
    { "ref": "w-002", "address": "0x..", "status": "failed", "reason": "string" }
  ],
  "retry_token": "string (REQUIRED if any failed; re-submits ONLY failed lines)"
}
```

- A `retry_token` **MUST** scope retries to failed lines only and **MUST** be idempotent against already-settled lines.
- A Facilitator **MUST NOT** report `SETTLED` unless every line is confirmed (§9).

**Atomicity by rail (reference implementation):** all three core rails settle `ATOMIC`. Base and Ethereum settle the entire batch in a single all-or-nothing transaction via `SprayContract` (any failed transfer reverts the batch). Solana achieves the same batch atomicity through its native transaction model rather than a custom contract; the Facilitator returns an unsigned transaction for the agent to sign and submit. No core rail operates in `BEST_EFFORT` mode; the mode remains defined for Facilitators that add non-atomic rails.

---

## 8. Task-Gated Disbursement

The hardest real-world batches are conditional (pay-on-approval for crowdwork, pay-on-delivery for devices, prize release on result finalization). For `trigger.type` of `on_approval` / `on_condition`:

- The condition **MUST** be referenced by a stable `condition_ref` the Facilitator can independently verify.
- Funds **MAY** be escrowed at `APPROVED` and **MUST** be released only when the condition resolves true.
- A condition that resolves false (rejection, dispute, expiry) **MUST** return escrowed funds to the Payer and emit a terminal audit record.
- Dispute windows, if any, **MUST** be disclosed and **MUST** be enforced before release.

> **Informative (non-normative for 1.0).** BPA 1.0 requires that a condition be *independently verifiable* but does not yet standardize *how* attestation is produced or trusted (e.g. signed approver attestation, oracle, multi-party sign-off). Until a normative attestation model lands in 1.1, a Facilitator offering task-gated triggers **MUST** document its own attestation and dispute mechanism. The reference implementation serves task-gated flows through dedicated escrow endpoints rather than the immediate batch endpoint.

---

## 9. Settlement Confirmation

- Each settled line **MUST** carry a verifiable settlement reference (e.g. transaction hash) resolvable on the named chain.
- A Facilitator **MUST** define its confirmation threshold per chain and **MUST NOT** report `settled` before it is met. For the core rails:
  - **Base** (rollup) — fast soft-confirmation with a documented lag to L1 finality; the Facilitator **MUST** disclose whether `settled` reflects soft-confirm or L1 finality.
  - **Ethereum** (L1) — `settled` on inclusion, with finality reached at the next checkpoint (~2 epochs); the Facilitator **SHOULD** state the block-depth it treats as final.
  - **Solana** — `settled` at the `confirmed`/`finalized` commitment level the Facilitator declares.
- The Payer **MUST** be able to verify settlement independently of the Facilitator (no trust-me confirmations). The Spraay `SprayContract` emits `SprayETHExecuted` / `SprayTokenExecuted` events carrying `(sender, [token,] totalAmount, recipientCount, feeAmount, timestamp)`, which serve as the independently verifiable on-chain settlement record on EVM chains.

---

## 10. Governance & Security Model

BPA treats every disbursement as a high-risk agent action. Controls are deterministic and evaluated **before** settlement.

A conformant Facilitator **MUST** support, and a Policy **MAY** specify:

- **Spend caps** — per-batch, per-recipient, and rolling-window ceilings. A request exceeding a cap **MUST** be denied or routed to approval.
- **Recipient allowlists / denylists** — settlement to a non-allowlisted address (when an allowlist is active) **MUST** be denied.
- **Approval gating** — a request matching an approval rule **MUST** pause in `APPROVED`-pending until a named approver authorizes it.
- **Tamper-evident audit** — every decision (policy active, request, allow/deny reason, settlement ref) **MUST** be recorded in an append-only, independently verifiable log.

This mirrors zero-trust agent governance: the goal is not to ask the agent to behave but to make an out-of-policy disbursement structurally impossible.

---

## 11. Fees

There are **two distinct fees**, and a Facilitator **MUST** disclose both:

1. **Access/service fee** — charged per call to build/quote the batch, paid over the transport rail (x402 or MPP) before the Facilitator responds. In the Spraay reference implementation this is a flat per-endpoint price (e.g. `POST /api/v1/batch/execute` = $0.02), surfaced in the `402` quote.
2. **On-chain protocol fee** — charged at settlement by the contract: `feeBps` is currently **30 (0.30%)**, capped by `MAX_FEE_BPS` at **500 (5%)**, paid to the on-chain `feeRecipient`. Agents **SHOULD** call `calculateFee(amount)` / `calculateTotalCost(totalAmount)` to obtain the exact fee and total before signing.

Rail- and chain-specific fees may differ and **MUST** be disclosed per rail. Fees **MUST** be itemized; the on-chain `feeAmount` is emitted in the settlement event.

---

## 12. On-Chain Reference Implementation

The reference settlement layer is the **Spraay batch contract (`SprayContract`) on Base**:

```
Chain:        Base (eip155:8453)
Contract:     0x1646452F98E36A3c9Cfc3eDD8868221E207B5eEC
Fee:          feeBps = 30 (0.30%), max 500 (5%); on-chain feeRecipient
Max batch:    MAX_RECIPIENTS = 200 per call
Controls:     Ownable, Pausable, ReentrancyGuard
```

The contract settles a batch in a single transaction (ATOMIC mode — any failed transfer reverts the whole batch). Verified interface:

```solidity
struct Recipient { address recipient; uint256 amount; }

// Native (ETH) batch, per-recipient amounts
function sprayETH(Recipient[] calldata recipients) external payable;

// ERC-20 batch, per-recipient amounts
function sprayToken(address token, Recipient[] calldata recipients) external;

// Equal-amount batch (same amount to every recipient)
function sprayEqual(address token, address[] calldata recipients, uint256 amountPerRecipient) external payable;

// Quote helpers (call before sending)
function calculateFee(uint256 amount) external view returns (uint256);
function calculateTotalCost(uint256 totalAmount) external view returns (uint256);

// Settlement events (independent verification)
event SprayETHExecuted(address sender, uint256 totalAmount, uint256 recipientCount, uint256 feeAmount, uint256 timestamp);
event SprayTokenExecuted(address sender, address token, uint256 totalAmount, uint256 recipientCount, uint256 feeAmount, uint256 timestamp);
```

Settlement is defined for **three core chains**, chosen because they are where autonomous-agent payment activity actually concentrates (Base and Solana) plus institutional reach (Ethereum):

**EVM — `SprayContract`.** Same verified, source-published contract; 30 bps fee, 200-recipient cap, atomic all-or-nothing settlement.

| Chain | CAIP-2 | Contract | Explorer |
|---|---|---|---|
| Base | `eip155:8453` | `0x1646452F98E36A3c9Cfc3eDD8868221E207B5eEC` | BaseScan |
| Ethereum | `eip155:1` | `0x15E7aEDa45094DD2E9E746FcA1C726cAd7aE58b3` | Etherscan |

**Solana — native, non-custodial.** No custom contract; batch atomicity comes from Solana's native transaction model. The Facilitator returns an **unsigned transaction** for the agent to sign and submit. USDC mint `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`.

**Live API shape (reference Facilitator):** `POST /api/v1/batch/execute` takes `{ token, recipients[], amounts[], sender }` and returns `{ transactions: [...] }` (unsigned). The parallel `recipients[]`/`amounts[]` arrays map to the contract's `Recipient[] { recipient, amount }` tuple. The model is **non-custodial end-to-end**: the gateway only builds the transaction; the agent's own on-chain signature authorizes the spend, and the gateway never holds funds.

**Supported assets (Base/Ethereum):** native ETH and any ERC-20; commonly USDC (Base `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`), USDT, DAI, EURC, WETH.

**Boundary note:** the on-chain contract provides only atomic disbursement + fee. BPA **idempotency (§5), escrow / task-gating (§8), partial-failure isolation (§7), and policy/governance (§10) are Facilitator-layer responsibilities** — not enforced by the contract. A task-gated batch that escrows funds requires a separate escrow mechanism (the reference Facilitator exposes dedicated escrow endpoints); `SprayContract` itself holds no escrow. When the contract is paused, settlement reverts, satisfying the fail-closed requirement of §6. Because settlement is non-custodial, request-layer authentication is a defense-in-depth control rather than the sole guard on funds — a request that is not signed-and-submitted by the agent cannot move money.

> **Non-normative — additional deployments.** The same `SprayContract` is also deployed on Arbitrum, Polygon, BNB, Avalanche, Unichain, Plasma, and BOB, with an equivalent Clarity contract on Stacks (testnet). These are outside the normative scope of BPA 1.0 and are not required for conformance; addresses are published in the Spraay gateway's `/api/v1/tokens` and `/.well-known/x402.json`.

---

## 13. Transport Bindings

### 13.1 x402
A BPA request is initiated against an `x402`-gated endpoint. The agent receives a `402`, constructs payment, and the Facilitator returns the BPA result (unsigned transaction) on success. In the reference implementation the endpoint advertises an `exact` scheme on `eip155:8453` (USDC) — and, where supported, a Solana `exact` scheme — via the Coinbase CDP facilitator.

Each endpoint declares a discovery payload in its `extensions` field using `declareDiscoveryExtension({ input, inputSchema, bodyType, output: { example, schema } })`. **The `extensions` field MUST be copied into the payment payload by the client** (`x402-fetch` and equivalents). If it is dropped, the endpoint settles correctly but becomes invisible to facilitator catalogs (the Bazaar index) — this is the single most common discoverability failure and the spec calls it out as a conformance requirement (§14.8).

### 13.2 MPP
BPA binds to the Machine Payments Protocol as an alternative access rail. The reference Facilitator runs `mppx` with the Tempo method: currency **pathUSD** (`0x20c0000000000000000000000000000000000000`) on network `tempo`, discovery at `/.well-known/mpp.json`, payment presented via an `Authorization: Payment` header. MPP middleware is evaluated before x402; a request with no payment credential falls through to x402 (default), so the two rails coexist without collision. The Disbursement Request body is **identical** across rails — only the access-payment mechanism differs.

### 13.3 MCP tool shape
For MCP agents the canonical tools are `spraay_batch_execute` ("batch pay up to 200 recipients") and `spraay_batch_estimate`, published on Smithery (`@plagtech/spraay-x402-mcp`). A conformant tool **SHOULD** expose the Disbursement Request fields directly and **SHOULD** be described by intent ("pay many recipients in one batch; optional settlement-on-approval"), so agents match it to user intent at runtime.

---

## 14. Conformance Checklist

An implementation is **BPA-1.0 conformant** if it:

1. Accepts the §4 request and rejects malformed requests.
2. Enforces idempotency per §5.
3. Declares its atomicity mode and honors §7 partial-failure semantics.
4. Supports task-gated triggers per §8 (or explicitly declares `immediate`-only).
5. Provides independently verifiable settlement per §9.
6. Enforces caps, allowlists, approval, and tamper-evident audit per §10, failing closed.
7. Discloses fees per §11.
8. Preserves the `extensions` field across x402 transport per §13.1.

---

## 15. Reference

- Spraay Gateway: `gateway.spraay.app` (discovery: `/.well-known/x402.json`, `/.well-known/mpp.json`, `/openapi.json`, `/llms.txt`)
- Spraay MCP server (Smithery): `@plagtech/spraay-x402-mcp` — tools `spraay_batch_execute`, `spraay_batch_estimate`
- Machine-readable request schema (JSON Schema 2020-12): `https://docs.spraay.app/bpa/1.0/disbursement-request.schema.json`
- Docs: `docs.spraay.app`
- Contracts: Base `0x1646452F98E36A3c9Cfc3eDD8868221E207B5eEC`, Ethereum `0x15E7aEDa45094DD2E9E746FcA1C726cAd7aE58b3`; Solana settles natively (non-custodial). Additional non-normative deployments listed in §12.
- Related Spraay specs: RTP (Robot Task Protocol), SCTP (Supply Chain Task Protocol)

---

## 16. Conformance Statement (Reference Implementation)

The Spraay Gateway (`gateway.spraay.app`) is the reference implementation of BPA 1.0 and satisfies §14 as follows:

| §14 item | Status | Notes |
|---|---|---|
| 1. Accept/reject §4 request | ✅ | Validated against the published JSON Schema; malformed requests return a structured `400`. |
| 2. Idempotency (§5) | ✅ | Facilitator-layer; keyed dedup with conflict rejection. |
| 3. Atomicity mode (§7) | ✅ | Declares `ATOMIC` on all three core rails. |
| 4. Task-gated triggers (§8) | ◑ | The batch endpoint declares **`immediate`-only** (permitted by §14.4). Task-gated flows are served by the separate escrow endpoints; see §8 informative note. |
| 5. Verifiable settlement (§9) | ✅ | On-chain tx hash + `SprayContract` events (EVM); native tx signature (Solana). |
| 6. Caps / allowlists / approval / audit (§10) | ✅ | Facilitator-layer policy + append-only audit, fail-closed. |
| 7. Fee disclosure (§11) | ✅ | Per-call price in the `402` quote; on-chain `feeBps`/`feeRecipient` for the protocol fee. |
| 8. `extensions` preserved across x402 (§13.1) | ✅ | Endpoints declare discovery via `declareDiscoveryExtension`; clients MUST copy `extensions` into the payment payload. |

A self-attestation of this form, kept current with the deployed gateway, is the recommended way for any third-party Facilitator to claim BPA-1.0 conformance.

---

## 17. Versioning and Governance

- **Versioning.** BPA uses semantic versioning. The canonical URL is version-pinned (`/bpa/1.0`); a published version is immutable. Backward-compatible clarifications increment the patch level; additive, non-breaking features increment the minor level (`1.1`); breaking changes increment the major level (`2.0`) at a new URL. Existing citations of `/bpa/1.0` never change meaning.
- **Proposing changes.** Changes are proposed as issues/PRs against the spec repository (`plagtech/bpa-spec`). Each proposal states the section affected, the motivation, and whether it is clarifying, additive, or breaking. Editors merge clarifying changes directly; additive/breaking changes require a versioned release and a changelog entry.
- **Normative vs informative.** Only sections (and sentences using RFC-2119 keywords) marked normative bind conformance. Informative notes, examples, and the §12 non-normative deployment footnote do not affect conformance.

---

## 18. License

This specification is published under **Creative Commons Attribution 4.0 International (CC-BY-4.0)**. Implementations of the specification — including independent Facilitators — are unrestricted and require no permission. The accompanying JSON Schema is released under the same terms.

---

## 19. Changelog

- **1.0 — initial release.** Core request format (§4), idempotency (§5), lifecycle (§6), partial-failure semantics (§7), task-gated disbursement (§8, attestation mechanism informative pending 1.1), settlement confirmation (§9), governance model (§10), two-fee model (§11), on-chain reference implementation on Base/Ethereum/Solana (§12), x402/MPP/MCP transport bindings (§13), conformance checklist + reference-implementation attestation (§14, §16), versioning/governance (§17). Additional `SprayContract` deployments documented as non-normative.
