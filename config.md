# AI Digest Config

## Output
output_dir: ~/Documents/ai-digests

## Arxiv Digest
categories: cs.AI, cs.LG, cs.CL
results: 5           # final digest count
filter_size: 10      # pool after hard signal filter (range: 5–15)
fetch_pool_size: 25  # initial fetch width (range: 15–50)
lookback_days: 3     # range: 1–7

## GitHub Trending
topics: AI, LLM, machine-learning, agents
results: 5           # final digest count
filter_size: 10      # pool after hard signal filter (range: 5–15)
fetch_pool_size: 25  # initial fetch width (range: 15–50)
time_window: 3       # range: 1–7 days
