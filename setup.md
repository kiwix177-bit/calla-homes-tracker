# Calla Homes price tracker - setup on the laptop

This folder is self-contained: copy the whole thing to the always-on laptop
and follow these steps. No `pip install` is required. The only two things it
needs are Python itself and `curl.exe`, which ships built into Windows 10
(1803+) and Windows 11 - virtually every modern Windows machine already has
both.

Why curl instead of Python's own HTTP client: the site sits behind
Cloudflare, which actively fingerprints and JS-challenges Python's built-in
`urllib`/OpenSSL TLS handshake even with a perfect browser `User-Agent`
header, but lets plain `curl.exe` through. Shelling out to `curl.exe` for
the actual page fetch (confirmed working against the live site during
development) sidesteps that without adding any pip dependency.

## 1. Confirm Python and curl are installed

Open a terminal (PowerShell or cmd) and run:

```
python --version
curl.exe --version
```

You need Python 3.8+ and any recent curl. If `curl.exe --version` fails,
your Windows build predates the bundled curl (rare on anything from the
last few years) - installing curl separately or updating Windows resolves
it. If you get "not found" or the Microsoft Store prompt instead of a
Python version number:

1. Download the installer from https://www.python.org/downloads/windows/
2. Run it, and **check "Add python.exe to PATH"** on the first screen before
   clicking Install.
3. Open a new terminal and re-run `python --version` to confirm.

## 2. Copy the folder

Copy this entire `CallaHomesTracker` folder anywhere on the laptop (e.g.
`C:\Users\<you>\CallaHomesTracker`). All paths in the scripts are relative to
this folder, so it doesn't matter exactly where it lives.

## 3. Test it manually once

From inside the folder:

```
python scrape.py
```

This should print a line ending in `OK: N floor plans, M units scraped,
report updated`, and create/update three files: `history.json`,
`report.html`, and `scrape.log`. Open `report.html` in a browser to see the
current listing.

Run it a second time (even a minute later) - it should still say `OK` and
`report.html` should look the same, just with a later "Last checked" time.
Nothing breaks from running it more than once a day; it only adds a new
price point to `history.json` per calendar day per unit (re-running same-day
just overwrites that day's price with the latest read).

## 4. Register the daily task

Still from inside the folder, run (default time is 8:00 AM - pass `-Time`
to change it, e.g. `-Time 18:30`):

```
powershell -ExecutionPolicy Bypass -File .\setup_task.ps1
```

This registers a Windows Scheduled Task named `CallaHomesTracker` that runs
`run_daily.bat` once a day.

To confirm it's registered:

```
Get-ScheduledTask -TaskName CallaHomesTracker
```

To change the time later, just re-run `setup_task.ps1 -Time <HH:mm>` - it
overwrites the existing task rather than creating a duplicate.

To run it on demand (without waiting for the schedule):

```
Start-ScheduledTask -TaskName CallaHomesTracker
```

## 5. View the report from your phone/other devices via GitHub Pages

Google Drive does **not** render `.html` files as a page when shared - it
just shows the source as text, so it doesn't work for this. GitHub Pages
does: it's free static web hosting, and the file is served as a real
webpage over HTTPS, viewable from any browser on any device, no port
forwarding needed.

`DRIVE_SYNC_DIR` in `scrape.py` still works if you want a private backup
copy in Drive, but it won't render as a page - use `GITHUB_PAGES_DIR` for
that.

### One-time setup

**A) Create a GitHub account** (skip if you already have one) at
https://github.com/signup.

**B) Create a new repository:**
1. Go to https://github.com/new
2. Repository name: `calla-homes-tracker` (or anything you like)
3. Set it to **Public** (GitHub Pages' free tier requires a public repo)
4. Leave "Add a README" unchecked
5. Click **Create repository**

**C) Turn on GitHub Pages for it:**
1. In the new repo, go to **Settings → Pages** (left sidebar)
2. Under "Build and deployment" → Source, choose **Deploy from a branch**
3. Branch: **main**, folder: **/ (root)** → **Save**
4. GitHub shows a URL like `https://<your-username>.github.io/calla-homes-tracker/`
   - it won't work yet (nothing's pushed) - that's expected for now.

