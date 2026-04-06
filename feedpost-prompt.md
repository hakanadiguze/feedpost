# FeedPost — Claude Code Build Prompt

## Overview

Build **FeedPost**, a real-time visual feedback collection tool that is part of a larger Agile platform. FeedPost allows session admins to create feedback boards where participants post sticky notes (post-its), rate questions on a 1–5 emoji scale, and upvote each other's ideas — all in real time across any device.

---

## Tech Stack

- **Single HTML file** — no build step, no framework, no npm
- **Firebase Realtime Database** — cloud storage + live sync via `onValue`
- **Firebase Authentication** — Email/Password (username mapped to `username@feedpost.app`)
- **Vanilla JS (ES modules)** — imported directly from CDN
- **CSS custom properties** — no Tailwind, no external CSS frameworks
- **Google Fonts** — Syne (700, 800) + DM Sans (400, 500)
- **Google Analytics** — gtag.js

---

## Design System

### Colors
```css
--or: #FF6B35      /* orange primary */
--or2: #FF8C5A     /* orange hover */
--or3: #FFF0EA     /* orange pale bg */
--or4: #FFD4C2     /* orange border */
--tl: #00C9B8      /* teal primary */
--tl2: #33D9CA     /* teal hover */
--tl3: #E0FAFA     /* teal pale bg */
--tl4: #009E91     /* teal dark */
--dk: #1A1A2E      /* dark bg */
--dk2: #252540     /* dark bg 2 */
--md: #7070A0      /* muted text */
--lt: #F5F5FC      /* light bg */
```

### Typography
- **Syne 700/800** — headings, logo, session names, numbers
- **DM Sans 400/500** — body text, buttons, form inputs

### Shadows
```css
--s1: 0 2px 8px rgba(26,26,46,.07)    /* card */
--s2: 0 8px 28px rgba(26,26,46,.12)   /* hover */
--s3: 0 24px 64px rgba(26,26,46,.2)   /* modal/overlay */
```

### Border radius
- `--r: 14px` — cards, modals
- `--r2: 10px` — buttons, inputs, badges

---

## Firebase Setup

```javascript
const cfg = {
  apiKey: "...",
  authDomain: "...",
  databaseURL: "...",  // Realtime Database URL (europe-west1 region)
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "..."
};
```

Import via CDN (ES module, version 10.12.0):
```javascript
import { initializeApp } from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js';
import { getDatabase, ref, set, get, push, update, onValue, off, remove }
  from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-database.js';
import { getAuth, createUserWithEmailAndPassword, signInWithEmailAndPassword, signOut, onAuthStateChanged }
  from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js';
```

Expose everything on `window._FP` so the non-module `<script>` block can use them. Dispatch `document.dispatchEvent(new Event('fp-ready'))` after setup.

### Firebase Database Rules
```json
{
  "rules": {
    "admins": {
      "$username": {
        ".read": "auth != null",
        ".write": "auth != null"
      }
    },
    "sessions": {
      ".read": true,
      ".write": true
    }
  }
}
```

### Username → Email mapping
Admins don't register with email — they pick a username. Map it internally:
```javascript
function toEmail(u) {
  return u.toLowerCase().replace(/[^a-z0-9]/g, '') + '@feedpost.app';
}
```

---

## Database Schema

```
/admins/{username}/
  username: string
  uid: string          // Firebase Auth UID
  createdAt: number    // timestamp

/admins/{username}/sessions/
  {pushKey}: string    // session code e.g. "HAUY"

/sessions/{code}/
  code: string         // 4-char alphanumeric e.g. "AB3X"
  name: string
  desc: string
  isPublic: boolean    // false = participants only see own posts
  nameReq: boolean     // true = name required to join
  questions: [         // array, can be empty (free-form board)
    { id, text, type: "text" | "rating" }
  ]
  feedbacks/
    {qid or "general"}/
      {pushKey}/
        id: string
        author: string | null
        text: string
        ts: number
        upvotes: number
        rot: string    // CSS rotation e.g. "-2.3deg"
  ratings/
    {qid}/
      1: number        // count of 1-star votes
      2: number
      3: number
      4: number
      5: number
  open: boolean
  createdAt: number
  owner: string        // username
```

