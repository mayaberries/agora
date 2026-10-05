# Agora Design

The design document: the problem Agora solves, the ideas behind it, and the technical decisions taken so far. For the build plan see [ROADMAP.md](ROADMAP.md).

Agora is the codename for a decentralized application to share books, articles and discussion around them without the need of a central authority to certify publishings. The idea is doing something similar to what p2p networks like torrents or source control tools like github do: a hash as an ebook/document identifier.

It should be a network that allows you to not only share documents but also grow a discussion around them. For example, if people are discussing the same book but different people are discussing different editions or if someone might want to forge a quote inside a book, so we’re capable of certifying the originality of a work in a decentralized way. We're gonna be using **content addressing**, like IPFS, BitTorrent, and Git already use to identify data. 

**Challenge to solve: a hash identifies bytes, not books.** Two EPUBs of the same edition, or the same file with edited metadata or a different cover, produce completely different hashes. If you key discussions on file hashes, conversations about the same book splinter. The usual fix, borrowed from library science (FRBR), is a hierarchy:

- **Work**: _Moby-Dick_ as an abstract thing, identified by an assigned ID or a community-agreed record.
- **Edition**: the 1851 text vs. a modern annotated edition (ISBN, or a hash of the normalized text).
- **File**: the exact bytes (a BLAKE3 hash, the content address Iroh uses).

**For verification, hash the text, not the file.** Extract the text, normalize it (whitespace, Unicode, hyphenation), split it into paragraphs, and build a **Merkle tree** over them. The root hash identifies the text. Anyone can then prove a quote appears in it by showing that one paragraph plus a short chain of hashes, without sharing the whole book, which also helps with copyright. A forged quote simply won't verify against the root.

We're gonna be starting to develop and test with creative commons or public domain books, to make this easy. The idea is by now to allow anyone to load books on the network and keep record that they registered it via this hashing. The user should upload an epub and declare authorship and/or that they published it. If you register an already available book like Moby Dick, we should save on the network the metadata, either via automatically borrowing it from the book or by filling it manually: Author, ISBN and editorial (if it's not exclusively published on the network), year, etc.

On a secondary layer we can allow them to upload other open formats like OEB, MUBI or even TXT but it should be as I mention a different layer, business logic or responsibility of the system. The core should be working specifically with EPUB.

If the author themselves decides to publish their own work, we should give the capability to give it their signature. That way we can protect the publicly available work from forgery. **The challenge is trust, not cryptography.** A hash proves "this matches that." It doesn't prove "that" is the authentic original. Someone could hash a doctored edition first. Decentralized ways to anchor authenticity:

- **Publisher  and/or author signatures** on the root hash, which is the strongest option when available.
- **Timestamping** (e.g. OpenTimestamps on Bitcoin) to prove a version existed at a given date, so earlier versions win disputes.
- **Attestations**: many independent people who own physical or purchased copies sign that a root matches their copy, like a web of trust.
- **Fuzzy hashes** (simhash, minhash) to detect that two texts are near-identical variants and link editions automatically.

