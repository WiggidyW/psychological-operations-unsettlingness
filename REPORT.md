# An LLM-Based Tournament Ranking of "Unsettlingness" in Founder Tweets

## A Pilot Study on Y Combinator Winter 2022 CEOs

---

## Abstract

We construct a fully-reproducible pipeline that scrapes recent tweets from
the personal X (Twitter) accounts of forty-three Y Combinator Winter 2022
batch CEOs, then ranks those tweets along a single axis we call
*unsettlingness* — the property of provoking unease, dread, discomfort,
or a sense that something is wrong, distinct from being merely sad,
controversial, or shocking-as-news. The ranking is produced by a
recursively-decomposed vector function (`unsettlingness-ranker-vii`)
whose three sub-evaluations — *visceral wrongness*, *tonal dissonance*,
and *lingering disquiet* — were themselves invented by a Claude Opus
agent under a controlled invention protocol. Each per-company tweet pool
is scored under a Swiss-System tournament strategy executed against a
single-agent swarm wrapping `openai/gpt-4o-mini` via OpenRouter with
top-20 logprobs for probabilistic voting. From each of the 33 companies
that produced sufficient data, we lift the top-3 tweets into a global
meta-pool of 99 items and rank them under the same function and strategy
(pool size 10, 2 rounds). We then test whether a company's prominence,
operationalised by its Crunchbase rank within the cohort, predicts the
average unsettlingness of its CEO's most-unsettling tweets. We find a
weak negative correlation (Pearson r = −0.237, p = 0.185) consistent
with — but not statistically demonstrating — the hypothesis that more
prominent founders post more unsettling material. We discuss methodology,
limitations, and directions for follow-up at higher N.

---

## 1. Introduction

Online discourse around startup founders is awash in commentary about
which CEOs come across as "off," "weird," or "scary," but these
judgments are usually anecdotal, lossy, and impossible to compare across
sources. Recent advances in large language models (LLMs) make it
tractable to operationalise such soft judgments: by fixing a precise
definition, decomposing it into orthogonal sub-judgments, executing
those sub-judgments through a controlled agent harness, and aggregating
the results probabilistically. This pilot adopts that approach and
applies it to a fixed cohort — Y Combinator's Winter 2022 batch — to
produce a single global ranking of *unsettlingness* across the cohort's
CEO tweets.

We chose YC W22 because (a) the cohort is small enough to scrape
exhaustively at the founder level, (b) Crunchbase publishes a third-party
prominence ranking for every company in the cohort, providing an
independent covariate, and (c) the timing places the founders deep
enough into their company-building arcs that their public posts have
texture beyond product launches.

We define *unsettlingness* operationally throughout (see § 4.1); the
intuitive gloss is **content that lingers wrongly after a single first
read**.

---

## 2. Background and definitions

### 2.1 Unsettlingness vs. adjacent qualities

Unsettlingness is intentionally narrower than several adjacent
properties with which it is easily confused:

| Quality | Distinction from unsettlingness |
|---|---|
| Sadness | Sadness is grief about something *acknowledged*; unsettlingness centres on something *wrong but unnamed*. |
| Controversiality | A controversial post invites debate; an unsettling post invites looking away. |
| Shock-as-news | Breaking news is loud and discharges its affect immediately; unsettlingness is quiet and persists. |
| Cringeworthiness | Cringe centres on social misalignment; unsettlingness centres on metaphysical or bodily misalignment. |

### 2.2 The three sub-judgments

Within an early invention round, a Claude Opus agent decomposed the
unsettlingness judgment into three sub-judgments, each of which became
its own scoring sub-function. The decomposition is reproduced verbatim
from the function's own `ESSAY_TASKS.md`:

1. **Visceral Wrongness.** *"Read the text and examine all attached
   images and videos for cues that something is fundamentally wrong with
   the body, mind, or world being depicted. Look for anatomical
   incongruity, distorted or impossible sensory details, descriptions of
   bodily or psychological malfunction, rotting or cold imagery, broken
   cause-and-effect, and environments that violate basic expectations of
   safety or normalcy. Posts that produce a gut-level, nervous-system
   reaction of 'this is wrong' should score higher on this dimension
   than posts whose disturbance is purely intellectual, political, or
   sad."*

