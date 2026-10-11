# XOLO — Football Auction & League Experience

XOLO is a real-time multiplayer football (soccer) fantasy auction game. Players join a shared room using a 6-character room code, bid live against each other (and CPU managers) for a pool of real football players, build a squad within a budget, and then move into a simulated league season complete with fixtures, standings, live match commentary, and end-of-season awards.

The entire application — UI, game logic, and real-time multiplayer sync — lives in a **single HTML file**. There is no separate backend server; Firebase provides all backend functionality directly from the browser.

**Live app:** `https://anshchhetri.github.io/Xolo/`

---

## Features

- **Live online rooms** — new rooms use an 8-character cryptographically random invite code; existing 6-character room codes remain joinable
- **Real-time bidding auction** — humans and CPU-controlled managers bid on a rotating pool of players until each squad is full
- **Budget & squad management** — every manager starts with a fixed budget (₹100 Cr) and must fill an 11-player squad without overspending
- **Skip votes & auction timers** — managers can vote to skip a stuck auction or vote on a bidding timer
- **CPU-simulated auctions** — the entire auction can be simulated instantly if players don't want to bid manually
- **League phase** — once the auction ends, teams are auto-generated and a full fixture list is created
- **Matchday & full-season simulation** — simulate one matchday at a time or the entire rest of the season
- **Live match ticker & penalty shootouts** — head-to-head fixtures between human managers are animated live, including penalty shootouts on draws
- **Standings, top scorers, and season awards** — Golden Boot, Golden Glove, and Player of the Season, computed automatically from simulated results
- **Light/dark theme**, persisted per device
- **Presence & reconnection** — if a manager disconnects, the room reflects it in real time and their progress is restored if they rejoin

---

## Tech Stack

