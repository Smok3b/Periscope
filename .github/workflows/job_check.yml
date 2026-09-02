#!/usr/bin/env python3
"""
Job ad monitor for Pracuj.pl, TheProtocol.it, and RocketJobs.pl.

Checks each site for job ads matching configured keyword phrases, and sends
a push notification (via ntfy.io) for any ad it hasn't already notified about.

Designed to be run on a schedule (see .github/workflows/job_check.yml).
Because true "every N days" scheduling isn't directly supported by cron,
this script is meant to be triggered daily, but will exit immediately
without doing any work if fewer than CHECK_INTERVAL_DAYS have passed
since the last real check. That gives a genuine 5-day cadence regardless
of month boundaries.
"""

import json
import os
import re
import sys
import time
import unicodedata
from datetime import datetime, timezone
from pathlib import Path
from urllib.parse import quote

import requests
from bs4 import BeautifulSoup
from playwright.sync_api import sync_playwright

# ---------------------------------------------------------------------------
# CONFIG — edit this section to change what you're looking for
# ---------------------------------------------------------------------------

# How often (in days) a real check should actually happen.
# The GitHub Actions workflow runs this script daily, but it will only
# do a full check every CHECK_INTERVAL_DAYS days.
CHECK_INTERVAL_DAYS = 5

# Each entry: a short label, plus a list of phrase variants.
# A job ad matches a keyword group if its TITLE contains ANY of the
# variants (case-insensitive, accent-insensitive substring match).
KEYWORD_GROUPS = [
    {
        "label": "Koordynator projektów gamingowych",
        "variants": [
            "koordynator projektow gamingowych",
            "koordynator projektu gamingowego",
            "koordynatorka projektow gamingowych",
            "game project coordinator",
            "gaming project coordinator",
        ],
    },
    {
        "label": "Wykładowca game development",
        "variants": [
            "wykladowca game development",
            "wykladowca gamedev",
            "wykladowca gamedevelopment",
            "wykladowca projektowania gier",
            "wykladowca tworzenia gier",
        ],
    },
    {
        "label": "Wykładowca project management",
        "variants": [
            "wykladowca project management",
            "wykladowca zarzadzania projektami",
        ],
    },
]

NTFY_TOPIC = os.environ.get("NTFY_TOPIC", "").strip()
NTFY_URL = f"https://ntfy.sh/{NTFY_TOPIC}" if NTFY_TOPIC else None

STATE_PATH = Path(__file__).parent / "state.json"

# ---------------------------------------------------------------------------
# SITE DEFINITIONS
# ---------------------------------------------------------------------------
# Each site provides a function that, given a Playwright page, returns a
# list of (title, url) tuples found on that site's listing/search page.


def _abs_url(base, href):
    if href.startswith("http"):
        return href
    return base.rstrip("/") + "/" + href.lstrip("/")


def fetch_rendered_html(page, url, wait_selector=None, timeout=25000):
    page.goto(url, timeout=timeout, wait_until="domcontentloaded")
    try:
        page.wait_for_load_state("networkidle", timeout=timeout)
    except Exception:
        pass
    if wait_selector:
        try:
            page.wait_for_selector(wait_selector, timeout=8000)
        except Exception:
            pass
    return page.content()


def extract_links(html, base_url, href_contains):
    soup = BeautifulSoup(html, "html.parser")
    found = {}
    for a in soup.find_all("a", href=True):
        href = a["href"]
        if href_contains not in href:
            continue
        title = a.get_text(strip=True)
        if not title:
            # sometimes the link wraps an image/logo with no text; look for
            # a heading inside a nearby parent instead
            heading = a.find(["h1", "h2", "h3"])
            if heading:
                title = heading.get_text(strip=True)
        if not title:
            continue
        url = _abs_url(base_url, href)
        # Keep the longest title seen for a given URL (avoids picking up
        # short duplicate/logo links pointing at the same job).
        if url not in found or len(title) > len(found[url]):
            found[url] = title
    return list(found.items())


def normalize(text):
    text = text.lower()
    # "ł" doesn't decompose via NFKD like ą/ę/ć/ó/ś/ź/ż do, so handle it explicitly.
    text = text.replace("ł", "l")
    text = unicodedata.normalize("NFKD", text)
    text = "".join(c for c in text if not unicodedata.combining(c))
    return text


def title_matches(title, variants):
    norm_title = normalize(title)
    for variant in variants:
        if normalize(variant) in norm_title:
            return True
    return False