---

## Application Screens

The app is a **single HTML file with 5 screens** (divs with `display:none` / `display:flex`). Only one screen visible at a time. No routing library.

```
#home          — Landing page
#admin-login   — Admin sign in / register
#admin-dash    — Admin dashboard (session list + create + detail)
#join-screen   — Participant join via code
#fb-screen     — Live feedback board (participant view)
```

---

## Screen 1: Home

**Layout:** Dark gradient background (`--dk` to `--dk2`). Centered hero section. Two decorative blur blobs (orange + teal, `filter: blur(90px)`).

**Content:**
- Logo: "Feed**Post**" (Post in teal) with orange gradient icon (SVG with two rectangles representing post-its)
- Topbar: logo left, "Built by Hakan" byline + "Admin Login" button right
- Hero eyebrow: "🟠 Real-time Feedback Platform" in teal uppercase
- H1: "Stick ideas like *post-its*" (*post-its* in orange)
- Subtext describing the platform
- Two CTA buttons: "Join a Session" (orange) + "Admin Panel" (teal)
- **How to Use** — collapsible section with 5 tabs: Participants, Admins, Post-its, Ratings, Upvotes. Each tab has numbered steps explaining how that feature works.
- Footer: "Built with ❤️ by **Hakan**"

---

## Screen 2: Admin Login

**Two forms** toggled by JS (not separate screens):
- **Sign In**: username + password → `signInWithEmailAndPassword(auth, toEmail(u), p)`
- **Register**: username + password + confirm → `createUserWithEmailAndPassword(auth, toEmail(u), p)` then write `admins/{username}` record. Wait 500ms after auth creation before DB write to ensure token is ready. Handle `PERMISSION_DENIED` gracefully (auth account created successfully even if DB write fails — user can sign in and DB record will be created then).

**Validation:**
- Username min 3 chars, password min 6 chars
- Passwords must match on register

**Enter key navigation:** username → password → submit, all with `onkeydown` handlers.

**On sign in:**
1. Call Firebase Auth sign in
2. Check if `admins/{username}` record exists; create it if missing (recovery for accounts where DB write failed during registration)
3. Navigate to admin dashboard

---

## Screen 3: Admin Dashboard

Three views inside one screen, toggled by JS:

### 3a. Session List (`#v-list`)
- Orange banner bar with welcome message + "New Session" button
- Grid of session cards (CSS grid, `repeat(auto-fill, minmax(300px, 1fr))`)
- **Session recovery**: On load, also scan all `sessions/` for `owner === S.admin` and merge with stored list (re-links orphaned sessions)
- Each card shows: name, date, code, Open/Closed badge, feedback count, question count
- Card footer buttons: **View**, **Close/Reopen**, **Copy Link**, **Delete** (with confirm dialog)
- Active (open) sessions have teal border

### 3b. Create Session (`#v-create`)
- Session name (required), description (optional)
- Two toggles: "Feedbacks are public" (default ON), "Name required" (default OFF)
- Question builder: add **Text Question** (post-it answers) or **Rated Question** (emoji 1–5). Questions can be removed. No questions = free-form board.
- On create: generate unique 4-char code (alphanumeric, no ambiguous chars: `ABCDEFGHJKLMNPQRSTUVWXYZ23456789`), save to Firebase, push code to `admins/{username}/sessions`, navigate to detail view

### 3c. Session Detail (`#v-detail`)
- Session name + meta (created date, public/private, name setting)
- Join code displayed as large styled pill (click to copy)
- Shareable URL: `location.origin + location.pathname + '?join=' + code`
- Stats card: total feedbacks, question count, status
- Action buttons:
  - Open sessions: **Close Session** (red), **Save PDF** (teal), **Copy Link**
  - Closed sessions: **Reopen** (orange), **Save PDF**, **Copy Link**
- Board rendered in **admin view** (read-only, shows all feedback regardless of public/private, shows rating results)
- Live listener (`onValue`) updates board in real time

---

## Screen 4: Join Screen

**Two steps** in one card:

**Step 1:** Enter 4-letter code (uppercase input, large font `26px`, letter-spacing). Look Up button + Enter key support.

