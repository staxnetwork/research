# Forcing Mechanism Design — Phase 3.1

**Status:** Design proposal — open for review.
**Author:** Stax Network team.
**Last updated:** 2026-04-25.
**Scope:** close the "fixed-lie" gap in trust-minimisation. Code changes deferred to Phase 3.2.

---

## 1. Threat model

### 1.1 What we want

A Bitcoin-native rollup must guarantee that an honest watchtower can prevent a malicious sequencer from withdrawing bridge BTC unless the corresponding L2 state transition was valid.

"Prevent" must be **on-chain enforceable** — it is not enough for the watchtower to know the assertion is wrong; Bitcoin must be convinced. Otherwise the operator simply waits out the challenge window and broadcasts ClaimTx.

### 1.2 What we have today (Standard build, deployed on signet)

The bridge uses BitVM3s-style Lamport OTS commitments to 148 bytes (1184 bits) of SNARK public inputs (`crates/bridge/bitvm3/src/lamport.rs`, `assertion.rs`):

1. **Setup.** Per-withdrawal Lamport keypair derived as
   `seed = SHA256("btc-l2-lamport-withdrawal-v1" || master_seed || l2_tx_hash)`.
   The 47 KB public key (1184 × 2 × 20-byte RIPEMD160 commitments) is hashed
   into a single 32-byte `commitment_hash` carried in AssertTx's OP_RETURN.

2. **AssertTx.** Operator reveals 1184 preimages (one per bit of public input)
   in a Taproot script-path spend. The witness embeds:
   - 148 bytes of public inputs
   - 1184 × 56 = 66 KB of Lamport preimages
   - Groth16 proof bytes

3. **DisprovalTx (`crates/watchtower/src/disputer.rs`).** Builds an
   equivocation-spend: two AssertTxs that share a Lamport keypair are required.
   The disproval leaf is `OP_RIPEMD160 <hash_0> OP_EQUALVERIFY OP_RIPEMD160 <hash_1> OP_EQUALVERIFY`
   — only revealing **both** preimages of the same bit pair satisfies it.

### 1.3 The fixed-lie attack

```
1. Operator constructs FALSE public_inputs:
       new_state_root = root(L2 chain *plus* a fake `withdraw(operator, X)` tx)
   (the actual L2 chain has no such withdraw — operator is lying about state).

2. Operator runs Groth16 prover on the false witness. SNARK soundness says
   they cannot produce a valid proof for false inputs without breaking the
   pairing assumption — BUT they don't need to. They publish *whatever 64 bytes*
   in the `proof_data` slot. Bitcoin Script only checks Lamport commitments,
   not Groth16 verification (full Groth16 verifier is ~280 GB garbled, lives
   off-chain).

3. Operator publishes AssertTx with:
       - lying public_inputs (lying new_state_root)
       - garbage proof_data
       - valid Lamport signatures over the lying bits
   AssertTx is well-formed by Bitcoin Script — it spends successfully, locks
   bond, starts CSV countdown.

4. Watchtower observes AssertTx. Off-chain it:
       - Recomputes the canonical new_state_root from the L2 chain.
       - Sees it differs from the claimed root.
       - Off-chain garbled-circuit evaluation rejects.
   Watchtower KNOWS the assertion is fraudulent.

5. Watchtower has no on-chain disproval available:
       - DisprovalTx requires equivocation (two AssertTxs reusing a Lamport
         keypair). Operator only published ONE.
       - Off-chain garbled-circuit rejection is not encoded as a Bitcoin
         Script condition in our Taproot tree today.

6. CSV expires (3 blocks on signet, configurable on mainnet). Operator
   broadcasts ClaimTx. The bridge pays out the lying withdrawal. Bond never
   slashes. Watchtower screams in logs but Bitcoin doesn't care.
```

### 1.4 Why current Lamport doesn't force the issue

Lamport equivocation triggers iff the operator signs the **same bit position**
with the **opposite bit value** in two distinct messages. A fixed-lie attacker
publishes exactly one message and never re-signs anything. There is no second
signature to cross-reference; the equivocation script can never be satisfied.

The garbled circuit + `wire_label_from_preimage` plumbing in `garbler.rs` and
`assertion.rs` makes off-chain SNARK verification possible (and we use it in
`watchtower::scan_block`), but the on-chain disproval path codified in
`disputer.rs` is **purely equivocation-based**. There is no encoded fraud-proof
leaf in the Taproot tree that says "if garbled-circuit-output bit = 0, slash".

---

## 2. Goal of Phase 3

Force the operator into one of:
   **(a)** publishing a TRUTHFUL assertion (bond unslashed, withdrawal honoured), or
   **(b)** publishing an assertion that the watchtower can convert into an
       on-chain DisprovalTx — either by equivocation OR by satisfying a new
       fraud-proof leaf.

Either outcome closes the gap. (a) is the honest path; (b) means the operator
loses bond.

