# Phase 3.2 — TEE Secrecy Model and the Forced-Reveal Question

**Status:** Draft v1 — design pass, needs cryptographer review.
**Author:** Stax Network team.
**Last updated:** 2026-05-01.
**Scope:** resolve the §10.6 deferral in `docs/forcing_mechanism_design.md`
under the trust model proposed in `docs/phase3_tee_design.md` (Candidate D,
TEE-attested garbler).
**Code changes:** none in this doc — paper work only. Determines what
Phase 3.2b actually has to build.

---

## 1. Why this doc exists

`forcing_mechanism_design.md` §10.6 (2026-04-25) deferred the Phase 3
forced-reveal pipeline because it discovered that none of Candidates
#1/#2/#A/#B/#C work as long as the operator generates their own Lamport
preimages — they trivially know both wire labels at every bit position and
satisfy any forcing leaf without committing to truth.

`phase3_tee_design.md` (2026-05-01) proposed Candidate D — a TEE-attested
garbler — and framed it as "the canonical-wire-label generator that
Candidates #1/#2 already needed but didn't have". That framing is correct
but **incomplete**: it leaves implicit a follow-on question that determines
the entire Phase 3.2b implementation surface.

> **The follow-on question.** If the TEE keeps the secret half of every wire
> label away from the operator, *who else gets it*? Specifically, can a
> watchtower obtain canonical labels in order to spend a forced-reveal leaf —
> without simultaneously leaking them to the operator (who, in a permissionless
> model, can also run a watchtower)?

Answering that question turns out to **collapse the forced-reveal pipeline
entirely**. This doc walks through why. The conclusion is:

- **Forced-reveal Taproot leaves should be dropped from the Phase 3 design.**
- **The fixed-lie attack is prevented at AssertTx construction time, not at
  AssertTx dispute time.**
- **Phase 3.2b implementation is dramatically smaller than `forcing_mechanism_design.md`
  §4 / Milestones B-C suggest.**

This is a non-trivial architectural simplification, and the rest of this doc
is the reasoning that supports it. If a cryptographer reviewer disagrees,
the fallback is the layered design in §6 (TEE + forced-reveal leaves +
canonical-label distribution), which costs more on-chain but adds
defence-in-depth.

---

## 2. The §10.6 problem, restated under TEE assumptions

### 2.1 What the operator could do without a TEE

In `forcing_mechanism_design.md` §10.6:

```
Lamport preimages s_i^0, s_i^1 are derived as
    s_i^b = SHA256("...op-secret..." || withdrawal_id || i || b)
where "op-secret" is the operator's local entropy.

⇒ operator knows BOTH preimages at every bit position.
⇒ operator can construct AssertTx claiming any bit pattern (truthful or lying).
⇒ forced-reveal leaves bound to canonical labels (Candidate #1) or
   equivocation gates (Candidate #C) are satisfiable by the operator
   regardless of whether they signed truth.
```

The fix `forcing_mechanism_design.md` §10.6 calls for explicitly:

> The garbler seed must be **external to the operator** — a setup ceremony
> (multi-party DKG or trusted attested setup) generates wire labels first,
> the operator only ever learns preimages corresponding to canonical-truth
> bits, and inverting SHA256 to recover the opposite preimage is infeasible.

That is exactly what TEE-as-garbler delivers:

```
TEE generates s_i^0, s_i^1 from a master_seed it never exports.
TEE publishes only RIPEMD160(s_i^0) and RIPEMD160(s_i^1) (the public Lamport pubkey).
Operator never sees s_i^0 or s_i^1 directly.
```

### 2.2 The follow-on: what does it mean to "release" preimages?

The TEE has the preimages. The operator needs **some** preimages to construct
a well-formed AssertTx. The release mechanism is the entire game:

```
Operator submits (claimed_state s, claimed_proof π) to TEE.
TEE evaluates:  Groth16Verify(vk, public_inputs(s), π).
If valid:       TEE releases  { s_i^{ bit_i(public_inputs(s)) } }_{i ∈ [0,1184)}.
If invalid:     TEE refuses.
```

