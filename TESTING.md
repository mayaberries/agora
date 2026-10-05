# Agora Test Plan

This document describes how Agora is tested: what we test, at which layer, with which tools, and what "done" means for each area. It is meant to evolve with the code. When a feature lands, its section here should be updated in the same PR.

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
|Adversarial|Malicious peers and records, across all layers|varies|varies|`adversarial`|
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

### 4.1 Determinism rules

- **No public infrastructure.** Tests use a local relay and direct addresses or tickets. Default discovery services are disabled.
- **Register the event loop.** Every coroutine that touches Iroh calls `uniffi_set_event_loop` (or the equivalent in `agora_core`) first. This lives in a fixture, never in individual tests.
- **Never `sleep()` and hope.** Replication is eventually consistent, so tests use an `eventually()` helper that polls a condition with a deadline.
- **Fixed keys where it matters.** Identity tests use seeded keypairs so failures are reproducible. Network tests use fresh keys per test to avoid state bleed.
- **Isolated storage.** Each node gets its own `tmp_path` directory.

### 4.2 Shared fixtures (`tests/conftest.py`)

```python
import asyncio
import time
import pytest

import agora_core


@pytest.fixture
async def node(tmp_path):
    """A single Agora node with isolated storage and local-only networking."""
    agora_core.set_event_loop(asyncio.get_running_loop())
    n = await agora_core.Node.spawn(data_dir=tmp_path, relay=LOCAL_RELAY, discovery=False)
    yield n
    await n.shutdown()


@pytest.fixture
async def cluster(tmp_path):
    """Factory for N connected nodes."""
    nodes = []

    async def make(count: int):
        agora_core.set_event_loop(asyncio.get_running_loop())
        for i in range(count):
            nodes.append(await agora_core.Node.spawn(
                data_dir=tmp_path / f"node{i}", relay=LOCAL_RELAY, discovery=False))
        return nodes

    yield make
    for n in nodes:
        await n.shutdown()


async def eventually(check, timeout=10.0, interval=0.05):
    """Poll an async predicate until it returns truthy or the deadline passes."""
    deadline = time.monotonic() + timeout
    while True:
        result = await check()
        if result:
            return result
        if time.monotonic() > deadline:
            raise AssertionError(f"condition not met within {timeout}s")
        await asyncio.sleep(interval)
```

`agora_core.Node` and its parameters are Agora's own API, to be defined in the binding crate.

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

Test IDs are stable. Reference them in issues and PRs (e.g. "fixes MRK-04").

### 6.1 EPUB ingestion and text normalization (`NRM`)

|ID|Case|Expected|
|---|---|---|
|NRM-01|Extract text from C1|Paragraphs in reading order (spine order), no markup|
|NRM-02|C1, C2, C3, C4|Identical normalized text|
|NRM-03|C1 vs C7|Identical normalized text after whitespace, Unicode NFC and hyphenation rules|
|NRM-04|Front matter and back matter|Handled by a documented, versioned rule (included or excluded, never ambiguous)|
|NRM-05|Footnotes and endnotes|Handled by a documented, versioned rule|
|NRM-06|Each C9 file|Clear, typed error; no crash, no partial record|
|NRM-07|Zip bomb in C9|Rejected within a size and time limit|
|NRM-08|Normalization version|Output records the normalizer version; changing rules bumps it|
|NRM-P1|_(property)_ normalize(normalize(x)) == normalize(x)|Idempotent|
|NRM-P2|_(property)_ Whitespace and Unicode-equivalent variants of any text|Same normalized output|

### 6.2 Merkle tree and quote proofs (`MRK`)

