---
name: social-signal-scraping
description: Scrape social signals (Reddit, Twitter/X, Product Hunt) about any company, product, or topic using TinyFish. Returns structured JSON with sentiment analysis, pain points, positive feedback, and trending topics.
---

# Social Signal Scraping — Multi-Platform Sentiment & Insights via TinyFish

Extract discussions, opinions, and sentiment about any company, product, or topic from Reddit, Twitter/X, and Product Hunt. Returns unified structured JSON with aggregated insights.

## Pre-flight Check (REQUIRED)

Before making any TinyFish call, always run BOTH checks:

**1. CLI installed?**
```bash
which tinyfish && tinyfish --version || echo "TINYFISH_CLI_NOT_INSTALLED"
```

If not installed, stop and tell the user:
> Install the TinyFish CLI: `npm install -g @tiny-fish/cli`

**2. Authenticated?**
```bash
tinyfish auth status
```

If not authenticated, stop and tell the user:

> You need a TinyFish API key. Get one at: https://agent.tinyfish.ai/api-keys
>
> Then authenticate:
>
> **Option 1 — CLI login (interactive):**
> ```
> tinyfish auth login
> ```
>
> **Option 2 — Environment variable (CI/CD):**
> ```
> export TINYFISH_API_KEY="your-key-here"
> ```
>
> **Option 3 — Settings file:** Add to your AI coding assistant's settings:
> ```json
> {
>   "env": {
>     "TINYFISH_API_KEY": "your-key-here"
>   }
> }
> ```

Do NOT proceed until both checks pass.

---

## Input

The skill accepts a single input:

```json
{
  "query": "string"   // company name, product, or topic
}
```

---

## Execution Plan

For a given query, execute these steps in order:

1. **Scrape all three platforms in parallel** (separate TinyFish calls)
2. **Parse and normalize** raw results into unified format
3. **Classify sentiment** for each entry
4. **Aggregate** into final structured output

---

## Step 1: Platform Scraping

Each platform MUST be scraped with its own TinyFish call. Run all three in parallel.

### Reddit

```bash
tinyfish agent run --sync --url "https://www.reddit.com/search/?q=<query>" \
  "Extract relevant discussions about '<query>' as a JSON array. For each discussion, capture: [{\"platform\": \"reddit\", \"title\": \"post title\", \"content\": \"post body or summary\", \"upvotes\": \"number as string\", \"subreddit\": \"subreddit name\", \"url\": \"post url\", \"timestamp\": \"date if available\", \"sentiment\": \"positive|negative|neutral\"}]. Focus on: complaints, comparisons to competitors, feature requests, and substantive discussions. Ignore memes, low-effort posts, and irrelevant results. Return up to 30 results, prioritizing posts with the most engagement. Return ONLY the JSON array, no other text."
```

### Twitter/X

```bash
tinyfish agent run --sync --url "https://twitter.com/search?q=<query>&src=typed_query" \
  "Extract tweets about '<query>' as a JSON array. For each tweet, capture: [{\"platform\": \"twitter\", \"content\": \"tweet text\", \"author\": \"@handle\", \"likes\": \"number as string\", \"retweets\": \"number as string\", \"timestamp\": \"date if available\", \"sentiment\": \"positive|negative|neutral\"}]. Ignore spam, ads, bot-like content, and promotional tweets. Prefer tweets with meaningful opinions, complaints, praise, or comparisons. Return up to 30 results, prioritizing tweets with the most engagement. Return ONLY the JSON array, no other text."
```

### Product Hunt

```bash
tinyfish agent run --sync --url "https://www.producthunt.com/search?q=<query>" \
  "Extract comments and discussions about '<query>' as a JSON array. For each entry, capture: [{\"platform\": \"producthunt\", \"product_name\": \"product name\", \"content\": \"comment or description text\", \"upvotes\": \"number as string\", \"url\": \"product or comment url\", \"timestamp\": \"date if available\", \"sentiment\": \"positive|negative|neutral\"}]. Focus on user reviews, maker responses, and substantive comments. Return up to 30 results. Return ONLY the JSON array, no other text."
```

### Retry Logic

If any TinyFish call fails, retry it exactly once. If it fails again, skip that platform and continue with partial results. Never block the entire skill on a single platform failure.

---

## Step 2: Parse and Normalize Results

After receiving raw output from TinyFish:

1. **Extract JSON** from the `resultJson` field of the TinyFish response (the COMPLETE event)
2. **Parse** the JSON array from each platform
3. **Normalize text**: trim whitespace, remove excess newlines, clean encoding artifacts
4. **Deduplicate**: remove entries with near-identical content (same core text across platforms)
5. **Filter**: remove entries with empty content, spam patterns, or very short content (< 10 characters)

---

## Step 3: Sentiment Classification

For each entry, if TinyFish did not already classify sentiment, apply keyword-based classification:

### Keyword-Based Sentiment Rules

**Positive indicators**: love, great, amazing, excellent, fantastic, best, awesome, helpful, impressive, recommend, easy, fast, reliable, solid, perfect, happy, pleased, smooth, intuitive, powerful

**Negative indicators**: hate, terrible, awful, worst, broken, slow, buggy, crash, frustrating, expensive, disappointing, useless, complicated, unreliable, poor, annoying, painful, confusing, lacking, bad

**Scoring**:
- Count positive and negative keyword matches in each entry's content
- If positive count > negative count: `"positive"` (score: `+0.5` to `+1.0` based on ratio)
- If negative count > positive count: `"negative"` (score: `-0.5` to `-1.0` based on ratio)
- If equal or no matches: `"neutral"` (score: `0.0`)

This classification is intentionally simple and designed to be replaceable with an LLM or ML model later. The sentiment field and score field should always be present in the output.

