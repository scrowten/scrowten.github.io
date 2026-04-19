---
layout: post
title: "🔭 GitSanity: arxiv-sanity, but for GitHub"
date: 2026-04-19 00:00:00
description: I built a personalized GitHub repo discovery engine that turns your own stars into a recommendation profile — so the good repos find you.
tags: github python fastapi nextjs recommendation-engine open-source
categories: machine-learning self-hosting
toc:
  sidebar: left
---

*You have 400 stars on GitHub. Somewhere in those 400 repos is the fingerprint of exactly what kind of engineer you are — your languages, your interests, your taste. Yet every time you open GitHub Trending, you get a Rust game engine or a JavaScript bundler you will never use. The signal is there. It is just pointing the wrong direction.*

---

# 🧠 1. The Inspiration: arxiv-sanity

In 2015, Andrej Karpathy built [arxiv-sanity](https://arxiv-sanity-lite.com). The problem: 200+ new ML papers posted to arxiv every single day. No one can read 200 papers a day. But the papers you *would* want to read form a pattern — and that pattern can be learned.

His solution was elegant: let users star papers they liked, extract TF-IDF vectors from abstracts, and serve a personalized ranked feed. The papers found you instead of you searching for them.

GitHub has the same problem, at a different scale. There are 420 million+ public repositories. GitHub Trending shows you what has the most star velocity *today*. That is not the same as what is relevant to *you*. A Python ML engineer and a Rust systems programmer should see completely different trending feeds. They do not.

[GitSanity](https://github.com/scrowten/gitsanity) is my attempt to build that layer — arxiv-sanity, but for GitHub repos.

---

# 🔍 2. The Problem With GitHub Discovery

Before getting into how GitSanity works, it is worth being precise about what is broken.

**GitHub Trending measures acceleration, not fit.** It shows repos gaining stars fastest over 24 hours, 7 days, or 30 days. This is useful for keeping up with the zeitgeist. It is not useful for finding a great async job queue library in your preferred language that has been quietly excellent for three years and does not need to trend.

**GitHub Search is pull-based.** You have to know what you are looking for. "async task queue python" returns 12,000 results with no personalization signal. The repos you would love most are buried on page 8.

**Awesome lists are manually maintained and static.** They reflect the curator's taste, not yours. They do not update when you develop new interests.

**Star counts are increasingly unreliable.** Research published in 2024–2025 identified 6 million suspected fake stars across 18,617 repositories. By mid-2024, roughly 16% of repos with 50+ stars showed signs of coordinated star campaigns. A 2,000-star repo is not necessarily a good repo.

**The signal that already exists, unused:** your own GitHub stars. You have been curating a preference profile for years. Every time you starred something, you were implicitly voting for a set of languages, topics, and interests. GitSanity reads that signal and turns it into recommendations.

---

# ⚙️ 3. How It Works

The flow is deliberately frictionless:

1. Log in with GitHub OAuth (read-only scope — starred repos only, no write access)
2. In the background, your starred repos are imported and a preference profile is built automatically
3. You see a personalized feed of repos ranked against your profile, each with a plain-English reason ("matches your interest in Python, LLM tooling")
4. Save repos you want to revisit. Dismiss ones that do not fit. Both signals feed back into future ranking.

No onboarding questionnaire. No manual tag selection. Zero friction from login to useful feed.

### The recommendation algorithm

The core scoring function in `recommender.py` combines three signals:

```python
score = lang_score * 0.4 + topic_score * 0.4 + keyword_score * 0.2

if repo_updated_within_90_days:
    score *= 1.2   # freshness boost

if repo.stars < 10:
    continue       # quality floor
```

**Language score (40%):** The preference builder counts your starred repos by primary language and normalizes by total repos analyzed — so if 60% of your stars are Python, Python gets weight 0.60 and a Python repo scores higher.

**Topic score (40%):** GitHub topics (those blue pills on repo pages: `machine-learning`, `cli`, `async`, etc.) are aggregated across all your stars and weighted. A repo whose topics overlap with your high-weight topics scores proportionally higher.

**Keyword score (20%):** Description text from your starred repos is tokenized and weighted. If "embedding", "vector", and "retrieval" appear frequently in repos you have starred, a new RAG library whose description uses those words will rank higher.

**Freshness multiplier:** A repo updated in the last 90 days gets a 20% score boost. This prevents excellent-but-abandoned projects from flooding the feed.

**Diversification:** A hard cap of 3 repos per GitHub owner in any result batch prevents any single prolific developer from dominating your feed.

### Preference profile building

```python
# Simplified from preference.py
for repo in user_starred_repos:
    if repo.language:
        lang_counts[repo.language] += 1
    for topic in repo.topics:
        topic_counts[topic] += 1
    for word in tokenize(repo.description):
        keyword_counts[word] += 1

# Normalize by total repos, not by max count
total = len(user_starred_repos)
lang_weights = {k: v/total for k, v in lang_counts.most_common(50)}
```

Normalizing by total (not by max count) is a deliberate choice. It means language weights are absolute interest fractions, not relative rankings — a user with 50% Python and 30% Go gets meaningfully different scores than one with 90% Python and 5% Go.

---

# 🏗️ 4. The Stack

GitSanity is a full-stack application with a clean separation between backend and frontend.

**Backend — FastAPI + Python:**

```
fastapi, uvicorn[standard]        # HTTP server
sqlalchemy[asyncio] + asyncpg    # async ORM + PostgreSQL driver
alembic                          # DB migrations
pydantic v2                      # settings + request/response schemas
httpx                            # async GitHub API client
python-jose[cryptography]        # JWT for session cookies
slowapi                          # rate limiting (30 req/min on feed endpoints)
```

The backend is structured as three routers (`auth`, `feed`, `saved`) and four services (`auth`, `github`, `preference`, `recommender`). Authentication uses GitHub OAuth 2.0 with JWTs stored in HTTP-only cookies — no localStorage, no XSS surface for tokens.

**Frontend — Next.js 16 + TypeScript:**

```
Next.js 16 App Router
Tailwind CSS v4
TanStack Query v5     # server state, caching, background refetch
Axios                 # API client
```

The UI has three pages: landing, feed, and saved. The feed page has a language filter bar — pill buttons showing your top languages with percentage weights, toggleable to narrow the feed. Skeleton loaders and toast notifications round out the polish.

**Infrastructure:**

```
PostgreSQL 16 (database)
Docker + Docker Compose (local dev + self-hosted)
Railway (backend deploy)
Vercel (frontend deploy)
```

The project ships with both a `docker-compose.yml` for self-hosting and a `railway.toml` for one-click Railway deploys. Two real deployment stories, both documented.

---

# 🆚 5. How GitSanity Compares

The GitHub discovery space is not empty. Here is where GitSanity fits:

| Tool | Approach | Personalized? |
|------|----------|---------------|
| GitHub Trending | Star velocity | No |
| Trendshift | Stars + Reddit/HN engagement | No |
| GitHunt | Curated trending by period | No |
| Awesome Lists | Manual curation | No |
| GitRec | Collaborative filtering (browser extension) | Yes |
| **GitSanity** | Your stars → profile → content scoring | **Yes, zero-friction** |

GitRec is the closest prior art — it uses collaborative filtering and injects recommendations into the GitHub homepage via a browser extension. GitSanity differs in two ways: it is a standalone app (no extension required), and its recommendation engine is content-based first (what *you* like) rather than collaborative first (what people *like you* like). Both approaches are valid; they will likely be combined in a future version.

---

# 🗺️ 6. Roadmap

The current version establishes the foundation — OAuth, preference profiles, content-based scoring, basic UI. Here is where it goes from here:

**Phase 2 — Smarter Personalization:**

- **Collaborative filtering** — "users who starred 40%+ of what you starred also starred X." The `starred_repos` table already exists; this is an algorithm addition, not a schema change.
- **Temporal weighting** — your interests shift over time. Stars from 6 months ago matter less than stars from last week. Weighting by recency will meaningfully improve recommendation quality.
- **Explicit feedback loop** — let users rate their topic/language weights from a profile page, not just via implicit save/dismiss signals.
- **Weekly email digest** via Resend — a curated "top 5 new repos matching your profile" delivered weekly without opening the app.
- **Fake star filtering** — integrate public fake-star detection datasets to filter repos with suspected coordinated campaigns. With 16% of 50+ star repos affected in 2024, this improves feed quality measurably.

**Phase 3 — Discovery Expansion:**

- **"More like this"** — click any repo card to get a filtered feed of similar repos. Useful for deep-diving a specific interest area.
- **Semantic similarity via embeddings** — embed repo descriptions with a small sentence-transformer model and score by cosine similarity to the centroid of your starred repo embeddings. This catches repos in unfamiliar languages whose *content* matches your interests even if the language does not.
- **"Why this" drill-down** — click the recommendation reason to see exactly which of your stars drove the suggestion. Increases trust and helps users refine their profile.
- **Browser extension** — inject a personal affinity score on any GitHub repo page ("82% match — based on your Python/LLM interests") without leaving GitHub. The highest-leverage distribution play in the roadmap.

**Phase 4 — Team and Integration Features:**

- **GitHub Actions integration** — a `gitsanity-report` Action that posts a weekly recommendation digest as a GitHub issue or Slack message. Zero-friction for teams.
- **Org/team mode** — aggregate stars from all members of a GitHub org to build a shared team preference profile. Useful for engineering leads evaluating new dependencies or monitoring the ecosystem.
- **Public API** — let power users build their own tools on top of the preference and recommendation layer.
- **Periodic star re-sync** — a background cron job that re-syncs stars weekly, keeping the preference profile fresh without any user action.

---

# 🚀 7. Running It Locally

```bash
git clone https://github.com/scrowten/gitsanity
cd gitsanity

# Copy env file and fill in your GitHub OAuth app credentials
cp .env.example .env

# Start everything (PostgreSQL + backend + frontend)
docker compose up --build
```

Frontend: `http://localhost:3000`
Backend API: `http://localhost:8000`
API docs: `http://localhost:8000/docs`

You will need a GitHub OAuth App — create one at [github.com/settings/developers](https://github.com/settings/developers) with callback URL `http://localhost:8000/auth/callback`.

---

*The repos you want to know about are out there. They just need a better way to find you.*

*Source: [github.com/scrowten/gitsanity](https://github.com/scrowten/gitsanity)*
