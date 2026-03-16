---
name: arxiv-digest
description: Fetch, filter, and summarize recent AI papers from arxiv. Reads config from ~/claude-ai-digests/config.md. Saves a dated markdown digest to output_dir.
---

> **Config path:** `~/claude-ai-digests/config.md`
> If you cloned this repo to a different location, update this path in the skill file.

## Instructions

You are the arxiv-digest skill. Follow every step below in order. Do not skip steps.

---

### Step 1: Read Configuration

Read the file at `~/claude-ai-digests/config.md`.

Extract from the `## Arxiv Digest` section:
- `categories` — split on commas, trim whitespace (e.g. `["cs.AI", "cs.LG", "cs.CL"]`)
- `results` — integer
- `filter_size` — integer
- `fetch_pool_size` — integer
- `lookback_days` — integer

Extract from `## Output`:
- `output_dir` — expand `~` to the home directory path

Record today's date as YYYY-MM-DD.

---

### Step 2: Fetch RSS Feeds

For each category in the `categories` list:

1. Fetch: `https://arxiv.org/rss/<category>` (e.g. `https://arxiv.org/rss/cs.AI`)
2. If the fetch fails or returns a non-200 response: record the category name in a `failed_feeds` list, skip it, continue to the next category.
3. From the XML response, extract each `<item>` element. For each item, parse:
   - `title`: from `<title>` (strip surrounding whitespace)
   - `authors`: from `<dc:creator>` or `<author>` (may contain multiple authors; treat the entire field value as a string)
   - `abstract`: from `<description>` (strip HTML tags, collapse whitespace)
   - `link`: from `<link>` (the `https://arxiv.org/abs/...` URL)
   - `paper_id`: the arxiv ID extracted from the link — take the path segment after `/abs/` and strip any trailing version suffix (e.g. `v2`). Examples: `https://arxiv.org/abs/2501.12345v2` → `2501.12345`; `https://arxiv.org/abs/hep-th/0309201` → `hep-th/0309201`
   - `published`: from `<pubDate>`, parsed to a date object
4. Sort entries for this category by `published` descending (most recent first). Keep at most `fetch_pool_size` entries **per category** (this is a per-category cap, not a global cap — the merged pool in Step 3 may contain up to `fetch_pool_size × number_of_categories` entries before deduplication).

---

### Step 3: Merge, Deduplicate, and Filter by Date

1. Combine all per-category entry lists into one flat list.
2. Deduplicate by `paper_id` — for duplicates, keep the first occurrence (preserves category ordering).
3. Discard any entry whose `published` date is more than `lookback_days` days before today.
4. Record `total_fetched` = count of entries after deduplication and date filtering.

**If the merged pool is empty after filtering:**
- Create the output directory if it does not exist.
- Write `<output_dir>/arxiv-digest-YYYY-MM-DD.md` with content:
  ```
  # Arxiv AI Digest — YYYY-MM-DD

  ⚠ No papers found. All fetched entries were older than the lookback window ({lookback_days} days).
  Check your config or try increasing lookback_days.
  ```
- Print the file path and stop.

---

### Step 4: Hard Signal Filter

Score each paper in the merged pool. Scoring is additive.

**A. Recency score** (based on days since `published`):
- 0 days old: 3 points
- 1 day old: 2 points
- 2 days old: 1 point
- 3+ days old: 0 points

**B. Lab affiliation score** (search `authors` field, case-insensitive):
- Recognized labs: Google, DeepMind, Meta, OpenAI, Microsoft Research, Apple, Stanford, MIT, CMU, Berkeley, Anthropic
- +2 points if at least one lab name is found. (Binary — doesn't stack.)

**C. Keyword density score** (search `title` + `abstract`, case-insensitive):
- Keywords: LLM, agent, inference, RAG, fine-tuning, reasoning, multimodal, transformer, benchmark, alignment
- +1 point per unique keyword match, capped at +5 points total.

Sort all papers by total score descending. Keep the top `filter_size`. If fewer than `filter_size` remain, keep all.

Record `filter_count` = count of papers entering this step (before truncation).
Record `selected_count` = count kept after truncation.

---

### Step 5: Relevance Ranking

You now have at most `filter_size` papers. Read each title and abstract.

Rank them by **practical relevance to working software engineers building with AI**. Ask yourself: "Would an engineer building LLM applications, agents, or AI pipelines care about this paper enough to spend 15 minutes reading it?"

**Rank higher:**
- New techniques that can be applied without large-scale compute (prompting, inference optimization, fine-tuning approaches)
- Benchmarks that reveal real capability gaps practitioners encounter
- Agent/RAG/retrieval architecture improvements
- Reproducible results with released code or models

**Rank lower:**
- Narrow domain applications (medical imaging, astronomy, etc.) unless the method generalizes
- Highly theoretical results with no near-term engineering application
- Papers that restate known results without novel contribution

Select the top `results` papers. If fewer than `results` papers remain at this stage, select all of them. These are your digest entries.

Record `final_count` = actual number of papers selected.

---

### Step 6: Write the Digest

Create the output directory if it does not exist.

Write to `<output_dir>/arxiv-digest-YYYY-MM-DD.md`.

**Header:**
```
# Arxiv AI Digest — YYYY-MM-DD

> {total_fetched} papers fetched · {selected_count} passed hard signal filter · {final_count} selected by relevance
> Categories: {categories joined by ", "} · Lookback: {lookback_days} days
```

If any feeds failed, add this line to the header block (one line per failed feed):
```
> ⚠ {category} feed failed to load — results may be incomplete
```

If fewer than `results` papers were available after all filtering, add:
```
> ⚠ Only {N} papers available after filtering — digest is shorter than configured
```

After the header, add a blank line, then `---`, then another blank line.

**For each selected paper (numbered 1 to N):**

```
## {N}. {title}
**Authors:** {first author} et al. · {lab name if identifiable, else omit the " · lab name" part}
(To extract first author: take the substring before the first comma or semicolon in the authors field.)
**Link:** {link}

**What they built:** {One paragraph in plain English. Cover: what problem they addressed, what approach they took, what result they achieved. Avoid unexplained jargon.}

**What this means for you:** {One concrete takeaway for a software engineer building with AI. What can they do with this, learn from it, or watch out for?}

---
```

After writing, print: `Digest saved to <full absolute path>`

---
