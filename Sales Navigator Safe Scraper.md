# Sales Navigator → Sheets

A pair of plain JavaScript snippets you paste into the browser console to turn a LinkedIn Sales Navigator search results page into a clean, spreadsheet-ready table.

No extension to install, no account credentials, no cookies handed to a third party, no background requests. The snippets read what is already rendered in your own browser and put it on your clipboard.

---

## Why this exists

Sales Navigator shows you the data but does not let you take it out. The usual workarounds are browser extensions and headless scrapers — both of which sign into your account and generate traffic on your behalf. This is the minimal alternative: two functions, ~40 lines each, that you can read in full before running.

---

## Key features

**Zero install, zero dependencies**
Pure vanilla JavaScript. Paste into DevTools console and run. Nothing to `npm install`, nothing to add to Chrome, nothing that persists after you close the tab.

**No credentials, no cookies, no network calls**
The snippets never ask for a password, never read your session cookie, and never send data anywhere. They only call `document.querySelectorAll` and `copy()`. Everything stays in your browser and your clipboard.

**No automated navigation**
The script does not click, scroll, or paginate for you. You control the pace entirely — it only reads the page in front of you at the moment you run it. This is a deliberate design choice, not a missing feature.

**Multi-page accumulator with deduplication**
Results are collected into `sessionStorage` across pages. Duplicates are detected by profile URN, so re-running on the same page is harmless and never creates double entries.

**Backfill on re-run**
If a record was captured before a field existed (or a field was empty), re-running on that page fills in the gaps on the existing record instead of skipping it. Useful when you extend the schema mid-collection.

**Resilient selectors with fallbacks**
Primary extraction uses LinkedIn's stable `data-anonymize` attributes. Where those are absent — for example a company with no LinkedIn page, which renders as plain text rather than a link — the script falls back to parsing the subtitle. Empty cells are the exception, not the norm.

**Lazy-render detection**
Sales Navigator defers rendering rows until they enter the viewport, which silently costs you leads if you extract too early. The script counts unrendered rows and warns you instead of quietly returning partial data.

**Permanent profile identifiers**
Every lead is stored with its URN (`ACwAAA...`), LinkedIn's internal member ID. Unlike a vanity URL (`/in/john-smith`), which the member can change at any time, the URN never changes. Both a public profile link and a Sales Navigator link are reconstructed from it, and the raw URN is kept as its own column so links can be rebuilt if LinkedIn's URL format ever changes.

**Company identifiers too**
The same treatment for companies: the numeric company ID is stored, and both the public company page URL and the Sales Navigator company URL are derived from it.

**TSV output, not CSV**
Output is tab-separated. Job titles and person names routinely contain commas (`Steven Charlap, MD, MBA`), and Google Sheets' *Split text to columns* ignores CSV quoting — which shifts entire rows out of alignment. Tabs never appear in the data, so a straight paste lands correctly every time with no import dialog.

**Progress visibility**
Each run reports how many records were added, how many were updated, the running total, and how many rows are still unrendered or missing a company — so you always know whether the dataset is complete.

**Readable and auditable**
Two short functions with no minification and no build step. You can read every line before you run it, which is the whole point of not using a black-box extension.

---

## Requirements

- A LinkedIn Sales Navigator seat (the script reads a page you already have access to)
- A Chromium-based browser or Firefox with DevTools
- Nothing else

---

## Usage

Open DevTools with **F12** and switch to the **Console** tab.

### Step 0 — Reset (only when starting a fresh batch)

```js
sessionStorage.removeItem('__leads'); console.log('Cleared');
```

### Step 1 — Scroll (manual)

Let the page load, then scroll to the bottom yourself until the pagination bar is visible. Sales Navigator renders rows only as they enter the viewport; skipping this means incomplete data.

### Step 2 — Collect (run on every page)

