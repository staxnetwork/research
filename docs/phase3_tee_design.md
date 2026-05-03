---
# Phase 3.1 — TEE-Attested Garbler (Candidate D)

**Status:** Design proposal — open for review.
**Author:** Stax Network team.
**Last updated:** 2026-05-01.
**Scope:** complement to `docs/forcing_mechanism_design.md`. That doc proposes
Candidates #1 (shared-randomness) and #2 (input-label commitment) — both
**require an honest canonical-wire-label generator** (the garbler). This doc
addresses *who runs the garbler and why we should trust them*.
**Code changes:** Phase 3.2 (separate doc).

---

## 1. Why TEE is on the table

### 1.1 The hidden assumption in Candidates #1 and #2

Both Σ-protocol candidates rely on a **canonical garbled-circuit wire-label
table** that the watchtower compares the operator's revealed preimages
against. Concretely:

```
forced_reveal_leaf(position i, canonical_label L_i^b)
    ↳ "the preimage you reveal at bit i must equal L_i^b"
```

If the operator generated `L_i^b` themselves, they know **both** labels
`(L_i^0, L_i^1)` and can choose to reveal either — the forcing primitive
collapses. The whole construction assumes there exists *some party* who knows
exactly one wire label per bit and whose output the watchtower can rely on as
"canonical".

In the BitVM2/Citrea/Bitlayer literature this party is an **N-of-N committee
ceremony**: N independent garblers run a DKG and discard their share so no
single party knows both labels. This is the right answer in a vacuum, but
operationally it requires:

- Recruiting N reputable independent parties.
- Coordinating a 6-9 week ceremony per circuit version.
- Repeating per major upgrade.
- Trusting that no N parties collude.

For a solo team launching to mainnet on a 2-3 month timeline, the ceremony is
not free. Candidate D asks: **can a TEE substitute for the ceremony**?

### 1.2 What "TEE substitution" buys us

A garbler running inside an attested TEE produces a binding statement:

> "The garbled-circuit commitment `C` was produced by code with hash `H_code`
> on hardware platform `P`, and no party including the host operator can
> recover the wire labels."

If we can verify this attestation on-chain (or off-chain in a way the
watchtower trusts), then the operator does not know both wire labels — the
TEE keeps one half secret. The forcing leaf works as designed.

**Trust assumption replacement:**

- Without TEE: trust an N-of-N ceremony (no single party = both labels).
- With TEE: trust the TEE vendor's attestation chain + the TEE silicon.

Neither assumption is free. The question is which is **cheaper to ship for
solo team mainnet timeline** while still being *honestly defensible* in
audit and threat-model writeups.

### 1.3 Why this is being proposed AFTER the Σ-protocol doc

`forcing_mechanism_design.md` (2026-04-25) recommended Candidate #1 with the
implicit assumption that someone — eventually — would run the garbler ceremony.
Phase 3.2 milestones A–C in that doc punt the garbler-trust question to
"Milestone D — Audit-ready, parallel with B/C".

When we tried to plan Milestone D concretely (2026-04-30), it became clear
that "recruit N independent garblers" is not a planning artifact — it is a
6-9 week sub-project with external dependencies. TEE is the *software-only*
substitute that lets the solo team ship Phase 3 end-to-end without blocking
on third parties.

---

## 2. Goal of Candidate D

Provide a canonical wire-label generator with the following properties:

1. **Honest by construction.** The host operator cannot extract both wire
   labels for any bit position.
2. **Attestable.** A watchtower (or auditor) can verify that the labels were
   produced by reviewed source code, not by the operator's whim.
3. **Reproducible.** Anyone with the same code hash and seed can re-derive the
   same canonical labels for cross-checking — the TEE is a *witness* of
   honest execution, not the only source of truth.
4. **Solo-team feasible.** No multi-party ceremony, no external dependencies.
5. **Replaceable.** When/if an N-of-N ceremony becomes available, the TEE can
   be retired without changing on-chain protocol — the canonical labels
   simply have a different provenance.

Property (3) is critical: if the TEE is the *only* honest party, a
TEE-vendor compromise is fatal. We want the TEE to be one of *multiple*
independent witnesses, where the others can re-derive labels from public
inputs and a published seed. Vendor compromise then degrades us back to "one
honest garbler is enough" — same as today.