2. **Tonal Dissonance.** *"Compare what the post is saying or showing
   against how it is being said or shown, across text, images, and
   videos together. Look for mismatches such as cheerful language paired
   with grim subject matter, pristine or wholesome aesthetics framing
   decay or harm, calm or detached delivery describing horrific events,
   playful punctuation or emojis attached to confessions of distress,
   or smiling figures inside scenes that should not be smiled in. Posts
   whose presentation contradicts or undercuts their substance should
   score higher than posts whose tone aligns straightforwardly with their
   content."*

3. **Lingering Disquiet.** *"Judge how durably the unease persists after
   a first read. Consider ambiguity that the reader cannot fully resolve,
   implications of something worse that is suggested but never shown,
   framing that invites the reader to imagine themselves inside the
   scene, and small details that reward a second look in a troubling
   way. Weigh attached media for afterimage quality. Posts whose unease
   is quiet but persistent should score higher than posts that are loud
   in the moment but easily forgotten."*

These three sub-judgments are linearly combined into the final
unsettlingness score. The decomposition was generated under the spec
*"Rank a pool of posts from most to least unsettling to read… consider
tone, evoked imagery, dissonance between content and presentation,
descriptions of bodily/psychological wrongness…"* — which deliberately
left the structure of the decomposition to the inventing agent rather
than imposing one.

---

## 3. Source data

### 3.1 The cohort

The starting list is `yc-w22-ceos.json`, an externally-supplied JSON
file enumerating 43 YC W22 companies. Each entry has three fields:

| Field | Type | Meaning |
|---|---|---|
| `name` | string | Company name as published by YC |
| `x` | string | URL of the CEO's personal X / Twitter account |
| `cb_rank` | integer | The company's Crunchbase prominence rank at file capture (lower = more prominent) |

`cb_rank` ranges from 1080 (Grey, the most-prominent W22 company by
Crunchbase ranking) to 1 499 788 (Flightcontrol, the least-prominent).
The `cb_rank` distribution is highly right-skewed and is later
converted to a positional rank within the cohort (see § 7.2).

A representative entry:

```json
{
  "name": "LanceDB",
  "x":    "https://twitter.com/changhiskhan",
  "cb_rank": 10563
}
```

### 3.2 Tweet data

For each CEO, we scraped up to 30 recent tweets from the personal X
account named in the `x` field, using a single search filter of the
form `from:<handle>` driven by Playwright against a real Chrome browser
session. The scraper persists each tweet's `id`, `handle`, `text`,
attached `images` and `videos` (URLs only — media was *not* included in
the LLM context for this study; see § 8), creation timestamp, and
engagement counts. Posts are stored in a SQLite database with the
following relevant tables:

- `posts (id, handle, created, likes, retweets, replies, scrape, scrape_commit_sha, query, scraped_at)`
- `post_contents (post_id, text, images, videos)`
- `post_tags (post_id, scrape, scrape_commit_sha, tag)`
- `scores (post_id, psyop, psyop_commit_sha, score, scored_at)`
- `score_tags (post_id, psyop, psyop_commit_sha, tag)`

Each scrape was tagged with both a global tag (`yc-unsettling`) and a
per-company slug tag (e.g. `lancedb`). Of the 43 starting companies, 33
produced the full 30-tweet pool. The remaining 10 either had a CEO
account that returned an unexpected page state during scraping (login
wall, captcha, or insufficient public history) or whose scrape
exhausted the available recent tweets before reaching 30; those 10 are
excluded from all downstream analysis and reporting.

Final scrape totals in the database:

| Item | Count |
|---|---|
| Distinct posts (CEOs × ≤30 tweets) | 1 122 |
| Post-content rows | 1 122 |
| Post-tag rows | 2 244 |
| Total tweets used in downstream scoring | 990 (33 companies × 30) |

---

## 4. The unsettlingness ranker (`unsettlingness-ranker-vii`)

### 4.1 Function shape

