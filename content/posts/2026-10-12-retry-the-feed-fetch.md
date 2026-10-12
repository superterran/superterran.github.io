---
title: "retry the feed fetch"
date: 2026-10-12
draft: false
categories: ["tools"]
tags: ["github-actions", "rss", "syndication"]
summary: "A scheduled syndication job failed on one transient SSL EOF, so I wrapped the feed fetch in a tiny retry loop instead of treating the first network burp as final."
---

The failing bit was boring in the exact way scheduled jobs are boring:

```text
SSLEOFError: EOF occurred in violation of protocol
```

The syndication job fetches the canonical feed, reads the newest item, then decides what to post elsewhere. One run hit an SSL EOF while fetching `https://doughatcher.com/feed.json`. The site was up. The next manual dry run could fetch the feed fine. Nothing about the content had changed.

So the fix was not a bigger rewrite. It was a three-try loop around the feed request:

```python
for attempt in range(1, MAX_FETCH_ATTEMPTS + 1):
    try:
        return feedparser.parse(FEED_URL)
    except Exception as exc:
        if attempt == MAX_FETCH_ATTEMPTS:
            raise
        print(f"feed fetch failed on attempt {attempt}: {type(exc).__name__}")
        time.sleep(attempt * 5)
```

That is about as fancy as it needed to get. Log the exception type, back off a little, try again. If the feed is really broken, the workflow still fails. If the network sneezes once, the job does not page me with fake work.

I like these little repairs because they make the automation less dramatic without hiding real failures. A retry loop is not a permission slip to ignore bad inputs. It is just an admission that the internet is allowed to be weird for five seconds.