---

## 3. Platform comparison

Three TEE platforms are mainstream and could host the garbler:

### 3.1 AWS Nitro Enclaves

- **Hardware:** custom AWS silicon, attestation rooted in AWS PKI.
- **Code model:** EIF (Enclave Image File) built from a Docker image; runs
  isolated from the parent EC2 instance.
- **Attestation:** signed COSE document containing PCR0 (image hash), PCR1
  (kernel), PCR2 (user data), plus AWS root certificate chain.
- **Memory limit:** up to ~120 GB on m5n.metal — enough for our 6.6 MB BABE
  circuit but not for the full 42 GB Yao GC. Standard build (47 KB pubkey,
  Lamport) fits trivially.
- **Hetzner availability:** none. AWS-only.
- **Cost (2026 pricing):** smallest viable instance is `m5n.large`
  ($0.149/hr → ~$110/mo on-demand, ~$70/mo with 1-yr reserved). Garbler is
  not a 24/7 service — runs on-demand per circuit version, so spot pricing
  (~$45/mo equivalent) is reasonable.
- **CVE history:** clean public record as of 2026-05; no published Nitro
  enclave breaks. Smallest TCB of the three (no x86 microcode surface).

### 3.2 AMD SEV-SNP

- **Hardware:** AMD EPYC 7xx3 / 9xx4 CPUs.
- **Code model:** runs an entire VM as the TEE; whole guest kernel is in TCB.
- **Attestation:** SEV-SNP attestation report signed by AMD VCEK (per-CPU
  key), rooted in AMD certificate chain.
- **Memory limit:** entire host memory. No constraint for our circuits.
- **Hetzner availability:** **yes** — Hetzner dedicated EPYC servers
  (AX102 with 9700X, AX162 with 9950X) expose SEV-SNP. €50-80/mo.
- **CVE history:** several published attacks (CVE-2023-20593 "Inception",
  CVE-2024-31157 "BadRAM", various memory-mapping issues). Most require
  privileged host access; a remote-only attacker against the enclave is hard,
  but the TCB is large (whole VM).

### 3.3 Intel SGX / TDX

- **SGX:** older, deprecated for new Xeon SKUs; small enclave, well-studied.
  Massive CVE history (Foreshadow, ÆPIC, SGAxe, Stealing-from-the-Brain).
  Skip — Intel themselves are deprecating it.
- **TDX:** Intel's modern equivalent of AMD SEV-SNP. Available on Sapphire
  Rapids / Emerald Rapids Xeons. Very new attestation tooling, fewer
  third-party validators. **Hetzner availability: limited** — only specific
  Intel dedicated lines, more expensive than AMD equivalent.

### 3.4 Comparison summary

| Axis | AWS Nitro | AMD SEV-SNP (Hetzner) | Intel TDX (Hetzner) |
|---|---|---|---|
| Vendor risk | AWS | AMD | Intel |
| CVE history | clean (2026-05) | several, host-priv mostly | new, low data |
| TCB size | small (custom OS) | large (whole VM) | large (whole VM) |
| Attestation tooling | mature, clean SDK | mature (snpguest, sevtool) | newer, less polished |
| Setup complexity | medium (EIF builds) | medium (libvirt + sev-guest) | medium-high |
| Cost | ~$45-110/mo | €50-80/mo | €100-150/mo |
| Hetzner-native | NO | YES | partial |
| Vendor lock-in | high (AWS-only) | low (any EPYC host) | low (any TDX host) |

### 3.5 Recommendation

**Default platform: AWS Nitro Enclaves.**

Justification:
- Smallest TCB of the three → smallest attack surface for a TEE-vendor
  compromise to exploit.
- Cleanest CVE record at the time of writing.
- Mature attestation SDK with clean Rust bindings (`aws-nitro-enclaves-cose`,
  `nsm-api`).
- Pay-per-use pricing matches the garbler's workload pattern (run for an hour
  to garble a circuit, idle for weeks).

**Secondary platform: AMD SEV-SNP on Hetzner dedicated.**

Use case: redundancy and vendor-diversity. We can run the garbler on **both**
platforms and require the canonical labels to match — degrading the trust
assumption from "AWS or AMD didn't backdoor us" to "AWS AND AMD didn't
collude". This is a Phase 3.2b polish, not Phase 3.2a critical path.

