# TestFlight CI Template (iOS + macOS, SwiftUI)

**This repository is a reusable template.** It is not an app. Its job is to give
you the files and instructions to add a TestFlight pipeline to *any* SwiftUI app
repo, so that **merging a PR into `master` automatically builds and uploads that
app to TestFlight.**

Works for iOS and macOS apps. No `produce`, no Apple ID password, no 2FA prompts.

## How to use this template

1. Publish this repo on GitHub (see "Publishing this as a template repo" below).
2. In the app you want to ship, follow **Setup — do these in order**.
3. After setup, every PR merged into `master` of that app fires the pipeline
   (a merge is a push to `master`, which is what the workflow listens for).

> Reading this for the first time? Skip to **Setup — do these in order**.

### Publishing this as a template repo (one time, ~3 min)

1. Create a new GitHub repo and push these files to it.
2. Repo → **Settings → General → check "Template repository"**.
3. Now anyone can click **"Use this template"** to start an app repo pre-loaded
   with these files — or just copy the files manually (Step 3 below).

---

## How it works (30-second version)

```
GitHub push to master
        │
        ▼
GitHub Actions runner (macOS)
        │
        ├─ load App Store Connect API key   (from secret ASC_KEY)
        ├─ match: pull signing certs        (from a SEPARATE private certs repo)
        ├─ run tests IF the scheme has them  (fails build on test failure)
        ├─ bump build number                 (starts at 1 on a brand-new app)
        ├─ build .ipa (iOS) or .pkg (macOS)
        └─ upload to TestFlight
```

Three repos are involved:
1. **This template repo** → you copy its files into an app repo (or "Use this template").
2. **The app repo** → holds the SwiftUI code + these files. This is what gets shipped.
3. **A private certs repo** → holds match's encrypted certificates. Created once, reused by every app.

---

## Setup — do these in order

Total time: ~45–60 min the first time. ~10 min for each additional app.

### Step 1 — Get your App Store Connect API key (~5 min)
1. App Store Connect → **Users and Access → Integrations → App Store Connect API**.
2. Create a key with the **App Manager** role (needed to manage TestFlight builds).
3. Download the `.p8` file. **You can only download it once.** Note the **Key ID** and **Issuer ID** shown on that page.

### Step 2 — Create the certs repo + generate certs ONCE locally (~15 min)
match stores your real certificates in a private git repo, encrypted. You generate them one time on your Mac; CI only ever reads them.

1. Create an **empty private GitHub repo**, e.g. `yourorg/certificates`.
2. On your Mac, in your app project folder:
   ```bash
   bundle install
   # iOS certs:
   bundle exec fastlane match appstore \
     --git_url "https://github.com/yourorg/certificates.git" \
     --app_identifier "com.yourcompany.yourapp" \
     --readonly false
   # macOS certs (run all three for a Mac app):
   bundle exec fastlane match appstore --platform macos --readonly false \
     --git_url "https://github.com/yourorg/certificates.git" \
     --app_identifier "com.yourcompany.yourapp"
   bundle exec fastlane match mac_installer_distribution --platform macos --readonly false \
     --skip_provisioning_profiles true \
     --git_url "https://github.com/yourorg/certificates.git" \
     --app_identifier "com.yourcompany.yourapp"
   ```
3. When prompted, **set a passphrase** — this is your `MATCH_PASSWORD`. Save it in your password manager.

> Why local-first: generating certs needs your Apple login. CI runs `readonly`, so it never needs that — it just decrypts what you already pushed.

### Step 3 — Copy the template files into your app repo (~2 min)
Copy into your app's root:
```
Gemfile
fastlane/Appfile
fastlane/Matchfile
fastlane/Fastfile
.github/workflows/ios-testflight.yml      # keep if you ship iOS
.github/workflows/mac-testflight.yml      # keep if you ship macOS
.gitignore
```
Delete whichever workflow you don't need.

### Step 4 — Add GitHub secrets (~10 min)
Repo → **Settings → Secrets and variables → Actions → Secrets** tab. Add these 6:

