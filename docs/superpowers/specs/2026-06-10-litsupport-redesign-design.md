# LitSupport Redesign — Design Spec

**Date:** 2026-06-10
**Repo:** Armigedon/litsupport
**Author:** Walter (with Claude)
**Status:** Approved for implementation planning

## Purpose

A no-backend, single-page PubMed lookup tool for clinicians. The user pastes a clinical question or a claim, and the tool returns 8 peer-reviewed cards with the *finding* surfaced front and center. Runs on GitHub Pages. No API key. No install.

## Audience

Clinicians (physicians, PAs, nurses) doing rapid literature lookups between patients or during peer/colleague conversations. Single primary persona; not optimized for academics, librarians, or laypeople.

## Failure Modes Being Fixed

The current production version (commits on 2026-03-25) was abandoned for two specific reasons identified during brainstorming:

1. **Irrelevant results.** The pipeline leaned on a hardcoded 70-item clinical-phrase list and treated NCBI's own MeSH query-translation as a fallback rather than as the primary concept block. Result: queries that didn't match the phrase list got mediocre keyword matches.
2. **Result starvation.** The pipeline required three hard filters simultaneously — PMC open-access full-text (`"pubmed pmc"[sb]`), abstract present (`hasabstract[text]`), and a 10-year date window. Combined, these excluded most relevant evidence and frequently returned fewer than three results.

The redesign drops the PMC filter and the date filter, promotes NCBI's translation to primary, and adds typo correction.

## Interaction Model

### Above the fold

A single hero with three elements and nothing else:

- Headline: **"What do you want to back up?"**
- Subline: *"Paste a question or a claim. Get peer-reviewed evidence."*
- Textarea (autofocus, min 6 chars to enable submit).
- Primary button: **"Find evidence"**.
- Footer line: *"Powered by PubMed · NIH NCBI E-utilities"*.

No mode picker. No stance buttons. No example prompts. No left sidebar.

### Input classifier (invisible, deterministic heuristic)

Run client-side before any network call. No API, no toggle.

```
classifyInput(text) → 'question' | 'claim'

question if any of:
  - text.trim().endsWith('?')
  - /^(does|do|can|could|should|is|are|will|what|which|how|when|why)\b/i.test(text)
  - /^in (patients|adults|children|women|men|the elderly) with\b/i.test(text)
  - /\b(vs|versus|compared (to|with))\b/i.test(text)
otherwise → claim
```

`'question'` mode uses no stance modifier on the PubMed query. `'claim'` mode defaults to a "supporting evidence" stance modifier (see Query Pipeline §4).

### Result list

Eight cards, vertical stack, max-width ~720px column centered on page.

### Per-card structure

- **Title** — purple `#3d1b51`, weight 600, 14px. Links to the PubMed entry in a new tab on click.
- **Byline** — first author *et al.* · journal · year, 12px grey.
- **Finding callout** — pulled from `<AbstractText Label="CONCLUSIONS">` (or fallback, see Query Pipeline §8). Rendered in Georgia 15.5px on a cream background (`#fffbe6`) with a 3px gold left border (`#FFC72C`).
- **Full-text badge** — right-aligned below the callout. Green pill *"Free full text"* if a PMC ID is present in EFetch articleids; muted purple pill *"Abstract only"* otherwise.
- **Swap action** — single button below the card: **"Wrong? Swap →"**. On click, re-queries PubMed with that PMID appended to the existing `excludeIds` `NOT [uid]` clause; the next-ranked result replaces this card in place. The other seven cards remain untouched.

If the swap query returns no new candidate, the button text becomes *"No more matches"* and disables on that card only.

### Counterevidence (claim mode only)

After results render in claim mode, the results-header line reads:

> *Showing supporting evidence · [see counterevidence →]*

Clicking the link re-runs the entire search with the disagree stance modifier (see Query Pipeline §4). The header then shows *"Showing counterevidence · [see supporting evidence →]"* — the toggle is reversible without losing the query.

### Sparse state (<8 results after auto-widen cascade)

Show what we got (1–7 cards), then a single panel below the cards:

> Only **N** results matched.
> Try removing specifics like **"[detected over-narrow term]"** or use simpler wording.