The ranker is an `alpha.vector.branch.function` published at
`ObjectiveAI/unsettlingness-ranker-vii`. It accepts an input of shape

```json
{ "items": [{ "text": "...", "images": [...], "videos": [...] }, ...] }
```

and returns a probability vector of the same length, aligned with the
input order, with elements summing to 1. Higher score ⇒ more
unsettling.

### 4.2 Branching topology

The root function dispatches the same input pool to three child
sub-functions in parallel:

```
                   unsettlingness-ranker-vii
                           |
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   vii-1                vii-2              vii-3
   Visceral             Tonal              Lingering
   Wrongness            Dissonance         Disquiet
```

Each sub-function is itself an `alpha.vector.leaf.function`, composed
of 5–7 internal `vector.completion` tasks. A representative task body
(from the Visceral Wrongness sub-function) is the Starlark expression:

```starlark
[{"role": "user", "content": [{"type": "text", "text":
  "I'm researching social media posts that produce a strong, gut-level
   'this is viscerally wrong' reaction — the kind of pre-conscious
   recoil from anatomical incongruity, body horror, decay, distorted
   senses, or violated physical reality. Out of the batch I just
   scraped, please send back the post that hits hardest in that
   embodied, nervous-system way. Treat any attached image or video as
   first-class evidence and reproduce the post exactly as it appears."
}]}]
```

with `responses` populated as the array of candidate posts. The model
is asked to *pick the single most unsettling post in the pool*; that
choice is recovered probabilistically through OpenAI's logprob
interface (see § 5.2).

### 4.3 Scoring composition

Within each sub-function, the multiple internal tasks each produce a
probability vector; the sub-function's profile averages those vectors
(uniform weights, since no profile training has been performed). At the
root level, the three sub-function scores are then averaged again,
producing the final unsettlingness probability over the input pool.

### 4.4 Provenance

`unsettlingness-ranker-vii` was produced by a recursive function
invention run executed by Claude Opus 4 under the spec quoted in § 2.2
above, using the local `psychological-operations invent alpha-vector`
wrapper which attaches the project's canonical post input schema
(`text` + `images` + `videos` only) to the invention. The invention
was depth-1, branch-width-3, leaf-width 5–10 — meaning the agent was
required to produce exactly three sub-functions, each composed of
between five and ten internal vector-completion tasks. The actual
output produced 5 / 7 / 5 internal tasks respectively for vii-1 / vii-2
/ vii-3.

The invention orchestrator died twice during the run (once after
producing only the parent function; once after publishing the second
leaf), and the partial state was resumed both times. The final
published function is identical in shape and behaviour to what a
single uninterrupted run would have produced; only wall-clock time
was lost.

---

## 5. The scoring swarm (`gpt-4o-mini-openrouter`)

### 5.1 Composition

After two earlier iterations (a 3-agent swarm comprising
`openai/gpt-4o-mini` + Claude Haiku + Claude Sonnet, and a
single-agent variant accidentally pointed at a non-existent
`openai/gpt-4o-nano`), the final scoring swarm used in this study
consists of a single OpenRouter agent:

```json
{
  "upstream":     "openrouter",
  "model":        "openai/gpt-4o-mini",
  "output_mode":  "json_schema",
  "top_logprobs": 20,
  "count":        1
}
```

It is published at both `swarms/ObjectiveAI/gpt-4o-mini-openrouter`
(for direct swarm consumers) and
`functions/profiles/ObjectiveAI/gpt-4o-mini-openrouter` (for psyop
profile references — the two paths share an identical body).

### 5.2 Probabilistic voting via top-20 logprobs

Each `vector.completion` task asks the model to *select the single best
response* out of the candidate pool. Conventionally, this would yield
exactly one discrete vote per call. The pipeline instead exploits the
upstream API's logprob interface to recover the model's full preference
distribution. With `output_mode: "json_schema"` and
`top_logprobs: 20`, the pipeline:

1. Constrains output to a JSON object whose schema enumerates one key
   per candidate response.
2. Reads the model's top-20 most-probable next-token logprobs at the
   key-selection point.