**Step 2:** After lookup — show session name + description. Always show name field (label changes: "Your Name *" if required, "Your Name (optional)" if not). Enter key → submit.

**On Enter Session:**
- Validate name if required
- Generate fresh `instanceId = uid()` — each join is fully independent
- Reset `S.voted = new Set()` and `S.rated = {}` — clean slate every entry
- Start Firebase live listener
- Navigate to feedback board

**URL param support:** `?join=CODE` — on page load, if this param exists, go directly to join screen with code pre-filled and looked up.

---

## Screen 5: Feedback Board (Participant View)

**Header bar:** Session name + description, participant badge (👤 name), public/private badge, 🔴 Live badge (pulsing animation).

**Board layout:** Scrollable area with question sections. Each section has:
- Question number circle (orange gradient) + title + **+ Add** button (orange pill) + feedback count badge
- Post-it grid + dashed hint zone (double-click or click + Add)

### Question types on board:

**Text question → Post-it grid:**
- Post-its displayed as colored sticky notes with slight random rotation (`--rot` CSS variable)
- 8 colors: yellow, orange, teal, pink, blue, green, purple, lime
- Color determined by seeded hash of post-it ID (deterministic, consistent)
- Each post-it: author name (uppercase, small, faded), text body, footer with timestamp + upvote button
- Own post-its (matching `S.who`) show a small ✕ delete button
- Click post-it → inline edit mode (textarea replaces text, Save/Cancel buttons)
- Dashed zone: double-click triggers new post-it creation

**Rated question → Emoji rating widget:**
- 5 large emoji cards: 😞(1) 😕(2) 😐(3) 😊(4) 😍(5)
- Click a card → instant submission (no submit button needed)
- Selected card turns orange gradient, others dim to 30% opacity
- Can change rating by clicking different card (old vote decremented, new incremented)
- Rating state stored **in memory only** (`S.rated` object) — NOT in localStorage
- Status text: "Click an emoji to submit" → "You rated 😊 4 — Good (Click another card to change)"
- Results (bar chart + average) shown **only in admin view** — not to participants

**Free-form board (no questions):** Single "💬 General Feedback" section with post-it grid.

**Private sessions:** Participants only see their own post-its (`filterVisible` function). Admin sees all.

---

## New Post-it Creation (Critical Implementation Detail)

**Do NOT** append the new post-it input inside the board DOM. Firebase's live listener re-renders the board and destroys any in-DOM inputs.

**Solution:** Use a **fixed overlay** (`position: fixed, inset: 0`) that sits above everything:

```html
<div id="new-pi-overlay" style="display:none" class="new-pi-overlay">
  <div class="new-pi-card">  <!-- yellow post-it style card -->
    <div class="pn-au">AUTHOR NAME</div>
    <textarea class="pn-ta" placeholder="Type your thought…"></textarea>
    <div class="pn-actions">
      <button onclick="postNew()">Post 📌</button>
      <button onclick="closeNewPI()">Cancel</button>
    </div>
  </div>
</div>
```

- `openNewPI(qid)` — stores `S.newPIqid`, shows overlay, focuses textarea
- `postNew()` — reads textarea, calls `push('sessions/{code}/feedbacks/{qid}', fb)`, closes overlay
- `closeNewPI()` — hides overlay, clears `S.newPIqid`
- Clicking backdrop (overlay itself) closes it

Post-it object structure:
```javascript
{
  id: uid(),
  author: S.who,    // null if anonymous
  text: string,
  ts: Date.now(),
  upvotes: 0,
  rot: piR()        // e.g. "-2.3deg"
}
```

---

## Post-it Interactions

### Editing
- Click post-it → if not upvote button or delete button → enter edit mode
- Replace `.pi-tx` div with `<textarea>` + Save/Cancel buttons inside the post-it
- Save: find Firebase push key by scanning `feedbacks/{qid}` for matching `id`, then `update` with `{text, editedAt}`
- Cancel: revert (board re-renders via live listener)
- Close edit when clicking outside `.postit`

### Deleting (own posts only)
- Small ✕ button in post-it footer, visible only when `fb.author === S.who`
- Confirm dialog → find push key → `remove` from Firebase

