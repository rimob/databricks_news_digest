# Databricks News Digest

A Databricks notebook that checks what changed around Databricks and sends one email with everything new from the last few days.

It looks at:
- **Release notes** – the official release notes feed (platform, SQL, AI/BI, what's coming)
- **Databricks blog**
- **Community** – platform updates, announcements, technical blog, MVP articles
- **Open source releases** – Apache Spark, Delta Lake, Unity Catalog OSS, MLflow, Databricks CLI, Databricks SDK for Python, dbt-databricks
- **Official documentation** – new, updated and removed pages on docs.databricks.com

The email starts with one table:

| release_date | link | short_description | role |
|---|---|---|---|
| when it was published | title linked to the source | first sentence of the description | Data Engineer, Data Architect, Data Analyst, ML / AI, Platform / Admin |

Roles are assigned by keywords, so an item can have more than one.

## How the docs check works

Release notes have an RSS feed, the rest of the documentation doesn't. To catch changes there:
- the sitemap is compared with the one from the previous run, which shows **new** and **removed or renamed** pages
- the *Last updated on* date is read from the pages in the sections listed in `DOCS_SECTIONS`, which shows **edited** pages

The previous sitemap and page details are kept in a JSON file in a Unity Catalog volume. The first run saves that baseline, so new and removed pages are reported starting with the second run. The docs check reads a few thousand pages, so expect it to take a few minutes.

## Setup

### Gmail App Password
Google account → Security → 2-Step Verification → App passwords. Create one and copy it. The regular account password does not work over SMTP.

### Unity Catalog secrets
Catalog Explorer → `workspace` → `default` → **Create → Secret**:

| Secret | Value |
|---|---|
| `smtp_user` | the Gmail address that sends the email |
| `smtp_password` | the App Password |
| `mail_to` | recipient address, or several separated by commas |

The notebook reads them through the Databricks SDK. No credentials are stored in the code.
If you create them in another schema, change the `secret_schema` parameter.

### Job
Jobs & Pipelines → **Create job**:
- **Type:** Notebook
- **Source:** Git provider → this repository, branch `main`
- **Path:** `databricks_news_digest.ipynb`
- **Compute:** Serverless
- **Parameters:** `send_email` = `true`
- **Schedule:** every day at 07:00, timezone Europe/Bucharest (or your own)

Run it once with **Run now** to check that the email arrives.

You can also import the notebook into your workspace and run it by hand. With `send_email` set to `false` the table is shown in the notebook instead of being sent.

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `days_back` | `3` | how many days back to look |
| `send_email` | `false` | `true` sends the email, `false` shows a preview in the notebook |
| `check_docs` | `true` | include the documentation check |
| `volume` | `workspace.default.databricks_news` | volume for the docs state file, created if missing |
| `secret_schema` | `workspace.default` | schema that holds the secrets |

If the same parameter is set both on the job and on the task, the job value wins.

## Customizing

Everything is in the **Configuration** cell:
- `FEEDS` – add or remove sources. Any RSS or Atom feed works, including GitHub `releases.atom` feeds
- `DOCS_SECTIONS` – documentation sections checked for edits. An empty list checks the whole site, which takes longer
- `ROLE_KEYWORDS` – keywords used to tag each item with a role

## Notes

- With a daily run and `days_back` = `3`, an item shows up in three emails in a row. Use `1` if you only want what's new since yesterday.
- No email is sent on days with no updates.
- Works on Databricks Free Edition. Outbound access there is limited to a set of trusted domains.
- If a feed can't be read, it is listed under *Notes* at the bottom of the email and the other sources still run. If the docs sitemap can't be reached, the run fails; set `check_docs` to `false` to skip the docs check.