|ID|Case|Expected|
|---|---|---|
|MRK-01|Root of C1 is stable across runs and platforms|Golden value in `tests/golden/`|
|MRK-02|C1 and C2/C3/C4/C7|Same root|
|MRK-03|C1 vs C6|Different root|
|MRK-04|Proof for a genuine paragraph of C1|Verifies against C1's root|
|MRK-05|Proof for C6's altered paragraph against C1's root|Fails|
|MRK-06|Valid proof with one sibling hash flipped|Fails|
|MRK-07|Valid proof with wrong paragraph index|Fails|
|MRK-08|Single-paragraph and odd-paragraph-count texts|Correct root; documented padding rule|
|MRK-09|Quote spanning two paragraphs|Proof includes both; verifies|
|MRK-10|Quote that is a substring of a paragraph|Verifies via paragraph proof plus substring check|
|MRK-11|Proof size|O(log n) hashes; asserted for C10|
|MRK-12|Leaf vs internal node hashing|Domain-separated (no second-preimage confusion)|
|MRK-P1|_(property)_ For any text and any paragraph, a generated proof verifies|Always|
|MRK-P2|_(property)_ Any single-byte change to a paragraph|Its proof fails|

### 6.3 Identity hierarchy: Work / Edition / File (`FRB`)

|ID|Case|Expected|
|---|---|---|
|FRB-01|Register C1|Creates Work, Edition (text root), File (BLAKE3/CID) records|
|FRB-02|Register C2 after C1|New File, linked to the existing Edition|
|FRB-03|Register C5 after C1|New Edition, linked to the same Work only via explicit link or accepted suggestion|
|FRB-04|Register C6 after C1|New Edition; not silently merged with C1|
|FRB-05|Metadata extracted automatically from C1|Author, title, language, year populated where present|
|FRB-06|Metadata entered manually|Overrides are stored and attributed to the registering user|
|FRB-07|ISBN validation|Invalid checksums rejected|
|FRB-08|Same file registered twice|Idempotent; no duplicate records|

### 6.4 Near-duplicate detection (`FZY`)

|ID|Case|Expected|
|---|---|---|
|FZY-01|C1 vs C5|Similarity above "same work" threshold|
|FZY-02|C1 vs C6|Flagged as near-identical variant (likely edit)|
|FZY-03|C1 vs C8|Below threshold|
|FZY-04|Threshold values|Documented and covered by tests; changing them requires updating this table|

### 6.5 Records and signatures (`REC`)

Every record type (writing, edition, post, comment, attestation, membership, moderation action) shares these tests via parametrization.

|ID|Case|Expected|
|---|---|---|
|REC-01|Canonical serialization|Same record → same bytes on every platform (golden vectors)|
|REC-02|Record ID|Equals hash of canonical bytes|
|REC-03|Valid signature|Accepted|
|REC-04|Signature by a different key|Rejected|
|REC-05|Any field mutated after signing|Rejected|
|REC-06|Unknown record version|Rejected or quarantined per documented policy|
|REC-07|Missing required fields|Rejected with a typed error|
|REC-08|Oversized fields (title, body)|Rejected at documented limits|
|REC-09|Timestamp far in the future|Rejected beyond documented skew|
|REC-P1|_(property)_ Round-trip serialize → parse → serialize|Identical bytes|

### 6.6 Author identity, keys and rotation (`KEY`)

|ID|Case|Expected|
|---|---|---|
|KEY-01|Create identity|Identity document lists current key and commitment to next key|
|KEY-02|Sign root hash of C8|Signature verifies under the identity|
|KEY-03|Rotate with pre-committed next key|New key valid; old key no longer valid for new signatures|
|KEY-04|Rotate using a key that doesn't match the commitment|Rejected (simulates thief with stolen current key)|
|KEY-05|Signatures made before rotation|Remain valid after rotation|
|KEY-06|Revocation with compromise date|Signatures anchored before the date stay valid; after are rejected|
|KEY-07|Signature with no time anchor after revocation|Treated per documented policy (default: untrusted)|
|KEY-08|Identity event log replay|Rebuilding state from the log gives the same current key set|
|KEY-09|Forked event log (two conflicting rotations)|Detected and surfaced, not silently resolved|
|KEY-10|Threshold signing (2 of 3)|2 shares succeed; 1 share fails|
|KEY-11|Social recovery|Quorum of designated recoverers authorizes a new key; less than quorum fails|
|KEY-12|Publisher/estate/library as root signer|Accepted with its own trust level, distinct from author|

Timestamping (e.g. OpenTimestamps) is **mocked** in unit and integration tests, since real Bitcoin confirmation takes hours. One nightly test verifies a real, pre-made timestamp proof stored in the corpus.