### Upvoting
- Toggle: click upvotes → +1; click again → -1 (remove vote)
- `S.voted` Set tracks which fbIds voted in this session entry
- Each `enterSession()` call resets `S.voted = new Set()` — every join is fully independent
- `S.voted` is NOT persisted to localStorage
- Find push key → `update` with new upvote count

---

## Live Sync (Firebase `onValue`)

```javascript
function startLive(code) {
  stopLive();
  S.liveRef = dOn('sessions/' + code, data => {
    if (!data) return;
    S.sess = data;
    // Update participant board
    if (fbScreen.classList.contains('on')) renderBoard(data);
    // Update admin detail board
    if (detailView visible && S.code === code) {
      dtBoard.innerHTML = buildBoard(data, true);
      renderDetailMeta(data);
    }
  });
}
```

`stopLive()` calls `off(S.liveRef)`. Call `stopLive()` when navigating away from board or detail view.

---

## PDF Export

`savePDF(code)` function:
1. Fetch fresh session data from Firebase
2. Build a complete HTML string with inline CSS
3. Render post-its as colored divs in a flex grid
4. Render rating questions as horizontal bar charts
5. Create `Blob` with `type: 'text/html'`, trigger download
6. Filename: `feedpost-{CODE}.html`
7. Toast: "Downloaded! Open in browser → Print → Save as PDF"

---

## State Object

```javascript
const S = {
  admin: null,        // logged-in username string
  sess: null,         // current session snapshot object
  code: null,         // current session code string
  who: null,          // participant name or null (anonymous)
  pendQs: [],         // pending questions during session creation
  pendQT: null,       // pending question type ('text'|'rating')
  liveRef: null,      // Firebase onValue reference for cleanup
  newPIqid: null,     // target question id for new post-it overlay
  instanceId: null,   // unique id per session entry (uid())
  voted: new Set(),   // fbIds upvoted this session entry (memory only)
  rated: {},          // { qid: n } ratings this session entry (memory only)
};
```

**Critical:** `S.voted` and `S.rated` are reset on every `enterSession()` call. They are **never persisted to localStorage**. This ensures each person joining on the same browser gets a clean slate.

---

## Key Utility Functions

```javascript
function genCode() {
  // 4-char code, no ambiguous chars
  const c = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';
  let r = '';
  for (let i = 0; i < 4; i++) r += c[Math.floor(Math.random() * c.length)];
  return r;
}

function uid() {
  return Date.now().toString(36) + Math.random().toString(36).slice(2, 7);
}

function piColor(seed) {
  // Deterministic color from post-it ID
  const colors = ['pi-y','pi-o','pi-t','pi-p','pi-b','pi-g','pi-v','pi-l'];
  let h = 0;
  for (const c of (seed || 'x')) h = (h * 31 + c.charCodeAt(0)) % colors.length;
  return colors[Math.abs(h)];
}

function piRot() {
  return (Math.random() * 7 - 3.5).toFixed(1) + 'deg';
}

function toArr(o) {
  // Firebase stores arrays as objects with numeric keys — normalize
  if (!o) return [];
  if (Array.isArray(o)) return o;
  return Object.values(o);
}

function sessUrl(code) {
  return location.origin + location.pathname + '?join=' + code;
}
```

---

## Init & Double-Init Prevention

```javascript
let _initDone = false;
function init() {
  if (_initDone) return;
  _initDone = true;
  // hide loader, show home screen, check URL param
}

document.addEventListener('fp-ready', () => setTimeout(init, 80));
// Safety fallback only — never fires if fp-ready works
setTimeout(() => { if (!_initDone) init(); }, 3500);
```

**URL param check — single run:**
```javascript
let _urlDone = false;
function checkUrl() {
  if (_urlDone) return;
  const c = new URLSearchParams(location.search).get('join');
  if (c) { _urlDone = true; setTimeout(() => goJoinScreen(c.toUpperCase()), 150); }
}
```

---

## UI Components

### Toggle Switch
```html
<button id="tg-pub" class="tg on" onclick="this.classList.toggle('on')"></button>
```
```css
.tg { width: 46px; height: 25px; border-radius: 13px; position: relative; }
.tg.on { background: var(--tl); }
.tg::after { /* white circle */ position: absolute; top: 3px; left: 3px; width: 19px; height: 19px; border-radius: 50%; transition: transform .25s; }
.tg.on::after { transform: translateX(21px); }
```

