# LitSupport Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild LitSupport's `index.html` as a clinician-focused single-screen PubMed lookup tool with the new query pipeline, finding-first cards, swap-on-wrong behavior, and a Chrome-only target.

**Architecture:** Single static `index.html` served from GitHub Pages. Inline `<style>` and `<script>`, no build, no dependencies. Pure functions for the testable logic (`classifyInput`, `extractFinding`, `rerankByEvidence`) live at the top of the script block; an inline test harness gated by `location.hash === '#test'` runs assertions to the console. All PubMed access is direct from the browser via NCBI E-utilities — exactly one external host (`eutils.ncbi.nlm.nih.gov`).

**Tech Stack:** Plain HTML5 + CSS + ES5-style JavaScript (matching the existing file's style). System fonts (Georgia + system-ui stack). Fetch API for NCBI calls. DOMParser for EFetch XML. No npm, no bundler, no framework.

**Spec:** `docs/superpowers/specs/2026-06-10-litsupport-redesign-design.md`

---

## File Structure

| Path | Action | Responsibility |
|---|---|---|
| `index.html` | Replace contents | Entire app — markup, styles, behavior, inline tests |
| `README.md` | Replace contents | User intro first, deploy guide moved below |
| `.gitignore` | Unchanged | Already excludes `.superpowers/` |
| `_config.yml` | Unchanged | `theme: null` stays |
| `docs/` | Unchanged | Spec + plan already committed |

No new files outside `docs/`. The spec mandates single-file architecture; do not add `src/`, `tests/`, or extracted JS files. The inline test harness inside `index.html` is the test suite.

---

## Task 0: Create feature branch

**Files:** none modified yet.

- [ ] **Step 1: Create the feature branch**

```bash
cd /c/Users/wyenk/OneDrive/Documents/GitHub/litsupport
git checkout -b redesign-2026-06
git status
```

Expected: `On branch redesign-2026-06`, working tree clean.

- [ ] **Step 2: Confirm the spec and plan files are present on this branch**

```bash
ls docs/superpowers/specs/2026-06-10-litsupport-redesign-design.md
ls docs/superpowers/plans/2026-06-10-litsupport-redesign.md
```

Expected: both files listed. They exist because they were committed to `main` before the branch was cut.

---

## Task 1: Scaffold new `index.html` with hero + test harness + empty stubs

**Files:**
- Modify: `index.html` (complete replacement)

This task replaces the current ~1018-line file with a clean ~250-line scaffold. The old behavior disappears until later tasks restore it. The branch protects `main` from this interim state.

- [ ] **Step 1: Replace `index.html` with the scaffold**

Write the entire file as shown below. Read the old file first if any inline comments need preserving (none should — the old pipeline is being replaced wholesale).

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<meta name="description" content="Find peer-reviewed PubMed evidence for a clinical question or claim. Powered by NCBI E-utilities.">
<meta name="theme-color" content="#3d1b51">
<title>LitSupport — PubMed Evidence Lookup</title>
<style>
*,*::before,*::after{box-sizing:border-box;margin:0;padding:0}

:root{
  --navy:#3d1b51; --navy-lt:#4B2E83;
  --gold:#FFC72C; --cream:#fffbe6;
  --bg:#fafafa; --card:#fff;
  --text:#1a0e22; --muted:#666; --faint:#999;
  --bd:#e0e0e0; --bd-strong:#d0d0d0;
  --green:#1A6B3C; --green-bg:#E8F5EE;
  --purple-bg:#f0eaf8;
  --red:#B03030; --red-bg:#FDF0F0;
  --ui:system-ui,-apple-system,"Segoe UI",Roboto,Helvetica,Arial,sans-serif;
  --read:Georgia,"Times New Roman",Cambria,serif;
  --r:8px;
}

html,body{height:auto;min-height:100%}
body{
  font-family:var(--ui);
  background:var(--bg);color:var(--text);
  font-size:15px;line-height:1.6;
  padding:2rem 1rem 4rem;
}

.shell{max-width:720px;margin:0 auto}

/* Hero */
.brand{font-size:11px;letter-spacing:.12em;color:var(--muted);text-transform:uppercase;text-align:center;margin-bottom:2rem}
.hero{text-align:center;margin-bottom:1.5rem}
.hero h1{font-family:var(--read);font-size:28px;font-weight:600;color:var(--text);margin-bottom:.4rem}
.hero p{font-size:14px;color:var(--muted);margin-bottom:1.25rem}
.input-wrap{margin-bottom:.75rem}
.input-wrap textarea{
  width:100%;background:var(--card);border:1.5px solid var(--bd-strong);border-radius:6px;
  padding:.85rem 1rem;font-family:var(--ui);font-size:14.5px;color:var(--text);
  min-height:80px;resize:vertical;
}
.input-wrap textarea:focus{outline:none;border-color:var(--navy-lt);box-shadow:0 0 0 3px rgba(75,46,131,.15)}
.go-btn{
  background:var(--navy);color:#fff;border:none;border-radius:6px;
  padding:.7rem 1.6rem;font-family:var(--ui);font-weight:600;font-size:14.5px;
  cursor:pointer;display:block;margin:0 auto;
}
.go-btn:hover:not(:disabled){background:var(--navy-lt)}
.go-btn:disabled{background:#bbb;cursor:not-allowed}
.foot{text-align:center;font-size:11px;color:var(--faint);margin-top:2rem}

/* Status / errors */
.err{background:var(--red-bg);color:var(--red);border:1px solid #e9b0b0;border-radius:6px;padding:.75rem 1rem;margin:1rem 0;font-size:13.5px;display:none}
.err.show{display:block}
.spell-notice{font-size:13px;color:var(--muted);margin:1rem 0;padding:.5rem .75rem;background:#f4f1f9;border-radius:4px;display:none}
.spell-notice.show{display:block}

/* Results */
.results{margin-top:2rem;display:none}
.results.show{display:block}
.results-header{font-size:12.5px;color:var(--muted);margin-bottom:1rem}
.results-header a{color:var(--navy);text-decoration:underline;cursor:pointer}

.card{background:var(--card);border:1px solid var(--bd);border-radius:10px;padding:1.5rem;margin-bottom:.85rem}
.card-title{font-size:13.5px;color:var(--navy);font-weight:600;line-height:1.4;text-decoration:none;display:block}
.card-title:hover{text-decoration:underline}
.card-byline{font-size:11.5px;color:var(--faint);margin-top:.25rem}
.card-finding{
  margin-top:.85rem;padding:.85rem 1rem;background:var(--cream);
  border-left:3px solid var(--gold);border-radius:0 6px 6px 0;
  font-family:var(--read);font-size:15.5px;line-height:1.5;color:var(--text);
}
.card-footer{margin-top:.75rem;display:flex;justify-content:space-between;align-items:center;font-size:11.5px}
.badge{padding:2px 10px;border-radius:10px;font-weight:600;font-size:10.5px}
.badge-free{background:var(--green-bg);color:var(--green)}
.badge-abs{background:var(--purple-bg);color:var(--navy)}
.swap-btn{background:none;border:none;color:var(--muted);cursor:pointer;font-family:var(--ui);font-size:11.5px;text-decoration:underline}
.swap-btn:hover{color:var(--navy)}
.swap-btn:disabled{color:var(--faint);cursor:default;text-decoration:none}

.sparse{margin-top:1.5rem;padding:1rem 1.25rem;background:#f4f1f9;border-radius:6px;font-size:13px;color:var(--muted)}

/* Loading */
.loading{display:none;text-align:center;color:var(--muted);font-size:13px;margin:2rem 0}
.loading.show{display:block}

/* Mobile */
@media (max-width:480px){
  body{padding:1.25rem .75rem 3rem}
  .hero h1{font-size:23px}
  .card{padding:1.25rem}
  .card-finding{font-size:14.5px}
  .card-footer{flex-direction:column;align-items:flex-start;gap:.5rem}
}
</style>
</head>
<body>

<div class="shell">

  <div class="brand">LitSupport</div>

  <div class="hero">
    <h1>What do you want to back up?</h1>
    <p>Paste a question or a claim. Get peer-reviewed evidence.</p>
    <div class="input-wrap">
      <textarea id="q" autofocus placeholder="e.g. Does rivaroxaban reduce stroke vs warfarin in afib?"></textarea>
    </div>
    <button class="go-btn" id="go" disabled>Find evidence</button>
  </div>

  <div class="err" id="err"></div>
  <div class="spell-notice" id="spell"></div>
  <div class="loading" id="loading">Searching PubMed…</div>

  <div class="results" id="results">
    <div class="results-header" id="rhead"></div>
    <div id="cards"></div>
    <div class="sparse" id="sparse" style="display:none"></div>
  </div>

  <div class="foot">Powered by PubMed · NIH NCBI E-utilities</div>

</div>

<script>
/* ============================================================
   LitSupport — clinician PubMed lookup
   Single-file static page. No build, no deps.
   ============================================================ */

var BASE = 'https://eutils.ncbi.nlm.nih.gov/entrez/eutils/';

/* ────────────────────────────────────────────────────────────
   PURE FUNCTIONS (testable via #test hash)
   ──────────────────────────────────────────────────────────── */

function classifyInput(text) {
  /* placeholder — implemented in Task 2 */
  return 'claim';
}

function extractFinding(efetchXmlDoc, pmid) {
  /* placeholder — implemented in Task 3 */
  return null;
}

function rerankByEvidence(articles) {
  /* placeholder — implemented in Task 4 */
  return articles;
}

/* ────────────────────────────────────────────────────────────
   UI HELPERS
   ──────────────────────────────────────────────────────────── */

function $(id) { return document.getElementById(id); }

function esc(s) {
  return String(s == null ? '' : s)
    .replace(/&/g, '&amp;').replace(/</g, '&lt;')
    .replace(/>/g, '&gt;').replace(/"/g, '&quot;');
}

function showErr(msg) {
  var e = $('err');
  e.textContent = msg;
  e.classList.add('show');
}

function clearUi() {
  $('err').classList.remove('show');
  $('spell').classList.remove('show');
  $('results').classList.remove('show');
  $('sparse').style.display = 'none';
}

/* ────────────────────────────────────────────────────────────
   INPUT WIRING
   ──────────────────────────────────────────────────────────── */

$('q').addEventListener('input', function() {
  $('go').disabled = $('q').value.trim().length < 6;
});

$('go').addEventListener('click', function() {
  var q = $('q').value.trim();
  if (q.length < 6) return;
  clearUi();
  $('loading').classList.add('show');
  /* network pipeline wired in Task 5 */
  setTimeout(function() {
    $('loading').classList.remove('show');
    showErr('Search not wired yet — implemented in Task 5.');
  }, 200);
});

/* ────────────────────────────────────────────────────────────
   TEST HARNESS — runs only when URL ends with #test
   ──────────────────────────────────────────────────────────── */

function runTests() {
  var passed = 0, failed = 0;
  function assert(name, cond) {
    if (cond) { console.log('PASS ' + name); passed++; }
    else      { console.error('FAIL ' + name); failed++; }
  }
  function eq(name, actual, expected) {
    assert(name + ' (got: ' + JSON.stringify(actual) + ')', actual === expected);
  }

  console.log('── LitSupport test suite ──');

  /* classifyInput — populated in Task 2 */
  /* extractFinding — populated in Task 3 */
  /* rerankByEvidence — populated in Task 4 */

  console.log('── ' + passed + ' passed, ' + failed + ' failed ──');
}

if (location.hash === '#test') {
  runTests();
}

</script>
</body>
</html>
```

- [ ] **Step 2: Open the page in Chrome and verify it loads cleanly**

Open `file:///C:/Users/wyenk/OneDrive/Documents/GitHub/litsupport/index.html` in Chrome.

Expected:
- "LitSupport" label at top centered.
- "What do you want to back up?" headline in serif.
- Subline below in grey.
- Empty textarea with placeholder text.
- "Find evidence" button below, disabled (greyed out).
- Footer line at bottom.
- No console errors.

- [ ] **Step 3: Type a few characters and verify the button enables**

Type `does it` (7 chars).

Expected: button enables and turns purple `#3d1b51`. Click it: a red banner appears reading *"Search not wired yet — implemented in Task 5."* Reload to dismiss.

- [ ] **Step 4: Open with `#test` hash and verify the harness runs**

Open `file:///C:/Users/wyenk/OneDrive/Documents/GitHub/litsupport/index.html#test`.

Expected in DevTools console:
```
── LitSupport test suite ──
── 0 passed, 0 failed ──
```

No errors. The harness ran with no assertions yet.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Scaffold new index.html with hero, test harness, function stubs"
```

---

## Task 2: TDD `classifyInput`

**Files:**
- Modify: `index.html` — replace the `classifyInput` placeholder and add tests to `runTests`

- [ ] **Step 1: Write failing tests inside `runTests()`**

In `index.html`, find the comment `/* classifyInput — populated in Task 2 */` and replace it with:

```js
  /* classifyInput */
  eq('classifyInput: trailing ?', classifyInput('Does aspirin reduce MI?'), 'question');
  eq('classifyInput: starts with Does', classifyInput('Does aspirin reduce MI'), 'question');
  eq('classifyInput: starts with In patients with', classifyInput('In patients with afib, rivaroxaban vs warfarin'), 'question');
  eq('classifyInput: contains vs', classifyInput('rivaroxaban vs warfarin'), 'question');
  eq('classifyInput: contains versus', classifyInput('rivaroxaban versus warfarin'), 'question');
  eq('classifyInput: starts with What', classifyInput('What is the best statin'), 'question');
  eq('classifyInput: plain claim', classifyInput('Aspirin reduces MI risk in adults over 50.'), 'claim');
  eq('classifyInput: short claim', classifyInput('vegetables are healthy'), 'claim');
  eq('classifyInput: trims whitespace', classifyInput('   Does it work?   '), 'question');
  eq('classifyInput: case insensitive question word', classifyInput('CAN I take this with food'), 'question');
```

- [ ] **Step 2: Run tests and verify they fail**

Open `index.html#test` in Chrome (or reload).

Expected in console:
```
── LitSupport test suite ──
FAIL classifyInput: trailing ? (got: "claim")
FAIL classifyInput: starts with Does (got: "claim")
... etc
── 0 passed, 10 failed ──
```

- [ ] **Step 3: Implement `classifyInput`**

Replace the placeholder body of `classifyInput` with:

```js
function classifyInput(text) {
  var t = String(text || '').trim();
  if (t.length === 0) return 'claim';
  if (t.endsWith('?')) return 'question';
  if (/^(does|do|can|could|should|is|are|will|what|which|how|when|why)\b/i.test(t)) return 'question';
  if (/^in (patients|adults|children|women|men|the elderly) with\b/i.test(t)) return 'question';
  if (/\b(vs|versus|compared (to|with))\b/i.test(t)) return 'question';
  return 'claim';
}
```

- [ ] **Step 4: Reload `#test` and verify all 10 pass**

Expected console:
```
── LitSupport test suite ──
PASS classifyInput: trailing ?
PASS classifyInput: starts with Does
PASS classifyInput: starts with In patients with
PASS classifyInput: contains vs
PASS classifyInput: contains versus
PASS classifyInput: starts with What
PASS classifyInput: plain claim
PASS classifyInput: short claim
PASS classifyInput: trims whitespace
PASS classifyInput: case insensitive question word
── 10 passed, 0 failed ──
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Implement classifyInput with question/claim heuristic"
```

---

## Task 3: TDD `extractFinding`

**Files:**
- Modify: `index.html` — implement `extractFinding`, add tests

The function takes a DOMParser-parsed XML document from EFetch and a PMID, returns the conclusion sentence or `null`.

- [ ] **Step 1: Add a tiny XML test fixture helper to `runTests()`**

Before the `/* classifyInput */` block, add:

```js
  function xml(s) {
    return new DOMParser().parseFromString(s, 'text/xml');
  }
```

- [ ] **Step 2: Add failing tests for `extractFinding`**

Replace the comment `/* extractFinding — populated in Task 3 */` with:

```js
  /* extractFinding */
  var structuredXml = xml(
    '<PubmedArticleSet>' +
      '<PubmedArticle>' +
        '<MedlineCitation>' +
          '<PMID>111</PMID>' +
          '<Article>' +
            '<Abstract>' +
              '<AbstractText Label="BACKGROUND">Background text.</AbstractText>' +
              '<AbstractText Label="METHODS">Methods text.</AbstractText>' +
              '<AbstractText Label="RESULTS">Results text.</AbstractText>' +
              '<AbstractText Label="CONCLUSIONS">Rivaroxaban reduced stroke by 21% versus warfarin.</AbstractText>' +
            '</Abstract>' +
          '</Article>' +
        '</MedlineCitation>' +
      '</PubmedArticle>' +
    '</PubmedArticleSet>'
  );
  eq('extractFinding: prefers CONCLUSIONS',
     extractFinding(structuredXml, '111'),
     'Rivaroxaban reduced stroke by 21% versus warfarin.');

  var resultsOnlyXml = xml(
    '<PubmedArticleSet><PubmedArticle><MedlineCitation>' +
      '<PMID>222</PMID>' +
      '<Article><Abstract>' +
        '<AbstractText Label="RESULTS">No significant difference in major bleeding.</AbstractText>' +
      '</Abstract></Article>' +
    '</MedlineCitation></PubmedArticle></PubmedArticleSet>'
  );
  eq('extractFinding: falls back to RESULTS',
     extractFinding(resultsOnlyXml, '222'),
     'No significant difference in major bleeding.');

  var unstructuredXml = xml(
    '<PubmedArticleSet><PubmedArticle><MedlineCitation>' +
      '<PMID>333</PMID>' +
      '<Article><Abstract>' +
        '<AbstractText>This was a cohort study of 1500 adults. Outcomes were measured at 12 months. Mortality was lower in the treatment arm by 15%.</AbstractText>' +
      '</Abstract></Article>' +
    '</MedlineCitation></PubmedArticle></PubmedArticleSet>'
  );
  var unstructured = extractFinding(unstructuredXml, '333');
  assert('extractFinding: unstructured returns last sentence(s) containing the finding',
    unstructured && unstructured.indexOf('Mortality was lower') !== -1);

  var emptyXml = xml(
    '<PubmedArticleSet><PubmedArticle><MedlineCitation>' +
      '<PMID>444</PMID>' +
      '<Article></Article>' +
    '</MedlineCitation></PubmedArticle></PubmedArticleSet>'
  );
  eq('extractFinding: returns null when no abstract', extractFinding(emptyXml, '444'), null);

  eq('extractFinding: returns null when PMID not in doc', extractFinding(structuredXml, '999'), null);
```

- [ ] **Step 3: Run tests and verify they fail**

Reload `index.html#test`. Expected: 5 new `FAIL extractFinding:` lines.

- [ ] **Step 4: Implement `extractFinding`**

Replace the body of `extractFinding`:

```js
function extractFinding(efetchXmlDoc, pmid) {
  if (!efetchXmlDoc || !pmid) return null;
  var articles = efetchXmlDoc.querySelectorAll('PubmedArticle');
  var target = null;
  for (var i = 0; i < articles.length; i++) {
    var pmidEl = articles[i].querySelector('PMID');
    if (pmidEl && pmidEl.textContent.trim() === String(pmid)) {
      target = articles[i]; break;
    }
  }
  if (!target) return null;

  var abstractTexts = target.querySelectorAll('AbstractText');
  if (abstractTexts.length === 0) return null;

  /* Pass 1: prefer CONCLUSIONS */
  for (var j = 0; j < abstractTexts.length; j++) {
    var lbl = (abstractTexts[j].getAttribute('Label') || '').toUpperCase();
    if (lbl === 'CONCLUSIONS' || lbl === 'CONCLUSION') {
      var txt = abstractTexts[j].textContent.trim();
      if (txt) return txt;
    }
  }
  /* Pass 2: fall back to RESULTS */
  for (var k = 0; k < abstractTexts.length; k++) {
    var lbl2 = (abstractTexts[k].getAttribute('Label') || '').toUpperCase();
    if (lbl2 === 'RESULTS' || lbl2 === 'RESULT' || lbl2 === 'FINDINGS') {
      var txt2 = abstractTexts[k].textContent.trim();
      if (txt2) return txt2;
    }
  }
  /* Pass 3: unstructured — concatenate, take last 1–2 sentences */
  var combined = '';
  for (var m = 0; m < abstractTexts.length; m++) {
    combined += (combined ? ' ' : '') + abstractTexts[m].textContent.trim();
  }
  if (!combined) return null;
  /* Split on . / ! / ? followed by whitespace; keep the last 2 non-empty sentences */
  var sentences = combined.split(/(?<=[.!?])\s+/).filter(function(s) { return s.trim().length > 0; });
  if (sentences.length === 0) return null;
  var lastTwo = sentences.slice(-2).join(' ').trim();
  return lastTwo || null;
}
```

- [ ] **Step 5: Reload `#test` and verify all extractFinding tests pass**

Expected console shows 5 new `PASS extractFinding:` lines and the total passed count rises by 5.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Implement extractFinding with CONCLUSIONS/RESULTS/fallback cascade"
```

---

## Task 4: TDD `rerankByEvidence`

**Files:**
- Modify: `index.html` — implement `rerankByEvidence`, add tests

The function takes an array of article objects (each with a `publicationTypes` string array) and returns the array stable-sorted by evidence tier (lower tier number = higher priority).

- [ ] **Step 1: Add failing tests**

Replace the comment `/* rerankByEvidence — populated in Task 4 */` with:

```js
  /* rerankByEvidence */
  var articles = [
    { pmid: 'A', publicationTypes: ['Journal Article', 'Cohort Studies'] },
    { pmid: 'B', publicationTypes: ['Journal Article', 'Meta-Analysis', 'Review'] },
    { pmid: 'C', publicationTypes: ['Journal Article', 'Randomized Controlled Trial'] },
    { pmid: 'D', publicationTypes: ['Journal Article'] },
    { pmid: 'E', publicationTypes: ['Journal Article', 'Systematic Review'] },
    { pmid: 'F', publicationTypes: ['Journal Article', 'Randomized Controlled Trial'] }
  ];
  var ranked = rerankByEvidence(articles);
  var order = ranked.map(function(a) { return a.pmid; }).join('');
  eq('rerankByEvidence: tier 1 (meta/sys) ahead of tier 2 (RCT) ahead of tier 3 (cohort) ahead of tier 4',
     order, 'BECFAD');

  var stable = rerankByEvidence([
    { pmid: '1', publicationTypes: ['Randomized Controlled Trial'] },
    { pmid: '2', publicationTypes: ['Randomized Controlled Trial'] },
    { pmid: '3', publicationTypes: ['Randomized Controlled Trial'] }
  ]);
  eq('rerankByEvidence: stable within tier',
     stable.map(function(a) { return a.pmid; }).join(''), '123');

  var empty = rerankByEvidence([]);
  assert('rerankByEvidence: empty array OK', empty.length === 0);

  var noTypes = rerankByEvidence([
    { pmid: 'X', publicationTypes: [] },
    { pmid: 'Y', publicationTypes: ['Meta-Analysis'] }
  ]);
  eq('rerankByEvidence: missing publicationTypes treated as tier 4',
     noTypes.map(function(a) { return a.pmid; }).join(''), 'YX');
```

- [ ] **Step 2: Run tests and verify they fail**

Reload `index.html#test`. Expected: 4 new `FAIL rerankByEvidence:` lines (the placeholder returns the array unchanged, so order checks fail).

- [ ] **Step 3: Implement `rerankByEvidence`**

Replace the body of `rerankByEvidence`:

```js
function rerankByEvidence(articles) {
  var TIER = {
    /* Tier 1 — synthesis */
    'meta-analysis': 1, 'systematic review': 1,
    /* Tier 2 — primary controlled */
    'randomized controlled trial': 2, 'clinical trial': 2,
    /* Tier 3 — observational */
    'cohort studies': 3, 'case-control studies': 3,
    'observational study': 3, 'cross-sectional studies': 3
  };
  function tierOf(a) {
    var types = (a.publicationTypes || []).map(function(t) { return String(t).toLowerCase(); });
    var best = 4;
    for (var i = 0; i < types.length; i++) {
      var t = TIER[types[i]];
      if (t && t < best) best = t;
    }
    return best;
  }
  /* Decorate with original index for stable sort */
  var decorated = articles.map(function(a, idx) {
    return { article: a, tier: tierOf(a), idx: idx };
  });
  decorated.sort(function(x, y) {
    if (x.tier !== y.tier) return x.tier - y.tier;
    return x.idx - y.idx;
  });
  return decorated.map(function(d) { return d.article; });
}
```

- [ ] **Step 4: Reload `#test` and verify all rerankByEvidence tests pass**

Expected: 4 new `PASS rerankByEvidence:` lines.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Implement rerankByEvidence with stable tier sort"
```

---

## Task 5: Wire the single-query happy path

**Files:**
- Modify: `index.html` — replace the `setTimeout` stub in the Go button handler with the real pipeline

This task wires ESearch translate → ESearch → EFetch + ESummary in parallel → rerank → render 8 cards. It does NOT yet handle: ESpell, auto-widen, swap, counterevidence, sparse state. Those are dedicated tasks.

- [ ] **Step 1: Add the pipeline helpers**

Above the input-wiring section (just below the UI HELPERS block), add:

```js
/* ────────────────────────────────────────────────────────────
   NCBI PIPELINE
   ──────────────────────────────────────────────────────────── */

var STANCE_SUPPORT = '("systematic review"[pt] OR "meta-analysis"[pt] OR "review"[pt] OR "clinical trial"[pt])';
var STANCE_COUNTER = '("journal article"[pt] NOT "review"[pt])';

function ncbiTranslate(query) {
  /* Returns the NCBI MeSH-mapped concept block, or null if unusable. */
  var url = BASE + 'esearch.fcgi?db=pubmed&retmax=0&retmode=json&usehistory=n&term=' + encodeURIComponent(query);
  return fetch(url)
    .then(function(r) {
      if (!r.ok) throw new Error('NCBI translate failed (HTTP ' + r.status + ')');
      return r.json();
    })
    .then(function(d) {
      var tr = d.esearchresult && d.esearchresult.querytranslation;
      if (!tr || tr.length < 6 || tr.indexOf('[uid]') !== -1) return null;
      /* Strip any embedded date clauses */
      return tr.replace(/\s*AND\s+\d{4}\/\d{2}\/\d{2}\[.*?\]/g, '').trim();
    })
    .catch(function() { return null; });
}

function buildQuery(conceptBlock, mode, counter, excludeIds, includeAbstract) {
  var q = conceptBlock;
  if (mode === 'claim' && !counter) q += ' AND ' + STANCE_SUPPORT;
  else if (mode === 'claim' && counter) q += ' AND ' + STANCE_COUNTER;
  if (includeAbstract) q += ' AND hasabstract[text]';
  if (excludeIds && excludeIds.length) {
    excludeIds.forEach(function(id) { q += ' NOT ' + id + '[uid]'; });
  }
  return q;
}

function ncbiSearch(query) {
  /* Returns array of PMIDs (up to 16). */
  var url = BASE + 'esearch.fcgi?db=pubmed&retmax=16&retmode=json&sort=relevance&term=' + encodeURIComponent(query);
  return fetch(url)
    .then(function(r) {
      if (!r.ok) throw new Error('PubMed search failed (HTTP ' + r.status + ')');
      return r.json();
    })
    .then(function(d) {
      return (d.esearchresult && d.esearchresult.idlist) || [];
    });
}

function ncbiSummary(pmids) {
  if (!pmids.length) return Promise.resolve({});
  var url = BASE + 'esummary.fcgi?db=pubmed&retmode=json&id=' + pmids.join(',');
  return fetch(url)
    .then(function(r) { return r.json(); })
    .then(function(d) {
      var out = {};
      var result = d.result || {};
      (result.uids || pmids).forEach(function(uid) {
        var r = result[uid] || {};
        var authors = 'Unknown authors';
        if (r.authors && r.authors.length) {
          authors = r.authors[0].name + (r.authors.length > 1 ? ' et al.' : '');
        }
        var year = '';
        if (r.pubdate) year = (String(r.pubdate).match(/\d{4}/) || [''])[0];
        var pmcId = null;
        if (r.articleids) {
          for (var i = 0; i < r.articleids.length; i++) {
            if (r.articleids[i].idtype === 'pmc') { pmcId = r.articleids[i].value; break; }
          }
        }
        out[uid] = {
          pmid: String(uid),
          title: r.title || '(Title unavailable)',
          authors: authors,
          journal: r.fulljournalname || r.source || '',
          year: year,
          pmcId: pmcId
        };
      });
      return out;
    });
}

function ncbiFetchAbstracts(pmids) {
  /* Returns { pmid: { finding: string|null, publicationTypes: [string] } } */
  if (!pmids.length) return Promise.resolve({});
  var url = BASE + 'efetch.fcgi?db=pubmed&rettype=abstract&retmode=xml&id=' + pmids.join(',');
  return fetch(url)
    .then(function(r) { return r.text(); })
    .then(function(text) {
      var doc = new DOMParser().parseFromString(text, 'text/xml');
      var out = {};
      var articles = doc.querySelectorAll('PubmedArticle');
      for (var i = 0; i < articles.length; i++) {
        var pmidEl = articles[i].querySelector('PMID');
        if (!pmidEl) continue;
        var pmid = pmidEl.textContent.trim();
        var pubTypes = [];
        var pts = articles[i].querySelectorAll('PublicationType');
        for (var j = 0; j < pts.length; j++) pubTypes.push(pts[j].textContent.trim());
        out[pmid] = {
          finding: extractFinding(doc, pmid),
          publicationTypes: pubTypes
        };
      }
      return out;
    });
}

function pubmedUrl(pmid) { return 'https://pubmed.ncbi.nlm.nih.gov/' + pmid + '/'; }
function pmcUrl(pmcId) { return 'https://www.ncbi.nlm.nih.gov/pmc/articles/' + pmcId + '/'; }

/* ────────────────────────────────────────────────────────────
   RENDER
   ──────────────────────────────────────────────────────────── */

function renderCard(article) {
  var titleUrl = article.pmcId ? pmcUrl(article.pmcId) : pubmedUrl(article.pmid);
  var byline = esc(article.authors);
  if (article.journal) byline += ' &middot; ' + esc(article.journal);
  if (article.year)    byline += ' &middot; ' + esc(article.year);
  var findingHtml = article.finding
    ? '<div class="card-finding">' + esc(article.finding) + '</div>'
    : '';
  var badgeHtml = article.pmcId
    ? '<span class="badge badge-free">Free full text</span>'
    : '<span class="badge badge-abs">Abstract only</span>';
  return '' +
    '<div class="card" data-pmid="' + esc(article.pmid) + '">' +
      '<a class="card-title" href="' + titleUrl + '" target="_blank" rel="noopener">' + esc(article.title) + '</a>' +
      '<div class="card-byline">' + byline + '</div>' +
      findingHtml +
      '<div class="card-footer">' +
        badgeHtml +
        '<button class="swap-btn" data-action="swap" data-pmid="' + esc(article.pmid) + '">Wrong? Swap →</button>' +
      '</div>' +
    '</div>';
}

function renderResults(state) {
  var rhead = '';
  if (state.mode === 'claim') {
    rhead = state.counter
      ? 'Showing counterevidence &middot; <a data-action="toggle-counter">see supporting evidence →</a>'
      : 'Showing supporting evidence &middot; <a data-action="toggle-counter">see counterevidence →</a>';
  } else {
    rhead = 'Showing ' + state.visible.length + ' result' + (state.visible.length === 1 ? '' : 's');
  }
  $('rhead').innerHTML = rhead;
  $('cards').innerHTML = state.visible.map(renderCard).join('');
  $('results').classList.add('show');
}
```

- [ ] **Step 2: Wire the Go button to the pipeline**

Replace the `$('go').addEventListener('click', function() { ... })` block with:

```js
/* Current search state — held in module-level closure for swap/counter access */
var currentState = null;

function runSearch(input, counterevidence) {
  clearUi();
  $('loading').classList.add('show');
  var mode = classifyInput(input);
  var state = {
    input: input,
    mode: mode,
    counter: !!counterevidence,
    excludeIds: [],
    visible: [],   // top 8 articles, rendered
    reserve: []    // articles 9–16, swap pool
  };
  currentState = state;

  return ncbiTranslate(input)
    .then(function(conceptBlock) {
      if (!conceptBlock) throw new Error('NCBI could not interpret your query. Try simpler terms.');
      state.conceptBlock = conceptBlock;
      var query = buildQuery(conceptBlock, mode, state.counter, [], true);
      return ncbiSearch(query);
    })
    .then(function(pmids) {
      state.allPmids = pmids;
      if (pmids.length === 0) {
        $('loading').classList.remove('show');
        showErr('No PubMed results matched. Try simpler terms or a broader question.');
        return null;
      }
      return Promise.all([ ncbiFetchAbstracts(pmids), ncbiSummary(pmids) ])
        .then(function(both) {
          var abstracts = both[0], summaries = both[1];
          var articles = pmids.map(function(pmid) {
            var s = summaries[pmid] || {};
            var a = abstracts[pmid] || {};
            return {
              pmid: pmid,
              title: s.title,
              authors: s.authors,
              journal: s.journal,
              year: s.year,
              pmcId: s.pmcId,
              finding: a.finding || null,
              publicationTypes: a.publicationTypes || []
            };
          });
          articles = rerankByEvidence(articles);
          state.visible = articles.slice(0, 8);
          state.reserve = articles.slice(8);
          $('loading').classList.remove('show');
          renderResults(state);
        });
    })
    .catch(function(err) {
      $('loading').classList.remove('show');
      showErr(err.message || 'PubMed is unreachable. Check your connection and try again.');
    });
}

$('go').addEventListener('click', function() {
  var q = $('q').value.trim();
  if (q.length < 6) return;
  runSearch(q, false);
});
```

- [ ] **Step 3: Open Chrome, run a real query**

Open `file:///C:/Users/wyenk/OneDrive/Documents/GitHub/litsupport/index.html` in Chrome.

Type: `Does rivaroxaban reduce stroke vs warfarin in afib?` → click "Find evidence".

Expected (within ~3 seconds):
- Loading line appears, then disappears.
- 8 result cards render.
- Each card has a title (purple link), authors/journal/year byline, and (most likely) a yellow-bordered finding callout.
- Each card has a "Free full text" or "Abstract only" badge and a "Wrong? Swap →" button (does nothing yet — Task 8).
- Results header reads *"Showing supporting evidence · see counterevidence →"* (because afib question matched `vs` → question mode, no wait — `vs` matches but trailing `?` also matches, both give question mode → header reads "Showing 8 results").

Verify visually: title is small purple bold; finding is large serif cream-bg; full layout is centered, max-width ~720px.

- [ ] **Step 4: Run a claim-mode query**

Reload, type: `Aspirin reduces cardiovascular events in adults over 50.` → click.

Expected: 8 cards. Results header reads *"Showing supporting evidence · see counterevidence →"*. (The link does nothing yet — Task 9 wires it.)

- [ ] **Step 5: Verify it still passes tests**

Open `index.html#test`. Expected: all 19 tests still pass.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Wire single-query happy path: translate, search, fetch, rerank, render"
```

---

## Task 6: Add the auto-widen cascade

**Files:**
- Modify: `index.html` — wrap `ncbiSearch` call in a cascade

The spec says: if a search returns <5 PMIDs, first drop the stance modifier (claim mode only), then drop `hasabstract[text]`, and retry each time. Question mode skips the stance-drop step (no stance to drop) but still does the abstract-drop step.

- [ ] **Step 1: Add the cascade helper**

Below the `buildQuery` function in the NCBI PIPELINE section, add:

```js
function searchWithCascade(state) {
  /* Try full query → drop stance → drop abstract requirement. Return first try that yields ≥5 PMIDs,
     or the last try's PMIDs if all return <5 (sparse state will handle in Task 10). */
  var attempts = [];
  /* Attempt 1: full query */
  attempts.push({
    label: 'full',
    query: buildQuery(state.conceptBlock, state.mode, state.counter, state.excludeIds, true)
  });
  /* Attempt 2: drop stance (claim mode only — question mode has no stance to drop) */
  if (state.mode === 'claim') {
    attempts.push({
      label: 'no-stance',
      query: buildQuery(state.conceptBlock, 'question', false, state.excludeIds, true)
    });
  }
  /* Attempt 3: drop abstract requirement */
  attempts.push({
    label: 'no-abstract',
    query: buildQuery(state.conceptBlock, state.mode, state.counter, state.excludeIds, false)
  });

  function tryNext(i, lastPmids) {
    if (i >= attempts.length) return Promise.resolve(lastPmids);
    return ncbiSearch(attempts[i].query).then(function(pmids) {
      if (pmids.length >= 5 || i === attempts.length - 1) {
        state.widened = (i > 0);
        return pmids;
      }
      return tryNext(i + 1, pmids);
    });
  }
  return tryNext(0, []);
}
```

- [ ] **Step 2: Replace the single `ncbiSearch` call in `runSearch` with `searchWithCascade(state)`**

Find this block inside `runSearch`:

```js
      state.conceptBlock = conceptBlock;
      var query = buildQuery(conceptBlock, mode, state.counter, [], true);
      return ncbiSearch(query);
```

Replace with:

```js
      state.conceptBlock = conceptBlock;
      return searchWithCascade(state);
```

- [ ] **Step 3: Test with a deliberately over-narrow query**

Reload `index.html`. Type a query that should starve the standard pipeline:

`In adults over 90 with end-stage renal failure on hemodialysis, does evolocumab reduce all-cause mortality at 36 months?`

Expected:
- Loading appears.
- After ~5–8 seconds (cascading retries), cards render — likely 5–8 results.
- No "no results" red banner.

The full query is likely to return <5; the no-stance retry might also; the no-abstract retry usually catches enough.

- [ ] **Step 4: Verify the happy path still works**

Type: `metformin in type 2 diabetes`. Expected: 8 cards, no cascade triggered (high-volume topic).

- [ ] **Step 5: Verify the existing tests still pass**

Open `#test`. Expected: 19 pass / 0 fail.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add auto-widen cascade: drop stance, then drop abstract requirement"
```

---

## Task 7: Add ESpell typo correction

**Files:**
- Modify: `index.html` — prepend an ESpell call to the pipeline, wire the notice UI

- [ ] **Step 1: Add the `ncbiSpell` helper**

Above `ncbiTranslate` in the NCBI PIPELINE section, add:

```js
function ncbiSpell(query) {
  /* Returns the corrected query if meaningfully different from input, else null. */
  var url = BASE + 'espell.fcgi?db=pubmed&term=' + encodeURIComponent(query);
  return fetch(url)
    .then(function(r) {
      if (!r.ok) return null;
      return r.text();
    })
    .then(function(text) {
      if (!text) return null;
      var doc = new DOMParser().parseFromString(text, 'text/xml');
      var corrected = doc.querySelector('CorrectedQuery');
      if (!corrected) return null;
      var c = corrected.textContent.trim();
      if (!c) return null;
      if (c.toLowerCase() === query.toLowerCase()) return null;
      /* Require at least one whole-token change to avoid trivial casing tweaks */
      var qTokens = query.toLowerCase().split(/\s+/);
      var cTokens = c.toLowerCase().split(/\s+/);
      var changed = false;
      for (var i = 0; i < cTokens.length; i++) {
        if (qTokens.indexOf(cTokens[i]) === -1) { changed = true; break; }
      }
      return changed ? c : null;
    })
    .catch(function() { return null; });
}
```

- [ ] **Step 2: Wire ESpell into `runSearch`**

Modify `runSearch` to call ESpell first. Find:

```js
function runSearch(input, counterevidence) {
  clearUi();
  $('loading').classList.add('show');
  var mode = classifyInput(input);
```

Replace the whole signature and start with:

```js
function runSearch(input, counterevidence, alreadyCorrected) {
  clearUi();
  $('loading').classList.add('show');
  var mode = classifyInput(input);
```

Then find this line near the top of the search chain:

```js
  return ncbiTranslate(input)
```

Replace with:

```js
  var spellPromise = alreadyCorrected ? Promise.resolve(null) : ncbiSpell(input);

  return spellPromise.then(function(corrected) {
    if (corrected) {
      showSpellNotice(corrected, input);
      state.input = corrected;
      return ncbiTranslate(corrected);
    }
    return ncbiTranslate(input);
  })
```

- [ ] **Step 3: Add the spell-notice UI helper**

Above `clearUi` in UI HELPERS, add:

```js
function showSpellNotice(corrected, original) {
  var html =
    'Showing results for: <strong>' + esc(corrected) + '</strong>. ' +
    '<a data-action="search-original">Search instead for <em>' + esc(original) + '</em></a>';
  var el = $('spell');
  el.innerHTML = html;
  el.classList.add('show');
}
```

- [ ] **Step 4: Wire the "search original" link**

At the bottom of the script (just above `if (location.hash === '#test')`), add a delegated click handler for the results header / spell notice / swap buttons (we'll extend this in Tasks 8–9):

```js
document.addEventListener('click', function(ev) {
  var t = ev.target;
  var action = t.getAttribute('data-action');
  if (!action) return;
  if (action === 'search-original') {
    var notice = $('spell');
    var em = notice.querySelector('em');
    if (em) runSearch(em.textContent, currentState && currentState.counter, true);
  }
});
```

- [ ] **Step 5: Test with a typo'd query**

Reload `index.html`. Type: `ribaroxaban vs warferin in afib` (two misspellings).

Expected:
- Cards render.
- Above the cards: a small grey notice reading *"Showing results for: **rivaroxaban vs warfarin in afib**. Search instead for **ribaroxaban vs warferin in afib**"*.
- Clicking the "Search instead for" link re-runs with the original spelling and skips ESpell (no infinite loop).

- [ ] **Step 6: Test with a correctly-spelled query (notice should NOT appear)**

Type: `aspirin in primary prevention`. Expected: no spell notice. Just cards.

- [ ] **Step 7: Verify tests still pass**

Open `#test`. Expected: 19 pass.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Add ESpell typo correction with 'Showing results for' notice"
```

---

## Task 8: Wire the swap button

**Files:**
- Modify: `index.html` — extend the delegated click handler, add `swapCard`

The "Wrong? Swap →" button consumes from the reserve cache first; only when the reserve is empty does it re-query PubMed with the cumulative `excludeIds` list.

- [ ] **Step 1: Add the `swapCard` function**

Below `renderResults` in the RENDER section, add:

```js
function swapCard(pmid, btn) {
  var state = currentState;
  if (!state) return;
  /* Find index of card to replace */
  var idx = -1;
  for (var i = 0; i < state.visible.length; i++) {
    if (state.visible[i].pmid === pmid) { idx = i; break; }
  }
  if (idx === -1) return;

  /* Track exclusion so re-queries don't re-suggest it */
  state.excludeIds.push(pmid);

  /* Cache-first */
  if (state.reserve.length > 0) {
    var replacement = state.reserve.shift();
    state.visible.splice(idx, 1, replacement);
    renderResults(state);
    return;
  }

  /* Reserve empty — re-query with cumulative excludeIds */
  if (btn) { btn.disabled = true; btn.textContent = 'Searching…'; }
  searchWithCascade(state).then(function(pmids) {
    /* Filter out PMIDs we're already showing */
    var currentPmids = state.visible.map(function(a) { return a.pmid; });
    pmids = pmids.filter(function(p) { return currentPmids.indexOf(p) === -1; });
    if (pmids.length === 0) {
      if (btn) { btn.textContent = 'No more matches'; btn.disabled = true; }
      return;
    }
    /* Fetch metadata for just the new PMIDs */
    return Promise.all([ ncbiFetchAbstracts(pmids), ncbiSummary(pmids) ]).then(function(both) {
      var abstracts = both[0], summaries = both[1];
      var newArticles = pmids.map(function(p) {
        var s = summaries[p] || {};
        var a = abstracts[p] || {};
        return {
          pmid: p, title: s.title, authors: s.authors, journal: s.journal, year: s.year,
          pmcId: s.pmcId, finding: a.finding || null, publicationTypes: a.publicationTypes || []
        };
      });
      newArticles = rerankByEvidence(newArticles);
      var replacement2 = newArticles.shift();
      state.reserve = newArticles;  // any leftover seeds the new reserve
      state.visible.splice(idx, 1, replacement2);
      renderResults(state);
    });
  }).catch(function() {
    if (btn) { btn.textContent = 'Search failed'; btn.disabled = true; }
  });
}
```

- [ ] **Step 2: Extend the delegated click handler**

Modify the click handler from Task 7 to also handle swap:

```js
document.addEventListener('click', function(ev) {
  var t = ev.target;
  var action = t.getAttribute('data-action');
  if (!action) return;
  if (action === 'search-original') {
    var notice = $('spell');
    var em = notice.querySelector('em');
    if (em) runSearch(em.textContent, currentState && currentState.counter, true);
  } else if (action === 'swap') {
    var pmid = t.getAttribute('data-pmid');
    if (pmid) swapCard(pmid, t);
  }
});
```

- [ ] **Step 3: Test swap from the reserve**

Reload, search `metformin in type 2 diabetes` (high-volume → full 16 PMIDs cached).

Click "Wrong? Swap →" on one card. Expected: that card silently and instantly replaces with a new one. The other 7 cards do not change.

Click "Wrong? Swap →" on 8 different cards (one each). Expected: 8 swaps work instantly. On the 9th swap, the network call kicks in (~2 seconds).

- [ ] **Step 4: Test swap exhaustion on a low-volume query**

Search: `In adults over 95 with stage 5 chronic kidney disease, does PCSK9 inhibition reduce major adverse cardiac events at 24 months?` Sparse query.

If cascade triggers, fewer PMIDs come back. After clicking swap a few times, expect to see "No more matches" on a swap button.

- [ ] **Step 5: Verify tests still pass**

`#test`: 19 pass.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Wire per-card swap with cache-first + re-query fallback"
```

---

## Task 9: Wire the counterevidence toggle

**Files:**
- Modify: `index.html` — extend the click handler

In claim mode, the results-header link "see counterevidence →" / "see supporting evidence →" re-runs the search with the stance modifier flipped.

- [ ] **Step 1: Extend the delegated click handler**

Add a `toggle-counter` branch to the click handler:

```js
document.addEventListener('click', function(ev) {
  var t = ev.target;
  var action = t.getAttribute('data-action');
  if (!action) return;
  if (action === 'search-original') {
    var notice = $('spell');
    var em = notice.querySelector('em');
    if (em) runSearch(em.textContent, currentState && currentState.counter, true);
  } else if (action === 'swap') {
    var pmid = t.getAttribute('data-pmid');
    if (pmid) swapCard(pmid, t);
  } else if (action === 'toggle-counter') {
    if (currentState) {
      runSearch(currentState.input, !currentState.counter, true);
    }
  }
});
```

(Note: pass `true` for `alreadyCorrected` so the toggle doesn't re-run ESpell on the corrected query.)

- [ ] **Step 2: Test the toggle**

Reload, type a claim: `Statins cause cognitive decline.` → click "Find evidence".

Expected: 8 cards (likely review-tier, since stance modifier = STANCE_SUPPORT prefers reviews). Header: *"Showing supporting evidence · see counterevidence →"*.

Click "see counterevidence →".

Expected: page re-runs, new 8 cards (likely RCTs/primary studies because counter modifier = STANCE_COUNTER excludes reviews). Header now reads *"Showing counterevidence · see supporting evidence →"*.

Click again — flips back.

- [ ] **Step 3: Verify question mode doesn't show the toggle**

Type: `Do statins cause cognitive decline?` → click. Expected: header reads *"Showing 8 results"* (no toggle link).

- [ ] **Step 4: Verify tests still pass**

`#test`: 19 pass.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Wire counterevidence toggle for claim-mode results header"
```

---

## Task 10: Add the sparse state panel

**Files:**
- Modify: `index.html` — extend `renderResults` to show the sparse hint when fewer than 8 cards rendered

- [ ] **Step 1: Add the over-narrow term detector**

Below `searchWithCascade` in the NCBI PIPELINE section, add:

```js
function detectOverNarrowTerm(input, conceptBlock) {
  /* Find the longest word in the user's input (>=4 chars) that does NOT appear in the
     NCBI query-translation. That term is the most likely culprit for narrowing the search. */
  if (!input || !conceptBlock) return null;
  var lowerBlock = conceptBlock.toLowerCase();
  var tokens = input.toLowerCase().split(/[^a-z0-9]+/).filter(function(w) {
    return w.length >= 4;
  });
  /* Sort longest first */
  tokens.sort(function(a, b) { return b.length - a.length; });
  for (var i = 0; i < tokens.length; i++) {
    if (lowerBlock.indexOf(tokens[i]) === -1) {
      return tokens[i];
    }
  }
  return null;
}
```

- [ ] **Step 2: Extend `renderResults` to show the sparse panel**

Modify `renderResults` to detect <8 results and populate the sparse panel:

```js
function renderResults(state) {
  var rhead = '';
  if (state.mode === 'claim') {
    rhead = state.counter
      ? 'Showing counterevidence &middot; <a data-action="toggle-counter">see supporting evidence →</a>'
      : 'Showing supporting evidence &middot; <a data-action="toggle-counter">see counterevidence →</a>';
  } else {
    rhead = 'Showing ' + state.visible.length + ' result' + (state.visible.length === 1 ? '' : 's');
  }
  $('rhead').innerHTML = rhead;
  $('cards').innerHTML = state.visible.map(renderCard).join('');

  var sparse = $('sparse');
  if (state.visible.length < 8) {
    var overNarrow = detectOverNarrowTerm(state.input, state.conceptBlock);
    var hint = 'Only ' + state.visible.length + ' result' + (state.visible.length === 1 ? '' : 's') + ' matched.';
    if (overNarrow) {
      hint += ' Try removing specifics like <strong>"' + esc(overNarrow) + '"</strong> or use simpler wording.';
    } else {
      hint += ' Try simpler wording or a broader question.';
    }
    sparse.innerHTML = hint;
    sparse.style.display = 'block';
  } else {
    sparse.style.display = 'none';
  }

  $('results').classList.add('show');
}
```

- [ ] **Step 3: Test sparse state**

Reload, type: `In Cuban-American men over 75 with chronic lymphocytic leukemia, does ibrutinib improve overall survival vs venetoclax at 5 years?`

Expected: < 8 cards (cascade triggers). Below the cards: a grey panel reading something like *"Only 4 results matched. Try removing specifics like 'Cuban-American' or use simpler wording."* (over-narrow term may vary.)

- [ ] **Step 4: Test the happy path doesn't show the panel**

Type: `metformin in type 2 diabetes`. Expected: 8 cards, no sparse panel.

- [ ] **Step 5: Verify tests still pass**

`#test`: 19 pass.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Show sparse-state panel with over-narrow term suggestion"
```

---

## Task 11: Final CSS polish pass

**Files:**
- Modify: `index.html` — refine spacing, hover states, mobile

The CSS already covers the basics from Task 1. This task is a deliberate visual review against the spec.

- [ ] **Step 1: Open the page in Chrome at desktop width**

Run a real query. Use DevTools to inspect spacing.

Checklist:
- [ ] Page background is `#fafafa` (not pure white).
- [ ] Hero block is centered, max-width 720px.
- [ ] Headline uses Georgia at 28px.
- [ ] Textarea has visible border and 3px focus ring in `--navy-lt`.
- [ ] "Find evidence" button is `#3d1b51` purple at rest, `#4B2E83` on hover.
- [ ] Cards have 1px `#e0e0e0` borders, 10px border-radius, 1.5rem padding.
- [ ] Title is `#3d1b51` weight 600.
- [ ] Finding callout has 3px gold left border, cream `#fffbe6` background, Georgia 15.5px.
- [ ] Free full text badge is green; abstract-only is purple.
- [ ] Swap button is grey link-style; hover turns navy.

If any item fails, edit the relevant CSS rule.

- [ ] **Step 2: Resize Chrome to 380px width (mobile sim)**

Checklist:
- [ ] Hero text wraps cleanly.
- [ ] Textarea is full width minus padding.
- [ ] Cards remain readable; padding shrinks to 1.25rem.
- [ ] Finding callout font drops to 14.5px.
- [ ] Card footer stacks vertically (badge above swap button).

- [ ] **Step 3: Use the page on an Android Chrome device via remote inspection or a phone**

If a phone isn't accessible right now, defer to Task 13's playtest checklist.

- [ ] **Step 4: Verify tests still pass**

`#test`: 19 pass.

- [ ] **Step 5: Commit (only if CSS changes were made)**

```bash
git add index.html
git commit -m "CSS polish: spacing, hover states, mobile breakpoint"
```

If no changes were needed, skip the commit and proceed.

---

## Task 12: Rewrite the README

**Files:**
- Modify: `README.md` (complete replacement)

- [ ] **Step 1: Replace `README.md`**

```markdown
# LitSupport

Find peer-reviewed PubMed evidence to back up a clinical question or a claim. Type, click, read. No login, no install, no data leaves your browser except the search itself.

→ Open it: https://armigedon.github.io/litsupport/

## How to use it

1. Paste a clinical question (`Does rivaroxaban reduce stroke vs warfarin in afib?`) or a claim (`GLP-1 agonists improve outcomes in heart failure.`).
2. Click **Find evidence**.
3. Skim 8 result cards. Each one shows the actual finding pulled from the abstract's conclusion.
4. Click a card title to open the full PubMed entry in a new tab.
5. If a card is irrelevant, click **Wrong? Swap →** to replace just that one with the next-best match.
6. For a claim, the header offers a one-click toggle to flip from supporting evidence to counterevidence.

## Need it on a locked-down work computer?

LitSupport is a webpage, not an installer. Open the URL above in Chrome and you're done.

If your network blocks GitHub Pages, save the page once from any unrestricted browser (`Ctrl+S` → save as `litsupport.html`) and move it to your work machine. Double-click the saved file: it opens in Chrome and runs locally. Searches still work as long as the browser has internet.

## How searches work

- Searches **PubMed** via the official [NCBI E-utilities API](https://www.ncbi.nlm.nih.gov/books/NBK25497/) (NIH).
- ESpell pre-corrects common typos (`ribaroxaban` → `rivaroxaban`).
- NCBI's own MeSH query-translation is used as the primary concept block.
- Results are ranked by PubMed relevance, then stable-sorted to put meta-analyses and systematic reviews ahead of RCTs ahead of cohort studies.
- If a search would return fewer than 5 results, LitSupport automatically widens (drops the stance modifier, then drops the abstract requirement) before giving up.
- For claims, a default "supporting evidence" stance modifier biases toward reviews and trials; the counterevidence toggle flips it to primary studies excluding reviews.
- No data leaves your browser except the search query sent to NIH. No analytics. No telemetry. No accounts.

## Deploy your own copy

If you want to host LitSupport at your own GitHub Pages URL:

1. Fork or clone this repo.
2. In Settings → Pages, set source to "Deploy from a branch", branch `main`, folder `/ (root)`.
3. Wait ~60 seconds. Open `https://YOUR-USERNAME.github.io/litsupport/`.

The whole tool is one `index.html` file with inline styles and scripts. No build step. No dependencies. Edit and push.

## Tests

Open the deployed URL or local file with `#test` appended (`index.html#test`). The inline test harness runs assertions against the three pure functions (`classifyInput`, `extractFinding`, `rerankByEvidence`) and reports pass/fail to the DevTools console. The harness does not run on the normal URL.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "Rewrite README: user-first, deploy guide moved below, Chrome-only escape hatch documented"
```

---

## Task 13: Manual playtest

**Files:** none modified unless bugs surface.

This task is the spec's explicit acceptance gate per the user's *playtest-blind bugs* preference: structural test passes are not sufficient.

- [ ] **Step 1: Ten real clinical questions**

For each of the following, run the search and verify (a) the results render without errors, (b) at least 5 cards have a finding callout, (c) the relevance is plausibly on-topic (the implementer's medical judgment is rough — flag a query as "looks wrong" only if the cards are clearly off-domain, e.g. you searched diabetes and got dermatology). Note any failures in `docs/playtest-notes-2026-06-10.md`.

1. `DOAC vs warfarin in non-valvular afib`
2. `GLP-1 agonists in patients with heart failure`
3. `Sepsis bundle compliance and mortality`
4. `Inhaled corticosteroids in mild persistent asthma in children`
5. `Vitamin D for fall prevention in adults over 70`
6. `Empagliflozin in chronic kidney disease without diabetes`
7. `Antibiotic prophylaxis for transrectal prostate biopsy`
8. `Methotrexate vs anti-TNF in early rheumatoid arthritis`
9. `Tranexamic acid in postpartum hemorrhage`
10. `Direct oral anticoagulants for cancer-associated VTE`

- [ ] **Step 2: Five claims with counterevidence toggle**

For each claim, run with default (supporting), then toggle counterevidence. Verify the result sets are *substantively different* (at least 3 of the 8 PMIDs differ between the two views).

1. `Statins reduce cardiovascular events in primary prevention.`
2. `Routine PSA screening reduces prostate cancer mortality.`
3. `Tight glycemic control reduces complications in critically ill patients.`
4. `Beta-blockers improve survival in stable heart failure.`
5. `Mediterranean diet prevents type 2 diabetes.`

- [ ] **Step 3: Three typo-heavy queries**

Verify the "Showing results for:" notice appears.

1. `ribaroxaban vs warferin in afib`
2. `acetylcystiene in acetaminofen overdose`
3. `meningacoccal vacine in adolesents`

- [ ] **Step 4: Three deliberately over-specific queries**

Verify the sparse state panel appears with a sensible over-narrow term suggestion.

1. `In Cuban-American men over 75 with chronic lymphocytic leukemia, does ibrutinib improve overall survival vs venetoclax at 5 years?`
2. `Effect of curcumin nanoparticles on left ventricular ejection fraction in postmenopausal women with HFpEF`
3. `Telemedicine intervention adherence in rural Appalachian adolescents with type 1 diabetes during pandemic lockdown`

- [ ] **Step 5: Mobile Chrome (Android)**

Open the local file or GitHub Pages URL on an Android Chrome browser. Run one query. Verify:
- [ ] Hero wraps cleanly on small width.
- [ ] Textarea is comfortable to tap and type into.
- [ ] Cards readable; finding callout legible.
- [ ] Swap button reachable with thumb.

- [ ] **Step 6: `file://` deployment test**

In desktop Chrome: open `index.html`, hit `Ctrl+S`, save as `litsupport-test.html` to Desktop. Open the saved file by double-click. Verify:
- [ ] Page renders identically.
- [ ] Type a query and click "Find evidence". Verify cards render.
- [ ] No CORS errors in console.

- [ ] **Step 7: Bug-fix loop**

If any of Steps 1–6 surfaced bugs, file them in `docs/playtest-notes-2026-06-10.md` with steps to reproduce, then fix them in `index.html`. Commit each fix separately:

```bash
git add index.html
git commit -m "Fix: <bug description>"
```

Re-run the failing playtest step until it passes.

- [ ] **Step 8: Commit playtest notes**

```bash
git add docs/playtest-notes-2026-06-10.md
git commit -m "Playtest notes from 2026-06-10 redesign acceptance run"
```

(If no bugs were found, write the file with just a single line: "All playtest checklist items passed on 2026-06-10." Still commit it as a record.)

---

## Task 14: Merge to main

**Files:** branch-level operation, no file changes.

- [ ] **Step 1: Verify the branch passes its own tests**

```bash
git status
```

Expected: clean working tree.

Open `index.html#test` in Chrome. Expected: 19 pass / 0 fail.

- [ ] **Step 2: Switch to main and merge**

```bash
git checkout main
git merge --no-ff redesign-2026-06 -m "Merge redesign-2026-06: clinician-focused LitSupport rebuild"
git log --oneline -5
```

Expected: the merge commit and all task commits appear in the log.

- [ ] **Step 3: Push to origin (only after explicit user confirmation)**

Do NOT push without checking with the user first. When approved:

```bash
git push origin main
```

GitHub Pages will rebuild in ~60 seconds. Verify at `https://armigedon.github.io/litsupport/`.

- [ ] **Step 4: Delete the local feature branch (optional, after user confirms)**

```bash
git branch -d redesign-2026-06
```

---

## Self-Review

After writing the plan I checked it against the spec.

**Spec coverage:**
- Purpose, audience, failure modes — captured in plan header and Task 1 scaffold.
- Above-fold interaction model — Task 1 scaffolds the hero exactly per spec (headline + subline + textarea + button + footer line).
- Input classifier — Task 2.
- Result list (8 cards), per-card structure (title, byline, finding callout, badge, swap action) — Tasks 5 and 6.
- Counterevidence toggle — Task 9.
- Sparse state — Task 10.
- Query pipeline §1 ESpell — Task 7.
- §2 ESearch translate — Task 5 (`ncbiTranslate`).
- §3 Concept block uses NCBI translation only — Task 5 (`runSearch` chain).
- §4 Stance modifiers — Task 5 (`buildQuery`).
- §5 No PMC, no date — Task 5 (`buildQuery` only adds `hasabstract`).
- §6 ESearch fetch retmax=16 — Task 5 (`ncbiSearch`).
- §7 Auto-widen cascade — Task 6.
- §8 EFetch + ESummary with 16-PMID cache and finding extraction — Task 5 + Task 3 (`extractFinding`).
- §9 PMC full-text badge from ESummary articleids — Task 5 (`ncbiSummary`, `renderCard`).
- §10 Post-fetch rerank — Task 4 + Task 5 application.
- Visual identity (palette, system fonts, layout) — Task 1 CSS + Task 11 polish.
- Architecture single-file, three pure functions, exactly one external host — Task 1 enforces.
- Locked-down PC: Chrome target, system fonts, save-locally — Task 1 (fonts) + Task 12 (README).
- Error states — Task 1 (red banner element) + Task 5 (catch handler) + Task 8 (swap failures).
- Testing inline #test harness — Task 1 + Tasks 2/3/4 populate.
- Playtest checklist — Task 13.
- Deployment — already on GitHub Pages, Task 14 push.
- README restructure — Task 12.
- Out-of-scope items — none built (verify by scanning the file structure table).

**Placeholder scan:** No TBDs. No "TODO" markers. All code blocks contain full implementations. No "implement later" notes. Test cases are concrete with specific input/output. Spell-notice html, sparse panel html, all helper functions written out.

**Type / signature consistency:**
- `classifyInput(text) → 'question' | 'claim'` — Task 2 defines, Task 5 consumes via `mode = classifyInput(input)`.
- `extractFinding(efetchXmlDoc, pmid) → string | null` — Task 3 defines, Task 5 consumes via `extractFinding(doc, pmid)` inside `ncbiFetchAbstracts`.
- `rerankByEvidence(articles) → articles` — Task 4 defines (each article has `pmid`, `publicationTypes`); Task 5 ensures rendered articles carry `publicationTypes` from EFetch.
- `searchWithCascade(state)` — Task 6 defines, Task 8 reuses.
- `state.visible` and `state.reserve` — Task 5 establishes, Tasks 8 and 10 read.
- `state.excludeIds` — Task 5 initializes empty, Task 8 mutates.
- `state.conceptBlock` — Task 5 sets, Task 10 reads.
- `currentState` module-level — Task 5 declares, Tasks 7/8/9 use.

All consistent. No undefined references.

The plan is ready to execute.
