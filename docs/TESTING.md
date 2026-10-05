# Agora Test Plan

This document holds the rules for testing Agora: what we test, at which layer, with which tools, and what "done" means for a release. The work to implement it (case lists, starting code and open decisions) is tracked in GitHub issue [#1](https://github.com/mayaberries/agora/issues/1) and its sub-issues. Development follows these tests stage by stage, test-first, as described in [ROADMAP.md](ROADMAP.md). When a rule changes, update this document in the same PR.

---

## 1. Goals

1. **Integrity is provable.** Every claim Agora makes ("this quote is in this edition", "this author signed this work", "this comment replies to that post") is backed by a test that tries to break it.
2. **Tests stay readable.** Tests are written in Python with pytest. Rust is used for the core library only, not for describing behavior.
3. **Networking is deterministic in CI.** Multi-node tests run locally, without public relays or discovery services, and do not flake on timing.
4. **Adversarial cases are first-class.** Forged quotes, bad signatures, stolen keys and malformed records are tested as carefully as the happy path.

## 2. Architecture assumptions

```
┌───────────────────────────────────────────────┐
│ tests/ (pytest)                               │
├───────────────────────────────────────────────┤
│ agora (Python package): app logic, validation │
├───────────────────────────────────────────────┤
│ agora_core (Rust, exposed via PyO3/maturin)   │
│   iroh · iroh-blobs · iroh-gossip · iroh-docs │
└───────────────────────────────────────────────┘
```

- The official `iroh` Python bindings cover the stable core (endpoints, connections, tickets) but **not** iroh-blobs, iroh-gossip or iroh-docs. Agora therefore ships its own thin binding crate, `agora_core`, that exposes the pieces we need to Python.
- Everything above the network is modeled as **signed, content-addressed records**. Every node validates every record it receives; there is no network-enforced rule set.
- Pure logic (normalization, Merkle trees, record validation, comment trees) must be testable **without starting a node**.

## 3. Test layers

|Layer|What it covers|Nodes|Speed target|Marker|
|---|---|---|---|---|
|Unit|Pure functions: normalization, hashing, Merkle proofs, record schemas, signature checks|0|< 10 ms each|_(none)_|
|Property|Invariants over generated input (Hypothesis)|0|< 1 s each|`property`|
|Component|One node: storing blobs, writing/reading docs, publishing records locally|1|< 1 s each|`component`|
|Integration|2–5 nodes in one process: transfer, gossip, sync, validation on receipt|2–5|< 10 s each|`integration`|
|Network|Separate processes, local relay, simulated loss/offline peers|3–20|minutes|`network`|
|Adversarial|Malicious peers and records, across all layers|varies|varies|`adversarial` plus the layer's marker|
|Performance|Large books, many comments, many peers|varies|nightly only|`perf`|

Unit and property tests run on every commit. Component and integration tests run on every PR. Network and performance tests run nightly and before releases.

## 4. Tooling

|Tool|Purpose|
|---|---|
|`pytest`|Runner|
|`pytest-asyncio`|Async tests and fixtures (Iroh APIs are async)|
|`hypothesis`|Property-based tests for normalization, Merkle trees, records|
|`pytest-timeout`|Hard ceiling on every networked test (no hung CI)|
|`pytest-xdist`|Parallel unit/property runs (not for network tests)|
|`iroh-relay` (dev mode)|Local relay server so tests never touch public infrastructure|
|`toxiproxy` or `tc netem`|Packet loss, latency and partitions in network tests|
|`maturin develop`|Builds `agora_core` into the test virtualenv|
|`allure-pytest`|Writes results in Allure format for the combined report (see section 9)|
|Allure 2 CLI|Generates the report in CI (needs Java)|

### 4.1 Determinism rules

- **No public infrastructure.** Tests use a local relay and direct addresses or tickets. Default discovery services are disabled.
- **Register the event loop.** Every coroutine that touches Iroh calls `uniffi_set_event_loop` (or the equivalent in `agora_core`) first. This lives in a fixture, never in individual tests.
- **Never `sleep()` and hope.** Replication is eventually consistent, so tests use an `eventually()` helper that polls a condition with a deadline.
- **Fixed keys where it matters.** Identity tests use seeded keypairs so failures are reproducible. Network tests use fresh keys per test to avoid state bleed.
- **Isolated storage.** Each node gets its own `tmp_path` directory.

### 4.2 Shared fixtures

Shared fixtures live in `tests/conftest.py`, and tests use them instead of starting nodes themselves.

- **`node`:** one Agora node with isolated storage (`tmp_path`), the local relay, and discovery disabled. Shut down after the test.
- **`cluster`:** a factory that starts N such nodes, each in its own subfolder, and shuts them all down afterwards.
- **`eventually(check, timeout, interval)`:** polls an async predicate until it is truthy, or fails with `condition not met within <timeout>s`.
- **Event loop registration** happens inside these fixtures, never in individual tests.

The starting code is in [#2](https://github.com/mayaberries/agora/issues/2).

## 5. Test corpus

Stored in `tests/corpus/`, all public domain or Creative Commons. Every file has a provenance line in `tests/corpus/SOURCES.md`.

|ID|File|Purpose|
|---|---|---|
|C1|Moby-Dick, EPUB A (e.g. Project Gutenberg)|Baseline work|
|C2|C1 with edited OPF metadata only|Same text, different file hash|
|C3|C1 with a different cover image|Same text, different file hash|
|C4|C1 re-packaged (different zip order/compression)|Same text, different file hash|
|C5|Moby-Dick from a second source/edition|Same work, different edition|
|C6|C1 with one paragraph altered|Forged/doctored edition|
|C7|C1 with whitespace, Unicode (NFC/NFD) and soft-hyphen changes|Normalization must absorb these|
|C8|Short CC-licensed original work|"Author publishes their own work" flow|
|C9|Malformed EPUBs (bad zip, missing OPF, invalid XHTML, zip bomb)|Robustness|
|C10|Very large EPUB (e.g. a long multi-volume work)|Performance|

Derived variants (C2–C4, C6, C7) are generated by a script, `tests/corpus/make_variants.py`, so they are reproducible and documented.

## 6. Test suites

Each suite has a three-letter code, and each case an ID built from it: `MRK-05` for example cases, `MRK-P1` for property tests. IDs are stable: never renumber or reuse one, and reference them in issues and PRs (e.g. "fixes MRK-04").

The case list of each suite is tracked in its issue until the tests exist. After that, the test's name and docstring are the record.

|Code|Area|Location under `tests/`|Issue|
|---|---|---|---|
|`NRM`|EPUB ingestion and text normalization|`unit/test_normalize.py`|[#4](https://github.com/mayaberries/agora/issues/4)|
|`MRK`|Merkle tree and quote proofs|`unit/test_merkle.py`|[#5](https://github.com/mayaberries/agora/issues/5)|
|`FRB`|Identity hierarchy: Work / Edition / File|`unit/test_identity.py`|[#6](https://github.com/mayaberries/agora/issues/6)|
|`FZY`|Near-duplicate detection|`unit/test_fuzzy.py`|[#7](https://github.com/mayaberries/agora/issues/7)|
|`REC`|Records and signatures|`unit/test_records.py`|[#8](https://github.com/mayaberries/agora/issues/8)|
|`KEY`|Author identity, keys and rotation|`unit/test_keys.py`|[#9](https://github.com/mayaberries/agora/issues/9)|
|`ATT`|Attestations and reputation|`unit/test_attestations.py`|[#10](https://github.com/mayaberries/agora/issues/10)|
|`ENT`|Entities and relationships|`unit/test_entities.py`|[#11](https://github.com/mayaberries/agora/issues/11)|
|`THR`|Posts and comment trees|`unit/test_threads.py`|[#12](https://github.com/mayaberries/agora/issues/12)|
|`NET`|Networking on Iroh|`integration/test_net.py`, `network/test_net_chaos.py`|[#13](https://github.com/mayaberries/agora/issues/13)|
|`ACL`|Access control for private groups|`integration/test_acl.py`|[#14](https://github.com/mayaberries/agora/issues/14)|
|`SHR`|Sharing and onboarding|`integration/test_sharing.py`|[#15](https://github.com/mayaberries/agora/issues/15)|
|`ADV`|Adversarial|`adversarial/test_adv.py`|[#16](https://github.com/mayaberries/agora/issues/16)|
|`FMT`|Secondary formats (outside the core)|Adapter module|[#17](https://github.com/mayaberries/agora/issues/17)|
|_(to be defined)_|Performance|`perf/test_perf.py`|[#18](https://github.com/mayaberries/agora/issues/18)|

Rules that apply across suites:

- **Records:** every record type (writing, edition, post, comment, attestation, membership, moderation action) runs the shared `REC` cases via parametrization.
- **Timestamping:** OpenTimestamps is mocked in unit and integration tests, since real Bitcoin confirmation takes hours. One nightly test verifies a real, pre-made proof stored in the corpus.
- **Adversarial:** `ADV` tests reuse the fixtures of other suites with a malicious peer.
- **Secondary formats:** the core handles EPUB only. Adapters for other formats live in a separate module and must produce the same normalized-text interface as EPUB ingestion.
- **Open questions** that affect expected results are tracked in the suite's issue, and resolving one updates its rows there.

## 7. Repository layout

```
tests/
├── conftest.py              # node/cluster fixtures, eventually(), markers
├── corpus/
│   ├── SOURCES.md           # provenance and licenses of every file
│   └── make_variants.py     # generates C2–C4, C6, C7
├── golden/                  # golden Merkle roots and serialization vectors
├── allure/
│   └── categories.json      # failure categories for the Allure report
├── unit/
│   ├── test_normalize.py    # NRM
│   ├── test_merkle.py       # MRK
│   ├── test_identity.py     # FRB
│   ├── test_fuzzy.py        # FZY
│   ├── test_records.py      # REC
│   ├── test_keys.py         # KEY
│   ├── test_attestations.py # ATT
│   ├── test_entities.py     # ENT
│   └── test_threads.py      # THR
├── integration/
│   ├── test_net.py          # NET-01..10
│   ├── test_acl.py          # ACL
│   └── test_sharing.py      # SHR
├── network/
│   └── test_net_chaos.py    # NET-11..13
├── adversarial/
│   └── test_adv.py          # ADV
└── perf/
    └── test_perf.py
```

Each test function starts its name with its ID (e.g. `test_mrk_05_forged_paragraph_fails`) so failures map back to the case and its issue.

## 8. Markers and CI

Markers are registered in `pytest.ini`, which runs with `asyncio_mode = auto` and a default `timeout = 60`.

|Marker|Use|
|---|---|
|_(none)_|Unit tests|
|`property`|Hypothesis property tests|
|`component`|Single-node tests|
|`integration`|Multi-node, in-process|
|`network`|Multi-process, chaos, local relay|
|`adversarial`|Malicious-peer scenarios|
|`perf`|Performance, nightly only|

**Every adversarial test also carries its layer marker.** `adversarial` describes intent, not cost, so stage selection relies on the layer marker. A node-free forged-record test is `adversarial` only and runs in the fast stage. A flood test such as ADV-01 is `adversarial` plus `integration`, so it waits for the PR stage. A chaos test is `adversarial` plus `network`, so it runs nightly.

**Timeouts follow the layer.** The 60-second default covers unit to integration tests. A hook in `tests/conftest.py` gives `network` tests 15 minutes and `perf` tests 1 hour. An explicit `@pytest.mark.timeout` on a test always wins.

|Stage|Trigger|Command|
|---|---|---|
|Fast|Every push|`pytest -m "not component and not integration and not network and not perf" -n auto`|
|PR|Pull request|`pytest -m "not network and not perf"`|
|Nightly|Schedule|`pytest -m "network or adversarial or perf"`|
|Release|Tag|Full suite, plus golden-vector check across Linux, macOS, Windows|

The workflows are tracked in [#19](https://github.com/mayaberries/agora/issues/19).

## 9. Reporting

Results from every CI stage and every OS are published as one Allure report on GitHub Pages, with history, so release health (section 10) is checked in one place. `allure-pytest` writes each job's results, and one report job merges them. Implementation details and open decisions are in [#20](https://github.com/mayaberries/agora/issues/20).

Every result is labelled as follows, by an autouse fixture in `tests/conftest.py`:

|Plan concept|Allure field|Example|
|---|---|---|
|Test ID (section 6)|`id`|`MRK-05`|
|Suite code|`suite`|`MRK`|
|Layer marker (section 3)|`parentSuite`|`integration`|
|Platform|parameter `os`|`Linux`|
|CI stage and run|`executor.json`|`Nightly #212`|

The `os` parameter is required. Without it, the same test from Linux, macOS and Windows looks like three attempts at one test, and Allure shows two of them as retries.

|Stage|Report|Why|
|---|---|---|
|Fast|None; results kept as artifacts|Too frequent; would bury nightly trends|
|PR|Generated and attached to the run as an artifact, not deployed|PRs (especially from forks) must not overwrite the published report|
|Nightly|Published|Tracks the "last 3 consecutive runs" rule in section 10|
|Release|Published|Release checkpoint across all three platforms|

Rules for keeping the report useful and safe:

- **History depends on stable names.** Rename a test's description, never its ID.
- **Shared files are written once.** `environment.properties`, `executor.json` and `categories.json` are written by the report job, not by each test job.
- **Failure categories** in `tests/allure/categories.json` separate missed `eventually()` deadlines and hard timeouts from integrity bugs.
- **The report is public.** The corpus is public domain / CC and test keys are throwaway, so this is acceptable. Never log real tokens or `.envrc` contents.
- **Versions are pinned:** `allure-pytest` in the dev dependencies and the Allure CLI in the workflow.

## 10. Exit criteria

A release candidate is accepted when:

- All unit, property, component and integration tests pass on Linux, macOS and Windows.
- Golden vectors (MRK-01, REC-01) match on every platform.
- No open failures in `KEY`, `REC`, `MRK` or `ADV`. These guard integrity and are never skipped.
- Nightly network suite has passed on the last 3 consecutive runs.
- Line coverage of the `agora` Python package ≥ 85%; validation and crypto modules ≥ 95%.