**D) Create a Personal Access Token** (lets the script push automatically,
without you typing a password every day):
1. Go to https://github.com/settings/personal-access-tokens/new
2. Token name: `calla-homes-tracker`
3. Expiration: 1 year (the longest fine-grained tokens allow - you'll just
   need to generate a new one when it expires)
4. Repository access: **Only select repositories** → pick `calla-homes-tracker`
5. Under **Permissions → Repository permissions**, find **Contents** and set
   it to **Read and write**
6. Click **Generate token**, then **copy the token immediately** - GitHub
   only shows it once. It looks like `github_pat_11ABCDEFG...`.

**E) Install Git for Windows** (skip if `git --version` already works):
download from https://git-scm.com/download/win and run the installer with
default options.

**F) One-time git identity** (only needed if you've never used git before -
this is what goes on your commits, can be anything):
```
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

**G) Clone the repo into a dedicated local folder**, with the token baked
into the URL so pushes don't prompt for credentials. Replace
`<TOKEN>` and `<your-username>` with your actual values:
```
git clone https://<TOKEN>@github.com/<your-username>/calla-homes-tracker.git C:\Users\<you>\calla-homes-site
```

**H) Point the script at it** - open `scrape.py` and edit the
`GITHUB_PAGES_DIR` line near the top:
```python
GITHUB_PAGES_DIR = r"C:\Users\<you>\calla-homes-site"
```

**I) Run it once:**
```
python scrape.py
```
The log should show `OK: report.html pushed to GitHub Pages (...)`. Check
https://github.com/`<your-username>`/calla-homes-tracker - `report.html`
should be there. Give GitHub Pages a minute or two to build the first time,
then open:
```
https://<your-username>.github.io/calla-homes-tracker/report.html
```
That's the link to bookmark on your phone/other computers - it's a real
page now, not a download or a source-code view, and it updates in place
every day the scheduled task runs.

### If something goes wrong

- `WARNING: GITHUB_PAGES_DIR is not a git repo` - the folder either doesn't
  exist or step G's `git clone` didn't complete. Re-run the clone command.
- `git push failed: ... Authentication failed` - the token in the clone
  URL is wrong, expired, or was pasted incorrectly. Easiest fix: delete the
  `calla-homes-site` folder and redo step G with a fresh token.
- The GitHub Pages URL 404s - double check Settings → Pages shows a green
  "Your site is live at ..." banner; it can take a minute or two after the
  first push.

## 6. Day to day

- Open `report.html` any time - it's a plain file, safe to open while the
  task is or isn't running, and gets fully overwritten (regenerated) on each
  daily run. Bookmark/pin it if you want quick access.
- Each unit card shows its current price, a small trend line of its price
  history, and how it changed since the last check. A unit that stops
  appearing on the site gets greyed out with a "Now gone since `<date>`"
  badge instead of being deleted.
- `scrape.log` has one line per run. A healthy line looks like:
  `2026-07-09T08:00:03 OK: 2 floor plans, 20 units scraped, report updated`
  A failed run looks like:
  `2026-07-09T08:00:05 ERROR: run aborted, history/report untouched: <reason>`
  On an ERROR line, `history.json`/`report.html` are left exactly as they
  were the day before - a failed run never wipes or corrupts existing data.
  If you see repeated ERROR lines, check what they say:
  - `curl failed (exit ...) fetching ...` with a 403 - Cloudflare is
    blocking even curl now (rare, but possible if they tighten rules).
    Usually resolves itself; if not, the fetch approach needs revisiting.
  - `No /floorplans/<slug> links found` - the site likely redesigned the
    page and the parser needs updating.
  - `curl.exe not found on PATH` - see the curl check in step 1.

## Files in this folder

| File | Purpose |
|---|---|
| `scrape.py` | The scraper + report generator. This is what actually runs daily. |
| `history.json` | Price history data store. Back this up if you care about long-term history. |
| `report.html` | The page you actually look at. Regenerated every run. |
| `run_daily.bat` | Thin wrapper Task Scheduler calls. |
| `setup_task.ps1` | One-time helper to register the daily Scheduled Task. |
| `scrape.log` | Append-only run log. |
