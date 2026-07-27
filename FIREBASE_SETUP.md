# Turning on live suggestions for Collector Mania

The game works fully offline right out of the box — pulls, the collection, coins, and even the suggestion box all work with no setup. The only thing that needs this guide is making suggestions **sync live** between you, your brother, and your cousin, each on their own computer.

This takes about 5 minutes and is free. You only need to do it **once** — then everyone pastes the same code into their copy of the game.

## Why this is needed

Three separate computers can't see each other's local files. To let everyone see the same live suggestion list, the game needs a small shared database in the cloud. Google's Firebase has a free tier that's more than enough for this.

## Step 1 — Create a free Firebase project

1. Go to [console.firebase.google.com](https://console.firebase.google.com) and sign in with any Google account.
2. Click **Add project**.
3. Name it anything, e.g. `collector-mania`.
4. You can disable Google Analytics for this project (not needed) — click **Continue** / **Create project**.
5. Wait for it to finish, then click **Continue** into the project dashboard.

## Step 2 — Enable Firestore (the database)

1. In the left sidebar, click **Build → Firestore Database**.
2. Click **Create database**.
3. Choose **Start in test mode** for now (we'll tighten it in Step 4). Pick any nearby location.
4. Click **Enable**.

## Step 3 — Register a web app and copy the config

1. In the left sidebar, click the gear icon → **Project settings**.
2. Scroll to **Your apps**, click the **</>** (web) icon.
3. Give it a nickname like `collector-mania-web` and click **Register app**. Skip the hosting step.
4. You'll see a code block that looks like this:

   ```js
   const firebaseConfig = {
     apiKey: "AIza...",
     authDomain: "collector-mania-xxxxx.firebaseapp.com",
     projectId: "collector-mania-xxxxx",
     storageBucket: "collector-mania-xxxxx.appspot.com",
     messagingSenderId: "...",
     appId: "..."
   };
   ```

5. **Copy that whole block** (or just the `{ ... }` part) — you'll paste it straight into the game, no code editing required.

## Step 4 — Lock down the security rules

Test mode leaves the database wide open, which is fine for a day but will stop working after 30 days (and isn't great practice). Tighten it so only the `suggestions` collection can be written to, with simple sanity limits:

1. In Firestore, click the **Rules** tab.
2. Replace the contents with:

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /suggestions/{suggestionId} {
         allow read: if true;
         allow create: if request.resource.data.text is string
                       && request.resource.data.text.size() < 400
                       && request.resource.data.author is string;
         allow update: if request.resource.data.diff(resource.data).affectedKeys()
                       .hasOnly(['votes', 'voterIds', 'downvotes', 'downvoterIds']);
         allow delete: if false;
       }
     }
   }
   ```

3. Click **Publish**.

This keeps things open enough for a no-login family game (anyone with the config can read/suggest/vote) while blocking anything that isn't a suggestion or a vote — no one can delete entries or write to other collections.

### If you set this up before the downvote feature existed

The original rule only whitelisted `['votes', 'voterIds']`. Once downvoting shipped, any vote that touches the new `downvotes`/`downvoterIds` fields (which is any downvote, or any upvote that switches an existing downvote) gets silently rejected by Firestore with a `permission-denied` error — the button will look like it does nothing. If your suggestion voting was set up before this feature existed, go back to **Firestore → Rules** and replace the `allow update` line with the version above (adding `'downvotes', 'downvoterIds'` to the `hasOnly([...])` list), then **Publish**. This is a one-time fix — everyone using the same Firebase project picks it up automatically, no client-side changes needed.

## Step 5 — Connect the game

1. Open `collector-mania.html` in your browser.
2. Go to **⚙️ Settings**.
3. Paste the config block from Step 3 into the **Live suggestion sync** box.
4. Click **Connect**. The badge on the Suggestions tab should switch to **Live sync connected**.
5. Send the *same config* to your brother and cousin (text it, email it, whatever's easiest) — they paste it into their own copy of the game, once, and they're connected too.

That's it — after this, any suggestion or vote anyone makes shows up for everyone else within a second or two.

## Notes

- This config isn't a secret password — it's fine to share it with your brother and cousin, and it's fine if it's visible in the game's code. Security is handled by the Firestore rules in Step 4, not by hiding the config.
- If you ever want to see or moderate suggestions directly, go to **Firestore Database → Data** in the Firebase console — the `suggestions` collection is right there.
- Each person's collection progress (coins, characters) stays local to their own device — only the suggestion box is shared. If you want that to change later, just ask.
- Free tier limits are generous (50k reads/20k writes per day) — a handful of family members clicking around won't come close.
