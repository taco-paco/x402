# Scheme: `job-escrow`

## Summary

`job-escrow` is a payment scheme in which the client funds a **job** into escrow, the resource server delivers and commits to a **deliverable**, and an **evaluator** — never the resource server itself — releases the escrow to the server or returns it to the client. Where `auth-capture` lets the payee's operator decide when to take held funds, `job-escrow` is for payments whose release the client will not leave to the payee: within the scheme, the payee can only submit work, and only the evaluator the client named in the job — an address distinct from the payee, which MAY be the client itself — can pay it out.

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

**Fund safety.** The amount escrowed is exactly the client-authorized budget. Within the x402 lifecycle, funds leave escrow only to the provider-side payout recipient selected by the provider (on completion), or return to the client (on rejection or expiry). The facilitator cannot select or alter either destination.

**Evaluator-controlled release.** Within the x402 lifecycle, the provider cannot release its own payment. The evaluator named in the client-created job controls terminal completion or rejection and MUST differ from the provider. The evaluator MAY be the client.

**Scope of the scheme.** `job-escrow` does not expose ERC-8183 claim-settlement operations. Within the x402 lifecycle, funds are released only through terminal evaluation. Parties MAY use claim settlement out of band; if they do, terminal completion, rejection, and expiry apply to the remaining unsettled budget, and the properties above hold for that remainder. A network binding MUST state how out-of-band claim activity interacts with the client's expiry refund, since a pending claim can delay it.

**What the second settle finalizes.** Under `escrow`, the first settle fixes the amount — the full budget, with no partial release in the scheme — and the second fixes the deliverable. Together they determine everything a payment flow can determine for this scheme; the evaluator's verdict that follows is adjudication of the delivered work, outside the payment protocol in the way a chargeback is outside a card authorization. This is the sense in which the second settle is `escrow`'s "final charge".

**Provider protection.** Once funded, there is no client-role withdrawal path before expiry. Before expiry, funds move only through evaluator-controlled completion or rejection.

**No stranding before funding.** A job that is created but not funded holds nothing. Either the client or the provider can close it at any time before funding; the server SHOULD close a job it refuses.

**The offer is verified against the job.** The server advertises the terms it will serve under; the client creates the job; the server reads the job back and refuses to fund one that does not match. An `accepts[]` entry is therefore a constraint set the client resolves into one job — the same relationship `upto` has between the advertised ceiling and the settled amount — and the signed object is always the concrete job.

**Payment identity.** Each job has a unique network-assigned identity, funded at most once and consumed by at most one resource request.

**Deliverable commitment.** A network binding MUST define a deterministic derivation of the submitted deliverable commitment from the resource content, such that the server and the client compute it over identical bytes. The provider commits that value to the network through the settlement mechanism; a client can independently derive the expected commitment from the content it received and compare it with the provider's submitted commitment. Nothing scheme-specific is returned in the response for this purpose: the committed value is on the network, and a copy of it from the server would carry no information the client lacks. Two things are kept distinct: the **submitted deliverable** is payment and job state; **proof of what was actually returned** is evaluation evidence. A mismatch between the two does not by itself establish misconduct. Establishing what content the provider actually returned is outside the payment scheme and MAY rely on evaluator or application evidence such as TLS transcript proofs, signed application receipts, application-layer evidence, zero-knowledge proofs, or other evidence agreed by the parties.

**Expiry enforcement.** The job expiry, published by the server, is the absolute deadline for the active job lifecycle, including the provider's ability to submit; the client's `reclaim` opens after it. A binding MAY define an evaluation grace period after submission. `maxTimeoutSeconds` bounds the client's initial payment authorization and the funding settlement; it does not bound the later escrow lifecycle, which is governed by the job expiry. The evaluator's verdict is outside both.

**Replay protection.** Every role-signed operation is single-use and bound to the job it names.

## Relationship to other schemes

| Aspect | `auth-capture` | `job-escrow` |
| --- | --- | --- |
| Who releases to the payee | the operator (payee-side) | the evaluator (distinct from the payee) |
| Payee's own release path | `capture` up to the hold | none within the scheme |
| Partial release | `capture` any amount | none within the scheme (see Appendix: claim settlement) |
| Facilitator's power | is or emulates the operator | none — relays role-signed calls only |
| Refund | operator-funded, needs new liquidity | escrow returns to the client on `reject` / `reclaim` |
| Policy | operator contract | per-job hook from the set the server accepts |
| Client gas | none | one transaction to create the job |

## Appendix

### Network requirements

Every `job-escrow` network binding MUST specify:

1. **Escrow contract profile** — the job primitive, its state machine, how each role is authenticated, and the exact contract surface the binding depends on, since ERC-8183 leaves hooks, grace periods, claim settlement, and administration to implementations.
2. **Job creation** — what the client submits to create the job, and which offered terms it MUST carry into it.
3. **Client authorization format** — the payload the client produces to fund the job it created.
4. **Provider price commitment** — how the server's price reaches the chain and is bound to the client's funding.
5. **Evaluator acceptance** — how the server advertises the evaluators it accepts and how a job's evaluator is checked against that.
6. **Policy acceptance** — how the server advertises the job-level policy (hooks or equivalent) it accepts, and how a job's policy is checked against that.
7. **Deliverable derivation** — the fixed rule deriving the commitment from the delivered content, and how the provider commits it to the network.
8. **Per-operation verification and settlement** — the checks a facilitator runs, and the calls it makes.
9. **Expiry, timeout, and grace** — how `maxTimeoutSeconds` bounds the client's initial payment authorization and funding settlement, how the absolute job expiry bounds subsequent lifecycle operations, and any evaluation grace period.
10. **Out-of-band claim activity** — how claim settlement on the underlying job, where the escrow supports it, interacts with the client's expiry refund.

### Claim settlement (future)

ERC-8183 also defines incremental **claim settlement** over the escrowed budget (`submitClaim` / `settleClaim` / `approveClaim` / `rejectClaim`). This version does not expose it as x402 operations; it remains available on the underlying job, and the scheme's guarantees apply to the unsettled remainder. It maps naturally onto metered work and could be added later as a further payment flow of this scheme or as a binding of `upto`; nothing in this version precludes it.

### Network bindings

- [`scheme_job_escrow_evm.md`](./scheme_job_escrow_evm.md) — EVM, on ERC-8183.

## Version History

| Version | Date | Changes | Authors |
| --- | --- | --- | --- |
| v0.1 | 2026-09-24 | Initial draft | — |