**Crucially, TEE only releases ONE preimage per bit position** — the one
matching the bit value of the claimed state's public inputs. The operator
never sees the OTHER preimage at any position. They cannot publish an
AssertTx claiming a different bit pattern because they would need a
preimage they do not possess.

### 2.3 Why this prevents the fixed-lie attack at *construction*

A "fixed-lie attacker" wants to publish AssertTx with `public_inputs = p_lie`
where `p_lie` does not correspond to any valid L2 state.

For AssertTx to be well-formed by Bitcoin Script, the witness must contain
preimages `s_i^{bit_i(p_lie)}` for all `i`. The operator needs these from
the TEE.

```
Path A — operator submits (state_lie, proof_garbage) to TEE.
         TEE runs Groth16Verify → INVALID.
         TEE refuses to release preimages.
         Operator has nothing to put in AssertTx witness.

Path B — operator submits (state_truthful, proof_valid) to TEE.
         TEE returns preimages for bit_i(public_inputs(state_truthful)).
         Operator publishes AssertTx — but it asserts the TRUTHFUL state.
         No fix-lie occurred.

Path C — operator forges Groth16 proof for state_lie.
         Soundness of Groth16 + circuit-specific knowledge soundness
         says this is computationally infeasible. (Same assumption the
         entire L2 already trusts.)
```

The fixed-lie attack reduces to **Groth16 soundness + TEE attestation
integrity**. Both are existing trust assumptions; we add no new ones.

This is structurally different from the Σ-protocol direction in
`forcing_mechanism_design.md`. There, the lie *could happen on-chain* and
forcing leaves were designed to *catch and slash it*. With TEE-as-verifier,
the lie *cannot reach Bitcoin in the first place* — there is nothing to
catch and nothing to slash via forcing leaves.

---

## 3. The architectural choice: TEE-as-Garbler vs TEE-as-Verifier

`phase3_tee_design.md` §1 frames TEE as the *garbler* — i.e., the entity
that generates the canonical wire-label table. Implicit in that framing is
that the canonical labels later flow somewhere (to watchtowers? to a
public table?) so that a Σ-forcing leaf can be spent.

This doc proposes a sharper framing: TEE is the *verifier* of the SNARK,
not just the *garbler* of the circuit. It also IS the garbler — the
distinction is what its API to the operator looks like.

### 3.1 TEE-as-Garbler API (the framing in `phase3_tee_design.md`)

```
TEE.generate_circuit(vk) -> (commitment_C, label_table T)
    // T = { (L_i^0, L_i^1) }_{i ∈ wires}
TEE.attest() -> AttestedCommitment(C, code_hash, public_seed, ...)

// Labels distributed somehow to watchtowers.
// Operator builds AssertTx using known labels.
// Forced-reveal leaves in Taproot tree commit to labels.
```

This is what Phase 3.2a infrastructure was built for: the attestation
crate, the watchtower verifier, the on-chain digest. It assumes labels
flow somewhere.

### 3.2 TEE-as-Verifier API (proposed in this doc)

```
TEE.generate_keypair_for_withdrawal(withdrawal_id)
    -> Lamport pubkey (47 KB)
TEE.attest_keypair(...) -> AttestedPubkey

// Pubkey hash committed in deposit P2TR.

TEE.release_preimages(withdrawal_id, claimed_state s, claimed_proof π)
    -> { s_i^{bit_i(public_inputs(s))} } if Groth16Verify(vk, ., π) is VALID
    -> error "INVALID" otherwise
```

The TEE is *both* garbler and verifier. The operator's only API to the TEE
is "verify this proof and give me the preimages for its public inputs".
The TEE never exports both preimages of any bit pair to the operator. The
TEE never exports the preimages it has not been asked for (i.e., never
exports preimages tied to a never-asserted state).

