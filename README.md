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
signed-in anonymous users and stop anyone from stealing another room's
`hostId`):

```bash
npm install -g firebase-tools   # once
firebase login
firebase deploy --only firestore:rules
```

Demo Mode never touches Firebase, so you can try the whole game loop before
doing any of this.

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

## Notes

- All Firebase/network calls are deferred until a Host or Join action needs
  them, so the landing screen and Demo Mode work instantly even if the
  network is slow or a CDN is blocked.
- There's no backend beyond Firestore — the security rules plus
  transaction-based match confirmation are the only integrity guarantees
  (fine for a casual party game; not designed to resist a determined
  cheater with devtools access).
