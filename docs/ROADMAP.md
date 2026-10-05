# Agora Roadmap

Agora is built in three stages: a **proof of concept (POC)**, a **minimum viable product (MVP)**, and a **v1.0** public release. Each stage is a GitHub milestone with a parent issue. Its sub-issues are split by area, the same way as the test suites in [TESTING.md](TESTING.md), and each one names the test IDs it must turn green.

The milestone pages show where development stands: [POC](https://github.com/mayaberries/agora/milestone/1), [MVP](https://github.com/mayaberries/agora/milestone/2), [v1.0](https://github.com/mayaberries/agora/milestone/3).

## How we work: test-driven

Every behaviour is specified as a test case before it is built, so nothing is assumed along the way.

1. **Pick a development issue.** Its "Behaviour to turn green" list is the acceptance criteria; the full cases live in the linked testing issue.
2. **Settle blocking decisions first.** If a case depends on an open question, decide it in the testing issue and update the case. Don't pick an answer in code.
3. **Red:** write the tests, named by ID (TESTING.md §7), and see them fail.
4. **Green:** implement the least code that makes them pass.
5. **Refactor** with the tests green, then open a PR that references the development issue and the test IDs.

Behaviour without a test ID doesn't get built. Add the case to the testing issue first, with a new ID.

## POC: Prove the core flow end to end

Prove the core claim on one machine, with no UI: a book can be identified by its text, a quote can be proven or refuted, and two nodes can share the book and a discussion about it.

Parent issue: [#21](https://github.com/mayaberries/agora/issues/21). Corpus: C1–C4, C6–C9 ([#3](https://github.com/mayaberries/agora/issues/3)).

What the stage proves:

- Node A ingests C1, registers its Work, Edition and File, and signs the root hash.
- Node B fetches the file from A over iroh-blobs and verifies it.
- A quote proof from C1 verifies against the root, and C6's altered paragraph fails.
- A post anchored to C1 gossips from A to B; an invalid record is dropped.

Exit criteria:

- Every test ID assigned to this stage is green in the fast and PR stages
- The flow above is covered by green tests (FRB-01, KEY-02, NET-03, MRK-04/05, NET-06/07)

|Area|Development issue|Test IDs|Testing issue|
|---|---|---|---|
|Scaffold the `agora` package and `agora_core` crate|[#24](https://github.com/mayaberries/agora/issues/24)|—|[#2](https://github.com/mayaberries/agora/issues/2)|
|Ingest EPUBs and normalize their text (NRM)|[#25](https://github.com/mayaberries/agora/issues/25)|NRM-01, 02, 03, 04, 05, 06, 07, 08, P1, P2|[#4](https://github.com/mayaberries/agora/issues/4)|
|Build Merkle trees and quote proofs (MRK)|[#26](https://github.com/mayaberries/agora/issues/26)|MRK-01, 02, 03, 04, 05, 06, 07, 08, 09, 10, 12, P1, P2|[#5](https://github.com/mayaberries/agora/issues/5)|
|Register Works, Editions and Files (FRB)|[#27](https://github.com/mayaberries/agora/issues/27)|FRB-01, 02, 04, 08|[#6](https://github.com/mayaberries/agora/issues/6)|
|Define signed, content-addressed records (REC)|[#28](https://github.com/mayaberries/agora/issues/28)|REC-01, 02, 03, 04, 05, 07, P1|[#8](https://github.com/mayaberries/agora/issues/8)|
|Create author identities and sign root hashes (KEY)|[#29](https://github.com/mayaberries/agora/issues/29)|KEY-01, 02|[#9](https://github.com/mayaberries/agora/issues/9)|
|Anchor posts to writings and build comment trees (THR)|[#30](https://github.com/mayaberries/agora/issues/30)|THR-01, 02, 05, 06, 08|[#12](https://github.com/mayaberries/agora/issues/12)|
|Share files and records between nodes over Iroh (NET)|[#31](https://github.com/mayaberries/agora/issues/31)|NET-01, 03, 06, 07|[#13](https://github.com/mayaberries/agora/issues/13)|
|Run the fast and PR stages on Linux|[#32](https://github.com/mayaberries/agora/issues/32)|—|[#19](https://github.com/mayaberries/agora/issues/19)|

## MVP: Ship a usable network for public communities

A non-technical person can install a client, register a public-domain book, discuss its passages in open communities and reading groups, and share them by QR code or short link. Author identities can rotate keys safely, and attestations are recorded.

Parent issue: [#22](https://github.com/mayaberries/agora/issues/22). Corpus: adds C5 ([#3](https://github.com/mayaberries/agora/issues/3)).

What the stage proves:

- Register C5 after C1: a new Edition, suggested as the same Work by near-duplicate detection.
- Rotate an author's key; earlier signatures stay valid and a thief's rotation fails.
- Join a reading group that needs approval, post on a passage, and share the thread by QR code.

Exit criteria:

- Every test ID assigned to this stage is green, and the nightly stage passes
- The first client is released on the platform chosen in its issue
- The Allure report is published ([#20](https://github.com/mayaberries/agora/issues/20))

|Area|Development issue|Test IDs|Testing issue|
|---|---|---|---|
|Link editions and take metadata (FRB)|[#33](https://github.com/mayaberries/agora/issues/33)|FRB-03, 05, 06, 07|[#6](https://github.com/mayaberries/agora/issues/6)|
|Detect near-duplicate editions (FZY)|[#34](https://github.com/mayaberries/agora/issues/34)|FZY-01, 02, 03, 04|[#7](https://github.com/mayaberries/agora/issues/7)|
|Enforce record limits and versioning (REC)|[#35](https://github.com/mayaberries/agora/issues/35)|REC-06, 08, 09|[#8](https://github.com/mayaberries/agora/issues/8)|
|Rotate and revoke keys with timestamps (KEY)|[#36](https://github.com/mayaberries/agora/issues/36)|KEY-03, 04, 05, 06, 07, 08, 12|[#9](https://github.com/mayaberries/agora/issues/9)|
|Record attestations (ATT)|[#37](https://github.com/mayaberries/agora/issues/37)|ATT-01, 02, 03, 06|[#10](https://github.com/mayaberries/agora/issues/10)|
|Model users, publishers, communities and reading groups (ENT)|[#38](https://github.com/mayaberries/agora/issues/38)|ENT-01, 02, 03, 04, 05, 06, 07, 09, 10|[#11](https://github.com/mayaberries/agora/issues/11)|
|Anchor posts to passages and harden threads (THR)|[#39](https://github.com/mayaberries/agora/issues/39)|THR-03, 04, 07, 09, 10, 11, P1|[#12](https://github.com/mayaberries/agora/issues/12)|
|Sync state and keep content available (NET)|[#40](https://github.com/mayaberries/agora/issues/40)|NET-02, 04, 05, 08, 09, 10|[#13](https://github.com/mayaberries/agora/issues/13)|
|Share by QR code and short link (SHR)|[#41](https://github.com/mayaberries/agora/issues/41)|SHR-01, 02, 03, 04|[#15](https://github.com/mayaberries/agora/issues/15)|
|Resist replays, doctored editions and oversized input (ADV)|[#42](https://github.com/mayaberries/agora/issues/42)|ADV-02, 03, 04, 05, 06|[#16](https://github.com/mayaberries/agora/issues/16)|
|Add the nightly stage, OS matrix and coverage gates|[#43](https://github.com/mayaberries/agora/issues/43)|—|[#19](https://github.com/mayaberries/agora/issues/19), [#20](https://github.com/mayaberries/agora/issues/20)|
|Build the first client app|[#44](https://github.com/mayaberries/agora/issues/44)|—|To be created|

## v1.0: Harden for a public release

Complete the trust and privacy features, prove resilience against bad networks and malicious peers, meet performance targets, support other formats, and ship on desktop and mobile.

Parent issue: [#23](https://github.com/mayaberries/agora/issues/23). Corpus: adds C10 ([#3](https://github.com/mayaberries/agora/issues/3)).

Exit criteria:

- Every test ID in [TESTING.md](TESTING.md) is green
- A release candidate meets the exit criteria in [TESTING.md](TESTING.md) §10

|Area|Development issue|Test IDs|Testing issue|
|---|---|---|---|
|Add forked-log detection, threshold signing and social recovery (KEY)|[#45](https://github.com/mayaberries/agora/issues/45)|KEY-09, 10, 11|[#9](https://github.com/mayaberries/agora/issues/9)|
|Weight attestations by reputation (ATT)|[#46](https://github.com/mayaberries/agora/issues/46)|ATT-04, 05, 07|[#10](https://github.com/mayaberries/agora/issues/10)|
|Encrypt private reading groups (ACL)|[#47](https://github.com/mayaberries/agora/issues/47)|ACL-01, 02, 03, 04, 05; ENT-08|[#14](https://github.com/mayaberries/agora/issues/14), [#11](https://github.com/mayaberries/agora/issues/11)|
|Survive packet loss, partitions and relay outages (NET)|[#48](https://github.com/mayaberries/agora/issues/48)|NET-11, 12, 13|[#13](https://github.com/mayaberries/agora/issues/13)|
|Resist floods, Sybil swarms and malformed messages (ADV)|[#49](https://github.com/mayaberries/agora/issues/49)|ADV-01, 07, 08|[#16](https://github.com/mayaberries/agora/issues/16)|
|Add adapters for secondary formats (FMT)|[#50](https://github.com/mayaberries/agora/issues/50)|FMT-01, 02|[#17](https://github.com/mayaberries/agora/issues/17)|
|Meet performance targets at scale|[#51](https://github.com/mayaberries/agora/issues/51)|MRK-11; THR-12|[#18](https://github.com/mayaberries/agora/issues/18), [#5](https://github.com/mayaberries/agora/issues/5), [#12](https://github.com/mayaberries/agora/issues/12)|
|Add the release stage and cross-platform golden checks|[#52](https://github.com/mayaberries/agora/issues/52)|—|[#19](https://github.com/mayaberries/agora/issues/19)|
|Ship the client on desktop and mobile|[#53](https://github.com/mayaberries/agora/issues/53)|—|To be created|

## Open decisions by stage

Each of these blocks the tests that depend on it. They are tracked in the issues listed.

|Stage|Decision|Issue|
|---|---|---|
|POC|Normalization rules for front matter, back matter and footnotes (NRM-04/05)|[#4](https://github.com/mayaberries/agora/issues/4), [#25](https://github.com/mayaberries/agora/issues/25)|
|POC|Whether `agora_core` gets its own `cargo test` stage|[#1](https://github.com/mayaberries/agora/issues/1)|
|MVP|Near-duplicate algorithm and thresholds (FZY-04)|[#7](https://github.com/mayaberries/agora/issues/7), [#34](https://github.com/mayaberries/agora/issues/34)|
|MVP|Policy for unknown record versions (REC-06)|[#8](https://github.com/mayaberries/agora/issues/8), [#35](https://github.com/mayaberries/agora/issues/35)|
|MVP|Marker for the nightly real-timestamp test|[#9](https://github.com/mayaberries/agora/issues/9), [#36](https://github.com/mayaberries/agora/issues/36)|
|MVP|Posts referencing unseen writings (THR-03)|[#12](https://github.com/mayaberries/agora/issues/12), [#39](https://github.com/mayaberries/agora/issues/39)|
|MVP|Client platform, UI framework and app test suite|[#44](https://github.com/mayaberries/agora/issues/44)|
|v1.0|Attestation weight formula (ATT-04/07)|[#10](https://github.com/mayaberries/agora/issues/10), [#46](https://github.com/mayaberries/agora/issues/46)|
|v1.0|Group encryption scheme (ACL-03/04)|[#14](https://github.com/mayaberries/agora/issues/14), [#47](https://github.com/mayaberries/agora/issues/47)|
|v1.0|First secondary formats|[#50](https://github.com/mayaberries/agora/issues/50)|
|v1.0|Performance cases and targets|[#18](https://github.com/mayaberries/agora/issues/18), [#51](https://github.com/mayaberries/agora/issues/51)|
|v1.0|Transparency-log test cases|[#9](https://github.com/mayaberries/agora/issues/9), [#45](https://github.com/mayaberries/agora/issues/45)|
|v1.0|Target platforms for "most devices"|[#53](https://github.com/mayaberries/agora/issues/53)|
|v1.0|Allure report paths, retention and version|[#20](https://github.com/mayaberries/agora/issues/20)|