Watchtowers do not need preimages. They only need:
- The TEE attestation (to know the canonical pubkey is from honest code).
- The operator's published AssertTx (to verify Lamport reveals match the
  attested pubkey commitments).
- Standard equivocation detection (a second AssertTx with conflicting bits).

### 3.3 Why TEE-as-Verifier is the right framing

| Property | TEE-as-Garbler | TEE-as-Verifier |
|---|---|---|
| Forcing surface on-chain | forced-reveal leaves needed | none needed |
| Taproot tree changes | +c per-bit leaves | none |
| AssertTx witness changes | +Σ-response | none |
| Watchtower needs canonical labels | yes | no |
| Label distribution channel | required (separate problem) | not required |
| Fixed-lie defence | catch-and-slash on-chain | prevent at construction |
| Equivocation defence | existing equivocation leaves | existing equivocation leaves |
| TEE compromise impact | one fix-lie possible per breach | one fix-lie possible per breach |

The two approaches are equivalent in security against TEE compromise (both
fail open if TEE is broken). The TEE-as-Verifier framing has strictly less
moving parts: no extra Taproot leaves, no extra witness bytes, no extra
distribution channel, no Σ-protocol parameters to tune.

The cost: the TEE has to host the Groth16 verifier. That is ~tens of MB of
verifier code; well within Nitro Enclave memory budgets per
`phase3_tee_design.md` §3.1.

---

## 4. Implications for the on-chain protocol

### 4.1 Deposit time

Unchanged from current Standard build *except*:

- Lamport keypair derivation moves into the TEE.
- TEE emits an attestation document tying the published pubkey commitment
  to the attested code/circuit/withdrawal_id.
- StaxBridge contract event `GarblerAttestationRegistered(digest, version)`
  fires for each attestation (already implemented in
  `contracts/src/StaxBridge.sol` 2026-05-01).

### 4.2 Withdrawal request

Unchanged.

### 4.3 AssertTx construction

```
1. Operator computes (state s, proof π) for the L2 transition under dispute.
2. Operator opens authenticated channel to TEE, submits (s, π, withdrawal_id).
3. TEE runs Groth16Verify(vk, public_inputs(s), π).
   - INVALID → TEE responds with error; operator cannot proceed.
   - VALID   → TEE returns 1184 preimages encrypted under operator's
               pre-attested key.
4. Operator constructs AssertTx witness with the released preimages.
5. Operator broadcasts AssertTx.
```

The on-chain AssertTx is identical in shape to today's. No new fields, no
new leaves spent.

### 4.4 Disproval (DisprovalTx)

Equivocation path unchanged. If the operator publishes two AssertTxs with
conflicting bit reveals at the same Lamport position, any watchtower
constructs a 234 vB DisprovalTx via the existing equivocation leaf.

**Forced-reveal disproval path: removed.** No corresponding Taproot leaf
exists in the deposit P2TR. The fixed-lie attack is prevented upstream of
AssertTx broadcast, so there is nothing to forced-reveal.

### 4.5 Claim (CSV expiry)

Unchanged.

### 4.6 Net diff vs `forcing_mechanism_design.md` §4 plan

| Item from §4 plan | Status under TEE-as-Verifier |
|---|---|
| `derive_for_withdrawal_with_beacon` | not needed |
| `crates/bridge/bitvm3/src/forcing.rs` (full module) | keep beacon helpers as POC; production not used |
| Taproot tree gains forced-reveal leaves | NOT NEEDED |
| `withdrawal.rs::post_pre_commit` | not needed (no beacon dependency) |
| `withdrawal.rs::post_assertion` Σ-response | not needed |
| `watchtower/disputer.rs::dispute_via_forced_reveal` | NOT NEEDED |
| Beacon resolver in watchtower | not needed for Phase 3 v1 |
| `StaxBridge.sol::registerGarblerAttestation` | KEEP (already shipped 2026-05-01) |

This is a substantial reduction in implementation surface.

---

## 5. What watchtowers actually need to do

### 5.1 Steady-state operation

