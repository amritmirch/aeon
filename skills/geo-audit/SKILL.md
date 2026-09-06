---
name: geo-audit
description: Generative Engine Optimization audit - asks a web-grounded assistant the questions your buyers actually ask, scores whether this project appears in the answers, flags name collisions, extracts the pages the answer cited, and reports the delta against the last run.
metadata:
  title: GEO Audit
  mode: read-only
  category: productivity
  var: ""
  tags:
    - monitoring
    - marketing
  capabilities:
    - external_api
    - read_only
    - sends_notifications
  cron: "0 9 1 * *"
---

Today is ${today}.

> **${var}** — what to audit.
> - **empty** → read the brand config from `memory/geo-audit/brand.md`.
> - **`Aeon Framework`** → audit that brand this run, using generated queries.
> - **`brand=<name> queries=<a>|<b>|<c>`** → audit `<name>` against exactly
>   those buyer-intent queries, pipe-separated. Overrides generation.
> - **`report`** → run and notify, but write no snapshot (dry run; the next
>   real run still diffs against the last real snapshot).
>
> **Empty `${var}` with no `brand.md` → there is nothing to audit.** Log
> `GEO_NO_TARGET`, send **no** notification, exit clean. A monthly "set a
> brand" ping just gets muted.

## What this does

SEO measures whether you rank. **GEO measures whether you get recommended.**
When someone asks an assistant "best open source agent framework", the answer
names three or four projects and the rest of the category does not exist. This
skill measures whether this project is in that answer, and tracks the number
month over month.

It runs a set of buyer-intent questions through **live web search**, reads the
grounded answer the way a buyer would, and scores three things per query:

- **Appear** — is the project named at all? (binary, the only score that matters
  at first)
- **Prominence** — first, in the main list, or a footnote?
- **Sentiment** — described accurately and favourably, hedged, or wrong?

Then it extracts two things the score alone will not tell you: the **competitors
winning each query**, and **the exact pages the answer cited**. That cited-page
list is the actionable output. Those pages are where the model actually looks,
so they are the placement target list for the next month of work.

## What this measures, and what it does not

Be honest about this in the report, because overclaiming here is easy:

- It samples **one web-grounded assistant**, not every engine. Directionally
  representative, not a census of ChatGPT + Claude + Perplexity + Gemini.
- Answers are **stochastic**. A single query flipping between runs is noise; the
  aggregate score and the cited-page set are the signal. Never report a
  one-query flip as a win or a regression on its own.
- It measures **answers**, not traffic. Pair it with real analytics before
  claiming business impact.

Because answers are grounded in live pages rather than frozen training data,
fixes can show up in **days to weeks**, not at the next model release. That is
why a monthly cadence is the right one: fast enough to see movement, slow enough
that noise averages out.

## Capability notes (read before editing this skill)

`mode: read-only` is load-bearing, same contract as `seo-audit` and
`competitor-monitor`. The read-only guard reverts writes to code and config
paths but **preserves `memory/`**, which is where every artifact here lands.
Keep the snapshot and log writes under `memory/geo-audit/` and
`memory/logs/`, done by shell redirection. Adding a Write/Edit step, or any
Python, forces `mode: write` and changes the skill's risk profile for no gain.

---

## Steps

### S0. Bootstrap + load state

```bash
mkdir -p memory/geo-audit
PREV=$(ls -1 memory/geo-audit/[0-9]*.json 2>/dev/null | sort | tail -1)
```

`PREV` is the newest existing snapshot, written *before* this run's snapshot, so
it is always the true previous baseline. Read `memory/MEMORY.md` for context.

### S1. Parse `${var}`

`MODE` = `report` if the token is present, else `audit`. `BRAND` = the
`brand=` value, else the bare string, else the brand named in
`memory/geo-audit/brand.md`. `QUERIES` = the pipe-separated `queries=` list, if
given.

If no brand resolves: log `GEO_NO_TARGET`, notify nothing, stop.

### S2. Build the query set

If `QUERIES` was supplied, use it verbatim. Otherwise generate **four to six**
queries from `memory/geo-audit/brand.md` plus what the repo says about itself
(`README.md`, `docs/`). A good set always contains all four kinds:

1. **The category query** — the highest-volume question in the space, where the
   giants win. ("best open source AI agent framework")
2. **The niche query** — the narrow question this project should already own.
   ("agent framework that runs on GitHub Actions")
3. **The branded query** — the project's own name, which is how collisions
   surface. ("Aeon framework AI agent")
4. **The problem query** — how a buyer describes the pain before they know any
   product names. ("run Claude Code autonomously unattended")

Phrase them the way a person types into a chat box, not as keyword strings. A
query set that is all category queries produces a flat 0/N that never moves and
teaches nothing; the mix is what makes the score diagnostic.

