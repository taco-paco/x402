# Scheme: `job-escrow` on `EVM`

## Summary

This is the EVM binding of [`job-escrow`](./scheme_job_escrow.md). It specifies the contract profile, wire fields, signatures, and facilitator logic that realize the scheme on EVM chains.

The binding builds on [ERC-8183](https://eips.ethereum.org/EIPS/eip-8183) with its Signed Authorizations extension. The client creates the job with a direct `createJob` call; every later operation is a `*WithAuthorization` entry point, where the contract verifies an EIP-712 signature from the acting role and executes the core function **with the signer as the acting party**. The facilitator is `msg.sender` of nothing that matters — it pays gas and holds no role.

There is no canonical deployment. ERC-8183 is a Draft standard that leaves hooks, grace periods, claim settlement, and administration to implementations, so this binding pins the surface it depends on as an [escrow profile](#escrow-profile). A facilitator advertises the escrows it relays for, each with its profile, in `/supported`; a server MUST name one of them in `extra.escrow`.

## Roles onchain

| Scheme role | ERC-8183 field | Set by | Onchain authentication |
| --- | --- | --- | --- |
| client | `job.client` | `createJob` caller | `FundAuthorization` signer MUST equal `job.client` |
| provider | `job.provider` | client, at `createJob` | `SetBudgetAuthorization`, `SetPayoutReceiverAuthorization`, and `SubmitAuthorization` signers MUST equal `job.provider` |
| evaluator | `job.evaluator` | client, at `createJob` | `complete` / `reject` callers MUST equal `job.evaluator` |

The contract enforces `client != provider` and `provider != evaluator`. A resource server therefore **cannot** be its own evaluator, and a client cannot buy from itself.

`payTo` identifies the ERC-8183 `provider`, not necessarily the final provider-side payout address. The provider MAY set an ERC-8183 `payoutReceiver` before funding (see [Completing the payload](#completing-the-payload-for-funding)); this changes neither the provider's role nor the client's payment obligation.

## Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant P as Provider
    participant F as Facilitator
    participant E as ERC-8183
    participant V as Evaluator

    C->>P: Request
    P-->>C: PaymentRequired(job-escrow)

    C->>E: createJob
    E-->>C: jobId

    C->>C: sign FundAuthorization
    C->>P: PaymentPayload with jobId and FundAuthorization

    P->>P: validate job locally and sign SetBudgetAuthorization
    P->>F: settle fund with completed payload

    opt payout receiver requested
        F->>E: setPayoutReceiverWithAuthorization
    end
    F->>E: setBudgetWithAuthorization
    F->>E: fundWithAuthorization
    E-->>F: Funded
    F-->>P: success

    P->>P: Perform work

    P->>P: sign SubmitAuthorization
    P->>F: settle submit
    F->>E: submitWithAuthorization
    E-->>F: Submitted
    F-->>P: success

    P-->>C: Resource

    V->>E: complete or reject
```

## Escrow profile

A facilitator relays only into escrows it has admitted, and admission is against a **profile**: the contract surface this binding calls and reads. This version defines one profile, `job-escrow-evm-1`. The ERC-8183 reference implementation satisfies it.

An escrow satisfies `job-escrow-evm-1` when it implements [ERC-8183](https://eips.ethereum.org/EIPS/eip-8183) — the job state machine, roles, core functions, claim settlement, and events as that specification defines them — **and** resolves the following points the standard leaves open or does not name:

| Point | ERC-8183 | `job-escrow-evm-1` requires |
| --- | --- | --- |
| Signed Authorizations extension | OPTIONAL (SHOULD) | REQUIRED, with the EIP-712 domain `{ name: "ERC8183", version: "1", chainId, verifyingContract: escrow }`, the types in [EIP-712 types used](#eip-712-types-used), and nonces packed as `bytes32((uint256(uint160(signer)) << 96) \| uint256(nonce))` |
| Hooks | OPTIONAL | REQUIRED: `createJob` reverts for a non-whitelisted hook, and `address(0)` is whitelisted |
| Claim settlement | defined; some behaviour implementation-dependent | REQUIRED: `rejectClaim` callable by the client; `submit` and terminal `reject` clear a pending claim; `claimRefund` reverts while a claim is pending on a `Funded` job and is not blocked on a `Submitted` one |
| Evaluation grace period | MAY be any length or omitted | REQUIRED to exist, so that a `Submitted` job cannot be force-refunded the moment it expires |
| Funding balance check | SHOULD | REQUIRED: `fund` reverts unless the escrow's balance increases by exactly `budget` |
| `JobStatus` ordinals | not fixed | `Open = 0, Funded = 1, Submitted = 2, Completed = 3, Rejected = 4, Expired = 5` |
| Event signatures | names only | `BudgetSet(jobId, token, amount)`, `PayoutReceiverSet(jobId, payoutReceiver)`, `JobFunded(jobId, client, amount)`, `JobSubmitted(jobId, provider, deliverable)`, `JobCompleted(jobId, evaluator, reason)`, `JobRejected(jobId, rejector, reason)`, `JobExpired(jobId)`, `PaymentReleased(jobId, recipient, amount)`, `Refunded(jobId, client, amount)`, with `jobId` topic-indexed |

and exposes the **views** this binding reads. ERC-8183 specifies the job's fields and the escrow's allowlists but not how they are read; the reference implementation exposes them as public state, and the profile fixes those accessors:

```
getJob(uint256 jobId) returns (Job)        // client, status, provider, expiredAt, evaluator, submittedAt, budget, hook, paymentToken, providerAgentId, description, settledAmount, payoutReceiver
whitelistedHooks(address) returns (bool)
allowedPaymentTokens(address) returns (bool)
authorizationNonceUsed(bytes32) returns (bool)
paused() returns (bool)
```

Where this binding names an ERC-8183 function, revert, or event below, it means the one that specification defines. An escrow that omits any required point above does not satisfy `job-escrow-evm-1`; a profile for it would be a separate definition.

A facilitator MUST NOT advertise an escrow under a profile it has not verified the escrow satisfies. Because a profile is checked against a deployment that may be upgradeable, admission is a review of the deployment and its admin, not of the code alone.

## PaymentRequirements

```json
{
  "x402Version": 2,
  "resource": {
    "url": "https://api.example.com/summarize",
    "description": "Summarize the POSTed document in at most 500 words, preserving all figures and dates."
  },
  "accepts": [
    {
      "scheme": "job-escrow",
      "network": "eip155:8453",
      "amount": "1000000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xProviderAddress",
      "maxTimeoutSeconds": 120,
      "extra": {
        "name": "USD Coin",
        "version": "2",
        "escrow": "0xErc8183EscrowAddress",
        "evaluators": ["0xEvaluatorA", "0xEvaluatorB", "client"],
        "hooks": ["0x0000000000000000000000000000000000000000", "0xReputationHook"],
        "jobExpiresAt": 1740762154,
        "providerAgentId": "0",
        "assetTransferMethod": "erc20-allowance",
        "paymentFlow": "escrow"
      }
    }
  ]
}
```

`PaymentRequirements` is the **offer**. It names every term the client commits to when it creates and funds the job. Material the server adds to execute that offer — its own signed authorizations — goes into `PaymentPayload.payload`, never into the requirements (see [Completing the payload](#completing-the-payload-for-funding)).

`resource.description` is the core `ResourceInfo` field and becomes the job's onchain description, see [Job description](#job-description).

### `extra` fields

| Field | Required | Type | Description |
| --- | --- | --- | --- |
| `name` | No | `string` | EIP-712 token-domain name, for an EIP-2612 `permit`. REQUIRED if the server expects clients without a standing allowance. |
| `version` | No | `string` | EIP-712 token-domain version, for an EIP-2612 `permit`. |
| `escrow` | Yes | `address` | The escrow contract. MUST be one the facilitator advertises for this network, under a profile this binding defines. |
| `evaluators` | Yes | `string[]` | Non-empty list of evaluators the server accepts, see [Evaluators](#evaluators). Each entry is an address, `"*"`, or `"client"`. Order is the server's preference. |
| `hooks` | No | `string[]` | List of job hooks the server accepts, see [Hooks](#hooks). Each entry is an address (the zero address meaning "no hook") or `"*"`. Default `["0x0000000000000000000000000000000000000000"]`. Order is the server's preference. |
| `jobExpiresAt` | Yes | `uint48` | Absolute Unix seconds; onchain `job.expiredAt`, exactly. The outer deadline for the job lifecycle, including `submit`. When constructing the offer the server MUST satisfy `jobExpiresAt > now + max(maxTimeoutSeconds, 300)`, see [`maxTimeoutSeconds` and `jobExpiresAt`](#maxtimeoutseconds-and-jobexpiresat). |
| `description` | If `resource.description` is absent | `string` | The job brief, see [Job description](#job-description). MUST NOT be present when `resource.description` is. |
| `providerAgentId` | No | `uint256` string | Onchain `job.providerAgentId`, the server's ERC-8004 agent identity. Default `"0"`. |
| `paymentFlow` | Yes | `"escrow"` \| `"upfront"` | `"authorization"` MUST be rejected. |
| `assetTransferMethod` | No | `"erc20-allowance"` | The only method this version defines; inferred when absent. The client's allowance to the escrow, set by `approve` or by an EIP-2612 `permit` carried in the payload. |

`amount` is the gross job budget in atomic units and is what the client escrows. Platform and evaluator fees, if the escrow charges them, are deducted from the **payout**, not added to the client's charge; a server reads `platformFeeBP` and `evaluatorFeeBP` from the escrow to know its net. The client is oblivious to them.

### `maxTimeoutSeconds` and `jobExpiresAt`

Two deadlines govern a payment, as in `auth-capture`, where `maxTimeoutSeconds` derives the client's pre-approval expiry and `captureDeadline` bounds the later lifecycle:

| | Derived from | Bounds |
| --- | --- | --- |
| `fundAuthorization.deadline` | `now + maxTimeoutSeconds` at signing | the client's funding authorization and the `fund` settlement, including the server authorizations executed atomically with it |
| `job.expiredAt` | `extra.jobExpiresAt`, fixed at `createJob` | every later job operation: `submit`, claims, and the start of the client's `claimRefund` window |

`maxTimeoutSeconds` therefore bounds the client's payment authorization and the initial funding settlement. It does not bound the later ERC-8183 lifecycle. Under `escrow` the `submit` settlement is provider-authored and is bounded independently by `submitAuthorization.deadline`, a provider-side staleness control, and by `job.expiredAt`; under `upfront` submission happens out of band under the same two bounds. Evaluator `complete` / `reject` and any evaluation grace period are governed by ERC-8183 and are outside both.

The two are related only when the offer is constructed: a server MUST publish `jobExpiresAt > now + max(maxTimeoutSeconds, 300)`, so that the whole funding window fits inside the job and the contract's five-minute creation minimum is met. This is an offer-construction rule; the facilitator does not recompute it against a later clock. The client MUST NOT construct a payload whose `fundAuthorization.deadline` exceeds `job.expiredAt`.

### Job description

ERC-8183 stores a `description` on every job — "a job brief, scope reference" — set by the client at creation. This binding takes it from the 402 rather than defining a field of its own: the job description is `resource.description` of the `PaymentRequired` (the core [`ResourceInfo`](../../x402-specification-v2.md#5-types) field, "human-readable description of the resource"), and when the server publishes none, `accepts[].extra.description`, which is then REQUIRED. The client MUST pass the resolved value verbatim to `createJob`, and the facilitator checks its hash at settlement. It is the brief the evaluator will judge against, so it SHOULD state what the resource delivers in terms a human or an evaluator can check. It is stored onchain at the client's gas cost, so a server SHOULD keep it short, and MAY point to fuller terms by URI within it.

### Evaluators

`extra.evaluators` names the evaluators the server will serve a job under. The client picks one and commits it at `createJob`; the job's evaluator is checked against the list before funding.

| Entry | `job.evaluator` accepted when |
| --- | --- |
| an address | `job.evaluator == entry` |
| `"client"` | `job.evaluator == job.client` — the client judges its own job |
| `"*"` | any address. The contract already excludes `payTo`. |

The list is ordered: the first entry is the server's preference, and a client SHOULD choose it absent a reason of its own. `"*"` MUST mean what it says: a server advertising it MUST NOT refuse a job on the ground of its evaluator. A server that would refuse some evaluators MUST enumerate the ones it accepts. A server whose acceptable set is not enumerable — because it is defined by a registry, a reputation threshold, or delegation to a routing evaluator — advertises the routing contract's address (see [Evaluator routing](#evaluator-routing-non-normative)), or waits for a registry-valued entry to be defined; it does not advertise `"*"`.

A list is an `accepts[]` entry's constraint, not a menu of entries: each entry remains independently selectable and the client resolves it to one concrete job before anything is signed, exactly as it resolves `upto`'s ceiling to one settled amount. A server that wants to pair different evaluators with different prices or expiries lists several `accepts[]` entries.

### Hooks

ERC-8183 attaches an OPTIONAL per-job hook, chosen by the client at `createJob` from the escrow's admin-managed whitelist; `createJob` reverts `HookNotWhitelisted` otherwise. A hook is client-side policy: it MAY revert `setBudget`, `fund`, `submit`, and the claim-settlement functions, and receives `optParams` on each.

`extra.hooks` names the hooks the server will serve a job under, with the same shape and rules as `extra.evaluators`:

| Entry | `job.hook` accepted when |
| --- | --- |
| an address | `job.hook == entry`. The zero address means a job with no hook. |
| `"*"` | any hook the escrow whitelists, including none. |

The default when absent is the zero address alone: no hook. A server that lists hooks but not the zero address **requires** one of them. `"*"` MUST mean what it says. In every case the facilitator also confirms `escrow.whitelistedHooks(job.hook)`; that check is defensive, since the job could not otherwise exist, and fails only if the escrow's admin removed the hook after creation.

Hooks consume `optParams`, and the ERC-8183 authorizations that take `optParams` bind `optParamsHash`. Each such authorization this binding carries therefore has an OPTIONAL `optParams` (hex bytes, default `0x`), supplied by the party that signs it: the client's on `fundAuthorization`, the server's on `setBudgetAuthorization` and `submitAuthorization`. `SetPayoutReceiverAuthorization` takes none. What a hook expects is documented by the hook, not by this binding.

## Job creation

Before paying, the client calls, from the address it will sign with:

```
escrow.createJob(
  provider        = payTo,
  evaluator       = one of extra.evaluators, resolved as above,
  expiredAt       = extra.jobExpiresAt,
  description     = resource.description, or extra.description when that is absent,
  hook            = one of extra.hooks, resolved as above,
  providerAgentId = extra.providerAgentId
)  → jobId
```

The client pays gas for this one call. It moves no funds. A client that lacks a standing allowance of at least `amount` from itself to `escrow` on `asset` either includes `approve(escrow, ≥ amount)` alongside (one-time per token, as Permit2 setup is in `exact`), or signs an EIP-2612 `permit` in the payload.

A created job that is never funded is `Open` and holds nothing. While `Open`, **either the client or the provider** MAY `reject(jobId, reason)` it; otherwise it becomes `Expired` after `expiredAt`, still holding nothing. A server that refuses a job before funding SHOULD `reject` it, so the client's open jobs reflect what can still be funded.

## Client payment payload

The client names no operation; the payload settles as `fund` under both flows.

```json
{
  "x402Version": 2,
  "resource": { "url": "https://api.example.com/summarize", "method": "POST" },
  "accepted": { "scheme": "job-escrow", "...": "..." },
  "payload": {
    "jobId": "4821",
    "fundAuthorization": {
      "signer": "0xClientAddress",
      "nonce": "0x0000000000000000f00e",
      "deadline": 1740758274,
      "optParams": "0x",
      "signature": "0x8b04...c3e1"
    },
    "permit": {
      "value": "1000000",
      "deadline": 1740758274,
      "signature": "0x71aa...0d5f"
    }
  }
}
```

| Payload field | Derived from |
| --- | --- |
| `jobId` | The id returned by the client's `createJob`, as a decimal string. |
| `fundAuthorization` | `FundAuthorization(signer, jobId, expectedToken = asset, expectedBudget = amount, optParamsHash = keccak256(optParams), nonce, deadline)` in the escrow's domain. |
| `fundAuthorization.signer` | `job.client` — the address that called `createJob`. ERC-1271 signers are valid. |
| `fundAuthorization.nonce` | Fresh random `uint72`; single-use across all ERC-8183 actions for this signer on this escrow. |
| `fundAuthorization.deadline` | `now + maxTimeoutSeconds`; MUST be `<= job.expiredAt`. See [`maxTimeoutSeconds` and `jobExpiresAt`](#maxtimeoutseconds-and-jobexpiresat). |
| `fundAuthorization.optParams` | OPTIONAL, default `0x`. Bytes for the job's hook on `fund`. |
| `permit` | OPTIONAL. EIP-2612 `Permit(owner = signer, spender = extra.escrow, value >= amount, nonce = token.nonces(owner), deadline)` in the token's domain `{ name, version, chainId, verifyingContract = asset }`. Omitted when `token.allowance(signer, escrow) >= amount`. |

`fundAuthorization` is both the payment and the proof: `fundWithAuthorization` reverts `Unauthorized` unless `signer == job.client`, so a `jobId` observed onchain cannot be presented by anyone but the address that created it.

## Completing the payload for funding

ERC-8183 lets only the provider set a job's budget, and a job's budget cannot be set before the job exists. The server therefore completes the client's payload before passing it to `settle(fund)`: it leaves the client's fields untouched and appends its own authorizations. **`PaymentRequirements` is the offer; `PaymentPayload.payload` is the scheme-specific material used to execute that offer.** Nothing the server signs goes into the requirements.

Before signing, the server SHOULD validate the job locally against the offer — the same checks the facilitator will run in [Verification](#verification) steps 4–7, all read-only — so that it does not spend a nonce on a job it will refuse.

| Added field | Required | Description |
| --- | --- | --- |
| `setBudgetAuthorization` | Yes, unless the budget is already set | `SetBudgetAuthorization(signer = payTo, jobId, token = asset, amount, optParamsHash = keccak256(optParams), nonce, deadline)`. The server cannot price the job differently from the offer: `fundWithAuthorization` reverts unless `job.budget == amount` and `job.paymentToken == asset`, which the client signed. |
| `setPayoutReceiverAuthorization` | No | `SetPayoutReceiverAuthorization(signer = payTo, jobId, payoutReceiver, nonce, deadline)`, carrying the `payoutReceiver` value itself. Routes provider-side payouts to another address. Provider-internal; not advertised to the client, whose obligation is to `payTo` as `provider`. MUST be executed while the job is `Open`, so it is part of the fund settle or not at all. |

Both are signed in the escrow's domain by `payTo` with fresh, distinct `uint72` nonces and `deadline == fundAuthorization.deadline`: they execute atomically with `fund`, so an independent deadline would add nothing.

A self-facilitating server MAY instead call `setBudget` (and `setPayoutReceiver`) directly before `settle(fund)`; the facilitator then finds `job.budget == amount && job.paymentToken == asset` already set and omits the call. A budget set on a job that then fails to fund is harmless: the job is `Open`, and the client's next `fund` for the same terms still matches.

### Completed payload

```json
{
  "x402Version": 2,
  "resource": { "url": "https://api.example.com/summarize", "method": "POST" },
  "accepted": { "scheme": "job-escrow", "...": "..." },
  "payload": {
    "jobId": "4821",
    "fundAuthorization": {
      "signer": "0xClientAddress",
      "nonce": "0x0000000000000000f00e",
      "deadline": 1740758274,
      "optParams": "0x",
      "signature": "0x8b04...c3e1"
    },
    "permit": {
      "value": "1000000",
      "deadline": 1740758274,
      "signature": "0x71aa...0d5f"
    },
    "setBudgetAuthorization": {
      "signer": "0xProviderAddress",
      "nonce": "0x0000000000000000f00a",
      "deadline": 1740758274,
      "optParams": "0x",
      "signature": "0x6b03...c2a7"
    },
    "setPayoutReceiverAuthorization": {
      "signer": "0xProviderAddress",
      "payoutReceiver": "0xProviderTreasury",
      "nonce": "0x0000000000000000f00b",
      "deadline": 1740758274,
      "signature": "0x91de...04ff"
    }
  }
}
```

## Lifecycle payload: `submit`

Under `escrow`, the second settle carries the server-authored `submit`:

```json
{
  "x402Version": 2,
  "accepted": { "scheme": "job-escrow", "...": "..." },
  "payload": {
    "type": "submit",
    "jobId": "4821",
    "deliverable": "0x3fa1...77c9",
    "submitAuthorization": {
      "signer": "0xProviderAddress",
      "nonce": "0x00000000000000000abd",
      "deadline": 1740758274,
      "optParams": "0x",
      "signature": "0x44d0...e6b3"
    }
  }
}
```

`submitAuthorization` is `SubmitAuthorization(signer = payTo, jobId, deliverable, optParamsHash = keccak256(optParams), nonce, deadline)` in the escrow's domain. Its `deadline` is the provider's own staleness bound and does not inherit the client's funding deadline; `job.expiredAt` is the outer limit. `payload.type` appears only here.

### Deliverable

`deliverable` is the `bytes32` the provider commits to on `submit`. This version fixes one derivation, so that the commitment is a deterministic function of the delivered content and nothing on the wire can reinterpret it:

> `deliverable = keccak256(content)`, where `content` is the exact resource representation bytes exposed to the x402 client after transport-level decoding and before any application-level deserialization.

The invariant is that the server and the client feed **identical bytes** to `keccak256`: the bytes the server hashes MUST be the bytes the client's x402 layer receives. Each x402 transport defines where that boundary is. Over HTTP it is the message body after transfer coding and content coding have been removed (so `Content-Encoding: gzip` is undone, chunking is reassembled) and before the body is parsed as JSON, text, or any other media type. A resource that cannot present one deterministic byte representation reproducible on both sides — streaming responses, side-effect-only resources, bodies that vary by connection or negotiation — MUST NOT be offered under this version's derivation. Provider-defined derivations are a candidate for a later version; a hook's `optParams` is where a method identifier would be bound if one were introduced.

The commitment is not returned to the client as response metadata. It is a deterministic function of content the client already holds, and a server-authored copy of it would prove nothing; the only authenticated record is the onchain one. The client therefore verifies against the chain: it computes `keccak256` over the content it received and compares it with the `deliverable` in `JobSubmitted(jobId, provider, deliverable)` on the escrow. The flows differ only in when that event exists:

- `escrow` — before the response: the facilitator settles `submit`, and the response's `transaction` is the submit transaction, so the event is already emitted and readable from the receipt.
- `upfront` — after the response, out of band, through the same `SubmitAuthorization` mechanism; the client observes the event when the provider submits.

The client-side rule is therefore:

```
expectedDeliverable = keccak256(content)
expectedDeliverable ?= JobSubmitted.deliverable
```

Where the two disagree, x402 exposes the disagreement — but a mismatch does not by itself prove provider misconduct. The onchain `deliverable` is payment state: it records what the provider *submitted*. What the provider actually *returned* is a separate fact that the payment layer does not witness, and establishing it is a question of evidence available to the evaluator — TLS transcript proofs, signed application receipts, application-layer logs, zero-knowledge proofs, or whatever the parties agreed. x402 defines the derivation, funds the job, records the submitted commitment, and exposes the settlement transaction and job state; the rest is the evidence layer's.

### What the second settle finalizes

The core `escrow` flow describes its second settle as recording "the final charge". Under `job-escrow` the **amount** is final at the first settle — `budget == amount`, and the scheme exposes no partial release — and the **deliverable** is final at the second. After `submit` the provider can neither change what it delivered nor withdraw it, the client can no longer reclaim on expiry without the evaluation grace period, and the payment is fully determined except for the evaluator's verdict. That verdict is adjudication, structurally outside the payment protocol in the way a chargeback is outside a card authorization. The second settle therefore finalizes everything a payment flow can finalize for this scheme, which is the sense in which it is `escrow`'s second settle.

## Verification

Both flows of this scheme omit `/verify` from their ordering: the first `/settle` is the pre-resource check. The facilitator runs every step below on `settle(fund)`. A server that wants a pre-check validates the job locally, read-only, before signing.

### Common

1. **Accepted requirements match**: `payload.accepted` represents the same `PaymentRequirements` supplied to settlement. The facilitator MUST compare, after applying this binding's defaults, `scheme`, `network`, `amount`, `asset`, `payTo`, `maxTimeoutSeconds`, `extra.escrow`, `extra.evaluators`, `extra.hooks`, `extra.jobExpiresAt`, `extra.providerAgentId`, `extra.paymentFlow`, `extra.assetTransferMethod`, and the resolved job description. An absent field and its default are equal: `hooks` ≡ `["0x0…0"]`, `providerAgentId` ≡ `"0"`, `assetTransferMethod` ≡ `"erc20-allowance"`. The server MAY append execution material to `payload`; it MUST NOT alter the requirements the client accepted.
2. **Extra validation**: `extra.escrow` is advertised for this network under a profile this binding defines; `extra.evaluators` is non-empty and every address entry is non-zero and `!= payTo`; `hooks`, if present, is non-empty; `paymentFlow` is `escrow` or `upfront`; `assetTransferMethod` is absent or `erc20-allowance`.
3. **Escrow state**: `escrow.paused() == false`; `escrow.allowedPaymentTokens(asset) == true`.

### `fund`

4. **Shape guard**: `jobId` and `fundAuthorization` present; `permit` present or `token.allowance(signer, escrow) >= amount`; `setBudgetAuthorization` present unless the budget is already set (step 7).
5. **Job exists**: read `job = escrow.getJob(jobId)`; a revert is `invalid_job_escrow_evm_job_not_found`.
6. **Job matches the offer**: `job.client == fundAuthorization.signer`; `job.provider == payTo`; `job.evaluator` is accepted by `extra.evaluators` per [Evaluators](#evaluators); `job.hook` is accepted by `extra.hooks` per [Hooks](#hooks) and `escrow.whitelistedHooks(job.hook) == true`; `keccak256(job.description) == keccak256(resolved description)` per [Job description](#job-description); `job.providerAgentId == extra.providerAgentId`; `job.expiredAt == extra.jobExpiresAt`.
7. **Job state**: `job.status == Open`; `now < job.expiredAt`; `job.paymentToken` is `address(0)` or `asset`; `job.budget` is `0` or `amount`.
8. **Fund authorization**: signature recovers (ECDSA or ERC-1271) to `signer` over the `FundAuthorization` digest built from `(jobId, asset, amount, keccak256(optParams), nonce, deadline)`; `now < deadline <= job.expiredAt`; `escrow.authorizationNonceUsed(pack(signer, nonce)) == false`.
9. **Set-budget authorization**: if the budget is unset, `setBudgetAuthorization` recovers to `payTo` over `(jobId, asset, amount, keccak256(optParams), nonce, deadline)`; nonce unused; `deadline == fundAuthorization.deadline`.
10. **Set-payout-receiver authorization**: if present, recovers to `payTo` over `(jobId, payoutReceiver, nonce, deadline)`; nonce unused and distinct from the set-budget nonce; `deadline == fundAuthorization.deadline`; `payoutReceiver` is neither the escrow nor `asset`.
11. **Permit**: if present, recovers to `signer` over the token's `Permit` digest with `spender == escrow`, `value >= amount`, `nonce == token.nonces(signer)`, `deadline > now`.
12. **Balance**: `token.balanceOf(signer) >= amount`.
13. **Simulation**: simulate the settlement batch below and confirm `PayoutReceiverSet` (if requested), `BudgetSet(jobId, asset, amount)` (unless already set), and `JobFunded(jobId, signer, amount)` are emitted by `extra.escrow`, and that `getJob(jobId).status == Funded` afterwards. A hooked job's `beforeAction` / `afterAction` run inside this simulation; a reverting hook is reported as `invalid_job_escrow_evm_hook_reverted`.

### `submit`

1. `payload.type == "submit"`; `jobId`, `deliverable != bytes32(0)`, `submitAuthorization` present.
2. `submitAuthorization` recovers to `payTo` over the `SubmitAuthorization` digest with `optParamsHash = keccak256(optParams)`; nonce unused; `now < deadline`.
3. `getJob(jobId)`: `provider == payTo`, `status == Funded`, `now < expiredAt`. A pending claim needs no check: `submit` supersedes it onchain.

## Settlement

### `fund`

The facilitator submits one atomic batch, in this order:

```
[ token.permit(signer, escrow, value, deadline, v, r, s) ]                                                   // only if payload.permit present
[ escrow.setPayoutReceiverWithAuthorization(jobId, payoutReceiver, setPayoutReceiverAuthorization) ]        // only if present
[ escrow.setBudgetWithAuthorization(jobId, asset, amount, setBudgetAuthorization.optParams, setBudgetAuthorization) ]   // unless already set
escrow.fundWithAuthorization(jobId, asset, amount, fundAuthorization.optParams, fundAuthorization)
```

None of these calls reads `msg.sender` for authorization, so the batch MAY go through the canonical Multicall3 `aggregate3` with `allowFailure = false`, or through any facilitator-controlled batcher. The facilitator SHOULD NOT submit the calls as separate transactions; a partial landing is safe for the client (the job stays `Open` and its `fund` nonce unused) but burns the server's nonces for nothing. The facilitator MUST cap the batch's gas, since a hook runs inside it.

After confirmation the facilitator MUST apply the step-13 outcome checks to the receipt and resulting state, and only then report success. It returns a `SettlementResponse` with `success = true`, `transaction` = the batch transaction hash, `network`, `payer = signer`, and `amount`.

### `submit`

The facilitator submits `escrow.submitWithAuthorization(jobId, deliverable, submitAuthorization.optParams, submitAuthorization)`, confirms `JobSubmitted(jobId, payTo, deliverable)`, and returns a `SettlementResponse` with `success = true`, `transaction` = the submit transaction hash, and `network`. It adds no scheme extension.

### After settlement

`complete` and `reject` are the evaluator's. `claimRefund(jobId)` is permissionless once `now >= expiredAt` (`Funded`) or `now >= expiredAt + EVALUATION_GRACE_PERIOD` (`Submitted`) and pays the client the unsettled remainder. None of these is a settle operation.

### Claim settlement and liveness

ERC-8183 claim settlement is part of the profile and remains callable on the job by its roles, but is not exposed by this scheme. Two consequences matter to the guarantees above:

- A provider MAY `submitClaim` while the job is `Funded`. A pending claim blocks `claimRefund`. The client clears it with `rejectClaim`, which the client may always call, and then `claimRefund`: for an **unhooked** job the client's refund after expiry is therefore guaranteed in at most two transactions. For a **hooked** job, a hook that reverts `rejectClaim` can keep the claim pinned and the escrow parked. This is the one path by which a hook can hold a client's funds past expiry, and the reason a client MUST vet a hook's `rejectClaim` behaviour before naming it — or name none.
- A provider MAY `submitClaim` between `fund` and `submit`. It does not affect the x402 flow: `submit` supersedes the pending claim onchain, and the facilitator's `submit` settle needs no claim check.
- Under `upfront` the provider submits after responding. A provider that never submits leaves the job `Funded` until `expiredAt`, when `claimRefund` returns the budget to the client. That is the provider's loss, not the client's: the client already holds the content, and the budget comes back to it. Submitting on time is the provider's only path to being paid.

Partial release through claim settlement, where parties use it out of band, reduces the unsettled remainder to which terminal completion, rejection, and expiry apply.

## SettlementResponse

The [`SettlementResponse`](../../x402-specification-v2.md#53-settlementresponse-schema) is the core type unchanged, and this scheme defines no extension to it. The facilitator returns the base settlement result for the operation it executed — `success`, `transaction`, `network`, `payer`, and `amount` on `fund`; `success`, `transaction`, and `network` on `submit` — and the server returns that result to the client as it is.

Under `escrow` the client receives the `submit` settlement's result, so `transaction` is the submit transaction and the `JobSubmitted` event in its receipt carries the committed `deliverable`. Under `upfront` the client receives the `fund` settlement's result, and observes `JobSubmitted` when the provider later submits. The client needs nothing else from the response: it already holds `jobId`, having created the job, and the content, having received it.

## `/supported`

```json
{
  "kinds": [
    {
      "x402Version": 2,
      "scheme": "job-escrow",
      "network": "eip155:8453",
      "extra": {
        "escrows": [
          { "address": "0xErc8183EscrowAddress", "profile": "job-escrow-evm-1" }
        ]
      }
    }
  ],
  "signers": { "eip155:*": ["0xFacilitatorSignerAddress"] }
}
```

`extra.escrows` lists the escrow deployments the facilitator relays into, each with the [profile](#escrow-profile) it has verified the deployment satisfies. A server MUST NOT name an escrow the facilitator does not list, and a facilitator MUST reject a requirements' `escrow` it does not list with `invalid_job_escrow_evm_escrow_not_supported`.

## Error Codes

Every reason this binding defines is namespaced `invalid_job_escrow_evm_*`; standard reasons keep their canonical names.

| Error | Meaning |
| --- | --- |
| `invalid_job_escrow_evm_accepted_mismatch` | `payload.accepted` differs from the `PaymentRequirements` supplied to settlement in a field this binding compares. |
| `invalid_job_escrow_evm_extra` | A required `extra` field is missing or malformed, `evaluators` is empty or has a zero / `payTo` address entry, `hooks` is present and empty, no job description resolves, `paymentFlow` is `authorization`, or `assetTransferMethod` is not `erc20-allowance`. |
| `invalid_job_escrow_evm_escrow_not_supported` | `extra.escrow` is not one the facilitator advertises for this network. |
| `invalid_job_escrow_evm_escrow_paused` | The escrow is paused. |
| `invalid_job_escrow_evm_token_not_allowed` | `asset` is not on the escrow's payment-token allowlist. |
| `invalid_job_escrow_evm_evaluator_not_accepted` | `job.evaluator` matches no entry of `extra.evaluators`. |
| `invalid_job_escrow_evm_hook_not_accepted` | `job.hook` matches no entry of `extra.hooks`. |
| `invalid_job_escrow_evm_hook_not_whitelisted` | `job.hook` is no longer on the escrow's hook whitelist. |
| `invalid_job_escrow_evm_hook_reverted` | The job's hook reverted in simulation or on chain. |
| `invalid_job_escrow_evm_job_expired` | `now >= job.expiredAt`. |
| `invalid_job_escrow_evm_job_not_found` | `jobId` is out of range on the escrow. |
| `invalid_job_escrow_evm_client_mismatch` | `job.client` is not `fundAuthorization.signer`. |
| `invalid_job_escrow_evm_job_mismatch` | A stored job field (provider, description, agent id, expiry, token, budget) does not match the requirements. |
| `invalid_job_escrow_evm_wrong_status` | The job is not in the state the operation needs. |
| `invalid_job_escrow_evm_authorization_signature` | A `*Authorization` signature does not recover to the required signer. |
| `invalid_job_escrow_evm_authorization_nonce_used` | An authorization nonce is already used or cancelled, or two authorizations in one payload share a nonce. |
| `invalid_job_escrow_evm_authorization_expired` | An authorization or permit `deadline` has passed. |
| `invalid_job_escrow_evm_authorization_deadline_mismatch` | A `setBudgetAuthorization` or `setPayoutReceiverAuthorization` `deadline` differs from `fundAuthorization.deadline`, or `fundAuthorization.deadline > job.expiredAt`. |
| `invalid_job_escrow_evm_insufficient_allowance` | No `permit` and `allowance(signer, escrow) < amount`. |
| `invalid_job_escrow_evm_insufficient_balance` | `balanceOf(signer) < amount`. |
| `invalid_job_escrow_evm_set_budget_authorization_missing` | `settle(fund)` for a job whose budget is unset, with no `setBudgetAuthorization`. |
| `invalid_job_escrow_evm_payout_receiver_invalid` | `payoutReceiver` is the escrow or the payment token. |
| `invalid_job_escrow_evm_payload_type` | A lifecycle payload's `type` is not `submit`. |
| `invalid_job_escrow_evm_deliverable_empty` | `deliverable` is `bytes32(0)`. |
| `invalid_job_escrow_evm_unexpected_outcome` | The simulated or confirmed batch did not emit the expected events or leave the job in the expected state. |

### Contract reverts

| ERC-8183 revert | Reported as |
| --- | --- |
| `InvalidJob` | `invalid_job_escrow_evm_job_not_found` |
| `WrongStatus` | `invalid_job_escrow_evm_wrong_status` |
| `Unauthorized` | `invalid_job_escrow_evm_client_mismatch` (fund) / `invalid_job_escrow_evm_job_mismatch` (setBudget, setPayoutReceiver, submit) |
| `BudgetMismatch`, `PaymentTokenMismatch` | `invalid_job_escrow_evm_job_mismatch` |
| `PaymentTokenNotAllowed` | `invalid_job_escrow_evm_token_not_allowed` |
| `ProviderNotSet` | `invalid_job_escrow_evm_job_mismatch` |
| `InvalidReceiver` | `invalid_job_escrow_evm_payout_receiver_invalid` |
| `HookNotWhitelisted` | `invalid_job_escrow_evm_hook_not_whitelisted` |
| revert bubbled from a hook | `invalid_job_escrow_evm_hook_reverted` |
| `AuthorizationExpired` | `invalid_job_escrow_evm_authorization_expired` |
| `AuthorizationNonceUsed` | `invalid_job_escrow_evm_authorization_nonce_used` |
| `InvalidAuthorizationSignature` | `invalid_job_escrow_evm_authorization_signature` |
| `UnexpectedFundedAmount` | `invalid_job_escrow_evm_unexpected_outcome` |
| `EnforcedPause` | `invalid_job_escrow_evm_escrow_paused` |

## Security Considerations

- **The facilitator holds no role.** Every call it submits is verified onchain against the signer of a role-bound authorization. It cannot fund, submit, complete, reject, redirect a payout, or alter the deliverable. Its only failure modes are not relaying and paying gas for a revert.
- **The evaluator is trusted by both sides.** The server accepts it by listing it (or `"*"`) and bears the risk that it never completes: after `jobExpiresAt` (+ grace once submitted) anyone can `claimRefund` and the client gets its escrow back regardless of delivery. The client chooses it and bears the risk that it completes bad work. Neither can be the provider. A server advertising `"*"` accepts that a client may name an evaluator that never acts; a server that wants no third party can only be paid by an evaluator the client trusts, and a client that wants no third party needs a server that lists `"client"`.
- **A hook is client policy the server has agreed to.** By listing a hook, or `"*"`, the server accepts it. A hook MAY revert `submit` until the job expires, after which `claimRefund` returns the escrow to the client and the provider is unpaid — the same shape as a dead evaluator, and the same answer: the server listed it. A hook MAY also revert `rejectClaim`, which is the client's exposure (see [Claim settlement and liveness](#claim-settlement-and-liveness)). The default of no hook is the safe choice for both.
- **The escrow's admin is a trusted party.** An escrow satisfying the profile is still an upgradeable, pausable, admin-configured contract with fee setters and allowlists. Facilitators, servers, and clients MUST treat admission of an escrow as trust in that admin.
- **Funded before execution.** Both defined flows establish `Funded` before the resource runs. An `authorization` ordering would instead have the provider work against a `FundAuthorization` that has been verified but not executed, and which may expire, be cancelled by its signer, or otherwise become unexecutable before settlement; the corresponding settle would have to land `setBudget`, `fund`, and `submit` in one batch afterwards. This version does not define it.
- **Nothing strands before funding.** A created job holds no funds; the client or the provider may `reject` it while `Open`. The client's only sunk cost for a payment that never settles is the gas of `createJob`.
- **Refund after expiry is guaranteed for unhooked jobs**, in at most two client transactions. For hooked jobs it is subject to the hook's `rejectClaim` behaviour.
- **The job is the offer, verified.** Because the client creates the job, every field the server relies on — provider, evaluator, hook, description, expiry — is re-read from the chain and checked against the offer before funding. A job created with other terms is refused; it cannot be funded under this offer.
- **The server cannot reprice.** `setBudgetAuthorization` is the server's, but `fund` reverts unless the budget equals the `amount` the client signed. Payout routing via `setPayoutReceiver` is the server's own money; it does not touch what the client pays or who is accountable as `provider`.
- **Deliverable binding.** This version fixes `deliverable = keccak256(content)` over bytes both sides hold identically, so the submitted commitment is a deterministic function of content and no wire field can reinterpret it. A provider that returns one content and submits the hash of another leaves the client holding content whose hash does not match `JobSubmitted` — a disagreement x402 exposes, but does not adjudicate: the mismatch alone does not prove which side's bytes are the ones the provider returned. That attribution is application-layer evidence (TLS transcript proofs, signed receipts, logs, zero-knowledge proofs) for the evaluator; x402 does not prove it.
- **Funding and lifecycle deadlines are separate.** `maxTimeoutSeconds` bounds the client's funding authorization and the authorizations executed atomically with it. `job.expiredAt` independently bounds the ERC-8183 lifecycle and is the latest time at which the provider may submit. Provider-authored lifecycle authorizations carry their own staleness deadlines and do not inherit the client's funding deadline.
- **No response metadata to trust.** This scheme adds nothing scheme-specific to the response. The client checks the content it holds against the onchain `JobSubmitted` event — immediately under `escrow`, once the provider submits under `upfront` — so there is no server-authored field whose honesty it has to assume.
- **`submittedAt` binding.** ERC-8183 binds `submittedAt` into `Complete` and `Reject` authorizations, so a pre-signed terminal action cannot be replayed against a later submission. Not an x402 operation, but the reason relayed evaluator actions would be safe if a facilitator chose to offer them.
- **Description size.** The resolved description is stored onchain by the client's `createJob`. A long `resource.description` raises the client's gas for every job; a server SHOULD keep it to a short brief.

## Appendix

### PaymentRequirements → Job

| `PaymentRequirements` | `Job` | Set at |
| --- | --- | --- |
| `payTo` | `provider` | `createJob` |
| one of `extra.evaluators` | `evaluator` | `createJob` |
| `extra.jobExpiresAt` | `expiredAt` | `createJob` |
| `resource.description`, else `extra.description` | `description` | `createJob` |
| `extra.providerAgentId` | `providerAgentId` | `createJob` |
| one of `extra.hooks` | `hook` | `createJob` |
| — | `client = msg.sender` | `createJob` |
| — | `payoutReceiver`, server's choice, default `address(0)` (payouts to `provider`) | `setPayoutReceiver`, in the fund settle |
| `asset` | `paymentToken` | `setBudget`, in the fund settle |
| `amount` | `budget` | `setBudget`, in the fund settle |

### EIP-712 types used

Escrow domain: `{ name: "ERC8183", version: "1", chainId, verifyingContract: escrow }`.

```
SetPayoutReceiverAuthorization(address signer,uint256 jobId,address payoutReceiver,uint72 nonce,uint256 deadline)
SetBudgetAuthorization(address signer,uint256 jobId,address token,uint256 amount,bytes32 optParamsHash,uint72 nonce,uint256 deadline)
FundAuthorization(address signer,uint256 jobId,address expectedToken,uint256 expectedBudget,bytes32 optParamsHash,uint72 nonce,uint256 deadline)
SubmitAuthorization(address signer,uint256 jobId,bytes32 deliverable,bytes32 optParamsHash,uint72 nonce,uint256 deadline)
```

`optParamsHash` is `keccak256(optParams)`, `keccak256("")` when omitted. Nonces are packed onchain as `bytes32((uint256(uint160(signer)) << 96) | uint256(nonce))` and checked via `authorizationNonceUsed(bytes32)`.

### Evaluator routing (non-normative)

An evaluator MAY be a contract that itself decides which of several judges to consult — a marketplace, a committee, a reputation-weighted pool. To this binding it is one address in `extra.evaluators`; how the client and the routing contract agree on a particular judge is outside x402, and MAY travel in the job's hook `optParams` if the routing contract is also the hook. Entries that gate acceptance on an external registry (for example `"registry:0x…"` resolved through an `isAccepted(address)` view) are a plausible future value of `extra.evaluators`, not defined here.

### Delegated provider (non-normative)

A facilitator could stand in as `provider` — `payTo` set to a facilitator address, `payoutReceiver` set to the server before funding, the server verifying that onchain before serving — the way `auth-capture`'s `"delegated"` operator stands in for the receiver. This binding does not define it. The reason `auth-capture` needs delegation is that its operator must *submit transactions*: gas, nonces, an RPC. Here the server submits nothing; it signs EIP-712 messages, which is what the Signed Authorizations extension exists to make sufficient. Against no motivation stands a real cost: `provider` is the accountable party in ERC-8183 — `providerAgentId`, reputation writes, and the `provider != evaluator` rule all key on it — and a stand-in blurs who did the work. If a class of servers that cannot hold a signing key turns up, this is the shape to revisit.

### Gasless job creation (non-normative)

`createJobWithAuthorization` exists, but a `FundAuthorization` binds `jobId`, which is assigned only when creation lands. A fully gasless collect therefore needs either a second round trip (client sends `createJobAuthorization`; the facilitator lands it; the server answers 402 again with `extra.jobId`; the client sends `fundAuthorization`) or a predicted `jobId`, which fails under any contention. Neither is specified here. A creation-time price path in ERC-8183 — a `createJob` variant taking `(token, budget)` with the provider's signature over them, moving straight to `Funded` — would make a gasless one-round-trip collect possible; the 402 already is the provider's price proposal.

## Version History

| Version | Date | Changes | Authors |
| --- | --- | --- | --- |
| v0.1 | 2026-10-01 | Initial draft | — |
