# DoraHacks Gnome Tavern Hackathon — Submission Packet

- **Hackathon**: Gnome Tavern Hackathon (`https://dorahacks.io/hackathon/gnome-tavern`)
- **Project Name**: `BountyQuorum` *(strictly complies with rule prohibiting "Gno" prefix)*
- **Tagline**: On-Chain Milestone Escrow & Multi-Reviewer Quorum Bounty Board built in native Gno
- **GitHub Repository**: `https://github.com/a415635511-alt/bountyquorum`
- **Demo Video**: `bountyquorum_demo.mp4` (local submission asset, 60s, 1280×720; shows the 7 passing Gno tests)
- **Testnet Chain**: `onyx-1` (`https://rpc.latest.testnets.gno.land:443`)
- **Deployer Address**: `g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd`
- **Deployed Package Paths**:
  - Pure Package: `gno.land/p/g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd/quorummath` (Block `1388466`, Tx `HUPw9dpvgVj4B/waLYdZ0Msgit2Oz2VBdcJK+RQ12DU=`)
  - Escrow Realm: `gno.land/r/g1yf6p3wr7lame69dqx6savrskkpu7qgalsxe5qd/bountyquorum` (Block `1388467`, Tx `fmNqmEO06/LJBnYhcGRSG+o2M7uihoClAHLTvgQvKd8=`)
  - Status: Both packages are `live` per `vm/qpkgmeta_json`.
- **Verified Testnet Interaction**: `CreateQuest` created **Quest #1** at block **1398381**; [RPC receipt (`QuestCreated`)](https://rpc.latest.testnets.gno.land:443/tx?hash=0xAF1D92687815A3244F079BBA9088DB2D49D05F3C5A3A1F9084F581CE12C8BD79), transaction `rx2SaHgVoyRPB5u6kIjbLUnQXzxaOh+QhPWBzhLIvXk=`. The realm readback shows `OPEN` with **1,000,000 `ugnot`** in escrow and a **1-of-1** reviewer quorum. The escrow uses testnet tokens; there was no payout.
- **Local Verification**: `gno test -v ./p/quorummath ./r/bountyquorum` — 7/7 tests passing with zero external package dependencies.

## Full Description (Copy-Paste into DoraHacks BUIDL)

BountyQuorum is a trustless on-chain milestone escrow and multi-reviewer quorum bounty board written in native Gno with zero external package dependencies.

### Problem
DAO grants, open-source bounties, and autonomous agent tasks usually fail on one of two extremes: upfront lump-sum payouts with no delivery accountability, or completed deliverables stuck waiting weeks on manual multisigs.

### Solution
BountyQuorum splits responsibilities into a pure deterministic math package (`p/quorummath`) and a stateful native `ugnot` escrow realm (`r/bountyquorum`):
- **Native `ugnot` Escrow & Co-Funding**: Sponsors lock `ugnot` via `CreateQuest` (`cur.Previous().IsUserCall()` guard prevents phantom `unsafe.OriginSend()` draining by intermediary realms), and anyone can co-fund open quests via `TopUpQuest`.
- **Zero-Dust Basis-Point Milestones**: Sponsors split quests into milestones summing up to 10,000 bps. The final milestone completing 100% disburses exact `remainingEscrow` so integer-division dust is never stranded.
- **M-of-N Reviewer Quorum & Iterative Resubmission**: Designated reviewers vote approve/reject per submission round (`ReviewMilestone`), blocking double votes and builder self-reviews. Reaching approval quorum atomically sends `ugnot` via `banker.BankerTypeRealmSend`. If quorum becomes mathematically unreachable, the milestone cleanly transitions to `REJECTED` with reviewer feedback logged on-chain so builders can iterate and resubmit.
- **Sponsor Safety Hatches**: Deadline-protected `ReclaimExpired` and pre-payout-only `CancelUnstarted`.
- **Multi-Route `gnoweb` Portal + Agent JSON Feed**: Serves `/`, `:quest/<id>`, `:builder/<addr>`, and `:stats` JSON metrics plus structured `chain.Emit` events.