```js
(() => {
  const all = [...document.querySelectorAll('li.artdeco-list__item')];
  const ready = all.filter(li => li.querySelector('[data-anonymize="person-name"]'));
  const pending = all.length - ready.length;

  const pick = (r, s) => { const e = r.querySelector(s); return e ? e.innerText.trim().replace(/\s+/g,' ') : ''; };

  // Linked companies expose data-anonymize="company-name". Companies without a
  // LinkedIn page render as plain text, so fall back to parsing the subtitle.
  const getCompany = (r, title) => {
    const linked = pick(r, '[data-anonymize="company-name"]');
    if (linked) return linked;
    const sub = r.querySelector('.artdeco-entity-lockup__subtitle');
    if (!sub) return '';
    let t = sub.innerText.replace(/\s+/g, ' ').trim();
    if (title && t.startsWith(title)) t = t.slice(title.length);
    return t.replace(/^[\s·•|,\-–]+/, '').trim();
  };

  const store = JSON.parse(sessionStorage.getItem('__leads') || '[]');
  const byUrn = new Map(store.map(x => [x.urn, x]));
  let added = 0, patched = 0;

  ready.forEach(r => {
    const a = r.querySelector('.artdeco-entity-lockup__title a[href*="/sales/lead/"]')
           || r.querySelector('a[href*="/sales/lead/"]');
    if (!a) return;
    const salesUrl = a.href.split('?')[0];
    const urn = (salesUrl.split('/sales/lead/')[1] || '').split(',')[0];
    if (!urn) return;

    const title = pick(r, '[data-anonymize="title"]');
    const company = getCompany(r, title);
    const ca = r.querySelector('a[href*="/sales/company/"]');
    const companyId = ca ? (ca.href.split('/sales/company/')[1] || '').split(/[?,/]/)[0] : '';

    const ex = byUrn.get(urn);
    if (ex) {
      let did = false;
      if (!ex.company   && company)   { ex.company   = company;   did = true; }
      if (!ex.companyId && companyId) { ex.companyId = companyId; did = true; }
      if (did) patched++;
      return;
    }

    const rec = {
      name:     pick(r, '[data-anonymize="person-name"]'),
      title,
      company,
      location: pick(r, '[data-anonymize="location"]'),
      about:    pick(r, '[data-anonymize="person-blurb"]'),
      urn, salesUrl, companyId
    };
    store.push(rec); byUrn.set(urn, rec); added++;
  });

  sessionStorage.setItem('__leads', JSON.stringify(store));
  const noCo = store.filter(x => !x.company).length;
  console.log(`+${added} new | ${patched} updated | total: ${store.length} | missing company: ${noCo}`);
  if (pending) console.warn(`${pending} row(s) not rendered — scroll further and run again.`);
})();
```

Then click **Next** yourself and repeat from Step 1.

### Step 3 — Export (at the end, and periodically as a backup)

```js
(() => {
  const d = JSON.parse(sessionStorage.getItem('__leads') || '[]');
  if (!d.length) return console.warn('No data collected.');
  const c = s => String(s || '').replace(/[\t\r\n]+/g, ' ').trim();

  const out = d.map(x => [
    c(x.name), c(x.title), c(x.company), c(x.location), c(x.about),
    'https://www.linkedin.com/in/' + x.urn,
    x.companyId ? 'https://www.linkedin.com/company/' + x.companyId : '',
    x.salesUrl,
    x.companyId ? 'https://www.linkedin.com/sales/company/' + x.companyId : '',
    x.urn,
    x.companyId || ''
  ].join('\t'));

  copy([
    'Name\tTitle\tCompany\tLocation\tAbout\tLinkedInURL\tCompanyLinkedInURL\tSalesNavURL\tCompanySalesNavURL\tURN\tCompanyID',
    ...out
  ].join('\n'));

  const withCo = d.filter(x => x.companyId).length;
  console.log(`Copied: ${d.length} leads | company link: ${withCo}/${d.length}`);
})();
```