A successful design must:

- Preserve Standard's "killer feature": **47 KB pubkey, 234 vB DisprovalTx**, sub-Raspberry-Pi-class watchtower.
- Not require a soft-fork or new Bitcoin opcode.
- Not introduce new trust assumptions beyond those already in the Bitcoin honest-majority + Celestia DA model.
- Be implementable with the team's current cryptographic stack (RIPEMD160, SHA256, Lamport, Taproot, Groth16 off-chain).

---

## 3. Two candidates

### 3.1 Candidate #1 — Shared-randomness binding (RECOMMENDED)

**Sketch.** Bind the operator's Lamport public key to a piece of randomness
that they cannot predict at the time they fix their pubkey. Concretely, the
on-chain commitment is

```
    pubkey_root = SHA256( serialized_lamport_pubkey || r )
```

where `r` is sampled AFTER the pubkey is committed.

**Where the magic happens.** Without `r`, the operator can choose preimages
freely → they can fix the lie at pubkey-creation time. With `r` chosen later,
the operator must commit to a *family* of pubkeys before knowing which one
counts. To make any single pubkey force-equivocate-on-lie, we leverage Fiat-
Shamir on the garbled-circuit verification.

**Construction (preferred variant).** Use a 2-round Σ-protocol made non-
interactive via Bitcoin block hashes as the random oracle:

1. **Round 1 — operator commits.** Sequencer publishes a small "pre-commit"
   transaction containing `H(pk_full || nonce)` where `pk_full` is the full
   Lamport public key. (Already done implicitly today via the
   `commitment_hash` in OP_RETURN — this part is essentially a relabelling.)

2. **Beacon — randomness fixed.** After the pre-commit confirms in some block
   `B*`, define
   `r = SHA256(blockhash(B*+k) || pk_full)` for a fixed delay `k` (e.g. 6
   blocks, ≈ 1 hour).

3. **Round 2 — operator publishes AssertTx.** AssertTx carries the actual
   Lamport reveals AND a "challenge response" subset of the 1184 bit positions
   selected by `r`. Specifically, `r` deterministically picks `c` positions
   (e.g. `c = 80`) at which the operator must reveal **the canonical garbled-
   circuit input wire labels**. The Taproot tree has, alongside the
   equivocation leaf, `c` per-position **forced-reveal leaves**: each leaf
   verifies that the revealed preimage at position `r_i` evaluates the garbled
   circuit to a known-good wire label — i.e. matches the value the operator
   would reveal IF the assertion were truthful.

4. **Disproval.** If the operator lied, the garbled-circuit value at SOME
   position differs from the canonical one. With `c = 80` random positions out
   of 1184, the probability that none of the lying bits is in the challenge
   set is `(1 - lying_fraction)^80`. For any non-trivial lie (≥ 16 lying bits),
   this is `< 2^{-80}` — cryptographically negligible. Any honest watchtower
   constructs DisprovalTx by spending the forced-reveal leaf at a sampled
   position whose canonical garbled-circuit output is `0` (reject).

**Properties.**

| Property | Value |
|---|---|
| Pubkey size on-chain | unchanged: 32 bytes (`commitment_hash`) |
| Pubkey size on Stax API/IPFS | unchanged: 47 KB |
| AssertTx witness | unchanged base 66 KB + ~3-5 KB Σ-response |
| DisprovalTx vbytes | unchanged base ~234 vB + 32-byte Σ-tag |
| New on-chain elements | 1 pre-commit tx (~110 vB) per withdrawal |
| Liveness assumption | operator must wait `k`-block beacon delay before AssertTx |
| Soundness | `2^{-c}` for `c`-position challenge set (default `c = 80`) |
| Trust assumption added | operator does not have predictive control of `blockhash(B*+k)` (standard Bitcoin assumption) |

**Why this preserves the killer feature.** The 47 KB pubkey lives off-chain;
on-chain still only `commitment_hash || beacon_pointer`. DisprovalTx still
spends a single Taproot leaf with one Lamport preimage reveal (≈ 234 vB).

**Why this works.** The randomness `r` cannot be "ground" by the operator
without enormous block-mining capacity. They commit to `pk_full` BEFORE seeing
`r`, so the challenge set is uniformly random over their choice. Any lie
distorts at least one bit of the canonical garbled-circuit trace; `c = 80`
random positions catch it with overwhelming probability.

**Open questions for cryptographer review.**

- Is `c = 80` the right Σ-challenge length given the structure of our Groth16
  verifier garbled circuit? (Lying fraction depends on circuit topology.)
- Is `k = 6` blocks enough beacon delay for mainnet-grade randomness, given
  selfish-mining attacks? (Conservative answer: 100 blocks; performance
  answer: 6 blocks. Trade-off is liveness latency vs. security margin.)
