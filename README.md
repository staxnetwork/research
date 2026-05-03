# Stax Network — Research

Public design docs and cryptographic research from the Stax Network team.

Stax is a Bitcoin L2 with a BitVM-class bridge, currently live on signet.
This repo holds the design documents we want public review on. Production
node, sequencer, watchtower and bridge code lives in a separate (currently
private) repository — see [stax.network](https://stax.network) for context.

## Documents

- **[forcing_mechanism_design.md](docs/forcing_mechanism_design.md)** —
  Σ-protocol forcing candidates for fixed-lie defence in BitVM-style
  bridges. §10 documents why all on-chain forcing variants we examined
  collapse without an external canonical-wire-label generator. §10.6 is
  the deferral decision that motivated the TEE work below.

- **[phase3_tee_design.md](docs/phase3_tee_design.md)** — Candidate D:
  TEE-attested garbler. Platform comparison (AWS Nitro vs AMD SEV-SNP vs
  Intel TDX/SGX), attestation flow, threat model, two-phase implementation
  plan.

- **[phase3_tee_secrecy.md](docs/phase3_tee_secrecy.md)** — TEE-as-Verifier
  framing that bypasses the canonical-label distribution problem entirely
  and collapses the on-chain forcing pipeline. Currently open for
  cryptographer review (§7 lists 7 questions).

## Status

These are working drafts, not finished specifications. We publish them to
get external review before committing to implementation. Substantive
feedback gets folded back into the docs with attribution.

## Contact

Issues and discussions on this repo are the canonical place for technical
feedback. For private notes: tirionartur@gmail.com.

## License

Documents are released under CC BY 4.0 — quote freely with attribution.
