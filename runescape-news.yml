#!/usr/bin/env python3
"""Post new RuneScape news to a Discord webhook.

Setup:
    pip install requests feedparser
    export DISCORD_WEBHOOK_URL="https://discord.com/api/webhooks/..."
    python runescape_news_webhook.py

Run it on a schedule (cron, Task Scheduler, GitHub Actions) every 15-30 min.
The first run only records what's currently published (no spam); later runs
post anything new.
"""
import argparse
import json
import os
import sys
from pathlib import Path

import feedparser
import requests

# Official RuneScape news feed. Open it in a browser once to confirm it loads.
FEED_URL = "https://secure.runescape.com/m=news/latest_news.rss"
STATE_FILE = Path(__file__).with_name("runescape_seen.json")
WEBHOOK_URL = os.environ.get("DISCORD_WEBHOOK_URL")
EMBED_COLOR = 0x1C2C34  # matches the site's theme color
MAX_PER_RUN = 10  # Discord allows 10 embeds per message; also a flood guard


def load_seen():
    if STATE_FILE.exists():
        return set(json.loads(STATE_FILE.read_text()))
    return None  # None = first run


def save_seen(seen):
    STATE_FILE.write_text(json.dumps(sorted(seen)))


def build_embed(entry):
    embed = {
        "title": entry.get("title", "RuneScape news")[:256],
        "url": entry.link,
        "color": EMBED_COLOR,
        "author": {"name": "RuneScape News"},
    }
    summary = entry.get("summary", "")
    if summary:
        embed["description"] = summary[:300]
    tags = [t.get("term") for t in entry.get("tags", []) if t.get("term")]
    if tags:
        embed["footer"] = {"text": ", ".join(tags)[:100]}
    if entry.get("published"):
        embed["timestamp"] = None  # filled below if parseable
        if entry.get("published_parsed"):
            from datetime import datetime, timezone

            embed["timestamp"] = datetime(
                *entry.published_parsed[:6], tzinfo=timezone.utc
            ).isoformat()
        else:
            del embed["timestamp"]
    return embed


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--latest",
        type=int,
        metavar="N",
        help="Post the N most recent articles right now, ignoring saved state.",
    )
    args = parser.parse_args()

    if not WEBHOOK_URL:
        sys.exit("Set DISCORD_WEBHOOK_URL first.")

    feed = feedparser.parse(FEED_URL)
    if feed.bozo and not feed.entries:
        sys.exit(f"Could not read feed: {feed.bozo_exception}")

    if args.latest:  # manual mode: grab the last N posts, don't touch saved state
        batch = feed.entries[: min(args.latest, MAX_PER_RUN)]
        for entry in reversed(batch):  # oldest first so newest lands at the bottom
            r = requests.post(
                WEBHOOK_URL,
                json={"username": "RuneScape News", "embeds": [build_embed(entry)]},
                timeout=15,
            )
            r.raise_for_status()
        print(f"Posted the latest {len(batch)} article(s).")
        return

    seen = load_seen()
    current_ids = {e.get("id") or e.link for e in feed.entries}

    if seen is None:  # first run: record, don't post
        save_seen(current_ids)
        print(f"Seeded {len(current_ids)} existing items.")
        return

    new_entries = [e for e in feed.entries if (e.get("id") or e.link) not in seen]
    new_entries.reverse()  # oldest first
    new_entries = new_entries[:MAX_PER_RUN]

    for entry in new_entries:
        r = requests.post(
            WEBHOOK_URL,
            json={"username": "RuneScape News", "embeds": [build_embed(entry)]},
            timeout=15,
        )
        if r.status_code == 429:  # rate limited; stop and retry next run
            print("Rate limited, will retry next run.")
            break
        r.raise_for_status()
        seen.add(entry.get("id") or entry.link)

    save_seen(seen | current_ids if not new_entries else seen)
    print(f"Posted {len(new_entries)} new item(s).")


if __name__ == "__main__":
    main()
