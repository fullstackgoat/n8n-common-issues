# Projects to Prove I Can Solve 90–100% of n8n's Top Support Tickets

> Companion to `top-10-issues.html`. Every project below is mapped 1-to-many against the ten issue clusters identified from a fresh Firecrawl scrape of `community.n8n.io` (May 2026). The coverage matrix at the bottom shows that the recommended sprint hits **100% of the Top 10**.

---

## Why this exists

I'm applying for the **Senior Support Engineer** role at n8n. Rather than send a resume that says "I can debug things," I'm shipping artifacts that show I've already been doing the job — for free, in public, against the live forum.

For each of the 10 ticket clusters in `top-10-issues.html`, I commit to producing a **reproduction**, a **knowledge-base article**, and a **public answer** on the forum. That's 30 deliverables. The 12 projects below are the *containers* for those deliverables, organized to maximize signal per hour.

---

## Reading guide

Every project has:

- **What it is** — one paragraph, plain English
- **Concrete deliverable** — what shows up on GitHub / blog / forum
- **JD bullets it proves** — copied verbatim from the job description so a hiring manager can tick boxes
- **Issues it covers** — the issue numbers from `top-10-issues.html`
- **Time-box** — realistic for a working professional

The recommended order at the end (`Sprint plan`) hits 100% coverage in 14 days.

---

## Project 1 — `n8n-repro-lab` (the centerpiece)

A public GitHub monorepo where each subfolder is a **minimum reproduction** of a real ticket from the forum. Each repro ships:

- `docker-compose.yml` that boots a clean stack (n8n + dependencies)
- `REPRO.md` — exact steps to reproduce, observed vs. expected output
- `ROOT_CAUSE.md` — why it happens, with citations to source code or release notes
- `WORKAROUND.md` — what to tell the customer today
- `FIX.md` (when a patch exists upstream) — link to the PR/commit that fixed it

**Concrete deliverable:** repo at `github.com/<you>/n8n-repro-lab` with at least 10 working repros, one per Top-10 cluster.

**JD bullets it proves:**
> "Investigate and reproduce complex technical issues across the n8n ecosystem"
> "Strong troubleshooting and problem-solving skills"
> "Experience with Docker or containerized environments"
> "Strong knowledge of Node.js environments and containerized deployments" (IC3)

**Issues covered:** 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 — *all of them*

**Time-box:** 5 days. ~4 hours per repro.

---

## Project 2 — `n8n-troubleshooting-kb` (10 KB articles)

Public knowledge-base articles, one per cluster, written in n8n's documentation voice (clear, second-person, scannable). Each follows the same structure:

```
# [Symptom phrased the way customers Google it]

## Confirm it's this issue
## Why it happens
## Fix
## Prevent it next time
## Related forum threads
```

Hosted on a small blog (Astro / Next / Hugo) at `<you>.dev/n8n-kb` so each article is a clean URL you can paste into a ticket reply.

**Concrete deliverable:** 10 published articles, each ≤ 1,200 words, each linked from the matching folder in `n8n-repro-lab`.

**JD bullets it proves:**
> "Create and maintain documentation, FAQs, and knowledge base articles"
> "Strong written and verbal communication skills"
> "Experience documenting troubleshooting steps and improving support knowledge bases"

**Issues covered:** 1, 2, 3, 4, 5, 6, 7, 8, 9, 10

**Time-box:** 3 days. ~2 hours per article since the repro lab gives you the content.

---

## Project 3 — Forum reputation push (highest ROI single move)

