# Job reports with n8n + Webapify

A self-hosted pipeline that collects job offers from a job portal **that has no RSS feed or public API** and turns
them into a **PDF report saved on your own machine**, all orchestrated by n8n and running in Docker Compose.

```mermaid
flowchart LR
    T[n8n trigger] --> A[HTTP Request<br/>jobs-api:3010/jobs]
    A --> J[JSON snapshot<br/>data/jobs/*.json]
    A --> H[Code node<br/>build report HTML]
    H --> G[Gotenberg<br/>HTML → PDF]
    G --> P[PDF report<br/>data/reports/*.pdf]
    A -. polite, cached,<br/>robots.txt-compliant .-> W[(Job portal<br/>website)]
```

The project has five parts:

1. [The goal](#1-the-goal): an n8n workflow that gathers jobs and produces a local PDF report
2. [Webapify](#2-webapify--an-agent-that-turns-job-websites-into-apis): an agent that turns job websites into APIs
3. [The generated API](#3-applying-webapify-to-a-job-portal) for one job portal, built with respect for the site
4. [Docker Compose](#4-putting-the-pieces-together-with-docker-compose): every service in one stack, including how to build the API image
5. [The n8n workflow](#5-the-n8n-workflow): everything wired together

---

## 1. The goal

Many job portals offer good, curated listings but **no feed, no RSS and no public API**, so they can't be automated.
The goal of this project is an **n8n workflow** that:

- asks a job portal for offers that match your criteria (for example remote engineering jobs paying €60k+ posted today),
- saves the raw results as a dated **JSON** file, and
- renders a clean, dated **PDF report** into a local folder (`./data/reports/`).

Nothing leaves your machine except the polite requests to the job portal itself.

---

## 2. Webapify: an agent that turns job websites into APIs

To fill the missing API, we wrote **Webapify**, a Claude Code agent
(`webipify/Webipify/.claude/agents/Webapify.md`). Given the URL of a search portal, it:

| Phase | What happens |
|---|---|
| **0. Interview** | Asks which endpoints you need (search, detail, filter values), which fields matter, who will use the API (e.g. n8n) and the port, and confirms you are allowed to access the site programmatically. |
| **1. Reconnaissance** | Reads `robots.txt` and the terms **first**. Then looks for structured data in order of preference: hidden feeds and sitemaps, embedded JSON (Next.js/RSC payloads, JSON-LD `JobPosting`), and HTML. Works out how filters and pagination really work, and saves every page it fetches as evidence. |
| **Checkpoint A** | Writes `ANALYSIS.md` with the chosen strategy and a **filter map** (API parameter → how the website expects it). Nothing is coded until you approve it. |
| **2. Contract** | Writes a formal **OpenAPI 3.1** spec (`openapi.yaml`) with clean, typed, documented parameters that don't depend on the site's internal names. |
| **3. Code** | Generates a small **TypeScript** server (Express + `express-openapi-validator` + cheerio). The spec validates every request, pure functions translate parameters into site requests and parse replies into JSON, and tests run against the saved pages. It also generates a `Dockerfile` and `docker-compose.yml`. |
| **4. Verify** | Builds, runs the tests, calls the live API (valid, invalid and unknown-id cases) and checks the results against the website itself. |
| **Checkpoint B** | Hands over run commands, curl examples, the final filter map, known limitations and n8n tips. |

To run it on a new site:

```bash
cd webipify/Webipify
claude --agent webapify      # then: "Webapify https://<job-portal>"
```

### Guardrails: an honest guest, not a scraper

The agent has a **non-negotiable guardrails** section that applies both to its own analysis and to the code it generates:

- **Honesty:** a truthful, identifiable User-Agent. No pretending to be a browser or search engine, no proxies or IP rotation.
  Every generated API is marked *unofficial* and names its source.
- **robots.txt and terms are binding:** disallowed paths are never requested, not even "just to look". If the terms
  forbid the intended use, the agent stops.
- **No bypassing:** no getting around logins, paywalls, CAPTCHAs, bot challenges, rate limits or internal APIs that robots.txt disallows.
- **No overload:** at most ~30 requests during analysis. The generated code sends one request at a time with at least 1 s between
  requests, caches results, backs off on `429`/`5xx`, makes only a few upstream requests per API call, runs no background crawlers
  and is deployed as a single replica.
- **Respect the content:** personal or internal use only, original links always kept, no personal data collected.

---

## 3. Applying Webapify to a job portal

We ran Webapify against a real job portal. Which site it is doesn't matter here. What matters
is **how** we interact with it:

> **The intention of this project is to respect the website.** The API only uses pages that the site's `robots.txt`
> allows, identifies itself honestly, sends at most one request every 1.5 seconds, caches every page
> (10 minutes for listings, 24 hours for sitemaps), backs off when the site asks it to, and is only ever queried by
> a manually triggered or low-frequency workflow. It is **not** a crawler, it does **not** mirror or republish the
> site's content, and under no circumstances should it be used to put load on the site or cause anything resembling a
> denial of service. Every result links back to the original offer on the website.

What the analysis found, and how the API works as a result:

- The site is server-rendered and **embeds clean, typed JSON** for each job in its pages (salary range, remote policy,
  seniority, visa sponsorship, language requirements, tags…), so no fragile HTML scraping is needed.
- Its filters are **SEO landing pages** (e.g. `/jobs/<city>/<category>`), all allowed by robots.txt. Its own internal
  search API is *disallowed*, so **we deliberately don't use it**.
- The generated API maps each request to the most specific allowed landing page, parses the embedded jobs and applies
  any remaining filters locally. Every response says which filters were applied by the site, which locally,
  and which couldn't be applied.
- Accepted trade-off: each landing page embeds a limited number of jobs (often 12), so results are shallow but
  honest. `upstreamTotal` reports how many jobs the site says exist.

Endpoints (full contract in the project's `openapi.yaml`, Swagger UI at `/docs`):

| Method | Path | Description |
|---|---|---|
| GET | `/jobs` | Search: `category`, `city`, `region`, `province`, `location`, `sector`, `tag`, `keyword`, `language`, `workMode`, `seniority`, `visaSponsorship`, `noSpanish`, `relocationSupport`, `withSalary`, `minSalary`, `postedSince`, `page`, `pageSize` |
| GET | `/jobs/{id}` | Full offer: description (HTML and text) and the original apply link |
| GET | `/filters` | Allowed values for each filter |
| GET | `/health` | Liveness check |

The generated project (spec, TypeScript sources, tests, `ANALYSIS.md`, `Dockerfile`) lives under
`webipify/Webipify/apis/<site-slug>/`.

---

## 4. Putting the pieces together with Docker Compose

`compose.yml` (based on n8n's official `get-n8n` stack) starts everything with one command:

| Service | Role | Exposed |
|---|---|---|
| `n8n` | Workflow engine. `./data` is mounted at `/files` (file access restricted to that path), timezone Europe/Madrid | `5678` (UI) |
| `runners` | n8n task runners that execute Code nodes | internal |
| `sandbox-certs`, `sandbox-api`, `sandbox-runner-1` | n8n's sandbox service (mTLS bootstrap, API, Docker-in-Docker runner) | internal |
| `searxng` | Private metasearch engine used by n8n's built-in AI features (`searxng-settings.yml` turns on the JSON output format) | internal |
| `gotenberg` | Converts HTML to PDF for the report (`http://gotenberg:3000`) | internal |
| `jobs-api` | **The API generated by Webapify** (`http://jobs-api:3010`) | `127.0.0.1:3010` (host only, for testing) |

Secrets and versions come from `.env` (`N8N_VERSION`, `N8N_SANDBOX_VERSION`, `N8N_RUNNERS_AUTH_TOKEN`, the sandbox
keys and tokens, `SEARXNG_SECRET`, …). Keep that file private and never commit it.

### Build the API image

`jobs-api` uses a **locally built image** (`pull_policy: never`), so build it before starting the stack. The image tag
must match the `image:` value of the `jobs-api` service in `compose.yml`:

```bash
cd /home/nacho/Projects/n8n
docker build -t <image-name-from-compose.yml> webipify/Webipify/apis/<site-slug>
```

The Dockerfile is a two-stage `node:22-alpine` build. It compiles the TypeScript, drops dev dependencies, runs as the
non-root `node` user and has a `HEALTHCHECK` on `/health`. The API listens on `$PORT`. The workflow expects
**3010**, so make sure the service sets it:

```yaml
  jobs-api:
    image: <image-name>
    pull_policy: never
    environment:
      PORT: '3010'
      UPSTREAM_MIN_INTERVAL_MS: '1500'   # politeness; the code never goes below 1000
    ports:
      - '127.0.0.1:3010:3010'
```

Keep **one replica**. The rate limiter and the cache live in memory, so scaling out would multiply the load on the website.

### Start everything

```bash
docker compose up -d
docker compose ps                         # jobs-api should be "healthy"
curl -s localhost:3010/health             # {"status":"ok"}
open http://localhost:3010/docs           # Swagger UI of the jobs API
open http://localhost:5678                # n8n
```

After changing the API code, rebuild the image and recreate the container:
`docker build -t <image-name> webipify/Webipify/apis/<site-slug> && docker compose up -d jobs-api`.

---

## 5. The n8n workflow

Import `n8n_workflow.json` into n8n (*Workflows → Import from file*). It is a single linear flow with two branches:

```
When clicking 'Execute workflow'
  └─ HTTP Request1 ── GET http://jobs-api:3010/jobs?category=engineering&workMode=remote
                         &minSalary=60000&postedSince=<today>&page=1&pageSize=20
       ├─ Response to JSON file → Save JSON  →  /files/jobs/<site>-<date>.json
       └─ Build report HTML → HTML to file → Render PDF (Gotenberg) → Save PDF  →  /files/reports/<site>-<date>.pdf
```

| Node | What it does |
|---|---|
| **When clicking 'Execute workflow'** | Manual trigger. Swap it for a *Schedule Trigger* to run every few hours (every 1–6 h is plenty; please don't poll more often). |
| **HTTP Request1** | Calls the generated API by its Compose service name. Change the query string to change the search; every parameter is validated against the OpenAPI spec, so typos return a clear `400`. |
| **Response to JSON file → Save JSON** | Keeps a dated JSON snapshot of the raw API response in `./data/jobs/`. |
| **Build report HTML** | Code node: sorts jobs by date and builds an A4-styled HTML page. Each job card shows title (linked to the original offer), company, posting time, salary range, location, work mode, seniority and experience, plus badges (visa sponsorship, relocation, Spanish requirement, agency), tags and a short summary. The header lists which filters were applied. |
| **HTML to file** | Turns the HTML string into an `index.html` binary, the file name Gotenberg expects. |
| **Render PDF (Gotenberg)** | `POST http://gotenberg:3000/forms/chromium/convert/html` (multipart). Returns the PDF. |
| **Save PDF** | Writes the report to `./data/reports/` on the host. |

The result: run the workflow and a fresh `./data/reports/<date>.pdf` with the day's matching jobs appears on your machine.
Each job links back to the original offer on the website.

---

## Folder layout

```
/home/nacho/Projects/n8n
├── README.md               ← this file
├── compose.yml             ← the whole stack
├── .env                    ← versions and secrets (private)
├── searxng-settings.yml    ← SearXNG config (JSON output enabled)
├── n8n_workflow.json       ← the n8n workflow to import
├── data/                   ← mounted at /files inside n8n
│   ├── jobs/               ← JSON snapshots
│   └── reports/            ← PDF reports
└── webipify/
    └── Webipify/
        ├── .claude/agents/Webapify.md   ← the agent
        └── apis/<site-slug>/            ← API generated by the agent (spec, code, tests, Dockerfile)
```
