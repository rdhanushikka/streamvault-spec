# StreamVault — Product Specification

**Version:** 0.2  
**Date:** 2026-04-25  
**Status:** Draft

---

## 1. Problem Statement

Streaming content is fragmented across Netflix, Hulu, Disney+, HBO Max, Apple TV+, Amazon Prime, and dozens of others. Users currently:

- Maintain separate watchlists on each platform that don't talk to each other
- Lose recommendations from friends because there's no single place to log them
- Keep overlapping subscriptions active just in case, wasting money
- Have no way to plan subscriptions around when shows they care about are available

StreamVault solves all four problems in one place.

---

## 2. Core Concepts

| Term | Definition |
|---|---|
| **Watchlist** | A user's centralized list of titles they want to watch, regardless of platform |
| **Recommendation** | A title added with a note about who suggested it |
| **Availability** | Which streaming platforms currently carry a title |
| **Subscription Window** | A user-defined active period for a streaming service |
| **Subscription Advisor** | The AI module that suggests when to activate or pause a subscription |

---

## 3. Feature Set

### 3.1 Universal Watchlist

A single table that aggregates everything the user wants to watch.

**Columns:**
- Title
- Type (Movie / TV Series / Mini-series / Documentary)
- Genre(s)
- Year
- Status: `Want to Watch` | `Watching` | `Watched` | `Dropped`
- Rating (user-assigned, after watching)
- Recommended by (name/contact, optional)
- Recommendation note (free text — "You'll love the cinematography")
- Date added
- Priority: `High` | `Medium` | `Low`
- Available on (auto-populated from availability data)
- Leaving soon (flag, auto-populated)

**Behaviors:**
- Users can add titles manually by searching the app's content directory
- Users can share a watchlist with another user (e.g., a household)
- Titles can be filtered/sorted by any column
- Watched titles are archived but searchable

---

### 3.2 Content Directory

A comprehensive catalog of movies and TV shows. StreamVault operates as an intelligence layer on top of existing data sources rather than building its own catalog from scratch.

**Data layer — availability and catalog:**
- **TMDb API** — primary catalog source: title metadata, cast, crew, genres, ratings, trailers, upcoming season premiere dates
- **TMDb Watch Providers endpoint** — streaming availability by region (sources from JustWatch; official, free, no agreement needed for POC)
- Long-term: formal JustWatch data partnership as scale requires more granular or real-time availability data

**Intelligence layer — content understanding:**
- **Wikipedia API** — full plot summaries, thematic analysis, critical reception, cultural context per title
- **Wikidata** — structured facts queryable via SPARQL: awards, "based on" relationships, similar works, cinematic movements
- Both are ingested, cleaned, and indexed into a vector database to power the AI recommendation engine (see Section 3.3)