Identical to today, with one addition: at startup, verify the TEE
attestation file matches the on-chain digest registered by
`registerGarblerAttestation`. This is **already implemented** in
`crates/watchtower/src/main.rs` as of Phase 3.2a (2026-05-01) — flag
`--garbler-attestation` plus `--garbler-code-hash`.

The watchtower's run loop:

1. Fetch latest TEE attestation (Stax API → IPFS fallback).
2. Verify attestation: vendor signature, code-hash match, freshness,
   on-chain digest match.
3. Cache the attested Lamport pubkey commitments.
4. Watch Bitcoin for AssertTxs spending bridge UTXOs.
5. On AssertTx detection:
   - Verify Lamport reveals are well-formed (Bitcoin already does this).
   - Cross-check pubkey commitments against the attested ones.
   - Independently compute `public_inputs(canonical_state)` from L2 + DA.
   - Compare bit-by-bit against AssertTx-revealed bits.
   - **If they match**: assertion is honest, no action.
   - **If they differ**: this means TEE released preimages for a non-
     canonical state. Either the TEE is compromised or the watchtower's
     L2 view is wrong. **Alert, do not auto-disprove.**
6. On second AssertTx for same withdrawal with conflicting bits:
   construct DisprovalTx via existing equivocation leaf.

### 5.2 What the watchtower cannot do

If the TEE is compromised and releases preimages for a single lying state
(no second AssertTx), the watchtower **cannot construct an on-chain
disproval**. There is no forced-reveal leaf to spend.

The watchtower's recourse in TEE-compromise-no-equivocation scenarios:

- Off-chain alert (logs, monitoring, social).
- Refuse to validate the bridge for downstream users.
- Public attestation revocation (StaxBridge contract owner reverts
  `currentAttestationDigest` → watchtowers stop trusting future AssertTxs
  under the compromised attestation).

This is the residual trust we accept by choosing TEE over a multi-party
ceremony. It is the same residual trust documented in
`phase3_tee_design.md` §6.3: "TEE failure modes are concentrated in vendor
and silicon trust rather than distributed across N parties." TEE-as-
Verifier does not change this — it inherits it.

### 5.3 Why no forced-reveal leaf would help anyway

Even if we kept the forced-reveal leaf design from
`forcing_mechanism_design.md` §3.1, in the TEE-compromise-no-equivocation
scenario the watchtower would still not be able to spend it. The leaf
script is

```
OP_SHA256 <expected_hash[i]> OP_EQUALVERIFY OP_TRUE
```

To spend, the watchtower needs the preimage of `expected_hash[i]`. In the
TEE model, the preimages are TEE-internal. Watchtowers never had them. The
forced-reveal leaf is unspendable by any party except the TEE and whoever
the TEE released preimages to (the operator).

So forced-reveal leaves do not provide additional defence-in-depth in the
TEE-compromise scenario. They are vestigial.

The §10.6 reasoning that flagged this is exactly right: "the operator
chose `expected_hash[i]` and the watchtower has no independent way to
derive the preimage". In the TEE model, the TEE chose `expected_hash[i]`
and the watchtower still has no way to derive the preimage. Same outcome,
different actor. Either way, the leaf does not slash bond.

---

## 6. Defence-in-depth fallback (if cryptographer review pushes back)

If a reviewer believes "fixed-lie at construction" is not a sufficient
defence and we should additionally have on-chain disprovability, the
fallback design is to KEEP the forced-reveal leaves AND distribute the
canonical labels to watchtowers via an attested side channel.

### 6.1 Construction

- TEE generates per-position labels as before.
- TEE encrypts the FULL label table `{ (L_i^0, L_i^1) }` under a
  watchtower-chosen key (e.g. published threshold-decryption pubkey or
  per-watchtower attested keys).
- Encrypted blob published via DA (Celestia) or Stax API.
- Watchtower decrypts, learns ALL labels.
- Forced-reveal leaves with `expected_hash = SHA256(L_i^{canonical_bit_i})`
  added to deposit P2TR.

