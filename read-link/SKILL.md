---
name: read-link
description: "Read the content of a web link that blocks automated fetching — X.com / Twitter posts and threads, or any article that returns a login wall, bot block, or empty body. For X/Twitter it uses the key-free fxtwitter/vxtwitter embed API; for articles it tries a direct fetch, then reader proxies (r.jina.ai, allorigins), then the Wayback Machine, and asks for pasted text or a screenshot only as a last resort. Use when the user shares an x.com / twitter.com link, when a WebFetch on a URL comes back empty / paywalled / login-gated, or when asked to 'read this link', 'open this tweet', or 'what does this article say'."
argument-hint: "[paste one or more URLs, or invoke right after sharing a blocked link]"
allowed-tools: ["WebFetch", "WebSearch", "Read", "AskUserQuestion", "Bash(curl:*)"]
---

# Read Link — fetch content behind bot blocks

Some sites refuse automated fetching: a direct `WebFetch` returns a login wall, a
bot-check page, or an empty body. X.com / Twitter is the worst offender (it returns
`402`/`403` to any non-browser). This skill gets the actual content in as few steps as
possible, using a different ladder for tweets vs. articles.

## Input

- One or more URLs, pasted as the argument OR present in the immediately preceding
  message. If none is given, ask which link.
- If an X/Twitter post's *point* is an outbound article link, read BOTH the tweet (for
  the author's take) and resolve the destination article.

## A. X / Twitter links — use the embed API (no key, no login)

Matches `x.com`, `twitter.com`, `mobile.twitter.com`, `fixupx.com`, `vxtwitter.com`,
`fxtwitter.com`, `nitter.*`.

1. **Parse** `<user>` and the numeric `<id>` from the URL
   (`.../<user>/status/<id>`). If the path is `/i/status/<id>` with no username, use
   `i` as the username.
2. **fxtwitter** — `WebFetch https://api.fxtwitter.com/<user>/status/<id>`. It returns
   JSON; read `tweet.text` (verbatim), `tweet.author.name`, `tweet.quote.text` for a
   quoted tweet, and `tweet.media` alt-text / URLs. This is the primary path and works
   without any credential.
3. **vxtwitter** — if fxtwitter errors, `WebFetch https://api.vxtwitter.com/<user>/status/<id>`
   (JSON: `text`, `user_name`, `qrtURL`/quoted text).
4. Fall through to section C only if BOTH embed APIs fail.

> These embed APIs render the tweet server-side and hand back plain JSON, so they sail
> past the `402`/`403` bot wall that blocks direct x.com fetches and (now) anonymous
> r.jina.ai. Prefer them for anything on X.

## B. Regular article / web links

1. **Direct fetch.** `WebFetch` the raw URL. If it returns substantive content, done.
   Treat as a FAILURE (fall through) any of: login/sign-in wall, "enable JavaScript",
   cookie/consent gate, CAPTCHA / bot-check, paywall stub, or a suspiciously empty body.
2. Fall through to section C.

## C. Shared fallback ladder (blocked article, or X embed APIs down)

Try in order; stop at the first that returns real content.

1. **Reader proxy — r.jina.ai.**
   - If `JINA_API_KEY` is set in the environment, it must be sent as a Bearer header,
     which `WebFetch` cannot do — use Bash:
     `curl -sS -H "Authorization: Bearer $JINA_API_KEY" "https://r.jina.ai/<full-url>"`
   - If no key is set, anonymous r.jina.ai now returns `403` — skip it rather than
     wasting the call.
2. **allorigins proxy.** `WebFetch https://api.allorigins.win/raw?url=<URL-ENCODED-url>`
   — a key-free proxy that returns the raw upstream body. Good for JS-light articles.
3. **Wayback.** `WebFetch https://web.archive.org/web/<full-url>` (latest snapshot).
   Note: some sandboxes block `web.archive.org` outright — if the fetch tool refuses the
   host, skip.
4. **Ask the human.** Only if everything above fails. Use `AskUserQuestion` (or a plain
   prompt) to request either **pasted text** or a **screenshot** (`Read` handles images).
   For image-heavy or long-thread tweets, a screenshot is often best — suggest it.

## Output

- Report the content the user actually wanted (summary, quote, or answer to their
  question), not a play-by-play of which fallback fired.
- Add one short line naming the source path used (e.g. "via fxtwitter embed API",
  "via Wayback snapshot") so the user knows if content may be reformatted or stale.
- Multiple URLs → handle each, label clearly.

## Notes

- The block is IP/bot-based on the platform's side; there is no credential that
  "unblocks" X. The embed API / proxy / archive rewrites are the whole trick.
- Nitter instances are mostly dead as of 2026 — don't rely on them; the fxtwitter/
  vxtwitter APIs are far more reliable for X.
- Never fabricate content when every fetch path fails. Say it's blocked and ask for text.