Spend 45 minutes/day for 14 days answering open threads in the [Find open questions](https://community.n8n.io/c/questions/12?max_posts=1) feed. Goal: visible spot on the [Support Leaderboard](https://community.n8n.io/leaderboard/18) and the **Answer & Earn** badge before the application is submitted.

**Concrete deliverable:** a forum profile with 30+ helpful answers, screenshot of leaderboard rank, link in the cover letter.

**JD bullets it proves:**
> "Provide technical support to users via tickets, chat, and community forums"
> "Contribute to the community by answering questions and sharing best practices"
> "Passion for helping users and improving product usability"

**Issues covered:** 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 (you'll see all of them in the queue)

**Time-box:** 14 days, 45 min/day. **Start today.**

---

## Project 4 — `n8n-debug-cli`

A small Node.js CLI (~300 LOC) that scales support work. Ships as an npm package and a single-file executable.

```bash
n8n-debug repro <cluster>           # boots a docker-compose repro from the lab
n8n-debug logs <execution.json>     # parses an execution export, surfaces error/stack/timing outliers
n8n-debug webhook <url> [--secret]  # sends test payloads with HMAC, header inspection
n8n-debug oauth <url>               # decodes JWT, traces redirect chain, checks callback URL
n8n-debug net <host>                # one-shot DNS/TLS/HTTP/proxy diagnostic
```

**Concrete deliverable:** repo at `github.com/<you>/n8n-debug-cli` with README, npm package, GH Actions CI.

**JD bullets it proves:**
> "Experience working with JavaScript, Node.js, or similar technologies"
> "Help improve support tooling and resolution efficiency"
> "Experience debugging APIs, integrations, webhooks, and authentication flows"

**Issues covered:** 1, 2, 5, 7, 8, 9

**Time-box:** 3 days. Start with `logs` and `webhook` (highest daily-use value).

---

## Project 5 — Auth Troubleshooting Master Guide + JWT/HMAC Decoder

A long-form article ("Debugging OAuth2, OIDC, JWT, and HMAC in n8n: a flowchart") + a small static web tool for decoding tokens and signatures locally (no upload). Covers the six most common failure modes from the forum: callback URL on `localhost`, `allowedDomains` Public-API contradiction, refresh-token expiry, ngrok loop-back, HMAC string-to-sign assembly, OAuth1 vs OAuth2 in legacy nodes.

**Concrete deliverable:** one blog post + one mini web tool. Both linked from KB article #2.

**JD bullets it proves:**
> "Experience debugging APIs, integrations, webhooks, and authentication flows"
> "Familiarity with authentication protocols (OAuth2, OIDC, SAML, JWT)"

**Issues covered:** 2, 5, 8

**Time-box:** 2 days.

---

## Project 6 — Self-Hosting Reference Bundle

Opinionated `docker-compose.yml` set + step-by-step README that solves the most common deployment tickets in one place: persistent volumes, encryption-key handling, `WEBHOOK_URL` correctness behind nginx/Caddy/Cloudflare, healthcheck, safe upgrade procedure, **SQLite → Postgres migration script** (bash + node), and a backup workflow.

**Concrete deliverable:** repo `github.com/<you>/n8n-selfhost-reference` with three docker-compose variants (single-node SQLite, single-node Postgres, queue-mode Postgres+Redis), a 30-step README, and a Makefile.

**JD bullets it proves:**
> "Experience with Docker or containerized environments"
> "Strong troubleshooting and problem-solving skills"
> "Solid understanding of networking fundamentals"

**Issues covered:** 1, 3, 4, 9

**Time-box:** 3 days.

---

## Project 7 — Webhook & Trigger Diagnostic Playbook

A decision tree (Mermaid in a public Gist or Notion page) that walks through every "my webhook isn't firing" symptom: test vs. production URL, double-trigger, idempotency, retry on 5xx, provider-specific quirks (WhatsApp delays, Google Apps Script intermittent 403). Includes companion bash script `webhook-trace.sh` that runs the diagnostic flow against any live n8n instance.

**Concrete deliverable:** a Notion/GitHub page + the bash helper, embedded in KB article #5.

**JD bullets it proves:**
> "Diagnose and troubleshoot issues related to n8n workflows, integrations, and performance"
> "Document troubleshooting workflows and build internal playbooks"

**Issues covered:** 1, 5, 7, 8

**Time-box:** 1.5 days.

---

## Project 8 — AI Agent / RAG Production Checklist

A reusable "is your RAG actually working?" checklist + a `docker-compose.yml` that boots n8n + Qdrant + a known-good corpus + a tiny eval harness (RAGAS-style, just precision@k for now). Designed so you can paste a customer's workflow, point it at the lab corpus, and get a green/red answer in under 5 minutes.

**Concrete deliverable:** repo subfolder `n8n-repro-lab/06-rag-production-checklist/` with README, compose stack, eval script.

**JD bullets it proves:**
> "Diagnose and troubleshoot issues related to n8n workflows, integrations, and performance"
> Also lands the implicit "AI orchestration" pitch in the JD intro.

**Issues covered:** 6

**Time-box:** 2 days.

---

## Project 9 — Performance Triage Toolkit + Observability Stack

A `docker-compose.yml` that runs n8n + Prometheus + Loki + Grafana, pre-wired with a starter dashboard (execution latency p50/p95/p99, error rate, queue depth, worker memory, Redis ops/sec). Plus a short methodology doc: how to triage "slow workflow" tickets in under 10 minutes using the dashboard.

**Concrete deliverable:** repo `github.com/<you>/n8n-observability-starter` + a methodology blog post.

**JD bullets it proves:**
> "Observability tools such as Grafana, Datadog, or Prometheus" (nice-to-have)
> "Strong experience troubleshooting complex production environments" (IC3)
> "Ability to identify systemic issues and drive improvements in product reliability" (IC3)

**Issues covered:** 4, 9

**Time-box:** 2 days.

---

## Project 10 — One PR to `n8n-io/n8n`

Pick the smallest real bug from the forum that an outsider can credibly fix — almost always a docs typo, a missing-field validation, or a localized node fix. Submit a clean PR with tests if applicable.

Examples of viable starter PRs from the forum data:
- Docs PR clarifying `WEBHOOK_URL` vs. `N8N_HOST` (Issue 1)
- Docs PR for safe SQLite → Postgres migration (Issue 3)
- Snowflake Insert regression repro added to test fixtures (Issue 10)

**Concrete deliverable:** one merged or under-review PR linked in the cover letter.

**JD bullets it proves:**
> "Collaborate with Engineering and Product teams to escalate, triage, and resolve product issues"
> "Experience collaborating closely with engineering on complex technical issues" (IC3)

**Issues covered:** 1 of {1, 3, 10} — pick the easiest

**Time-box:** 1 day, including review feedback.

---

## Project 11 — "Top Systemic Issues in n8n — Q2 2026" public report

A single Substack/blog post that opens with: *"I scraped the n8n forum and analyzed N threads. Here are the top 5 systemic issues, ranked by ticket volume × business impact, plus what I'd ship to fix the top 3."* Use the data already in `top-10-issues.html` as the source.

This is the IC3-vs-IC2 differentiator. It's one thing to fix tickets; it's another to look across 200 tickets and tell product *what to fix*.

**Concrete deliverable:** one ~2,000-word post with a chart, linked from the cover letter.

**JD bullets it proves:**
> "Identify recurring issues and propose improvements to product reliability and user experience"
> "Ability to identify systemic issues and drive improvements in product reliability and support processes" (IC3)

**Issues covered:** meta — operates above all 10

**Time-box:** 1 day.

---

## Project 12 — Networking Cheat Sheet (DNS / HTTP / TCP / TLS)

A single dense reference page covering the network primitives that show up in n8n tickets: DNS resolution inside Docker, container-to-host networking, HTTP/2 vs HTTP/1.1, TLS handshake debugging, IPv6 fallback, Cloudflare proxy interactions, ngrok mechanics, and corporate proxy configuration. Each section ends with a copy-pasteable `curl` / `dig` / `tcpdump` invocation.

**Concrete deliverable:** one long blog post, linked from KB articles #1, #5, and #8.

**JD bullets it proves:**
> "Solid understanding of networking fundamentals (DNS, HTTP/HTTPS, TCP/IP)"

**Issues covered:** 1, 5, 8

**Time-box:** 1 day.

---

## Coverage matrix — proving 100%

Each row is a project. Each column is one of the Top 10 issue clusters. A `●` means that project produces a tangible deliverable that a customer with that ticket type would benefit from.

| Project | 01 Webhook URL | 02 OAuth | 03 Docker | 04 Queue mode | 05 Webhook delivery | 06 AI/RAG | 07 Loops | 08 HTTP debug | 09 Performance | 10 Node regressions |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| 1 — `n8n-repro-lab` | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● |
| 2 — KB articles | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● |
| 3 — Forum push | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● |
| 4 — `n8n-debug-cli` | ● | ● |   |   | ● |   | ● | ● | ● |   |
| 5 — Auth guide |   | ● |   |   | ● |   |   | ● |   |   |
| 6 — Self-host bundle | ● |   | ● | ● |   |   |   |   | ● |   |
| 7 — Webhook playbook | ● |   |   |   | ● |   | ● | ● |   |   |
| 8 — RAG checklist |   |   |   |   |   | ● |   |   |   |   |
| 9 — Observability stack |   |   |   | ● |   |   |   |   | ● |   |
| 10 — Upstream PR | ◐ |   | ◐ |   |   |   |   |   |   | ◐ |
| 11 — Systemic report | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● |
| 12 — Networking cheat sheet | ● |   |   |   | ● |   |   | ● |   |   |
| **Issue covered by ≥ 3 projects?** | **✓** | **✓** | **✓** | **✓** | **✓** | **✓** | **✓** | **✓** | **✓** | **✓** |

Every issue is hit by at least four independent projects. The lowest-covered issue (10 — provider node regressions) is still addressed by the repro lab, KB, forum push, and the systemic report.

**Coverage achieved: 100%.**

---

## Sprint plan — 14 days to application-ready

> Working ~3 hrs/day weekdays, ~5 hrs/day weekends. Adjust as needed.

| Day | Morning (1.5h) | Afternoon (1.5h) | Cumulative deliverable |
|:---:|---|---|---|
| 1 | Forum push: 5 answers | Repro #1 (Webhook URL) | 1 repro, 5 forum answers |
| 2 | Forum push: 5 answers | Repro #2 (OAuth) + KB #2 | 2 repros, 1 KB, 10 answers |
| 3 | Forum push | Repro #3 (Docker) + Self-host bundle skeleton | 3 repros, scaffolded P6 |
| 4 | Repro #4 (Queue mode) | Self-host bundle: queue-mode variant | 4 repros, P6 50% |
| 5 | Repro #5 (Webhook delivery) | KB #1, #3, #4, #5 (4 articles) | 5 repros, 5 KB |
| 6 | Repro #6 (AI/RAG) + RAG checklist | `n8n-debug-cli`: `logs` + `webhook` commands | 6 repros, P4 40% |
| 7 | Repro #7 (Loops) + Repro #8 (HTTP debug) | KB #6, #7, #8 | 8 repros, 8 KB |
| 8 | Repro #9 (Performance) + Observability stack | Auth guide draft | 9 repros, P9 done |
| 9 | Repro #10 (Node regressions) + KB #9, #10 | `n8n-debug-cli`: `oauth` + `net` commands | 10 repros, 10 KB, P4 done |
| 10 | Webhook playbook + Networking cheat sheet | Polish auth guide, publish | P5, P7, P12 done |
| 11 | Pick + prepare upstream PR | Submit upstream PR | P10 done |
| 12 | Draft systemic-issues report | Polish all READMEs | P11 done |
| 13 | Forum push (heavy) | Record 3 Loom videos for top KB articles | Loomed evidence |
| 14 | Final pass: cross-link everything | Submit application | **All 12 projects shipped** |

---

## Cover-letter framing (use verbatim)

> *Before applying, I scraped community.n8n.io with Firecrawl and analyzed roughly 210 recent threads to identify the ten recurring issues that arrive most often as support tickets. I then built a public reproduction lab covering all ten, wrote ten matching knowledge-base articles, shipped a Node.js debug CLI to scale my own work, and personally answered N questions on the forum (profile linked). The full report and project portfolio are in `top-10-issues.html` and `projects.md`. I'm sending these because they are the closest available preview of what I would do as a Senior Support Engineer at n8n during my first 90 days.*

That paragraph + the linked artifacts is what puts you in the top 1% of applicants for this role.

---

## What this portfolio is *not*

It's deliberately not:

- A flashy product or AI demo (wrong role)
- A long architecture treatise (you're applying to a support team, not a platform team)
- A massive open-source project that takes 6 months (you need to apply this month)

It is, intentionally:

- **Reproducible work** that mirrors the daily job
- **Documentation that helps real users today**
- **Forum reputation that the hiring panel can audit live during the interview**

Ship it.
