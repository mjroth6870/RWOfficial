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

## The admin screen (Firebase copy)
Any account can see whether it has admin access from **My meets**: an admin sees an **Admin** section there, opening a
screen that lists every account and the meets each one owns. The **first account ever created** on a fresh deployment
becomes an admin automatically, with no console step needed. From then on, an existing admin can make or remove
another account's admin access from that same screen.

From an account's own meet list, an admin can also **archive or restore** any of that account's meets and **reset an
official's codes** on one, without needing to ask its owner first -- each with a confirmation first, matching the
wording the owner themselves would see. From the accounts list, an admin can **suspend** an account: it can still see
its own meets, but cannot change them, start a new one, or hand out a new code, until an admin **lifts** the
suspension. An official already recording entries on a suspended organizer's meet keeps working -- suspension only
stops the organizer's own management of it. Editing someone else's meet setup directly, and deleting a meet or a login
outright, are not offered here.

If you deployed before this feature existed, re-paste `firestore.rules` into Firestore > Rules and publish again: the
rules add the `accounts` and `admins` collections this screen depends on (and, more recently, let an admin write to
another account's meets and suspend an account), alongside everything that was already there.

On the Claude-hosted copy, admin access is not a separate thing this app manages: it follows Claude's own permission
for who can edit that artifact (the owner, and anyone they add as an editor there). There is no Make admin button on
that copy, and no `admins` collection is used there at all -- but archiving, resetting codes and suspending an account
all work the same way, since they only need that same "can edit" permission the platform already tracks.

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
| Organizer `X` (not an admin), get | `/accounts/Y` | Denied |
| Organizer `X` (not an admin), set | `/accounts/X` | Allowed |
| Organizer `X` (an admin), list | `/accounts` | Allowed |
| Organizer `X` (an admin), get | `/cfg/Y/meets/m1` | Allowed |
| Organizer `X`, get | `/admins/X` | Allowed (this is how the app checks its own admin status) |
| Organizer `X` (not an admin), set | `/admins/Y` | Denied |
| Organizer `X` (an admin), update | `/cfg/Y/meets/m1` | Allowed (archive/restore/reset codes on another account's meet) |
| Organizer `X` (not an admin), update | `/cfg/Y/meets/m1` | Denied |
| Organizer `X` (suspended), update | `/cfg/X/meets/m1` | Denied (their own meet, while suspended) |
| Organizer `X` (suspended), get | `/cfg/X/meets/m1/event/main` | Allowed (suspension never blocks reading) |
| Anonymous guest, update | `/ent/m1/judges/j1` | Allowed, even if the meet's owner is suspended |
| Organizer `X` (an admin), update | `/accounts/Y` | Allowed (suspend or lift suspension) |
| Organizer `X` (not an admin), update | `/accounts/Y` | Denied |

Two more worth running: **list** `ent` as a guest should be **Denied**, and **create** `cfg/orgX/meets/m1/races/d1h1` as a guest should be **Denied**.

"An admin" above means a document already exists at `/admins/<their uid>`. To test that case in the Playground, first
create one by hand under Firestore > Data: a document at `admins/<some uid>` with any field (for example `by: "test"`). "Suspended" means a document
exists at `/accounts/<their uid>` with `suspended: true`; create or edit one the same way to test that case.

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
