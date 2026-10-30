<!-- Org profile README: this file renders at https://github.com/Pactwright -->
<!-- Optional: copy brand/pactwright-logo-light.svg from any repo into this repo and put a <picture> banner above the h1. -->

# Pactwright

**Escrow and verifiable settlement for AI-agent purchases on Stellar — the money moves only when delivery is proven, and it comes back when it isn't.**

---

## The Problem

An AI agent can already buy something. [x402](https://developers.stellar.org/docs/build/agentic-payments/x402) works on Stellar today: the seller answers `402 Payment Required`, the agent pays in USDC, retries, and gets the resource. What the protocol does not do is anything after the money moves. If the delivery is wrong, partial, or never arrives, the payment is gone — "delivered" means whatever the seller's server says it means, a refund takes the seller's goodwill, and a seller or a judge who goes quiet can hold the funds indefinitely. An agent has no one to call.

Pactwright exists to put the rules for getting money back **inside the payment**. A purchase is opened as a *pact*: the buyer's cap is locked, the condition for "delivered correctly" is frozen before anything is paid, and the payout is decided against that frozen condition rather than renegotiated after the fact.

## Approach: Two Ways a Purchase Fails, Four Repos

Failures in an autonomous purchase split into two kinds that need fundamentally different machinery:

- **The money has already left.** No amount of off-chain bookkeeping reverses a settled payment. Only the ledger can hold the funds and decide afterwards — so escrow, deadlines, and release rules live in a Soroban contract, not in a service someone runs.
- **Nobody can say whether the delivery was right.** This cannot be settled by code alone: it needs a condition written down before payment, something that reads the delivery against that condition afterwards, and a record of that judgement that any third party can re-check without trusting the judge.

That split defines the repos, and each one owns exactly one layer:

| Tier | Repo | Owns | Status today |
|---|---|---|---|
| 1 — Hold the money | [`Pactwright-contracts`](https://github.com/Pactwright/Pactwright-contracts) | The `pact` escrow contract: cap, frozen condition hash, two deadlines, one verdict per pact, metered pacts, release | Deployed on testnet; 90 tests, eight invariants each enforced by a named test |
| 2 — Judge the delivery | [`Pactwright-Kit`](https://github.com/Pactwright/Pactwright-Kit) `services/verifier` | A model reads the evidence, deterministic rules decide, every verdict is a signed envelope anyone can re-verify | Hosted on testnet; 29/29 on the labelled real-model eval set |
| 3 — Pay and get paid | [`Pactwright-Kit`](https://github.com/Pactwright/Pactwright-Kit) | `@pactwright/core`, agent/seller/contracts-client packages, seller middleware, MCP tools and CLI, the facilitator's `pact` scheme | 418 passing tests; the whole flow runs as real testnet transactions nightly |
| 4 — Prove it afterwards | [`Pactwright-studio`](https://github.com/Pactwright/Pactwright-studio) · [`pactwright-evidence`](https://github.com/Pactwright/pactwright-evidence) | Studio indexes contract events, batches receipts into a Merkle tree and anchors the root on-chain; the evidence store is what the verifier actually read | Both live on testnet; a pact page and its receipt are public |

Two modes only, and the boundary is deliberate: **M0** is an exact refund through the seller's own `RefundVault`, for cheap high-volume calls from a seller you already trust; **M2** is the escrow contract above, for everything else. There is no middle mode — `upto`/M1 is not implemented and will not be, because a partially-agreed payout has no honest settlement rule. And nothing here asks you to trust Pactwright: there is no admin path to pact funds, the verifier named in a pact can sign a verdict but cannot move anything, and the buyer trusts that one key and nobody else.

## What is actually known

Measured, not asserted. Every figure below is recorded in a file in the repo it came from, and none of it is quoted from memory:

- On 2026-10-29 every MVP scenario ran end to end on Stellar testnet — **18 of 18 transactions SUCCESS** — including two complete x402 purchases through the hosted facilitator, and M2 pacts that opened, took a verdict, then settled or released. Full table with per-step transaction links: [`Pactwright-Kit` → Testnet evidence](https://github.com/Pactwright/Pactwright-Kit#testnet-evidence), machine-readable report in `e2e/testnet/results/2026-10-29.json`.
- The **hosted verifier judged a real purchase with its real model** (Google `gemini-3.5-flash-lite`) and the contract accepted its signature: `open` at ledger 5453755, `record_verdict` of ALLOW 1 USDC at 5453761 (`0225f78b…`), `settle` at 5453762 (`1f4ab38c…`). The script that drove it never held the verifier key, which is what makes the acceptance meaningful.
- Real-model accuracy is **29/29 (100%)** on the labelled `document.translation` case set through the full pipeline ([run 40897633766](https://github.com/Pactwright/Pactwright-Kit/actions/runs/40897633766)) — one condition kind, one model, one provider. Treat it as the size of the eval, not as a general accuracy claim.
- A refund has been signed by a **HashiCorp Vault transit key** (`fcb0d7a8…`), so the seller's signing key never left the KMS.
- OpenSSF Scorecard on Kit `main` is **6.4/10** as of 2026-10-30, up from 4.7 the same day. Pinned-Dependencies, Token-Permissions and Security-Policy are 10; Code-Review is 0 because branch protection requires a review but `enforce_admins` is off, so the org's own merges bypass it.

What is *not* known, said plainly: the per-delivery readings in the e2e table are fixtures, so those verdicts exercise the rules and the signing, not model judgement; nothing is published to npm yet; there is no pubnet deployment, no pubnet facilitator and no third-party audit; the verifier is a single service with a single key; both hosted services sit on Render's free tier, where a cold start measured 42s (facilitator) and 23s (verifier), so the first call after a sleep is slow rather than failing.

## Documentation

Everything is in one place: **[pactwright.github.io/docs](https://pactwright.github.io/docs/)** — the contract reference, the agent and seller guides, the verifier's operations and key-handling doc, the facilitator hardening notes, and the security model.

- Contract reference and deployment addresses: [pactwright.github.io/docs/reference/contract/](https://pactwright.github.io/docs/reference/contract/) · [`DEPLOYMENTS.md`](https://github.com/Pactwright/Pactwright-contracts/blob/main/DEPLOYMENTS.md)
- Live testnet pages: [Studio](https://pactwright-studio-testnet.onrender.com) · [verifier `/health`](https://pactwright-verifier-testnet.onrender.com/health) · [facilitator `/supported`](https://pactwright-facilitator-testnet.onrender.com/supported)
- The `pact` contract on the explorer: [CDUXJJUNYF…FEV6](https://stellar.expert/explorer/testnet/contract/CDUXJJUNYFILEYHE3LITANJEKCDFERNAABOSG75OWATFVM776BCMFEV6)

## Contributing

Each repo runs its own issue tracker, and each repo's `ROADMAP.md` is its source of truth — phases, task IDs, dependencies, and an update log with the commit or PR that moved each one. The rule that catches most new contributors: **a pull request that doesn't touch `ROADMAP.md` fails the `roadmap-check` job.** Read `ARCHITECTURE_ESSENTIALS.md` first; it states the invariants that are not negotiable (no admin path to pact funds, two modes, two deadlines, one attestation per pact, hashing in exactly one file per language, amounts as `i128` stroops on-chain and decimal strings in TypeScript).

- Open issues: [Pactwright-Kit](https://github.com/Pactwright/Pactwright-Kit/issues) · [Pactwright-contracts](https://github.com/Pactwright/Pactwright-contracts/issues) · [Pactwright-studio](https://github.com/Pactwright/Pactwright-studio/issues)
- Core-team-only paths, by design: `lifecycle.rs`, `verdict.rs`, `meter.rs`, `hashing.rs`, `services/verifier/src/sign.ts`, and the facilitator's `pact` scheme validation.
- Vulnerabilities: report privately through each repo's [SECURITY.md](https://github.com/Pactwright/Pactwright-Kit/security/policy) — never a public issue.

## Community

<!-- socials: add Discord / Telegram / X links here when they exist. Until then this section stays link-free on purpose. -->

Written for people building agents that have to spend money they can't claw back. If that is you, the fastest way in is the [Kit's 60-second section](https://github.com/Pactwright/Pactwright-Kit#-overview) and then the contract's [step-by-step purchase walkthrough](https://github.com/Pactwright/Pactwright-contracts#a-purchase-step-by-step).