| Secret | What it is | How to get it |
|---|---|---|
| `ASC_KEY` | Your `.p8`, base64-encoded | `base64 -i AuthKey_XXXX.p8 \| pbcopy` then paste |
| `ASC_KEY_ID` | Key ID | from Step 1 |
| `ASC_ISSUER_ID` | Issuer ID | from Step 1 |
| `MATCH_PASSWORD` | Cert encryption passphrase | from Step 2 |
| `MATCH_GIT_URL` | HTTPS URL of your certs repo | `https://github.com/yourorg/certificates.git` |
| `MATCH_GIT_BASIC_AUTHORIZATION` | Read access to the certs repo | see below |

**`MATCH_GIT_BASIC_AUTHORIZATION`**: create a GitHub Personal Access Token (fine-grained, read-only **Contents** on the certs repo), then:
```bash
echo -n "your-github-username:github_pat_xxx" | base64
```
Paste the output as the secret. (GitHub blocks reusing one SSH deploy key across two repos, so a token is the clean path.)

### Step 5 — Add GitHub variables (~5 min)
Same screen, **Variables** tab. These are non-secret project facts:

| Variable | Example | Notes |
|---|---|---|
| `APP_IDENTIFIER` | `com.yourcompany.yourapp` | |
| `DEVELOPER_TEAM_ID` | `ABCDE12345` | developer.apple.com → Membership |
| `SCHEME` | `MyApp` | your shared Xcode scheme |
| `XCODEPROJ` | `MyApp.xcodeproj` | |
| `XCWORKSPACE` | `MyApp.xcworkspace` | leave **empty** if you have no workspace |
| `MAC_INSTALLER_CERT_NAME` | `3rd Party Mac Developer Installer: Your Co (ABCDE12345)` | macOS only; from `security find-identity -v` |

> **Make the Xcode scheme "Shared"** (Xcode → Manage Schemes → check "Shared"), or CI can't find it.

### Step 6 — Ship it (~2 min)
Commit, push to `master`. Watch repo → **Actions** tab. First run pushes build 1 to TestFlight.

---

## Security model — why this is safe

- **The `.p8` private key never lives in any repo.** It exists only as the base64 GitHub secret `ASC_KEY`, decoded in memory on the runner.
- **Real certificates never live in your app repo.** They sit in the separate certs repo, encrypted with `MATCH_PASSWORD`. Even someone who clones that repo sees only ciphertext.
- **CI is `readonly`.** The pipeline can never create or revoke certificates — it only decrypts existing ones.
- **`.gitignore` blocks** `*.p8`, `*.p12`, `*.cer`, profiles, and build artifacts from being committed by accident.
- **Secrets are masked** in GitHub Actions logs automatically.
- **Rotation:** to rotate the API key, generate a new one in App Store Connect, re-base64 it, update `ASC_KEY` / `ASC_KEY_ID` / `ASC_ISSUER_ID`. To rotate certs, run `fastlane match nuke` then regenerate (Step 2).

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Build hangs at signing | `setup_ci` not run — it's already in the lane; ensure `CI: "true"` is set (it is, in the workflow). |
| "No matching profiles found" | Certs not generated for that platform/bundle id. Re-run Step 2. |
| First build is number 2, not 1 | Expected only if you removed `initial_build_number: 0`. Keep it. |
| macOS build "Not Available for Testing" | Mac build needs a Mac App Store provisioning profile embedded + `com.apple.application-identifier` entitlement. Check your target's signing & entitlements. |
| `.pkg` upload fails with `io_fread` | Known intermittent fastlane bug on some versions. Update fastlane (`bundle update fastlane`) or upload that build via Apple Transporter as a fallback. |
| "scheme not found" | Mark the scheme **Shared** in Xcode. |

---

## What this template deliberately does NOT do

- **No `produce`** — the app must already exist in App Store Connect.
- **No notarization** — TestFlight/App Store builds are scanned and re-signed by Apple. Notarization is only for Developer-ID apps distributed outside the store.
- **No external-tester distribution by default** — builds go to TestFlight internal testing. Add `distribute_external: true` + `groups:` to `upload_to_testflight` when you want that (and drop `skip_waiting_for_build_processing`).
