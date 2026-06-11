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