**Pieces we build on:** Iroh for the network (see [Technical decisions](#technical-decisions)), OpenTimestamps for anchoring signatures in time, and the W3C Web Annotation standard (used by Hypothesis) for anchoring comments to exact passages with "text quote selectors" that survive formatting changes.

We want to make this network available in most devices and for most audiences, contrary to some tools for torrents and cryptocurrency that require a certain level of expertise to manage it. Thus, we want to create both a network layer and a UI/UX layer that encapsulate all the levels of complexity. The network layer runs on Iroh, which gives us multiplatform support. Then we can build applications on top of it that makes the adoption smooth. We can give different types of capabilities to share works, discussion groups, profiles, etc via different tools like QR or simplified hashes via links. That way we can hide the complexity that makes it hard to adopt technologies like Tor or Crypto (huge tor links or wallet identifiers instead of domains or CLABEs/Card Numbers).

The author’s signature can be a good way to keep verification intact. If you publish, you get to sign the document, and when someone shares it, you get to give that document your good faith signature. In a way someone can still steal the author’s signature, but also, you could create a system that renews them. The author signature is the **root of trust**, the sharers' good-faith signatures are **attestations**, and "renewing" is **key rotation**. 

The hard part is stolen keys:

**Separate identity from keys.** The author shouldn't _be_ a key; they should be an identity that _controls_ keys. Decentralized Identifiers (DIDs) work this way: the author's identity document lists which keys are currently valid, and they can swap keys without losing their identity or reputation.

**Pre-rotation** handles theft well. When you create your current key, you also commit to the hash of your _next_ key, kept offline. If your current key is stolen, the thief can't rotate it because they don't have the pre-committed next key, but you can. KERI (Key Event Receipt Infrastructure) is built around this idea.

**Rotation needs timestamps, or it breaks old signatures.** If a key is revoked, what happens to everything it signed? You don't want the author's whole bibliography to become invalid. The answer is to anchor each signature in time (OpenTimestamps or a public append-only log). Then the rule becomes: signatures made _before_ the compromise date stay valid, and anything signed after is rejected. A thief's forgeries all fall after the revocation, or in a small window you can dispute.

**Transparency logs** make silent forgery hard. If every signature is published to a public, append-only log (like Certificate Transparency for websites), the author can monitor it and immediately spot "I never signed that."

**For extra security:**

- **Threshold signatures**: the author splits signing power across devices or trusted people (e.g. 2 of 3), so one stolen key isn't enough.
- **Social recovery**: if the author loses everything, a group of designated people can jointly authorize a new key.

**Sharer attestations need weight.** A signature from a random new account is worth little, since anyone can create a thousand of them. Attestations are more useful when they come from people with history in the network, or who prove they own a copy, so the "good faith" layer is really a reputation system on top of cryptography.

One edge case: what about dead authors, anonymous authors, or works published before the system existed? There, the publisher, an estate, or a library could act as the signer, which suggests our network should allow multiple kinds of root signers with different trust levels.

In this first stage, we can allow a new user to have their own publisher/library. Eventually we can work towards the trustability of each entity. So, besides being a new user, you can create a new publishing house, book or blog (one user to many publishing entities relationship and one publisher to many published works).

So, when it comes to the relationships of the entities among the network:

- A user. A plain old registered user but traduced to a decentralized network capabilities.
- An author identity that works similar to e-signatures managed by mexican SAT, Cryptocurrency or doc signature platforms.
- A publishing entity: being this a library, an editorial, a blog or a zine.
- Writings: an article, a book, a blog post, etc.
- A community, around a topic, a book, etc. Similar to Reddit, but without capability to claim ownership of the topic.
- A close reading group/subreddit/substack, again, similar to reddit, but here we can claim it as a public or private one, have moderators, ask for permission to join or join openly.
- Posts for these communities. With title, content, author, etc. We can make use of Markdown for better legibility and not reinventing the wheel. Something similar as what Github makes with issues. Also, since this is not made specifically as an isolated social network, the posts must need a writing to anchor to.
- Comments. Same as the posts. These can work as the same entity but nested via it's "am I a comment or a post" and the hierarchy: "where am I in the thread? Who am I responding to? another comment, a post". This way we can create something like a tree where I don't need as a posted entity the whole conversation but if you link all the interlinked entities, you're able to fill a whole comment section.

## Technical decisions

- **Network: Iroh.** Nodes connect directly using tickets; relays only help peers reach each other and never hold authority. We use three Iroh protocols:
  - **iroh-blobs** stores and transfers files, verified by their BLAKE3 hash.
  - **iroh-gossip** spreads new posts, comments and other records to interested peers.
  - **iroh-docs** syncs shared state that several peers write to.
- **Code: Python on top of a Rust core.** The official Iroh Python bindings only cover the basics (endpoints, connections, tickets), not blobs, gossip or docs. A thin Rust crate, `agora_core`, exposes what we need to Python through PyO3/maturin. App logic and validation live in the Python package `agora`.
- **Everything is a signed record.** Writings, editions, posts, comments, attestations, memberships and moderation actions are signed, content-addressed records. Every node validates every record it receives; the network itself enforces no rules.
- **Private reading groups are encrypted.** Only members can read their content, and removed members can't read anything posted after they leave. The encryption scheme is still open.
- **Testing:** Python with pytest, from pure functions up to multi-node networks and malicious peers. See [TESTING.md](TESTING.md), and [ROADMAP.md](ROADMAP.md) for the stages.
