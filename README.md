# ai-digests

Two Claude Code skills that deliver daily AI digests to your machine.

- `/arxiv-digest` — fetches, filters, and summarizes recent AI papers from arxiv
- `/github-ai-trending` — fetches, filters, and summarizes trending AI/ML repos from GitHub

Output is a dated markdown file saved to a local directory you configure. No APIs, no keys — uses public RSS feeds and GitHub's trending page.

## Setup

**1. Clone anywhere:**

```bash
git clone https://github.com/YOUR_USERNAME/claude-ai-digests
```

**2. Install config:**

```bash
mkdir -p ~/.claude/ai-digests
cp config.md ~/.claude/ai-digests/
```

Open `~/.claude/ai-digests/config.md` and set:
- `output_dir` — where digests are saved (default: `~/Documents/ai-digests`)
- `categories` — arxiv categories you care about (default: `cs.AI, cs.LG, cs.CL`)
- `topics` — keywords for GitHub filtering (default: `AI, LLM, machine-learning, agents`)
- Adjust `results`, `filter_size`, `fetch_pool_size`, `lookback_days`, `time_window` as needed

**3. Copy skills to Claude Code:**

```bash
cp -r skills/arxiv-digest ~/.claude/skills/
cp -r skills/github-ai-trending ~/.claude/skills/
```

**4. Run in Claude Code:**

```
/arxiv-digest
```
or
```
/github-ai-trending
```

Digests are saved to `output_dir` as `arxiv-digest-YYYY-MM-DD.md` and `github-trending-YYYY-MM-DD.md`.

## Config Reference

| Key | Section | Default | Description |
|-----|---------|---------|-------------|
| `output_dir` | Output | `~/Documents/ai-digests` | Where digest files are saved |
| `categories` | Arxiv Digest | `cs.AI, cs.LG, cs.CL` | Arxiv category codes |
| `topics` | GitHub Trending | `AI, LLM, machine-learning, agents` | Keywords to filter repos |
| `results` | Both | `5` | Papers/repos in the final digest |
| `filter_size` | Both | `10` | Pool size after hard signal filter |
| `fetch_pool_size` | Both | `25` | Initial fetch width |
| `lookback_days` | Arxiv Digest | `3` | Max age of papers (1–7) |
| `time_window` | GitHub Trending | `3` | Trending time window in days (1–7) |

## How It Works

Both skills follow the same three-stage pipeline:

1. **Fetch wide** — pull `fetch_pool_size` candidates from the source
2. **Hard signal filter** — narrow to `filter_size` using objective signals (recency, lab affiliation, keyword density for arxiv; star growth and total stars for GitHub)
3. **Claude relevance ranking** — score remaining candidates on practical engineering value, select top `results`

## Out of Scope

- Automatic scheduling (trigger manually)
- LinkedIn post generation
- API keys