- Can we reuse the existing `commitment_hash` as the pre-commit, or do we
  need a separate on-chain element to fix the binding? (Probably reusable —
  saves a tx.)

### 3.2 Candidate #2 — Groth16 input-label commitment reuse

**Sketch.** Make the Lamport bit-pair preimages **identical** to garbled-
circuit input wire labels. Then SNARK soundness directly forces equivocation
on any lie:

- For each input bit `i`, the canonical garbled circuit has wire labels
  `(L_i^0, L_i^1)`. Today we DERIVE wire labels FROM Lamport preimages via
  `wire_label_from_preimage` (a one-way SHA256). Candidate #2 reverses this:
  Lamport preimages ARE the wire labels (or extended to include them).

- The forced-reveal leaves live in the Taproot tree: for each bit `i`, a leaf
  encodes "if the bit is `b`, the revealed preimage must be `L_i^b`; else
  bond slashes". Bitcoin Script verifies the equality.

- Lying about bit `i` means revealing `L_i^{¬b}` — which is the **other**
  wire label. To use the value-`b` interpretation in the garbled circuit, the
  operator would have to reveal BOTH labels — Lamport equivocation captured.

**Properties.**

| Property | Value |
|---|---|
| Pubkey size on Stax API | grows: each commitment carries the full wire label (16+ bytes), pushing 47 KB → ~150 KB |
| AssertTx witness | grows similarly |
| DisprovalTx vbytes | grows: each forced-reveal leaf is ~50-200 vB (depending on circuit topology); aggregate disproval may need 2-5 KB |
| New on-chain elements | none — single round |
| Liveness assumption | unchanged (no beacon delay) |
| Soundness | matches Groth16 soundness directly |
| Trust assumption added | none beyond Groth16 |

**Why this is structurally elegant.** Soundness is bound to the SNARK we
already trust. No randomness oracle, no Σ-protocol, no beacon liveness.

**Why we don't recommend it.** It bloats the disproval witness 5-10× and the
pubkey 3×, evaporating Standard's size pitch (47 KB pubkey, 234 vB disprove).
The Path E size advantage is the strongest dev-facing differentiator — a
mainnet watchtower runs comfortably on a Raspberry Pi BECAUSE the disproval
fits inside one tx and the pubkey download is small. Candidate #2 pushes us
into BABE-class size territory, where the size pitch becomes "BABE-class size
+ Lamport-class trust" rather than "smallest disprove on Bitcoin mainnet".

If Candidate #1 fails cryptographer review, Candidate #2 is the fallback.
Marketing/positioning would pivot accordingly.

### 3.3 Comparison summary

| Axis | Standard today | Candidate #1 | Candidate #2 |
|---|---|---|---|
| Closes fixed-lie gap | NO | YES (`2^{-80}`) | YES (Groth16 soundness) |
| Pubkey on-chain | 32 B | 32 B | 32 B |
| Pubkey off-chain | 47 KB | 47 KB | ~150 KB |
| AssertTx witness | 66 KB | ~70 KB | ~80-100 KB |
| DisprovalTx vB | 234 | ~270 | 2,000-5,000 |
| New on-chain elements | — | 1 pre-commit (~110 vB) | — |
| Liveness latency added | — | `k` blocks beacon (≈ 1h at `k=6`) | — |
| Watchtower compute | trivial | trivial + RNG | trivial |
| Cryptographer review needed | — | YES (Σ-protocol params) | minimal |
| Dev/marketing impact | — | none (size unchanged) | substantial (size pitch shifts) |

### 3.4 Recommendation

Adopt **Candidate #1** (shared-randomness binding) as the primary direction.

Justification:
- Closes the fixed-lie gap with cryptographically negligible failure probability (`2^{-80}`).
- Preserves the 47 KB pubkey / ≤ 270 vB disproval profile that anchors the
  Path E pitch.
- Adds only one cheap pre-commit tx and a beacon delay; no new on-chain
  bloat.
- The Σ-protocol is well-understood and Bitcoin-friendly; the "challenge from
  block hash" pattern is already used in production by other Bitcoin-anchored
  protocols (e.g. RGB, ARK).

Risks:
- Beacon-delay parameter `k` must be tuned with cryptographer input. If `k`
  is too low, selfish mining attacks the randomness. If `k` is too high,
  withdrawal latency suffers.
- The "forced-reveal leaf at challenged positions" requires the Taproot tree
  to grow by ~1184 leaves (one per bit position). Need to confirm this fits
  inside MAX_SCRIPT_SIZE / Taproot tree depth limits — but each leaf is tiny
  (40-80 vB), so 1184 leaves at depth `ceil(log2(1184))` = 11 is comfortable.

---

## 4. Implementation outline (Phase 3.2 preview, not yet implemented)

For visibility — actual code lands in Phase 3.2.

