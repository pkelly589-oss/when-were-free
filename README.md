# When We're Free

A lightweight scheduling poll for finding a day that works for a group of friends. The organizer proposes a few candidate dates, shares one link, and everyone marks each date **Good / Maybe / No**. Responses collect in one place and the best day is highlighted automatically.

The whole app is a **single, self-contained `index.html`** — no build step, no server code. It runs as a static page (e.g. on GitHub Pages) and connects to a free Firebase Firestore database for live, one-click responses.

---

## Quick start (for the group)

1. **Organizer** opens the site and creates a poll: an event name, an optional note, and candidate dates picked from a calendar.
2. The organizer gets two links:
   - a **Friend link** to share with everyone, and
   - an **Organizer link** to bookmark for themselves.
3. **Friends** open the Friend link, tap the dates on a calendar to mark their availability, and submit.
4. The **Organizer** watches responses roll in, sees a tally per date, and gets a **Best pick** badge on the day that works best.

Nothing to install, and (in live mode) friends don't need any account.

### Finding your links again

Every poll you create is saved to a **"Your polls"** list on the home screen, with buttons to open the **Results** or copy the **Friend link** again — no need to bookmark or dig through history.

By default this list lives in the browser you created the poll on. If Google sign-in is set up (see Setup below), a **"Sign in with Google"** button appears on the home screen: sign in and your poll list is tied to your Google account, so it follows you to any device you sign in on. It's optional — signed out, the list still works locally.

Beyond that list, the links are always recoverable because everything about a poll is encoded in its URL:

- **Bookmark** the organizer link and open it any time; its results view has a button to copy the friend link again.
- Either link leads to the other — the friend screen has an *"Are you the organizer?"* link, and the results screen has a *copy friend link* button.
- Worst case, **recreate the poll with the same name and dates**: you'll get the identical links back, and in live mode it reconnects to the responses friends already submitted (responses are tied to the name + dates, not to the link).

---

## How it works

### One file, two "screens" driven by the URL

All state needed to render a poll travels in the URL's `#hash`, so the same `index.html` serves every screen:

- **No hash** → the *Create* screen (build a new poll).
- `#poll=…` → the *Friend* screen (mark availability).
- `#poll=…&org=1` → the *Organizer* screen (see results).

When you create a poll, the event name, note, dates, and labels are packed into a compact token in the hash. Dates are stored as small day-offsets from an anchor date (rather than full `YYYY-MM-DD` strings) and text is URL-encoded, which keeps the shareable links short.

### How responses come in

Friends see a **Submit** button. Submitting writes their response to a shared Firestore database, and the organizer's results view subscribes to it and updates in real time — no copying, no accounts for friends, and it works across every device.

Because responses live in that shared database, collecting them requires a working connection to Firebase. If the page can't reach it (for example, offline, or the preview copy hosted inside Claude, whose sandbox blocks outside database calls), the app says so plainly and disables submitting rather than silently failing. Setting up Firebase is covered under **Setup** below.

### Marking availability

On the Friend screen, candidate dates appear highlighted on a month calendar (one grid per month if the dates span several). **Tapping a date cycles through its state:**

| Taps | State | Meaning |
|------|-------|---------|
| 1 | 👍 Good | works for me |
| 2 | 🤔 Maybe | could go either way |
| 3 | 👎 No | doesn't work |
| 4 | *(blank)* | no opinion / didn't answer |

Leaving a day blank counts as **"no reply"** — it never hurts or helps that date in the ranking, and the organizer can tell "nobody's against it" apart from "nobody answered."

### Picking the best day

Each date is scored across everyone who responded, ranked by **most 👍, then fewest 👎, then most 🤔** (ties broken by the earliest date). A single "No" doesn't rule a date out, but it does count against it. On the results screen the dates are listed **best-first**, and the top one gets a **Best pick** badge. The organizer can tap any date to see exactly who said what.

### Exporting the results

The organizer view has a **📋 Copy results** button that copies a plain-text summary to your clipboard — the event name, the best pick with who's available, a per-date breakdown, and the party size. It looks like this:

```
When We're Free — "October dinner"
Note: dinner + a movie
7 responses

★ Best pick: Fri, Oct 17, 2026 — 5 available
   Available: Alex, Sam, Jordan, Priya, Chris

All dates:
• Fri, Oct 17, 2026 — 5 good, 1 maybe, 1 no
• Sat, Oct 18, 2026 — 4 good, 2 maybe, 1 no
• Sun, Oct 19, 2026 — 2 good, 1 maybe, 3 no, 1 no reply

Party size (available on best date): 5
```