### S3. Run the queries

For each query: run a **live web search** and read the grounded answer plus the
sources it draws on. Record, per query:

- the projects named, **in order**
- whether `BRAND` is among them
- how `BRAND` is described, verbatim, if named
- the URLs cited

Do not paraphrase away the disagreement between runs — record what the answer
actually said this time.

### S4. Score

Per query, per `references/rubric.md`:

- `appear`: `true` / `false`
- `prominence`: `first` | `listed` | `mention` | `absent`
- `sentiment`: `accurate` | `hedged` | `wrong` | `n/a`

A query counts as a **clean win** only when `appear` is true **and** sentiment is
`accurate`. Headline score is `clean wins / total queries`. Being named
inaccurately is not a win; it is a disambiguation problem wearing a win's
clothes.

### S5. Collisions and competitors

- **Collision:** on the branded query, any unrelated entity sharing the name.
  Record each one's name, what it does, and why it outranks (funding, PR,
  academic citations, age). A collision is the single most expensive GEO
  problem, because every competitor marketing dollar makes it worse.
- **Competitors:** the projects that *did* get named on each lost query, with
  the count of queries each won. That ranking is the real competitive set, which
  is often not the one the team believes it has.

### S6. Cited pages → the target list

Collect every URL cited across all queries, deduped, with the count of queries
each was cited in. Sort by count. **This is the deliverable most worth acting
on:** a page cited on three queries is worth more than a page cited on one, and
the ones open to contribution (awesome-lists, community roundups, comment
threads) are where a month of placement work should go.

Mark each target as `open` (accepts PRs or comments) or `editorial` (needs an
outreach email), because the two need completely different work.

### S7. Diff against `PREV`

Compute, and report only what moved:

- **score delta** (`1/4 → 3/4`), the headline
- **queries newly won**, and **queries newly lost** (a loss is more urgent than
  a win is reassuring)
- **prominence or sentiment shifts** on queries already won
- **new competitors** appearing, and any that dropped out
- **new cited pages** since last run — new placement opportunities
- **collision status**: unchanged, worse, or resolved

First run has no `PREV`: report the absolute state, say plainly it is the
baseline, and skip every delta line rather than printing "n/a" six times.

### S8. Write this run's snapshot

Skip entirely when `MODE` is `report`. Otherwise, by redirection:

```bash
STAMP=$(date -u +%Y-%m-%dT%H-%M-%SZ)
mkdir -p memory/geo-audit
# ... > "memory/geo-audit/${STAMP}.json"
```

One JSON object: `brand`, `run_at`, `queries[]` (each with `query`, `appear`,
`prominence`, `sentiment`, `named_projects[]`, `cited_urls[]`, `description`),
`score`, `collisions[]`, `competitors[]`, `targets[]`. Write it **before**
notifying, so a notification failure never costs the baseline.

### S9. Notify

Lead with the score and its delta, because that is the whole product:

```
GEO · Aeon Framework · 3/4 (was 1/4)
```

Then, and only then, at most three lines:

- **the biggest move**, won or lost, named as a real question a buyer asks, not
  a keyword
- **one concrete next action**, drawn from the target list, specific enough to
  start: which page, and whether it is a PR or an email
- **the collision line**, only if it changed

Rules that decide whether this gets acted on or muted:

- **Absolute URLs.** Build them from `GITHUB_SERVER_URL`/`GITHUB_REPOSITORY` the
  way `seo-audit` does; a naked `memory/...` path is dead text in a chat client.
- **Name the question, quote the answer.** "We are now first for *agent
  framework on GitHub Actions*, cited from the dev.to piece" beats "prominence
  improved".
- **Silent when nothing moved.** Same score, same targets, no new competitors ⇒
  log `GEO_NO_CHANGE` and send nothing. A monthly identical message trains the
  team to ignore the channel, and GEO genuinely has flat months.
- **Never report a single flipped query as a trend.** Say it looks like noise
  and let the next run confirm.

### S10. Log

Append to `memory/logs/${today}.md` under a `### geo-audit` heading, by
redirection: the per-query scores, the delta, new targets found, and any query
that failed to return a grounded answer. If `${var}` was empty, log a single
`GEO_NO_TARGET` line and nothing else.

## Prioritizing what to report

Rank by what changes behaviour this month, not by what is most flattering:

1. **A newly lost query.** Something that used to recommend you stopped.
2. **A worsening collision.** The namesake is pulling ahead; every week costs.
3. **A newly winnable target** — a cited page that is open to contribution.
4. **A won query.** Good news, but it does not tell anyone what to do next.

`references/rubric.md` holds the scoring definitions and the fix that maps to
each failure mode.
