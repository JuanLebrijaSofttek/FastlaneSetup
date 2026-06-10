# Auto-Upload Your App to TestFlight

This is a **template**. It adds an automatic pipeline to an iOS or macOS app so
that every time you merge code into your `master` branch on GitHub, your app is
automatically built and uploaded to **TestFlight** (Apple's app for sharing test
versions). You stop uploading builds by hand.

You do **not** need to understand how it works. Follow the steps in order and
copy-paste the commands exactly.

---

## Plain-English glossary (read once, 2 min)

- **Terminal** — a text window where you type commands. On a Mac, open it via
  Spotlight: press `Cmd + Space`, type `Terminal`, press Enter.
- **Repo (repository)** — a project folder stored on GitHub.
- **Fastlane** — a free tool that builds and uploads apps for you.
- **Certificate / provisioning profile** — Apple's ID badges that prove your app
  is really from you. Apps won't upload without them. A tool called **match**
  handles these so you don't have to.
- **Secret** — a password or key you store safely in GitHub. GitHub hides it and
  never shows it in logs.
- **CI (Continuous Integration)** — a robot computer GitHub runs for you that
  does the build automatically.

---

## What you need before starting

Tick all of these off first. If any is missing, stop and get it.

1. A **Mac** (required — iOS/macOS apps can only be built on a Mac).
2. **Xcode** installed (from the Mac App Store).
3. An **Apple Developer account** (the paid one, $99/year).
4. Your app **already created in App Store Connect**
   (appstoreconnect.apple.com → My Apps → "+"). This template does NOT create
   the app for you.
5. Your app's code **already on GitHub** in a repo.
6. An **app icon set** in your project. Open `Assets.xcassets` → `AppIcon` and
   drop in a **1024×1024 PNG**. Apple rejects uploads with no icon.
7. **Homebrew + fastlane** installed on your Mac. If you don't have them, run
   these two commands in Terminal (copy-paste each, press Enter, wait):
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   brew install fastlane
   ```

---

## The big picture (you'll touch THREE repos)

```
1. THIS template repo  →  you copy its files into your app.
2. YOUR app repo       →  your code + these files. This is what ships.
3. A "certificates" repo →  a NEW empty repo that safely stores Apple's ID badges.
                            You create it once. You never open it again.
```

Don't worry about why — the steps below tell you exactly what to do with each.

---

# SETUP — do these steps in order

First time: about 1 hour. Take it slow. Each step says how you'll know it worked.

---

## STEP 1 — Get your Apple API key (~5 min)

This key lets the robot upload on your behalf without your Apple password.

1. Go to **appstoreconnect.apple.com**.
2. Click **Users and Access** → tab **Integrations** → **App Store Connect API**.
3. Click the blue **+** to make a new key.
4. Name it anything (e.g. "CI"). For **Access**, choose **App Manager**.
5. Click **Generate**.
6. Click **Download** to save the file. It ends in **`.p8`**.
   ⚠️ You can only download it ONCE. Keep it somewhere safe.
7. On that same page, write down two values shown next to your key:
   - **Key ID** (short, like `A1B2C3D4E5`)
   - **Issuer ID** (long, like `12a3b4c5-...`)

✅ **Done when:** you have the `.p8` file saved and both IDs written down.

---

## STEP 2 — Create your one shared "certificates" repo (~3 min, done only once ever)

This is a brand-new, empty, **private** GitHub repo that will safely store your
Apple ID badges. You make it **one time** and **reuse it for every app** you ever
set up — you do NOT make a new one per app.

1. Go to **github.com** → click **New repository**.
2. Name it `certificates`.
3. Choose **Private** (very important — never public).
4. Do NOT add a README or anything. Leave it empty.
5. Click **Create repository**.
6. Copy its web address (looks like
   `https://github.com/YOURNAME/certificates.git`). Save it for later.

✅ **Done when:** you have an empty private `certificates` repo and its URL.

> ❗️You will NOT open or put files in this repo yourself. A tool fills it
> automatically in the next step. Do not `cd` into it.

---

## STEP 3 — Generate your Apple badges (run once PER APP, ~15 min)

> **Read this first — it clears up the most common confusion:**
> You run this step **once for each app** you want to ship (iOS and macOS apps
> alike). It is NOT a single run that covers everything.
>
> - Your **certificate** (proves *you* are the developer) is created on your
>   very first run, then **reused automatically** for every app afterward.
> - A **provisioning profile** (ties the badge to *one specific app*) is made
>   fresh on every run. That's why the command needs `--app_identifier` each
>   time — it has to know which app.
>
> So: first app = sets up the shared certificate + that app's profile. Every
> app after = reuses the certificate, adds a new profile. Everything is stored
> in the same one `certificates` repo from Step 2.

You run this from **this template's folder** (the one with the file named
`Gemfile`). If you haven't downloaded this template yet, do that first and open
its folder in Terminal.