### 6.7 Attestations and reputation (`ATT`)

|ID|Case|Expected|
|---|---|---|
|ATT-01|User attests that C1's root matches their copy|Attestation record links user, edition, root|
|ATT-02|Attestation for a root that doesn't exist|Rejected|
|ATT-03|Duplicate attestation by same user|Idempotent|
|ATT-04|Attestations from brand-new accounts|Carry minimal weight per documented formula|
|ATT-05|1,000 attestations from fresh accounts vs 5 from established ones|Established outweigh (Sybil resistance)|
|ATT-06|Attestation by a revoked key after revocation|Ignored|
|ATT-07|Weight formula|Unit-tested against a table of documented examples|

### 6.8 Entities and relationships (`ENT`)

|ID|Case|Expected|
|---|---|---|
|ENT-01|User creates multiple publishing entities|One-to-many link verified|
|ENT-02|Publishing entity publishes multiple writings|One-to-many link verified|
|ENT-03|Publishing entity operated by a user who is not its owner|Rejected unless delegated|
|ENT-04|Open community about a topic|Anyone can post; no owner field|
|ENT-05|Attempt to claim ownership of an open community|Rejected|
|ENT-06|Reading group: public, open join|Join succeeds without approval|
|ENT-07|Reading group: public, approval required|Join pending until a moderator approves|
|ENT-08|Reading group: private|Non-members can't read content (see `ACL`)|
|ENT-09|Moderator actions (remove post, ban member)|Valid only when signed by a current moderator|
|ENT-10|Moderator actions after moderator is removed|Rejected|

### 6.9 Posts and comment trees (`THR`)

|ID|Case|Expected|
|---|---|---|
|THR-01|Post anchored to a writing|Accepted|
|THR-02|Post with no anchoring writing|Rejected|
|THR-03|Post anchored to a nonexistent writing|Rejected or held until the writing is seen (documented)|
|THR-04|Post anchored to a passage (text quote selector)|Selector resolves in the edition; survives C1→C2/C7 formatting differences|
|THR-05|Comment replying to a post|Parent link verified|
|THR-06|Comment replying to a comment|Parent link verified|
|THR-07|Comment whose parent is in a different thread|Rejected|
|THR-08|Reconstruct a full thread from individual records|Correct tree, correct order|
|THR-09|Records arrive out of order (child before parent)|Tree converges once parent arrives|
|THR-10|Cycles (A replies to B, B replies to A)|Impossible by construction (parent hash must exist first); asserted|
|THR-11|Markdown body|Rendered safely; scripts/HTML injection neutralized|
|THR-12|Deep thread (10,000 levels)|No recursion overflow|
|THR-P1|_(property)_ Any permutation of a thread's records|Reconstructs the same tree|

### 6.10 Networking on Iroh (`NET`)

Integration layer (in-process cluster) unless marked _(network)_.

|ID|Case|Expected|
|---|---|---|
|NET-01|Two nodes connect via ticket|Connection established over local relay|
|NET-02|Two nodes connect via direct address|Connection established without relay|
|NET-03|Transfer C1 via iroh-blobs|Received bytes hash to the same BLAKE3 hash|
|NET-04|Peer serves corrupted bytes|Transfer fails verification; nothing stored|
|NET-05|Resume interrupted transfer of C10|Completes without restarting from zero|
|NET-06|Gossip: post published on node A|Visible on nodes B–E within deadline|
|NET-07|Gossip: invalid record published|Dropped by every receiving node; not re-broadcast|
|NET-08|Docs sync: two nodes write concurrently|Both converge to the same state|
|NET-09|Node offline during writes, then rejoins|Catches up to current state|
|NET-10|Library node (always-on) holds content|New node fetches content while original publisher is offline|
|NET-11|_(network)_ 20 nodes, 5% packet loss|All nodes converge within deadline|
|NET-12|_(network)_ Network partition, then heal|Both sides converge; no lost records|
|NET-13|_(network)_ Relay down|Direct connections continue; relayed ones fail cleanly with typed errors|

### 6.11 Access control for private groups (`ACL`)