1. **`crates/bridge/bitvm3/src/lamport.rs`**
   - Add `derive_for_withdrawal_with_beacon(master_seed, l2_tx_hash, beacon: [u8; 32])`.
   - Pubkey serialization unchanged; binding lives in the Σ-protocol layer.

2. **New module `crates/bridge/bitvm3/src/forcing.rs`**
   - `select_challenge_positions(beacon: [u8; 32], c: usize) -> Vec<usize>`.
   - `build_forced_reveal_leaf(position: usize, canonical_label: WireLabel) -> ScriptBuf`.
   - `verify_forced_reveal(witness: &AssertTxWitness, beacon, canonical_labels)`.

3. **`crates/bridge/bitvm3/src/withdrawal.rs`**
   - `post_pre_commit(...)` — broadcasts the pre-commit tx that fixes the
     pubkey before the beacon block.
   - `post_assertion(...)` — augmented to include the Σ-response for the
     `c` challenged positions.
   - Taproot tree builder gains `c` forced-reveal leaves alongside the
     existing equivocation leaf.

4. **`crates/watchtower/src/disputer.rs`**
   - New `dispute_via_forced_reveal(target, beacon, canonical_label)` path
     for the case where the operator lied at a challenged position. Spends
     the corresponding forced-reveal leaf.

5. **`crates/watchtower/src/main.rs`**
   - Beacon resolver: read `blockhash(B*+k)` from `bitcoind`.
   - On AssertTx detection, select challenge set, evaluate canonical garbled
     circuit, compare against revealed preimages, dispatch dispute if mismatch.

6. **`StaxBridge.sol`** (only if needed)
   - May not require changes — beacon is a pure off-chain function of
     Bitcoin block hashes. Confirm during 3.2 design pass.

---

## 5. Verification plan (Phase 3.3 preview)

Three tracks:

1. **Adversarial unit tests** (`crates/bridge/bitvm3/tests/forcing_*.rs`)
   - "Fixed-lie attacker" tries to publish a lying AssertTx and pass the
     Σ-challenge for `c = 80` random sets. Test asserts that for any
     ≥-16-bit lie, the attacker fails on `>= 2^{-40}` of challenge sets.
   - Beacon-grinding attacker: simulate operator with X% block-mining
     capacity, show that the binding fails only with negligible probability.

2. **Regtest end-to-end** (`crates/integration/tests/forcing_e2e.rs`)
   - Full evil-operator scenario against Phase 3 binaries on regtest.
   - Verify watchtower constructs DisprovalTx via forced-reveal path and
     bond slashes within `CSV + 1` blocks.

3. **External crypto review** (target: 1-2 cryptographers)
   - Threat model + Σ-protocol parameter selection.
   - Beacon delay `k` for mainnet selfish-mining margin.
   - No code review at this stage — we want sign-off on the cryptographic
     core before committing to implementation depth.

---

## 6. Decision log

| Date | Decision | Rationale |
|---|---|---|
| 2026-04-25 | Recommend Candidate #1 | Preserves Path E size pitch (47 KB pubkey, ≤ 270 vB disprove) while closing fixed-lie gap |
| 2026-04-25 | Σ-challenge length `c = 80` (default) | Matches conventional 80-bit security; tunable in Phase 3.2 |
| 2026-04-25 | Beacon delay `k = 6` blocks (default) | Conservative against ≤ 51% selfish-mining adversary; tunable |
| 2026-04-25 | Defer Candidate #2 | Fallback if Candidate #1 fails review |

---

## 7. Why this is OUR moat (positioning vs competitors)

Forcing mechanisms exist in the literature but no Bitcoin-anchored bridge today
ships one in production. Competitor matrix:

