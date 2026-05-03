# Stax Bridge — Signet Operational Record

The Stax bridge has been running on a Bitcoin signet since 2026-04-22. This
document records concrete transaction hashes for the three end-to-end
scenarios that exercise the bridge's defence mechanisms.

> **Note on signet visibility.** Stax operates a private signet (custom
> challenge), so these transaction hashes are **not resolvable on the public
> mempool.space/signet explorer**. They are reproduced here as a record from
> the operator's `bitcoind` and watchtower journals. Independent
> reproducibility on a public network is part of the mainnet-readiness
> roadmap; until then, treat this file as a verifiable claim by the
> operator, not a trust-minimised proof. We will repeat each scenario on
> public Bitcoin mainnet as part of the mainnet launch.

---

## 1. Equivocation detection + DisprovalTx (Phase 1.1)

**Date:** 2026-04-23
**Scenario:** A simulated lying operator broadcasts two `AssertTx`
transactions with conflicting Lamport bit reveals at the same position.
Watchtower detects the equivocation, derives both Lamport preimages from
the conflict, and broadcasts a `DisprovalTx` that slashes the operator's
bond.

| Event | Bitcoin block | Transaction |
|---|---|---|
| `AssertTx#1` (truthful claim) | 10108 | `98ea1013ed9c7d8db177b7fed6d0821873a54cee0833bbb3ca81d3b796564dd7` |
| `AssertTx#2` (conflicting claim) | 10110 | `06262d57ef1959631d1cedecd2f8faf69bd207db3218c5457032070d5524abf9` |
| Watchtower `EQUIVOCATION DETECTED` (bit=0) | n/a | log event 19:23:28 UTC |
| `DisprovalTx` (bond slashed) | 10112 | `9ca4a53f93184a718cf034c67f165e2d7120894b97ef21bd801224841f97aaa9` |

- Bounty: **2,096,500 sats** to watchtower address
  `tb1qghsqratstptnp0nn9pr6dyt23sdgmzklm49gaz`.
- Time from second AssertTx broadcast to DisprovalTx broadcast: **~334 ms**
  (watchtower polled the block, ran equivocation classifier, constructed
  and signed DisprovalTx, broadcast).
- Demonstrates: standard BitVM-class equivocation defence works end-to-end
  with real Lamport preimage derivation, real Taproot leaf spending, real
  bond slashing on Bitcoin signet.

## 2. Celestia DA strict-mode refusal (Phase 1.2)

**Date:** 2026-04-23
**Scenario:** A fake `AssertTx` is broadcast with a Celestia DA commitment
referencing a block height that does not contain the claimed blob. The
watchtower runs in `--celestia-strict` mode (per Phase 2.5+ DA enforcement
work), queries the Celestia light node, fails inclusion attestation, and
**refuses to watch the AssertTx** rather than proceeding into a dispute it
cannot validate.

| Event | Bitcoin block | Transaction |
|---|---|---|
| Fake-CLST `AssertTx` | 10123 | `07d288789e4e0c23b5fa3412cc1ad6ed9f9d2634d5597207375c22b9fca40d10` |
| Watchtower `Celestia inclusion attestation FAILED ... strict mode — treating as fraud` | n/a | log event 20:28:57 UTC |

- The attestation failed because the AssertTx referenced
  `height=1 commitment=eeee...` which was below the Celestia Mocha tail
  (`10944466`).
- Demonstrates: the operator-withholding attack (operator publishes
  AssertTx referencing data they did not actually publish to DA) is
  detected before the watchtower commits to a dispute path.

## 3. Honest deposit + mint (Phase 1.4)

**Date:** 2026-04-24
**Scenario:** Happy path. A BTC deposit is sent to a derived P2TR address.
Bridge watcher confirms the deposit, the sequencer queues a `mint()` call
on the bridge contract, both native BTC and wBTC are credited to the L2
recipient address inside one L2 block.

| Event | Identifier |
|---|---|
| BTC deposit (80,000 sats) | `aea35e67df87e748b6e8a3a88a657b55a4f982438febccd0cce9c0b95a71750a` |
| Deposit BTC address (P2TR) | `tb1pgn5uhysredlvh8zgddx0kuptv85uvcxwnuqlc4vjzqfw3jwp7lnq6vfu9m` |
| L2 recipient | `0xDeAdBeEf00000000000000000000000000001406` |
| StaxBridge contract | `0x5FaE0F9B8c0CB7220101Ac45F0eD3fFF167AfdCD` |
| wBTC ERC-20 contract | `0x6f4B1fd44F70A540294E065741E1473c890A9840` |
| L2 block | `#68635` (2 transactions, both succeeded) |

- Demonstrates: the bridge happy-path with both native BTC accounting and
  ERC-20 wBTC minting in a single L2 block, against the same redeployed
  StaxBridge contract that survived Bug #6/#8/#9 fixes.

---

## How to reproduce on a public network

Mainnet repeat is on the roadmap. The bridge code is in a private
repository pending publication strategy (see
[project_stax_github.md] in the team's planning notes); reproduction
instructions on Bitcoin testnet4 will accompany the public release of the
bridge crates.

For private verification, contact the team — we can give you read access
to the signet `bitcoind` RPC and the watchtower journal so you can pull
the same log lines we cite above.