The over-narrow term is selected by picking the longest token from the user's input that is *not* present in the NCBI query-translation's MeSH expansion (i.e., a term NCBI couldn't map to a controlled vocabulary). If no such term exists, the suggestion line is omitted.

## Query Pipeline

Replaces the existing v2 pipeline in `index.html:602-748`.

1. **ESpell typo correction.** `GET espell.fcgi?db=pubmed&term=<input>` on submit. If `<CorrectedQuery>` differs meaningfully from the raw input (Levenshtein > 2 and at least one whole token changed), use the corrected query AND render a small "Showing results for: [corrected]. Search instead for [original]" line above the results. If ESpell fails or returns nothing useful, silently use the raw input. **The corrected query (or raw input if no correction) is the input to every subsequent step.**

2. **ESearch translate.** `GET esearch.fcgi?db=pubmed&retmax=0&retmode=json&term=<spell-corrected-query>` purely to obtain `esearchresult.querytranslation`. This becomes the **primary** concept block (was fallback in v2). Strip any embedded `AND YYYY/MM/DD[pdat]` clauses from the returned translation.

3. **Concept block.** Use the cleaned query-translation as-is. The 70-item hardcoded phrase list from v2 is deleted entirely.

4. **Stance modifier:**
   - Question mode → no stance modifier.
   - Claim mode (default "support") → `AND ("systematic review"[pt] OR "meta-analysis"[pt] OR "review"[pt] OR "clinical trial"[pt])`.
   - Claim mode + counterevidence → `AND ("journal article"[pt] NOT "review"[pt])`.

5. **Final filter clause.** `AND hasabstract[text]` only. **No PMC filter. No date filter.** This is the single biggest behavior change.

6. **ESearch fetch.** `GET esearch.fcgi?db=pubmed&retmax=16&retmode=json&sort=relevance&term=<full-query>`. Request 16 to allow for swap headroom (8 visible + 8 reserve).

7. **Auto-widen cascade.** If the result count is <5:
   - Step 1: drop the stance modifier, retry.
   - Step 2: drop `hasabstract[text]`, retry.
   - If still <5, accept the result count and proceed to render (sparse state will trigger downstream).

8. **EFetch abstracts + ESummary metadata.** Fetch BOTH for all 16 PMIDs returned by step 6, not just the top 8. The extra 8 form a pre-warmed swap reserve so the *"Wrong? Swap →"* gesture resolves instantly without another network round-trip.
   - `GET efetch.fcgi?db=pubmed&rettype=abstract&retmode=xml&id=<pmids>` and `GET esummary.fcgi?db=pubmed&retmode=json&id=<pmids>` run in parallel.
   - Render the top 8 (post-rerank, see step 10). Cache the remaining 8 in memory keyed by PMID.
   - On swap: pull the next-highest-ranked cached PMID, drop it into the swapped card's slot. Only when the cache is empty does swap re-query PubMed (esearch + efetch + esummary for the new candidate, with the cumulative `excludeIds` list).
   - Finding extraction: prefer `<AbstractText Label="CONCLUSIONS">` text. Fallback: `<AbstractText Label="RESULTS">` text. Final fallback: last 1–2 sentences of the concatenated abstract body. If all are empty, mark the article as `noFinding: true` and the card collapses to title + byline only (no callout, no broken visual).

9. **PMC full-text badge.** From the ESummary response in step 8, inspect `articleids` for an entry with `idtype: 'pmc'`; if present, show "Free full text" (linked to `https://www.ncbi.nlm.nih.gov/pmc/articles/PMC<id>/`). Otherwise show "Abstract only" linked to PubMed.

10. **Post-fetch rerank.** Keep PubMed's relevance order as primary, but apply a stable tiebreaker by evidence tier:
    - Tier 1: meta-analysis, systematic review.
    - Tier 2: RCT, clinical trial.
    - Tier 3: cohort study, case-control.
    - Tier 4: everything else.

    Each `<PubmedArticle>` typically carries multiple `<PublicationType>` entries (e.g., a meta-analysis is also tagged "Review" and "Journal Article"). **Use the highest tier present in the list** — i.e., if any tier-1 type appears, the article is tier 1. Stable sort means PubMed's original relevance ordering is preserved within each tier.

## Visual Identity

Keep the existing Ketchum Health purple family but lighter and cleaner.

- **Palette:**
  - `#3d1b51` navy — primary brand (headline accents, title text, primary button).
  - `#4B2E83` lighter purple — hover states, focus rings.
  - `#FFC72C` gold — finding-callout left border.
  - `#fffbe6` cream — finding-callout background.
  - `#fafafa` page background.
  - `#fff` card background.
  - `#1A6B3C` green / `#E8F5EE` green-bg — "Free full text" badge.
  - `#3d1b51` purple / `#f0eaf8` purple-bg — "Abstract only" badge.
- **Typography (no external fonts):**
  - Sans / UI: `system-ui, -apple-system, "Segoe UI", Roboto, Helvetica, Arial, sans-serif`.
  - Serif / reading: `Georgia, "Times New Roman", Cambria, serif`.
  - This eliminates the Google Fonts CDN dependency. Headlines and finding callouts use the serif stack; everything else uses the sans stack.
- **Layout:** single-column, max-width 720px, centered on `#fafafa`. No left sidebar. Mobile: full-width cards, "Swap →" stacks below card body on narrow screens.

## Architecture

### File layout

- `index.html` — single file containing inline `<style>` and `<script>`. No build step. No bundler. No dependencies installed.
- `README.md` — restructured (see §README).
- `_config.yml` — unchanged (`theme: null`).

### Code structure inside `<script>`

Three pure functions extracted to the top of the script block for unit testing:

```js
function classifyInput(text) { /* returns 'question' | 'claim' */ }
function extractFinding(efetchXmlDoc, pmid) { /* returns string | null */ }
function rerankByEvidence(articles) { /* returns articles in stable tier order */ }
```

The rest (UI handlers, fetch orchestration, rendering) sits below these and calls them.

### External dependencies at runtime

Exactly one external host: `eutils.ncbi.nlm.nih.gov`. Nothing else loads. No fonts, no analytics, no telemetry, no service worker.

## Locked-Down Work PC Considerations

Target browser: **Chrome only**. No support burden for IE, Edge legacy, Firefox, Safari.

The tool is a webpage, not an installer, so `.exe` blocks do not apply. The remaining failure modes on a restricted hospital workstation:

| Risk | Mitigation |
|---|---|
| `*.github.io` blocked by URL allowlist | "Save for offline use" escape hatch (see below). |
| `eutils.ncbi.nlm.nih.gov` blocked | None possible — no PubMed tool works without it. NIH is almost always allowlisted in healthcare. |
| `fonts.googleapis.com` blocked | Already handled: design uses system fonts only. |
| JavaScript disabled | Not handled — out of scope for a query tool. |
| Browser is locked to an old version | Chrome auto-updates on managed installs in most environments. The page uses ES5-compatible JS (matches existing code style) with no modern syntax. |

### "Save for offline use" footer note

Add a small line to the footer:

> *Need offline access? Press Ctrl+S to save this page anywhere on your computer. Searches still work as long as your browser has internet.*

Because `index.html` is one self-contained file, saving it locally with `Ctrl+S` produces a working `file:///` deployment. NCBI fetches still succeed from `file://` origins in Chrome. This is the escape hatch for IT shops that block GitHub Pages but allow NIH.

## Error States

| Condition | Behavior |
|---|---|
| NCBI 5xx, network error, CORS failure | Red banner reusing existing `#eb` element: *"PubMed is unreachable. Check your connection and try again."* |
| ESpell returns nothing useful | Silent fallback to raw input; no UI signal. |
| EFetch succeeds but no abstracts parse | Cards render in title-only mode (no finding callout). |
| Input length < 6 chars | Submit button stays disabled (existing `chk()` logic). |
| Swap on a card returns no new candidate | Button text becomes *"No more matches"* and disables on that card only. |
| Final search returns 0 results | Show no cards. Below the hero: *"No PubMed results matched. Try simpler terms or a broader question."* |

## Testing

### Unit tests (inline, dev-only)

A `<script>` block guarded by `if (location.hash === '#test')` runs assertions on the three pure functions and writes pass/fail to `console.log`. Not shipped in production traffic. Test cases minimum:

- `classifyInput`:
  - `"Does aspirin reduce MI?"` → question
  - `"In adults with afib, rivaroxaban vs warfarin"` → question
  - `"Aspirin reduces MI risk in adults over 50."` → claim
  - `"vegetables are healthy"` → claim
- `extractFinding`:
  - Structured abstract with `<AbstractText Label="CONCLUSIONS">` → returns conclusion text.
  - Unstructured abstract → returns last sentence pair.
  - Empty abstract → returns `null`.
- `rerankByEvidence`:
  - Mixed pub-type input → meta-analyses bubble before RCTs which bubble before cohort, with original order preserved within tier.

### Manual playtest checklist

Per the user's *playtest-blind bugs* preference, structural tests do not certify UX. Before declaring done:

1. **Ten real clinical questions** from different domains:
   - Cardiology: *"DOAC vs warfarin in non-valvular afib"*
   - Endocrinology: *"GLP-1 agonists in patients with heart failure"*
   - Infectious disease: *"Sepsis bundle compliance and mortality"*
   - Pediatrics: *"Inhaled corticosteroids in mild persistent asthma in children"*
   - Geriatrics: *"Vitamin D for fall prevention in adults over 70"*
   - Plus five additional domains chosen at test time.
2. **Five claims of varying strength** — at least one strong consensus claim, at least one fringe claim, at least one ambiguous claim. Verify the counterevidence toggle returns substantively different results.
3. **Three typo-heavy queries** — verify ESpell catches them and surfaces the "Showing results for:" line.
4. **Three deliberately over-specific queries** — verify the sparse-state panel triggers and the suggested over-narrow term is sensible.
5. **Mobile Chrome (Android)** — full-width layout, finding callouts readable, swap button accessible.
6. **`file://` deployment** — save the page via Ctrl+S, open the saved file, run two searches successfully.

### Out of scope for testing

No automated end-to-end browser tests. No CI. No NCBI mocking. The tool is small enough and the contract with NCBI stable enough that the playtest checklist is sufficient.

## Deployment

Already deployed: GitHub Pages from `main` / root, served at `https://armigedon.github.io/litsupport/`. No infra change. Push to `main`, GitHub Pages rebuilds in ~60 seconds, refresh the URL.

Single-file architecture means every commit is a working deployment; there are no broken intermediate states by construction.

## README Restructure

The current README is written for *deploying* LitSupport. The new README leads with the end-user surface and pushes the deploy guide below it.

```markdown
# LitSupport

Find peer-reviewed PubMed evidence to back up a clinical question or a claim.
Type, click, read. No login, no install, no data leaves your browser except
the search itself.

→ Open it: https://armigedon.github.io/litsupport/

## How to use it

1. Paste a clinical question ("Does rivaroxaban reduce stroke vs warfarin
   in afib?") or a claim ("GLP-1 agonists improve outcomes in heart failure.").
2. Click **Find evidence**.
3. Skim 8 result cards. Each one shows the actual finding pulled from the
   abstract's conclusion.
4. Click a card to open the full PubMed entry in a new tab.
5. If a card is irrelevant, click **Wrong? Swap →** to replace just that one.

## Need it on a locked-down work computer?

LitSupport is a webpage, not an installer. Open the URL above in Chrome
and you're done. If your network blocks GitHub Pages, save the page locally
(Ctrl+S) and open the saved file — searches still work.

## How searches work

[existing technical paragraph]

## Deploy your own copy

[existing GitHub Pages instructions, moved below]
```

## Out of Scope

These are explicit non-goals, documented so future feature requests don't relitigate the brainstorm:

- **No "Compose response" feature.** Approach A was chosen over Approach C.
- **No persistent shortlist, star toggle, or localStorage state.** The swap-on-wrong gesture is sufficient.
- **No copy-citation button.** Defer until clinicians ask for it.
- **No filter rail or power-user controls.** Defer until the 30-second-lookup model proves insufficient.
- **No accounts, no history, no analytics, no telemetry.** Single-file pure client-side stays that way.
- **No support for non-Chrome browsers.** Document Chrome; do not test Firefox/Safari/Edge.
- **No mobile-app PWA wrapper or service worker.** The save-locally escape hatch is enough.

## Open Questions

None at spec write time. All design decisions resolved during brainstorming.
