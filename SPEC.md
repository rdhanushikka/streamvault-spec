# StreamVault — Product Specification

**Version:** 0.1  
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

A comprehensive, searchable catalog of movies and TV shows with metadata.

**Data sourced from:**
- Primary: TMDb (The Movie Database) API — free, comprehensive, well-maintained
- Availability data: JustWatch API or Streaming Availability API (RapidAPI) — maps titles to platforms by region
- Ratings: TMDb + Rotten Tomatoes (where available)

**Metadata per title:**
- Title, year, runtime, genres, cast, director(s)
- Synopsis
- Trailer link (YouTube embed)
- Audience and critic scores
- Content rating (PG, R, TV-MA, etc.)
- Platform availability (real-time, by user's region)
- Leaving-soon flag (if known departure date exists)

---

### 3.3 Mood-Based Chatbot

A conversational interface powered by an LLM (Claude API) that takes natural-language input about how the user is feeling and returns curated recommendations.

**Example prompts:**
- "I want something funny but not stupid, like 30 minutes per episode"
- "I'm in the mood for a slow burn thriller — no jump scares"
- "Something my 10-year-old and I can watch together tonight"

**How it works:**
1. User submits a free-text mood/preference prompt
2. LLM interprets mood, extracts genre signals, length preference, tone, audience
3. LLM queries the content directory (via function calling / RAG over the catalog) using those signals
4. Returns 3–6 ranked recommendations with a one-sentence explanation for each
5. Each result shows real-time availability on the user's active subscriptions
6. User can add any result directly to their watchlist

**Constraints:**
- Recommendations are biased toward titles available on the user's current active subscriptions first
- User can toggle "show me everything, even if I need a subscription"
- Chatbot remembers prior session recommendations within a conversation to avoid repeating

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

```
┌─────────────────────────────────────────────────────┐
│                   Frontend (Web + Mobile)            │
│   React (web) / React Native (iOS + Android)        │
│   Tabs: Watchlist | Discover | Chatbot | Advisor     │
└────────────────────┬────────────────────────────────┘
                     │ REST / GraphQL
┌────────────────────▼────────────────────────────────┐
│                  Backend API (Node.js / FastAPI)     │
│  - Auth (email + OAuth for social login)            │
│  - Watchlist CRUD                                   │
│  - Subscription manager                             │
│  - Chatbot orchestration (Claude API)               │
│  - Advisor engine                                   │
│  - Notification scheduler                           │
└──────┬──────────────────┬───────────────────────────┘
       │                  │
┌──────▼──────┐   ┌───────▼──────────────────────────┐
│  Database   │   │       External APIs               │
│  PostgreSQL │   │  - TMDb (catalog + metadata)      │
│             │   │  - JustWatch / Streaming Avail.   │
│             │   │  - Claude API (chatbot + advisor) │
└─────────────┘   └──────────────────────────────────┘
```

**Key technology choices:**
- **Frontend:** React + React Native (shared logic, separate UIs)
- **Backend:** FastAPI (Python) or Node.js/Express
- **Database:** PostgreSQL (relational — watchlists, subscriptions, users)
- **LLM:** Claude API (Sonnet) for chatbot and advisor text generation
- **Content data:** TMDb API (free tier is sufficient for MVP)
- **Availability data:** JustWatch Streaming Availability API or similar
- **Notifications:** Push via Firebase Cloud Messaging; email via Resend or Postmark
- **Auth:** Supabase Auth or Auth0

---

## 6. MVP Scope

The MVP focuses on the core loop: add titles, see where to watch them, get subscription advice.

**In MVP:**
- Universal watchlist (add, edit, status, recommended-by)
- Content search via TMDb
- Platform availability display (self-declared subscriptions via onboarding tile selection)
- Basic chatbot (mood → recommendations from TMDb catalog)
- Subscription manager (declare subscriptions, see monthly cost)
- Simple advisor (rule-based: highlight watchlist titles on subscribed platforms)

**Post-MVP:**
- Leaving-soon alerts and notifications
- Advanced advisor (ML-weighted value scoring)
- Deep-link integration to streaming apps
- Household/shared watchlists
- Upcoming season premiere tracking
- Mobile apps (MVP is web-first)
- Direct platform OAuth integration for automatic subscription detection (as platforms open APIs)

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
