# BountyQuorum

**On-Chain Milestone Escrow & Quorum-Verified Bounty Board** built in native Gno for DAOs, grant programs, and builder/agent quests.

## Why BountyQuorum?

Ecosystem grants and technical bounties routinely fail on two fronts: either funds are sent upfront with no milestone accountability, or contributors finish work and wait weeks for manual multisig payouts.

**BountyQuorum** fixes both on-chain with zero external dependencies:

- **Trustless Native `ugnot` Escrow**: Sponsors lock `ugnot` directly inside the realm when creating a quest (`CreateQuest`), and anyone in the community can co-fund an active quest via `TopUpQuest`.
- **Zero-Dust Weighted Milestones**: Sponsors break work into basis-point weighted milestones (sum ≤ 10,000 bps). When the final milestone completing 100% is approved, the realm pays out exact `remainingEscrow` so integer-division dust is never stranded.
- **Multi-Reviewer Quorum Verification**: Sponsors specify a reviewer set and quorum threshold (e.g., 2-of-3). The instant approval quorum is reached, `ReviewMilestone` atomically transfers the escrowed `ugnot` straight to the builder.
- **Iterative Resubmission Loop**: If enough reviewers reject a submission that quorum can no longer be reached, the milestone cleanly transitions to `REJECTED` with reviewer comments preserved in the audit log, letting the builder fix issues and resubmit a new round without redeploying or losing escrow.
- **Safety Guarantees**:
  - Direct EOA guard (`cur.Previous().IsUserCall()`) before reading `unsafe.OriginSend()`, preventing phantom `OriginSend` draining via intermediary realms.
  - Per-round double-vote prevention (`<round>:<reviewer>`) and builder self-review prohibition.
  - Deadline-protected `ReclaimExpired` and pre-payout-only `CancelUnstarted`.
- **Multi-View `gnoweb` Portal + Agent JSON Feed**:
  - `/` — Live protocol metrics, quest board with ASCII progress bars, verified builder leaderboard.
  - `:quest/<id>` — Milestone table, escrow breakdown, deliverable proof links, full review audit log.
  - `:builder/<addr>` — Cumulative verified earnings and completed milestone history.
  - `:stats` — Compact JSON metrics feed for off-chain indexers and autonomous agents.

## Architecture

| Module | Path | Purpose |
| --- | --- | --- |
| Pure Package | `p/quorummath` (`gno.land/p/g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd/quorummath`) | Deterministic bps payout math (zero-dust), quorum state machine, CSV address validation, `ugnot` & progress-bar formatting |
| Stateful Realm | `r/bountyquorum` (`gno.land/r/g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd/bountyquorum`) | Native `ugnot` escrow (`chain/banker`), quest/milestone lifecycle, structured `chain.Emit` events, multi-route `Render()` |

## Running the Test Suite

```bash
gno test -v ./p/quorummath ./r/bountyquorum
```

All 7 unit and end-to-end realm tests pass (including 2-of-3 rejection → resubmission → approval flow, zero-dust 3333/3333/3334 bps payout verification, bank balance accounting, `Render()` route checks, deadline enforcement, and cancellation guards).

## Deploying to Testnet (`onyx-1`)

```bash
# 1. Deploy pure math package
gnokey maketx addpkg \
  -pkgpath "gno.land/p/g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd/quorummath" \
  -pkgdir "./p/quorummath" \
  -gas-fee 1000000ugnot -gas-wanted 5000000 \
  -broadcast -chainid onyx-1 -remote https://rpc.latest.testnets.gno.land:443 \
  <your_key>

# 2. Deploy BountyQuorum realm
gnokey maketx addpkg \
  -pkgpath "gno.land/r/g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd/bountyquorum" \
  -pkgdir "./r/bountyquorum" \
  -gas-fee 1000000ugnot -gas-wanted 10000000 \
  -broadcast -chainid onyx-1 -remote https://rpc.latest.testnets.gno.land:443 \
  <your_key>
```


## Verified Testnet Deployment (`onyx-1`)

Both modules are deployed on Gno.land `onyx-1` (`https://rpc.latest.testnets.gno.land:443`) by `g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd`:

| Module | Package Path | Block Height | Tx Hash | Status |
| --- | --- | --- | --- | --- |
| Pure Math Package | `gno.land/p/g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd/quorummath` | `1388466` | `HUPw9dpvgVj4B/waLYdZ0Msgit2Oz2VBdcJK+RQ12DU=` | `live` (verified via `vm/qpkgmeta_json`) |
| Escrow Realm | `gno.land/r/g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd/bountyquorum` | `1388467` | `fmNqmEO06/LJBnYhcGRSG+o2M7uihoClAHLTvgQvKd8=` | `live` (verified via `vm/qpkgmeta_json`) |

Verify on-chain metadata directly via `gnokey`:

```bash
gnokey query vm/qpkgmeta_json -data "gno.land/r/g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd/bountyquorum" -remote https://rpc.latest.testnets.gno.land:443
```

## Verified `CreateQuest` on `onyx-1`

Quest **#1**, **Genesis Ecosystem Indexer Bounty (Testnet Demo)**, was created on-chain at block **1398381** with **1,000,000 `ugnot`** in testnet escrow and a **1-of-1** reviewer quorum. The RPC receipt records `QuestCreated`; an independent `vm/qrender` query confirmed the quest is `OPEN`. No milestones or payouts were executed in this demo.

- **Transaction:** [`rx2SaHgVoyRPB5u6kIjbLUnQXzxaOh+QhPWBzhLIvXk=`](https://rpc.latest.testnets.gno.land:443/tx?hash=0xAF1D92687815A3244F079BBA9088DB2D49D05F3C5A3A1F9084F581CE12C8BD79)
- **Quest readback:**

```bash
gnokey query vm/qrender -data "gno.land/r/g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd/bountyquorum:quest/1" -remote https://rpc.latest.testnets.gno.land:443
```

## Example On-Chain Walkthrough

```bash
# Create a 2-of-3 quest funded with 1 GNOT (1000000ugnot) for 72 hours
gnokey maketx call -pkgpath "gno.land/r/g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd/bountyquorum" -func CreateQuest \
  -args "Build Indexer" -args "Index realms to SQLite" \
  -args "g1rev1...,g1rev2...,g1rev3..." -args 2 -args 72 \
  -send 1000000ugnot -gas-fee 1000000ugnot -gas-wanted 5000000 \
  -broadcast -chainid onyx-1 -remote https://rpc.latest.testnets.gno.land:443 <sponsor_key>

# Add weighted milestones (50% + 50%)
gnokey maketx call -pkgpath "gno.land/r/g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd/bountyquorum" -func AddMilestone \
  -args 1 -args "Core Parser & Tests" -args 5000 ...

# Builder submits proof -> Reviewers call ReviewMilestone -> Instant ugnot payout on quorum!
```

## License

MIT
