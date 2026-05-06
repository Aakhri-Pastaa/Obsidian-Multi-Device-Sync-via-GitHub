# Obsidian Multi-Device Sync via GitHub

A free, self-hosted alternative to Obsidian Sync. Works across one or more laptops and an Android phone, with full version history and zero monthly cost.

This guide is the result of building this setup, breaking it, and rebuilding it correctly. Every step here is verified end-to-end on Windows + Android. The mistakes are documented so you don't have to repeat them.

> **Who this guide is for:** Anyone who wants a free, reliable Obsidian sync solution — including people who don't use git daily. Every command is explained, every error you might hit is covered, and there's no assumed prior knowledge of git, SSH, or Termux. If you've used a terminal before and can follow instructions, you can do this. **If you're comfortable with git already**, skim the Phase headers and skip the explanations — the commands are all there.

---

## Goal

By the end of this guide you will have:

- A private GitHub repository as the central sync backend
- Your **primary laptop** running auto-sync every 5 minutes via the Obsidian Git plugin
- **Any additional laptops** (Mac, Linux, second Windows machine — one or many) cloned from the primary, syncing the same way
- An **Android phone** syncing via a one-tap home screen widget powered by Termux
- SSH authentication on every device — no passwords, no tokens
- Version history for every note (every commit is a restorable snapshot)

---

## Architecture

```
   Primary Laptop          Other Laptop(s)             Android
   ┌────────────┐          ┌────────────┐          ┌────────────┐
   │  Obsidian  │          │  Obsidian  │          │  Obsidian  │
   │     +      │          │     +      │          │     +      │
   │ Git Plugin │          │ Git Plugin │          │   Termux   │
   └──────┬─────┘          └──────┬─────┘          └──────┬─────┘
          │                       │                       │
          │   SSH push/pull       │   SSH push/pull       │  SSH push/pull
          │   auto every 5 min    │   auto every 5 min    │  on widget tap
          │                       │                       │
          └───────────┬───────────┴───────────┬───────────┘
                      │                       │
                      ▼                       ▼
                 ┌─────────────────────────────────┐
                 │   GitHub Private Repository     │
                 │         (your-vault.git)        │
                 └─────────────────────────────────┘
```

The primary laptop is where the repo originates. Any number of additional laptops can clone from it - one or many, the steps for each additional one are identical. Android uses a separate path because the Obsidian Git plugin is unreliable on mobile.

---

## Do's and Don'ts

These are the behavioral rules. The technical setup is solid, but human habits cause 90% of conflicts. Follow these and you'll never see a sync error.

### Do

- **Treat sync as one-direction at a time.** Edit on one device, let it push, then move to another device.
- **Wait a few seconds after committing** before opening the vault on another device. GitHub takes a moment to propagate, and your other device needs time to pull.
- **Manually sync before switching devices.** If the 5-minute auto-sync window hasn't elapsed, press `Ctrl+P` → `Git: Commit-and-sync` before closing.
- **On Android, tap the widget twice per session** - once before opening Obsidian (pull), once after closing it (push).
- **Trust the auto-sync** for normal background work. Manual sync is for when you're actively switching devices.
- **Configure the Obsidian Git plugin separately on each laptop you add.** Plugin settings stay local by design — that's not a bug.
- **Verify pushes occasionally** by checking the GitHub repo page. Confirms everything is reaching the backend.

### Don't

- **Don't edit the same vault on two devices simultaneously.** This is the only reliable way to trigger a real merge conflict.
- **Don't ignore conflict prompts.** If the plugin shows a conflict, resolve it before continuing. Stacking edits on top of an unresolved conflict makes recovery harder.
- **Don't sync `.obsidian/plugins/` across devices.** Plugin internal files (especially `data.json`) change in the background constantly and cause conflicts. Install plugins individually on each laptop.
- **Don't run `git rebase` or `git stash` in your sync script.** They cause detached HEAD states on Android. Use plain `git pull --no-rebase`.
- **Don't store the `.git` folder in Android shared storage.** Android's file system has a known issue that can corrupt git data files (creates empty zero-byte files). The fix is in Phase 4 below.
- **Don't install Termux from the Play Store.** That version is outdated and broken. Use F-Droid only.
- **Don't commit before adding `.gitignore`.** Files committed before being ignored stay tracked permanently — even after you add them to `.gitignore` later. Order matters.

