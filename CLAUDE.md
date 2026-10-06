# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

Agora is a decentralized network for sharing books and articles and holding discussions anchored to them. Nothing has a central authority to certify what gets published. The repo is at the **design stage**: `src/`, `tests/` and `data/` exist but are empty. The stack is chosen (Python on a Rust core, see below) but there is no code or build tooling yet. Documentation lives in `docs/`:
- `docs/DESIGN.md`: the design document and the source of truth for intent.
- `docs/ARCHITECTURE.md`: layers, technical decisions, record validation and the domain model.
- `docs/ROADMAP.md`: the POC → MVP → v1.0 stages and their GitHub issues.
- `docs/TESTING.md`: the testing rules. Test cases live in the GitHub testing issues (parent #1).

`README.md` is the public entry point and only summarizes these. When the code lands, add its build/lint/test commands (including how to run a single test) to this file.

The repo root also serves as an Obsidian vault (`.obsidian/`), so Markdown in `docs/` may be authored and linked from Obsidian.

## Core design decisions (from docs/DESIGN.md)

**Identity of content uses a three-level hierarchy (FRBR-style), not raw file hashes.** A file hash identifies bytes, not books, so discussions must not be keyed on file hashes.
- **Work**: the abstract work (an assigned or community-agreed ID).
- **Edition**: a specific text (ISBN, or the hash of the normalized text).
- **File**: the exact bytes (BLAKE3 hash, as used by iroh-blobs).

**Verification hashes the text, not the file.** Pipeline: extract the text from the EPUB → normalize it (whitespace, Unicode, hyphenation) → split it into paragraphs → build a Merkle tree. The root hash identifies the text. A quote is proven with one paragraph plus its Merkle path, which also avoids distributing the full copyrighted text. Fuzzy hashes (simhash/minhash) link near-identical editions.

**EPUB is the core format.** Other formats (TXT, OEB, etc.) belong in a separate ingestion layer or module, not in the core.

**Trust model: cryptography proves matches, and trust anchors authenticity.**
- Root signers: author, publisher, estate, or library, each with a different trust level. Multiple root-signer kinds are needed for dead, anonymous, or pre-existing works.
- Sharers' signatures are *attestations*. They are weighted by reputation or proof of owning a copy, not counted raw (to resist Sybil attacks).
- Identity is separate from keys: DID-style identity documents list the currently valid keys. Key rotation uses KERI-style **pre-rotation** (commit to the hash of the next key).
- Every signature is timestamped (OpenTimestamps or an append-only transparency log). After a key compromise, signatures made before the compromise stay valid and later ones are rejected.
- Optional: threshold signatures and social recovery.

**Stack:**
- **Network: Iroh.** iroh-blobs for file storage and transfer (BLAKE3), iroh-gossip for spreading records, iroh-docs for synced shared state.
- **Code:** Python package `agora` (app logic, validation) on a thin Rust crate `agora_core` that exposes iroh-blobs/gossip/docs via PyO3/maturin, since the official Iroh Python bindings don't cover them.
- **Records:** everything above the network is a signed, content-addressed record that every node validates on receipt; the network enforces no rules.
- **Also used:** W3C Web Annotation text-quote selectors for anchoring comments to passages, OpenTimestamps for timestamping.
- **Tests:** pytest, organized by the layers and test IDs in `docs/TESTING.md`.

**Two layers:** a network layer, and a UI/UX layer that hides cryptographic complexity from non-technical users (QR codes, short links instead of raw hashes or keys).

## Domain model

- **User**: a network participant. It owns many **publishing entities**.
- **Author identity**: signing identity (e-signature-like) that controls rotatable keys.
- **Publishing entity**: a library, editorial, blog, or zine. It has many writings.
- **Writing**: a book, article, or post. It is registered with metadata (author, ISBN, publisher, year), either extracted from the EPUB or entered manually.
- **Community**: an open, topic-based space that nobody owns.
- **Reading group**: a claimable space that can be public or private, with moderators and open or approval-based joining.
- **Post / Comment**: a single entity type with Markdown content. Every post must anchor to a writing. Threads form a tree through parent references: each node knows only its parent (a post or a comment), and the full thread is rebuilt by following links.

Development starts with public-domain / Creative Commons books.

## Workflow: test-driven

Development follows `docs/ROADMAP.md`. Every development issue lists the test IDs it must turn green, and links the testing issue holding their cases.
- Write the failing tests first, named by their IDs (`test_mrk_05_...`), then implement until they pass.
- Never assume an answer to an open question. If a case depends on an undecided rule (a "Blocking decisions" item), stop and ask, or record the decision in the testing issue first.
- Behaviour without a test ID doesn't get built. Propose the new case for the testing issue first.

# Git

The repo has connection to the Github MCP. The aim is to give support with PRs, Issues and so on. Never commit automatically. There's a skill for the way we should open issues in Github to avoid over-explaining and to keep a standardized format

Also, you can give support to navigate the repo via gh CLI, make more higher level changes to the repo:

- **MCP for reading and reviewing.** It returns structured data, so we can rely on MPC for PRs, issues and of the like.
- **`gh` for actions tied to local checkout.**  Everything related to local can be sourced to the gh capabilities.
- Other not mentioned cases: use the ad hoc tool.

At SETUP.md you can find information on how to give support when creating a fresh clone.