Skip Intel SGX entirely. Skip Intel TDX for now (revisit if AMD turns out to
have a Phase 3 blocker).

---

## 4. Attestation flow design

### 4.1 What the attestation must bind

The watchtower needs cryptographic confidence that a given canonical wire-
label table came from honest garbler execution. The minimum attestation
payload is:

```
{
    code_hash:    sha256(garbler_binary || vk || circuit_params),
    public_seed:  32 bytes (entropy used for label derivation, public),
    output_hash:  sha256(canonical_wire_label_table),
    timestamp:    unix seconds (for freshness),
    platform:     "aws-nitro" | "amd-sev-snp" | ...
}
```

Plus the platform-specific attestation document signed by the TEE vendor's
attestation key (Nitro PCR document, SEV-SNP attestation report).

### 4.2 Verification

A watchtower verifies in this order:

1. **Vendor signature.** Verify the platform attestation against the vendor
   root CA (AWS Nitro root, AMD SEV root). This binds the payload to "ran
   inside a real TEE on this platform".
2. **Code hash match.** Recompute `code_hash` from the published source +
   verifying key + circuit params. This binds the payload to "ran reviewed,
   open-source code". Source must be fully reproducible-build (Nix or
   `cargo build --locked` with a fixed Rust toolchain).
3. **Output hash match.** Recompute `output_hash` from the locally-derived
   wire-label table (using the same `public_seed`). This binds the
   attestation to the actual labels the watchtower will use.
4. **Freshness.** `timestamp` must be within the validity window for this
   circuit version (e.g. attestation issued within the last 90 days).

If all four pass, the watchtower trusts the canonical label table and the
forcing leaves built from it.

### 4.3 On-chain footprint

The attestation itself does NOT need to land on Bitcoin. Watchtowers
download it from:

- **Primary:** Stax API (`GET /bridge/garbler-attestation/:circuit_version`).
- **Fallback:** IPFS (CID published in Stax bridge contract event).
- **Auditor copy:** Git-tagged release in the public docs repo.

On-chain we publish only the SHA256 of `(output_hash || code_hash)` in a
StaxBridge contract event so the attestation cannot be silently swapped.
~32 bytes per circuit version. Negligible.

---

## 5. Integration points

### 5.1 The `bitvm3-garbler` crate

Already structured for this — it lives at
`crates/bridge/bitvm3-garbler/Cargo.toml` as a **standalone binary** outside
the main workspace, because of the `c-kzg v1.x` vs `v2.x` conflict between
`revm-primitives` and `sp1-sdk`. The isolation makes TEE wrapping clean:

- The garbler is already a single binary with deterministic CLI input/output.
- No network I/O — reads VK file, writes commitment + label table.
- Can be packaged directly into a Nitro EIF or a SEV-SNP guest disk image
  with no architectural surgery.

This was a happy accident from the c-kzg dependency split. Phase 3.2a
integration is "wrap the existing binary", not "rewrite the garbler".

### 5.2 New code surface

- **`crates/bridge/tee-attest/`** (new crate) — attestation document parser
  and verifier. Generic over platform.
  - `verify_nitro(doc: &[u8], expected_code_hash) -> Result<Attestation>`
  - `verify_sev_snp(doc: &[u8], expected_code_hash) -> Result<Attestation>`
  - `verify_mock(doc: &[u8], expected_code_hash) -> Result<Attestation>` (Phase 3.2a)
- **`crates/bridge/bitvm3-garbler/src/main.rs`** — extended to optionally
  emit an attestation document alongside the commitment, when running inside
  a TEE.
- **`crates/watchtower/src/main.rs`** — on startup, fetch attestation,
  verify, cache result. On AssertTx detection, look up canonical labels
  from the attested table.
- **`StaxBridge.sol`** — new event `GarblerAttestationRegistered(bytes32
  attestation_digest, uint256 circuit_version)` so the on-chain history of
  which attestation was active when matters for disputes.

### 5.3 What does NOT change

- On-chain Taproot tree shape, AssertTx witness layout, DisprovalTx vbytes,
  Lamport derivation — all per `forcing_mechanism_design.md`.