def check_pracuj(page, all_matches, seen):
    base = "https://www.pracuj.pl"
    for group in KEYWORD_GROUPS:
        search_term = group["variants"][0].replace("_", " ")
        url = f"{base}/praca/{quote(search_term)};kw"
        try:
            html = fetch_rendered_html(page, url, wait_selector="a[href*=',oferta,']")
        except Exception as e:
            print(f"[pracuj.pl] failed to load search for '{group['label']}': {e}")
            continue
        links = extract_links(html, base, ",oferta,")
        for job_url, title in links:
            if title_matches(title, group["variants"]) and job_url not in seen:
                all_matches.append(("Pracuj.pl", group["label"], title, job_url))


def check_theprotocol(page, all_matches, seen):
    base = "https://theprotocol.it"
    for group in KEYWORD_GROUPS:
        search_term = group["variants"][0].replace("_", " ")
        url = f"{base}/praca?kw={quote(search_term)}"
        try:
            html = fetch_rendered_html(page, url, wait_selector="a[href*=',oferta,']")
        except Exception as e:
            print(f"[theprotocol.it] failed to load search for '{group['label']}': {e}")
            continue
        links = extract_links(html, base, ",oferta,")
        for job_url, title in links:
            if title_matches(title, group["variants"]) and job_url not in seen:
                all_matches.append(("TheProtocol.it", group["label"], title, job_url))


def check_rocketjobs(page, all_matches, seen):
    base = "https://rocketjobs.pl"
    # RocketJobs doesn't have a simple reliable keyword-in-URL search we could
    # confirm, so we scan the broader listing categories most likely to
    # contain these roles, then filter by title ourselves.
    category_urls = [
        f"{base}/oferty-pracy/wszystkie-lokalizacje/pm",     # project management category
        f"{base}/oferty-pracy/wszystkie-lokalizacje/edukacja",  # education / lecturer roles
        f"{base}/oferty-pracy/wszystkie-lokalizacje/it",
    ]
    for url in category_urls:
        try:
            html = fetch_rendered_html(page, url, wait_selector="a[href*='/oferta-pracy/']")
        except Exception as e:
            print(f"[rocketjobs.pl] failed to load {url}: {e}")
            continue
        links = extract_links(html, base, "/oferta-pracy/")
        for job_url, title in links:
            for group in KEYWORD_GROUPS:
                if title_matches(title, group["variants"]) and job_url not in seen:
                    all_matches.append(("RocketJobs.pl", group["label"], title, job_url))


# ---------------------------------------------------------------------------
# STATE HANDLING
# ---------------------------------------------------------------------------

def load_state():
    if STATE_PATH.exists():
        try:
            return json.loads(STATE_PATH.read_text(encoding="utf-8"))
        except Exception:
            pass
    return {"seen_urls": [], "last_run": None}


def save_state(state):
    STATE_PATH.write_text(json.dumps(state, indent=2, ensure_ascii=False), encoding="utf-8")


def days_since(iso_timestamp):
    if not iso_timestamp:
        return None
    last = datetime.fromisoformat(iso_timestamp)
    return (datetime.now(timezone.utc) - last).total_seconds() / 86400


# ---------------------------------------------------------------------------
# NOTIFICATIONS
# ---------------------------------------------------------------------------

def send_notification(site, label, title, url):
    if not NTFY_URL:
        print(f"[notify skipped, NTFY_TOPIC not set] {site} | {label} | {title} | {url}")
        return
    try:
        requests.post(
            NTFY_URL,
            data=title.encode("utf-8"),
            headers={
                "Title": f"New job: {site} ({label})".encode("utf-8"),
                "Click": url,
                "Priority": "high",
                "Tags": "briefcase",
            },
            timeout=15,
        )
    except Exception as e:
        print(f"Failed to send notification for {url}: {e}")


# ---------------------------------------------------------------------------
# MAIN
# ---------------------------------------------------------------------------

def main():
    state = load_state()

    elapsed = days_since(state.get("last_run"))
    if elapsed is not None and elapsed < CHECK_INTERVAL_DAYS:
        print(f"Only {elapsed:.1f} days since last check (need {CHECK_INTERVAL_DAYS}); skipping.")
        return

    seen = set(state.get("seen_urls", []))
    all_matches = []

    with sync_playwright() as p:
        browser = p.chromium.launch()
        page = browser.new_page(
            user_agent=(
                "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
                "(KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
            )
        )

        check_pracuj(page, all_matches, seen)
        check_theprotocol(page, all_matches, seen)
        check_rocketjobs(page, all_matches, seen)

        browser.close()

    print(f"Found {len(all_matches)} new matching job ad(s).")
    for site, label, title, url in all_matches:
        print(f"  - [{site}] ({label}) {title} -> {url}")
        send_notification(site, label, title, url)
        seen.add(url)

    # Keep state file from growing forever: cap to the most recent 2000 URLs.
    seen_list = list(seen)[-2000:]

    state["seen_urls"] = seen_list
    state["last_run"] = datetime.now(timezone.utc).isoformat()
    save_state(state)


if __name__ == "__main__":
    main()
