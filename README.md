# Cursor @Docs Boundary Audit

**Date:** 2026-05-07
**Author:** Ian Chou
**Cursor version:** _(fill in from screenshot 01)_
**Platform:** Windows
**Audit context:** verification of community-reported @Docs issues against current Cursor; characterization of @Docs's effective boundary in mixed-mode usage.

---

## TL;DR

In current Cursor on Windows, the underlying @Docs **retrieval layer is healthy** — verbatim docs content surfaces, chapter badges appear, type-ahead works. But three layered UX problems combine to make the feature appear unreliable to careful users, which likely explains why forum thread [#141454](https://forum.cursor.com/t/cursor-docs-feature-still-broken/141454) persisted unresolved from 2025-11-06 to its 2026-02-21 auto-close:

1. **Hard A-prefix cap on the picker's default view.** Without typing, the @Docs picker shows only A-prefix built-in docs, scrollable to A's end and no further. Every non-A doc — built-in or user-added — is unreachable until the user starts typing. There is no UI hint that typing is required. This single rendering quirk reproduces `jrista`'s 2025-11-11 #11 complaint ("docs no longer appearing in command palette") under a common interaction pattern.
2. **Convergence with autonomous web fetch.** Across both Agent and Ask modes, Cursor silently fires `Fetch <indexed-doc-URL>` confirmation prompts, retrieving live web content alongside (or instead of) @Docs. For any indexed doc with a public URL, "with @Docs" and "without @Docs" produce substantively identical answers — making @Docs's marginal value invisible to A/B comparison.
3. **Pointing @Docs at a wrong-scope or stale built-in entry can make the answer *worse* than not using @Docs at all.** With network access, autonomous web fetch retrieves the correct Pydantic AI answer even without @Docs attached. But the moment @Docs is pointed at Cursor's built-in Official **Pydantic** entry, the model refuses to answer ("snippet not present") — because that curated index covers the data validation library, not Pydantic AI. Only the user-added **Pydantic AI** doc (indexed 2026-05-05, 166 pages) returns the verbatim answer. The structural pattern is the point: with @Docs attached, the model prioritizes the indexed content and **stops reaching for autonomous web fetch** — so a wrong-scope or stale built-in entry actively replaces the more reliable fallback with either a refusal or a stale answer. This is why users in forum thread #141454 stopped trusting @Docs, and it raises a sharper product question: if autonomous web fetch is often more reliable for public docs than a stale or wrong-scope built-in index, what distinct role should built-in @Docs play?

This audit's recommendations (in [§ Recommended fixes](#recommended-fixes)) address the picker UX, curated-list freshness, and the product-boundary question of how built-in @Docs should handle wrong-scope or stale entries. None require retrieval-engine work.

---

## Why this audit exists

The Cursor community forum thread [Cursor @Docs feature STILL BROKEN](https://forum.cursor.com/t/cursor-docs-feature-still-broken/141454) was opened 2025-11-06 by `jrista` and accumulated 32 posts before Discourse auto-closed it on 2026-02-21 with no resolution from the maintainer team. Cursor staff members `sanjeed5` (post #14, 2025-12-01) and `deanrie` (post #20, 2026-01-10) acknowledged the issue was being tracked, but no fix announcement followed.

Reading the thread carefully reveals a non-trivial pattern:

- 2026-01-18 (Jason Downs): tentatively reports Cursor 2.3.41 may have fixed @Docs, citing the agent's "searching through docs for [X]" tool-call message as evidence.
- 2026-01-18 (liquefy, same day): contradicts in same version — docs are indexed and visible in settings, but the agent does not actually retrieve from them; it "tries looking for an mcp server for some reason but does not find anything."
- 2026-01-19 (post #28, jrista): codifies a falsifiable heuristic in response: *"Don't let the agent fool you... If you do not see those badges [for each indexed 'chapter' of the docs], then @Docs is not working."*

This audit takes the next step the thread did not: **reproduce in current Cursor with deliberate controls**, and articulate the boundary between "@Docs is working" and "@Docs appears to be working but isn't."

---

## Method

### Environment

- Cursor: latest version as of 2026-05-07 _(see screenshot 01)_
- OS: Windows
- Modes tested: Agent mode and Ask mode
- Web access: enabled (default); specific tests conducted with web access **disabled** to isolate @Docs retrieval behavior (noted per query)
- Network: standard residential connection, no VPN

### Fixture: Pydantic AI

Selected as the test target because:
- Recent enough that base model knowledge is incomplete in places (released 2024)
- Has both narrative pages (`/agents`, `/tools`) and API reference pages (`/api/agent/`)
- Crucially: not included in Cursor's Official curated list (which only has the older Pydantic validation library), forcing the user to add it manually — making the built-in staleness finding directly observable.

User-added Pydantic AI doc was indexed via `Settings → Features → Docs → +Add Doc` with URL `https://ai.pydantic.dev/`. Indexing completed at 2026-05-05 13:02 with **166 pages** crawled _(screenshot 03)_, including `pydantic_ai.capabilities`, `pydantic_evals.lifecycle`, and other API reference pages.

### Test queries

Two queries, deliberately chosen to probe different parts of the response space:

**Query A — base-model-easy:**
> *"In Pydantic AI, what is the exact type signature of `RunContext` when an Agent has dependencies of type `MyDeps`? Show the import path, the generic parameter, and quote the exact paragraph from the docs that defines it. Include the source URL of the docs page you used."*

This asks for stable, well-trained API surface. Base model is expected to answer correctly without retrieval.

**Query B — base-model-hard:**
> *"What is the exact default value of the `retries` parameter in the `Agent()` constructor? Quote the exact line from the docs including the parameter name, type annotation, and default value. Include the docs page URL."*

This asks for a specific parameter default buried in a markdown table on `/api/agent/`. Base model can guess, but cannot reliably produce a verbatim quote with URL. This query is the discriminator.

### Heuristic for "is retrieval actually happening"

Per [forum post #28 (jrista, 2026-01-19)](https://forum.cursor.com/t/cursor-docs-feature-still-broken/141454/28):

> *"Don't let the agent fool you... If you do not see those badges, then @Docs is not working."*

Operationalized:
- ✅ A doc badge appears at the top of the chat input (e.g., `📖 Pydantic AI`).
- ✅ The response contains a verbatim quote not derivable from base-model knowledge.
- ✅ A specific source URL is cited matching an indexed page.
- ❌ The agent says "searching docs..." in a tool-call message — by itself, this is not evidence of retrieval. It is the false signal jrista's heuristic was written against.

---

## Findings

### Finding 1 — type-ahead works, but the default picker is hard-capped at A-prefix entries; B–Z docs are unreachable without typing

**Claim from forum thread:** Post #11 (jrista, 2025-11-11) reported "docs no longer appearing in command palette."

**Reproduced in current Cursor:** Partially — and the reproduction is more interesting than the original report suggested.

**The part that works:**
- Typing `@Docs` opens a picker; type-ahead filtering operates correctly.
- Typing `@Docs Pyd` correctly surfaces both **Pydantic** (built-in / Official curated) and **Pydantic AI** (user-added) entries _(screenshot 05)_.
- User-added docs appear in the same picker as built-in curated docs once the user starts typing.

**The part that doesn't:**

When the picker first opens (no typing), it shows Cursor's built-in Official docs in alphabetical order — and **the list is hard-capped at the A-prefix region**. Scrolling within the picker reveals more A-prefix entries (Accord.NET, Active Admin, ActiveRecord, Active Storage, Amazon EC2, ..., AWS Amplify, AWS CLI, ..., Auth0, ...) and then **the list ends**. The picker never transitions to B, C, D, or anything else. To reach a built-in doc starting with any other letter — or any user-added doc — the user must type.

What's missing as a UI affordance:
- No placeholder text indicating "type to search the full doc catalog"
- No "showing N of M docs" counter
- No section header distinguishing built-in (Cursor curated) from user-added entries
- No visual cue that the visible list is a *subset*, not the full catalog

A user opening the picker for the first time, expecting to see their just-added custom doc, sees only A-prefix built-in entries. They scroll to the bottom; still only A-prefix. The reasonable conclusion — "my custom doc was never indexed" or "the picker is broken" — is exactly what `jrista` reported in #11. The retrieval engine is fine; the picker's default render is silently hiding 95% of what the user can actually access.

![Default picker state — only A-prefix built-in docs visible](cursor-docs-audit-2026-05-06/06_at-menu-built-in-docs-only-a-list.png)

![Scrolled to the bottom of the picker — still only A-prefix; the list ends here without ever showing B or beyond](cursor-docs-audit-2026-05-06/08_at-menu-built-in-docs-no-pydantic-visible.png)

After typing `Pyd`, the built-in **Pydantic** entry surfaces — proving the picker *can* reach non-A entries once typing begins; they were not missing, just hidden in the default render:

![Filtered picker — built-in Pydantic surfaces once typing starts](cursor-docs-audit-2026-05-06/05_at-menu-manual-pydantic-visible.png)

**Why this matters for forum thread #141454:** A non-trivial number of "docs not appearing in command palette" reports could be users who never tried typing into the picker. Without type-ahead, *every* non-A-prefix doc — built-in and user-added alike — is unreachable from the default view. The picker's behavior matches jrista's complaint exactly under that interaction pattern.

**Conclusion:** The retrieval layer works. The picker's default state is broken: it shows a misleading subset of the catalog with no signal that more is reachable via typing. This is a render/UX bug with a small fix surface, but it has likely been a meaningful contributor to community confusion about whether @Docs is "working."

### Finding 2 — Autonomous web fetch fires across both Agent and Ask modes, hiding the marginal value of @Docs

**Observation:** Both Agent and Ask modes silently issue a `Fetch https://ai.pydantic.dev/api/agent/` confirmation prompt when the model decides to consult web sources, regardless of whether `@Docs` is attached.

**Evidence:**
- Query A without `@Docs` triggers the fetch confirmation dialog targeting `https://ai.pydantic.dev/api/agent/`, returning content substantively identical to Query A with `@Docs Pydantic AI`.
- Repeating the same query in Ask mode reproduces the autonomous fetch behavior. (User-observed: *"Ask mode 也有 autonomous web fetch."*)
- The fetch dialog offers `Skip / Allowlist '<host>' / Fetch` — meaning the user can in principle deny the fetch, but doing so requires noticing the dialog every time and clicking through it on every turn.

![autonomous fetch confirmation dialog, fired in Ask mode](cursor-docs-audit-2026-05-06/10_fetch-permission-dialog.png)

**Implication:** For any indexed doc that has a public URL accessible without authentication, @Docs and autonomous web fetch are **functional substitutes**. A user comparing "with @Docs" vs "without @Docs" in their daily workflow will, in most queries, observe negligible difference — because the same content is being retrieved either way.

This is the most likely explanation for why forum thread #141454 persisted in confused state: users testing "did the @Docs fix land?" were inadvertently measuring autonomous web fetch's behavior. Both worked; both produced citations; the @Docs-specific code path's contribution was invisible.

But this convergence has a darker corollary documented in Finding 3: when a built-in curated doc is stale or incomplete, @Docs *overrides* the superior autonomous web fetch with inferior indexed content, producing a worse answer than doing nothing. The convergence that hides @Docs's value when it works also amplifies its harm when it doesn't.

**Where @Docs *should* have distinct value (untested in this audit):**
- Internal / private docs with no public URL
- Latency: local vector search vs live HTTP fetch
- Token cost: chunked retrieval vs full-page fetch
- Determinism: indexed snapshot vs live page that may change between queries

**Boundary of this finding:** The audit does not measure latency, token cost, or determinism. The functional-substitute claim applies only to *answer correctness on public-URL docs*.

### Finding 3 — When the built-in entry is wrong-scope or stale, attaching @Docs can be worse than doing nothing

This is the audit's **central finding**, and it is told from the perspective of a *normal Cursor user* — someone who relies on the built-in `@Docs` catalog and would not think to manually add a custom doc index.

**The normal-user path:**

A typical Cursor user wanting docs context for Pydantic AI (the agent framework) types `@Docs Pyd` and sees the built-in **Pydantic** entry. They select it — it is the obvious match. They then ask a question about `Agent(...)` constructor parameters.

A typical user **does not know**, and has no reason to know, that:

- Cursor's Official `Pydantic` entry covers the [Pydantic data validation library](https://docs.pydantic.dev/) — `BaseModel`, `Field`, `validator` — released 2017+.
- [Pydantic AI](https://ai.pydantic.dev/) is a *different* library: an agent framework released 2024 by the same maintainer organization, with no overlapping API surface relevant to this query. It is **not in Cursor's Official curated list at all** (verified 2026-05-07).
- They could in principle add `https://ai.pydantic.dev/` themselves via `+Add Doc`. But knowing to do that requires already knowing the curated list is missing the right library — exactly the kind of pre-knowledge that makes @Docs unnecessary in the first place.

For most users, the built-in Pydantic entry is the only path the @ menu invites them to take. The user-added doc, in this audit, is **not part of the user-experience story** — it is the audit's control, used below only to prove the correct answer is reachable.

**What happens when the normal-user path meets a wrong-scope built-in:**

The same query (Query B: default value of `Agent()`'s `retries` parameter) was run through three configurations. The third row uses the user-added Pydantic AI doc strictly as a control to prove the correct answer exists.

| Configuration | Badge | Response |
|---|---|---|
| Ask mode, no @Docs (web access enabled) | _(none)_ | Correct answer via autonomous web fetch. (This audit's "no @Docs" runs were conducted with web access **disabled** to isolate @Docs behavior; with web enabled, the model autonomously fetches the correct page and answers correctly.) |
| Ask mode, `@Docs` set to **Pydantic** (built-in / Official, validation lib) | `📖 Pydantic` | "I can't quote that exact signature line from the provided docs context in this session, because the snippet containing `Agent(...)` parameters (including `retries`) is not present." |
| Ask mode, `@Docs` set to **Pydantic AI** (user-added, 166 pages indexed) | `📖 Pydantic AI` | "The exact default value is `1`." Followed by verbatim markdown table row from `/api/agent/` quoting `retries`, type annotation `int`, full description, default `1`. Source URL `https://ai.pydantic.dev/api/agent/` cited correctly. |

![Query B without any @Docs — base model hedges, guesses 1](cursor-docs-audit-2026-05-06/11_prompt-no-docs-retries-cannot-quote.png)

![Query B with built-in Pydantic doc selected — refuses, "snippet not present"](cursor-docs-audit-2026-05-06/12_prompt-built-in-docs-cannot-quote.png)

![Query B with user-added Pydantic AI doc selected — verbatim markdown table row from /api/agent/](cursor-docs-audit-2026-05-06/13_prompt-manual-pydantic-ai-success-retries.png)

**The structural inversion:**

With web access enabled — Cursor's default — *not using @Docs at all* produces the best answer: autonomous web fetch retrieves the correct page and the model quotes it verbatim. Attaching the built-in Official `📖 Pydantic` produces the *worst* answer: the model trusts the attached index, finds nothing relevant, and refuses. The user went from "model can find the right answer" to "model can't answer" **by clicking the @Docs button and selecting what looked like the obvious match.** The model prioritizes the indexed content it is explicitly given over its own web fetch, so a wrong-scope or stale built-in entry actively replaces a more reliable fallback with a refusal (or, in the staleness case, with a stale answer).

**Why this likely explains forum thread #141454 better than "@Docs is buggy":**

Users describing @Docs as "broken" were probably observing this exact inversion: they attached @Docs (the obvious path the @ menu invited them to take), got worse answers than they would have gotten by leaving it alone, and concluded @Docs doesn't work. They were right about the experience; the diagnosis was just imprecise. The retrieval engine isn't broken — the problem is that the built-in curated catalog cannot keep up with the rate at which new libraries appear, and when it points at a wrong-scope or stale entry, it actively suppresses a more reliable fallback.

**The product-boundary question:**

If autonomous web fetch routinely outperforms a stale or wrong-scope built-in index for public-URL libraries, what distinct value should built-in @Docs provide? Three scenarios remain where built-in @Docs retains clear value:

1. Private/internal docs with no public URL.
2. Offline or network-restricted environments.
3. Determinism — an indexed snapshot doesn't change between queries (useful for testing or compliance).

But for the most common case — a developer asking about a public library on a connected machine — built-in @Docs's value depends entirely on the curated catalog being both complete and current. Neither holds today: the Pydantic AI gap (an 18-month-old library, still absent from the Official list as of 2026-05-07) is one example among many.

---

## Recommended fixes

All five address the picker UX, curated-list freshness, and the built-in docs' counterproductive overlap with autonomous web fetch. None require retrieval-engine work.

### R1 — Surface the full catalog from the picker's default view

Today, opening the picker without typing shows only A-prefix built-in docs; the list ends without indicating that more docs are reachable via type-ahead. This is the single highest-impact fix because it addresses what users actually experience as "@Docs is broken."

Any one of these would fix it; combinations are better:

- **Placeholder text in the picker:** `"Type to search 200+ built-in docs and your custom additions"` — makes type-ahead discoverable without changing list rendering.
- **Pinned section at top for user-added docs:** They are by definition higher-intent than built-in entries; show them above the alphabetic list.
- **Recently-used docs section:** Surface the last 3–5 docs the user actually invoked, regardless of letter.
- **A "Browse all" entry at the end of the visible A-prefix list:** Click to expand into a full alphabetic list with section letters as anchors.
- **Render a "Showing N of M" counter** at the bottom of the picker, so the user understands the visible list is a subset.

Implementation cost: low (UI-only). Effect on the forum-thread complaint surface: large — most "docs not appearing" reports likely resolve here.

### R2 — Show source metadata next to picker entries

Today, picker entries show name + "Official" tag for curated docs and nothing extra for user-added. Proposed:

| Picker entry | Show |
|---|---|
| `Pydantic` | `Official · Pydantic data validation library · last indexed YYYY-MM-DD` |
| `Pydantic AI` | `Custom (you added) · ai.pydantic.dev · 166 pages · indexed 2026-05-05` |

Implementation cost: low. The metadata is already known internally — only the rendering changes.

Effect on Finding 3: a user who can see "Pydantic data validation library" next to `Pydantic` (Official) will immediately understand it does not cover Pydantic AI's agent API. More importantly, this metadata also exposes when the built-in index was last updated — making its staleness visible.

### R3 — Detect overlap when user adds a doc near a curated entry

When `+Add Doc` is invoked with a URL or name close to an existing Official entry (Levenshtein distance, or shared prefix), prompt:

> _"Cursor already includes an Official doc named **Pydantic**. Your new doc **Pydantic AI** appears to be a different library. Both will appear in the @Docs picker — would you like to add a clarifying tag to distinguish them?"_

This makes the user aware of the collision *at the moment they create it*, rather than discovering it through a confusing retrieval response weeks later.

### R4 — Keep the built-in curated docs list current, and flag wrong-scope or stale entries

The Official curated list does not include Pydantic AI (18 months old, prominent in the agent ecosystem). Any library not yet curated forces the user to discover, add, and correctly select a Custom index — and if they pick a similarly named but wrong-scope built-in entry instead, @Docs can produce *worse* results than doing nothing (Finding 3). This is not just a freshness issue; it is an active harm issue: wrong-scope or stale built-in docs degrade the answer by overriding autonomous web fetch with irrelevant indexed content.

Two options, not mutually exclusive:

1. **Keep the list current.** Add newly prominent libraries to the Official catalog on a regular cadence (monthly at minimum for the fast-moving agent ecosystem). Mark each built-in entry with its last-indexed date so staleness is visible.
2. **Flag wrong-scope or stale entries.** If a built-in doc has not been re-indexed in N days, show a warning in the picker (`⚠️ Last indexed 2024-XX-XX`) or, for entries where the name is likely to collide with a newer library, mark them more explicitly, e.g. `"Pydantic (data validation library)"`, so users don't select them for Pydantic AI / agent-framework queries. In the worst case, remove or de-emphasize entries that are actively misleading.

### R5 — Document the @Docs vs autonomous web fetch decision

In Cursor's docs (and ideally in-product onboarding), add a short explainer:

> *"@Docs uses your indexed snapshot of the doc and is faster, costs fewer tokens, and works offline. Cursor's autonomous web fetch retrieves live pages when needed — useful for content that changes frequently. For most public-URL docs, both produce similar answers; @Docs's distinct value is for private/internal docs, when web is unavailable, or when you want a stable indexed snapshot."*

This single paragraph would have prevented most of forum thread #141454's confusion. Users testing "is @Docs working?" would understand they need to deny autonomous fetch (via the `Skip` button) to actually isolate @Docs's contribution.

---

## What this audit did NOT cover

Honest boundaries:

- **Single fixture (Pydantic AI)**. Findings 1 and 2 likely generalize. Finding 3's *structural pattern* (stale built-in curated index producing worse results than no @Docs) is general; specific behavior on other library pairs was not verified.
- **Single platform (Windows)**. Cursor on macOS / Linux not tested; picker behavior is likely cross-platform but unverified.
- **No latency or token-cost measurement.** Finding 2's "functional substitute" claim is about answer correctness only.
- **Did not test offline / VPN / private-doc scenarios.** These are exactly the scenarios where @Docs *should* have distinct value, and an extension audit could quantify the gap.
- **Did not engage Cursor's MCP layer.** liquefy's [post #29](https://forum.cursor.com/t/cursor-docs-feature-still-broken/141454) on 2026-01-18 observed: *"it tries looking for an mcp server for some reason but does not find anything."* This MCP fallback behavior could not be reproduced without an installed MCP server. It deserves a separate investigation.

---

## Reproduction steps (any reader, ~30 minutes)

1. **Environment check.** Cursor → Help → About; record version.
2. **Add doc.** `Settings → Features → Docs → +Add Doc`. URL: `https://ai.pydantic.dev/`. Wait for index to complete; verify page count > 100.
3. **Type-ahead test.** In a chat, type `@Docs Pyd`. Confirm both `Pydantic` and `Pydantic AI` appear in the picker.
4. **Query B, no @Docs, web access ENABLED (Ask mode).** Run Query B from §Method without attaching any doc, with normal web access. Note response: autonomous web fetch should produce a correct answer. This establishes the baseline — @Docs is *worse than nothing* if the built-in index is stale.
5. **Query B, no @Docs, web access DISABLED (Ask mode).** Run Query B with web access disabled (or deny the Fetch dialog). Note response: should be hedged, producing `1` as a guess without verbatim source. This is the "true no @Docs" baseline used for the audit's screenshots.
6. **Query B with `@Docs Pydantic` (built-in / Official).** Pick the **Pydantic** (Official) entry from the picker. Run Query B. Note response: should refuse with "snippet not present" or equivalent — *worse* than the no-@Docs baseline with web access.
7. **Query B with `@Docs Pydantic AI` (user-added).** Pick the **Pydantic AI** (user-added) entry. Run Query B. Note response: should produce a verbatim markdown table row from `/api/agent/` with `1` as default.
8. **Compare.** The four responses should show: (a) no @Docs + web = correct; (b) no @Docs + no web = hedged guess; (c) built-in @Docs = "snippet not present" (worse than a); (d) user-added @Docs = correct with verbatim source. This is the core inversion: built-in @Docs produces a worse result than not using @Docs at all.

If the four responses do not show the inversion (built-in @Docs worse than no @Docs), this audit's central claim does not reproduce — and @Docs may have been fixed in the intervening Cursor version. That outcome would itself be a useful update.

---

## Appendix: Forum thread #141454 timeline

Reference data for anyone doing a similar audit. All times are UTC unless noted; dates verified against Discourse JSON metadata for the thread.

| Date | Author | Post | Substance |
|---|---|---|---|
| 2025-11-06 | jrista | #1 | Thread opened. Reports @Docs broken across multiple weeks. Speculates: *"I do wonder if this is part of a deeper issue with MCP support in Cursor in general... Is there a bug in how the @Docs context and tooling is exposed?"* |
| 2025-11-09 | MendyLanda | #5 | Confirms "@docs haven't worked for weeks now!" |
| 2025-11-11 | jrista | #11 | Reports docs no longer appearing in command palette. *(Not reproduced in 2026-05-07 audit.)* |
| 2025-12-01 | sanjeed5 (Cursor staff) | #14 | Acknowledges "known issue being tracked." |
| 2026-01-08 | jason-downs | #25 (approx) | "COMPLETELY BROKEN, DOES NOT WORK AT ALL" |
| 2026-01-10 | deanrie (Cursor staff) | #20 | "The issue with @Docs is definitely being tracked by the team." |
| 2026-01-18 | Jason Downs | #27 (approx) | Tentative report: 2.3.41 may have fixed it; cites "searching through docs for [X]" agent message. |
| 2026-01-18 | liquefy | #29 (approx, same day) | Contradicts in same version: indexed and visible in settings, but agent does not retrieve. *"It tries looking for an mcp server for some reason but does not find anything."* |
| 2026-01-19 | jrista | #28 | Codifies the falsifiable heuristic: *"Don't let the agent fool you... If you do not see those badges, then @Docs is not working."* |
| 2026-02-21 | _(last activity)_ | — | Last visible reply before Discourse auto-locked the thread. |
| 22 days later | _(system)_ | — | "This topic was automatically closed 22 days after the last reply." |

The auto-close is a Discourse default behavior, not a maintainer action, and does not indicate the issue was resolved.

---

## Screenshot index

All screenshots are in `cursor-docs-audit-2026-05-06/` adjacent to this file.

| File | Description |
|---|---|
| `00_cursor-version.png` | Cursor → Help → About, version captured |
| `01_add-doc-url-popover.png` | `+Add Doc` URL popover, entering `https://ai.pydantic.dev/` |
| `02_add-doc-dialog-fields.png` | Add-doc dialog field state |
| `03_indexed-list-basic.png` | Settings → Docs showing Pydantic AI entry, indexed 2026-05-05 13:02 |
| `04_indexed-list-page-tooltip.png` | Tooltip detail showing "Indexed 166 pages" + sample page list |
| `05_at-menu-manual-pydantic-visible.png` | `@Docs Pyd` filter — built-in Pydantic surfaces (proves typing reaches non-A entries) |
| `06_at-menu-built-in-docs-only-a-list.png` | Default @ menu state (no typing) — only A-prefix built-in docs |
| `07_at-menu-built-in-docs-still-a-list.png` | Default @ menu after attempting scroll — still A-prefix |
| `08_at-menu-built-in-docs-no-pydantic-visible.png` | Default @ menu — Pydantic / Pydantic AI not in default view |
| `09_prompt-with-docs-success-dependencies.png` | Query A (RunContext) with `📖 Pydantic AI` — successful answer |
| `10_fetch-permission-dialog.png` | Autonomous fetch dialog: `Skip / Allowlist 'ai.pydantic.dev' / Fetch` |
| `11_prompt-no-docs-retries-cannot-quote.png` | Query B (retries), Ask mode, no @Docs — base model hedges, guesses `1` |
| `12_prompt-built-in-docs-cannot-quote.png` | Query B with `📖 Pydantic` (built-in, validation lib) — refuses |
| `13_prompt-manual-pydantic-ai-success-retries.png` | Query B with `📖 Pydantic AI` (user-added) — verbatim table row |

---

*This audit is the artifact of a 90-minute reproduction session on 2026-05-07. It is intended as a starting point for Cursor's DX team; comments and corrections welcome.*