- The Σ-protocol candidate (#1 or #2) chosen for forcing — TEE is orthogonal,
  it just provides the canonical labels both candidates need.
- The watchtower portability story — verifying a TEE attestation is a few
  hundred ms of cryptography, well within Pi-class budget.

---

## 6. Threat model

### 6.1 Threats this addresses

| Threat | Without Candidate D | With Candidate D |
|---|---|---|
| Operator garbles their own circuit and reveals either label per bit | YES (forcing collapses) | NO (TEE keeps one half secret) |
| Operator publishes a swapped canonical-label table to watchtowers | YES | NO (attestation binds to source code) |
| Operator front-runs a future label table by publishing one early and switching at AssertTx time | YES | NO (on-chain attestation digest is tamper-evident) |

### 6.2 Threats this introduces

| Threat | Severity | Mitigation |
|---|---|---|
| TEE vendor (AWS, AMD) backdoors the platform | catastrophic | Multi-vendor cross-attestation in Phase 3.2b; auditor reproducible builds |
| Vendor attestation key leaks (e.g. AWS Nitro root key compromise) | catastrophic | No software mitigation — same severity as a CA root compromise; mitigated by short attestation validity windows + multi-vendor |
| TEE silicon CVE that lets host operator extract enclave secrets | high | Track CVE feed, rotate attestations on disclosure, accept ~weeks of vulnerability before patched build |
| Reproducible-build divergence makes attestation un-verifiable by third parties | medium | Pin Rust toolchain + use Nix or `cargo --locked`; CI-gate attestation regeneration |
| Operator runs a fake TEE (e.g. emulator returning forged attestation) | high | Vendor signature check (step 1 of §4.2) — emulators cannot forge AWS/AMD root signatures |

### 6.3 Threat model honesty

The honest characterization for SECURITY.md and audit:

> Stax bridge in Phase 3 with TEE garbler trusts: (a) Bitcoin honest majority,
> (b) Celestia DA, (c) at least one honest watchtower, (d) the TEE vendor's
> attestation chain. Assumption (d) is strictly weaker than the BitVM2 N-of-N
> committee assumption — TEE failure modes are concentrated in vendor and
> silicon trust rather than distributed across N parties. We accept this
> trade-off for solo-team mainnet feasibility and plan to retire (d) by
> migrating to a multi-party garbler ceremony post-mainnet.

This wording does not over-claim. Audit reviewers will respect honest
positioning more than "TEE = trustless".

### 6.4 Why this is still better than where we are today

Without Phase 3 we have **no forcing at all** — the operator can lie and
watchtowers cannot stop it on-chain. The trust assumption is "operator is
honest about state transitions" — full custody trust.

With Candidate D we trust TEE vendors **only for the canonical-label
generation step**. State validity, equivocation detection, DA, and slashing
all run on Bitcoin/Celestia primitives without TEE involvement. A TEE
compromise would let the operator successfully fix-lie ONE withdrawal — but
they still cannot rewrite L2 history, double-spend deposits, or steal funds
that aren't in their own withdrawal queue.

The strict ordering: today = full custody trust. With Candidate D = bounded
forcing trust. With multi-party ceremony = cryptographic forcing. We're
moving along this axis.

---

## 7. Two-phase implementation plan

### Phase 3.2a — Mock TEE (FREE, on existing infra)

**Duration:** 1-2 weeks solo.
**Cost:** €0. Runs on existing Stax server.

Goal: validate the integration architecture without committing to a TEE
platform or paying for hardware.

- Implement `verify_mock(doc, expected_code_hash) -> Attestation` — accepts
  any attestation signed by a hardcoded test key. This is OBVIOUSLY insecure
  but exercises every code path.
- Wrap `bitvm3-garbler` to emit a mock attestation alongside its output.
- Watchtower fetches and verifies mock attestations.
- StaxBridge contract event ships with mock attestation digest.
- Adversarial unit tests:
  - Operator-published label table that doesn't match attestation → rejected.
  - Stale attestation (timestamp > 90 days) → rejected.
  - Code-hash mismatch → rejected.
- Regtest e2e: full withdrawal cycle with mock-attested labels.

**Exit:** all forcing-leaf machinery works against mock-attested labels;
flipping to real TEE is a contained code change in `tee-attest/` crate.

### Phase 3.2b — Real TEE (PAID, when funds available)

**Duration:** 1-2 weeks once hardware is provisioned.
**Cost:** $45-110/mo (AWS Nitro spot) or €50-80/mo (Hetzner SEV-SNP).

Decision points to confirm before starting:

- **Platform.** Default = AWS Nitro per §3.5. Hetzner SEV-SNP is acceptable
  if cost dominates over TCB-size concern.
- **Funding source.** Personal credit card (Artur, ~$50-150/mo for first
  6 months) vs. wait for SAFT/seed funding (likely Q3 2026).
- **Reproducible build setup.** Nix or pinned Rust toolchain — must be
  decided BEFORE building the production EIF/disk image.

Implementation:
- Replace `verify_mock` calls with `verify_nitro` (or `verify_sev_snp`) in
  watchtower production config. Mock remains for tests.
- Reproducible-build CI for the garbler binary.
- Operational runbook: how to attest a new circuit version, how to publish
  the attestation digest on-chain, how to roll back if attestation
  verification fails after deploy.

**Exit:** mainnet-ready Phase 3 with real TEE attestation; mock retained for
tests/CI.

### Phase 3.2c (optional) — Multi-vendor cross-attestation

**Duration:** 1 week.
**Cost:** marginal (run on both platforms).

Run the garbler on **both** AWS Nitro AND Hetzner SEV-SNP. Watchtower
requires both attestations to match before trusting labels. Trust assumption
degrades from "AWS XOR AMD honest" to "AWS AND AMD didn't collude".

Defer until 3.2b is in production and ops cost is understood.

---

## 8. Decision log

| Date | Decision | Rationale |
|---|---|---|
| 2026-05-01 | Pursue Candidate D (TEE) over multi-party ceremony for mainnet v1 | Solo-team can ship in weeks vs. 6-9 weeks for ceremony; honest trust trade-off documented |
| 2026-05-01 | Default platform = AWS Nitro Enclaves | Smallest TCB, cleanest CVE history, mature SDK |
| 2026-05-01 | Two-phase plan: Mock TEE first | Lets development proceed FREE on existing infra; real TEE deferred until funding |
| 2026-05-01 | Skip Intel SGX entirely | Deprecated by Intel; massive CVE history |
| 2026-05-01 | TDX deferred | New tooling, less data, AMD SEV-SNP covers same niche cheaper |
| 2026-05-01 | Multi-vendor (3.2c) deferred to post-mainnet | Polish, not critical path |

---

## 9. Open questions

- **Reproducible builds for `garbled-snark-verifier`.** The dependency chain
  pulls SP1 SDK; non-trivial to make fully deterministic. May require
  vendoring or upstream patches. Investigate before committing to TEE.
- **Attestation rotation policy.** How often do we re-attest the same
  circuit? On every mainnet release? On vendor key rotation? Pick a default
  in Phase 3.2a, revisit after operational data.
- **Watchtower behavior on attestation expiry.** Refuse to attest new
  AssertTxs but continue to honour in-flight ones? Or refuse all? Conservative
  default: refuse all — but that's a liveness hit. Decide in 3.2a.
- **Public key for mock attestation.** Hardcode in source vs. generate at
  build-time. Source-hardcoded is obviously-insecure-by-design and harder to
  forget to swap. Recommendation: hardcode + add a startup warning when in
  use.
- **Migration path to ceremony.** If we successfully recruit garblers
  post-mainnet, what does the rollover look like? Probably: ceremony output
  becomes the new "canonical" labels, TEE attestation continues in parallel
  for a transition period, then TEE retires. Document explicitly when
  ceremony becomes available.

---

## 10. Relationship to other docs

- `docs/forcing_mechanism_design.md` — defines Candidates #1 (Σ-protocol) and
  #2 (input-label commitment) and the fixed-lie threat model. **This doc
  presupposes that one of those candidates is chosen.** Candidate D is the
  *garbler-trust* layer underneath either.
- `docs/SECURITY.md` (TBD, Phase 4) — will absorb §6 of this doc into the
  formal threat-model writeup.
- `docs/RUNBOOK_MAINNET.md` (TBD, Phase 4) — will absorb the operational
  runbook from Phase 3.2b.