Paste into cell **A1** of a blank sheet. **Do not run *Split text to columns*** — the data is already tab-delimited and splitting on commas will break rows whose names or titles contain commas.

---

## Output schema

| # | Column | Example |
|---|--------|---------|
| 1 | Name | Gregory Haardt |
| 2 | Title | Co-Founder - CTO/CPO |
| 3 | Company | Apps & Bits |
| 4 | Location | San Mateo, California, United States |
| 5 | About | Profile blurb (empty when the member has none) |
| 6 | LinkedInURL | `linkedin.com/in/ACwAAAAIQ6kB...` |
| 7 | CompanyLinkedInURL | `linkedin.com/company/12345678` |
| 8 | SalesNavURL | `linkedin.com/sales/lead/ACwAAAAIQ6kB...` |
| 9 | CompanySalesNavURL | `linkedin.com/sales/company/12345678` |
| 10 | URN | `ACwAAAAIQ6kB...` |
| 11 | CompanyID | `12345678` |

### How the URLs are built

LinkedIn's `/in/` route accepts a member URN in place of a vanity slug and redirects to the member's real profile. The same applies to `/company/` with a numeric company ID. Both were verified working at the time of writing; since this is undocumented behaviour, verify one link before relying on a large export. The raw URN and company ID are kept as separate columns precisely so that links can be regenerated if that behaviour ever changes.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Fewer rows than expected | Rows below the fold never rendered | Scroll further, run Step 2 again (no duplicates created) |
| `missing company: N` | Some companies have no LinkedIn page | Expected — the name is still captured, only the link column is empty |
| Rows shifted in the spreadsheet | *Split text to columns* was applied | Re-paste; do not split. Output is tab-delimited |
| Some company names appear as hyperlinks | Google Sheets auto-links anything resembling a domain (`OnePint.ai`) | Cosmetic only, not from LinkedIn. Select column → right-click → Remove link |
| All data gone | `sessionStorage` is cleared when the tab closes | Export to a sheet every few pages as a backup |

---

## Limitations

- **No email addresses.** They are not present in the page's DOM. Any tool that returns emails obtains them from a separate data source, not from this page.
- **No company websites.** Only the LinkedIn company page is available on the search results page.
- **No vanity profile slugs.** Resolving `ACwAAA...` to `/in/john-smith` requires loading each profile individually.
- **Selectors are LinkedIn's, not yours.** A front-end change on their side can break extraction at any time. The inspector approach used to build this (enumerate `data-anonymize` keys, dump one row's markup) is the fastest way to repair it.
- **`sessionStorage` is per-tab and non-persistent.** Close the tab and the buffer is gone.

---

## Legal and ethical notes

Read this before using or forking.

**This conflicts with LinkedIn's User Agreement.** Section 8.2 ("Don'ts") prohibits using "software, devices, scripts, robots or any other means or processes ... to scrape the Services or otherwise copy profiles and other data from the Services," and separately prohibits copying or distributing information obtained from the Services without LinkedIn's consent. A console snippet is a script. The fact that it issues no network requests and is unlikely to be detected does not make it permitted — those are different questions, and this README does not claim otherwise.

**LinkedIn enforces this.** Consequences range from account restriction to termination, and LinkedIn has litigated against data scrapers (*hiQ Labs*, *Mantheos*, *Proxycurl*).

**You become a data controller.** Exported records are personal data. Under GDPR, UK GDPR, CCPA and similar regimes, storing and using them for outreach puts obligations on you: a lawful basis for processing, disclosure of the source on first contact, and honouring deletion requests. This applies regardless of how the data was obtained.

**Compliant alternatives exist.** Sales Navigator's own CRM Sync and list export, LinkedIn's partner APIs, and licensed third-party data providers all cover the same use case without this conflict. If you are operating commercially or at scale, use them.

This code is published for reference and educational purposes. You are responsible for how you use it.

---

## License

MIT — see `LICENSE`.
