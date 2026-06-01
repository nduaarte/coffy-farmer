# Coffy Farmer — Case Study

> A browser extension + admin dashboard ecosystem for automating farming in WolvesVille, with a ticket-based access system and real-time session monitoring.

---

## Overview

Coffy Farmer is a browser extension that automates farming actions in the online game WolvesVille. The project grew into a full product with paying users, requiring me to build an entire backend infrastructure around it — access control, session tracking, subscription management, and an admin panel to operate everything.

**Users:** Active paying players with subscription-based access  
**Stack:** Browser Extension · Supabase (PostgreSQL + Auth + Edge Functions) · Vanilla HTML/CSS/JS

---

## The Problem

WolvesVille players wanted to automate repetitive farming tasks without being detected or losing progress. Beyond the extension itself, the real challenge was **operating it as a product**:

- How do I control who has access and for how long?
- How do I know who's online and using the tool right now?
- How do I manage subscriptions and payments manually without a complex e-commerce setup?
- How do I communicate with users via Discord without leaving the admin panel?

---

## Architecture

```
┌─────────────────────┐     ┌──────────────────────┐
│  Browser Extension  │────▶│  Supabase (Backend)  │
│  (Content Script)   │     │  - PostgreSQL         │
└─────────────────────┘     │  - Auth (RLS)         │
                            │  - Edge Functions     │
┌─────────────────────┐     └──────────┬───────────┘
│    Admin Panel      │────────────────┘
│  (HTML/CSS/JS)      │
└─────────────────────┘
```

The extension authenticates users via a **ticket code** — a short alphanumeric key with an expiry date. This avoids the need for traditional login flows inside the extension and makes access easy to grant, revoke, or transfer.

---

## Key Technical Decisions

### 1. Ticket-based access instead of user accounts
Rather than building a full auth system for end users, I designed a ticket system: each paying user receives a unique code with an expiry date. The extension validates the code against Supabase on startup. This kept the UX simple for users and gave me full control over access management.

**Trade-off:** Less data per user, but simpler to operate and harder to share credentials (codes are single-use per session).

### 2. Supabase as the entire backend
I used Supabase's REST API directly (no ORM, no separate server), relying on Row Level Security (RLS) to protect data and Edge Functions for Discord integration. This meant zero server maintenance and near-instant deployment.

**Trade-off:** Less flexibility than a custom Node.js API, but dramatically faster to ship and maintain solo.

### 3. Real-time session visibility via `player_status` view
Instead of polling raw tables, I created a Supabase view that aggregates session data — online status, last access, timezone — into a single query. The admin panel polls this view every 30 seconds and renders it as a live dashboard.

**Trade-off:** The view adds a layer of abstraction that requires care when the underlying tables change, but it keeps the frontend clean and fast.

### 4. Discord integration via Edge Functions
Rather than exposing a bot token to the frontend, I built two Edge Functions (`send-dm`, `post-channel`) that act as a secure proxy. The admin panel sends requests with the user's Supabase JWT; the function validates it before touching the Discord API.

**Trade-off:** Adds a cold-start latency to Discord actions, but keeps secrets server-side and auditable.

---

## Admin Panel Features

The panel is a single HTML file with no framework — intentionally lean to keep it fast and easy to maintain.

| Tab | What it does |
|---|---|
| **Sessions** | Live view of online players, last access time, timezone |
| **Tickets** | Create, manage, and expire access codes |
| **Subscriptions** | Track manual payment/item deliveries with completion flow |
| **App Config** | Key-value config editor for the extension's behavior |
| **Discord** | Send DMs or post to channels without leaving the panel |

Notable UX decisions:
- Expired tickets are **hidden by default** and loaded on demand — reduces noise and unnecessary queries
- Copy-to-clipboard on ticket codes with a hover-reveal button
- Auto-refresh every 30 seconds when the Sessions or Tickets tab is active
- Animated modals with CSS transitions (no JS animation libraries)

---

## Challenges

**Keeping the extension undetected:** The automation had to mimic natural user behavior — timing randomization, event simulation — to avoid being flagged by the game.

**Manual subscription workflow:** Since payments were handled outside the platform (Discord, PIX), I needed a simple way to track who paid for what and confirm delivery. The Subscriptions tab was built specifically for this, with a one-click "Complete" action that timestamps the confirmation.

**Single-file admin panel:** Keeping everything in one HTML file was a constraint I set intentionally for portability and simplicity. Managing state without a framework required careful organization of module-level variables and explicit render functions.

---

## Results

- Multiple active paying subscribers
- Real-time visibility into active sessions across different timezones
- Zero server infrastructure costs (Supabase free tier handled the load)
- Fully operational Discord notification pipeline from the admin panel

---

## What I'd do differently

- Add proper logging/audit trail for ticket operations (currently no history of who created what)
- Move the admin panel to a lightweight framework (React or even Preact) as the feature set grew — vanilla JS state management started showing its limits
- Implement webhook-based session updates instead of polling to reduce Supabase read usage

---

*This is a case study — source code is private. Available to discuss architecture and decisions in detail.*
