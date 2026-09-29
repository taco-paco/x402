# Scheme: `job-escrow`

## Summary

`job-escrow` is a payment scheme in which the client funds a **job** into escrow, the resource server delivers and commits to a **deliverable**, and an **evaluator** — never the resource server itself — releases the escrow to the server or returns it to the client. Where `auth-capture` lets the payee's operator decide when to take held funds, `job-escrow` is for payments whose release the client will not leave to the payee: within the scheme, the payee can only submit work, and only a third party the client named in the job can pay it out.

## Example use cases

- Agent-to-agent work whose output the buying agent cannot judge, delegated to an evaluator contract, model, or reputation-backed attester.
- Deliverables that need a verifiable commitment: the response the client received is hashed and committed onchain by the provider.
- Refundable work where a refund does not depend on the payee: the evaluator rejects, or the job expires and the client reclaims.
- Jobs carrying policy — allowlists, reputation writes, atomic side transfers — attached by the client as an escrow-level hook.

## Roles

| Role | Party | What it does |
| --- | --- | --- |
| **client** | x402 client (payer) | Creates the job, naming its evaluator and any hook from the sets the server accepts, and funds it. Reclaims after expiry. |
| **provider** | resource server (`payTo`) | Advertises which evaluators and hooks it accepts, prices the job, delivers the resource, submits the deliverable commitment. |
| **evaluator** | an address the client names in the job, from the set in `accepts[].extra.evaluators` | Completes (pays the provider) or rejects (refunds the client). MUST NOT be the provider. MAY be the client. |
| **facilitator** | relayer | Submits role-signed operations. Holds no role in the job and cannot move funds by itself. |

## Lifecycle operations

| Operation | Effect | Who | Relayed by facilitator |
| --- | --- | --- | --- |
| `create` | Opens the job naming provider, evaluator, hook, expiry and terms. Moves no funds. | client | No — the client submits it before paying. |
| `fund` | Fixes the job's price and escrows the client's funds. | client's authorization + provider's price commitment | Yes — the collect settle. |
| `submit` | Commits the provider's deliverable; moves the job to evaluation. | provider | Yes under `escrow`; out of band under `upfront`. |
| `complete` | Releases the escrow to the provider. | evaluator | No — not an x402 operation. |
| `reject` | Returns the escrow to the client. While the job is still unfunded, closes it with nothing to return. | evaluator once funded; client **or provider** while unfunded | No. |
| `reclaim` | Client recovers the escrow after the job's expiry passes without completion. | anyone, pays the client | No — the client's unilateral escape hatch. |

`fund` is the only operation with a client payload. `submit` is authored by the resource server. `complete` and `reject` are between the evaluator and the network; the scheme fixes what they are attesting to (the deliverable), not how the evaluator is invoked.

## Payment flows

| `extra.paymentFlow` | Ordering | Lifecycle |
| --- | --- | --- |
| `escrow` (default) | settle → resource → settle | `fund` before the resource runs; `submit` after it returns, committing the response. `complete` / `reject` / `reclaim` later. |
| `upfront` | settle → resource → respond | `fund` before the resource runs; the server submits out of band. |

`authorization` (verify → resource → settle) is not defined in this version. It could be supported by verifying the client's funding authorization before resource execution and atomically funding and submitting the job afterwards. Doing so, however, means the provider performs the work before the budget is durably escrowed; until settlement, the authorization may expire, be cancelled, or otherwise become unexecutable. This version therefore defines only flows that establish the funded state before resource execution, preserving the scheme's pre-execution escrow guarantee. A later version MAY define `authorization`.

## Core properties

**Fund safety.** The amount escrowed is exactly the client-authorized budget. Within the x402 lifecycle, funds leave escrow only to the provider (on completion) or to the client (on rejection or expiry). No party can direct them elsewhere, and the facilitator cannot move them at all.

**Third-party release.** Within the x402 lifecycle, the provider cannot release its own payment. The address that releases is fixed in the client-created job and MUST differ from the provider.