### 6.2 Cost vs §3.2 baseline

| Item | Cost |
|---|---|
| Off-chain pubkey size | 47 KB → ~85 KB (per §10.2 Candidate A) |
| Taproot tree depth | +log2(c) leaves (c=80 → +7 levels, depth 11→12) |
| AssertTx witness | +Σ-response (~3-5 KB) |
| Implementation work | Phase 3.2 §4 plan from `forcing_mechanism_design.md` in full |
| New cryptographic component | watchtower-key registry + attested encryption |
| Liveness | unchanged (no new on-chain rounds) |

### 6.3 What this buys

- TEE-compromise scenarios that DON'T involve operator equivocation can
  still be slashed on-chain by an honest watchtower with the canonical
  labels.
- "Fixed-lie attacker who also happens to control the TEE" is detectable
  by any watchtower with the encrypted label channel.

### 6.4 Why this doc recommends NOT doing this

- All extra mechanisms add attack surface for *anyone* who acquires
  watchtower keys (which is meant to be permissionless — anyone can run a
  watchtower, so anyone can have the labels, so we are back to operator
  having both labels via a sock-puppet watchtower).
- The threat model collapses to "honest watchtower exists somewhere whose
  decryption key is not leaked" — which is already implied by the
  optimistic 1/N challenger assumption. The on-chain leaf adds nothing
  the off-chain alert + revocation flow does not.
- Implementation cost is large (Σ-response, forced-reveal leaves, label
  distribution, key management) for a defence-in-depth that does not
  cleanly compose.

The fallback exists in case external review insists we want belt-and-
braces. Default direction stays §3.2.

---

## 7. Open questions for cryptographer review

1. **Groth16 soundness as our only fixed-lie defence.** Is reducing
   "fixed-lie defence under TEE" to "Groth16 soundness + TEE-as-honest-
   verifier" acceptable for a mainnet bridge? Existing comparable systems
   (zkSync, Polygon Hermez) trust Groth16 soundness for their entire
   security model — we are no worse positioned, but want this confirmed.

2. **TEE statefulness for equivocation prevention.** Should the TEE
   internally track "I have already released preimages for withdrawal_id X
   under bit pattern P" and refuse to release a second pattern for the
   same withdrawal? If yes, this is *prevention* of equivocation rather
   than detection. Cost: TEE must persist state durably (Nitro KMS
   integration, AMD SEV-SNP requires equivalent). Benefit: removes the
   equivocation attack surface entirely.

3. **TEE-to-operator channel encryption.** When TEE returns preimages,
   the channel to the operator must be encrypted (otherwise eavesdroppers
   gain the ability to publish AssertTx). Standard answer: TEE-attested
   TLS with operator's pre-published key. Confirm this is sufficient.

4. **TEE replication for liveness.** A single TEE is a SPOF for bridge
   liveness. Multiple TEE instances (with the same code_hash, same
   master_seed) need a coordination protocol so they don't release
   preimages for conflicting states. Question: is "deterministic master
   seed across N TEEs + per-instance state log + take-the-first-replied"
   safe, or does it open a race that an attacker can exploit?

5. **TEE upgrade with active deposits.** If we ship a new garbler binary
   (different code_hash), existing deposits committed to the old pubkey
   are stuck unless either (a) the old TEE binary remains operationally
   available indefinitely, or (b) we ship a migration path (e.g. cooperative
   refund, then re-deposit). Open question: which?

6. **Operator's TEE attestation visibility to watchtowers.** Watchtowers
   verify the attestation digest registered on-chain. But the digest
   covers (code_hash, public_seed, output_hash). If `output_hash` is the
   pubkey commitment for ONE withdrawal, every withdrawal needs its own
   on-chain digest? Or do we group? Open: per-withdrawal, per-batch, per-
   epoch?

7. **Fallback: would forced-reveal leaves ever be useful?** §6.3 argues
   no, because watchtowers possessing labels means operator-via-sock-
   puppet possesses labels. Cryptographer to confirm or reject.

