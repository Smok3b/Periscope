# Job Ad Monitor

Checks Pracuj.pl, TheProtocol.it, and RocketJobs.pl every 5 days for job ads
matching your keywords, and pushes a notification via [ntfy.io](https://ntfy.sh)
when it finds a new one.

## Setup (about 5 minutes)

### 1. Get a notification channel (ntfy.io)
1. Install the **ntfy** app on your phone ([iOS](https://apps.apple.com/app/ntfy/id1625396347) /
   [Android](https://play.google.com/store/apps/details?id=io.heckel.ntfy)), or just use
   https://ntfy.sh in a browser.
2. Pick a **unique, hard-to-guess topic name** — e.g. `bartosz-job-alerts-x7f2`.
   Anyone who knows the exact topic name can see your notifications, since ntfy's
   free tier has no authentication, so avoid something guessable like `jobs123`.
3. In the app (or on ntfy.sh), subscribe to that topic name.

### 2. Create the GitHub repo
1. Create a new **private** GitHub repository.
2. Upload all the files from this folder, keeping the folder structure
   (the `.github/workflows/job_check.yml` file must stay at that exact path).

### 3. Add your ntfy topic as a repo variable
1. In your new repo: **Settings → Secrets and variables → Actions → Variables** tab.
2. Click **New repository variable**.
3. Name: `NTFY_TOPIC`, Value: the topic name you picked above.
4. Save.

(It's a "variable" not a "secret" because it's easy to change, but treat it as
semi-private — don't publish it anywhere.)

### 4. Enable Actions and test it
1. Go to the **Actions** tab of your repo, and enable workflows if prompted.
2. Click into "Job Ad Monitor" → **Run workflow** to trigger it manually and
   confirm it runs without errors (this first run always does a full check,
   regardless of the 5-day timer).
3. Check your phone / ntfy.sh for a notification if any matches were found.

From then on, it runs automatically every day, but only does a real
check — and only notifies you — every 5 days.

## Customizing

Open `job_monitor.py` and edit:

- **`KEYWORD_GROUPS`** — the phrases to match against job titles. Add more
  variants (different wordings, English translations, etc.) to catch more ads.
- **`CHECK_INTERVAL_DAYS`** — how many days between checks (currently 5).
- **`category_urls`** in `check_rocketjobs()` — RocketJobs.pl doesn't have a
  confirmed keyword-search URL, so it currently scans a few likely categories
  and filters by title. If you find ads slipping through, you can add more
  category URLs there (browse rocketjobs.pl and copy a category URL).

## Notes & limitations

- Matching is done against **job titles only**, not full descriptions — this
  keeps it fast and low on false positives, but a role might be relevant
  without the exact phrase in its title. Add more variants if that happens.
- If a site changes its page layout, the scraper may need small updates
  (the `extract_links()` function's `href_contains` filter is the first
  thing to check).
- GitHub Actions' free tier is generous enough that a 1–2 minute run every
  day comfortably fits within the free monthly minutes for a private repo.