1. In Terminal, go into the template folder. Example:
   ```bash
   cd ~/Downloads/FastlaneSetup
   ```
   (Replace the path with wherever this folder actually is. Tip: type `cd `
   then drag the folder from Finder into the Terminal window, then press Enter.)

2. Install fastlane's helpers:
   ```bash
   bundle install
   ```

3. **Create your API key file** (this is what lets you skip the Apple ID login
   and 2FA codes entirely). First, put your `.p8` file from Step 1 into this
   template folder. Then run the command below, replacing the 3 values:
   - `YOUR_KEY_ID` → the Key ID from Step 1
   - `YOUR_ISSUER_ID` → the Issuer ID from Step 1
   - `AuthKey_XXXX.p8` → your real `.p8` filename
   ```bash
   python3 -c "import json; print(json.dumps({'key_id':'YOUR_KEY_ID','issuer_id':'YOUR_ISSUER_ID','key':open('AuthKey_XXXX.p8').read(),'in_house':False}))" > api_key.json
   ```
   (Python is already installed on every Mac. This makes a small file called
   `api_key.json` that match will use to log in for you.)

4. Generate the **iOS** badges (replace the two values in quotes):
   ```bash
   bundle exec fastlane match appstore \
     --api_key_path "api_key.json" \
     --git_url "https://github.com/YOURNAME/certificates.git" \
     --app_identifier "com.yourcompany.yourapp" \
     --readonly false
   ```
   - `--api_key_path` = the file you just made. This is what avoids the Apple ID
     login prompt.
   - `--git_url` = the certificates repo URL from Step 2.
   - `--app_identifier` = your app's bundle ID (find it in Xcode → your target →
     Signing & Capabilities → "Bundle Identifier").

5. When it asks you to **"Create a new passphrase"**, type any password you
   like, twice. **Write this down — it is your `MATCH_PASSWORD`.** You'll need
   it later and there's no way to recover it.

   > You should NOT be asked for an Apple ID or a 2FA code. If you are, it means
   > the `--api_key_path` line was missing or the `api_key.json` file is wrong —
   > stop, fix it, and re-run.

6. **Only if your app is macOS**, also run these two (same `--api_key_path`):
   ```bash
   bundle exec fastlane match appstore --platform macos --readonly false \
     --api_key_path "api_key.json" \
     --git_url "https://github.com/YOURNAME/certificates.git" \
     --app_identifier "com.yourcompany.yourapp"

   bundle exec fastlane match mac_installer_distribution --platform macos --readonly false \
     --api_key_path "api_key.json" \
     --skip_provisioning_profiles true \
     --git_url "https://github.com/YOURNAME/certificates.git" \
     --app_identifier "com.yourcompany.yourapp"
   ```

7. **Clean up** — delete the key file, since it holds your private key:
   ```bash
   rm api_key.json
   ```

✅ **Done when:** the command finishes without red errors, and your once-empty
`certificates` repo on GitHub now shows some encrypted files inside it.

> **If you get a "permission" or "forbidden" error while it creates the
> certificate:** your API key needs more access. Go back to Step 1, make a new
> key with the **Admin** role instead of App Manager, rebuild `api_key.json`
> with the new Key ID/Issuer ID/.p8, and re-run.

---

## STEP 4 — Put the template files into YOUR app (~3 min)

Now copy this template's files into your **app's** repo folder — the folder that
contains your `.xcodeproj` file.

Copy these into your app's main folder:
- `Gemfile`
- `.gitignore`
- the whole `fastlane` folder
- the `.github` folder — but inside `.github/workflows`, **keep only one file**:
  - keep `ios-testflight.yml` if your app is **iOS**
  - keep `mac-testflight.yml` if your app is **macOS**
  - delete the other one.

After copying, your app folder should look like:
```
YourApp/
├── YourApp.xcodeproj        (already there)
├── Gemfile                  (new)
├── .gitignore               (new)
├── fastlane/                (new)
│   ├── Appfile
│   ├── Matchfile
│   └── Fastfile
└── .github/
    └── workflows/
        └── ios-testflight.yml   (or mac-testflight.yml)
```

✅ **Done when:** those files sit next to your `.xcodeproj`.

> One more thing in Xcode: open your project → **Product menu → Scheme → Manage
> Schemes** → make sure your app's scheme has the **"Shared"** box ticked. If
> it's not shared, the robot can't find your app. (1 min.)

---

## STEP 5 — Add your secrets to GitHub (~10 min)

Go to **YOUR app's repo on GitHub** → **Settings** → (left sidebar)
**Secrets and variables** → **Actions**.

### 5a. On the "Secrets" tab, click "New repository secret" and add these 5:

| Name (type exactly) | What to paste |
|---|---|
| `ASC_KEY` | Your `.p8` file's full text. Get it by running in Terminal: `cat AuthKey_XXXX.p8 \| pbcopy` (replace with your real `.p8` filename), then paste. Include the `-----BEGIN/END PRIVATE KEY-----` lines. |
| `ASC_KEY_ID` | The Key ID from Step 1. |
| `ASC_ISSUER_ID` | The Issuer ID from Step 1. |
| `MATCH_PASSWORD` | The passphrase you made in Step 3. |
| `MATCH_GIT_URL` | Your certificates repo URL **with a login token built in** so the robot can read it. Format below. |

