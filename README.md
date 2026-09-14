# 🔗 Job Search Automation — Make Workflow

> Final challenge for the *Low-Code / No-Code & AI* MOOC by Le Wagon — **grade: A**.

**🛠️ Built with:** [Make](https://www.make.com) (formerly Integromat), Google Sheets, Apify, BetterContact

---

## 💡 What this does

A 4-scenario Make workflow that automates the front end of a job search: scraping relevant LinkedIn job postings based on custom search criteria, pulling full job details, then identifying and enriching missing recruiter/contact emails — all logged and kept in sync in a single Google Sheet.

The four scenarios are chained together via webhooks and scheduled polling, so the workflow runs mostly hands-off once triggered.

---

## 🗺️ Workflow overview

<img src="screenshots/all-scenarios-overview.png" alt="All scenarios overview" width="800">

| # | Scenario | Trigger | Role |
|---|---|---|---|
| 1 | Job scraping | Manual / scheduled | Reads search criteria from Sheets, scrapes matching LinkedIn job postings via Apify, logs results, fires a webhook per job |
| 2 | Job details | Webhook (from #1) | Fetches full job details for each posting via Apify, completes the Sheet row |
| 3 | Contact enrichment | Manual | Finds contacts with a missing email, requests enrichment via BetterContact |
| 4 | Contact update | Scheduled polling | Retrieves completed BetterContact enrichments, writes the found email back to the Sheet |

Tools used across the workflow: **Make, Google Sheets, Apify, BetterContact**.

---

## Scenario 1 — Filtering & job scraping

<img src="screenshots/scenario1-overview.png" alt="Scenario 1 overview" width="800">

**Trigger:** manual (or recurring schedule)

**Modules:**
1. **Google Sheets — Search Rows**: reads the `Research` tab to pull active job search criteria
2. **Apify — Run an Actor**: passes the criteria to a LinkedIn Jobs Scraper actor to launch the extraction
3. **Apify — Get Dataset Items**: retrieves the raw scraping results (titles, companies, links)
4. **Google Sheets — Add a Row**: logs each job posting found into the `JobResult` tab
5. **HTTP — Make a Request (POST)**: immediately forwards the extracted `linkedin_job_id` to Scenario 2's webhook

**Filter applied:** `job title IS NOT EMPTY AND location IS NOT EMPTY AND experienceLevel IS NOT EMPTY` — skips empty rows in the search sheet.

**Link to Scenario 2:** each new job posting instantly triggers Scenario 2 via an HTTP webhook call.

<details>
<summary>Screenshots — module configuration</summary>

<img src="screenshots/scenario1-googlesheets-apify-config-1.png" alt="Google Sheets & Apify config 1" width="400">
<img src="screenshots/scenario1-googlesheets-apify-config-2.png" alt="Google Sheets & Apify config 2" width="400">
<img src="screenshots/scenario1-addrow-config.png" alt="Add a Row config" width="400">
<img src="screenshots/scenario1-http-config-1.png" alt="HTTP module config 1" width="400">
<img src="screenshots/scenario1-http-config-2.png" alt="HTTP module config 2" width="400">

</details>

**Blueprint:** [`scenario1-job-scraping.blueprint.json`](./scenario1-job-scraping.blueprint.json)

---

## Scenario 2 — Job details

<img src="screenshots/scenario2-overview.png" alt="Scenario 2 overview" width="800">

**Trigger:** webhook received from Scenario 1

**Modules:**
1. **Webhook**: receives the `linkedin_job_id` sent by Scenario 1
2. **Apify — Run Actor**: launches LinkedIn Job Details with the received `jobId`
3. **Apify — Get Dataset Items**: retrieves the full job details
4. **Google Sheets — Update a Row**: fills in the missing columns in `JobResult` (description, job functions, company description, company industries, company specialities)

**Update filter:** `linkedin job id = linkedin_job_id received`

<details>
<summary>Screenshots — module configuration & results</summary>

<img src="screenshots/scenario2-apify-config-1.png" alt="Apify config 1" width="400">
<img src="screenshots/scenario2-apify-config-2.png" alt="Apify config 2" width="400">
<img src="screenshots/scenario2-googlesheets-config.png" alt="Google Sheets config" width="400">
<img src="screenshots/scenario2-results.png" alt="Results in Google Sheets" width="500">

</details>

**Blueprint:** [`scenario2-job-details.blueprint.json`](./scenario2-job-details.blueprint.json)

---

## Scenario 3 — Contact enrichment

<img src="screenshots/scenario3-overview.png" alt="Scenario 3 overview" width="800">

**Trigger:** manual

**Modules:**
1. **Google Sheets — Search Rows**: reads the `CompanyContact` tab, filters rows with a missing email address
2. **HTTP — Make a Request (POST)**: sends `firstName`, `lastName`, and `company website` to the BetterContact API to start an email search
3. **Google Sheets — Update a Row**: writes the unique search ID returned by BetterContact into the `request_id` column (column H) of the matching row

**Filter applied:** `email column (G) is null`
**Row identification:** via the original row number, to attach the `request_id`.

<details>
<summary>Screenshots — module configuration</summary>

<img src="screenshots/scenario3-googlesheets-config.png" alt="Google Sheets config" width="400">
<img src="screenshots/scenario3-http-config-1.png" alt="HTTP module config 1" width="400">
<img src="screenshots/scenario3-http-config-2.png" alt="HTTP module config 2" width="400">
<img src="screenshots/scenario3-googlesheets-config-2.png" alt="Google Sheets config 2" width="400">

</details>

**Blueprint:** [`scenario3-contact-enrichment.blueprint.json`](./scenario3-contact-enrichment.blueprint.json)

---

## Scenario 4 — Contact update

<img src="screenshots/scenario4-overview.png" alt="Scenario 4 overview" width="800">

**Trigger:** scheduled (polling BetterContact at regular intervals)

**Modules:**
1. **BetterContact — Enrich a Contact**: polls BetterContact for the latest successfully completed enrichments (found email + associated `request_id`)
2. **Google Sheets — Search Rows**: scans `CompanyContact` to find the exact row matching the received `request_id` against column H
3. **Google Sheets — Update a Row**: writes the enriched email address into the `email` column of the matched row

**Matching filter:** `column H (request_id) = request_id received from BetterContact`

<details>
<summary>Screenshots — module configuration</summary>

<img src="screenshots/scenario4-googlesheets-config-1.png" alt="Google Sheets config 1" width="400">
<img src="screenshots/scenario4-googlesheets-config-2.png" alt="Google Sheets config 2" width="400">

</details>

**Blueprint:** [`scenario4-contact-update.blueprint.json`](./scenario4-contact-update.blueprint.json)

---

## 🎯 Skills demonstrated

- Designing a **multi-scenario, event-driven automation** (webhooks + scheduled polling) rather than a single linear flow
- Orchestrating **third-party APIs** (Apify actors, BetterContact) inside a no-code tool
- Data integrity practices: filtering on empty/null fields, matching rows via unique IDs (`linkedin_job_id`, `request_id`) rather than row position alone
- End-to-end thinking: from raw scraping to a clean, continuously enriched dataset in Google Sheets

---

## ⚠️ Known limitations

This workflow was built and configured during a training program with limited API credits (Apify, BetterContact). Scenario logic and module configuration were fully built and reviewed, but the full end-to-end run could not be validated past a certain point once credits ran out — some edge cases (rate limits, malformed scraper output, concurrent webhook calls) haven't been stress-tested.

---

## 📄 License

Personal training project (Le Wagon MOOC), shared for portfolio purposes. Blueprint files have been stripped of personal account identifiers before publishing.
