---
title: AI-Powered X Engagement Digest with n8n
description: A production-style n8n workflow that turns a noisy X feed into a focused, AI-assisted engagement brief.
date: 2026-08-25
tags:
  - n8n
  - Claude
  - PostgreSQL
  - Telegram
featured: true
---
# AI-Powered X Engagement Digest with n8n

I built this n8n workflow to solve a small but persistent problem: 
keeping up with high-signal conversations on X without treating the feed like a full-time job.

The workflow checks posts from accounts I follow, filters them to the relevant time window, uses an LLM to identify genuine reply opportunities, and sends a concise Telegram digest with suggested angles and draft replies. It also logs each evaluated post to PostgreSQL to avoid duplicate recommendations and create an audit trail.

This is less about automating social posting and more about building an AI-assisted decision system: one that reduces scanning time while preserving human judgment and authenticity.

## The problem

My X feed includes researchers, AI practitioners, security professionals, builders, and technical writers. The posts worth responding to are often time-sensitive, but manually finding them requires repeatedly scanning a high-volume timeline.

I wanted a system that could answer a narrower question several times a day:

> Which recent posts from people I follow are genuinely worth responding to, and what is a useful angle I could add?

The system had to avoid two common failure modes:

1. Sending too much noise and recreating the feed in Telegram.
2. Producing generic AI replies that sound automated rather than informed.

## What the workflow does

At scheduled intervals, the workflow:

1. Retrieves or reuses a cached list of X accounts I follow.
2. Builds X Recent Search queries in batches, staying under query-length limits.
3. Fetches recent original posts, excluding reposts.
4. Deduplicates posts against a persisted list of previously seen IDs.
5. Sends the new posts to Claude for ranking and reply drafting.
6. Delivers a concise Telegram digest.
7. Writes each evaluated post, metrics, recommendation status, and suggested reply to PostgreSQL.

```text
Schedule / manual run
        |
        v
Following-list cache ----> X API: following accounts
        |
        v
Batch query builder -----> X API: Recent Search
        |
        v
Extract + deduplicate new posts
        |
        +---------------------> PostgreSQL audit log
        |
        v
Claude: rank + draft replies
        |
        v
Telegram engagement digest
```

## Scheduling design

The workflow uses different lookback windows depending on how it starts:

| Execution | Lookback | Why |
| --- | ---: | --- |
| Manual run | 12 hours | Lets me catch up after time away. |
| Scheduled 6:30am SGT run | 8 hours | Covers the overnight period. |
| Scheduled runs from 7:30am onward | 70 minutes | Covers the hourly interval with a 10-minute overlap for API/scheduling drift. |

The workflow timezone is explicitly set to `Asia/Singapore`. 
That matters: cron schedules should describe business time, not the timezone of an arbitrary server or container.

## Key engineering decisions

### Cache the following list

The workflow refreshes my following list at most once per day and stores it in n8n workflow static data. Every later run uses the cached handles.

This reduces unnecessary API calls while still keeping the monitored account list current.

### Batch search queries safely

X search queries are built as grouped `from:username` clauses. T
he workflow measures each candidate query before adding it to a batch, keeping a conservative buffer below the API query-length limit.

```text
(from:account_a OR from:account_b OR from:account_c) -is:retweet
```

This allows the workflow to scale beyond a small hand-maintained list of accounts.

### Use overlapping windows plus durable deduplication

An hourly workflow that queries exactly the previous hour can miss posts because of scheduler delay, API indexing delay, or network retries. I use a 70-minute query window and maintain a rolling list of seen post IDs.

The overlap makes ingestion more reliable; deduplication prevents the same post from appearing in subsequent digests.

### Treat LLM output as untrusted integration data

The LLM step returns structured picks containing the source-post index, a reason, a reply angle, and a draft comment. The parser is defensive:

- It locates text content even when the model returns multiple content blocks.
- It tolerates JSON wrapped in Markdown fences or surrounding text.
- It validates that a `picks` array exists before using it.
- It reports a parse problem clearly instead of silently turning a malformed response into “no recommendations.”

This was an important reliability lesson. A model integration should have the same validation and observability as any other external API.

### Keep the human in the loop

The workflow drafts replies; it does not publish them. Telegram is the review surface where I decide whether an idea is accurate, useful, and written in my voice before engaging.

That design preserves agency and avoids low-quality automated engagement.

## The AI evaluation layer

The LLM is prompted to identify *reply opportunities*, not merely viral or exceptional posts. It is instructed to choose three suitable posts when possible and to look for specific ways to contribute:

- a concrete technical observation, caveat, or example;
- a non-obvious follow-up question;
- a concise, well-grounded counterpoint; or
- a witty but substantive perspective.

Each recommendation includes:

```json
{
  "index": 1,
  "reason": "Why this post is worth engaging with",
  "angle": "The perspective I can add",
  "draft_comment": "A short, specific reply draft"
}
```

The prompt explicitly rejects generic praise. The goal is not to maximize replies; it is to make it easier to spot contributions that are genuinely worth making.

## Stack

| Layer | Technology |
| --- | --- |
| Orchestration | n8n |
| Social data | X API v2 Recent Search and Following endpoints |
| AI ranking and drafting | Anthropic Claude API |
| Notification and review | Telegram Bot API |
| Persistence and audit log | PostgreSQL |
| Runtime state | n8n workflow static data |

## Example digest

```text
X Digest — last about 70 min
11 new posts from your list

Worth engaging with:

1. @example_author: [post excerpt]
   Why: Opens a useful discussion about operational trade-offs.
   Angle: Add a practical SecOps perspective on the rollout risk.
   Draft: The rollout path matters as much as the capability here—...
```

## Challenges and lessons learned

### A zero-result search is not necessarily a failure

The X API returns `result_count: 0` for a valid query with no matches. It is important to distinguish that expected outcome from an authentication, syntax, or workflow-routing error.

### Workflow wiring deserves automated-style checks

During iteration, renamed nodes can leave stale connections in an exported workflow. 
I validated that each connection target resolves to an actual node before treating the export as deployment-ready.

### Prompt wording can create false negatives

An early version of the prompt emphasized skipping weak posts and allowed “up to five” choices. The model often returned zero recommendations even when the feed contained worthwhile discussion opportunities.

The fix was not simply changing models. 
It was clarifying the decision policy: select three opportunities when they exist, define what a useful contribution looks like, and require a specific angle for every draft.

### Content context matters

Text, metrics, and a post URL are a useful starting point, but they do not fully capture images, quoted posts, threads, or linked articles. A future iteration could enrich the prompt with thread context, quote-post text, article metadata, and media descriptions before ranking.

## Future improvements

- Add an explicit scoring pass before drafting, using a reply-opportunity score and confidence level.
- Include thread, quote-post, article, and media context in the LLM input.
- Build a feedback loop from posts I actually reply to, so the ranking policy learns my preferences.
- Track outcomes such as replies sent, likes, follow-up conversation, and profile visits.
- Compare models through controlled A/B tests using the same post batches and human-rated recommendations.
- Add a lightweight approval action in Telegram for saving high-quality drafts to a content queue.

## Why this project matters

This project demonstrates the kind of AI automation I find most useful: 
systems that combine APIs, scheduling, state management, data persistence, and LLM reasoning to support a real workflow.

The value is not “AI writes posts for me.” 

The value is a reliable pipeline that helps me spend attention where I can contribute something meaningful.

----

Interact with me at
X - https://www.X.com/LANC3lee
BlueSky - https://memepapa.bsky.social
Telegram - https://t.me/AIvampires (Group for projects)
Discord - https://discord.gg/89vqJWaCet
Linkedin - https://www.linkedin.com/in/lanc3/