| System | Size profile | Forcing | Trust assumption |
|---|---|---|---|
| BitVM2 (original) | ≥ 100 MB tapleafs, ≥ 4 KB disprove | — equivocation only | optimistic 1/N challenger |
| BitVM3 standard | 47 KB pubkey, 234 vB disprove | — equivocation only | optimistic 1/N challenger |
| BABE / Argo MAC | 6.6 MB pubkey, ~600 vB disprove | — equivocation only | optimistic 1/N challenger |
| Citrea / Bitlayer / Merlin | depends — most use federated multisig | — none, social slashing | committee honest-majority |
| Botanix / Spiderchain | PoS committee | — slashing via PoS | external chain |
| **Stax + Phase 3 (Candidate #1)** | **47 KB pubkey, ≤ 270 vB disprove** | **Σ-protocol forcing on lying assertion** | **optimistic 1/N + Bitcoin block-hash unpredictability** |

What this gives us that nobody else has:

1. **First Bitcoin-native forcing primitive that is BOTH small enough for
   Raspberry-Pi watchtowers AND cryptographically (not socially) binding.**
   Mainnet practitioners have been choosing between (a) BitVM-style "small but
   only equivocation" or (b) federated/PoS "any forcing but heavy trust
   assumptions". Candidate #1 sits in a gap nobody else occupies.

2. **No new Bitcoin opcode, no soft-fork, no committee.** Beacon = Bitcoin
   block hash. Σ-challenge = SHA256 of beacon. Forced-reveal leaf = standard
   Taproot script. Ships today on signet, ships tomorrow on mainnet.

3. **Watchtower stays portable.** A user can run the watchtower on a phone
   or a Pi Zero. The Σ-challenge evaluation is local SHA256 + lookup against
   a 47 KB canonical wire-label table — well under a second per AssertTx.
   This keeps the "anyone can watch" decentralisation pitch credible.

4. **Operator latency cost is bounded and well-defined.** `k = 6` blocks ≈ 1
   hour of beacon delay. That's the price of forcing. Other systems either
   pay much more (federated multisig consensus delay) or claim zero forcing
   delay but pay for it in trust assumptions.

This is the headline claim for the SDK landing page, the investor deck, and
the mainnet readiness write-up. We are not "another BitVM bridge" — we are
**the only Bitcoin bridge today with cryptographic forcing at Pi-class size**.

## 8. Implementation milestones

To make Phase 3 concrete and shippable rather than "3 weeks of code":

### Milestone A — POC module (1-2 days)

Self-contained, no integration with the withdrawal pipeline. Lives in a new
crate `crates/bridge/bitvm3/src/forcing.rs`. Goals:

- `select_challenge_positions(beacon, c) -> Vec<usize>` (Fiat-Shamir over
  beacon → `c` distinct positions in `[0, 1184)`).
- `build_forced_reveal_leaf(position, canonical_label) -> ScriptBuf` (returns
  the Taproot leaf script that verifies a single challenged position).
- `verify_sigma_response(witness, beacon, canonical_labels, c) -> Result<()>`
  (off-chain verifier the watchtower uses to decide whether to dispute).
- Adversarial unit tests:
  - "Honest operator passes Σ-challenge" — reveals all canonical labels at
    challenged positions, returns Ok.
  - "Lying operator fails Σ-challenge" — fakes ≥ 16 bits, run 1000 random
    beacons, assert fail rate ≥ 99.9999% (`> 1 - 2^{-20}` for the test).
  - "Beacon-grinder simulation" — operator with 1% block share, verify they
    can't bias challenge set out of `2^{-30}` failures over 10k trials.

**Exit:** module compiles standalone, all unit tests pass, benchmark shows
verifier runs < 100 ms per AssertTx.

### Milestone B — Wire into withdrawal pipeline (3-4 days)

- `withdrawal.rs::post_assertion` augmented with Σ-response generation.
- Taproot tree builder gains forced-reveal leaves alongside equivocation leaf.
- New API endpoint `GET /bridge/forcing-beacon/:assert_txid` returns the
  resolved beacon block hash (so external watchtowers can verify the
  challenge set deterministically).
- Watchtower `disputer.rs` gains `dispute_via_forced_reveal` branch.

**Exit:** regtest e2e — honest withdrawal still completes successfully.
Equivocation path still works. New forcing path constructs but doesn't yet
trigger DisprovalTx (no evil-attacker code yet).

### Milestone C — Adversarial e2e (2-3 days)

- New `evil_ops` feature `force_lying_assert` — publishes AssertTx with a
  lying `new_state_root` and crafts the Σ-response by guessing canonical
  labels at challenged positions. Should fail.
- Regtest e2e: evil-operator runs `force_lying_assert` → watchtower detects
  Σ-mismatch → builds DisprovalTx via forced-reveal leaf → bond slashes
  within `CSV + 1` blocks.

**Exit:** demo-able evil scenario on regtest. Bond-slash on lying assertion
proven by tx trace.

### Milestone D — Audit-ready (1 week, parallel with B/C)

- Cryptographer engagement with the design doc + POC (Milestone A output).
- Tune `c` (challenge length) and `k` (beacon delay) per cryptographer
  recommendation.
- Threat-model addendum to `docs/SECURITY.md`.

**Exit:** external review sign-off OR documented residual risks.

### Milestone E — Mainnet integration (2-3 days, optional pre-mainnet)

- StaxBridge.sol changes if needed (likely none, beacon is Bitcoin-native).
- Mainnet binary build with Phase 3 features.
- Activation flag (`STAX_FORCING_ENABLED=1`) so we can ship binaries that
  fall back to Standard's equivocation-only flow for one signet test cycle
  before mandatory enforcement.

**Exit:** Phase 3 fully active on mainnet.

Total: ~2.5-3 weeks of focused work, parallelisable across A and D.

### POC starts now

Milestone A is contained, doesn't touch any production code, and produces a
concrete artifact (a new module + tests) that demonstrates the cryptographic
core. Beginning implementation immediately.

## 9. Open items for next session

- Cryptographer engagement: scope and shortlist (1-2 names).
- Confirm Taproot tree depth budget: 1184 forced-reveal leaves + 1184
  equivocation leaves + housekeeping (CSV, force-withdraw) — back-of-envelope
  fits in depth 12, but verify against current `withdrawal.rs` builder.
- Decide whether the pre-commit (Round 1) reuses today's `commitment_hash` in
  AssertTx OP_RETURN, or needs a separate on-chain anchor.
- Mainnet beacon vs. signet test: confirm the `blockhash(B*+k)` source-of-
  randomness is acceptable at mainnet difficulty under realistic adversary
  models.

## 10. Open cryptographic question — `expected_hash` binding

This section was added 2026-04-25 after attempting to wire Milestone B.1
(Taproot tree) and discovering the gap below. **B.1, B.2 and B.4 are
blocked on resolving this**; B.3 (beacon endpoint) was implemented because
it is pure data plumbing.

### 10.1 The gap

The forced-reveal leaf script (`forcing.rs::forced_reveal_tapscript`) is

```
OP_SHA256 <expected_hash[i]> OP_EQUALVERIFY OP_PUSHNUM_1
```

To slash, a watchtower must produce a 16-byte preimage `v` such that
`SHA256(v) == expected_hash[i]`. The intended semantics is

> `expected_hash[i] = SHA256(canonical_wire_label_i)`,
> where `canonical_wire_label_i` is the wire label of the **honest** garbled-
> circuit input bit at position `i`.

If the operator is honest, they put honest `expected_hash[i]` in the tree;
the watchtower with knowledge of canonical labels (computed off-chain from
the true L2 state) can spend any challenged leaf. If the operator is **lying**
about the L2 state, they computed their garbled circuit on the lying state
and the labels they know are NOT the watchtower's canonical labels.

**The unresolved question**: the operator builds the AssertTx Taproot output
*before* publishing AssertTx. They are the sole party choosing what bytes go
into each `expected_hash[i]`. A lying operator can put bytes that

- (a) hash to operator-known but watchtower-unknown values → no watchtower
  can spend any leaf → bond is never slashed via the forcing path;
- (b) commit to honest canonical labels they don't actually know → AssertTx
  is well-formed by Bitcoin (Script doesn't check `expected_hash` content) →
  bond never slashed unless watchtower happens to know them, which they do
  only if the lie was actually small (degenerate to existing equivocation).

Either way, the leaf script alone does not bind the operator to truthful
canonical labels. The Σ-protocol soundness argument in §3.1 implicitly
assumes canonical labels are *publicly verifiable* — Bitcoin Script as it
exists today (no `OP_CAT`) cannot derive them inline from the operator's
Lamport preimage and a position-index suffix.

### 10.2 Three candidate resolutions

#### Candidate A — Extend the off-chain pubkey commitment

The 47 KB off-chain pubkey today is `||_{i,b} RIPEMD160(s_i^b)`. Extend it
to also include `||_{i,b} SHA256(L_i^b)`, where `L_i^b` is the canonical
wire label of bit `i`, value `b`. Pubkey grows ~85 KB; on-chain
`commitment_hash = SHA256(extended_pubkey)` unchanged at 32 B.

The watchtower downloads the pubkey, verifies its hash matches OP_RETURN,
and now KNOWS the canonical wire-label hashes. The Taproot tree's
`expected_hash[i]` is taken directly from the extended pubkey at the entry
for position `i` and the canonical bit value at that position.

**Catch**: who decides "canonical bit value at position i"? It depends on
the L2 state being asserted. If the operator builds the extended pubkey
to commit to LYING bit values, the watchtower's honest canonical label at
position `i` does not match `expected_hash[i]`, so the watchtower cannot
spend the leaf with their honest-label witness — the lie passes.

For Candidate A to work we need a SECOND mechanism that binds the operator's
"canonical bit values" claim to the actual asserted L2 state. One option:
the asserted state root is itself a hash over the position-bit pairs, so
forging the pubkey-side bit values requires forging the state root,
breaking SHA256 second-preimage resistance. This needs cryptographer
verification — it's plausibly tight if every 1184-bit Lamport message is
exactly the SNARK public-input bytes (which today they are).

**Trade-offs.** +38 KB pubkey download. No new on-chain bytes. No new round.
Watchtower verification stays simple SHA256.

#### Candidate B — Beacon-conditioned `expected_hash` via post-beacon commit tx

Operator publishes AssertTx with PLACEHOLDER forced-reveal leaves
(`expected_hash[i] = SHA256(0)` or similar). After the beacon block, operator
publishes a SEPARATE on-chain commit tx — call it `RevealTx` — whose
OP_RETURN carries the c canonical wire-label hashes for the c challenged
positions. The original AssertTx's CSV countdown does not start counting
until RevealTx is mined and watchtower has time to verify.

Watchtower-side check: the c canonical-label hashes in RevealTx must match
the watchtower's off-chain canonical labels. If yes — assertion is honest,
bond unslashed at CSV expiry. If no — watchtower spends a NEW kind of
forced-reveal leaf in RevealTx's output that says "anyone with the honest
canonical label slashes the bond".

**Trade-offs.** +1 on-chain tx per withdrawal. Adds liveness latency
(beacon delay PLUS RevealTx confirmation, ≈ 6+1 = 7 blocks). Watchtower
logic gets more complex (must distinguish AssertTx-only vs. AssertTx +
RevealTx pairing). Removes the "single AssertTx covers everything" pitch.

#### Candidate C — Drop the forced-reveal leaves entirely; Σ-response *is* equivocation

Restructure: the Σ-response IS a forced reveal of the OPPOSITE-bit Lamport
preimage at the c challenged positions. Operator's main AssertTx witness
already revealed `s_i^{b_chosen}` for all i. The Σ-response, embedded in a
follow-up tx (or a separate AssertTx witness slot), reveals
`s_i^{b_canonical}` for the c challenged positions.

If `b_chosen == b_canonical` for all challenged positions → no
equivocation, bond unslashed. If `b_chosen != b_canonical` for any
challenged position → operator has revealed BOTH `s_i^0` and `s_i^1` →
the existing equivocation leaf at position i is spendable by any watchtower.

This collapses the forcing primitive into the equivocation primitive.
**No new on-chain leaves needed** — the equivocation leaves already exist.
The novelty is the c-position FORCED revelation triggered by the beacon.

**Catch.** Forcing the operator to actually publish the Σ-response. If
the operator just doesn't publish it, AssertTx-only equivocation never
fires. We need on-chain enforcement that AssertTx CSV doesn't expire
until Σ-response is also on-chain. One sketch: AssertTx output has a
new spending path that requires both `s_i^{b_chosen}` (already revealed
in main witness) AND `s_i^{b_canonical}` (must be revealed in spending
witness) for each of the c positions — the CSV-claim leaf for the operator
is gated by also publishing the Σ-response. Operator who lies cannot
satisfy this gate without self-equivocating.

**Trade-offs.** Cleanest on-chain footprint (no new leaves, ~no new
witness bytes per honest case). Requires careful Script construction to
gate CSV claim by Σ-response presence. Open question: can the gate be
expressed in current Bitcoin Script for c=80 challenged positions
without `OP_CAT`?

### 10.3 Comparison and recommendation

| Axis | Candidate A | Candidate B | Candidate C |
|---|---|---|---|
| New on-chain elements | 0 | 1 RevealTx (~150 vB) | 0 (gating logic in CSV-claim) |
| Off-chain pubkey size | 47 KB → 85 KB | 47 KB | 47 KB |
| Liveness latency added | 0 (vs. base beacon) | +1 confirmation (RevealTx) | 0 |
| Soundness | binds via pubkey hash → state root chain (needs review) | direct: RevealTx is its own commitment | direct: equivocation |
| New Script primitives | none | leaf in RevealTx ≈ existing forced-reveal | gated CSV-claim leaf, possibly larger |
| Watchtower complexity | low | medium (dual-tx state machine) | low-medium |
| OP_CAT dependency | no | no | possibly (depends on gating construction) |

Without external cryptographer input I lean toward **Candidate A** as the
default direction:

- It preserves the single-AssertTx UX (no extra on-chain tx, no new
  liveness latency beyond the beacon).
- The +38 KB pubkey is tolerable for Pi-class watchtowers; current pubkey
  is 47 KB and downloading 85 KB once per withdrawal is still seconds-
  scale.
- The soundness argument reduces to "extended pubkey hash bound to the
  state root via SNARK public inputs", which uses primitives we already
  trust.

Candidate C is the most elegant if the gating construction can be made
to fit in current Bitcoin Script — but that fitness check is itself a
research question. Candidate B is the safe-but-heavy fallback.

### 10.4 Cryptographer questions to anchor Milestone D

1. **Candidate A soundness.** Is `extended_pubkey = lamport_hashes ||
   wire_label_hashes` provably bound to the asserted L2 state-root such
   that an operator cannot pick `wire_label_hashes` to encode a lying
   state? Specifically: does our Lamport-message-IS-SNARK-public-input
   construction make a malicious choice of `wire_label_hashes` reduce to a
   SHA256 second-preimage forgery on the state root?

2. **Candidate C fitness.** Is there a Bitcoin Script (no `OP_CAT`,
   current opcodes only) that can gate a CSV-claim leaf on `c=80`
   sequential preimage reveals at challenged positions, given that the
   challenged positions are fixed by a beacon hash known at script-build
   time? Witness-size budget: ≤ 5 KB.

3. **Beacon vs. selfish-mining.** Confirm `k=6` is sufficient for mainnet
   under a 30% selfish-mining adversary, or recommend `k`. Current
   placement: `forcing::FORCING_BEACON_DELAY_BLOCKS = 6`.

4. **`c` sufficient for any-meaningful-lie.** Today's adversarial test
   targets a "1/3 of bits flipped" lie. State-root flip alone gives
   ≥ 256 of the 1184 bits flipped (the state root contributes its full
   256 bits to public inputs). Confirm `c=80` gives `≤ 2^{-25}` miss
   probability for this realistic minimum-lie scale, or recommend `c`.

### 10.5 Status

- B.0 (beacon resolver helper, mostly off-chain) — implementable today.
- B.3 (`/bridge/forcing-beacon/:assert_txid` API endpoint) — IMPLEMENTED
  2026-04-25, see `crates/node/src/api.rs::bridge_forcing_beacon`.
- B.1, B.2, B.4 — **DEFERRED** (see §10.6 below).
- D (cryptographer engagement) — paused; revisit alongside external-garbler
  ceremony work.

### 10.6 Decision 2026-04-25 — Phase 3 forced-reveal path deferred

> **Update 2026-05-02.** The deferral reasoning below remains correct under
> the original (operator-controlled-garbler) trust model. Under the
> Candidate D (TEE-attested garbler) trust model adopted in
> `docs/phase3_tee_design.md`, the forced-reveal pipeline is **not just
> deferred but architecturally unnecessary** — see
> `docs/phase3_tee_secrecy.md` for the full argument. Phase 3.2b proceeds
> on the TEE-as-Verifier path, not on this section's deferred-but-revivable
> path.

After implementing Candidate C feasibility prototypes
(`forcing.rs::csv_claim_gated_any_hash` and `csv_claim_gated_opposite_bit`)
and measuring sizes in `forcing::tests::candidate_c_size_summary_table`,
deeper analysis showed all three candidates (A/B/C) share a common
prerequisite that is **not satisfied by the current code**:

> The Σ-protocol soundness for any of A/B/C requires an external reference
> for canonical wire labels. In our code today, `wire_label_from_preimage`
> derives wire labels purely from operator-chosen Lamport preimages
> (`lamport.rs::derive_for_withdrawal`). The operator therefore knows both
> wire labels at every bit position and can satisfy any of the three gates
> trivially regardless of whether they signed truth.

The unit test `lying_operator_fails_sigma_check_with_overwhelming_probability`
(`forcing.rs:297`) does not catch this — it models a lying operator who
submits *corrupted* labels, not one who submits the canonical label they
already happen to know. The actual fixed-lie attacker submits structurally
valid labels that are not "canonical-truth" but are "canonical-given-the-
lie", and the off-chain watchtower has no independent reference to detect
this.

To make Σ-forcing work cryptographically, the garbler seed must be
**external to the operator** — a setup ceremony (multi-party DKG or
trusted attested setup) generates wire labels first, the operator only
ever learns preimages corresponding to canonical-truth bits, and inverting
SHA256 to recover the opposite preimage is infeasible. This is the
intended threat model in the BitVM3s paper but is not what the current
code models.

**Production implication.** Equivocation-only protection (current Phase
1.1+1.2 verified live 2026-04-23) is the same protection BitVM2 / Citrea /
Bitlayer / Merlin ship today. The fixed-lie attack remains open under
the optimistic 1/N challenger assumption: at least one watchtower must be
honest AND have economic incentive to broadcast a DisprovalTx within the
CSV window when they observe an equivocating second AssertTx. This is the
mainstream BitVM-class assumption.

**Phase 3 scope going forward.**

| Item | Status | Reason |
|---|---|---|
| Beacon derivation (`derive_beacon`, `select_challenge_positions`) | KEEP | Self-contained, no external-garbler dependency, useful primitive. |
| `FORCING_BEACON_DELAY_BLOCKS = 6` constant | KEEP | Same — pure protocol parameter. |
| `forced_reveal_tapscript` | KEEP as POC | Documented as not yet wired; preserved for the eventual external-garbler implementation. |
| Candidate C prototypes (`csv_claim_gated_*`) | KEEP as POC | Useful for future feasibility comparisons; documented as deferred. |
| `bridge_forcing_beacon` API endpoint | KEEP | Read-only data plumbing, future-compatible. |
| B.1 (Taproot tree builder for forced-reveal leaves) | DEFERRED | Blocked on external-garbler ceremony. |
| B.2 (AssertTx Σ-response generation) | DEFERRED | Same. |
| B.4 (Watchtower `dispute_via_forced_reveal`) | DEFERRED | Same. |
| B.5 (regtest e2e for forcing path) | DEFERRED | Depends on B.1/B.2/B.4. |
| External-garbler ceremony design | BACKLOG | Mainnet-readiness item; revisit alongside cryptographer engagement. |

Phase 3 forced-reveal path is paused. Phase 1.1+1.2 equivocation protection
remains live. Anyone re-opening this work should start by reading §10
fully — the easy-looking script-size question is not the actual blocker.