**Building `MATCH_GIT_URL` (with the token inside it):**
1. github.com → your avatar → **Settings** → **Developer settings** →
   **Personal access tokens** → **Tokens (classic)** → **Generate new token (classic)**.
2. Tick the **`repo`** scope (covers private-repo read). Generate it and copy the
   token (starts with `ghp_...`).
3. Build the URL by dropping your username and token into this shape:
   ```
   https://YOUR_USERNAME:ghp_xxxxx@github.com/YOUR_USERNAME/certificates.git
   ```
4. Paste that whole URL as the `MATCH_GIT_URL` secret value.

> Why the token goes in the URL: it's the simplest auth that works on CI — match
> just clones the URL. A **classic** token avoids the org-approval and
> header-format problems that commonly break fine-grained tokens here.

### 5b. Switch to the "Variables" tab, click "New repository variable", add these:

| Name | Example value |
|---|---|
| `APP_IDENTIFIER` | `com.yourcompany.yourapp` |
| `DEVELOPER_TEAM_ID` | `ABCDE12345` (find at developer.apple.com → Membership) |
| `SCHEME` | your scheme name (e.g. `MyApp`) |
| `XCODEPROJ` | `MyApp.xcodeproj` |
| `XCWORKSPACE` | `MyApp.xcworkspace` — **or leave blank** if you don't have one |
| `MAC_INSTALLER_CERT_NAME` | macOS only. Get it by running `security find-identity -v` and copying the line starting with "3rd Party Mac Developer Installer". Leave blank for iOS. |

✅ **Done when:** the Secrets tab shows 6 secrets and the Variables tab shows
your variables.

---

## STEP 6 — Turn it on (~2 min)

1. Commit and push your app (with the new files) to GitHub:
   ```bash
   git add .
   git commit -m "Add TestFlight pipeline"
   git push
   ```
2. From now on, **whenever a pull request is merged into `master`**, the robot
   runs automatically and uploads a new build to TestFlight.
3. Watch it work: your app repo on GitHub → **Actions** tab. Green check = success.

✅ **Done when:** the Actions tab shows a finished green run, and a new build
appears in App Store Connect → your app → TestFlight (it can take a few minutes
for Apple to process).

🎉 That's it. You never upload by hand again.

---

## If something goes red — common fixes

| What you see | What to do |
|---|---|
| `Could not locate Gemfile` | You're in the wrong folder. `cd` into the folder that has the `Gemfile` (the template folder in Step 3, or your app folder in Step 6). |
| `string contains null byte` at `app_store_connect_api_key` | Your `ASC_KEY` secret isn't the clean key text. Re-copy with `cat AuthKey_XXXX.p8 \| pbcopy` and paste it again (including the BEGIN/END lines). |
| `could not read Username for 'https://github.com'` at `match` | The certs-repo auth failed. Make sure `MATCH_GIT_URL` includes the token: `https://USER:ghp_xxx@github.com/USER/certificates.git`. Test it with `git clone <that-url> /tmp/t`. |
| `No profiles for '...' were found` at `build_app` | Your project uses Automatic signing. The Fastfile's `update_code_signing_settings` step fixes this — make sure it's present before `build_app`/`build_mac_app`. |
| `Missing required icon` / `CFBundleIconName` at upload | No app icon. Add a 1024×1024 PNG to `Assets.xcassets` → `AppIcon`, commit, push. |
| `built with the iOS 18.5 SDK ... must be built with iOS 26 SDK` | The runner used an old Xcode. The workflow's `setup-xcode` step picks the newest; if it still fails, change `runs-on: macos-latest` to `runs-on: macos-26`. |
| Build hangs forever at "signing" | Make sure `CI: "true"` is set — it already is in the workflow file, so just confirm you didn't edit that line. |
| `No matching profiles found` | The badges weren't generated for this app. Re-run Step 3 with the correct `--app_identifier`. |
| `scheme not found` | You forgot to tick **"Shared"** on your scheme in Xcode (end of Step 4). |
| macOS build says "Not Available for Testing" | A macOS-specific signing detail. Make sure you ran BOTH extra macOS commands in Step 3 step 6. |
| `.pkg upload` fails on macOS | A known occasional fastlane hiccup. Re-run the action; if it keeps failing, run `bundle update fastlane` and push again. |

---

## What this template will NOT do (on purpose)

- It does **not** create your app in App Store Connect — do that yourself first.
- It does **not** send builds to outside testers automatically — builds go to
  your internal TestFlight. (Advanced users can change this in the `Fastfile`.)
- It does **not** need your Apple password at all — it logs in with your API key
  (the `.p8`). You never type an Apple ID or a 2FA code.
