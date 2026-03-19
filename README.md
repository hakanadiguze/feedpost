# FeedPost 📌

A real-time visual feedback platform where participants stick post-it notes to share thoughts.

**Built by Hakan**

## Features

- 🟠 **Post-it board** — participants double-click to create sticky notes, click to edit
- ⭐ **Rated questions** — 1–5 number selector with live bar chart results
- 🔴 **Real-time sync** — new post-its and ratings appear for everyone within ~1.5s
- 🔗 **Share by code or link** — 4-letter join code + auto-generated URL
- 👍 **Upvoting** — in public sessions, users can upvote post-its
- 🔐 **Admin accounts** — username/password login, persistent session history
- 🌐 **Public / Private** — admin controls whether feedback is visible to all
- 👤 **Anonymous or named** — admin can require names or allow anonymous posts
- ❓ **How-to guide** — built-in help panel on every screen

## Question Types

| Type | How participants respond |
|------|------------------------|
| **Text question** | Creates a post-it note with free text |
| **Rated question** | Clicks a number 1–5; results shown as bar chart |
| *(no questions)* | Free-form general feedback board |

## Usage

### As Admin
1. Open `index.html` in a browser
2. Click **Admin Login** → create an account
3. Click **+ New Session** → configure options and add questions
4. Share the generated **code** or **link** with participants
5. Watch feedback arrive live in the session detail view
6. Click **Close Session** when done

### As Participant
1. Open the shared link directly, **or**
2. Click **Join a Session** and enter the 4-letter code
3. Enter your name (if required) and click **Enter Session**
4. For text questions: **double-click** the board or click **+** to add a post-it
5. For rated questions: click a number (1–5) then **Submit Rating**
6. Click any existing post-it to **edit** it

## Technical Notes

- **Single HTML file** — no build step, no server required
- **Persistence** — uses `localStorage` to store all data
- **Real-time** — polls `localStorage` every 1.5s (works across browser tabs on the same machine; for multi-device real-time, a backend is needed)
- **No dependencies** — pure HTML/CSS/JavaScript

## GitHub Pages Deployment

1. Upload `index.html` to a GitHub repository
2. Go to **Settings → Pages → Source** and select `main` branch
3. Your app will be live at `https://yourusername.github.io/repo-name/`

> **Note:** Because the app uses `localStorage` for storage, data is stored per-browser. For a true multi-device real-time experience, you would need a backend (e.g. Firebase, Supabase, or a custom server).

## License

MIT — free to use, modify, and distribute.