---

## 8. Decision log (tentative — to revisit after review)

| Date | Decision | Rationale |
|---|---|---|
| 2026-05-01 | TEE-as-Verifier framing over TEE-as-Garbler-only | Strictly fewer moving parts; eliminates label distribution problem |
| 2026-05-01 | Drop forced-reveal Taproot leaves from Phase 3 design | Vestigial under TEE-as-Verifier; do not provide defence-in-depth in TEE-compromise scenarios either |
| 2026-05-01 | Keep equivocation leaves + on-chain attestation digest | Already shipped or in code; no reason to remove |
| 2026-05-01 | Defer §6 fallback unless cryptographer pushes back | Cost/benefit does not favour belt-and-braces given threat model |
| 2026-05-01 | Mark §7 open questions as pre-commit-to-3.2b blockers | We can pause and clarify before writing more code |

---

## 9. What this means for the Phase 3.2 task list

`forcing_mechanism_design.md` Milestone B (3-4 days) and Milestone C
(2-3 days) describe the forced-reveal pipeline. Under this doc's
recommendation, those milestones reduce to:

- **Milestone B (Phase 3.2b core)**: 2-3 days
  - Wire `bitvm3-garbler` as TEE-resident binary (mock or real).
  - Operator-to-TEE request protocol (gRPC or stdin/stdout JSON).
  - `withdrawal.rs` learns to call out to TEE for preimages instead of
    deriving them locally.
  - Per-withdrawal TEE attestation registration on Stax bridge contract.

- **Milestone C (Phase 3.2b adversarial)**: 1 day
  - Adversarial test: operator submits fake proof to TEE → assert TEE
    refuses; assert AssertTx cannot be constructed without preimages.
  - No new evil-ops mode for forced-reveal disproval (no such path exists).

- **Milestone D (cryptographer review)**: unchanged scope, smaller surface
  to review (no Σ-protocol parameters, no leaf-script analysis).

Total: ~1 week of code instead of 2-3 weeks. Phase 3.2b becomes shippable
inside the original Phase 3.2 budget.

---

## 10. Relationship to other docs

- `docs/forcing_mechanism_design.md` §10.6 — this doc proposes
  *bypassing* §10.6's open question rather than *answering* it. The
  bypass is "switch the trust model so the question doesn't bind."
  Section 10 of that doc remains historically accurate but its
  conclusions ("forced-reveal pipeline is deferred") get a new footnote:
  *"...and replaced by TEE-as-Verifier construction, which does not
  require forced-reveal leaves at all."*
- `docs/phase3_tee_design.md` — sibling doc. This one resolves an
  ambiguity that document leaves implicit (TEE-as-Garbler vs TEE-as-
  Verifier). Both docs land on Candidate D as the trust model; this
  doc sharpens the API and removes the on-chain footprint.
- Phase 3.2a infrastructure (already shipped 2026-05-01) — fully
  compatible with TEE-as-Verifier. The mock attestation, watchtower
  verifier flag, and `registerGarblerAttestation` contract event are
  exactly what TEE-as-Verifier needs and nothing more.

---

## 11. Action items before writing more Phase 3.2b code

1. Get cryptographer eyes on §3 and §5.3 specifically — those are the
   load-bearing arguments for dropping forced-reveal leaves.
2. Decide §7.2 (TEE statefulness for equivocation): if YES, equivocation
   leaves can also be deprecated and we collapse Phase 3 to "TEE attests,
   nothing else"; if NO, equivocation leaves stay as the only on-chain
   disproval path.
3. Decide §7.6 (per-withdrawal vs batch attestation): determines
   `registerGarblerAttestation` call frequency and on-chain cost
   profile.
4. Resolve §7.5 (TEE upgrade migration): blocks any production
   deployment that has user funds in flight.

Until items 1-2 resolve, no further Phase 3.2b implementation should
land. The attestation infrastructure already in tree is correct under
either decision.
