# MATCH 🔥

A mobile-web party game. Everyone answers the same either/or question
privately on their phone, then walks around the room asking people what
they picked, hunting for a match. Find one, confirm it in the app, and
watch two matchsticks meet and catch fire. Most confirmed matches by the
end wins.

No install — players open a URL and type a room code, like Kahoot/Jackbox.
It's a single self-contained `index.html` file: Tailwind (CDN) for styling,
vanilla JS, canvas-confetti + a QR code library (CDN), and the Firebase v11
modular SDK loaded lazily via dynamic `import()` for realtime sync between
the host's device and every player's phone.

## Play it

Open `index.html` on a static host (see **Deploy** below) on the host's
phone, tap **Host a Game**, and share the room code or QR code. Everyone
else opens the same URL, taps **Join a Game**, and enters the code + their
name.

Want to try the loop solo, with no second phone? Tap **Try Demo Mode** — it
spins up three instant-answering bots on your own device, with a small
floating host-control bar so you can also advance questions and end the
game.

## One-time Firebase setup

The app already ships with a `firebaseConfig` pointing at a Firestore
project. Before it can host real games you need to turn on two things in
the [Firebase console](https://console.firebase.google.com/) for that
project:

1. **Authentication → Sign-in method → Anonymous** — enable it. MATCH signs
   every device in anonymously; that uid is a player's identity for the
   session (no accounts, no passwords).
2. **Firestore Database** — create it (Native mode, any region) if it
   doesn't exist yet.

Then deploy the security rules in this repo (they scope writes to
signed-in anonymous users):

```bash
npm install -g firebase-tools   # once
firebase login
firebase deploy --only firestore:rules
```

**If you deployed an earlier version of this app**, redeploy
`firestore.rules` again — it changed to support the host recovery/reclaim
flow below.

Demo Mode never touches Firebase, so you can try the whole game loop before
doing any of this.

### Optional: physically delete expired games via Firestore TTL

By default, expired rooms aren't deleted from Firestore — the app just
refuses to join/reclaim/resume them (see **Data retention** below), which
needs no extra setup and costs nothing. If you'd rather old room documents
actually get erased, Firestore has a native TTL (time-to-live) feature that
does this automatically in the background — but the API that turns it on
requires the project to have a **billing account linked** (Blaze plan),
even though normal usage for a game like this stays well within the free
tier (Blaze only charges for usage *beyond* the free quotas). This is a
Google Cloud policy on that specific API, not a cost you'll likely see.

If you want that: Firebase console → **upgrade to Blaze** (add a payment
method) → then, in [Cloud Shell](https://console.cloud.google.com/) or any
terminal with `gcloud` installed and authenticated:
```bash
gcloud config set project match-57f24
gcloud firestore fields ttls update expiresAt --collection-group=rooms --enable-ttl
```
Firestore sweeps expired documents in the background afterward (not
instantly — usually within 24h of expiry). This is purely an optional
cleanup step; skip it and the app still behaves correctly for players.

## Deploy

Any static host works — this is one HTML file with no build step.

**Firebase Hosting** (already configured in `firebase.json`):
```bash
firebase deploy --only hosting
```

**GitHub Pages**: enable Pages on this repo pointing at the branch/root, or
push `index.html` to a `gh-pages` branch.

**Anything else**: copy `index.html` to any static file server / CDN.

## How it works

- **Rooms**: a 4-letter code maps to a Firestore doc at `rooms/{code}` holding
  host info, status (`lobby` / `question` / `ended`), settings, the
  session's shuffled question list, the current question index + start
  timestamp, a `players` map, an `answers` map (question index → player →
  choice), and a `confirmedPairs` map (question index → pair key → match
  record) that prevents double-counting.
- **Matching is server-checked**: confirming a match runs a Firestore
  transaction that reads both players' *actual submitted* answers and only
  records a match if they're equal — there's no way to fake a match by lying
  in the UI. Once a player has a confirmed match for a question, neither
  they nor their partner can rack up a second one that round.
- **Timer**: the host's device (or, in Demo Mode, your own device) advances
  the question automatically when the countdown hits zero; **Next
  Question** always works as a manual override, and **End Game** ends the
  session early from any question.
- **Questions**: a ~20-question default pack adapts prompts from Arthur
  Aron's "36 Questions" study into swipeable either/or pairs (kept
  light/work-safe — no Set III deep-cuts). The host can check/uncheck any
  question, add their own (text + two labeled, emoji'd options), and set
  how many questions the session runs.
- **Fire animation**: a match triggers one of three random matchstick-meets-fire
  variations (a classic strike, a campfire drop, or a spark clash), then a
  confetti burst using flame-emoji shaped particles where supported.

## Data retention

**Games only stay active for 2 weeks.** Every room stores an `expiresAt`
(creation time + 14 days). By default this is enforced app-side: joining,
reclaiming, or resuming a room past its `expiresAt` is refused with a
"this game has expired" message, same as if it didn't exist — the room
document itself is small and harmless and is left in Firestore rather than
physically deleted, unless you've opted into the Firestore TTL policy
described above, in which case it's genuinely erased in the background.
The host setup screen shows a live "stays active for N more days"
countdown so this isn't a surprise.

When a host creates a room, they set a **recovery password**. If they get
disconnected, close the tab, or switch phones before the 2 weeks are up,
they can tap **"Hosted a game before? Reclaim it"** on the landing screen
and enter the room code + that password to become host again (this also
resets the room's 2-week clock). Note this password is a casual shared
secret at the same trust level as the room code itself — see **Security
model** below for why it can't be more than that without a backend.

## Security model

There's no backend beyond Firestore — no accounts, no server holding a
secret. The security rules only gate *authentication* (you must be
signed in, anonymously, to write anything) and prevent a bare `create`
from claiming someone else's uid as host. Beyond that, integrity —
match confirmation, the host recovery password, question-set contents —
is enforced by the app's own client-side logic, not by rules that could
resist a user opening devtools and issuing writes directly. That's an
intentional, documented trade-off for a zero-backend casual party game,
not an oversight: making the host password or match confirmation
tamper-proof against a determined cheater would require a Cloud Function
holding a secret the client never sees, which this project doesn't have.
The rules and their reasoning are commented in `firestore.rules`, and the
new-fields behavior is covered by an emulator-based rules test suite (not
checked into the repo, but reproducible with
`@firebase/rules-unit-testing` against `firestore.rules`).

## Notes

- All Firebase/network calls are deferred until a Host, Join, or Reclaim
  action needs them, so the landing screen and Demo Mode work instantly
  even if the network is slow or a CDN is blocked.