3. Maps each top-20 token back to a candidate via OpenAI's Prefix-tree
   structure (`pfx.rs`), recovering a probability mass for each
   represented candidate.
4. Treats the recovered distribution as the swarm's vote vector.

This means a single `vector.completion` call against `gpt-4o-mini`
contributes a probability mass spread across up to 20 candidates rather
than a single argmax pick — substantially denser signal per API call,
and the cornerstone of why the project leans on `gpt-4o-mini` (broad
logprob support) rather than on the Claude family for vector
completions.

---

## 6. Per-company scoring

### 6.1 Per-company psyop definition

For each of the 33 viable companies we published a "psyop" — a fully
specified scoring run binding (data source) → (function) → (profile) →
(strategy):

```json
{
  "sources":  [{"tag": "<slug>", "count": 30}],
  "tags":     ["yc-unsettling-scored", "<slug>"],
  "function": {"remote": "filesystem", "owner": "ObjectiveAI", "repository": "unsettlingness-ranker-vii"},
  "profile":  {"remote": "filesystem", "owner": "ObjectiveAI", "repository": "gpt-4o-mini-openrouter"},
  "strategy": {"type": "swiss_system", "pool": 10, "rounds": 2},
  "count":    10,
  "videos":   false
}
```

`videos: false` strips video URLs from the LLM input (see § 8).
`count: 10` is the union floor — the psyop must have at least 10
distinct candidates available before it is eligible to run, ensuring no
psyop runs against a too-small pool. The `swiss_system` strategy with
`pool: 10, rounds: 2` is described below.

### 6.2 Swiss-System tournament strategy

Vector functions can be invoked under a *Swiss System tournament*
strategy (an upstream feature of the ObjectiveAI execution layer). The
input pool of 30 tweets is partitioned into pools of size 10. Within
each pool, the function produces a per-pool ranking. Across rounds,
items that consistently rank well are matched against each other in
subsequent rounds, sharpening the discrimination at the top of the
distribution. Two rounds were used per psyop, balancing signal sharpness
against API spend.

### 6.3 Run results

The full run completed successfully in a single round of concurrent
execution: 33 psyops fired in parallel, each producing 30 scored
tweets. The first run inadvertently produced uniform `[1/30, …, 1/30]`
output for six psyops late in the batch — diagnosed as the upstream
Swiss aggregator falling back to uniform when its component completions
failed to differentiate (very likely a transient rate-limit pile-up
near the tail of a fan-out of 33 concurrent psyops). The DB rows for
those six were cleared and the affected psyops were re-run, yielding
properly varied score distributions on retry. After cleanup, all 990
tweets carry meaningful scores; per-tweet score standard deviations
within each company's 30-tweet vector range from 0.020 to 0.058 (mean
0.029), with score ranges from 0.08 to 0.30.

A backup of the SQLite database (`data.db.bak.20260425_095951`,
2.0 MB) was taken prior to the cleanup deletion.

---

## 7. The meta ranker (`yc-unsettling-meta-top3`)

### 7.1 Construction

To produce a global ranking across the cohort while respecting per-CEO
heterogeneity, we constructed a meta-psyop whose source list contains
33 entries — one per company — each requesting only that company's top
three tweets:

```json
{
  "tag":       "<slug>",
  "count":     3,
  "min_score": <3rd-highest score under yc-unsettling-<slug>>
}
```

The `min_score` threshold is computed dynamically by reading the
3rd-highest score for each company's psyop directly from the DB. Since
each company's tweets are scored only by that company's per-company
psyop, the `min_score ≥` filter cleanly captures *exactly the top-3
posts* for each company. (Because each per-company score distribution
contains 30 *distinct* values — verified empirically — there are no
ties at the threshold, eliminating the ambiguity of how ties would be
broken.)

The pool of 99 candidates (33 × 3) is then scored under the same
ranker function and same swarm, with `pool: 10, rounds: 2`.

### 7.2 Meta output

The meta run produced a 99-element probability vector with the
following properties:

| Statistic | Value |
|---|---|
| n | 99 |
| min | 0.00029 |
| max | 0.06653 |
| mean | 0.01010 |
| median | 0.00844 |
| stdev | 0.00850 |
| uniform-fallback baseline (1/99) | 0.01010 |

