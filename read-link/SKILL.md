---
name: read-link
description: "Read the content of a web link that blocks automated fetching — X.com / Twitter posts and threads, or any article that returns a login wall, bot block, or empty body. Rewrites the URL through a reader proxy (r.jina.ai), falls back to the Wayback Machine, and asks for pasted text or a screenshot only as a last resort. Use when the user shares an x.com / twitter.com link, when a WebFetch on a URL comes back empty / paywalled / login-gated, or when asked to 'read this link', 'open this tweet', or 'what does this article say'."
argument-hint: "[paste one or more URLs, or invoke right after sharing a blocked link]"
allowed-tools: ["WebFetch", "WebSearch", "Read", "AskUserQuestion"]
---

# Read Link — fetch content behind bot blocks

Some sites (X.com / Twitter above all, plus many paywalled or JS-only articles) refuse
automated fetching: a direct `WebFetch` returns a login wall, a bot-check page, or an
empty body. This skill gets the actual content in as few steps as possible.

## Input

- One or more URLs, pasted as the argument OR present in the immediately preceding
  message. If none is given, ask which link.
- If a URL is an X/Twitter `t.co` wrapper or a post whose *point* is an outbound article
  link, prefer resolving the destination article over the tweet chrome.

## Procedure

Run per URL. Stop at the first step that returns real content.

1. **Classify.**
   - `x.com`, `twitter.com`, `nitter.*` → these are hard-blocked. **Skip straight to
     step 3** (the reader proxy). Do not waste a direct fetch — it will hit a login wall.
   - Any other URL → try step 2 first.

2. **Direct fetch.** `WebFetch` the raw URL. If it returns substantive content, done.
   Treat as a FAILURE (fall through) any of: login/sign-in wall, "enable JavaScript",
   cookie/consent gate, CAPTCHA / bot-check, paywall stub, or a suspiciously empty body.

3. **Reader proxy.** Rewrite to `https://r.jina.ai/<full-original-url>` and `WebFetch`
   that — it renders the page (JS included) and returns clean markdown.
   - Keep the original scheme in the path: `https://r.jina.ai/https://x.com/user/status/123`.
   - This is the primary path for X links and the first fallback for blocked articles.

4. **Wayback.** If the proxy is rate-limited or returns nothing, `WebFetch`
   `https://web.archive.org/web/<full-original-url>` (latest snapshot).

5. **Ask the human.** Only if 2–4 all fail. Use `AskUserQuestion` (or a plain prompt)
   to request either the **pasted text** or a **screenshot** (`Read` handles images).
   For image-heavy or long-thread tweets, a screenshot is often better than any proxy —
   suggest it.

## Output

- Report the content the user actually wanted (summary, quote, or answer to their
  question), not a play-by-play of which fallback fired.
- If you fell back to the proxy or Wayback, note it in one short line so the user knows
  the source may be slightly stale or reformatted.
- Multiple URLs → handle each, label clearly.

## Notes

- The block is IP/bot-based on the platform's side; there is no credential or setting that
  "unblocks" X. The proxy/archive rewrite is the whole trick.
- Nitter instances are mostly dead as of 2026 — don't rely on them; the reader proxy is
  more reliable.
- Never fabricate content when every fetch path fails. Say it's blocked and ask for text.
