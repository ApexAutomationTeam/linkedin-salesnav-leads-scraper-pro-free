# linkedin-salesnav-leads-scraper-pro-free
Zero-dependency browser console utility that extracts LinkedIn Sales Navigator search results into spreadsheet-ready TSV format with deduplication and permanent URN tracking. Free tool by Apex Automation Team.


# 📊 Sales Navigator → Sheets (Safe Scraper)

> 💡 **Community Project**: This zero-dependency productivity snippet is open-sourced and provided for free as part of the automation initiatives by **[Apex Automation Team](https://apexautomationteam.com/)**. Visit our official portal for custom enterprise data pipelines, CRM integrations, and sales operations automations!

A pair of lightweight, dependency-free JavaScript snippets pasted into the browser console to convert a LinkedIn Sales Navigator search results page into a clean, spreadsheet-ready TSV table without installing untrusted third-party extensions.

---

## 📺 Video Walkthrough & Release Assets

To see how manual scroll detection and the TSV export behave during live usage:

* 📦 **[View Official Demo Video & Assets v3.2.0](https://github.com/ApexAutomationTeam/linkedin-salesnav-leads-scraper-pro-free/releases/tag/v3.2.0)**
* 📥 Download demonstration assets directly to review execution in the DevTools console.

---

## 🎯 Why This Exists (Safety & Transparency)

Sales Navigator displays search data but does not provide a direct native export. Conventional workarounds rely on heavyweight browser extensions or headless scrapers that require your session cookies and generate background network traffic on your behalf. 

This utility offers a minimal, inspectable alternative:
* **Two concise functions (~40 lines each)** that can be fully audited before pasting into your browser.
* **No network calls, zero credentials, and no cookies**: It only reads what is already rendered in your browser DOM and places clean text directly on your clipboard.

---

## ⚡ Key Features

* **Zero Install & Zero Dependencies**: Runs entirely in the standard DevTools console. Nothing to install via npm, no Chrome extension permissions, and nothing persists once the tab is closed.
* **No Automated Navigation (Account Safety)**: The script deliberately does not click, auto-paginate, or auto-scroll. You maintain complete control over navigation pacing.
* **Multi-Page Accumulator + Deduplication**: Records are held in `sessionStorage` across pagination changes. Profiles are deduplicated automatically by member URN.
* **Backfill on Re-run**: If an existing record was missing fields or captured earlier, re-running on that page updates the empty fields without creating duplicate rows.
* **Resilient Fallback Selectors**: Reads LinkedIn's stable `data-anonymize` attributes and falls back to subtitle parsing for companies without dedicated LinkedIn pages.
* **Lazy-Render Detection**: Warns you if rows remain unrendered below the viewport fold, preventing silent data loss.
* **Permanent Identifiers (URN & Company ID)**: Captures internal numeric IDs (`ACwAAA...` and Company IDs) so profile URLs can be reconstructed even if vanity handles change.
* **TSV Format (Not Broken CSV)**: Copies tab-separated values to prevent formatting breaks when person names or job titles contain commas (e.g., `Steven Charlap, MD, MBA`).
* **Progress Visibility**: Outputs continuous status metrics (`+X new | Y updated | total: Z | missing company: N`) after every collection step.
* **NO Linkedin warnings beacuse we only collect rendered information**
---

## 🛠️ Step-by-Step Usage Guide

Open Chrome DevTools by pressing **`F12`** (or `Right-Click` → `Inspect`) and navigate to the **Console** tab.

### Step 0: Reset Buffer (When Starting a Fresh Batch)
```javascript
sessionStorage.removeItem('__leads'); console.log('Cleared');


Step 1: Scroll the Page (Manual Step)Let the page load and scroll smoothly down to the bottom until the pagination bar is visible. Sales Navigator only renders DOM nodes as they enter the active viewport.Step 2: Collect Leads (Run on Each Page)Copy and paste this snippet into the console on each page:JavaScript(() => {
  const all = [...document.querySelectorAll('li.artdeco-list__item')];
  const ready = all.filter(li => li.querySelector('[data-anonymize="person-name"]'));
  const pending = all.length - ready.length;

  const pick = (r, s) => { const e = r.querySelector(s); return e ? e.innerText.trim().replace(/\s+/g,' ') : ''; };

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
Navigate to the next page and repeat Steps 1 & 2.Step 3: Export to Clipboard (TSV)When finished (or periodically as a backup), run:JavaScript(() => {
  const d = JSON.parse(sessionStorage.getItem('__leads') || '[]');
  if (!d.length) return console.warn('No data collected.');
  const c = s => String(s || '').replace(/[\t\r\n]+/g, ' ').trim();

  const out = d.map(x => [
    c(x.name), c(x.title), c(x.company), c(x.location), c(x.about),
    '[https://www.linkedin.com/in/](https://www.linkedin.com/in/)' + x.urn,
    x.companyId ? '[https://www.linkedin.com/company/](https://www.linkedin.com/company/)' + x.companyId : '',
    x.salesUrl,
    x.companyId ? '[https://www.linkedin.com/sales/company/](https://www.linkedin.com/sales/company/)' + x.companyId : '',
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
Paste directly into cell A1 of Google Sheets or Microsoft Excel. Do not use "Split text to columns".📋 Data Schema OutputColumn HeaderDescriptionExampleNameLead nameGregory HaardtTitleJob titleCo-Founder - CTO/CPOCompanyCompany nameApps & BitsLocationGeographic locationSan Mateo, California, United StatesAboutProfile blurb / snippet(Text snippet from results)LinkedInURLPermanent public link via URNhttps://www.linkedin.com/in/ACwAAA...CompanyLinkedInURLPublic company link via IDhttps://www.linkedin.com/company/12345678SalesNavURLSales Navigator profile linkhttps://www.linkedin.com/sales/lead/...CompanySalesNavURLSales Navigator company linkhttps://www.linkedin.com/sales/company/12345678URNRaw internal member URNACwAAAAIQ6kB...CompanyIDRaw numeric company ID12345678⚖️ Legal & Compliance ConsiderationsPlatform Terms of Service: Automating collection from web interfaces may conflict with platform agreements (e.g., Section 8.2 of LinkedIn's User Agreement). This code is provided for research, architectural analysis, and educational evaluation.Privacy Regulations (GDPR/CCPA): Exported business records constitute personal identifiable data. Ensure you maintain a lawful basis for storing and processing B2B data under relevant data protection frameworks.🏢 About Apex Automation TeamWe engineer custom data integrations, CRM automation pipelines, and enterprise workflow solutions.Website: https://apexautomationteam.com/Inquiries & Custom Tools: contact@apexautomationteam.com