---

## Step 4: Aggregation

Combine all platform results into the final output structure.

### Insight Extraction

From the combined data:

- **pain_points**: Extract content from entries with negative sentiment. Summarize the top 5-10 recurring complaints or issues.
- **positive_feedback**: Extract content from entries with positive sentiment. Summarize the top 5-10 recurring praises.
- **trending_topics**: Extract the most frequently mentioned keywords/phrases across all entries (exclude common stop words). Return top 5-10 topics.

### Overall Sentiment Calculation

- Count positive, negative, and neutral entries across all platforms
- Overall sentiment = whichever category has the most entries
- If tied, default to `"neutral"`

---

## Output Format

Return ONLY this JSON structure. No markdown, no logs, no console output.

```json
{
  "query": "the original query string",
  "scraped_at": "ISO 8601 timestamp",
  "summary": {
    "overall_sentiment": "positive|negative|neutral",
    "sentiment_breakdown": {
      "positive": 12,
      "negative": 5,
      "neutral": 8
    },
    "total_mentions": 25,
    "platforms_scraped": ["reddit", "twitter", "producthunt"],
    "platforms_failed": []
  },
  "insights": {
    "pain_points": [
      "Users report frequent crashes on mobile",
      "Pricing is considered too high for small teams"
    ],
    "positive_feedback": [
      "Intuitive UI praised by multiple users",
      "Fast customer support response times"
    ],
    "trending_topics": [
      "pricing",
      "mobile app",
      "integrations",
      "customer support"
    ]
  },
  "data": [
    {
      "platform": "reddit",
      "title": "Post title",
      "content": "Post content or summary",
      "upvotes": "142",
      "subreddit": "technology",
      "url": "https://reddit.com/r/...",
      "timestamp": "2026-03-15",
      "sentiment": "negative",
      "sentiment_score": -0.7
    },
    {
      "platform": "twitter",
      "content": "Tweet text",
      "author": "@handle",
      "likes": "89",
      "retweets": "12",
      "timestamp": "2026-03-20",
      "sentiment": "positive",
      "sentiment_score": 0.8
    },
    {
      "platform": "producthunt",
      "product_name": "Product Name",
      "content": "Comment text",
      "upvotes": "5",
      "url": "https://producthunt.com/...",
      "timestamp": "2026-02-10",
      "sentiment": "neutral",
      "sentiment_score": 0.0
    }
  ]
}
```

---

## Implementation Reference

When implementing this skill in TypeScript, use this code structure:

### Core Utility

```typescript
async function runTinyFish(url: string, goal: string): Promise<any> {
  // Execute: tinyfish agent run --sync --url "<url>" "<goal>"
  // Parse the SSE output, find the COMPLETE event, extract resultJson
  // On failure: retry once, then return null
}
```

### Platform Scrapers

```typescript
async function scrapeReddit(query: string): Promise<SocialEntry[]> {
  const url = `https://www.reddit.com/search/?q=${encodeURIComponent(query)}`;
  const goal = `Extract relevant discussions about '${query}' as a JSON array...`;
  const raw = await runTinyFish(url, goal);
  return normalize(raw, "reddit");
}

async function scrapeTwitter(query: string): Promise<SocialEntry[]> {
  const url = `https://twitter.com/search?q=${encodeURIComponent(query)}&src=typed_query`;
  const goal = `Extract tweets about '${query}' as a JSON array...`;
  const raw = await runTinyFish(url, goal);
  return normalize(raw, "twitter");
}

async function scrapeProductHunt(query: string): Promise<SocialEntry[]> {
  const url = `https://www.producthunt.com/search?q=${encodeURIComponent(query)}`;
  const goal = `Extract comments about '${query}' as a JSON array...`;
  const raw = await runTinyFish(url, goal);
  return normalize(raw, "producthunt");
}
```

### Aggregator

```typescript
async function aggregateResults(allData: SocialEntry[]): Promise<AggregatedOutput> {
  // 1. Deduplicate entries
  // 2. Apply sentiment classification where missing
  // 3. Extract pain_points, positive_feedback, trending_topics
  // 4. Calculate overall sentiment
  // 5. Return structured output
}
```

### Main Entry Point

```typescript
async function getSocialSignals(query: string): Promise<AggregatedOutput> {
  // 1. Run all three scrapers in parallel
  const [reddit, twitter, producthunt] = await Promise.allSettled([
    scrapeReddit(query),
    scrapeTwitter(query),
    scrapeProductHunt(query),
  ]);

  // 2. Collect successful results, track failures
  const allData: SocialEntry[] = [];
  const failed: string[] = [];

  if (reddit.status === "fulfilled" && reddit.value) allData.push(...reddit.value);
  else failed.push("reddit");

  if (twitter.status === "fulfilled" && twitter.value) allData.push(...twitter.value);
  else failed.push("twitter");

  if (producthunt.status === "fulfilled" && producthunt.value) allData.push(...producthunt.value);
  else failed.push("producthunt");

  // 3. Aggregate and return
  return aggregateResults(allData, query, failed);
}
```

---

## Key Rules

- **Parallel scraping**: Always run all three platform scrapes simultaneously, never sequentially.
- **Separate TinyFish calls**: Never combine multiple platforms into one TinyFish call.
- **Always use --sync**: Every TinyFish call must use the `--sync` flag.
- **Graceful degradation**: If a platform fails, return partial results from the others.
- **JSON only**: Final output must be valid JSON. No markdown wrapping, no console logs, no extra text.
- **URL-encode the query**: Always encode the query parameter in URLs.
- **Limit results**: Cap at 20-30 entries per platform to keep output manageable.
- **Match user's language**: Respond in whatever language the user is writing in.
