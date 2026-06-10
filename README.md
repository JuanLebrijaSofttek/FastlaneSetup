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
6. **Homebrew + fastlane** installed on your Mac. If you don't have them, run
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

## STEP 2 — Create your "certificates" repo (~3 min)

This is a brand-new, empty, **private** GitHub repo that will safely store your
Apple ID badges. You make it once and reuse it for all your apps.

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

## STEP 3 — Generate your Apple badges, once (~15 min)

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

3. Generate the **iOS** badges (replace the two values in quotes):
   ```bash
   bundle exec fastlane match appstore \
     --git_url "https://github.com/YOURNAME/certificates.git" \
     --app_identifier "com.yourcompany.yourapp" \
     --readonly false
   ```
   - `--git_url` = the certificates repo URL from Step 2.
   - `--app_identifier` = your app's bundle ID (find it in Xcode → your target →
     Signing & Capabilities → "Bundle Identifier").

4. When it asks you to **"Create a new passphrase"**, type any password you
   like, twice. **Write this down — it is your `MATCH_PASSWORD`.** You'll need
   it later and there's no way to recover it.

5. It may ask for your **Apple ID login** — enter it. This is the only step that
   needs your Apple password.

6. **Only if your app is macOS**, also run these two:
   ```bash
   bundle exec fastlane match appstore --platform macos --readonly false \
     --git_url "https://github.com/YOURNAME/certificates.git" \
     --app_identifier "com.yourcompany.yourapp"

   bundle exec fastlane match mac_installer_distribution --platform macos --readonly false \
     --skip_provisioning_profiles true \
     --git_url "https://github.com/YOURNAME/certificates.git" \
     --app_identifier "com.yourcompany.yourapp"
   ```

✅ **Done when:** the command finishes without red errors, and your once-empty
`certificates` repo on GitHub now shows some encrypted files inside it.

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

### 5a. On the "Secrets" tab, click "New repository secret" and add these 6:

| Name (type exactly) | What to paste |
|---|---|
| `ASC_KEY` | Your `.p8` file as text. Get it by running in Terminal: `base64 -i AuthKey_XXXX.p8 \| pbcopy` (replace with your real `.p8` filename), then paste — it's now on your clipboard. |
| `ASC_KEY_ID` | The Key ID from Step 1. |
| `ASC_ISSUER_ID` | The Issuer ID from Step 1. |
| `MATCH_PASSWORD` | The passphrase you made in Step 3. |
| `MATCH_GIT_URL` | Your certificates repo URL (e.g. `https://github.com/YOURNAME/certificates.git`). |
| `MATCH_GIT_BASIC_AUTHORIZATION` | A login token so the robot can read your certificates repo. How to make it: see the box below. |

**Making `MATCH_GIT_BASIC_AUTHORIZATION`:**
1. github.com → click your avatar → **Settings** → **Developer settings** →
   **Personal access tokens** → **Fine-grained tokens** → **Generate new token**.
2. Give it access to **only** your `certificates` repo, with **Contents: Read**.
3. Generate it and copy the token (starts with `github_pat_...`).
4. In Terminal, run (use your GitHub username and the token):
   ```bash
   echo -n "YOUR_GITHUB_USERNAME:github_pat_xxxxx" | base64
   ```
5. Paste the result as the secret value.

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
| Build hangs forever at "signing" | Make sure the variable/secret `CI` is set — it already is in the workflow file, so just confirm you didn't edit that file. |
| `No matching profiles found` | The badges weren't generated for this app. Re-run Step 3 with the correct `--app_identifier`. |
| `scheme not found` | You forgot to tick **"Shared"** on your scheme in Xcode (end of Step 4). |
| macOS build says "Not Available for Testing" | A macOS-specific signing detail. Make sure you ran BOTH extra macOS commands in Step 3 step 6. |
| `.pkg upload` fails on macOS | A known occasional fastlane hiccup. Re-run the GitHub action; if it keeps failing, run `bundle update fastlane` and push again. |

---

## What this template will NOT do (on purpose)

- It does **not** create your app in App Store Connect — do that yourself first.
- It does **not** send builds to outside testers automatically — builds go to
  your internal TestFlight. (Advanced users can change this in the `Fastfile`.)
- It does **not** need your Apple password during normal runs — only once, in
  Step 3.
