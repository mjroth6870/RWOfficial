# Race Walk Officiating: standalone copy (Firebase)

`index.html` is the whole app. It is the same file as the Claude-hosted copy. Inside Claude it uses Claude's own
live data and sign-in; anywhere else it uses your Firebase project (`rwofficial-d9330`). Both copies run side by side.

## Copyright

Copyright © 2020–2026 Michael J. Roth. All rights reserved. See `LICENSE`. (Update the end year every January 1.)

## What is here
- `index.html`: the app
- `firestore.rules`: the security rules for your Firestore database
- `firebase.json`: lets the Firebase command line tool deploy the rules (optional)
- `.github/workflows/deploy-pages.yml`: publishes `index.html` to GitHub Pages every time you push to `main`

## One-time setup

### 1. Firebase console (project `rwofficial-d9330`)
1. **Authentication > Sign-in method**: turn on **Anonymous**. Turn on **Email/Password** and, inside it, **Email link (passwordless sign-in)**. Save.
2. **Firestore Database**: it should already exist (region NAM5, production mode).
3. **Publish the rules**: Firestore Database > **Rules** tab. Replace everything with the contents of `firestore.rules` and press **Publish**.
4. **Authentication > Settings > Authorized domains**: add your website's domain (for GitHub Pages: `YOURNAME.github.io`). Email sign-in links do not work without it.

### 2. Host the site (GitHub Pages)
1. Create a GitHub repository and upload everything in this folder, including the `.github` folder.
2. In the repository: **Settings > Pages > Build and deployment > Source: GitHub Actions**.
3. Push to `main` (or run the workflow from the Actions tab). The site appears at `https://YOURNAME.github.io/REPOSITORY/`.
4. Do step 1.4 above with that domain.

Any static host works instead (Cloudflare Pages, Netlify): publish `index.html` and add its domain to Authorized domains.

### 3. First use
1. Open the site. Enter your email, press **Email me a sign-in link**, and open the link on the same device.
2. **Create my account**, then **New meet**, or **Import from file** to bring in a meet exported from the Claude-hosted copy.
3. Give officials the QR code from their judge tab. They never sign in.
4. To broadcast standings, open the DQ Board section on the Setup tab and share its link: cast it to a TV with Chrome's Cast option, or open it full-screen over HDMI. It needs no sign-in.

## Test the rules before a real meet
In the Firebase console, Firestore > Rules > **Rules Playground**. Expected results:

| Simulate | Path | Expected |
|---|---|---|
| Unauthenticated get | `/cfg/anyone/meets/m1/event/main` | Denied |
| Signed in as organizer `X`, set | `/cfg/X/meets/m1` | Allowed |
| Signed in as organizer `X`, set | `/cfg/Y/meets/m1` | Denied |
| Anonymous guest, set | `/ent/m1/judges/j1` | Allowed |
| Anonymous guest, set | `/cfg/X/meets/m1` | Denied |
| Anonymous guest, get | `/cfg/X/meets/m1/event/main` | Allowed |
| Anonymous guest, list | `/cfg/X/meets` | Denied |
| Signed in, get | `/join/ABCDE` | Allowed |
| Signed in, list | `/join` | Denied |
| Organizer `X`, get | `/data/users/Y/account` | Denied |

Two more worth running: **list** `ent` as a guest should be **Denied**, and **create** `cfg/orgX/meets/m1/races/d1h1` as a guest should be **Denied**.

These rules were checked in advance with Firebase's own rules parser (syntax) and with an independent evaluator that ran all
of the above plus every read and write the app makes. That is not the same as Firebase's own emulator, so treat the
Playground as the final confirmation. If the console reports an error in the rules, copy the message to Claude.

Note: an earlier version of these rules let any signed-in guest list an owner's meets. The current file fixes that (a `{coll}`
segment before each `{rest=**}`). Make sure the Rules editor contains the current `firestore.rules`.

## Keeping the two copies in step
The Claude-hosted copy stays available as a backup. When Claude changes the app, update both:
- Claude-hosted: Claude republishes it.
- This copy: replace `index.html` in the repository with the new file and commit. The site redeploys in a minute or two.
  (With Claude Code on your computer, ask it to make the change, run the tests, commit and push.)

## Moving a meet between the two copies
In either copy: open the meet > **Setup** > **Export meet data**. In the other copy: **My meets** > **Import from file**.
The file holds the setup and every judge's entries, so it is also a backup you can keep. Imported meets get new QR codes.

## Free plan limits
The free plan allows 50,000 document reads, 20,000 writes and 20,000 deletes per day, and 1 GiB of storage.
When a daily limit is used up, that part of Firebase stops responding until it resets. Every entry is one write, and every
device watching that race reads it, so very large meets can use up the reads. Check **Firestore > Usage** after a big meet.
Upgrading to the Blaze plan removes the cutoff but has no built-in spending cap: set a budget alert if you do.

## Security notes
- Organizer accounts are verified by email link. Only the owner can change a meet's setup.
- Officials are anonymous guests. What protects a meet's entries is its long random id, carried by the QR code: anyone who has a meet's link can record entries for it. Replace the codes (Setup > Official codes) if a QR code is shared too widely.
- The Firebase web configuration in `index.html` is public by design. The rules and the authorized-domains list are what protect the data.
