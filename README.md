# Thaa Atoll Police — Firebase Edition

Same app you already had, with Firestore swapped in as the backend instead of `localStorage`/Claude's runtime storage. Data now syncs live across every device that opens this app — add a crime statistic on one phone and it shows up on another within a second or two, no refresh needed.

**Current mode: open access.** Anyone with the URL and the Firebase config can read and write. That's deliberate for now (you asked to hold off on sign-in) but read the **Locking it down later** section before this goes anywhere production-facing.

## 1. Create your Firebase project

1. Go to [console.firebase.google.com](https://console.firebase.google.com) → **Add project** → name it (e.g. `south-central-police-thaa`) → follow the prompts (Google Analytics is optional, skip it if you don't need it).
2. Once created, click **Build → Firestore Database → Create database**.
   - Choose a location close to your users (e.g. an `asia-south1`-ish region for Maldives-adjacent latency).
   - Start in **test mode** for now — you'll paste in the specific rules below anyway.
3. Click the **gear icon → Project settings → General**, scroll to **Your apps**, click the **</> (Web)** icon to register a web app (no Firebase Hosting needed, any nickname is fine).
4. Firebase will show you a `firebaseConfig` object. Copy it.

## 2. Paste your config into the app

Open `index.html`, find this block near the top of the `<body>` (search for `EDIT ME`):

```js
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT_ID.appspot.com",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID"
};
```

Replace it with the values Firebase gave you. That's the only edit required.

## 3. Set Firestore security rules

In the Firebase console: **Firestore Database → Rules**, replace the contents with:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /scp_thaa_data/{docId} {
      allow read, write: if true;
    }
  }
}
```

**This allows anyone with your `firebaseConfig` — which is visible in your page's source, by design, for client-side Firebase apps — to read and write every document in this collection.** That's fine for a pilot with a trusted group, but it means:
- Anyone could wipe or falsify crime statistics if they found the URL.
- There's no record of *who* changed what beyond the app's own "Notes" fields.

## 4. Deploy it

Any static host works — GitHub Pages, Firebase Hosting itself, Netlify, Vercel, or just open `index.html` locally. If you want it installable as an app (Add to Home Screen), it needs to be served over **https** — file:// won't register the service worker or manifest correctly on most browsers. GitHub Pages (like the setup from before) or `firebase deploy` (see below) both give you https for free.

### Optional: deploy via Firebase Hosting instead of GitHub Pages
Since you already have a Firebase project:
```bash
npm install -g firebase-tools
firebase login
firebase init hosting   # point it at this folder, single-page app: No
firebase deploy
```

## Locking it down later

When you're ready to move off open access, two things need to happen together — doing just one leaves a gap:

1. **Add Firebase Authentication** (email/password, or Google sign-in for `@yourdomain` accounts) so `firebase.auth().currentUser` exists.
2. **Tighten the Firestore rules** to require it, e.g.:
   ```
   allow read, write: if request.auth != null;
   ```
   or, for real role-based access matching the app's Regional/Atoll/Station/Viewer roles, store each user's role in a `users/{uid}` document and check it in the rules.

I can build the sign-in screen and role-aware rules whenever you're ready — it's a self-contained addition on top of what's here, nothing about today's Firestore wiring needs to change.

## What's in this folder

| File | Purpose |
|---|---|
| `index.html` | The app itself |
| `manifest.json` | PWA manifest (name, icons, colors) — enables "Add to Home Screen" |
| `service-worker.js` | Caches the app shell for offline loading (data still needs a connection) |
| `icon-*.png`, `apple-touch-icon.png` | App icons |

## Testing it worked

Open the deployed URL on two devices (or two browser tabs). Add a crime statistic on one. Within a couple seconds, the other should update on its own with a small "Synced latest data" toast — that's Firestore's live sync working. If nothing happens and the console shows a permissions error, double check the security rules from step 3 were saved.
