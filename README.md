# Coffy Farmer: Case Study

> Browser extension + admin panel for automating farming in WolvesVille, with ticket-based access control and real-time session monitoring.

---

## Overview

Coffy Farmer started as a browser extension to automate repetitive farming in WolvesVille. At some point it had enough paying users that I had to actually treat it as a product. That meant building the whole operation around it: access control, session tracking, subscription management, and a panel to run everything without jumping between tools.

**Users:** Paying players with subscription-based access  
**Stack:** Browser Extension · Supabase (PostgreSQL + Auth + Edge Functions) · Vanilla HTML/CSS/JS

---

## The Extension

![Coffy Farmer Extension](https://raw.githubusercontent.com/nduaarte/coffy-farmer/main/assets/extension.png)

The extension runs directly in the browser alongside the game. Users authenticate with a ticket code, choose their region and farm mode, and hit Start. The XP ranking updates in real time, showing where each player sits in the last 7 days. Ticket validation happens on load and shows the status and remaining days right in the UI.

---

## The Problem

The extension itself was the easy part. The harder problem was operating it: who has access, for how long, are they online right now, did they pay, did I deliver what they bought? None of that was solved by writing the extension alone.

The questions I had to answer:

- How do I control access without building a full auth system?
- How do I know who's online at any given moment?
- How do I track manual payments and deliveries without an e-commerce setup?
- How do I send Discord messages to users without leaving the admin panel?

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

Users authenticate via a **ticket code**: a short alphanumeric key with an expiry date. The extension validates it against Supabase on startup. No login screen, no password reset flow, no account creation.

---

## Key Technical Decisions

### 1. Ticket-based access instead of user accounts

Each paying user gets a unique code with an expiry date. Simple to generate, simple to revoke. The extension validates it on startup and blocks access if it's expired or invalid.

The downside is less data per user. The upside is that I can grant, revoke, or extend access in seconds from the panel, without touching user accounts or sending password resets.

### 2. Supabase as the entire backend

I used Supabase's REST API directly, no ORM, no separate server. Row Level Security handles data protection, Edge Functions handle anything that needs a secret (like the Discord token). Zero infrastructure to maintain.

It's less flexible than a custom API, but for a solo project this size, flexibility wasn't the bottleneck. Speed of shipping was.

### 3. `player_status` view for session monitoring

Instead of querying raw tables on every refresh, I created a Supabase view that aggregates everything: online status, last access, timezone. The admin panel polls it every 30 seconds and renders it as a live table.

The abstraction means I have to be careful when the underlying tables change, but it keeps the frontend query simple and fast.

### 4. Discord integration via Edge Functions

The Discord bot token never touches the frontend. The admin panel calls two Edge Functions (`send-dm` and `post-channel`) with the user's Supabase JWT. The function validates the session before hitting the Discord API.

Cold start adds a bit of latency, but the token stays server-side and every call is auditable.

---

## Admin Panel

![Coffy Farm Admin](https://github.com/nduaarte/coffy-farmer/blob/main/assets/admin_panel.png)

The panel is a single HTML file with no framework. Intentionally lean so it's fast to load and easy to change. The screenshot above shows 15 players online simultaneously, with 49 active sessions in the last 24 hours.

| Tab | What it does |
|---|---|
| **Sessions** | Live view of online players, last access time, timezone |
| **Tickets** | Create, manage, and expire access codes |
| **Subscriptions** | Track manual payments and item deliveries |
| **App Config** | Key-value config editor for extension behavior |
| **Discord** | Send DMs or post to channels directly from the panel |

A few decisions worth noting:

- Expired tickets are hidden by default and only loaded on demand. No reason to fetch history on every page load.
- Ticket codes have a hover-reveal copy button so they're easy to grab without selecting text.
- Auto-refresh runs every 30 seconds, but only on the active tab.
- Modals animate with pure CSS transitions. No animation library.

---

## Challenges

**Keeping the extension undetected.** The automation had to look like a real user. That meant randomizing timing between actions and simulating proper browser events rather than triggering them programmatically in ways the game could fingerprint.

**Manual subscription tracking.** Payments came in through Discord and PIX, outside the platform entirely. I needed a way to record who paid, for what, and confirm when the delivery happened. The Subscriptions tab handles this with a simple pending/complete flow and a timestamp on completion.

**State management without a framework.** Keeping everything in one HTML file was a deliberate choice for portability. But as features grew, managing state with module-level variables and explicit render functions got harder to follow. It works, but it's the part I'd revisit first.

---

## Results

- 49 active sessions in a single day, 15 simultaneous online users at peak
- Full visibility into live sessions across timezones
- Zero server costs (Supabase free tier covered the load)
- Discord communication handled entirely from the admin panel

---

## What I'd do differently

- Add an audit log for ticket operations. Right now there's no record of who created or modified what.
- Move the admin panel to a lightweight framework. Vanilla JS was fine to start, but state management started showing cracks as more tabs and flows were added.
- Replace polling with Supabase Realtime for session updates. Polling every 30 seconds works, but it's wasteful and adds unnecessary read volume.

---

*Source code is private. Happy to discuss architecture and decisions in more detail.*
