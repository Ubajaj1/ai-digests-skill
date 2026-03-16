---
name: github-ai-trending
description: Fetch, filter, and summarize trending AI/ML repos from GitHub. Reads config from ~/claude-ai-digests/config.md. Saves a dated markdown digest to output_dir.
---

> **Config path:** `~/claude-ai-digests/config.md`
> If you cloned this repo to a different location, update this path in the skill file.

## Instructions

You are the github-ai-trending skill. Follow every step below in order. Do not skip steps.

---

### Step 1: Read Configuration

Read the file at `~/claude-ai-digests/config.md`.

Extract from the `## GitHub Trending` section:
- `topics` — split on commas, trim whitespace (e.g. `["AI", "LLM", "machine-learning", "agents"]`)
- `results` — integer
- `filter_size` — integer
- `fetch_pool_size` — integer
- `time_window` — integer (1–7)

Extract from `## Output`:
- `output_dir` — expand `~` to the home directory path

Record today's date as YYYY-MM-DD.

Determine the `since` parameter:
- `time_window` 1 or 2 → `since=daily`
- `time_window` 3–7 → `since=weekly`

Determine the growth label for output:
- `time_window` 1 → `1-day growth`
- `time_window` 2 → `2-day growth`
- `time_window` 3–6 → `{time_window}-day growth`
- `time_window` 7 → `7-day growth`

---

### Step 2: Fetch GitHub Trending

Fetch: `https://github.com/trending?since={since}`

If the fetch fails or returns a non-200 response:
- Create the output directory if it does not exist.
- Write `<output_dir>/github-trending-YYYY-MM-DD.md` with:
  ```
  # GitHub AI Trending — YYYY-MM-DD

  ⚠ Failed to fetch GitHub trending page. Error: {error reason}
  Try again later or check your network connection.
  ```
- Print the file path and stop.

From the HTML response, extract each repository listed on the trending page. Repos are structured as `<article>` elements (class `Box-row` or similar). For each repo, parse:
- `full_name`: owner and repo name (e.g. `anthropics/claude-code`)
- `description`: the repo description text (may be empty)
- `stars_total`: total star count as an integer (strip commas)
- `stars_growth`: the recent growth figure shown on the page (e.g. 840 from "840 stars today" or 3200 from "3,200 stars this week") — extract as integer, strip commas. If no growth figure is visible for a repo, set `stars_growth = 0`.
- `topics`: list of topic tags shown on the repo card (may be empty)
- `link`: `https://github.com/{full_name}`

The trending page is already sorted by growth. Take the first `fetch_pool_size` repos.

Record `fetch_count` = number of repos actually taken (at most `fetch_pool_size`).

---

### Step 3: Filter by Topic Keywords

Keep only repos where at least one word from the configured `topics` list appears (case-insensitive) in:
- The repo `description`, OR
- Any tag in the repo `topics`

Match as substrings — "machine-learning" matches "machine-learning" in topics; "AI" matches "AI" anywhere in description.

Record `topic_filtered_count` = count of repos passing this filter.

**If zero repos pass the filter:**
- Create the output directory if it does not exist.
- Write `<output_dir>/github-trending-YYYY-MM-DD.md` with:
  ```
  # GitHub AI Trending — YYYY-MM-DD

  ⚠ No repos matched the topic filter. Topics configured: {topics joined by ", "}
  Try broadening your topics list in config.md or increasing fetch_pool_size.
  ```
- Print the file path and stop.

If fewer than `results` repos pass the filter (but more than zero), proceed with all that are available.

---

### Step 4: Hard Signal Filter

Score each remaining repo. Scoring is additive.

**A. Star growth score** (based on `stars_growth`):
- ≥ 1000: 4 points
- 500–999: 3 points
- 200–499: 2 points
- 50–199: 1 point
- < 50: 0 points

**B. Absolute star count score** (log scale, prevents new repos dominating on growth):
- ≥ 10,000 stars: 3 points
- 1,000–9,999 stars: 2 points
- 100–999 stars: 1 point
- < 100 stars: 0 points

Sort all repos by total score descending. Keep the top `filter_size`. If fewer remain, keep all.

Record `selected_count` = count kept after truncation.

---

### Step 5: Relevance Ranking

You now have at most `filter_size` repos. Read each name and description.

Rank them by **practical value to working engineers building with AI**. Ask: "Would an engineer building with LLMs, agents, or AI tools actually clone this, learn from it, or use it?"

**Rank higher:**
- Libraries, frameworks, or tools that solve real workflow problems
- High-quality reference implementations, curated benchmarks, or useful datasets
- Repos from credible maintainers with active recent development

**Rank lower:**
- Awesome lists or link collections without novel content
- Toy demos or tutorial projects aimed at beginners
- Repos with growth that appears inflated or manufactured

Select the top `results` repos. If fewer than `results` repos remain, select all of them.

Record `final_count` = actual number of repos selected.

---

### Step 6: Write the Digest

Create the output directory if it does not exist.

Write to `<output_dir>/github-trending-YYYY-MM-DD.md`.

**Header:**
```
# GitHub AI Trending — YYYY-MM-DD

> {fetch_count} repos fetched · {selected_count} passed hard signal filter · {final_count} selected by relevance
> Topics: {topics joined by ", "} · Time window: {time_window} days
```

If fewer than `results` repos passed the topic filter, add:
```
> ⚠ Only {topic_filtered_count} repos passed topic filter — digest is shorter than configured
```

After the header, add a blank line, then `---`, then another blank line.

**For each selected repo (numbered 1 to N):**

```
## {N}. {full_name}
**Stars:** {stars_total formatted with commas} · **{growth_label}:** +{stars_growth formatted with commas}
**Link:** {link}

**What it does:** {One clear sentence describing what the repo is and what problem it solves.}

**Worth your time?** {Yes / Maybe / Probably not} — {1-2 sentences: what you'd actually use it for, why it's interesting, or what type of engineer would care.}

---
```

After writing, print: `Digest saved to <full absolute path>`