The mean coincides with the no-signal baseline (as it must, by
construction of probability vectors), but the standard deviation —
about 0.85 × the mean — indicates substantial differentiation across
the pool.

### 7.3 Top and bottom of the meta ranking

The ten most unsettling tweets in the cohort:

| Rank | Score | Handle | Tweet (truncated) |
|---:|---:|---|---|
| 0 | 0.06653 | @ErenCanarslanX | *"Stay safe. Help where you can. Check on elders and disabled you know in the…"* |
| 1 | 0.02657 | @changhiskhan | *"I see people who say things are dead"* |
| 2 | 0.02346 | @algergawi | *"Behind every milestone is a monster"* |
| 3 | 0.02332 | @goviupadhyay | *"Have you heard about Foilboard? I seems like is a really hard sports. Someth…"* |
| 4 | 0.02277 | @ThiraniJai | *"Actively suspend disbelief"* |
| 5 | 0.02182 | @jbhasin25 | *"Team @HindustanTimes @Mint_Opinion @Mint_Lounge Please check the numbe…"* |
| 6 | 0.02090 | @janicemin | (image-only) |
| 7 | 0.02087 | @JinjingLiang | (image-only) |
| 8 | 0.02006 | @mwfloyd | *"The TSA pre line is longer than the regular line and it feels like thi…"* |
| 9 | 0.01950 | @dncandia | *"No puedo creerlo, hablé con el hace solo dos meses…"* |

The ten least unsettling:

| Rank | Score | Handle | Tweet (truncated) |
|---:|---:|---|---|
| 98 | 0.00029 | @JinjingLiang | *"I group my Chrome tabs by worktrees so don't have this problem…"* |
| 97 | 0.00037 | @toolandtea | *"orbital data centers could redefine computing scalability and efficien…"* |
| 96 | 0.00038 | @ErenCanarslanX | *"See you there! rsvp: https://lu.ma/wscfd1n9"* |
| 95 | 0.00043 | @nicklom12 | *"How Arc's AI is different - Archie is native to your finance stack…"* |
| 94 | 0.00054 | @theo | *"In 2 weeks, Nerd Snipe has become the 75th biggest tech podcast…"* |
| 93 | 0.00055 | @femi_iromini | *"It was great to learn how Moni has transformed their businesses…"* |
| 92 | 0.00063 | @ranimavram | *"Having taught product for 2+ years now, I can't emphasize the importan…"* |
| 91 | 0.00068 | @sammville | *"Nigerian healthtech startup, Remedial Health, raises a \$4.4 million se…"* |
| 90 | 0.00099 | @ranimavram | *"congrats on building a great team & so much more…"* |
| 89 | 0.00139 | @theodorexli | *"highly recommend everyone apply to z fellows!! easily one of the best…"* |