### Frontend
- **[React 18](https://react.dev/)** — loaded via CDN (`react` + `react-dom`, production UMD builds) with subresource integrity checks
- **[Babel Standalone](https://babeljs.io/docs/babel-standalone)** — compiles JSX directly in the browser at runtime (`<script type="text/babel">`), with a subresource integrity check
- **[Tailwind CSS](https://tailwindcss.com/)** — loaded from the original Tailwind browser CDN so the existing UI styling and layout remain unchanged
- **Custom CSS** — a small hand-written stylesheet for theme variables, fonts, and finer visual details (`Inter` and `Teko` from Google Fonts)
- **Plain JavaScript / JSX** — all game logic (auction rules, CPU bidding AI, fixture generation, match simulation, standings calculation, etc.) is hand-written, with no external state-management library

### Backend (Firebase)
No custom server exists — [Firebase](https://firebase.google.com/) is used entirely as a Backend-as-a-Service, called directly from the browser via the Firebase JS SDK (v12, modular, loaded as ES modules from `gstatic.com`):

- **Firebase Authentication** — anonymous sign-in, so every browser tab gets a unique, persistent identity without requiring accounts or passwords
- **Firebase Realtime Database** — the single source of truth for every room's live state, structured as:
  ```
  rooms/{roomId}/
    meta/        → room code, host UID, creation time
    members/     → who's connected, online status, presence
    state/       → the full game state (managers, budgets, squads, pool, current auction, fixtures, league data, logs, etc.)
    commands/    → one validated pending action per member, processed by the host

  roomCodes/{code}/  → maps a human-readable invite code to a roomId
  ```
- **Firebase Security Rules** — `database.rules.json` is the reviewed rules source for member-only room reads, invite-code-verified membership, host-only state writes, and validated one-slot commands. These rules must be published in Firebase Console separately from the static site.

### Multiplayer Architecture (Host-Authoritative Model)
XOLO uses a **host-authoritative** sync pattern instead of a traditional server:

1. Whoever creates the room becomes the **host**. All game logic (bidding validation, CPU AI, match simulation, etc.) runs *only* on the host's device.
2. The host continuously writes the full game state to `rooms/{roomId}/state` in Firebase whenever it changes.
3. Every other player (a "guest") only **listens** to `rooms/{roomId}/state` in real time via Firebase's `onValue` and re-renders their UI to match — they never compute game logic themselves.
4. When a guest wants to do something (place a bid, vote to skip, join as a manager), they don't modify the state directly. Instead, they write one small **command** to `rooms/{roomId}/commands/{memberUid}`. The host listens for new commands, validates and applies them, updates its own local state (which then syncs back out to everyone), and deletes the processed command so that member can send the next action.
5. **Presence** is handled with Firebase's `onDisconnect()`, so if a manager's tab closes or their connection drops, the room updates automatically.

This keeps the game consistent across every device without needing a dedicated game server — Firebase Realtime Database's low-latency sync is what makes it feel instant.

### Resilience
An **error boundary** wraps the whole React tree so that any unexpected rendering issue (e.g. an edge case in the synced data) shows a clear "Something went wrong" screen with a reload button, instead of silently leaving a blank page.

---

## Project Structure

```
index.html          → the full app: markup, styles, Firebase setup, and React/JSX game logic
database.rules.json → Firebase Realtime Database rules; publish separately in Firebase Console
firebase.json       → points Firebase CLI to the rules file
SECURITY_AUDIT.md   → review findings, validation, and remaining console actions
```

The app has no runtime build step: host `index.html` on any static server. Build tools and `node_modules` are not required to run it.

---

## Running Locally

1. Clone or download this repository.
2. Serve the folder with any static file server (opening the file directly via `file://` can cause issues with Firebase Auth), e.g.:
   ```bash
   python3 -m http.server 8000
   ```
3. Open `http://localhost:8000/index.html` in your browser.
4. Make sure `localhost` is listed under **Firebase Console → Authentication → Settings → Authorized domains**.

## Deployment

The live version is hosted for free on **GitHub Pages**, served directly from `index.html` at the repo root. Any static host (Netlify, Vercel, Firebase Hosting itself, etc.) would work identically since there's no server-side code.

Whichever domain the app is hosted on must be added to **Firebase Console → Authentication → Settings → Authorized domains**, or anonymous sign-in will fail.

Before deploying the game changes, publish the matching `database.rules.json` in **Firebase Console → Realtime Database → Rules** (or run `firebase deploy --only database --project projectfootball-bfe26` after authenticating Firebase CLI). The Firebase web `apiKey` in `index.html` is a public client identifier, not a private credential; restrict it to the game's domains and required Google APIs in Google Cloud Console, and enable Firebase App Check to reduce scripted abuse.

---

## Tools Used

**AI tools**
- [Manus AI](https://manus.im/) — initial app scaffolding and generation
- [Claude](https://claude.ai/) (Anthropic) — debugging, multiplayer sync fixes, and documentation
- [ChatGPT](https://chatgpt.com/) (OpenAI) — additional development assistance

**IDEs / Editors**
- [Visual Studio Code](https://code.visualstudio.com/)
- [Cursor](https://cursor.sh/)

**Core technologies**
- [React](https://react.dev/) — UI framework
- [Firebase](https://firebase.google.com/) — backend (Realtime Database + Authentication)
- [Tailwind CSS](https://tailwindcss.com/) — styling
- [Babel](https://babeljs.io/) — in-browser JSX compilation

---

## Known Limitations

- Anonymous sign-in remains public by design. The database rules restrict room access and command abuse, but Firebase App Check and Google API-key referrer/API restrictions should also be enabled to reduce automated abuse.
- Tailwind's original browser CDN is retained to preserve the design exactly; it remains an executable third-party dependency without Subresource Integrity. Replacing it safely would require matching the original runtime styling before switching to a local build.
- Because the host's device runs all game logic, if the host disconnects mid-game, the room currently has no automatic host handover.
- The host browser remains the authoritative game engine; a compromised host browser can still submit whatever state its own rules permit.
