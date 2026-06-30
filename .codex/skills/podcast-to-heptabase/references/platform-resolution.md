# Platform Resolution Reference

Use this only when the audio URL or RSS feed is not obvious.

## Apple Podcasts

Apple episode URLs usually contain a show id and `i=<episode-id>`.

1. Extract the show id from `/id<digits>`.
2. Fetch show metadata:

```bash
curl -s "https://itunes.apple.com/lookup?id=<show-id>"
```

3. Read `feedUrl` from the JSON.
4. Parse the RSS with `feedparser` and match the item by title, GUID, publish date, or audio enclosure. Apple `i=` ids are not always present in RSS, so use the page title/description if needed.

Example parser:

```bash
uv run --with feedparser --with beautifulsoup4 python - <<'PY'
import feedparser
from bs4 import BeautifulSoup

feed_url = "https://example.com/rss"
needle = "EP115"
d = feedparser.parse(feed_url)
for e in d.entries:
    if needle in e.get("title", ""):
        text = BeautifulSoup(e.get("description", ""), "html.parser").get_text("\n")
        print(e.get("title"))
        print(e.get("published"))
        print(e.get("itunes_duration"))
        print(e.get("enclosures"))
        print(text[:2000])
PY
```

## Firstory

Firstory feeds often use:

```text
https://feed.firstory.me/rss/user/<user-id>
```

The RSS item usually contains a stable GUID and an audio `enclosure` URL. If an `open.firstory.me/story/<id>` URL is provided, search the RSS for that story id.

## SoundOn

Try RSS/page discovery first:

- Inspect the episode page for `<link rel="alternate" type="application/rss+xml">`.
- Search the page HTML for `rss`, `mp3`, `m4a`, `episode`, `audio`, or JSON-LD.
- If the page hides the RSS, search the web for the show title plus `RSS` or use another public podcast directory to find the same feed.

Avoid assuming one SoundOn URL pattern is stable; verify by opening the page or RSS.

## Spotify

Spotify episode pages often do not expose the RSS directly.

Recommended path:

1. Extract the episode/show title from the page metadata.
2. Search for the show title plus `RSS`, Apple Podcasts, Firstory, SoundOn, Anchor, or the publisher website.
3. Use the canonical RSS enclosure from the publisher when found.
4. If public extraction with `yt-dlp` is available and appropriate, use it as a fallback:

```bash
yt-dlp --print title --print duration --print urls '<spotify-or-platform-url>'
```

If Spotify requires login or DRM-protected access, stop and ask the user for another public source.

## Direct RSS

If the user provides an RSS URL, parse it directly and match the desired item. If no episode is specified and the URL is a show feed, ask whether to use the latest episode or a specific one.

## Common Fallbacks

- Open the page HTML and search for `application/ld+json`, `enclosure`, `audio`, `mp3`, `m4a`, `feed`, `rss`, `episode`.
- Use `curl -L -I` to follow redirects and verify content type.
- Use `ffprobe` after download to confirm the audio file is valid and complete.
- If installing command-line tools is needed, follow local dotfiles rules; for Homebrew tools, update `/Users/chswei/.dotfiles/Brewfile` via `brew bundle dump` and commit with `dotsync` when appropriate.