### Toast Notifications
```html
<div id="toast"></div>
```
Fixed bottom-center, transforms in/out. `.ok` prefix = teal ✓, `.er` prefix = red ✕. Auto-dismisses after 2.8s.

### Help Panels
Each screen has a `?` button in topbar that opens a fixed panel (top-right) with numbered steps. Clicking again or clicking another `?` button closes it.

### Loading Screen
Full-screen dark overlay with pulsing orange logo icon. Fades out on init.

### Spinner
Used in button loading states: `<div class="spin"></div>` (CSS border animation).

---

## Enter Key Support

All form inputs must have `onkeydown` handlers:
- Username → focus password
- Password (login) → submit
- Password (register) → focus confirm
- Confirm password → submit register
- Session code input → look up
- Name field → enter session
- Question text modal → confirm add

---

## Post-it Visual Style

```css
.postit {
  width: 190px;
  min-height: 170px;
  padding: 14px;
  border-radius: 2px;  /* subtle, not rounded */
  transform: rotate(var(--rot, 0deg));
  box-shadow: 3px 4px 14px rgba(0,0,0,.13), 0 1px 3px rgba(0,0,0,.07);
  display: flex;
  flex-direction: column;
}
.postit::before {
  /* dark strip at top simulating the "torn" edge */
  content: '';
  position: absolute;
  top: 0; left: 0; right: 0;
  height: 26px;
  background: rgba(0,0,0,.07);
  border-radius: 2px 2px 0 0;
}
.postit:hover {
  transform: rotate(0deg) scale(1.04);
  box-shadow: 6px 8px 24px rgba(0,0,0,.18);
  z-index: 20;
}
```

New post-it overlay card style: yellow (`#FFF176`) with `3px solid teal` ring, `rotate(-1.5deg)`, pop-in animation.

---

## Responsiveness

Mobile breakpoint at `600px`:
- Topbar padding reduces
- Admin body + board padding reduces
- Post-it width: 155px (from 190px)
- Auth/join cards padding reduces
- Help panel: `calc(100vw - 32px)` width
- Rating cards: min-width 46px (from 56px)

---

## Google Analytics

```html
<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXXXX"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-XXXXXXXXXX');
</script>
```

Place immediately after `<meta charset>` in `<head>`.

---

## Integration with Agile Platform

When integrating FeedPost into the larger Agile platform:

1. **The tool should be embeddable** — wrap in a container div rather than using `body` as root if needed
2. **Auth can be shared** — the `toEmail(username)` pattern and Firebase Auth can be shared across tools if they share the same Firebase project
3. **Sessions are isolated** — `/sessions/{code}` data is self-contained, no cross-tool dependencies
4. **Navigation hooks** — the `go(screenId)` function controls screen visibility; replace with platform router if needed
5. **The Firebase config** — same project can be reused across all Agile tools; each tool writes to its own path (`/sessions/`, `/retros/`, `/boards/` etc.)
6. **Branding** — colors (`--or`, `--tl`) and the "Built by Hakan" byline should match the platform's design system

---

## What Makes This Work Well

- **No DOM-dependent post-it creation** — overlay approach prevents Firebase re-renders from destroying inputs
- **Single init guard** — prevents double-init from `fp-ready` + setTimeout fallback both firing
- **URL join flag** — `?join=CODE` handled once with `_urlDone` flag
- **Per-entry independence** — `instanceId` generated fresh on every `enterSession()`, voted/rated reset
- **Session recovery** — admin grid scans all sessions by `owner` field as fallback for orphaned sessions
- **Deterministic post-it colors** — same post-it always same color across all clients (seeded hash of ID)
- **Rating state in memory only** — different people on same browser never see each other's ratings

---

## Deliverable

A single `index.html` file (~80KB) that:
- Works when opened directly in a browser (no server required for local testing)
- Works deployed to Vercel, GitHub Pages, Netlify, or any static host
- Connects to Firebase for all data storage and real-time sync
- Is fully self-contained with no external dependencies except Firebase SDK (CDN) and Google Fonts