|ID|Case|Expected|
|---|---|---|
|ACL-01|Member reads private group content|Succeeds|
|ACL-02|Non-member requests private content|Denied; content not served|
|ACL-03|Non-member obtains raw bytes anyway|Content is encrypted; unreadable|
|ACL-04|Member removed|Cannot read content created after removal (key rotation of group)|
|ACL-05|Removed member|Retains content already received (documented, not a bug)|

### 6.12 Sharing and onboarding (`SHR`)

|ID|Case|Expected|
|---|---|---|
|SHR-01|Ticket → QR → ticket|Round-trips exactly|
|SHR-02|Short link → resolved reference|Resolves to the same writing/community/profile|
|SHR-03|Tampered short link or QR|Rejected; never resolves to a different object|
|SHR-04|Expired or revoked invite|Rejected with a clear message|

### 6.13 Adversarial (`ADV`)

Cross-cutting. These reuse fixtures from other suites with a malicious peer.

|ID|Case|Expected|
|---|---|---|
|ADV-01|Peer floods 10,000 records per second|Rate-limited; honest traffic still flows|
|ADV-02|Replay of a valid old record|Idempotent; no duplicate effects|
|ADV-03|Replay of a moderator action after demotion|Rejected (see ENT-10)|
|ADV-04|Doctored edition (C6) registered before original (C1)|Both exist as separate editions; author signature/timestamp resolves which is authentic|
|ADV-05|Forged author signature using stolen pre-rotation key|Rejected after rotation (see KEY-04/06)|
|ADV-06|Oversized or deeply nested records over the wire|Rejected before full parse|
|ADV-07|Sybil swarm of attesting accounts|Negligible effect on trust score (see ATT-05)|
|ADV-08|Malformed protocol messages (fuzzed)|No crash; connection dropped|

### 6.14 Secondary formats (`FMT`), out of core scope

The core handles EPUB only. Adapters for other open formats (TXT, OEB, etc.) live in a separate module and are tested separately.

|ID|Case|Expected|
|---|---|---|
|FMT-01|Each adapter produces the same normalized-text interface as EPUB ingestion|Shared contract tests pass|
|FMT-02|Core rejects non-EPUB input directly|Typed error pointing to adapter layer|

## 7. Repository layout

```
tests/
├── conftest.py              # node/cluster fixtures, eventually(), markers
├── corpus/
│   ├── SOURCES.md           # provenance and licenses of every file
│   └── make_variants.py     # generates C2–C4, C6, C7
├── golden/                  # golden Merkle roots and serialization vectors
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

Each test function names its ID in the docstring or name (e.g. `test_mrk_05_forged_paragraph_fails`) so failures map back to this document.

## 8. Markers and CI

```ini
# pytest.ini
[pytest]
asyncio_mode = auto
timeout = 60
markers =
    property: Hypothesis property tests
    component: single-node tests
    integration: multi-node, in-process
    network: multi-process, chaos, local relay
    adversarial: malicious-peer scenarios
    perf: performance, nightly only
```

|Stage|Trigger|Command|
|---|---|---|
|Fast|Every push|`pytest -m "not component and not integration and not network and not perf" -n auto`|
|PR|Pull request|`pytest -m "not network and not perf"`|
|Nightly|Schedule|`pytest -m "network or adversarial or perf"`|
|Release|Tag|Full suite, plus golden-vector check across Linux, macOS, Windows|

## 9. Exit criteria

A release candidate is accepted when:

- All unit, property, component and integration tests pass on Linux, macOS and Windows.
- Golden vectors (MRK-01, REC-01) match on every platform.
- No open failures in `KEY`, `REC`, `MRK` or `ADV`. These guard integrity and are never skipped.
- Nightly network suite has passed on the last 3 consecutive runs.
- Line coverage of the `agora` Python package ≥ 85%; validation and crypto modules ≥ 95%.

## 10. Open questions

- Exact normalization rules for front matter, footnotes and hyphenation (NRM-04/05).
- Whether a post can reference a writing the node hasn't seen yet (THR-03).
- Trust weight formula for attestations (ATT-04/07).
- Group encryption scheme for private reading groups (ACL-03/04).
- Policy for unknown record versions (REC-06).

Resolving each question should update the matching test rows.