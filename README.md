# iOS Link Handling Audit — How Third-Party Apps Handle Reddit Links

**Author:** Pavel Kozlov
**Contributors / reviewers:** Sean Sassenrath, Darien French-Owen, Ekaterina Kuzmicheva, Prashant Singh
**Status:** Draft / In Progress
**Created:** Jul 17, 2026
**Based on:** [Design Doc Template (Thin)](https://docs.google.com/document/d/1mGXWQmnbFO5Nu2tTqteEGlxLtlbrSOhRpjxNqjbc2S0/edit)

## Background

A user (Maria) reported that tapping a shared Reddit link inside a third-party app (X) did not open the Reddit native app — it opened inside the third-party app's in-app browser instead. Initial investigation (Ekaterina, Prashant) found this is caused by the third-party app's outbound-link handling, not by Reddit:

- X wraps outbound links using `t.co` and opens the resolved destination in its **in-app browser (SFSafariViewController / custom WebView)**, which prevents the normal OS-level Universal Link / App Link handoff. As a result, Reddit cannot force the initial tap in X to open the native app.
- This pattern (opening external links in in-app browsers to keep users on-platform) is becoming more common across social apps.

Source threads: [primary](https://reddit.slack.com/archives/C0B1H5P24SZ/p1784295980082129), [related](https://reddit.slack.com/archives/C0AU84WF2B1/p1784265832596649).

## Scope & direction

**Direction under audit:** *Inbound to Reddit* — i.e., a user is inside a third-party app, taps a Reddit link, and we want it to open the **Reddit native app** (if installed) via iOS Universal Links, falling back to mWeb only if the app is not installed.

This audit is **not** about the outbound direction (Reddit app → other installed apps).

**Key question (per Sean):** Do any third-party apps have *Reddit-specific* link handling, or do they uniformly trap **all** outbound links in an in-app browser regardless of destination? Early evidence strongly suggests the latter — this is generic on-platform-retention behavior, not Reddit singling. The audit should confirm this and identify any referrers/conditions where the OS Universal Link handoff **does** still succeed.

**Platform scope:** iOS first. Android to be covered in a follow-up (see Android section, deferred).

### In scope
- Behavior when a Reddit link is tapped inside major third-party iOS apps.
- Behavior across the different Reddit link formats we emit (`reddit.com`, short links, MMP links).
- Identifying conditions where native-app handoff succeeds vs. falls back to in-app browser / mWeb.

### Out of scope
- Outbound linking from the Reddit app into other apps.
- Changing third-party apps' behavior (not in our control).
- Android (deferred to follow-up).
- Deep-link routing correctness *after* the app opens (covered by existing deep-link work).

## Why the handoff breaks (technical primer)

iOS Universal Links open the registered app **only** when the link is followed through a context that respects the OS handoff (Safari, Mail, Messages, most native `openURL` calls). They are bypassed when:

- The link is opened inside an in-app browser (`SFSafariViewController` or a custom `WKWebView`) — the OS treats it as in-app web content, so no app handoff occurs. This is the X / Instagram case.
- The link goes through a redirect chain (e.g., `t.co` → destination), which can additionally break Universal Links unless the intermediate domain is itself a registered associated domain.
- The user previously long-pressed a link from that domain and chose "Open in browser" — iOS then sticks with the browser for that domain (not fixable by us).
- The AASA (`apple-app-site-association`) file failed to download on the last app update.

Reference: [iOS Branch integration](https://docs.google.com/document/d/1Cs_bJU-GC7NdiMi4l0xnSkNjLlaDhccFvGr0OrbT3eo) (Universal Links mechanics + failure modes).

## Reddit link formats to test

Each third-party app should be tested against every link format we actually emit, since behavior can differ per format:

| Format | Example | Where it's generated |
|---|---|---|
| Canonical web URL | `https://www.reddit.com/r/pics/comments/abc123/title/` | Copy link, most shares |
| Short link | `https://share.reddit.com/<hash>` (or GraphQL-backed shortener) | Custom share sheet / ShortURL project |
| MMP / app link | `https://reddit.app.link/...` (Branch), migrating to `https://reddit.onelink.me/...` (AppsFlyer) | X-promo, some sharing |

Note: the Branch → AppsFlyer migration is in flight ([design doc](https://docs.google.com/document/d/1HLD0v27BT8ByEUwLl-UuIjvXDYq8FoCprCNHPLFWZKw)), so test both `app.link` and `onelink.me` where applicable.

## Audit matrix — iOS (inbound: third-party app → Reddit)

Legend for "Result": **Native app** = opens Reddit app · **In-app browser** = trapped in 3P WebView/SFSafariViewController · **mWeb (Safari)** = external Safari · **App Store** = install prompt.

| # | Third-party app | Link format | Result | Reddit-specific handling? | Source | Notes |
|---|---|---|---|---|---|---|
| 1 | **X (Twitter)** | canonical / all | **In-app browser** | No — traps all outbound links via `t.co` | Confirmed (Ekaterina, Sean) | `t.co` wrap + in-app browser bypasses Universal Link handoff |
| 2 | **Instagram** (feed/DM) | canonical / all | **In-app browser** | No — embedded browser for all links | Confirmed (Prashant) | Known challenge; bypasses OS URL handling |
| 3 | **Instagram Stories** | canonical | **[TO TEST]** | **[TO TEST]** | Sharing Strategy doc flags "broken link-back" | Also: Stories don't render Reddit content preview (text only) |
| 4 | **Facebook** | canonical | **[TO TEST]** | **[TO TEST]** | Sharing Initiative doc: "share only within Facebook or Messenger" | Likely in-app browser |
| 5 | **Facebook Messenger** | canonical | **[TO TEST]** | **[TO TEST]** | — | — |
| 6 | **Snapchat** | canonical / app link | **[TO TEST]** | **[TO TEST]** | Sharing Strategy doc: "broken link-back functionality" | Reddit uses Branch link for Snapchat sharing today |
| 7 | **WhatsApp** | canonical | **[TO TEST]** — expected **Native app** | **[TO TEST]** | — | WhatsApp typically respects Universal Links (opens Safari/app) |
| 8 | **iMessage** | canonical | **[TO TEST]** — expected **Native app** | No | — | Baseline "good" case; Apple respects Universal Links |
| 9 | **Slack** | canonical | **[TO TEST]** — expected **Native app** | No | iOS Branch doc uses Slack as the canonical working example | Should hand off to Reddit app |
| 10 | **Telegram** | canonical | **[TO TEST]** | **[TO TEST]** | — | Has in-app browser; check default behavior |
| 11 | **LinkedIn** | canonical | **[TO TEST]** | **[TO TEST]** | — | Known aggressive in-app browser |
| 12 | **TikTok** | canonical / short link | **[TO TEST]** | **[TO TEST]** | — | In-app browser expected |
| 13 | **Discord** | canonical | **[TO TEST]** | **[TO TEST]** | — | — |
| 14 | **Gmail (iOS app)** | canonical / `click.redditmail.com` | **[TO TEST]** | **[TO TEST]** | iOS Branch doc: email redirect via `click.redditmail.com` | Redirect handling relevant here |
| 15 | **Chrome (iOS)** | canonical | **[TO TEST]** — expected **Native app** | No | — | Should respect Universal Links |
| 16 | **Safari** | canonical | **Native app** (baseline) | No | Baseline | Control case — confirm handoff works |

For each app, also record:
- Does long-pressing the link offer an "Open in Reddit" option?
- Does behavior differ for a **logged-in vs. logged-out** Reddit app state after handoff?
- Does the in-app browser at least land on **mWeb** with a working content preview, or a broken page?

## Test methodology

1. Device: physical iPhone, latest iOS, Reddit app **installed and logged in**.
2. For each app in the matrix, post/receive a Reddit link in that app and tap it from a natural surface (feed, DM, profile bio, etc.).
3. Record the Result column + notes; capture a screen recording for any failure (as Maria did).
4. Repeat key rows with the Reddit app **not installed** to confirm mWeb / App Store fallback.
5. Repeat with each link format (canonical / short / MMP) where the app is a known sharing target.
6. Note any OS/app version specifics.

## Preliminary findings (from threads + docs, pre-hands-on)

- **X** and **Instagram** trap all outbound links in in-app browsers → Reddit native app does not open. **Confirmed.**
- This is **generic on-platform-retention behavior**, not Reddit-specific handling (aligns with Sean's hypothesis). To be confirmed for the remaining apps.
- **iMessage / Slack / Safari / Chrome / WhatsApp** are expected to respect Universal Links and hand off to the Reddit app — these are the "good path" baselines.
- Deep-link fragility exists even on the good path: users are *sometimes* routed to Web/mWeb despite having the app installed ([Sharing Strategy & Plan](https://docs.google.com/document/d/1IS9H2zIL-qoxnK7Bs8Ok3SPgBTp2kHM63MokQI6cqX0)). The audit should quantify how often.

## Opportunities / recommendations (to refine after audit)

- For apps that **do** respect Universal Links: ensure our AASA config, associated domains, and redirect chains (short link, `click.redditmail.com`, `app.link`/`onelink.me`) reliably trigger native-app handoff. Preferred fallback order: **native app → mWeb** (never a broken/blank page) — Darien: "if there's a way to open in app vs. mWeb… would always be preferred."
- For apps that **trap links in in-app browsers** (X, IG): the initial tap is outside our control, but investigate whether we can improve the **in-app-browser → app** transition once the user lands on mWeb (e.g., a prominent "Open in app" banner / smart app banner / deferred deep link).
- Consider whether Reddit should experiment with our **own** in-app browser behavior for consistency (raised by Sean/Darien) — separate track.

## Android (deferred — follow-up)

Android App Links behave analogously (verified-domain handoff), with its own failure modes (in-app WebViews, referrer stripping). Prashant noted mixed results on Android already (X opened correctly for one link but `reddit.app.link` did **not** open the app). Full Android matrix to be completed in a follow-up pass.

## Appendix — related docs

- [Sharing Initiative - Eng](https://docs.google.com/document/d/1nHuhYX3gtMYJ7cllLvTpkMoNFSAhHgI3MH07MqA0yiA)
- [[WIP] 2026 Sharing Strategy & Plan](https://docs.google.com/document/d/1IS9H2zIL-qoxnK7Bs8Ok3SPgBTp2kHM63MokQI6cqX0)
- [Sharing Opportunity & Plan](https://docs.google.com/document/d/1Jw_z76wqkJiGHRu0yQGCWb5t2oZfb-PIBfC8ztLQPsU)
- [iOS Branch integration](https://docs.google.com/document/d/1Cs_bJU-GC7NdiMi4l0xnSkNjLlaDhccFvGr0OrbT3eo)
- [Branch → AppsFlyer Links Migration](https://docs.google.com/document/d/1HLD0v27BT8ByEUwLl-UuIjvXDYq8FoCprCNHPLFWZKw)
- [reddit-service-share-url](https://docs.google.com/document/d/1GpCboDpzlnLD2qSGFeStnZS_lLkVNLxeIla0THminD0)
- [Sharing Telemetry Audit (Sheet)](https://docs.google.com/spreadsheets/d/1UiyVUJT_lj5Nct4iXsUtCqVq9t7ZuFpNjyd_f4pdspg)
