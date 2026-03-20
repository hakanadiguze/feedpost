# FeedPost 📌
**Built by Hakan** — Real-time visual feedback platform

## Setup (Required: Firebase Realtime Database)

### Step 1 — Enable Realtime Database
1. Go to [Firebase Console](https://console.firebase.google.com)
2. Select your project (`feed-post-30aa9`)
3. Left sidebar → **Build** → **Realtime Database**
4. Click **Create Database**
5. Choose a region (e.g. `europe-west1`)
6. Select **Start in test mode** → Enable

> **This is why "session not found" error appears** — the app uses Realtime Database for cross-device sync. Without it, nothing is stored in the cloud.

### Step 2 — Deploy to Vercel / GitHub Pages
Upload `index.html` to your repo. That's it — Firebase config is already embedded.

---

## Features
- 📌 **Post-it board** — double-click to create, click to edit
- 😞😕😐😊😍 **Emoji rating scale** — 1–5 with live bar chart
- 🔴 **Real-time sync** — Firebase pushes updates to all devices instantly
- 🔗 **Share by code or URL** — `?join=CODE` links work from any device
- 👍 **Upvoting** — in public sessions
- 🔐 **Admin accounts** — persistent, cloud-stored sessions
- ⏱️ **Sessions stay open indefinitely** — until admin closes them (1 week, 1 month — no problem)

## Why admins can't post directly from the admin panel
The admin detail view is read-only (for oversight). To participate:
1. Copy the session link from the detail page
2. Open it in a **new browser tab**
3. Join as a participant — you can post, rate, and upvote

## Session Lifetime
Sessions **never expire automatically**. They stay open until the admin clicks **Close Session**. Participants can add feedback days or weeks after creation.