**Scope of the scheme.** `job-escrow` does not expose ERC-8183 claim-settlement operations. Within the x402 lifecycle, funds are released only through terminal evaluation. Parties MAY use claim settlement out of band; if they do, terminal completion, rejection, and expiry apply to the remaining unsettled budget, and the properties above hold for that remainder.

**Provider protection.** Once funded, the client cannot withdraw before the job's expiry. The provider is protected for the whole delivery window it agreed to.

**No stranding before funding.** A job that is created but not funded holds nothing. Either the client or the provider can close it at any time before funding; the server SHOULD close a job it refuses.

**The offer is verified against the job.** The server advertises the terms it will serve under; the client creates the job; the server reads the job back and refuses to fund one that does not match. An `accepts[]` entry is therefore a constraint set the client resolves into one job — the same relationship `upto` has between the advertised ceiling and the settled amount — and the signed object is always the concrete job.

**Payment identity.** Each job has a unique network-assigned identity, funded at most once and consumed by at most one resource request.

**Deliverable commitment.** The provider commits a hash of what it delivered, and the client receives that commitment with the response, together with the method that produced it. The client can therefore recompute the commitment from what it received, and present the delivered work and the method to the evaluator as evidence; the evaluator needs nothing else from the payment layer. The commitment identifies the provider's submitted result. Determining whether that result satisfies the job is outside the scheme: evaluators MAY use the job description, application-layer data, signed offers or receipts, TLS transcript proofs, zero-knowledge proofs, or any other evidence agreed by the parties.

**Expiry enforcement.** A single absolute expiry bounds funding, submission, and the client's `reclaim`. The server publishes a floor (`extra.jobExpiresAt`); the client MAY set a later one. A binding MAY add an evaluation grace period after it.

**Replay protection.** Every role-signed operation is single-use and bound to the job it names.

## Relationship to other schemes

| Aspect | `auth-capture` | `job-escrow` |
| --- | --- | --- |
| Who releases to the payee | the operator (payee-side) | the evaluator (third party) |
| Payee's own release path | `capture` up to the hold | none within the scheme |
| Partial release | `capture` any amount | none within the scheme (see Appendix: claim settlement) |
| Facilitator's power | is or emulates the operator | none — relays role-signed calls only |
| Refund | operator-funded, needs new liquidity | escrow returns to the client on `reject` / `reclaim` |
| Policy | operator contract | per-job hook from the set the server accepts |
| Client gas | none | one transaction to create the job |

## Appendix

### Network requirements

Every `job-escrow` network binding MUST specify:

1. **Escrow contract** — the job primitive, its state machine, and how each role is authenticated.
2. **Job creation** — what the client submits to create the job, and which offered terms it MUST carry into it.
3. **Client authorization format** — the payload the client produces to fund the job it created.
4. **Provider price commitment** — how the server's price reaches the chain and is bound to the client's funding.
5. **Evaluator acceptance** — how the server advertises the evaluators it accepts and how a job's evaluator is checked against that.
6. **Policy acceptance** — how the server advertises the job-level policy (hooks or equivalent) it accepts, and how a job's policy is checked against that.
7. **Deliverable transport** — how the commitment and its method are carried to the network and returned to the client, and which methods the binding defines.
8. **Per-operation verification and settlement** — the checks a facilitator runs, and the calls it makes.
9. **Expiry and grace** — how `jobExpiresAt` maps onchain and any evaluation grace period.

### Claim settlement (future)

ERC-8183 also defines incremental **claim settlement** over the escrowed budget (`submitClaim` / `settleClaim` / `approveClaim` / `rejectClaim`). This version does not expose it as x402 operations; it remains available on the underlying job, and the scheme's guarantees apply to the unsettled remainder. It maps naturally onto metered work and could be added later as a further payment flow of this scheme or as a binding of `upto`; nothing in this version precludes it.

### Network bindings

- [`scheme_job_escrow_evm.md`](./scheme_job_escrow_evm.md) — EVM, on ERC-8183.

## Version History

| Version | Date | Changes | Authors |
| --- | --- | --- | --- |
| v0.1 | 2026-09-24 | Initial draft | — |