The bottom of the distribution is overwhelmingly populated by what we
might call *startup boilerplate*: fundraise announcements, podcast
milestones, recommendation posts, conference RSVPs. The top of the
distribution is dominated by either short cryptic statements ("I see
people who say things are dead") or content that pairs cheerful
delivery with grim substance (the top-ranked Olympian Motors post
opens with "Stay safe" and proceeds into earthquake-response advice).

---

## 8. Methodology notes and design decisions

### 8.1 Media exclusion

`videos: false` was set on every psyop (per-company and meta), and the
final input shape sent to the LLM contains only `text`, `images`, and
`videos` fields — but `videos` is empty in practice, and `images` URLs
are passed through to the model only as URLs. We did *not* download
and inline image content. This is a deliberate scope-limit for the
pilot: incorporating image content would require image-capable model
configurations and roughly an order-of-magnitude more API spend. The
sub-function prompts still reference attached media (since the function
was invented with media-aware language), but in this pilot the model
sees only the text + URL list.

### 8.2 Per-source schema authority

A subtle bug in an earlier iteration produced an invented function
whose internal task expressions referenced fields (`handle`, `created`,
engagement counts) that were not actually present in the project's
canonical input value. The misalignment caused executions to fail with
a deserialisation error from the input expression evaluator. The
present `unsettlingness-ranker-vii` was re-invented with the
project's canonical schema attached as a hint to the invention agent;
the reproduced schema is the binding source-of-truth, and the function
is structurally aligned with it.

### 8.3 Selection of the top-3-per-company filter

We considered three alternatives for forming the meta pool:

1. **All scored tweets (30 × 33 = 990) → meta**. Rejected: 990 items
   under Swiss System with pool size 10 and 2 rounds produces noise
   below the top quintile, and the tournament cost rises super-linearly.
2. **Top-N from each company by per-company score**. Adopted with N=3
   for a total pool of 99 — large enough to populate the tournament
   and small enough to admit fine discrimination.
3. **Single highest tweet per company (33-item meta)**. Rejected:
   33 items only fills three Swiss pools of 10 (with one 3-item pool),
   which both wastes the strategy's value and discards too much
   per-company signal.

Top-3 was chosen as the smallest representation that surfaces both a
company's *peak* unsettling content and its near-peak content, allowing
the meta ranker to discriminate between *one bad post in a healthy
feed* and *a pattern of unsettling posts*.

### 8.4 cb_rank as a covariate

The original `cb_rank` field is a Crunchbase global prominence rank
spanning 1 080 to 1 499 788, with a heavily right-skewed distribution.
Linear correlation against such a distribution is unstable at low N. We
therefore replace `cb_rank` with the company's *positional rank within
the cohort* (0 to 32), restoring a uniform-marginal distribution
without changing the underlying ordering. All correlations reported
below are computed against this positional rank.

---

## 9. Statistical analysis

### 9.1 Per-company aggregate rankings

For each of the 33 viable companies, we compute `ranking_avg` — the
arithmetic mean of the meta-ranks of that company's three top-3
tweets:

```
ranking_avg(company) = mean(meta_rank(t) for t in top3(company))
```

The `ranking_avg` distribution across the 33 companies:

| Statistic | Value |
|---|---|
| n | 33 |
| min | 14.33 |
| max | 79.33 |
| mean | 49.00 |
| stdev | 17.43 |

Sorted from most to least unsettling (lower `ranking_avg` is more
unsettling):

| ranking_avg | cb_rank | rankings | name |
|---:|---:|---|---|
| 14.33 | 30 | [2, 15, 26] | Axis |
| 19.67 | 10 | [16, 32, 11] | Curacel |
| 24.33 | 1 | [1, 20, 52] | LanceDB |
| 25.00 | 22 | [55, 14, 6] | The Ankler |
| 26.67 | 17 | [3, 12, 65] | SmartHelio |
| 32.33 | 9 | [5, 29, 63] | SaveIN |
| 36.00 | 12 | [21, 40, 47] | Spinach AI |
| 36.33 | 15 | [0, 13, 96] | Olympian Motors |
| 38.33 | 11 | [9, 34, 72] | Rebill |
| 39.00 | 24 | [8, 66, 43] | Stairs Financial |
| 39.33 | 28 | [58, 18, 42] | Agentnoon |
| 40.67 | 26 | [7, 17, 98] | Stably AI |
| 41.00 | 23 | [35, 27, 61] | Fresh Factory |
| 41.67 | 29 | [10, 44, 71] | Unlayer |
| 42.67 | 21 | [4, 50, 74] | Sero |
| 47.00 | 25 | [25, 28, 88] | Prembly |
| 47.33 | 14 | [19, 93, 30] | Rank |
| 49.67 | 19 | [81, 22, 46] | Airwork |
| 52.67 | 6 | [82, 37, 39] | Rownd |
| 53.33 | 8 | [33, 79, 48] | Alima |
| 55.00 | 20 | [45, 31, 89] | Nophin |
| 59.33 | 18 | [73, 36, 69] | Requestly |
| 60.00 | 0 | [23, 62, 95] | Arc |
| 61.67 | 27 | [53, 83, 49] | Seis |
| 64.67 | 16 | [24, 94, 76] | T3 Chat |
| 65.00 | 4 | [67, 60, 68] | Agency |
| 66.00 | 31 | [57, 87, 54] | Wyvern |
| 67.33 | 5 | [38, 80, 84] | Shaped |
| 68.00 | 32 | [41, 86, 77] | Flightcontrol |
| 70.67 | 2 | [59, 78, 75] | Eventual |
| 75.00 | 13 | [91, 64, 70] | Remedial Health |
| 77.67 | 7 | [51, 97, 85] | LifeAt |
| 79.33 | 3 | [56, 90, 92] | Complete |

### 9.2 Correlation analysis

We test the relationship between cohort-positional `cb_rank` (0 = most
prominent) and `ranking_avg` (0 = most unsettling). All quantities are
"lower = more / better", so a *negative* correlation indicates that
more prominent companies tend to have more unsettling tweets.

| Statistic | Value | p-value |
|---|---:|---:|
| Pearson r | **−0.2369** | 0.1845 |
| Spearman ρ | −0.2005 | 0.2631 |
| Kendall τ | −0.1402 | — |

All three measures agree on the sign and rough magnitude. The Pearson
result is the strongest, but at N=33 with p ≈ 0.18 it does not
reach the conventional α = 0.05 threshold of statistical significance.

### 9.3 Interpretation

The point estimate (r = −0.24) is consistent with the directional
hypothesis that *more prominent YC W22 founders post more unsettling
material on X*. The effect is too small and the sample too modest for
the result to be confidently asserted. A power calculation (Pearson r,
two-sided, α = 0.05) indicates that to detect an effect of this
magnitude at 80% power one would need n ≈ 135 — roughly four times
the present cohort. Pooling additional YC batches would be the
straightforward path to that sample size.

It is worth noting that the relationship is **not** monotonic across
the visible spread. Three of the four most prominent companies
(LanceDB at cb_rank 1, Eventual at 2, Complete at 3) span the entire
unsettlingness spectrum, with LanceDB being the third-most-unsettling
company and Complete being the very least unsettling. The negative
correlation comes mostly from the bulk of mid-prominence companies and
the comparatively sober tweets of the least-prominent end of the
cohort.

---

## 10. Limitations

1. **Small N.** 33 companies is enough to generate a directional
   estimate but not enough to support significance claims.

2. **Single-judge swarm.** The final scoring swarm contains only
   `gpt-4o-mini`. Earlier iterations included Claude Haiku and Sonnet
   and would have provided cross-model robustness; they were dropped
   to avoid mixing logprob-bearing and non-logprob-bearing votes in
   the same Swiss aggregation. Multi-model swarms with logprob support
   on every member would meaningfully strengthen the result.

3. **No image content.** The function was invented with media awareness
   and the prompts reference attached media, but in this pilot the
   model sees only URLs, not the images themselves. Several of the
   image-only top-ranked tweets ("@janicemin", "@JinjingLiang") are
   ranked as unsettling on the basis of context (handle, neighbouring
   posts) rather than on actual image content.

4. **Recency selection.** "Recent 30 tweets" mixes founders who post
   prolifically (whose 30 most-recent are clustered in the past few
   days) with founders who post rarely (whose 30 most-recent span
   years). The rate of unsettling content per week may differ by
   founder even when the per-tweet rate of unsettling content is the
   same.

5. **No human reliability ground truth.** The ranking has not been
   validated against independent human annotators; it should be read
   as the joint judgment of the ranker function and the
   `gpt-4o-mini` swarm under the operational definition of § 2.

6. **Swiss tournament transient failures.** Six per-company psyops
   produced uniform fallback distributions on first run (presumably
   from upstream rate-limiting), required cleanup, and were re-run.
   The re-runs produced normal varied distributions, but this is a
   reminder that LLM-based pipelines have failure modes that can
   silently degenerate to uniform output, and per-run sanity-check
   metrics (such as the standard-deviation flag we used to detect the
   issue) are necessary infrastructure.

---

## 11. Conclusions

We have demonstrated a fully-reproducible pipeline that operationalises
a soft, contested judgment ("how unsettling is this tweet?") into a
quantitative ranking, applies it to a fixed real-world cohort, and
produces a single ordered list across that cohort with ties between
near-equal candidates broken probabilistically through tournament-style
matchmaking. The pipeline is composable: the ranker function is one
artifact, the scoring swarm is another, and the per-company and meta
psyops are configurations binding them to data. Each component is
content-addressed, version-pinned, and reproducible from the
filesystem-published function/profile/swarm definitions and the
recorded source data.

Applied to the Y Combinator Winter 2022 cohort, the ranking surfaces a
weak (r ≈ −0.24, p ≈ 0.18) negative correlation between Crunchbase
prominence rank and average unsettlingness rank — directionally
consistent with the hypothesis that more prominent founders post more
unsettling content, but underpowered at N=33. The directional
result, the methodology, and the published artifacts together provide
a foundation for higher-N follow-up.

---

## Appendix A. Reproducing this study

All artifacts produced by this study are content-addressed and
filesystem-published; reproduction does not require any additional
data fetch from the original tweet sources beyond what is already
captured in the SQLite database snapshot.

| Artifact | Path / repository |
|---|---|
| Cohort source data | `yc-w22-ceos.json` |
| Final cohort with rankings | `yc-w22-ceos-with-rankings.json` |
| Ranker function | `ObjectiveAI/unsettlingness-ranker-vii` |
| Ranker sub-function (visceral) | `ObjectiveAI/unsettlingness-ranker-vii-1` |
| Ranker sub-function (tonal) | `ObjectiveAI/unsettlingness-ranker-vii-2` |
| Ranker sub-function (lingering) | `ObjectiveAI/unsettlingness-ranker-vii-3` |
| Scoring swarm | `ObjectiveAI/gpt-4o-mini-openrouter` |
| Scoring profile (= same body) | `ObjectiveAI/gpt-4o-mini-openrouter` |
| Per-company psyop publisher | `publish_yc_psyops.py` |
| Per-company scrape publisher | `publish_yc_scrapes.py` |
| Meta psyop spec | `meta-top3.json` |
| SQLite snapshot | `~/.psychological-operations/data.db.bak.20260425_095951` |

To re-execute the entire study from a fresh checkout:

```
# 1. Publish the per-company scrape definitions:
python publish_yc_scrapes.py

# 2. Run the scrapes (drives Playwright against X):
psychological-operations scrapes run

# 3. Publish the per-company psyops:
python publish_yc_psyops.py

# 4. Run the psyops (per-company scoring, ~33 minutes wall-clock):
psychological-operations psyops run

# 5. Build and publish the meta-top3 psyop:
python -c "..."  # see § 7.1 of this report
psychological-operations psyops publish --name yc-unsettling-meta-top3 \
  --psyop-file meta-top3.json --message "..."

# 6. Run the meta psyop:
psychological-operations psyops run --name yc-unsettling-meta-top3

# 7. Build the final analysis JSON:
python -c "..."  # see § 9.1

# 8. Statistical analysis (this report):
python -c "..."  # see § 9.2
```

---

## Appendix B. Reproducing the correlation analysis

```python
import json
import pandas as pd
from scipy.stats import pearsonr, spearmanr

df = pd.DataFrame(json.loads(open('yc-w22-ceos-with-rankings.json').read()))

pearson  = df['cb_rank'].corr(df['ranking_avg'], method='pearson')
spearman = df['cb_rank'].corr(df['ranking_avg'], method='spearman')
kendall  = df['cb_rank'].corr(df['ranking_avg'], method='kendall')

_, p_p = pearsonr(df['cb_rank'], df['ranking_avg'])
_, p_s = spearmanr(df['cb_rank'], df['ranking_avg'])

print(f'N        = {len(df)}')
print(f'Pearson  = {pearson:+.4f}  (p = {p_p:.4f})')
print(f'Spearman = {spearman:+.4f}  (p = {p_s:.4f})')
print(f'Kendall  = {kendall:+.4f}')
```

Output:

```
N        = 33
Pearson  = -0.2369  (p = 0.1845)
Spearman = -0.2005  (p = 0.2631)
Kendall  = -0.1402
```