Paste it into a message, your notes, or a Claude chat to hand off the outcome — the date and party size are exactly what a follow-up task (like booking a table) needs.

---

## Setup

### 1. Host the page

Any static host works. For **GitHub Pages**:

1. Create a public repository and add `index.html` to it.
2. In **Settings → Pages**, set the source to **Deploy from a branch**, branch `main`, folder `/ (root)`, and save.
3. Your site goes live at `https://<your-username>.github.io/<repo-name>/`.

To update the app later, upload a new `index.html` and commit — the live site refreshes within a minute.

### 2. Connect Firebase (required to collect responses)

Responses are stored in a free Firebase Firestore database, so the app needs a Firebase project to gather availability. (The create screen and links still work without it, but friends can't submit until it's connected.)

1. Create a project at [console.firebase.google.com](https://console.firebase.google.com) (Google Analytics not needed).
2. **Build → Firestore Database → Create database**, in **production mode**.
3. In the Firestore **Rules** tab, paste the rules below and **Publish**:

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       // Poll responses — open, since friends aren't signed in
       match /responses/{id} {
         allow read: if true;
         allow create, update: if request.resource.data.pollId is string
           && request.resource.data.n is string && request.resource.data.n.size() <= 60
           && request.resource.data.a is string && request.resource.data.a.size() <= 80;
         allow delete: if true;
       }
       // Per-organizer "Your polls" list — only that signed-in user can touch theirs
       match /organizers/{uid} {
         allow read, write: if request.auth != null && request.auth.uid == uid;
       }
     }
   }
   ```

4. **Project settings → Your apps →** register a **Web app** (`</>`), and copy the `firebaseConfig` values.
5. In `index.html`, find the `FIREBASE_CONFIG` block near the top of the `<script>` and replace `null` with your config object:

   ```js
   var FIREBASE_CONFIG = {
     apiKey: "…",
     authDomain: "<project>.firebaseapp.com",
     projectId: "<project>",
     appId: "…"
   };
   ```

6. Re-upload `index.html`. The app switches to live mode automatically.

The Firebase `apiKey` is **not a secret** — it only identifies the project. The Firestore security rules above are what actually control access.

### 3. (Optional) Turn on cross-device sync for "Your polls"

This lets the **Your polls** list follow you across devices via Google sign-in. It's optional and only affects the organizer's own convenience list — responses already sync across devices through the organizer link.

1. In the Firebase console: **Authentication → Get started → Sign-in method →** enable **Google**, set a support email, and save.
2. **Authentication → Settings → Authorized domains → Add domain**, and add your GitHub Pages domain — just the host part, e.g. `your-username.github.io` (not the full path).
3. Make sure the Firestore rules include the `organizers` block shown above (it's already in the rules in step 2.3).

That's it — a **"Sign in with Google"** button appears on the home screen, and signing in with the same Google account on another device shows the same poll list.

---

## Data & privacy

- Responses live in a single Firestore collection called **`responses`**. Each document is one person's answer: `{ pollId, n (name), a (answers), m (note), ts }`. Re-submitting under the same name updates that person's answer.
- A poll's `pollId` is a short hash of its dates and title, so responses are grouped by poll.
- Your **"Your polls"** list is kept in your browser (`localStorage`) by default, so it's specific to each device. If you sign in with Google, it's also mirrored to an **`organizers/<your-uid>`** document in Firestore (just poll titles and their links) that only you can read or write — that's what makes it sync across devices. Removing a poll from the list doesn't delete the poll or its responses — it just clears the shortcut.
- There are no accounts or emails collected from friends. Anyone with a Friend link can view and submit — fine for a casual group; the short, unguessable `pollId` is the only thing tying responses to a poll.

---

## Customizing

Everything is in `index.html`:

- **Colors / fonts** — CSS variables in the `:root` block at the top (with light and dark themes).
- **Copy / labels** — inline in the HTML and the render functions.
- **Ranking logic** — see the sort in the `renderRank()` function (most Good → fewest No → most Maybe).
- **States** — the tap cycle and their meaning live in `cycleDate()` and `refreshRespondCells()`.

## Files

| File | Purpose |
|------|---------|
| `index.html` | The entire app — HTML, CSS, and JavaScript in one file. |
| `README.md` | This document. |