**Metadata per title:**
- Title, year, runtime, genres, cast, director(s)
- Synopsis (TMDb) + full plot and themes (Wikipedia)
- Trailer link (YouTube embed)
- Audience and critic scores
- Content rating (PG, R, TV-MA, etc.)
- Platform availability (real-time, by user's region)
- Leaving-soon flag (if known departure date exists)

---

### 3.3 Unified Recommendation Engine

Mood-based discovery and personalized recommendations are the same operation under the hood — a vector similarity search against the movie embedding space. They differ only in what generates the query vector. Both are handled by a single AI engine rather than two separate systems.

**The two query types:**

| Type | Query vector source | When used |
|---|---|---|
| Mood query | Encoded from user's free-text input right now | "I want something with a rainy Sunday feeling" |
| Preference query | Learned from user's watchlist, ratings, behavior over time | Background recommendations, homepage suggestions |

At inference time, both vectors are **blended** so results reflect the user's immediate mood *filtered through their long-term taste*. The blend weight is user-adjustable — a "surprise me" slider shifts weight toward the mood vector; "stick to my taste" shifts toward the preference vector.

**Example prompts:**
- "I want something funny but not stupid, like 30 minutes per episode"
- "I'm in the mood for a slow burn thriller — no jump scares"
- "Something my 10-year-old and I can watch together tonight"

**How it works:**
1. User submits free-text mood prompt
2. The custom movie embedding model encodes it into a vector (same embedding space as the movie corpus)
3. That mood vector is blended with the user's long-term preference vector
4. Nearest-neighbor search returns the most semantically similar titles from the movie index
5. Results are filtered and re-ranked by: availability on active subscriptions → priority → recency
6. Claude handles natural language in and out — understanding the prompt, explaining each result in plain language
7. Returns 3–6 results; user can add any directly to their watchlist

**Constraints:**
- Results biased toward active subscriptions first; toggle to show all
- Chatbot remembers prior session results to avoid repeating within a session
- Cold-start users (no preference history) get mood-only results until enough interaction data is collected

**See Section 5.1 for the full AI architecture underlying this engine.**

---

### 3.4 Platform Availability View

When a user clicks on any title, a detail panel shows:

- Full metadata
- **Where to watch:** a list of all platforms carrying the title, flagged against the user's subscriptions
  - Green badge: you have this subscription active now
  - Yellow badge: you have this subscription but it's paused
  - Gray: not subscribed — shows monthly cost
- If available on multiple active subscriptions, sorted by quality (4K > HD > SD)
- Deep-link button: opens the title directly in the platform's app or website (where platform deep-link URLs are available)
- "Leaving [platform] on [date]" warning if departure date is known

**Note on platform integration:**  
No major streaming platform (Netflix, Hulu, Disney+, HBO Max, etc.) currently offers a public OAuth API that exposes subscription status to third-party apps. This is a deliberate business decision on their part, not a technical limitation.

**Current approach — self-declaration:**  
During onboarding, the user is shown logo tiles for all major platforms and taps to indicate which ones they actively subscribe to. This takes under 30 seconds and gives StreamVault everything it needs to filter availability. The user can update their subscriptions at any time from the Subscription Manager.

**Long-term goal — direct platform integration:**  
As streaming platforms evolve their developer ecosystems, the goal is to replace self-declaration with direct OAuth connections. A simple read-only OAuth scope returning `{ subscribed: true/false }` would eliminate manual input and keep subscription status automatically in sync. StreamVault will pursue platform partnerships and monitor API availability as the product grows.

**Other approaches considered and ruled out:**  
- *Financial data APIs (Plaid):* Can auto-detect subscriptions by reading recurring charges on a linked bank or credit card. Ruled out for early stages — users are unlikely to connect financial accounts to a new, unproven app. Remains a viable option to revisit once user trust is established.  
- *Email scanning (Gmail API):* Can detect active subscriptions by scanning for billing confirmation emails. Ruled out due to the sensitivity of inbox access and the friction of Gmail authorization for a non-email product.

---

### 3.5 Subscription Manager

Users declare which streaming services they subscribe to and when.

**Per subscription record:**
- Service name
- Status: `Active` | `Paused` | `Cancelled`
- Monthly cost
- Billing date
- Active window (start date → end date, or indefinite)

**Features:**
- Dashboard showing monthly streaming spend across all active subscriptions
- Calendar view of subscription windows
- "Content expiring from paused services" alert — if something on your watchlist is leaving a service you've paused, you get a heads-up

---

### 3.6 Subscription Advisor

The most differentiated feature. The advisor analyzes the user's watchlist, subscription status, and content availability data to suggest when to activate or pause each subscription.

**Inputs:**
- User's watchlist (titles + priority)
- Current subscriptions and their status
- Platform availability data for each watchlist title
- Known premiere dates for upcoming seasons (sourced from TMDb)
- Known leaving dates for titles on platforms (where available)

**Outputs — example advisor suggestions:**
- "You have 14 titles on your watchlist available on HBO Max, but your subscription is paused. Season 3 of _The White Lotus_ drops May 18. Consider reactivating for May–June."
- "Your Peacock subscription has been active for 4 months. You've only watched 2 titles from your watchlist there. Consider pausing it — nothing priority is leaving soon."
- "_Severance_ Season 2 is on Apple TV+. Your watchlist has 8 Apple TV+ titles. A 1-month subscription ($9.99) would cover all of them."
- "3 titles on your watchlist are leaving Netflix in the next 30 days. You're already subscribed — now's a good time to watch them."

**Advisor logic:**
- Calculates "value score" per subscription: (watchlist titles available) × (priority weights) ÷ (monthly cost)
- Surfaces subscriptions with low value scores as candidates to pause
- Surfaces upcoming season premieres and content expiry dates as activation triggers
- Delivers suggestions as a weekly digest (push notification / email) or on-demand via the Advisor tab

---

## 4. User Flows

### Flow 1: Adding a Recommendation from a Friend
1. Friend says "Watch Severance, it's incredible"
2. User opens StreamVault → Watchlist → Add Title
3. Searches "Severance" → selects result
4. Fills in: Recommended by = "Sarah", Note = "it's incredible", Priority = High
5. Title is added; app shows it's on Apple TV+ (user currently subscribed)

### Flow 2: Mood-Based Discovery
1. User opens Chatbot tab
2. Types: "Something I can watch with my partner, not too heavy, maybe 2 hours"
3. App returns 4 suggestions, 2 available on active subscriptions (highlighted)
4. User taps one → sees detail view → clicks "Watch on Netflix" → Netflix opens to the title

### Flow 3: Subscription Planning
1. User opens Advisor tab
2. Advisor shows: "Hulu has 6 high-priority titles on your watchlist. _The Bear_ Season 5 premieres July 3."
3. User sees their Hulu subscription is currently paused
4. Taps "Plan Reactivation" → sets Hulu to reactivate July 1
5. Calendar updates; Advisor clears the suggestion

### Flow 4: Leaving-Soon Alert
1. User gets a notification: "Parasite is leaving Netflix in 7 days. It's on your watchlist."
2. User opens app → taps the alert → sees the title's detail view
3. Clicks "Watch on Netflix" → watches it

---

## 5. Technical Architecture (Proposed)

### 5.1 AI Engine

This is the technical core of StreamVault and the primary differentiator. It has three layers:

**Layer 1 — Movie Embedding Model**

A sentence transformer fine-tuned specifically on movie and TV discourse. General-purpose embedding models don't deeply understand cinematic language ("slow burn," "unreliable narrator," "feels like early Fincher"). By fine-tuning on a corpus of Wikipedia plot articles, Rotten Tomatoes reviews, Letterboxd reviews, and Reddit discussions (r/movies, r/TrueFilm), the model learns a vector space where movies that *feel* similar end up geometrically close — regardless of shared cast, director, or genre tags.

- Base model: a sentence transformer (e.g., `all-MiniLM-L6-v2` or similar)
- Fine-tuning objective: contrastive learning — pull together movies users consistently group together; push apart movies users distinguish
- Output: one vector per title, stored in a vector database
- All ~500K titles in the catalog are pre-embedded offline; new titles are embedded on ingest

**Layer 2 — Two-Tower Recommendation Model**

Two neural networks trained jointly:

```
Movie tower:  movie embedding (Layer 1) ──► movie vector
User tower:   watchlist + ratings + behavior ──► user preference vector
```

At inference time, recommendation = nearest-neighbor search: find movies whose vector is closest to the user's preference vector. Training the two towers jointly ensures the spaces align — a user who loves Korean thriller cinema ends up with a preference vector that's geometrically close to Korean thriller movie vectors.

The model improves continuously as users interact: items added to watchlist, ratings given, chatbot suggestions accepted or rejected, titles watched vs. dropped all feed back into the user tower.

**Layer 3 — Claude (Language In / Language Out)**

Claude handles the natural language boundary:
- Interprets the user's free-text mood prompt
- Extracts a mood description to encode via Layer 1
- Explains each recommendation result in a single natural sentence
- Handles follow-up conversational turns ("something shorter," "more like the second one")

Claude does not do the retrieval or ranking — that's Layers 1 and 2. Claude only handles language understanding and generation around the results.

**Unified query flow:**

```
User mood prompt
      │
      ▼
[Claude] extract mood intent
      │
      ▼
[Layer 1] encode mood → mood vector
      │
      ├──────────────────────────────┐
      │                              │
[Layer 2] user preference vector    │
      │                              │
      └──────────► blend ◄──────────┘
                     │
                     ▼
           nearest-neighbor search
           (vector database)
                     │
                     ▼
           filter by subscription availability
                     │
                     ▼
           [Claude] explain results in plain language
                     │
                     ▼
              3–6 recommendations
```

**Subscription Advisor AI:**

The advisor starts as rule-based (value score = watchlist titles × priority ÷ monthly cost) and evolves into a learned model that predicts the optimal subscribe/pause schedule. Inputs include watchlist priorities, content leaving/arriving dates (time-series), historical watch velocity per user, and price per subscription. This is a sequential optimization problem — the learned version models it as a constrained scheduling task.

---

### 5.2 System Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  Frontend (Web + Mobile)                  │
│   React (web) / React Native (iOS + Android)             │
│   Tabs: Watchlist | Discover | Chatbot | Advisor          │
└──────────────────────┬───────────────────────────────────┘
                       │ REST / GraphQL
┌──────────────────────▼───────────────────────────────────┐
│                   Backend API (FastAPI / Python)          │
│  - Auth                    - Watchlist CRUD               │
│  - Subscription manager    - Notification scheduler       │
│  - Recommendation engine   - Advisor engine               │
└───────┬──────────────────────────┬────────────────────────┘
        │                          │
┌───────▼───────┐       ┌──────────▼──────────────────────┐
│   PostgreSQL  │       │         AI / Data Layer          │
│  users        │       │  ┌─────────────────────────┐    │
│  watchlists   │       │  │  Vector DB (Pinecone /   │    │
│  subscriptions│       │  │  Qdrant) — movie embeddings   │
│  interactions │       │  └─────────────────────────┘    │
└───────────────┘       │  ┌─────────────────────────┐    │
                        │  │  Embedding model          │    │
                        │  │  (fine-tuned transformer) │    │
                        │  └─────────────────────────┘    │
                        │  ┌─────────────────────────┐    │
                        │  │  Two-tower model          │    │
                        │  │  (user + movie towers)    │    │
                        │  └─────────────────────────┘    │
                        │  ┌─────────────────────────┐    │
                        │  │  Claude API              │    │
                        │  │  (language in/out)        │    │
                        │  └─────────────────────────┘    │
                        └─────────────────────────────────┘
                                       │
                        ┌──────────────▼──────────────────┐
                        │         External Data APIs       │
                        │  - TMDb (catalog + watch provs.) │
                        │  - Wikipedia + Wikidata           │
                        │  - Notification: FCM + Resend    │
                        └─────────────────────────────────┘
```

**Key technology choices:**
- **Frontend:** React + React Native
- **Backend:** FastAPI (Python — aligns with ML stack)
- **Database:** PostgreSQL
- **Vector database:** Pinecone or Qdrant (stores movie embeddings for nearest-neighbor search)
- **Embedding model:** Fine-tuned sentence transformer (trained offline, served via API)
- **LLM:** Claude API (Sonnet) for language understanding and generation
- **Content data:** TMDb API (catalog + Watch Providers endpoint for availability)
- **Content intelligence:** Wikipedia API + Wikidata SPARQL
- **Notifications:** Firebase Cloud Messaging (push) + Resend (email)
- **Auth:** Supabase Auth or Auth0

---

## 6. MVP Scope

The MVP focuses on the core loop: add titles, see where to watch them, get subscription advice.

**Phase 1 — MVP (validate the product):**
- Universal watchlist (add, edit, status, recommended-by)
- Content search via TMDb + Watch Providers for availability
- Self-declared subscription onboarding (logo tile selection)
- Subscription manager (declare subscriptions, see monthly cost)
- Rule-based advisor (value score per subscription)
- Basic chatbot: Claude interprets mood → TMDb genre/keyword search → filtered by subscriptions
  *(no custom embeddings yet — validates whether users engage with the feature before investing in the ML stack)*

**Phase 2 — AI core (make it defensible):**
- Ingest Wikipedia + Wikidata into vector database
- Fine-tune sentence transformer on movie corpus → replace TMDb keyword search with semantic search
- Build user preference vectors from watchlist and rating interactions
- Deploy two-tower model; blend mood + preference vectors at inference
- Leaving-soon alerts and notifications

**Phase 3 — scale and polish:**
- Learned subscription advisor (sequential optimization model)
- Deep-link integration to streaming apps
- Household/shared watchlists
- Upcoming season premiere tracking
- Mobile apps (Phase 1–2 are web-first)
- Direct platform OAuth integration (as platforms open APIs)

---

## 7. Open Questions

1. **Availability data freshness:** JustWatch data can lag reality by days. How do we handle a user clicking "Watch on Netflix" only to find the title is gone?
2. **Platform deep links:** Deep-link URL formats vary by platform and region. Need to audit what's stable.
3. **Monetization:** Free tier vs. premium (advisor, notifications)? Subscription cost?
4. **Social features:** Should users be able to follow friends' watchlists, or keep this private-first?
5. **TV episode tracking:** Does the watchlist track at the show level, season level, or episode level for series?

---

## 8. Success Metrics

- Weekly active users returning to check watchlist
- % of watched titles that came through the chatbot
- Advisor suggestions acted on (subscription reactivations / pauses)
- Monthly streaming spend reduction reported by users
- Time from "heard about a show" to "added to watchlist" (should be < 30 seconds)
