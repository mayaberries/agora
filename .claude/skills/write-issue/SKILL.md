---
name: write-issue
description: How to write GitHub issues and Jira tickets for this suite concisely. Use whenever drafting, opening, or editing an issue or ticket
---

# Writing issues and tickets

The reader is a teammate who will act on the ticket. Give them what they need to act and nothing else. Build every claim on what is actually in the repo, not on memory.

## Before writing

- Read the code, docs and open issues the ticket touches. Name files, classes by their real name.
- Check existing titles and labels (`gh issue list`, `gh label list`) and reuse them. If no ad hoc label is available, ask to create it.
- Write dates as absolute dates (2026-09-11), never relative ones.

## Title

`Area · what needs to happen`. Match the prefixes already in use: `Docs ·`, `CI ·`, `Coverage ·`, `Backlog ·`. Name a backlog item by its area, never by a ticket id.

## Body

1. **Opening paragraph (2–3 sentences):** what is wrong or missing, and what the ticket covers. If the examples come from one place, name it here.
2. **One `##` section per deliverable**, numbered when there are several. Split a section into `###` subsections only when the request itself lists sub-topics.
3. **Each section or subsection gets at most one paragraph and one bullet list:**
   - The paragraph gives the rule or the reason, stated once.
   - The bullets are concrete: a file, a name or a case, plus a short clause saying why.
   - Bold the lead-in of a bullet when the list works like a lookup table (marker → when to use it).
4. **Code examples only when they settle a question the prose can't.** Take them from real code in the repo, trimmed to 3–8 lines with `...`. Never invent an API.
5. **`## Done when`** is a checklist of verifiable outcomes, one per deliverable.

## Style

- Lead with the rule, then the reason ("Make it a `@property` so it re-resolves on every access, because the board re-renders under the pointer").
- Drop anything that doesn't help someone act: no history, no "this issue aims to", no closing summary.
- Cite the source of truth instead of repeating it (`CLAUDE.md`, `Spec Findings.md`, the docstring).

## Mechanics

- Write the body to a file in the scratchpad and pass it with MCP or if needed `gh issue create --title … --label … --body-file …`. Don't use a heredoc, because backticks get mangled.
- If we later need an upstream repo created and it needs changes, open an **issue, not a PR**: a feature request to review first.
- After opening it, reply with the URL and a few lines on the choices you made. Don't paste the body back.