---

## Prerequisites

Install these before starting:

| Device | Software |
|--------|----------|
| Each laptop (primary + any additional) | Git (latest), Obsidian (latest) |
| Android | F-Droid, Termux (from F-Droid), Termux:Widget (from F-Droid), Obsidian (from Play Store) |
| GitHub | Free account |

**On Windows**, install Git via PowerShell:
```powershell
winget install --id Git.Git -e --source winget
```

**Verify Git on any platform:**
```
git --version
```

---

# Phase 1 — Create the GitHub Repository

This is your central backend. All devices will push to and pull from this repo.

1. Go to [github.com](https://github.com) → click **+** (top right) → **New repository**
2. Repository name: `KBVault` (or any name you prefer)
3. Visibility: **Private**
4. **Do not** initialize with a README, `.gitignore`, or license — leave it empty
5. Click **Create repository**
6. Leave the page open. You'll need the SSH URL in Phase 2 (looks like `git@github.com:yourusername/KBVault.git`)

---

# Phase 2 — Primary Laptop (Where the Vault Originates)

This is your starting point. The `.gitignore` and initial commit are created here, and any other devices you add later will clone from this state. **You only do Phase 2 once**, on the laptop where your vault lives (or where you want it to live).

### 2.1 Generate an SSH Key

SSH lets you push and pull without typing passwords. Each device gets its own key.

In **PowerShell**:
```powershell
ssh-keygen -t ed25519 -C "your@email.com"
```

When prompted:
- **File path:** press `Enter` to accept the default (`~/.ssh/id_ed25519`)
- **Passphrase:** optional, but recommended. Skip if you don't want to type it on every push.

### 2.2 Add the SSH Key to GitHub

Copy the public key to clipboard:
```powershell
Get-Content ~/.ssh/id_ed25519.pub | clip
```

Then in GitHub:
1. Profile picture → **Settings**
2. Left sidebar → **SSH and GPG keys**
3. **New SSH key**
4. Title: `Primary Laptop` (or any descriptive name)
5. Key type: **Authentication Key**
6. Paste (`Ctrl+V`) → **Add SSH key**

Verify it works:
```powershell
ssh -T git@github.com
```
Expected output: `Hi <username>! You've successfully authenticated...`

If you see `Permission denied`, the key wasn't added correctly. Re-copy it and try again.

### 2.3 Initialize Your Vault

Navigate to your existing Obsidian vault folder. If you don't have one, create an empty folder anywhere — Obsidian can use any folder as a vault.

```powershell
cd "C:\Users\YourName\Documents\Obsidian Vault"
git init
git config user.name "Your Name"
git config user.email "your@email.com"
git config core.autocrlf false
git branch -M main
```

> **Why `core.autocrlf false`?** Windows uses CRLF line endings, Linux/Mac/Android use LF. Without this setting, you'll get warnings on every commit. We handle line endings with `.gitattributes` instead.

### 2.4 Create `.gitignore` (Critical Step)

This file tells git which files to ignore. **It must be created before your first commit.** Files committed before being ignored stay tracked permanently.

```powershell
Set-Content .gitignore "# Obsidian device-specific files
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/community-plugins.json

# Plugin folder — install plugins individually on each device
.obsidian/plugins/

# Obsidian cache
.obsidian/cache/

# Nested vault folders (subfolders that contain their own .obsidian)
**/.obsidian/

# Conflict markers from the Obsidian Git plugin
conflict-files-obsidian-git.md

# Trash and OS junk
.trash/
.DS_Store
Thumbs.db
._*"
```

**What this excludes and why:**

| File / folder | Why it's ignored |
|---------------|------------------|
| `workspace.json` | Tracks which notes are open per device. Constantly changes. |
| `workspace-mobile.json` | Same as above, but for mobile. |
| `community-plugins.json` | Stores which plugins are enabled. If synced, disabling a plugin on one device disables it everywhere. |
| `.obsidian/plugins/` | Plugin internals. The `data.json` file inside changes in the background and causes constant conflicts. |
| `.obsidian/cache/` | Local cache. Useless to sync. |
| `**/.obsidian/` | Catches subfolders that have their own `.obsidian` folder (nested vaults). |
| `conflict-files-obsidian-git.md` | The plugin generates this when it detects conflicts. Ignoring it prevents recursive conflicts on the conflict file itself. |

### 2.5 Create `.gitattributes`

This normalizes line endings across Windows, Linux/Mac, and Android. Without it, you'll get phantom diffs every time you switch devices.

```powershell
Set-Content .gitattributes "* text=auto eol=lf
*.md text eol=lf"
```

### 2.6 First Commit and Push

```powershell
git add .
git status
```

**Verify before committing.** The output of `git status` should NOT show `workspace.json`, `community-plugins.json`, or any file inside `.obsidian/plugins/`. If you see them, the `.gitignore` didn't take effect — stop and check the file path.

If clean:
```powershell
git commit -m "Initial vault setup"
git remote add origin git@github.com:yourusername/KBVault.git
git push -u origin main
```

Replace `yourusername/KBVault.git` with your actual SSH URL from Phase 1.

Verify on GitHub: refresh the repo page. You should see your notes, `.gitignore`, and `.gitattributes` listed.

### 2.7 Install and Configure the Obsidian Git Plugin

In Obsidian:
1. **Settings** → **Community plugins** → **Turn on community plugins** if it's off
2. **Browse** → search `obsidian-git` → **Install** → **Enable**
3. Open the plugin's settings tab and configure:

| Setting | Value |
|---------|-------|
| Auto commit-and-sync interval (minutes) | `5` |
| Auto pull interval (minutes) | `5` |
| Pull updates on startup | `ON` |
| Merge strategy | `Merge` |
| Commit message | `vault backup: {{date}}` |
| Conflict resolution preference (if available) | `Theirs` |

> **Why "Theirs"?** In a real conflict (same line edited on two devices), this means the most recently pushed version wins. Combined with the "sync before switching devices" habit, this is the safest default — you'll almost never hit a real conflict, and if you do, the most recent edit is preserved.

Test the plugin: press `Ctrl+P` → type `git commit` → run **Git: Commit-and-sync**. Should run without errors. The plugin will commit nothing if there's nothing to commit, which is normal.

---

# Phase 3 — Additional Laptops (Optional)

Skip this phase if you only have one laptop. If you have a second Windows machine, a Mac, or a Linux laptop you also want synced, follow these steps **for each one**. The steps are the same regardless of how many additional laptops you're adding.

The key difference from Phase 2: instead of initializing a new repo, you **clone** the existing one. This brings down all your notes, the `.gitignore`, and the `.gitattributes` automatically.

### 3.1 SSH Key for the Additional Laptop

```powershell
ssh-keygen -t ed25519 -C "your@email.com"
Get-Content ~/.ssh/id_ed25519.pub | clip
```

Add to GitHub: **Settings** → **SSH and GPG keys** → **New SSH key** → label it descriptively (e.g. `Work Laptop`, `Mac`, `Linux Desktop`).

> **Important:** Each device must have its own SSH key. Don't copy keys between machines — it's a security anti-pattern and makes key revocation impossible if one device is compromised.

Verify:
```powershell
ssh -T git@github.com
```

### 3.2 Clone the Vault

Pick a location for the vault, then clone:
```powershell
cd "D:\"
git clone git@github.com:yourusername/KBVault.git "Obsidian Vault"
cd "Obsidian Vault"
git config user.name "Your Name"
git config user.email "your@email.com"
git config core.autocrlf false
```

### 3.3 Open in Obsidian

1. Open Obsidian → **Open folder as vault**
2. Navigate to the cloned folder → **Use this folder**
3. All notes from Laptop 1 should appear

### 3.4 Install the Plugin Locally

The plugin must be installed individually on each laptop because we excluded `.obsidian/plugins/` from sync.

1. **Settings** → **Community plugins** → **Browse** → install **Obsidian Git**
2. Configure with the same settings as your primary laptop (Phase 2.7 table)

That's it. This laptop is now syncing. Repeat Phase 3 for each additional laptop you want connected.

---

# Phase 4 — Android (with FUSE Corruption Fix)

Android sync uses Termux + a one-tap widget instead of the Obsidian Git plugin. The plugin's mobile implementation is unreliable on Android.

The **FUSE issue:** Android's shared storage (`/storage/emulated/0/`) is mounted via FUSE, which can occasionally corrupt git data files when many small files are written at once. The fix is to keep `.git` (the data) on Termux's native filesystem, while the vault folder (the working tree) stays on shared storage where Obsidian can read it. Git supports this natively via `--separate-git-dir`.

### 4.1 Install the Apps

| App | Where from |
|-----|------------|
| F-Droid | [f-droid.org](https://f-droid.org) — install the APK |
| Termux | F-Droid (search Termux) |
| Termux:Widget | F-Droid (search Termux:Widget) |
| Obsidian | Google Play Store |

### 4.2 Termux Initial Setup

Open Termux:
```bash
termux-setup-storage
```
Accept the storage permission popup on Android. This creates `~/storage/shared/` which maps to your phone's main storage.

```bash
pkg update && pkg upgrade -y
pkg install -y git openssh
```

### 4.3 SSH Key on Android

```bash
ssh-keygen -t ed25519 -C "android"
```
Press `Enter` for default path. Skip the passphrase (no secure keychain on mobile).

Display the public key:
```bash
cat ~/.ssh/id_ed25519.pub
```

Long-press the output → **Select all** → **Copy**.

In Chrome on your phone: **github.com → Settings → SSH and GPG keys → New SSH key** → label `Android` → paste → save.

Verify:
```bash
ssh -T git@github.com
```

### 4.4 Clone with the FUSE Fix

This is the critical command. The `--separate-git-dir` flag stores the `.git` data in Termux's native filesystem (`~/obsidian-git`) while the vault folder lives in shared storage where Obsidian can access it.

```bash
git clone --separate-git-dir=$HOME/obsidian-git \
  git@github.com:yourusername/KBVault.git \
  ~/storage/shared/repos/KBVault
```

> **Important:** Don't pre-create `~/obsidian-git`. Git creates it. If it already exists from a previous attempt, remove it first: `rm -rf ~/obsidian-git`.

Set git config:
```bash
cd ~/storage/shared/repos/KBVault
git config user.name "Your Name"
git config user.email "your@email.com"
git config --global safe.directory '*'
git config --global core.pager cat
```

Verify the FUSE fix worked:
```bash
ls -la ~/storage/shared/repos/KBVault | head -5
cat ~/storage/shared/repos/KBVault/.git
```

The `.git` should be a **file** (not a folder), and its contents should be: `gitdir: /data/data/com.termux/files/home/obsidian-git`

### 4.5 Open the Vault in Obsidian

1. Open Obsidian on Android → **Open folder as vault**
2. Navigate to **Internal Storage → repos → KBVault** → **Use this folder**
3. **Disable the Obsidian Git plugin** if it's enabled — we don't need it on Android (Settings → Community plugins → toggle Obsidian Git off)

### 4.6 The Sync Script

Create the script:
```bash
mkdir -p ~/.shortcuts
nano ~/.shortcuts/sync.sh
```

Paste this exactly:
```bash
#!/bin/bash
eval "$(ssh-agent -s)" > /dev/null
ssh-add ~/.ssh/id_ed25519 2>/dev/null
cd ~/storage/shared/repos/KBVault
git add -A
git commit -m "Android sync $(date +'%Y-%m-%d %H:%M')" 2>/dev/null || true
GIT_MERGE_AUTOEDIT=no git pull --no-rebase origin main
git push origin main
sleep 2
exit
```

Save: `Ctrl+X` → `Y` → `Enter`.

Make it executable:
```bash
chmod +x ~/.shortcuts/sync.sh
```

**What each line does:**

| Line | Purpose |
|------|---------|
| `eval "$(ssh-agent -s)"` | Starts the SSH agent for this session |
| `ssh-add ~/.ssh/id_ed25519` | Loads your SSH key |
| `cd ~/storage/shared/...` | Navigate to vault |
| `git add -A` | Stage all local changes |
| `git commit ...` | Commit changes (the `|| true` prevents script failure if nothing to commit) |
| `GIT_MERGE_AUTOEDIT=no git pull --no-rebase` | Pull latest, auto-accept merge messages |
| `git push origin main` | Push to GitHub |
| `sleep 2 && exit` | Show output briefly, then close Termux |

### 4.7 Add the Widget

1. Long-press your Android home screen → **Widgets**
2. Find **Termux:Widget** → drag it to home screen
3. When the picker appears, select `sync.sh`
4. The widget appears as a tappable shortcut

First time you tap it, Android will ask for **"Display over other apps"** permission. Grant it — Termux needs this to run scripts from background widgets. This is normal and safe.

### 4.8 Workflow on Android

- **Before opening Obsidian:** tap the widget (pulls latest from GitHub)
- **After editing notes:** tap the widget (pushes your changes)

That's the entire Android workflow.

---

# Phase 5 — Daily Workflow

| Action | Laptop(s) | Android |
|--------|-----------|---------|
| Start a session | Open Obsidian — auto-pulls on startup | Tap widget first |
| While editing | Plugin auto-commits every 5 min | Edit normally |
| Switching to another device mid-session | `Ctrl+P` → **Git: Commit-and-sync** | Tap widget |
| Ending a session | Optional: `Ctrl+P` → **Git: Commit-and-sync** | Tap widget |

**Critical timing rule:** When switching devices, wait ~10 seconds after the source device finishes pushing before you open the vault on the target device. GitHub needs a moment to propagate, and the target device needs to pull. Skipping this wait is the #1 cause of phantom "missing notes" complaints.

---

# Phase 6 — Troubleshooting

### Error: `Permission denied (publickey)`

The SSH key isn't added to GitHub, or the wrong key is loaded.

```powershell
# Re-copy and re-add
Get-Content ~/.ssh/id_ed25519.pub | clip
# Paste into GitHub → Settings → SSH and GPG keys → New SSH key
ssh -T git@github.com
```

### Error: `src refspec main does not match any`

You forgot to commit before pushing.
```powershell
git add .
git commit -m "init"
git push -u origin main
```

### Error: `Updates were rejected because the remote contains work...`

Another device pushed before you. Pull first:
```powershell
git pull --no-rebase origin main
git push origin main
```

If a merge editor opens (Vim/nano), save with `Ctrl+X` → `Y` → `Enter` (nano) or `:wq` → `Enter` (Vim).

### Plugin shows conflict in `data.json` or `workspace.json`

These files shouldn't be tracked. Untrack them:
```powershell
git rm --cached .obsidian/plugins/obsidian-git/data.json
git rm --cached .obsidian/workspace.json
git commit -m "untrack device-specific files"
git push
```

### Android: detached HEAD state

The sync script ran into a partial rebase. Reset to GitHub state (Android edits are discarded — make sure your laptop is the source of truth):
```bash
cd ~/storage/shared/repos/KBVault
git fetch origin
git reset --hard origin/main
```

### Android: "git pull asks for merge commit message"

The merge editor is opening because of an unhandled merge. The script line `GIT_MERGE_AUTOEDIT=no git pull --no-rebase origin main` prevents this — make sure your script has it. If you see the editor anyway, save with `Ctrl+X` → `Y` → `Enter`.

### Plugin settings reset after sync

Expected behavior. `community-plugins.json` and `.obsidian/plugins/` are intentionally not synced (per the Do's and Don'ts). Reconfigure plugin settings once on each device — they'll persist locally.

### Vault appears empty after clone on Android

The clone may have skipped files due to FUSE issues during the initial pull. Reset cleanly:
```bash
rm -rf ~/storage/shared/repos/KBVault
rm -rf ~/obsidian-git
# Then re-run the clone command from Phase 4.4
```

---

## Appendix A — Why This Setup Works

**Why GitHub instead of a self-hosted git server?**
Free, reliable, has uptime guarantees, and doesn't require you to maintain infrastructure. Private repos are unlimited on the free tier. If you'd prefer self-hosted, this guide works identically with Gitea, Forgejo, or any SSH-accessible git server — just replace the remote URL.

**Why SSH instead of HTTPS + Personal Access Tokens?**
SSH keys don't expire. PATs do, and rotating them across three devices is friction you don't need.

**Why the Obsidian Git plugin on PC but Termux on Android?**
The plugin uses `isomorphic-git`, a pure-JavaScript git implementation. On desktop it's reliable. On Android it has known issues with SSH keys, large repos, and certain merge scenarios. Termux runs real `git`, the same binary used on a server, and has none of these issues.

**Why `--separate-git-dir` on Android?**
Android's shared storage uses FUSE (Filesystem in Userspace), a virtual file layer. FUSE can occasionally produce zero-byte files when git writes pack files or loose objects rapidly. Storing `.git` on Termux's native ext4 filesystem (where FUSE isn't involved) eliminates this risk completely.

---

## Appendix B — Recommended `.gitignore` (Copy-Paste)

```
# Obsidian device-specific files
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/community-plugins.json

# Plugin folder — install plugins individually on each device
.obsidian/plugins/

# Obsidian cache
.obsidian/cache/

# Nested vault folders
**/.obsidian/

# Conflict markers from the Obsidian Git plugin
conflict-files-obsidian-git.md

# Trash and OS junk
.trash/
.DS_Store
Thumbs.db
._*
```

---

## Appendix C — Recommended `.gitattributes`

```
* text=auto eol=lf
*.md text eol=lf
```

---

## Appendix D — Sync Script for Android (Copy-Paste)

`~/.shortcuts/sync.sh`:
```bash
#!/bin/bash
eval "$(ssh-agent -s)" > /dev/null
ssh-add ~/.ssh/id_ed25519 2>/dev/null
cd ~/storage/shared/repos/KBVault
git add -A
git commit -m "Android sync $(date +'%Y-%m-%d %H:%M')" 2>/dev/null || true
GIT_MERGE_AUTOEDIT=no git pull --no-rebase origin main
git push origin main
sleep 2
exit
```

Make executable:
```bash
chmod +x ~/.shortcuts/sync.sh
```

---

## Verified On

- **Windows 10 / 11** with PowerShell
- **Android 13 / 14** with Termux from F-Droid
- **Obsidian 1.5+** with Obsidian Git plugin v2.x
- **Git 2.40+** on both Windows and Termux

Linux and macOS commands follow the same pattern — replace `Get-Content ... | clip` with `cat ... | xclip -sel clip` (Linux) or `cat ... | pbcopy` (macOS), and use your distribution's package manager to install Git. The rest of the guide applies unchanged.

---

*If this guide saved you a subscription, consider starring the repo. Issues and PRs welcome — particularly from anyone running this setup on Linux, macOS, or with self-hosted git backends.*
