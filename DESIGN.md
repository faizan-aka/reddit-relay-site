# reddit-relay: design

Design only, written 28 Oct 2026. Nothing has been built yet. The Devvit, Discord and Gemini facts were re-checked
against their docs on 28 Oct 2026; §7 lists the sources. Anything marked **(verify)** is not stated in the docs and is
checked in the playtest (§4.7) before anything depends on it.

---

## 0. In one page

**Why.** The perfume, sneaker and protein notifiers read Reddit through the Arctic Shift archive, and the archive stops
when Reddit ends public Data API access (new requests close 31 Oct 2026, unregistered apps lose access from 12 Jan
2027, all public access ends March 2027). The replacement is a private app on Reddit's Developer Platform (Devvit).
It runs on Reddit's servers, reads the watched subreddits on a schedule, and sends each new post or comment to Discord.
Each PC reads its notifier's Discord channel. Reddit's Devvit team and the perfume subreddits' moderators
(r/DesiFragranceAddicts, r/IndiaFragMarketplace, r/PerfumeIndia) said yes on 27 Oct 2026, and the sneaker and protein
subs' moderators agreed too (confirmed 28 Oct 2026). The sneaker and protein subs are read only once those notifiers
are switched on.

**What gets built.**

1. **`reddit-relay`**: a Devvit Web app in TypeScript, server only, at `/Users/faizan/Documents/Code/reddit-relay`. It is
   installed in the private playtest subreddit that `npx devvit upload` creates (you are its moderator, under 200
   members). Its jobs:
   - Every minute: read the newest 100 posts of every watched sub, and comments post by post where they are switched on.
   - Every hour: read 400 posts of r/IndiaFragMarketplace (IFM) to catch held posts that mods approved later.
   - Every 10 minutes: re-check relayed posts for deletion, and send a heartbeat.
   - Each new item goes out as **one signed JSON message** to the notifier's channel through that channel's webhook.
2. **`src/sources/discordRelay.js` + `src/relayStep.js`** in the perfume notifier. They read the `#perfume` channel with
   one bot token, turn messages into the item shape that `toItem` makes, apply purge notices and heartbeats, then
   delete what they read. A `relay.source` setting picks `"archive"` (today), `"shadow"` (read and compare, never
   alert) or `"relay"` (relay first, archive second while it lives).
3. **Paid Gemini** for Reddit images in the perfume notifier (README, config and WINDOWS text). No logic change is
   needed; one small safety guard is recommended (§3.6).

**Decisions at a glance**

| Question | Decision | Why (short) |
|---|---|---|
| Transport | Discord: one channel and one webhook per notifier; one reader bot | Your choice. discord.com is on Devvit's global fetch allowlist, and Discord keeps history so a PC can catch up |
| Message format | One JSON payload per item, signed with HMAC-SHA256. **Inline in `content` when the JSON is ≤ 1,900 characters, otherwise as an attached `.json` file** | Exact text at any length, one Discord message per item (purging means one delete), no reassembly on the PC (§1.9.2) |
| Text caps | Title ≤ 300, post body ≤ 8,000, comment ≤ 4,000 code points | This is 2× `TEXT_CAPS`. `cleanText` in identify.js and sneaker's decide.js never read more than that, so data is kept to a minimum and decisions don't change |
| Usernames | Never sent or stored. Instead: `ak` (author kind), `op` (comment by the post's author), `same` (ids of the author's other posts in that same listing), `seller` (registry seller id) | Devvit Rules on tracking and on deleted accounts. The notifiers only need these relations, not names |
| Seller trust | The app works out the **registry seller id** (`handleHash` match, or the IFM seller-flair path). The notifier still decides trust (sheet, your picks) | The notifier owns trust, and the app never sends hashes |
| Held IFM posts | Seen only once approved. The shallow pass covers about 2 days, the hourly deep pass 400 posts (about 8 days). Dedupe is **by id**. `late: true` | A non-moderator app can't see AutoMod's queue (§5, risk R2) |
| Deletions | The app polls relayed posts at +1 h, +6 h, +1 d, +3 d, +7 d, +14 d, +28 d, then every 30 days while the notifier keeps records (perfume and sneaker 400 d, protein 14 d). It deletes the Discord message and sends a `purge` notice; the PC deletes everything derived from it | No delete events reach an app from other subs. The PCs keep Reddit-derived prices for 400 days |
| Redis expiry | Anything with Reddit content ≤ 48 h; dedupe sets ≤ 30 d; **one exception:** id-only re-check records live for the notifier's `keepDays` | Content and user data within 30 days. Id-only records are what makes the 400-day history deletable (open question Q1) |
| Discord failures | Redis outbox, ≤ 1 request/s, obey `X-RateLimit-*` and `retry_after`, back off on 5xx, drop after 48 h | Discord's limits; Devvit's 30-s request limit |
| PC health | New `relay-down` and `relay-stalled` kinds in the Reddit family. A quiet note first; loud only after a day of observed trouble *and* no other source covering | Your quiet-first rule (sneaker SPEC §15.17, perfume `health.js`) |
| Gemini | A new Google Cloud project with billing on, about ₹0.1–0.2 an image, at most about ₹360 a month on the PC | Devvit Rules on GenAI. Paid-tier inputs aren't used to improve Google's products |

---

## 1. The Devvit app

### 1.1 Platform facts this design rests on

| Fact (checked 28 Oct 2026) | Value | Source |
|---|---|---|
| Config file | `devvit.json` (`devvit.yaml` support removed in 0.15.0, 19 Oct 2026; 0.15.1 on 26 Oct had no user-facing changes) | changelog.md |
| Server bundle | must be CommonJS; `server.dir` default `dist/server`, `entry` default `index.js` | devvit_web_configuration.md |
| Server limits | "Max request time: 30s", "Max payload size: 4MB" | devvit_web_overview.mdx, faq.mdx |
| App name | 3–20 characters, must start with a letter (it is also the app account's name) | devvit_web_configuration.md |
| Scheduler | per installation; standard cron "one run per minute"; six-field seconds cron is experimental; **at most 10 live recurring jobs**; `runJob()` ≤ 60 calls/min; how often a job really runs "will depend on how long the job takes to complete and how many jobs are running in parallel"; cron jobs in devvit.json are "rescheduled on installation and upgrade" | scheduler.mdx |
| Task endpoint | `/internal/...`, POST with body `{ data }`, answer `{ status: "ok" }` | scheduler.mdx |
| HTTP fetch | https only; GET, POST, PUT, DELETE, OPTIONS, PATCH; 30-s timeout; **discord.com and api.telegram.org are on the global allowlist** (no review); other domains are reviewed (usually 1–2 business days); apps that fetch need Terms, a Privacy Policy and a README "Fetch Domains" section | http-fetch.mdx |
| Settings | `global` (developer) and `subreddit` (moderators, on the Install Settings page); types string, paragraph, number, boolean, select, multiSelect; **2 KB per value**; secrets are global only, set with `npx devvit settings set <name>`, and need at least one installation first | settings-and-secrets.mdx, devvit_web_configuration.md |
| Redis | per installation; 5 GB, 5 MB per request, 40,000 commands/s, 20 concurrent transactions, 5-s transaction timeout; strings, hashes, sorted sets, expire, watch/multi/exec; **no SCAN, no plain sets or lists** | redis.mdx |
| Reading other subs | `reddit.getNewPosts({ subredditName, limit (max 1000), pageSize, after, before })` returns `Listing<Post>` | RedditClient.md, GetPostsOptions.md, ListingFetchOptions.md |
| Comments | `reddit.getComments({ postId, commentId?, depth?, limit?, pageSize?, sort? })`, sort one of confidence, top, new, controversial, old, random, qa, live. **The client has no subreddit-wide "new comments" listing.** `getEdited`, `getModQueue`, `getUnmoderated` and `getSpam` are moderator listings | RedditClient.md, GetCommentsOptions.md, CommentSort.md |
| Post model | id, authorId, authorName, subredditName, permalink, title, body (Markdown), url, thumbnail, gallery (`{ url, width, height, status }`, "VALID" when OK), flair and authorFlair (`{ text, templateId, … }`), createdAt, edited, removed, removedByCategory, approved, approvedAtUtc ("0 when unavailable"), spam, locked, nsfw, spoiler, stickied, numberOfComments, crosspostParentId, distinguishedBy | Post.md, CommonFlair.md, GalleryMedia.md |
| Comment model | id, authorId, authorName ("may be [deleted]"), body, createdAt, parentId (`t1_` or `t3_`), postId, removed, spam, edited, distinguishedBy, stickied, replies (`Listing<Comment>`) | Comment.md |
| Triggers | per installed sub. The only cross-sub trigger is `onMentionInCommentCreate` (limited access). **So no delete events come from the watched subs** | global-triggers.mdx, triggers.mdx |
| Playtest | `devvit upload` creates a private playtest sub (made by u/devvit-dev-bot) where you are a moderator, with the app pre-installed, under 200 subscribers. Ctrl+C on `playtest` leaves that version installed; `devvit install <sub>` reverts to the uploaded version | playtest.md, devvit_cli.mdx |
| Logs | `npx devvit logs <sub> [app] --since 2h --verbose`; "5,000 events or up to seven days" | faq.mdx, devvit_cli.mdx |
| Tests | `@devvit/test` is Vitest with in-memory Redis, scheduler and settings; HTTP is blocked by default; the Reddit API is only partly mocked | devvit_test.mdx |
| External endpoints | limited access (a form); managed tokens `devvit_at_…`; 5 requests/s; 10 MB bodies | external-endpoints.mdx (fallback only, R7) |
| "Apps keep working within 2–3 versions" | **not found in the current docs** (FAQ, changelog); treat it as unverified | — |

### 1.2 Project layout

```
reddit-relay/
  devvit.json
  package.json                 devvit CLI, @devvit/web, hono, typescript, vite, vitest, @devvit/test
  tsconfig.json
  vite.config.ts               server-only build -> dist/server/index.cjs (CommonJS)
  README.md                    the app README, with "Fetch Domains" (draft in §1.14.3)
  docs/DESIGN.md               this file
  site/terms.md                Terms (GitHub Pages; draft §1.14.1)
  site/privacy.md              Privacy Policy (draft §1.14.2)
  src/server/index.ts          Hono app on the template's server bootstrap; routes /internal/cron/*,
                               /internal/triggers/*, /internal/settings/validate-*
  src/server/deps.ts           the real Deps: { reddit, redis, settings, fetch, nowMs, log } from @devvit/web/server
  src/server/jobs/shallow.ts   runShallow(deps)
  src/server/jobs/deep.ts      runDeep(deps)
  src/server/jobs/recheck.ts   runRecheck(deps)
  src/server/jobs/heartbeat.ts runHeartbeat(deps)   (also the 14-day age-out, 10 deletes a run)
  src/server/jobs/daily.ts     runDaily(deps)       (trims)
  src/server/triggers.ts       onAppInstall, onAppUpgrade (seed, migrate, send "hello")
  src/server/drain.ts          drainOutbox(deps, deadlineMs): sends queued messages to Discord
  src/core/                    pure TypeScript, no @devvit imports, unit-tested with Vitest
    types.ts                   the v1 contract: RelayPost, RelayComment, RelayPurge, RelayHeartbeat, RelayHello
    config.ts                  parseInstallSettings(raw) -> RelayConfig | { errors }
    sha256.ts                  sha256Hex(s), hmacSha256Hex(key, s): pure JS, so the runtime's crypto doesn't matter
    sellers.ts                 canonicalHandle, handleHash, flairKey, sellerOf, parseSellerAccounts,
                               parseSellerFlair, sellerSetHash
    item.ts                    postToRelay, commentToRelay, authorKind, firstImage, capCodePoints
    passes.ts                  newPosts, commentsDue, changedPosts, sameAuthorIds, flattenComments
    encode.ts                  encodeRelay(payload, key) -> WebhookRequest (inline or multipart)
    discord.ts                 parseWebhookUrl, rateLimitWait, classifyDiscord
    recheck.ts                 RECHECK_STEPS_SEC, nextCheckAt, classifyPost, classifyComment
    heartbeat.ts               buildHeartbeat(stats)
    outbox.ts                  enqueue/claim/ack/retry on a small RedisLike interface
  test/                        *.test.ts (Vitest; @devvit/test for the Redis-backed modules)
  test/fixtures/handle-vectors.json   made-up handles -> 16-hex hashes, written by the perfume repo's
                                      scripts/relay-vectors.mjs, so both sides hash alike
  test/fixtures/contract/*.json       made-up relay messages; the notifiers' tests use copies of these files
```

Every job is `run*(deps)` with `Deps` injected, so tests pass fakes for reddit, fetch and the clock.

### 1.3 devvit.json

```json
{
  "$schema": "https://developers.reddit.com/schema/config-file.v1.json",
  "name": "hyd-deal-relay",
  "server": { "dir": "dist/server", "entry": "index.cjs" },
  "permissions": {
    "http": { "enable": true, "domains": ["discord.com"] },
    "redis": true,
    "reddit": { "enable": true, "scope": "user" }
  },
  "scheduler": {
    "tasks": {
      "relay-shallow":   { "endpoint": "/internal/cron/shallow",   "cron": "* * * * *" },
      "relay-deep":      { "endpoint": "/internal/cron/deep",      "cron": "7 * * * *" },
      "relay-recheck":   { "endpoint": "/internal/cron/recheck",   "cron": "3,13,23,33,43,53 * * * *" },
      "relay-heartbeat": { "endpoint": "/internal/cron/heartbeat", "cron": "*/10 * * * *" },
      "relay-daily":     { "endpoint": "/internal/cron/daily",     "cron": "30 21 * * *" }
    }
  },
  "triggers": {
    "onAppInstall": "/internal/triggers/install",
    "onAppUpgrade": "/internal/triggers/upgrade"
  },
  "settings": {
    "global": {
      "perfumeWebhookUrl": { "type": "string", "label": "Perfume channel webhook URL", "isSecret": true },
      "sneakerWebhookUrl": { "type": "string", "label": "Sneaker channel webhook URL", "isSecret": true },
      "proteinWebhookUrl": { "type": "string", "label": "Protein channel webhook URL", "isSecret": true },
      "testWebhookUrl":    { "type": "string", "label": "Test channel webhook URL", "isSecret": true },
      "relaySigningKey":   { "type": "string", "label": "HMAC key shared with the PCs", "isSecret": true }
    },
    "subreddit": {
      "relayPaused":           { "type": "boolean",   "label": "Pause new relays", "defaultValue": false },
      "notifiers":             { "type": "multiSelect", "label": "Notifiers on",
                                 "options": [{ "label": "perfume", "value": "perfume" }, { "label": "sneaker", "value": "sneaker" },
                                             { "label": "protein", "value": "protein" }, { "label": "test", "value": "test" }],
                                 "defaultValue": ["test"] },
      "perfumeSubs":           { "type": "string",    "label": "Perfume subs", "defaultValue": "DesiFragranceAddicts, IndiaFragMarketplace, PerfumeIndia",
                                 "validationEndpoint": "/internal/settings/validate-subs" },
      "perfumeDeep":           { "type": "string",    "label": "Perfume deep subs (name=posts)", "defaultValue": "IndiaFragMarketplace=400",
                                 "validationEndpoint": "/internal/settings/validate-deep" },
      "perfumeComments":       { "type": "string",    "label": "Perfume comments (name=every Nth pass)", "defaultValue": "",
                                 "validationEndpoint": "/internal/settings/validate-comments" },
      "perfumeKeepDays":       { "type": "number",    "label": "Perfume: re-check deletions for (days)", "defaultValue": 400 },
      "perfumeSellerAccounts": { "type": "paragraph", "label": "Perfume seller accounts (hash=sellerId lines)", "defaultValue": "",
                                 "validationEndpoint": "/internal/settings/validate-seller-accounts" },
      "perfumeSellerFlair":    { "type": "paragraph", "label": "Perfume seller flair (sub|prefix|templates|Name=id;…)", "defaultValue": "",
                                 "validationEndpoint": "/internal/settings/validate-seller-flair" },
      "sneakerSubs":           { "type": "string",    "label": "Sneaker subs", "defaultValue": "SneakerDeals_India, SneakersOfferIndia, SneakerheadsIndia, SneakersIndia",
                                 "validationEndpoint": "/internal/settings/validate-subs" },
      "sneakerComments":       { "type": "string",    "label": "Sneaker comments", "defaultValue": "SneakerDeals_India=1, SneakersOfferIndia=1, SneakerheadsIndia=3, SneakersIndia=3",
                                 "validationEndpoint": "/internal/settings/validate-comments" },
      "sneakerKeepDays":       { "type": "number",    "label": "Sneaker: re-check deletions for (days)", "defaultValue": 400 },
      "proteinSubs":           { "type": "string",    "label": "Protein subs", "defaultValue": "protein_deals",
                                 "validationEndpoint": "/internal/settings/validate-subs" },
      "proteinComments":       { "type": "string",    "label": "Protein comments", "defaultValue": "protein_deals=1",
                                 "validationEndpoint": "/internal/settings/validate-comments" },
      "proteinKeepDays":       { "type": "number",    "label": "Protein: re-check deletions for (days)", "defaultValue": 14 },
      "testSubs":              { "type": "string",    "label": "Test subs (your relay sub)", "defaultValue": "" },
      "testComments":          { "type": "string",    "label": "Test comments", "defaultValue": "" },
      "testFastRecheck":       { "type": "boolean",   "label": "Test: re-check at +2 and +5 min", "defaultValue": false },
      "sameAuthorLinks":       { "type": "select",    "label": "Same-author links",
                                 "options": [{ "label": "off", "value": "off" }, { "label": "same sub", "value": "same-sub" },
                                             { "label": "the notifier's subs", "value": "notifier-subs" }],
                                 "defaultValue": ["same-sub"] }
    }
  },
  "scripts": { "build": "vite build", "dev": "vite build --watch" },
  "dev": { "subreddit": "<relay-sub>" }
}
```

Notes:
- Cron times are UTC (verify). `30 21 * * *` is 03:00 IST.
- That is 5 recurring jobs out of the 10 allowed, leaving room for R12 (a split shallow job).
- discord.com is listed in `domains` even though it needs no review. Webhooks live at
  `https://discord.com/api/webhooks/<id>/<token>`.
- `name`: any free 3–20 character name starting with a letter. It becomes the public app account u/<name>, so leave
  personal names out of it.
- Exact field spellings (`defaultValue` for selects, `options`, and the request/response body of
  `validationEndpoint`) follow the schema at the `$schema` URL (verify on the first `npx devvit upload`: the CLI
  validates the file).

### 1.4 Settings

#### 1.4.1 Global secrets (set from the Mac with `npx devvit settings set <name>`, after the first upload)

| Name | Value | Used for |
|---|---|---|
| `perfumeWebhookUrl`, `sneakerWebhookUrl`, `proteinWebhookUrl`, `testWebhookUrl` | the webhook URL Discord shows for that channel | sending to, editing in and deleting from that channel only |
| `relaySigningKey` | 64 hex characters (`node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`) | the HMAC on every message (§1.9.1); the same value goes into each PC's `relay.signingKey` |

Secrets are global, and that suits this design: there is one installation, and one webhook per notifier.

#### 1.4.2 Install settings (moderator settings on the Install Settings page of r/<relay-sub>)

| Setting | Format | Default |
|---|---|---|
| `relayPaused` | boolean | false. Jobs return at once. The heartbeat still goes out, with `paused: true` |
| `notifiers` | multiSelect | perfume, sneaker, protein |
| `<n>Subs` | names separated by commas, each `^[A-Za-z0-9_]{2,21}$`, at most 6 | as in §1.3 |
| `perfumeDeep` | `Name=N`, N 100–1000 in steps of 100 | `IndiaFragMarketplace=400` (4 requests) |
| `<n>Comments` | `Name=N` with N = 1–60 (every Nth shallow pass; 60 is hourly); a sub left out = no comments | sneaker `…India=1` ×2 and `=3` ×2; protein `protein_deals=1`; perfume none |
| `<n>KeepDays` | 14–400 | perfume 400, sneaker 400, protein 14 |
| `perfumeSellerAccounts` | one `<16 hex>=<seller id>` per line, `#` comments allowed | empty |
| `perfumeSellerFlair` | one `<sub>|<prefix>|<template ids, comma-separated>|<Flair name>=<seller id>;…` per line | empty |
| `testSubs`, `testComments`, `testFastRecheck` | for the playtest (§4.7) | off |
| `sameAuthorLinks` | `off` \| `same-sub` \| `notifier-subs` | `same-sub` (Q3) |

Formats and validation (`src/core/config.ts`, reused by the validation endpoints):
- **Sub names** are checked with the regex. A name may appear under only one notifier; the PCs check that the channel
  matches through `n`.
- **Seller accounts:** each line is `^[0-9a-f]{16}=[a-z0-9][a-z0-9-]{1,47}$`, and one hash may not appear twice. At
  about 35 bytes a line, the 2 KB limit allows about 55 accounts; the registry has 6 today. If it ever outgrows that,
  add `perfumeSellerAccounts2` (Q6).
- **Seller flair:** the same rules as `sellerOfAuthor` in the perfume repo's `src/core/sellers.js`. An empty template
  list accepts no flair, which matches today (the IFM template ids are still a fixture, so this path stays dormant
  until they are filled).
- **Where the lines come from:** a new perfume script, `node scripts/relay-settings.mjs`, prints both values from
  `data/sellers.json` plus your user-added sellers' hashes in `state/`. It also prints each value's size in bytes and
  `sellersHash`, the first 8 hex of sha256 over the sorted lines. The heartbeat carries the app's `sellersHash`, and the
  PC compares it with its own; a mismatch gives a quiet note ("The relay's seller list is out of date: run
  node scripts/relay-settings.mjs and paste it into the app settings").
- Settings are read once at the start of every job run, so a change takes effect within a minute.

### 1.5 Scheduled jobs and budgets

| Job | Cron | Lock (Redis, NX) | Reddit calls per run | Discord calls per run | Time budget |
|---|---|---|---|---|---|
| `shallow` | every minute | `rr:lock:shallow`, 55 s | 1 listing (100 posts) per watched sub, 8 today, in parallel; plus ≤ 4 posts' comment trees per comment-sub when due (≤ 4 requests each) | drains the outbox (≤ 1/s) | stops new reads at 12 s; drains until 25 s |
| `deep` | hourly at :07 | `rr:lock:deep`, 9 min | IFM 400 (4 pages); after app downtime ("catch-up", `rr:meta.lastShallowAt` > 30 min old) also 300 per other sub | drains | 25 s; pages left over wait for the next run |
| `recheck` | :03, :13 … :53 | `rr:lock:recheck`, 9 min | ≤ 40 `getPostById` + ≤ 10 comment trees + ≤ 5 `getCommentById` | Discord deletes for purged items, then drains (purge notices go first) | 25 s |
| `heartbeat` | every 10 min | `rr:lock:hb`, 5 min | 0 | 1 heartbeat per notifier that is on + ≤ 10 age-out deletes | 15 s |
| `daily` | 21:30 UTC (03:00 IST) | `rr:lock:daily`, 10 min | 0 | 0 | trims `rr:seen:*`, `rr:cc:*`, `rr:recheck`, `rr:sent`, stats |

- **Why the budgets:** a Devvit request may run for 30 s (§1.1), and a job is a request. Every loop checks
  `deadlineMs` and leaves its work in Redis for the next run. The dedupe set, the outbox and the re-check queue make
  every step safe to repeat.
- **Locks:** `redis.set(key, runId, { nx: true, expiration })` (verify the SetOptions; if they are missing, use a
  `watch`/`multi`/`exec` transaction). A run that finds the lock taken answers `{ status: "ok" }` at once. Only one job
  drains Discord at a time (`rr:lock:drain`, 30 s); the others only enqueue.
- **Reddit requests a day:** shallow ≈ 8 × 1,440 = 11,520, comment trees ≈ 1,000–3,000, deep ≈ 100, re-checks
  ≈ 3,000–3,500. That is about 16–18k a day, about 12 a minute. Devvit doesn't publish a Reddit API rate limit
  (R1). The lighter profile, if limits show up: shallow every 2 minutes (`*/2`), half the calls.
- **Discord requests a day:** about 150 posts, 300 comments, 432 heartbeats (3 channels × 144), plus purges and
  age-out deletes, about 1,000 in all. That is well under the limits in §1.10.

### 1.6 Reading posts

#### 1.6.1 The shallow pass (`runShallow`)

```
pass = HINCRBY rr:meta pass 1
for each notifier n that is on, for each sub s in n.subs (in parallel per notifier):
  listing = reddit.getNewPosts({ subredditName: s, limit: 100, pageSize: 100 }).all()   // 1 request
  on error: HSET rr:sub:<s> lastErr, lastErrAt; continue                             (the heartbeat reports it)
  HSET rr:sub:<s> lastOkAt now, newestAt max(createdAt)
  seen = ZRANGE rr:seen:<s> (now − 30 d) +inf BYSCORE                                // 1 call; set in memory
  if rr:sub:<s>.seededAt is unset:                                                  // first sight: relay nothing old
      ZADD every id; HSET rr:cc:<s> id → numberOfComments; seededAt = now; continue
  for p in newPosts(listing, seen), oldest first:                                   // dedupe by id, never by time
      if classifyPost(p) != 'ok': ZADD rr:seen (never relayed); continue            // removed, spam, [deleted]
      item = postToRelay(p, { n, now, listing (+ the notifier's other listings for 'notifier-subs'),
                              sellers: n === 'perfume' ? cfg.sellers : null })
      enqueue(n, item)  // one MULTI: SET rr:msg, ZADD rr:out:<n>, HSET+EXPIRE rr:item:p:<id>,
                        //            ZADD rr:seen:<s>, ZADD rr:recheck p:<id> (now + 1 h)
  if commentsDue(pass, n.comments[s]): readComments(n, s, listing)                  // §1.7
drainOutbox(deps, start + 25 s)
```

- **`postToRelay(p, ctx)`** maps a Post to a `RelayPost` (fields in §1.9.3):
  - `late = now − createdAt > 1,800 s`.
  - `same = sameAuthorIds(p, listing, 24 h, 5)`: the ids of other posts in the listing just read, with the same
    `authorId`, created up to 24 h before `p`. This is computed in memory from the listing; **no author id is ever
    stored.** It is off when `sameAuthorLinks` is `off`.
  - `seller = sellerOf(…)` (§1.6.4), perfume only.
- **One listing call reads 100 posts.** For IFM, at about 40–50 posts a day, that is about 2 days. So posts approved
  within about 2 days are relayed within a minute of their approval. The hourly deep pass covers 2–8 days.

#### 1.6.2 The deep pass (`runDeep`)

`getNewPosts({ subredditName: 'IndiaFragMarketplace', limit: 400, pageSize: 100 })` is 4 requests. The new-id logic is
the same as the shallow pass. Anything new here is almost always a held post approved later: it goes out with
`late: true` and `approvedAt` (Post.approvedAtUtc, or `null` when it is 0, which is probably always the case for a
non-moderator app; verify). IFM is "about 300–400 posts" for the 2–8-day approval spread, so 400 covers it. A post
approved after more than about 8 days is missed; today's archive re-check gives up at 9 days + 1 day anyway.

**Catch-up:** when `rr:meta.lastShallowAt` is more than 30 minutes old (the app was down, or Reddit failed), the next
deep pass also reads 300 posts of each other sub. The only cron that stops during downtime is the one that is down,
so the gap closes by itself.

#### 1.6.3 Seeding, upgrades, `late`

- **On install** (`onAppInstall`) or the first pass after a sub is added, that sub is seeded: its ids are marked
  seen and its comment counts stored, with nothing relayed. A `hello` message (§1.12) tells the PC when.
- **On upgrade** (`onAppUpgrade`), Redis is kept, so nothing is relayed twice. A schema migration runs if
  `rr:meta.schema` changed, and then `hello` is sent.
- **`late`** is a hint, not a rule: the PC still judges freshness by `at` (createdUtc) against its own windows (§2.8).

#### 1.6.4 Seller decision (`src/core/sellers.ts`, a port of the perfume repo's `handleHash` and `sellerOfAuthor`)

```ts
export function canonicalHandle(h: string): string | null       // trim, drop one leading "/u/", "u/" or "@" (case-insensitive),
                                                                 // lowercase, then /^[a-z0-9_-]{3,20}$/ or null
export function handleHash(h: string): string | null            // sha256Hex("pdn:" + canonical).slice(0, 16)
export function flairKey(name: string): string                  // lowercase, drop (…) […] {…}, [^\p{L}\p{N}&'’.+-] -> " ", collapse
export function sellerOf(a: { authorName?: string; authorFlairText?: string; authorFlairTemplateId?: string; sub: string },
                         cfg: SellerConfig): { id: string; via: 'account' | 'flair' } | null
// 1. cfg.accounts.get(handleHash(authorName)) -> via 'account'
// 2. cfg.flair.get(sub.toLowerCase()): templateId in templates (a non-empty set) and the flair text matches
//    ^\s*<prefix>(?:\s*/\s*decanter)?\s*:\s*(.+?)\s*$ (case-insensitive) and names.get(flairKey(m[1])) -> via 'flair'
```

- **Test vectors**, generated with the notifier's own `handleHash` on made-up handles:
  - `relay_test_handle`, `u/Relay_Test_Handle` and `@relay_test_handle` all give `1bebdbf264ea315a`.
  - `Relay-Test-2` gives `ac2bdf6637b03fe8`.
  - `ab` gives `null`.
  - `scripts/relay-vectors.mjs` in the perfume repo writes more of them to `test/fixtures/handle-vectors.json`.
- **What leaves the app:** only `{ id, via }`. The username, the hash and the user flair never leave it, and only the
  registry id is sent.
- **Pure-JS SHA-256 and HMAC** (`src/core/sha256.ts`, about 80 lines, tested against `node:crypto`), so the design does
  not depend on which crypto the Devvit runtime exposes (not documented).

#### 1.6.5 Author kind, images, crossposts

- **`authorKind(name, id)`:**
  - `deleted` for `[deleted]` or no authorId.
  - `automod` for AutoModerator.
  - `modteam` for a name matching `/^[a-z0-9_]{2,21}-modteam$/i` (sneaker `observe.js` MOD_TEAM).
  - `bot` for `subscriber-goal`.
  - otherwise `user`.
  - This gives back everything `isSkippableAuthor` and MOD_TEAM read from the name today.
- **`firstImage(p)`:** the first gallery item with status VALID whose URL is https on `i.redd.it` or `preview.redd.it`.
  That is the same host rule as `redditImageUrl` in http.js and `AI_HOSTS` in aiPhase.js. It is `null` otherwise.
  - Crossposts: when `crosspostParentId` is set and there is no image, one `getPostById(parent)` is made and its first
    image used. That covers what the archive's `crosspost_parent_list[0]` gave.
  - Which host Devvit's gallery URLs use (i.redd.it or preview.redd.it) is checked in P4.
- **`hint`:** `image` when the url host is i.redd.it, `gallery` when the gallery isn't empty, `self` for a self post,
  else `link`. Nothing makes decisions with `postHint` (it is only carried), so this best-effort value is safe.

### 1.7 Comments (sneaker, protein)

There is no subreddit-wide comment listing (§1.1), so comments are read post by post. `numberOfComments` from the
listing already in hand says which posts to read:

```
readComments(n, s, listing):
  if rr:sub:<s>.commentsSeededAt is unset: HSET rr:cc:<s> for the whole listing; set it; return    (no backlog)
  counts  = HMGET rr:cc:<s> <ids>                                                                 // 1 call
  changed = changedPosts(listing, counts, cap 4)       // numberOfComments > stored; ties: larger change, then newer post
  for post in changed (≤ 4, parallel ≤ 2):
    tree = flattenComments(reddit.getComments({ postId, sort: 'new', limit: 500, pageSize: 100, depth: 10 }), budget 4 requests)
    relayed = ZRANGE rr:pc:<postId> 0 -1                                                          // ids already sent
    for c in tree, oldest first, not in relayed, created ≥ commentsSeededAt − 300 s:
      if classifyComment(c) != 'ok': continue                                         // [removed], [deleted], spam
      if post not relayed in the last 3 d: enqueue(n, postToRelay(post, { ctx: true }))   // thread context for the PC
      if c.parentId is t1_ and that parent wasn't relayed: enqueue(n, commentToRelay(parent, { ctx: true }))
      enqueue(n, commentToRelay(c, { post }))         // op = c.authorId === post.authorId (both defined)
      ZADD rr:pc:<postId>; bring the post's next re-check forward to ≤ now + 1 h; re-check base = c.createdAt
    if tree.complete: HSET rr:cc:<s> <postId> post.numberOfComments   // incomplete trees stay "changed" for the next pass
```

- **Cadence:** `N=1` means every shallow pass (every minute). `N=3` means every third pass, mirroring sneaker's
  `"third"` (`runIndex % 3 === 0`). Today the sneaker reads `"every"` subs each 2-minute run and `"third"` subs every
  6 minutes. The relay is about twice as fast; set 2 and 6 for the same cadence at fewer calls.
- **Comments on posts older than the newest 100 are not seen.** These subs get about 20–60 posts a day, so that is
  posts older than about 2–5 days. In deal subs, comments after that age are rare, but it is a known limit (R10).
- **Comment ids** may be up to 16 characters (pipeline.js notes 13 since 21 Oct 2026), so the regex is
  `^[a-z0-9]{1,16}$`.
- **`flattenComments`** walks the top-level comments, then `comment.replies` breadth-first, within the request
  budget (each `replies` listing may cost a request; verify how much one `getComments` call returns with `depth`).
  It returns `{ comments, complete }`.
- **Perfume comments:** off. The live perfume path reads no comments. History comments are phase 2 (§2.12).

### 1.8 Redis keys and expiry

Rules: anything holding **Reddit content** expires within **48 h** (only the outbox holds content). **Dedupe and
counters** expire within **30 days**. **No user data** is stored, ever: no author names or ids, no flair. The **only
key that lives past 30 days** is the id-only re-check record, and only for as long as its notifier keeps derived
records (Q1).

| Key | Type | Holds | Expiry |
|---|---|---|---|
| `rr:meta` | hash | schema, installedAt, pass, lastShallowAt, lastDeepAt, appVersion | none (no Reddit data) |
| `rr:lock:<job>` | string | run id | 55 s (shallow) to 10 min (daily) |
| `rr:sub:<sub>` | hash | seededAt, commentsSeededAt, lastOkAt, lastErr (a kind, no text), lastErrAt, newestAt | none (no Reddit content) |
| `rr:seen:<sub>` | zset | post id → createdUtc | members older than 30 d removed daily (`ZREMRANGEBYSCORE`); key `EXPIRE` 31 d after each write |
| `rr:cc:<sub>` | hash | post id → numberOfComments handled | fields of posts older than 8 d dropped daily; key 31 d |
| `rr:pc:<postId>` | zset | comment ids relayed for that post → createdUtc | same as its post's `rr:item` |
| `rr:item:p:<id>` | hash | n, sub, at, relayedAt, msg (Discord message ids, comma-separated, including ctx copies), state (`live`/`purged`), checks, base | `at + keepDays + 1 d` (perfume and sneaker 401 d; protein and test 15 d) |
| `rr:item:c:<id>` | hash | n, post, at, msg | 15 d (the Discord copy is gone after 14 d; its id stays in `rr:pc` for deletion checks) |
| `rr:recheck` | zset | `p:<id>` → next check time | the member is removed after the last check, or on purge |
| `rr:out:<n>` | zset | outbox keys → priority (purges 0, the rest the enqueue ms) | key 3 d |
| `rr:msg:<key>` | string | the encoded message + `{ attempts, nextTryMs }` | **48 h** |
| `rr:dc:<n>` and `rr:dc:pace` | hash | blockedUntil, lastOkAt, lastStatus, failStreak, nextSendMs | none (no Reddit data) |
| `rr:sent` | zset | `<n>:<msgId>` → sentAt | each removed at the 14-day age-out |
| `rr:stats:<yyyymmdd>` | hash | counters (relayed, comments, purged, 429, dropped, errors) | 8 d |

Size is negligible: about 200 B per item record × about 150 posts a day × 400 days ≈ 12 MB, against 5 GB.

### 1.9 The relay message contract (v1)

#### 1.9.1 Envelope and signature

Every message from the app is one Discord message:
- `content` is `<sig>\n<payload>` where `sig` = `hmacSha256Hex(relaySigningKey, payloadText)`, cut to its first 32
  hex.
- **Inline:** `payloadText` is the JSON itself, in `content`.
- **Attachment:** `content` is `<sig>\n<header JSON>`, and `payloadText` is the attached file's exact text. The sig
  covers the file. The PC uses only the file's fields once the sig checks out; the header is just for routing and
  for humans.
- Why sign at all: anyone holding a webhook URL can post *as* that webhook. A forged message with
  `"seller": {"id": …}` and a scam link could ring as a trusted seller, so `webhook_id` alone is no proof. One shared
  key (§1.4.1) and a constant-time compare on the PC.

Every webhook call is `POST …/webhooks/<id>/<token>?wait=true` with `allowed_mentions: { "parse": [] }` (Reddit text
can never ping anyone) and `flags: 4100` (SUPPRESS_EMBEDS 1<<2 + SUPPRESS_NOTIFICATIONS 1<<12: no link previews of
Reddit pages, no buzz on your phone). "Only SUPPRESS_EMBEDS, SUPPRESS_NOTIFICATIONS and IS_COMPONENTS_V2 can be set"
on a webhook. `wait=true` "returns the created message body", which is where the message id comes from.

#### 1.9.2 Inline or attachment: the choice

**Chosen: inline when the payload is ≤ 1,900 UTF-16 code units, otherwise an attached file `<t>-<id>.json`.**

- **The limits:**
  - `content` is at most 2,000 characters. 1,900 plus the 33-character signature line fits whether Discord counts code
    points or UTF-16 units.
  - Embeds would carry at most 4,096 per description and 6,000 per message.
  - Payloads here go up to about 33 KB: an 8,000-code-point body plus about 500 characters of envelope. A sample made-up
    post payload is 534 characters, a comment 257, so most messages are inline.
- **Why an attachment, not split embeds or chunked messages:**
  1. The PC gets byte-for-byte the JSON the app built, at any length. Splitting across embed descriptions caps a message
     at 6,000 characters, and if Discord trims whitespace at field edges (possible; not documented), split text would be
     corrupted.
  2. There is one message per item. A purge is one DELETE, ordering is the channel's order, and the PC holds no
     half-built items between runs.
  3. The cost is small: one extra GET from `cdn.discordapp.com` per long item on the PC, and a multipart body on the
     app side.
- **How the app builds the multipart body:** as a plain string, with CRLF lines and boundary `rr<24 hex>`:
  - a `payload_json` part: `{ content, allowed_mentions, flags, attachments: [{ id: 0, filename }] }`;
  - a `files[0]` part with `Content-Type: application/json; charset=utf-8`.
  - It is sent with `Content-Type: multipart/form-data; boundary=…`. Any fetch that accepts a string body can send this,
    so it doesn't need `FormData` in the Devvit runtime.
  - Discord's request limit is 25 MiB.
- **Fallback if Devvit's fetch mangles multipart** (P3 in §4.7): chunked inline messages
  `{ "v":1, "t":"part", "id":"<item key>", "i":1, "of":3, "s":"<slice>" }`, each signed. The PC reassembles them per
  key, waits up to 10 minutes for missing parts, then drops the item and counts it. The PC code for this exists from
  day one, behind a `parts` branch.

#### 1.9.3 Fields

**Post** (`t: "post"`). Keys are short but readable, and a reader must ignore any key it doesn't know.

| Key | Type / cap | From | Maps to (toItem) | Notes |
|---|---|---|---|---|
| `v` | 1 | — | — | contract version |
| `t` | `"post"` | — | `kind` | |
| `n` | `"perfume"` \| `"sneaker"` \| `"protein"` \| `"test"` | settings | — | the PC drops messages for another notifier |
| `id` | `^[a-z0-9]{1,16}$` | `post.id` minus `t3_` | `id`, `postId` | |
| `sub` | sub name | `post.subredditName` | `subreddit` | protein's toItem has no subreddit; it ignores it |
| `at` | epoch s | `post.createdAt` | `createdUtc` | |
| `seen` | epoch s | the app's clock when read | `retrievedOn` | lag = seen − at |
| `title` | ≤ 300 code points | `post.title` | `title` | Reddit's own maximum is 300 |
| `body` | ≤ 8,000 code points | `post.body ?? ""` | `body` | 2 × `TEXT_CAPS.body` (§0) |
| `cut` | bool | — | `bodyCut` (extra) | true when the body was cut |
| `flair` | ≤ 64 or null | `post.flair?.text` | `flair` | |
| `flairId` | uuid or null | `post.flair?.templateId` | — | spare, for flair classes later |
| `url` | ≤ 2,000 or null | `post.url` | `url` | `&amp;` already undone |
| `hint` | `image` \| `gallery` \| `self` \| `link` \| null | derived | `postHint` (`gallery` → null, as the archive gave) | carried, not decided on |
| `xpost` | id or null | `post.crosspostParentId` | `crosspostParent` | |
| `img` | URL or null | §1.6.5 | `galleryImageUrl` | the image the AI may read |
| `imgs` | int | `post.gallery.length` | — | |
| `ak` | `user` \| `deleted` \| `automod` \| `modteam` \| `bot` | §1.6.5 | `author: ""` + `authorKind` | instead of the handle |
| `seller` | `{ id, via: "account" \| "flair" }` or null | §1.6.4 | `relaySeller` | perfume only; absent for the others |
| `same` | ≤ 5 ids | §1.6.1 | `sameAuthor` | `[]` when off or none |
| `late` | bool | seen − at > 30 min | `late` | |
| `approvedAt` | epoch s or null | `approvedAtUtc` (0 → null) | `approvedAt` | |
| `edited`, `nsfw`, `spoiler`, `locked`, `stickied` | bool | Post | extras | |
| `dist` | `moderator` \| `admin` \| null | `distinguishedBy` | `distinguished` | |
| `ctx` | bool | — | `context` | a copy for comment context: old, never alerted on |

Never sent: `authorName`, `authorId`, `authorFlair` and its template, the handle hash, avatars, karma, score, report
data.

**Comment** (`t: "comment"`): `v`, `t`, `n`, `id`, `post` (post id), `parent` (`"t1_<id>"` or `"t3_<id>"`, exactly
what `parentOf()` in http.js reads), `sub`, `at`, `seen`, `body` (≤ 4,000 code points, 2 × `TEXT_CAPS.comment`), `cut`,
`ak`, `op` (bool, or null when either author is unknown), `dist`, `stickied`, `edited`, `ctx`.

**Purge** (`t: "purge"`): `v`, `t`, `n`, `at`, `ids` (≤ 100 fullnames, `t3_…` and `t1_…`), and `why`, one of
`deleted` | `removed` | `spam` | `author-deleted` | `test`.

**Heartbeat** (`t: "hb"`): `v`, `t`, `n`, `at`, `app` (version), `seq`, `up` (installedAt), `paused`, `off` (§8.10:
the notifier is not on; `paused` is then true too), and:
- `subs`: `{ "<Name>": { "ok": lastOkAt, "err": kind | null, "errAt", "newest": newest createdUtc seen, "items24h" } }`;
- `deep: { "at" }`;
- `recheck: { "due", "purged24h" }`;
- `out: { "queued", "oldestSec", "dropped24h" }`;
- `dc: { "r429_24h", "blockedUntil" }`;
- `sellers` (8 hex, perfume only).

**Hello** (`t: "hello"`): `v`, `t`, `n`, `at`, `app`, `subs`, `comments`, `keepDays`, `seededAt` (per sub). Sent on
install, on upgrade, and when a notifier's settings change.

#### 1.9.4 Examples (made up; key `example-signing-key-not-real`)

Inline post (534 characters of payload):
```
a48d72c48af48dfe…   (32 hex in the real message; 16 shown)
{"v":1,"t":"post","n":"perfume","id":"1zzexmp","sub":"IndiaFragMarketplace","at":1793179800,"seen":1793179848,"title":"[WTS] Example Perfume 100ml BNIB","body":"Sealed bottle, Rs 3,500 shipped. DM if interested.","cut":false,"flair":"Seller Ad","url":"https://www.reddit.com/r/IndiaFragMarketplace/comments/1zzexmp/wts_example_perfume_100ml_bnib/","hint":null,"xpost":null,"img":null,"imgs":0,"ak":"user","seller":{"id":"example-seller","via":"account"},"same":[],"late":false,"dist":null,"nsfw":false,"locked":false,"stickied":false}
```
Comment (protein): `{"v":1,"t":"comment","n":"protein","id":"o9zexmp","post":"1zzexmq","parent":"t3_1zzexmq","sub":"protein_deals","at":1793180100,"seen":1793180140,"body":"Available in Hyderabad too, got it at 2,199","cut":false,"ak":"user","op":false,"dist":null,"ctx":false}`

Purge: `{"v":1,"t":"purge","n":"perfume","at":1793187000,"ids":["t3_1zzexmp"],"why":"deleted"}`

#### 1.9.5 Versioning

- Adding keys keeps `v: 1`, and readers ignore keys they don't know.
- Renaming or removing a key, or changing what it means, makes it `v: 2`.
- A PC that gets an unknown `v` skips the message, counts it, and sends one quiet note a day ("The relay sends
  messages this notifier doesn't understand (v2): update it").
- During a change-over the app can send both versions to a channel for a week (setting `dualVersion`, later).

### 1.10 Sending to Discord (`src/server/drain.ts`)

- **Outbox:** `enqueue` writes `rr:msg:<key>` (48 h) and adds `<key>` to `rr:out:<n>`. The key is `<t>:<id>[:ctx]`, so
  a repeated enqueue is a no-op. `drainOutbox` takes the drain lock and goes round-robin over notifiers, oldest first;
  purges go first (score 0).
- **Pacing:** at most one request a second across all webhooks (`rr:dc:pace.nextSendMs`). A user reported (GitHub
  discord-api-docs #6753, 2024) that webhooks in one server share a bucket, at about 5 requests a second. A common rule
  of thumb is 5 requests per 2 s per webhook (about 30 a minute). We send about 1,000 a day.
- **Headers:** after each response, read `X-RateLimit-Remaining` and `X-RateLimit-Reset-After`. When Remaining is 0,
  wait Reset-After if the budget allows, else stop and leave the rest for the next run.
- **What each answer does** (`classifyDiscord(status)`):

| Answer | Action |
|---|---|
| 2xx (with `wait=true`, the message JSON) | store `msg.id` in `rr:item:*.msg`, `ZADD rr:sent`, delete `rr:msg:<key>`, `ZREM rr:out`; `rr:dc:<n>.lastOkAt` |
| 429 | `retry_after` from the JSON body (else the `Retry-After` header), clamped to 1–600 s. `global: true` or `X-RateLimit-Scope: global` blocks every notifier, otherwise only this one (`rr:dc:<n>.blockedUntil`). Stop draining it. Counted in `dc.r429_24h` |
| 5xx, network error, timeout (fetch's 30-s limit; we use 10 s) | `attempts + 1`, `nextTryMs = now + min(15 min, 30 s × 2^(attempts−1))`. The item stays in the outbox. Its 48-h TTL drops it after that (counted in `dropped24h`) |
| 400 / 413 | a bug in our payload: log the key and status (never the body), drop the item, count `rejected` |
| 401 / 403 / 404 on the webhook | the webhook was deleted, or its URL secret is wrong: `rr:dc:<n>.broken = now`, stop that notifier. Its heartbeats stop too, so the PC raises relay-stalled (§2.7); the logs say why |

- **Duplicates:** a timeout after Discord saved the message causes a resend. The PC dedupes by item id (seen keys), so
  this is harmless.
- **Invalid-request ceiling:** Discord bans an IP after 10,000 invalid requests (401, 403, 429) in 10 minutes, and that
  counts per IP. Devvit's egress IPs are shared with every other Devvit app, so we never retry 401/403 and never
  ignore a 429 (R8).
- **Heartbeats** are not queued: they are sent directly when not blocked, still paced. A missed heartbeat matters
  little; a late one would mislead.

### 1.11 Deletions and removals

No delete events reach the app from other subs (§1.1). Reddit's rule is "remove any user content that has been
deleted from Reddit, including from your Devvit app(s)" and from "any external systems" (Discord and the PCs), and
"any retention of content and data that has been deleted, even if disassociated, de-identified or anonymized, is a
violation". So the app polls.

- **Schedule** (`RECHECK_STEPS_SEC`, measured from `base` = max(post created, newest relayed comment created)):
  - +1 h, +6 h, +24 h, +3 d, +7 d, +14 d, +28 d;
  - then every 30 d while `age < keepDays`;
  - perfume and sneaker: about 19 checks over 400 days; protein: 5 checks over 14 days;
  - `testFastRecheck`: +2 min, +5 min.
- **Load:** steady state about 3,000–3,500 checks a day, about 2.3 a minute. That is under the capacity of 40 per run ×
  144 runs = 5,760 a day.
- **`classifyPost(p)`** (pure, tested on made-up objects):
  - `deleted`: authorName `[deleted]` and (body `[deleted]` or removedByCategory `deleted`).
  - `removed`: `removed`, or removedByCategory in {moderator, automod_filtered, reddit, anti_evil_ops,
    content_takedown, copyright_takedown, community_ops}, or body/title `[removed]`.
  - `spam`: `spam`.
  - `author-deleted`: authorName `[deleted]` with the content still there. Reddit requires removing "all references to
    the author-identifying information": the `seller` id and `same` links count.
  - otherwise `ok`.
  - An error from `getPostById` is never a verdict: retry in 30 min, with no limit, until `keepDays` ends.
- **Comments:** at each check of a post that has relayed comments, walk its tree (§1.7). A relayed comment that comes
  back `[deleted]`, `[removed]` or removed is purged. One missing from an incomplete tree is confirmed with
  `getCommentById` (≤ 5 a run); errors count as "unknown, try later".
- **The purge flow** (`purgeItem`), one transaction per item:
  1. DELETE every Discord message of it (`msg` list, ctx copies included):
     `DELETE …/webhooks/<id>/<token>/messages/<msgId>` gives 204. 404 means the PC already deleted it, which is fine.
  2. Drop any unsent `rr:msg:*` for it.
  3. Enqueue `{ t: "purge", ids: [post fullname + its relayed comment fullnames], why }` at priority 0.
  4. `state = purged`, `ZREM rr:recheck`. The id stays in `rr:seen` so it is never relayed again.
- **Age-out:** each heartbeat run deletes up to 10 of the app's messages older than 14 days (`rr:sent`), so Discord
  never holds relay content past 14 days, even if a PC is gone. Messages the PC already deleted answer 404 quickly.
- **Alerts already on your phone:** Telegram and ntfy alerts may quote a post that is later deleted. The PC deletes
  everything it stores (§2.6). Pulling back delivered alerts (Telegram `deleteMessage` works only within 48 h) is
  left open (Q8).

### 1.12 Heartbeat and hello

- `runHeartbeat` builds `buildHeartbeat(stats)` (fields in §1.9.3) per notifier that is on and sends it directly.
  (As built: every notifier with a webhook gets one, through the outbox, and each is deleted after 3 days, §8.10.)
- The PC treats **30 minutes without a valid heartbeat** as relay-stalled. That is 3 missed heartbeats, so a slow
  scheduler minute doesn't trip it.
- Each heartbeat also tells the PC that the app can read every one of its subs (`subs.*.ok`).
- `hello` (§1.9.3) marks a (re)start, so the PC can log a gap and check that its subs match.

### 1.13 Logs and secrets

- **App logs** (`npx devvit logs`, kept 5,000 events or 7 days) hold only:
  - job names, counts and durations;
  - item ids and status codes;
  - sub names.
- **They never hold** texts, author names, hashes, webhook URLs or the signing key. `log()` passes everything through
  `redact()`, which replaces `/webhooks\/\d+\/[\w-]+/` with `webhooks/<id>/<token>`.
- **Errors:** only the error kind and the HTTP status go into Redis (`lastErr`) and into the heartbeat.

### 1.14 Terms, Privacy and the app README (drafts)

(As built: `docs/terms.md` and `docs/privacy.md` are the current texts, kept in step with the code by
`test/docs.test.js`; the drafts below are history, §8.10.)

Host the two pages on GitHub Pages from a small public repo (for example `reddit-relay-site`, holding only `index.md`,
`terms.md` and `privacy.md`), so the app's code can stay private. Paste both links into the app's details form on
developers.reddit.com. Use the same Privacy link for the Discord bot (Developer Portal → General Information →
Privacy Policy URL).

#### 1.14.1 Terms of Use (draft)

> **<app> — Terms of Use.** Last updated <date>.
> 1. <app> is a private, non-commercial app on Reddit's Developer Platform, made and used by one person, u/<you>
>    ("the developer"), for their own deal notifications. It is installed only in r/<relay-sub>, a private subreddit the
>    developer moderates. It is not offered to anyone else and is not listed in the App Directory.
> 2. On a schedule it reads public posts and comments of r/DesiFragranceAddicts, r/IndiaFragMarketplace and
>    r/PerfumeIndia; r/SneakerDeals_India, r/SneakersOfferIndia, r/SneakerheadsIndia, r/SneakersIndia and
>    r/protein_deals only once those notifiers are switched on. The moderators of all eight subreddits agreed to
>    this use. It forwards them to the developer's private Discord server, which only the
>    developer's own computers read.
> 3. It posts, comments, votes, messages and moderates nothing on Reddit.
> 4. It follows Reddit's Developer Terms, the Devvit Rules and the User Agreement, and Discord's Terms. Reddit data is
>    never sold, licensed, shared or commercialised, and never used to train or improve AI models.
> 5. It comes with no warranty and may stop at any time.
> 6. Contact: modmail to r/<relay-sub>, or u/<you>.

#### 1.14.2 Privacy Policy (draft)

> **<app> — Privacy Policy.** Last updated <date>.
> - **Who:** u/<you> (the developer). Contact: modmail to r/<relay-sub>.
> - **What it reads:** public posts and comments in the subreddits listed in the Terms.
> - **What it forwards, per item:**
>   - the post or comment id, the subreddit and the time;
>   - the title, and the text up to 8,000 characters (4,000 for comments);
>   - the post flair text, the link, the first image link and the crosspost id;
>   - simple flags: edited, locked, NSFW, spoiler, pinned, distinguished;
>   - whether a comment is by the post's author;
>   - the ids of the author's other posts in the same listing from the last 24 hours (to spot reposts);
>   - for the perfume list only, the id of a seller in the developer's public seller registry, when the author's
>     account is one that registry lists. This is decided inside the app by comparing a one-way hash; neither the
>     username nor the hash is forwarded.
> - **Never collected, stored or forwarded:** usernames, account ids, profile links, avatars, user flair, karma, votes,
>   private messages, or anything that isn't public.
> - **Where data goes and for how long:**
>   - Reddit-hosted storage: the send queue, at most 48 hours. Ids and check dates for deletion checks, up to 400 days,
>     with no text.
>   - Discord: the developer's private server. Messages are deleted by the developer's computer after reading
>     (normally within minutes), and by the app after 14 days at the latest.
>   - The developer's computers: prices and ids derived from posts, up to 400 days. Texts are kept only while being
>     processed.
>   - Google Gemini, on a paid plan: images of some sale posts with their titles (usernames, links, phone numbers and
>     e-mail addresses removed). Google does not use paid-plan data to improve its products.
> - **Deletions:** the app re-checks forwarded posts and comments after 1 hour, 6 hours, 1, 3, 7, 14 and 28 days, then
>   monthly while derived records are kept. Deleted or removed content is removed from Discord, and every record
>   derived from it is deleted from the developer's computers. Notifications already shown on the developer's phone
>   may remain.
> - **Sharing:** none. No sale, no profiling, no tracking, no advertising.
> - **Security:** webhook addresses and keys are kept in Reddit's encrypted app secrets and the developer's private
>   config. Every message is signed.
> - **Changes:** dated updates on this page.

#### 1.14.3 App README.md (draft outline; Reddit rejects a vague README)

```md
# <app>

A private, personal relay. It reads new public posts and comments of up to eight deal subreddits (listed below; the
moderators of all eight agreed; the sneaker and protein subs only once those notifiers are switched on) and forwards
them, without any usernames, to the developer's private Discord
server, where their own deal notifiers read them. It is installed only in r/<relay-sub>. It posts and moderates
nothing on Reddit.

## How it works
- Every minute: the newest 100 posts of each listed subreddit; comments post by post where they are switched on.
- Every hour: the last 400 posts of r/IndiaFragMarketplace (posts approved by its moderators later).
- Every 10 minutes: checks earlier posts for deletion (and deletes their Discord copies), and sends a status message.

## Configure
- Secrets (developer, CLI): `npx devvit settings set perfumeWebhookUrl` (and sneaker, protein, test), `relaySigningKey`.
- Install settings (moderator): the subreddit lists, comment cadence, deletion-check days, seller list.

## Deploy
`npm install`, `npx devvit login`, `npx devvit upload`, then fill in the settings. `npx devvit logs r/<relay-sub>` shows
what it does.

## Data and deletions
See the Privacy Policy: <link>. Terms: <link>.

## Fetch Domains

The following domains are requested for this app:

- `discord.com` - Sends each new post or comment of the listed subreddits (JSON, no usernames) to the developer's
  private Discord channel through its webhook; deletes those messages when the Reddit content is deleted or removed,
  and after 14 days at the latest.
```

---

## 2. The perfume reader

### 2.1 Config keys (`config.example.json`, validated in `src/config.js`)

```json
  "relay": {
    "_note": "Reddit through the Discord relay (the reddit-relay app). source: \"archive\" (today), \"shadow\" (read and compare, never alert) or \"relay\" (relay first; the archive too while archiveBackup is true). botToken: the relay-reader bot's token (a secret). channelId: the #perfume channel's id. webhookId: the number after /webhooks/ in the perfume webhook URL (only the id, never the part after it). signingKey: the same 64 hex characters as the app's relaySigningKey (a secret).",
    "source": "archive",
    "archiveBackup": true,
    "botToken": "",
    "channelId": "",
    "webhookId": "",
    "signingKey": "",
    "deleteRead": true
  },
```

- **Validation (problems name the setting, never its value):**
  - `source` is one of the three values.
  - When it isn't `archive`, these are required:
    - `botToken`: `^[A-Za-z0-9_-]{20,}\.[A-Za-z0-9_-]{4,}\.[A-Za-z0-9_-]{20,}$`;
    - `channelId` and `webhookId`: `^\d{17,20}$`;
    - `signingKey`: `^[0-9a-f]{64}$`.
  - If any of them is missing, the relay is off (with a problem line), and the archive is read as if `source` were
    `archive`.
- **Secrets:** `botToken` and `signingKey` join the log redactor's list (`createLogger(… secrets)`), as the Telegram
  token does.
- **`RELAY_RUN`** (in `src/relayStep.js`): `{ pageLimit: 100, maxPages: 3, maxItems: 150 (= RUN.maxItems),
  maxAttachments: 20, timeoutMs: 6000, attachmentMaxBytes: 1 MiB, bulkMaxAgeSec: 13 d, maxPendingDeletes: 1000,
  blockMinSec: 5, blockMaxSec: 600, limitedRunsForFailure: 3 }`.

### 2.2 `src/sources/discordRelay.js` (no dependencies, Node ≥ 20 `fetch`, `node:crypto`)

```js
export const RELAY_SOURCE = 'discord-relay';
const API = 'https://discord.com/api/v10';
const CDN_HOSTS = new Set(['cdn.discordapp.com', 'media.discordapp.net']);

/** GET /channels/{id}/messages?after=&limit= (VIEW_CHANNEL + READ_MESSAGE_HISTORY). Discord answers newest first;
 *  this returns them oldest first. → { ok, messages, rateLimit: { remaining, resetAfterSec }, serverDate } | failure
 *  kind 'rate-limited' (retryAfterSec from the body's retry_after) | 'auth' (401/403) | 'missing' (404) | 'http' |
 *  'bad-body' | 'network'. Never throws. */
export async function fetchMessages({ botToken, channelId, after = null, limit = 100, fetchImpl, timeoutMs, userAgent }) {}

/** One attachment's text: https on a CDN host only, ≤ maxBytes, filename ^(post|comment)-[a-z0-9]{1,16}\.json$. */
export async function fetchAttachmentText(attachment, { fetchImpl, timeoutMs, maxBytes }) {}

/** Bulk delete (POST …/messages/bulk-delete, 2–100 ids, all younger than 13 d), single DELETE for a lone id. It needs
 *  MANAGE_MESSAGES. Ids older than 13 d are left to the app's 14-day age-out. → { deleted, failed } */
export async function deleteMessages(ids, { botToken, channelId, nowMs, fetchImpl, timeoutMs }) {}

/** timingSafeEqual of hmac-sha256(key, text).hex.slice(0, 32) against sig. */
export function verifySig(text, sig, key) {}

/** A Discord message → { type: 'item' | 'purge' | 'hb' | 'hello' | 'part' | 'skip', reason?, payload?, needsFile? }.
 *  It skips a message whose webhook_id isn't ours, a bad sig, v !== 1, n !== 'perfume', or bad JSON. Pure. */
export function parseRelayMessage(msg, { webhookId, key, notifier = 'perfume', fileText = null }) {}

/** A v1 post or comment payload → the item shape of http.js toItem (§2.3), or null when unusable. Pure. */
export function relayToItem(p) {}

export const snowflakeMs = (id) => Number(BigInt(id) >> 22n) + 1420070400000;
```

### 2.3 Relay payload → item (the shape every later stage reads)

| Item key (http.js toItem) | From the relay | Notes |
|---|---|---|
| `kind`, `id`, `postId` | `t`, `id`, (`post` for comments) | |
| `parentKind`, `parentId` | `parent` through the same `parentOf()` | null for posts |
| `author` | `""` | no handle. `handleHash("")` is null, so no hash comes out |
| `createdUtc`, `retrievedOn`, `subreddit` | `at`, `seen`, `sub` | `subreddit` must be one of `config.subreddits`, else the item is skipped |
| `title`, `body`, `flair`, `url`, `postHint`, `crosspostParent` | same names (`hint` → `postHint`, `xpost` → `crosspostParent`) | |
| `galleryImageUrl` | `img` (a gallery's first image, or the crosspost parent's; null for a plain image post, whose `url` is the image the AI reads, as today) | |
| `source` | `'discord-relay'` | |
| `authorFlair`, `authorFlairTemplate`, `removalType`, `secondReadAt` | null | relay items are live reads, and the seller comes from `relaySeller` |
| **extras:** `authorKind`, `relaySeller`, `sameAuthor`, `isOp`, `late`, `approvedAt`, `context`, `bodyCut`, `distinguished` | from `ak`, `seller`, `same`, `op`, `late`, `approvedAt`, `ctx`, `cut`, `dist` | the other stages ignore keys they don't read |

A test checks that `Object.keys(relayToItem(x))` contains every key that `toItem` makes (both shapes stay in step).

### 2.4 `src/relayStep.js` and the runner

```js
/** Phase 1a. deps = { relay: { fetchMessages, fetchAttachmentText }, clockMs, log }. Never throws for a failed fetch.
 *  → { items, purges: string[] (fullnames), heartbeat, hello, toDelete: string[], cursor, tally: { calls, pages,
 *      attachments, foreign, badSig, unknownVersion, skipped }, obs: { relayOk, relayStalled, lastHeartbeatAt, detail } } */
export async function relayStep(deps, state, rc) {}
```

- **Reading:** from `state.relay.lastMessageId`, page oldest first, until a page is short, or `maxPages`, or
  `maxItems` items, or `maxAttachments` files.
  - The cursor stops *before* a message it couldn't finish (an attachment cap, or a fetch error), so nothing is
    skipped.
  - In a dry run nothing is saved or deleted.
- **When relayStep runs in each source mode:**

| `relay.source` | Phase 1a (relayStep) | Phase 1b (redditStep, archive) | Phase 2 items | Alerts from |
|---|---|---|---|---|
| `archive` | not run | as today | archive items | archive |
| `shadow` | every run; purges and heartbeats applied; items → `relayShadow` only (§2.10) | as today | archive items | archive |
| `relay` | every run | only when `archiveBackup` (hourly cadence and slot, as today) | `mergeItems(relayItems, archiveItems)` minus `state.purged`; relay copies first | both, deduped by seen keys |

- **Order in a run:**
  1. relayStep starts together with redditStep's slot wait. It doesn't touch the archive, so it needs no slot, and
     the 6-s timeouts keep it inside the archive's 20-s window.
  2. Purges are applied first (§2.6), so an item purged in the same batch is never processed.
  3. Then `seedItems` or `processItems` run.
  4. Then the state is saved.
  5. Only after the save: `deleteMessages(toDelete ∪ state.relay.pendingDeletes)`. A crash between processing and the
     save re-reads instead of losing items; failed deletes wait in `pendingDeletes`.
- **Seeding:** the first relay read in `relay` mode with `state.relay.seeded !== true` (a fresh PC, or a gap of more
  than 14 days) goes to `seedItems`: marked seen, no alerts. This is the same rule as the archive's first fetch.
  Coming from `shadow`, the PC was already reading, so `seeded` is already true and the cursor just moves on.
- **`state.relay`:** `{ v: 1, source, since, lastMessageId, lastReadAt, lastOkAt, limitedRuns, blockedUntil,
  lastHeartbeat: { at, app, seq, subs, sellers, out, paused }, lastHelloAt, seeded, pendingDeletes: [] }`. It holds
  ids, times and counts only.
- **Sub cursors:** `state.subs[lower].lastPostAt` is also raised from relay items, and from the heartbeat's
  `subs.<Name>.newest`. sub-quiet keeps working whichever source brought the post. The lag numbers go to
  `info.relayLags` (seen − at), next to the archive's `lags`.

### 2.5 identify, pipeline and aiPhase changes

- **`src/core/identify.js`, §A16.8 step 1 (the author):**
  ```js
  const relayed = it.source === 'discord-relay';                       // a test pins it to discordRelay.RELAY_SOURCE
  const skippable = relayed ? (it.authorKind ?? 'user') !== 'user' : isSkippableAuthor(it.author);
  const authorHash = relayed ? null : /* today's line */;
  const authorSeller = relayed ? relaySellerOf(it, compiled.sellers)  // { id, via } when compiled.sellers.byId has the id, else null
                               : sellerOfAuthor({ … }, compiled.sellers);
  ```
- **The individual's identity** (`indHash`, used in `offerKey` as `i:<indHash>`) for relay items is
  `'r' + earliest(it.id, ...it.sameAuthor)`. Base36 ids grow over time, so this is the shorter id, then the
  lexicographic order. Two posts by one author in the listing get the same party, so the 48-h "also on …" gate
  (`POST_LIMITS.offerQuietSec`) still works within a sub (or across the perfume subs with `notifier-subs`, Q3). With
  `same` empty, it is a party of one post.
- **`src/pipeline.js`:** `galleryCandidates` is unchanged. Relay posts already carry `galleryImageUrl` and are skipped
  by its first check.
- **`src/aiPhase.js`:** `ITEM_FIELDS` gains `authorKind`, `relaySeller`, `sameAuthor`, `isOp`, `late`, `context`, so a
  pending AI entry reloaded from state.json reads the same seller and party. For relay items `authorHash` stays null.
- **`context: true` items** (comment context; perfume gets none in v1) are marked seen and give no alerts.

### 2.6 Purge notices: `src/relayPurge.js`

```js
/** ids: 't3_…' / 't1_…' fullnames. Removes every Reddit-derived record of those posts (and their comments) and
 *  comments, and remembers the ids. → { counts: { history, priceLog, recheck, ai, offers, channels, outbox, warnings } } */
export function applyRelayPurges(state, ids, { history, now }) {}
```

For a post id `P`, and a comment id `C`:

| Where | What goes |
|---|---|
| The run's history handle (§C17) and `state/price-log.json` | observations with `pid === P`, or `id` starting `c:C:` (observe.js ids are `p:<post>:<m>:<ml>:<slot>`, `c:<comment>:…`; every record carries `pid`). Saved through priceLog's atomic write |
| `data/price-history.json` (shipped from the Mac; not edited on the PC) | filtered when loaded: records whose `pid` or comment id is in `state.purged` are left out |
| `state.recheck[P]`, `state.ai.pending` entries of P or C, `state.offers.recent` and `state.channels` slots keyed `post:P` / `comment:C`, `state.sellerWarnings` with `postId === P`, `state.inProgress` | deleted |
| `state.outbox` | unsent alerts about P or C are dropped (alerts gain a `ref: 'post:P'` field so they can be matched) |
| `state.seen` | **kept** (ids only, 7 d), so an archive copy of P is never read again |
| `state.purged` | `{ [fullname]: purgedAt }`. Ids only; kept 400 d (= `PRICE_LOG.keepSec`) |
| Logs | kept (ids only, 7 days, `src/log.js`) |

`state.purged` also filters archive items in `redditStep`, the history sweep's pages and `--check-post`, because the
archive may still serve a post that Reddit no longer has.

### 2.7 Health: `relay-down`, `relay-stalled`

New in `src/health.js`:
- `HEALTH.relayDownSec = 30 * MIN`, `HEALTH.relayStalledSec = 30 * MIN`.
- `RELAY_KINDS = ['relay-down', 'relay-stalled']`.
- The archive family becomes the **Reddit family**: ARCHIVE_KINDS + RELAY_KINDS + `sub-quiet:*`. The shared fields
  are kept: `archiveLoudAt` is now the family's last loud warning, `archiveTroubleAt` its trouble time.

Observations (from `relayStep`, into `updateHealth(health, obs, now)`):
- **`obs.relayOk`:** true when a GET answered 2xx; false on network, 5xx, `auth` or `missing`; null when it wasn't
  tried (blocked after a 429). Three limited runs in a row count as false, as `arcticLimitedRuns` does.
- **`obs.relayStalled`** (only judged when `relayOk` is true): true when there has been no valid heartbeat for
  `relayStalledSec`, counted from `state.relay.since` if there was never one; also true when the newest heartbeat says
  every sub's `ok` is older than 30 min ("the app runs but can't read Reddit"); false otherwise.
- **`obs.relayMode`:** `shadow` or `relay`, and whether the archive covers (in relay mode with `archiveBackup`,
  `arcticOk` true and not stalled).

Rules (all reuse `step()`, `loudLater()`, the 08:00 IST rule, `/pause`, reminders every 24 h, and "back to normal"
after 30 healthy minutes):

| Mode | relay-down / relay-stalled | archive-down / archive-stalled |
|---|---|---|
| `archive` | not observed (records forgotten after 7 d) | as today |
| `shadow` | **quiet only**: note, reminders, recovery. Never loud, and never counted for the family's one loud a day. Text: "Reddit relay (shadow) …: alerts still come from the archive" | as today |
| `relay`, the archive covering | a quiet note: "Reddit relay down since 10:42 IST; the archive still covers Reddit (deals still arrive). Loud warning only if both are down for a day." `loud(r).from = Infinity` while the archive covers, the same mechanism sub-quiet uses while an archive problem is open | quiet note only while the relay is healthy (`from = Infinity`): "the relay covers Reddit" |
| `relay`, nothing covering (the archive down, ended, or `archiveBackup: false`) | the full family rule: a note first, one loud warning once the trouble (counted from when *both* failed) has been observed for 24 h, at most one loud a day across the family. Loud text: "Reddit reads off for a day: relay down" / "…: relay stalled" | when `archiveBackup` is false: not observed |

- **Texts:**
  - relay-down: "Discord refused or failed the relay read since {since} ({kind}). {coverage}". An `auth` hint adds:
    "Check relay.botToken and the bot's channel permissions (README → Reddit relay)".
  - relay-stalled: "The relay has sent nothing since {lastHeartbeatAt}: the Reddit app may be stopped or blocked.
    On the Mac: npx devvit logs r/<relay-sub>."
- **The `gapSec` rule (2 h)** and the restart-after-gap handling apply unchanged. The relay is read every run, so a
  gap means the PC itself was off.
- **sub-quiet:** its texts name the source that is primary: "The relay may be missing this sub" in relay mode.

### 2.8 Re-checks (`src/recheck.js`)

Today a held IFM post is read live as title + flair, queued (`recheckCandidate`), and re-read from the archive at +3 d
and +9 d. With the relay:

1. **Relay items are never held.** The app sees a post only once it has been approved, so `bodyHeld` is false and
   relay items never create queue entries. Archive items in `relay` mode with backup, and in `shadow` mode, still do,
   exactly as today.
2. **A relay copy settles a queued entry.** In phase 2, before `processItems`, a relay post whose id is in
   `state.recheck` (seen earlier from the archive as held) is passed to `applyRecheckResults(state, [id], [item], ctx)`
   instead of being skipped as seen. It takes the existing "readable" path: identify with `recheck: true`, `rd: 1` history records, at
   most one quiet "seller post now readable", never a ring. This replaces the archive's +3 d / +9 d id lookups once
   the archive is gone.
3. **A late relay post that was never seen** (once the archive is gone, every approved IFM post):
   - within the freshness windows (`POST_LIMITS.saleFreshSec` 13 h, `dealFreshSec` 6 h, judged with
     `firstSeen = now`): the normal path, so it can ring;
   - older: the re-check path through a new export, `applyLateRelayItem(state, item, ctx)` (the body of today's
     `readable()`, without needing a queue entry). That keeps today's noise level: history, plus at most one quiet
     note.
4. When the archive is off, `dueRecheckIds` is no longer fetched. Leftover entries age out through `pruneRecheck`
   (+9 d + 1 d grace), and `importShippedRechecks` still queues the Mac build's list, which the relay copies then
   settle.

### 2.9 Retention

| Data | Where | Kept | Change |
|---|---|---|---|
| Relay texts | memory; `state.ai.pending` (6 h); `state.outbox` until sent; `state.inProgress` | as today | no new text on disk |
| History records (ids, perfume, size, condition, price; no text, no usernames: `PRIVATE_KEYS`) | `state/price-log.json`, `data/price-history.json` | 400 d | purge notices delete them; `state.purged` filters the shipped file |
| `state.purged`, `state.relay`, shadow counters | `state/` | 400 d / always / 30 d | new, ids and counts only |
| Discord copies | Discord | minutes (deleted after reading); ≤ 14 d (the app's age-out) | new |
| The app's re-check horizon | Redis | `perfumeKeepDays` 400 | keeps the 400-day history deletable |

If you pick the strict 30-day horizon (Q1): set `perfumeKeepDays` to 30 in the app and `relay.redditHistoryDays: 30`
on the PC. priceLog then drops Reddit records (`k` `p` or `c`) older than 30 d, while site and store spells keep
400 d.

### 2.10 Shadow mode: `src/relayShadow.js` and `--relay-report`

```js
/** Records what both sources said, by id, without text: { a?: Seen, r?: Seen } with
 *  Seen = { seenAt, at, titleH, bodyH, flair, img: bool, sellerId, held: bool, offers: [{ m, ml, cd, p, sl }] },
 *  where titleH and bodyH are sha256(normalizeText(...)).slice(0, 8) and offers come from identifyItem (pure; no
 *  decisions, no state writes). Pairs wait up to 26 h (the archive reads each sub hourly; held posts take days, so
 *  they are kept 10 d). Daily counts go to state.relayShadow.days[ISTdate][sub]. */
export function recordShadow(state, { archiveItems, relayItems, compiled, now }) {}
/** --relay-report [--days 7]: the numbers of §4.8, per sub and per day, with the ids of mismatches (ids only). */
export function shadowReport(state, { days = 7, now }) {}
```

Shadow mode also checks the seller: for archive items, `sellerOfAuthor(handleHash(author))` → id, compared with the
relay's `seller.id` for the same post. They must agree 100% of the time. A mismatch means the app's
`perfumeSellerAccounts` is out of date.

### 2.11 Commands, status, docs

- **`node src/run.mjs --check-relay`** reads the channel once, without deleting or saving, and prints:
  - whether the bot token and channel work, and whether `content` is empty (the Message Content intent is off);
  - messages by type, and whether every signature checks out;
  - the last heartbeat (time, app version, per-sub ok times, queued, sellers hash match), and the last purge.
- **`--status` and `/status`** gain one line:
  `Reddit: relay (archive backup) · heartbeat 12:04 IST · app 0.1.3 · 3 subs ok · queue 0 · last purge 27 Oct`.
- **README:**
  - a new section "Reddit relay": what it is, the setup of §4.1 and §4.4, `--check-relay`, and the source modes;
  - "Known limits": the IFM paragraph becomes "with the relay, IFM posts arrive when approved"; the archive line gets
    the relay's status;
  - "Privacy": the relay flows of §1.14.2.
- **docs/WINDOWS.md:** a step "3b. The Reddit relay" (config keys, then `node src\run.mjs --check-relay`).

### 2.12 After the archive closes (phase 2, not v1)

- **The live history sweep** (`historySweep.js`) pages the archive. Its replacement:
  - The app relays IFM and PerfumeIndia comments hourly (`perfumeComments: "IndiaFragMarketplace=60, PerfumeIndia=60"`).
  - The PC calls `observe(item, { post: threadFacts, parent, isOp: item.isOp })` on relay comments directly, using the
    relay's `op` instead of the sweep's `opHash`.
- **`--check-post <id>`** and the AskReddit freshness probe need the archive. After it closes, `--check-post` accepts a
  saved relay payload (`--check-post-file x.json`), and the heartbeat takes the probe's place.
- **The Mac's history build** reads the archive until it ends; after that, history grows only through the PC's price
  log.
- **A request channel** where the PC asks the app for specific post ids (the app would read it with a bot token
  secret) is possible later; not v1.

### 2.13 Sneaker and protein (later, the same pattern, built as siblings)

Until a notifier's PC reader exists, leave its webhook unset and the notifier off in the app (§8.10): nothing would
read or delete what the app sends to its channel. Each gets its own `discordRelay.js` + `relayStep.js` (copied, with `notifier` set to `sneaker` / `protein`) and the
same config block. They also need these item-level changes:

| Where | Today (reads `author`) | With relay items |
|---|---|---|
| sneaker `core/decide.js` ≈ line 806, the repost check | `x.author === item.author` | `sameAuthor(x, item)` = names equal when both are present, else `item.sameAuthor.includes(x.id)`; recentPosts store `id`, not the name |
| sneaker `decide.js:1296`, protein `decide.js:223`, skippable | `isSkippableAuthor(item.author)` | `item.authorKind !== 'user'` for relay items |
| sneaker `decide.js:1334`, protein `decide.js:330`, isOp | `item.author === post.author` | `item.isOp === true` |
| sneaker `observe.js:1964`, MOD_TEAM | the name regex | `item.authorKind === 'modteam'` |
| protein `decide.js:167`, `findDuplicatePost` | same author + title within 15 min | `sameAuthor(...)` as above |
| sneaker `format.js:156`, protein `format.js:73` | `u/<author>` in the alert | the line drops the user for relay items (sub and time stay) |
| protein `runner.js` missing posts and parents (archive id lookups) | `fetchPostsByIds`, `fetchCommentsByIds` | the relay's `ctx: true` copies arrive first; no lookups |
| sneaker comments cadence `"every"` / `"third"` | the PC's run index | the app's `sneakerComments` N (§1.7) |

---

## 3. Paid Gemini for Reddit images (perfume)

### 3.1 Why

Once Reddit data comes from a Devvit app, the Devvit Rules apply to it. An app must use an approved LLM (Google
Gemini or OpenAI ChatGPT), and must not "allow any third-party access to create, improve, modify, train or fine-tune
any Generative AI, LLM, ML, or NLP models using Reddit Data".

Gemini's free tier fails that:
- "Google uses the content you submit to the Services and any generated responses to provide, improve, and develop
  Google products";
- "human reviewers may read, annotate, and process your API input and output";
- the pricing page lists "Used to improve our products: Yes" for free and "No" for paid.

The API is a Paid Service "only when accessing the API through a Cloud Project associated with an active billing
account". Then "Google doesn't use your prompts (including associated system instructions, cached content, and files
such as images…) or responses to improve our products", and keeps them "for a limited period of time, solely for
detecting and preventing violations of the Prohibited Use Policy".

### 3.2 Cost (prices from the Gemini pricing page, "Last updated 2026-10-27 UTC")

**Rates:**
- Gemini 3.5 Flash-Lite (`gemini-3.5-flash-lite`), paid: **$0.30 per 1M input tokens** (text, image, video, audio),
  **$2.50 per 1M output tokens** (thinking included).
- Batch is half price; it is not used here, because the reads are live.
- Image tokens: 1,120 per image at Gemini 3's default resolution (media-resolution docs).

**One call, measured from `buildGeminiRequest`:** the request without its image is 3,420 characters (system prompt
1,392, JSON schema 1,606, title and focus). That is about 850 tokens. With the image, about 2,000 input tokens
(≈ $0.0006). The output is usually 100–600 tokens of JSON (≈ $0.00025–0.0015); "most answers are 'not a perfume
listing'". Thinking is `minimal` for Flash-Lite (gemini.js `defaultThinkingLevel`).

| Case | Per image | At the cap | In rupees (₹95.3/$, 27 Oct 2026) |
|---|---|---|---|
| typical | $0.0008–0.0021 | — | ₹0.08–0.20 |
| PC, `ai.maxPerDay` 60 | — | ≤ $0.13/day, ≤ $3.8/month | ≤ ₹12/day, ≤ ₹360/month |
| Mac nightly build, at most 150 images | — | ≤ $0.32 a night | ≤ ₹30 a night |
| ceiling (every answer at `MAX_OUTPUT_TOKENS` 8,192) | $0.021 | $1.27/day PC, $3.2 a night Mac | ₹120/day, ₹300 a night |

Add 18% GST on an Indian billing account (verify on the first invoice). A billed project pays from the first call, as
far as the pricing page shows. The notifier's own caps (`maxPerDay`, `maxPerRun`) bound the spend.

### 3.3 README.md: replace "### 4. Free AI for image price lists (optional)" (lines 224–239)

> ### 4. AI for image price lists (optional, paid Gemini)
> Some sellers post their prices only as an image (a price list, a store screenshot). With a key, Gemini 3.5
> Flash-Lite reads such an image when the post has no price in its text: each item it finds goes through the same
> rules, labelled "from image". Only list prices and public codes can ring; anything unclear goes quiet ("from image,
> offer not clear"). Without a key everything else works the same.
> 1. Ideally on a **secondary Google account**, create a **new Google Cloud project used only for this notifier** (not
>    the sneaker notifier's). **Turn billing on** for it: console.cloud.google.com → Billing → link a billing account.
>    Then create the key at https://aistudio.google.com/apikey **in that project**.
> 2. Restrict the key to the **Generative Language API** (APIs & Services → Credentials → your key → API
>    restrictions).
> 3. Billing → **Budgets & alerts**: add a budget of ₹500 a month for this project, with e-mail alerts at 50%, 90% and
>    100%.
> 4. In `config.local.json`, set `"paidTier": true` in `ai` (the notifier refuses to send images without it).
>
> **Why billing:** Reddit posts reach this notifier through a Reddit Developer Platform app, and Reddit's rules for
> those apps forbid letting anyone use Reddit data to improve AI models. On the free tier Google uses what you send to
> improve its products, and its staff may read it. With billing on, the project is a paid service: Google does not use
> the images, titles or answers to improve its products, and keeps them for a limited time only to detect abuse.
>
> **Cost:** about ₹0.1–0.2 an image (Gemini 3.5 Flash-Lite, $0.30 per million input and $2.50 per million output
> tokens, 27 Oct 2026). At this notifier's limit of 60 images a day that is at most about ₹360 a month plus GST, and
> most days read fewer. The Mac's overnight history build reads at most 150 images, about ₹30 a night.
>
> **In plain words:** strangers' post images and titles go to Google, with links, u/ and r/ names, phone numbers and
> e-mail addresses removed; never the post text or the author. They go on the paid tier, which Google doesn't use to
> improve its products. The AI stays off unless you add the key; empty `geminiKey` to turn it off. Limits: 60 images a
> day and 2 per run, counted per Pacific day (it starts at 12:30 IST, 13:30 IST from 2 Nov to 14 Mar).

In the same README, the §5 example line gains `"paidTier": true`, and "Troubleshooting → AI screenshot reading is off"
gains: "…or the key's project has no billing (a free-tier quota answered): turn billing on (step 4)".

### 3.4 config.example.json, `ai`

```json
  "ai": {
    "_note": "Optional: a Gemini API key from a NEW Google Cloud project with billing ON, then set paidTier to true. Reddit's Developer Platform rules need the paid tier (Google doesn't use paid-tier requests to improve its products). About ₹0.1–0.2 an image; maxPerDay caps the cost (60 ≈ at most ₹360 a month). Leave geminiKey empty to keep it off. maxPerDay 1-200, maxPerRun 1-4.",
    "provider": "gemini",
    "geminiKey": "",
    "paidTier": false,
    "model": "gemini-3.5-flash-lite",
    "maxPerDay": 60,
    "maxPerRun": 2,
    "nightReserveFrac": 0
  },
```

### 3.5 docs/WINDOWS.md

- Step 3, "Do README setup steps 1–6 (topics, new bot, new healthchecks check, optional AI key, …)", becomes "…, optional
  **paid** AI key (README step 4: a new Google Cloud project with billing on, then `"paidTier": true`), …".
- A row is added to its update notes: "Updating from a free AI key: create the paid project and key (README step 4),
  replace `geminiKey`, set `paidTier`".
- The new step 3b (the relay) is described in §2.11.

### 3.6 Code changes

The API, endpoint, header, quota handling and caps are the same on the paid tier, so **none of that has to change**.
Recommended additions:

1. **`ai.paidTier`** in `src/config.js` DEFAULTS: `false`, a boolean. With a key but `paidTier !== true`, `ai.active`
   is false, with the problem line "ai.paidTier is not true: turn billing on for the key's project (README step 4),
   then set it". This stops an old free-tier key from carrying on unnoticed. It applies to all Reddit images, since
   archive images were the same question all along.
2. **Detecting a free-tier key at runtime:** in `classifyGeminiError` (gemini.js), a 429 whose `QuotaFailure.violations[].quotaId`
   contains `FreeTier` becomes kind `'free-tier'`. `recordAiError` then sets `ai.off = { kind: 'free-tier' }`, and
   health's `ai-off` note reads "the key's project has no billing". Verify the quotaId text in a test call; if it
   differs, keep only item 1.
3. The comment on line 1 of `src/vision/gemini.js`, "Gemini free tier reads one post image", becomes "Gemini (paid
   tier)". Tests: the paidTier guard, the free-tier 429 fixture, and the README and example text (as v14's doc tests
   do).
4. **`store: false`** in every request body (added after the third review): a top-level key beside
   `system_instruction`, `contents` and `generationConfig`, not inside `generationConfig`. Since 26 Oct 2026 new
   paid-tier projects store Generate Content API requests by default (`store=true`; AI Studio's Logs and datasets, 55
   days unless set to 7, 14 or 28), and a project made for this notifier now would be one of them; older projects
   keep their setting (off unless it was turned on). The field "takes precedence over the project-level logging
   config", so `false` keeps the image, the title and the answer out of those logs whatever the project says.
   Turning off Generate Content API logging in the Logs and Datasets dashboard's Settings panel does the same for the
   whole project. Google's separate abuse-monitoring log (55 days) has no switch. Test: the request body carries
   `store: false`. The Privacy Policy's AI line relies on it.

The sneaker notifier (`maxPerDay` 150; its README and config also say "never enable billing") needs the same change
once it reads the relay: up to about ₹900 a month at its cap.

---

## 4. Setup and tests

### 4.1 Discord (about 15 minutes, any device)

1. Create a server ("Deal relay"), private, with no invite links. Create text channels `#perfume`, `#sneaker`,
   `#protein`, `#relay-test`.
2. For `#perfume` and `#relay-test` only, for now (`#sneaker` and `#protein` once their PC readers exist, §2.13): ⚙ →
   Integrations → Webhooks → New Webhook. Name it after the channel and **Copy Webhook URL**. That URL is a secret;
   paste it only into the `npx devvit settings set` prompt.
3. Turn on **User Settings → Advanced → Developer Mode**, then right-click each channel → **Copy Channel ID** (for the
   PCs, and `#relay-test`'s for the playtest).
4. Create the bot at https://discord.com/developers/applications:
   - New Application "relay-reader".
   - **Bot** → **Reset Token** and copy it (a secret; it goes to the PCs, and for the playtest into the Mac's perfume
     `config.local.json`).
   - **Privileged Gateway Intents → Message Content Intent: on.** Without it, `content` and `attachments` come back
     empty over HTTP too, apart from the app's own messages, DMs and mentions (gateway docs). Small apps just toggle
     it.
   - **Public Bot: off**, so only you can add it. If the portal refuses ("Private application cannot have a default
     authorization link"), set **Installation → Install Link** to None first. Keep **Requires OAuth2 Code Grant** off,
     or the plain authorize link below won't add the bot.
   - **General Information → Privacy Policy URL**: the page from §4.2, once it exists (it comes after the upload).
5. Add the bot with the authorize URL
   `https://discord.com/oauth2/authorize?client_id=<application id>&scope=bot&permissions=74752`. 74752 is View
   Channels 1024 + Manage Messages 8192 + Read Message History 65536, so the bot can read and delete only; it can't
   send. Pick the relay server.
6. Keep @everyone without access at **server** level: Server Settings → Roles → @everyone → View Channels off, with no
   channel overwrites. The bot's own role then gives it the three permissions in every channel, `#relay-test`
   included (P3 and P16 read it). Don't deny @everyone in a channel's own Permissions instead (a private channel does
   that): Discord applies that overwrite to the bot too, and a denied View Channel denies the rest, so its reads fail
   with 403 Missing Access. A channel that has such a deny needs the bot (or its role) added to it with View Channel
   allowed. Without Read Message History a read returns no messages, not an error. Only you and the bot are in the
   server.

### 4.2 GitHub Pages (about 10 minutes, after the upload: the pages name the relay sub it creates)

Make a public repo `reddit-relay-site` with copies of `docs/terms.md` and `docs/privacy.md` (the current texts, §1.14),
with your username and the relay sub filled in (YOUR_USERNAME, YOUR_RELAY_SUB; the app name too if it changed). Then
Settings → Pages → Deploy from a branch: `main`, `/ (root)`. The two URLs go into the README's links
(YOUR_GITHUB_USER), the app's details form and the bot's Privacy Policy URL.

### 4.3 Devvit, on the Mac

In this order (the README's "Setup on the Mac" has every step): Discord (§4.1), the build and the upload, the
`dev.subreddit` check, the private-sub check, the pages (§4.2), the secrets, the Install Settings, the playtest.

```sh
cd ~/Documents/Code/reddit-relay
npm install                                  # devvit CLI, @devvit/web, …
npm test                                     # node --test (as built; the README's "Tests")
npm run typecheck
npm run build
npx devvit login                             # browser sign-in; the token is kept in ~/.devvit/token
npx devvit whoami
npx devvit upload                            # private app + private playtest sub r/<relay-sub> (you're its moderator), installed there
# the first upload writes "dev": { "subreddit": "<relay-sub>" } into devvit.json itself: check it; add it only if it warned
# check that r/<relay-sub> is private (Mod Tools → General Settings → Privacy & Discovery), then the pages (§4.2)
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"   # the signing key (§1.4.1), also for the PCs
npx devvit settings set relaySigningKey      # paste at the prompt
npx devvit settings set testWebhookUrl
npx devvit settings set perfumeWebhookUrl    # sneaker and protein: once their readers exist
npx devvit settings list                     # names only: not which ones have values (P2 reads the logs instead)
npx devvit logs r/<relay-sub> --since 15m --verbose
```

Then:
1. Open the app's Install Settings for r/<relay-sub>: `https://developers.reddit.com/r/<relay-sub>/apps/<app>` (also
   developers.reddit.com → My communities → r/<relay-sub> → the app). Keep `notifiers` at `test` only, set `testSubs`
   to `<relay-sub>`, `testComments` to `<relay-sub>=1` and `testFastRecheck` to on. Each field is validated before it
   is saved; the CLI sets only the global secrets.
2. Before P3, on the Mac: the perfume notifier's
   `node src/run.mjs --check-relay test <#relay-test channel id> <its webhook id>` needs `relay.botToken` and
   `relay.signingKey` in the Mac's perfume `config.local.json` (`source` can stay `"archive"`). It saves and deletes
   nothing, so the playtest needs no Windows install.
3. Run the playtest (§4.7); P4, P7 and P20 need perfume on, so they come after step 4.
4. Then, in this order (§8.10): on the Windows PC, the bot and the `relay` block with `"source": "shadow"` (§4.4
   step 1); on the Windows PC, `node scripts\relay-settings.mjs` in the perfume folder (it reads the sellers added
   with /trust from `state\`, which only the PC has), pasting `perfumeSellerAccounts` and `perfumeSellerFlair` as
   printed; then switch `notifiers` to perfume only (test off) and `testFastRecheck` off; then `--check-relay` on the
   PC (§4.4 step 2). Sneaker and protein come on, with their webhooks, only once their PC readers exist (§2.13).
5. To update the app later: `npx devvit update app` (keeps @devvit packages in step with the CLI), `npm test`,
   `npx devvit upload`, `npx devvit install r/<relay-sub>`. Roll back with `npx devvit install r/<relay-sub> <app>@<version>`.

### 4.4 The PCs (each notifier)

1. Add the `relay` block to `config.local.json`: `source: "shadow"`, `botToken`, `channelId`, `webhookId` (the number in
   that channel's webhook URL), and `signingKey`. Then run `node scripts\relay-settings.mjs` and paste its two seller
   settings into the app, and switch the notifier on in the app (§4.3 step 4).
2. Run `node src\run.mjs --check-relay`. It should show signatures OK, a heartbeat within 10 minutes, and the subs ok.
   (Before the notifier is on in the app, its heartbeat says paused and off, and every sub reads "never".)
3. The task picks it up at the next run.
4. After the switch criteria (§4.8) hold for 14 days, set `source: "relay"`.
5. When the archive closes, set `archiveBackup: false`.

### 4.5 Unit tests: the app (Vitest; `@devvit/test` for Redis)

| Module | Cases (made-up data only) |
|---|---|
| `sha256.ts` | NIST vectors; hmac against `node:crypto` on random strings (emoji, Devanagari) |
| `sellers.ts` | `handle-vectors.json`; `/u/`, `u/`, `@` and case; too short or long → null; flair with and without a template; `Seller/Decanter : Name`; brackets dropped; empty templates accept nothing; `parseSellerAccounts` errors and duplicates; the 2 KB warning; `sellerSetHash` equals the perfume script's |
| `item.ts` | post and comment mapping; caps at 300 / 8,000 / 4,000 code points with an emoji on the edge; `cut`; author kinds; image host and status filter; crosspost image; `hint`; never an author name or id in the output (a key scan) |
| `passes.ts` | `newPosts` by id when an old id reappears (approval); never by time; `sameAuthorIds` window and cap, off; `commentsDue` N=1/3; `changedPosts` order and cap; `flattenComments` budget and `complete` |
| `encode.ts` | ≤ 1,900 → inline; 1,901 → multipart; the multipart string parses back (`new Response(body, { headers }).formData()`); sig over the exact text; `allowed_mentions` empty; `flags` 4100 |
| `discord.ts` | 429 body and headers, global and scope; Remaining 0 + Reset-After; classifying 2xx/4xx/5xx; webhook URL parsing |
| `recheck.ts` | step times; the 30-day tail up to keepDays; fast mode; `classifyPost` for every category; an error is no verdict |
| `outbox.ts` (Redis) | enqueue is idempotent; order; purges first; backoff; 48-h drop; drain lock; pacing |
| jobs (fakes) | first pass seeds; the second relays only new ids; deep finds an approved old id once; comments: changed posts only, context copies first, `op`; recheck purge flow (Discord delete 204 and 404, notice queued, seen kept); heartbeat contents; a run past its deadline leaves work for the next |
| contract | every file in `test/fixtures/contract/` encodes and decodes to the same payload; the same files sit in the notifiers' tests |

### 4.6 Unit tests: perfume (node:test, made-up posts only)

- `discordRelay`:
  - parse inline, attachment (fake fetch), `part`; a foreign `webhook_id`; a bad sig; `v: 2`; the wrong `n`; bad JSON;
  - `relayToItem` keys ⊇ `toItem` keys;
  - `snowflakeMs`;
  - bulk-delete age split.
- `relayStep`:
  - cursor paging, oldest first; the caps (items, attachments), with the cursor stopping before an unfinished message;
  - deletes only after save, and pendingDeletes kept on failure;
  - a 429 blocks; 401 gives `relayOk` false with kind `auth`;
  - seeding in relay mode, none in shadow;
  - a dry run saves nothing.
- identify and offers: `relaySeller` → `author.seller` (an unknown id → null); `authorKind` skippable; no
  `authorHash`; `indHash` `r<earliest>`; two posts with `same` share an offer key and the 48-h gate holds.
- `relayPurge`:
  - removes price-log records by `pid` and `c:<id>:`; filters the shipped file; recheck, ai.pending, offers,
    channels and outbox;
  - keeps seen;
  - `state.purged` filters a later archive item;
  - purging an id twice is a no-op.
- health:
  - shadow is quiet only;
  - relay with the archive covering gives a note and no loud; both down gives one loud a day after both failed;
  - one loud a day across the family; 08:00 IST and `/pause`; recovery;
  - stalled from `since` with no heartbeat yet; stalled because every sub failed;
  - the gap rule.
- recheck: a relay copy settles a queued entry (quiet note at most); a late never-seen item, fresh → normal path,
  old → `applyLateRelayItem`.
- shadow: the counts of §4.8 from fake archive and relay streams; a seller mismatch is flagged.
- config: relay key validation; secrets redacted in logs and `--status`.
- AI: the `paidTier` guard; the `FreeTier` quotaId → `ai-off`.
- docs: README and example text tests for "billing", "paidTier" and "relay".

### 4.7 Playtest checklist (`notifiers: test`, then perfume)

P4, P7 and P20 need perfume on: they come after `notifiers` is switched to perfume (§4.3 step 4). P3, P11 and
P16 read `#relay-test` on the Mac with the perfume notifier's `--check-relay test` (§4.3 step 2).

| # | Check | Pass when |
|---|---|---|
| P1 | `npx devvit upload` | app and r/<relay-sub> exist; you are its moderator; `npx devvit list installs` shows it |
| P2 | secrets and settings | `settings list` names all five secrets but not which have values, so: after the last `settings set` the logs show no new `config.problems` line (a missing or malformed signing key, or a webhook URL that isn't one, is reported there), and a `hello` reaches `#relay-test` and `#perfume` within 10 min; a bad seller line is refused by validation |
| P3 | Discord | `hello` in `#relay-test`; a short post inline; a 3,000-character test post goes **as an attachment** and on the Mac the perfume notifier's `node src/run.mjs --check-relay test <#relay-test channel id> <its webhook id>` reads it (every signature OK, the file read). Note whether a body written with `&` arrives with `&amp;`. **If multipart fails, switch to `part` messages (§1.9.2)** |
| P4 | reading other subs (after perfume is on) | logs show `getNewPosts` on r/IndiaFragMarketplace with titles, bodies, flair and gallery count; compare 3 posts by hand with Reddit. Note the gallery URL host, and whether `approvedAtUtc` is 0 |
| P5 | seeding | the first pass relays nothing old |
| P6 | the full flow in your own sub | create a post, a comment and a reply, then edit and delete. Expect relays within 1–2 min; `op` right; context copies; with `testFastRecheck`, the Discord message deleted and a `purge` within about 5 min |
| P7 | deep pass (after perfume is on) | 4 pages, no duplicate ids, within the 25-s budget |
| P8 | comments | in r/<relay-sub>, read with `testComments` = `<relay-sub>=1` (sneaker stays off): a post with new comments has them relayed; posts without changes cost nothing |
| P9 | rate limits | a test burst of 20 messages: headers logged, no loss, no 429 storm |
| P10 | webhook broken | delete `#relay-test`'s webhook: logs show 404 (`drain.webhook-broken`) and the notifier stopped. A deleted webhook can't be restored: make a new one, set it with `npx devvit settings set testWebhookUrl`, and use its id in `--check-relay test` from then on (the app can't delete what the old one sent) |
| P11 | heartbeat | every 10 min; `--check-relay test` shows the newest |
| P12 | a bad sub name | its heartbeat `err` is set; other subs unaffected |
| P13 | duration and locks | every job under 20 s in the logs; no overlapping runs |
| P14 | hashing | the app's `handleHash("relay_test_handle")` = `1bebdbf264ea315a` |
| P15 | upgrade | upload a new version and install it: `hello`, no re-relay flood, crons rescheduled |

### 4.8 Shadow numbers to watch (per notifier, at least 14 days; `--relay-report --days 14`)

| Number | Target to switch to `relay` |
|---|---|
| Visible archive posts the relay also delivered | ≥ 99.5% (every miss listed by id) |
| Held posts (archive saw `[removed]`) later delivered by the relay as `late` | ≥ 99% of those the archive re-check found readable; approval delay median and p90 shown |
| Posts only the relay saw | shown (expected > 0) |
| Latency, relay `seen − at` | median ≤ 90 s, p95 ≤ 5 min (next to the archive's `retrievedOn − at`) |
| Normalised title and body equal | ≥ 98% (differences should be edits) |
| Flair equal | 100%; image presence ≥ 99% |
| Relay seller id = the PC's own hash-based seller id | **100%** |
| identify offers equal (m, ml, cd, p, seller) | ≥ 99%; every difference explained |
| Comments delivered (sneaker and protein) | ≥ 98% for every-pass subs, ≥ 95% for every-third; median latency ≤ 3 min |
| Heartbeat gaps over 30 min | 0, apart from announced upgrades |
| Discord 429s and `dropped24h` | 429s low and spread out; dropped = 0 |
| Purges | count per day; every purged id absent from the price log (self-check) |
| Duplicates (the same id delivered twice) | ≤ 0.5% (harmless) |

---

## 5. Open questions and risks (each with its fallback)

### 5.1 Questions for you

| # | Question | Default / recommendation | Fallback |
|---|---|---|---|
| Q1 | The 400-day re-check records in Redis (ids only), so the 400-day Reddit price history stays deletable? Or a strict 30-day horizon for everything? | 400 d for perfume and sneaker | 30 d, with the PC dropping Reddit records older than 30 d (§2.9) |
| Q2 | Should IFM's held posts (title + flair right away) still reach you after the archive closes? Today a BNIB title with a price and a size can ring before approval | Accept approval-time relays (`late`), with history and quiet notes as today's re-checks (your choice, 28 Oct 2026: the subs' moderators shouldn't have to change anything) | Ask IFM's mods. It needs a **published** (unlisted) version installed in IFM by its mods, using `onAutomoderatorFilterPost` and the mod queue, and it relays only title + flair for held posts |
| Q3 | Same-author links: `same-sub` (repost checks in sneaker and protein, the perfume gate within a sub), `notifier-subs` (also perfume cross-posts across DFA, IFM and PerfumeIndia), or `off` | `same-sub` | `off`: protein dedupes by title within 15 min, sneaker by models + title within 60 min, and perfume individuals may ring twice for one cross-post within 48 h |
| Q4 | Shallow pass every minute (about 11.5k listing calls a day) or every 2 minutes? | every minute | `*/2` when the logs show rate limits |
| Q5 | One reader bot for all three notifiers, or one per notifier (like the separate Telegram bots)? | one bot, with channel permissions | three bots, each seeing only its own channel |
| Q6 | More than about 55 seller accounts some day? | one setting | `perfumeSellerAccounts2` |
| Q7 | App name | `hyd-deal-relay` | any free 3–20 character name starting with a letter |
| Q8 | Delete Telegram alerts of purged posts (possible within 48 h)? | no (v1) | v2: outbox entries keep the Telegram message id, and a purge calls `deleteMessage` |

### 5.2 Risks

| # | Risk | Fallback |
|---|---|---|
| R1 | Devvit's Reddit API limits are not documented, and reading other subs could be limited later | the lighter profile (`*/2`, deep every 2 h); ask r/Devvit (Discord #data-api-app-migration); in the end, the subs' mods install a published version in their subs and triggers replace polling |
| R2 | AutoMod-held IFM posts are invisible until approved (2–8 days at worst), so title-only rings end when the archive does | Q2 |
| R3 | A job runs past 30 s, or scheduler minutes slip under load | deadlines (25 s), Redis locks, work left in queues; split shallow into 2 jobs (2 of the 5 free cron slots) |
| R4 | Devvit changes fast (0.15.0 removed devvit.yaml a week ago); "works within 2–3 versions" isn't in the docs | monthly `npx devvit update app` + `npm test` + upload; watch the changelog; the heartbeat's `app` version shows in `/status`; roll back with `devvit install … @version` |
| R5 | Devvit's fetch mangles a hand-built multipart body | `part` messages (§1.9.2), already in the PC parser |
| R6 | A webhook URL leaks (fake items, fake purges, fake seller ids) | HMAC on every message; rotate the webhook (new URL → `settings set`) and the key; the PC ignores unsigned messages and counts them in `--check-relay` |
| R7 | Discord disables the bot or webhook, or changes the Message Content intent | Content intent: put `<@botId>` in `content` (mentions are exempt). Discord gone: Telegram (api.telegram.org is allowlisted; a private channel the PC's bot reads), or Devvit External Endpoints (request access; the PC pulls with a managed token, 5 req/s, 10 MB) |
| R8 | Shared Devvit egress IPs trip Discord's Cloudflare ban (10,000 invalid requests in 10 min, counted per IP), whoever caused it | never retry 401/403, always obey 429; outbox up to 48 h; relay-stalled tells you; the R7 fallbacks |
| R9 | Deletions are seen by polling (hours, not instantly), and delivered alerts stay on your phone | first checks at +1 h and +6 h; the Privacy page says so; Q8 |
| R10 | Comments on posts older than the newest 100 are missed | raise to 200 for one sub (2 requests); a weekly deep comment sweep of the last 7 days' relayed posts |
| R11 | `numberOfComments` doesn't change when one comment is deleted and another added | every 5th comment pass also re-reads the 5 newest posts' trees |
| R12 | Settings over 2 KB | split settings (Q6) |
| R13 | An unpublished app could lose features, or a playtest sub could be cleaned up | a published unlisted version (review about a week, README required) |
| R14 | The app's view of a post differs from the archive's (Markdown vs entity-escaped text) | the shadow text-equality number shows it; `normalizeText` decodes entities either way |
| R15 | The AI on relay images goes to a free-tier key | `ai.paidTier` + FreeTier detection (§3.6); Google budget alerts |
| R16 | The PC is off for over 14 days | relay items past the app's age-out are gone (the archive era only read 13 h back anyway); deals are stale by then |
| R17 | Clocks: cron in UTC, the IST rules on the PC | the PC judges by its own clock; heartbeat `at` is compared using the Discord `Date` header (the existing clock-skew check) |

---

## 6. Build order

1. **App core** (pure modules + tests, no Reddit): `sha256`, `sellers`, `item`, `passes`, `encode`, `discord`, `recheck`,
   `outbox`; the contract fixtures; the perfume `scripts/relay-vectors.mjs` and `scripts/relay-settings.mjs`.
2. **App jobs + playtest** P1–P15 with `notifiers: test`.
3. **Perfume reader in shadow:** `discordRelay.js`, `relayStep.js`, `relayPurge.js`, `relayShadow.js`, health, config,
   `--check-relay`, `--relay-report`; paid Gemini text + `paidTier`. Then 14 days of shadow.
4. **Sneaker and protein readers in shadow** (§2.13).
5. **`source: "relay"`** for each notifier whose numbers pass §4.8 (archive backup on).
6. **After the archive closes:** `archiveBackup: false`; the phase-2 items of §2.12 as needed.

---

## 7. Sources (checked 28 Oct 2026)

Devvit docs, `https://raw.githubusercontent.com/reddit/devvit-docs/main/docs/…`:
- `capabilities/devvit-web/devvit_web_configuration.md`: keys, CommonJS server, settings, permissions, scheduler tasks,
  name rules, the triggers list
- `capabilities/devvit-web/devvit_web_overview.mdx`: max request time 30 s, payload 4 MB
- `capabilities/server/scheduler.mdx`: per installation, once a minute, seconds cron experimental, 10 recurring,
  `runJob` limits, TaskRequest/TaskResponse
- `capabilities/http-fetch.mdx`: global allowlist (discord.com, api.telegram.org), review 1–2 business days, https
  only, methods, 30-s timeout, Terms/Privacy, the README "Fetch Domains" format
- `capabilities/server/settings-and-secrets.mdx`: global vs subreddit, types, `isSecret`, CLI, 2 KB, at least one
  install before secrets
- `capabilities/server/redis.mdx`: commands, 5 GB / 5 MB / 40k/s / 20 transactions / 5 s, no SCAN, no sets or lists,
  per-installation namespace
- `capabilities/server/reddit-api.mdx`; `api/redditapi/RedditClient/classes/RedditClient.md`: method list,
  getNewPosts, getComments, getPostById, getCommentById, searchPosts, getEdited
- `api/redditapi/models/classes/Post.md`, `Comment.md`; `models/type-aliases/GetPostsOptions.md`,
  `ListingFetchOptions.md`, `GetCommentsOptions.md`, `CommentSort.md`, `CommonFlair.md`, `GalleryMedia.md`
- `devvit_rules.md`: Handle data with care; external services (Terms, Privacy); Enable and respect user deletions
  (30 days, "even if … anonymized"); Generative AI (approved LLMs Gemini and ChatGPT; no training); README requirements
- `guides/faq.mdx`: upload vs publish, logs 5,000 events / 7 days, limits, review times
- `guides/tools/playtest.md`, `guides/tools/devvit_cli.mdx`, `guides/tools/devvit_test.mdx`
- `capabilities/server/triggers.mdx`, `capabilities/server/global-triggers.mdx`, `capabilities/server/external-endpoints.mdx`
- `guides/migrate/public-api.md`, `guides/migrate/public-api/basics.md`: the migration program, External Endpoints and
  HTTP allowlisting exemptions, "apps install per-subreddit"
- `changelog.md`: 0.15.0 (19 Oct 2026, devvit.yaml removed), 0.15.1 (26 Oct 2026)

Discord docs:
- `https://docs.discord.com/developers/resources/webhook`: Execute Webhook, `wait`, `flags` (only SUPPRESS_EMBEDS,
  SUPPRESS_NOTIFICATIONS, IS_COMPONENTS_V2), files / `payload_json`, Edit/Delete Webhook Message (204)
- `https://docs.discord.com/developers/resources/message`: Get Channel Messages (VIEW_CHANNEL + READ_MESSAGE_HISTORY,
  `after`, limit 1–100, newest first), Delete Message (MANAGE_MESSAGES), Bulk Delete (2–100, under 2 weeks), content
  2,000, embed limits (description 4,096, 6,000 in total, 10 embeds), request 25 MiB
- `https://docs.discord.com/developers/topics/rate-limits`: headers, the 429 body `retry_after`/`global`, 50
  requests/s, 10,000 invalid requests per 10 min
- `https://docs.discord.com/developers/events/gateway`: the Message Content intent covers the HTTP API too, and its
  exceptions
- `https://github.com/discord/discord-api-docs/issues/6753`: webhooks in one server sharing a bucket (a 2024 report)

Gemini:
- `https://ai.google.dev/gemini-api/docs/pricing` (last updated 2026-10-27 UTC): `gemini-3.5-flash-lite` paid $0.30 / $2.50
  per 1M tokens, batch $0.15 / $1.25, "Used to improve our products" free Yes / paid No
- `https://ai.google.dev/gemini-api/docs/media-resolution`: 1,120 tokens per image by default (Gemini 3)
- `https://ai.google.dev/gemini-api/terms`: Paid Services (only with a billing-enabled Cloud project), data use,
  human review on unpaid
- Exchange rate: USD/INR 95.35 on 27 Oct 2026 (tradingeconomics.com / wise.com)

Added after the third review (28 Oct 2026):
- Gemini: `https://ai.google.dev/gemini-api/docs/changelog` (26 Oct 2026: new paid-tier projects store Generate
  Content API requests by default), `https://ai.google.dev/gemini-api/docs/logs-datasets` (paid tier only, 55 days by
  default, the Settings panel), `https://ai.google.dev/api/generate-content` (the request's `store` field wins over the
  project setting), `https://ai.google.dev/gemini-api/docs/usage-policies` (the 55-day abuse log).
- Devvit: `@devvit/cli` 0.15.1 `dist/commands/upload.js` and `dist/util/project.js` (the first upload creates the
  playtest sub, installs the app there and writes `dev.subreddit` into devvit.json), `dist/commands/settings/list.js`
  (names only); `@devvit/public-api` 0.15.1 `Devvit.d.ts` (Install Settings at
  `https://developers.reddit.com/r/<sub>/apps/<app>`); `guides/tools/playtest.md` (the playtest sub is private).
- Discord: `https://docs.discord.com/developers/topics/permissions` (how @everyone, role and member overwrites
  combine), `https://docs.discord.com/developers/resources/application` (`bot_public`),
  `https://github.com/discord/discord-api-docs/issues/5292` (Public Bot off is refused while an Install Link is set).
- Reddit Help, "Changing your Community Type": Mod Tools → General Settings → Privacy & Discovery.

The notifiers (read, not changed):
- perfume `src/redditStep.js`, `src/sources/http.js` (toItem), `src/sources/arcticShift.js`, `src/recheck.js`,
  `src/health.js`, `src/core/sellers.js`, `src/core/identify.js`, `src/core/offers.js`, `src/core/decidePost.js`,
  `src/core/observe.js`, `src/priceLog.js`, `src/pipeline.js`, `src/runner.js`, `src/aiPhase.js`,
  `src/vision/gemini.js`, `src/config.js`, `src/state.js`, `src/log.js`, `config.example.json`, `README.md`,
  `docs/WINDOWS.md`
- sneaker `src/sources/http.js`, `src/core/decide.js`, `src/core/observe.js`, `src/core/format.js`, `src/config.js`,
  `config.example.json`
- protein `src/sources/http.js`, `src/sources/arcticShift.js`, `src/core/decide.js`, `src/core/format.js`,
  `src/runner.js`, `src/state.js`, `config.example.json`

---

## 8. Review corrections (28 Oct 2026)

This is a sceptical review of §0–§7. It was checked against the Devvit docs and Rules (re-read on 28 Oct 2026 at
`raw.githubusercontent.com/reddit/devvit-docs/main/docs/…`), Discord's docs, and the code of the three notifiers.
**Where this section and §0–§7 disagree, this section wins** (and §8.10, the third review, wins over the rest of §8). Each item says what breaks, the evidence and the fix:
- **[must]**: it breaks a Devvit Rule, loses or floods data, or doesn't run as written.
- **[should]**: a real gap with a cheap fix.
- **[verify]**: it goes into the playtest (§8.8).

**Checked and holding (no change):**
- the handle-hash vectors of §1.6.4 (recomputed with the notifier's rule);
- `flags` 4100 and the bot permissions 74752;
- discord.com and api.telegram.org are on the global fetch allowlist, and are still listed in `domains` ("Apps must
  request each individual domain that it intends to fetch, even if the domain is already globally allowed",
  http-fetch.mdx);
- the README "Fetch Domains" format; the Terms and Privacy links "must be saved in the app details form";
- settings: 2 KB per value; secrets are global only and need one installation first;
- RedditClient.md has no subreddit-wide comment listing and no batch lookup by ids;
- Discord: content 2,000; bulk delete 2–100 ids under two weeks; without the Message Content intent, `content` and
  `attachments` come back empty over HTTP too;
- the Gemini 3.5 Flash-Lite price: $0.30 / $2.50 a million tokens, on a page "Last updated 2026-10-27 UTC";
- the sneaker and protein line references of §2.13.

### 8.1 Reading posts and dedupe (app)

**RC1 [must] The seen-set read stops at 1,000 ids, so the newest posts are sent again every minute.**
- §1.6.1 loads `ZRANGE rr:seen:<s> (now − 30 d) +inf BYSCORE` into memory. redis.mdx: "BYSCORE/BYLEX: LIMIT count
  capped to 1000 per call (server default) … Client default for by: 'score'/'lex' is count=1000 when no limit is
  provided."
- The range is ascending, so the 1,000 ids that come back are the oldest. IFM alone passes 1,000 in about three
  weeks: 40–50 posts a day, plus the removed posts marked seen. From then on, every post in the shallow listing looks
  new.
- The outbox key doesn't stop the repeat either: after a send, `rr:msg:<key>` is deleted (§1.10, the 2xx row).
- **Fix:**
  - Never load the set. Check the listing's own ids instead, with one `zScore(rr:seen:<s>, id)` per id. That is 100
    calls per sub, well within Redis's limit of 40,000 commands a second.
  - `enqueue` also refuses an id that already has an `rr:item` record.
  - Every other `zRange` by score passes an explicit `limit`, and reads the next page when exactly that many come back.

**RC2 [must] Old posts of slow subs come back after 30 days.**
- `rr:seen` is scored by createdUtc and trimmed at 30 days (§1.8), and the read leaves out members older than 30 days.
- So in two cases, posts older than 30 days count as "new" on every pass:
  - when a sub's newest 100 posts reach back more than 30 days (a quiet month in r/protein_deals or a slow sneaker sub);
  - in the catch-up read of 300 posts per sub (§1.6.2).
- **Fix:** an age gate in `newPosts`.
  - Posts: one with `createdAt < now − 12 d` is never relayed as new. That is past the deep pass's 10-day reach (RC4).
    The app marks it seen and skips it.
  - Comments: one created more than 48 h ago is never relayed as new (ctx copies are exempt). Sneaker and protein drop
    comments first seen more than 6 h after posting anyway (sneaker `src/core/decide.js:20,1323`, protein
    `src/core/decide.js:13,227`).
  - The rule for the builders: missing state never means "relay everything".

**RC3 [must] The catch-up never runs.**
- In §1.6.2, the deep pass checks `rr:meta.lastShallowAt`. But the shallow job runs every minute and rewrites that
  value as soon as the app is back, so the deep pass at :07 always sees a fresh value.
- **Fix:**
  - The shallow pass checks the gap before it writes `lastShallowAt`. If `now − lastShallowAt > 30 min`, it sets
    `HSET rr:meta catchUpFrom <the old value>`.
  - The deep pass reads `catchUpFrom` and clears it.

**RC4 [should] 400 IFM posts don't reach back 8–9 days in a busy week.**
- "About 300–400 posts" holds only at 40–50 posts a day; at 60 a day, 8 days is 480 posts.
- Today's re-check reads at +9 d, with a day of grace (`src/recheck.js:13-15`).
- **Fix:**
  - Page `getNewPosts` 100 posts at a time, until the oldest post on a page is older than `now − 10 d`. Stop at 1,000
    posts (10 requests).
  - `perfumeDeep` becomes `IndiaFragMarketplace=10d`.
  - The shadow report's late deliveries decide whether PerfumeIndia or DFA need a deep entry too.

**RC5 [must] Read errors are retried every 30 minutes, forever.**
- §1.11: "An error from getPostById is never a verdict: retry in 30 min, with no limit, until keepDays ends."
- Some posts then fail at every check: those of a sub that goes private, is banned or is quarantined, and posts that
  are hard-deleted (not found).
- IFM alone builds up about 45 × 400 = 18,000 records, and the recheck job reads 40 per run (5,760 a day). The failing
  records fill every run, so other posts' deletion checks starve, and nothing that fails is ever purged.
- **Fix:**
  1. Each record backs off: 30 min, 2 h, 8 h, 1 d, then every 3 d.
  2. A post that answers "not found" at 3 checks, spread over at least 24 h, is purged with `why: "gone"`.
  3. Each sub gets a breaker. When the sub's own listing has failed with 403 or 404 for 24 h, its re-checks stop and
     the heartbeat sets `subs.<Name>.err`. After 7 days, everything relayed from it is purged with
     `why: "unavailable"`: its content isn't public any more, and its deletions can't be seen.
  4. Tell 403 and 404 apart from 5xx and timeouts by the error the client throws (P17).

**RC6 [must] `classifyPost` misses a removal category.**
- Post.md gives these values for `removedByCategory`: `"anti_evil_ops"`, `"author"`, `"automod_filtered"`,
  `"community_ops"`, `"content_takedown"`, `"copyright_takedown"`, `"deleted"`, `"moderator"` and `"reddit"`. §1.11
  leaves out `author`.
- **Fix:**
  - Any non-empty `removedByCategory` is not `ok`, including values Reddit adds later.
  - `deleted` and `author` give the verdict `deleted`; all the others give `removed`.
  - `isRemoved()`, `isSpam()` and the `[removed]` / `[deleted]` markers stay as extra signals.

**RC7 [must] Deletion checks must not depend on what is switched on.**
- `relayPaused` makes "Jobs return at once" (§1.4.2), and the passes loop "for each notifier n that is on".
- So any of these would stop re-checks and purges for records the PCs still hold: a pause, a notifier turned off, or a
  sub taken out of `<n>Subs`.
- **Fix:**
  - `relayPaused`, `notifiers` and the sub lists stop **new** relays only.
  - Re-checks, purges, the age-out and heartbeats always run over every `rr:item` record, whatever its notifier or sub.
  - A notifier's webhook secret stays required until its last record expires.

**RC8 [should] A reinstall loses the re-check records.**
- Redis is per installation ("Each installation of an app is uniquely name-spaced", redis.mdx). Upgrades keep Redis;
  reinstalls don't.
- An uninstall and reinstall, or a new playtest sub (R13), starts with empty Redis. The PCs still keep up to 400 days
  of relay-derived records, and nothing checks them for deletion any more.
- **Fix:**
  - `onAppInstall` writes a random `rr:meta.installId`, and `hello` and every heartbeat carry it.
  - A PC that sees the id change sends a quiet note: "the relay app was reinstalled: posts relayed before <date> are
    no longer checked for deletion". It then applies your answer to Q11.

**RC9 [should] Use one Reddit-call budget for all jobs.**
- Devvit documents no Reddit API limit (faq.mdx, reddit-api.mdx). For comparison, Reddit's Data API wiki allows 100
  queries a minute per OAuth client, averaged over 10 minutes.
- The design averages 12 calls a minute, but its worst minute is well over 60. That minute adds up:
  - 8 listings;
  - 5 comment subs × 4 posts × 4 requests;
  - the hot listings of RC10;
  - re-checks.
- **Fix:**
  - A token bucket in Redis (`incrBy rr:budget:<minute>` + `expire`) allows 60 Reddit calls a minute across all jobs.
  - Work over budget waits in its queue.
  - The heartbeat reports the day's busiest minute.

### 8.2 Comments (sneaker, protein)

**RC10 [must] Comments in older threads are missed, and sneaker can ring on them.**
- §1.7 reads comments only on the newest 100 posts. Sneaker today reads every new comment of its subs
  (`src/runner.js:240-281`).
- A comment in a thread older than 2 h can still ring there, with a shop link or a live cue
  (`src/core/decide.js:1072-1084`).
- Stickied megathreads and AutoModerator weekly threads drop out of the newest 100 within days.
- **Fix:**
  - Each comment pass also reads `getHotPosts({ subredditName, limit: 25 })`. That is 1 request, and stickied posts
    come first.
  - The hot listing is merged with the new listing for `changedPosts`. Hot posts feed only the comment pass and ctx
    copies; they are never relayed as posts.
  - `rr:cc` drops a post's field only once the post has been in neither listing for 2 days (not after "8 d").
  - The first time a post is seen, its count is stored and no backlog is relayed.
  - R10's fallbacks stay.

**RC11 [should] `changedPosts` can starve other posts.**
- "Incomplete trees stay changed for the next pass", and the tie rule ("larger change, then newer post") lets one big
  thread take all 4 slots on every pass.
- **Fix:**
  - Take changed posts round robin: by `lastReadAt`, oldest first.
  - Keep `doneUpTo` per post (the createdUtc of the newest comment fully read), so progress through a big tree
    carries over.
  - Every changed post is read at least once every 5 passes.

**RC12 [must] Context copies follow the wrong clock.**
- §1.7 sends a post's ctx copy only "if post not relayed in the last 3 d". The PCs keep post facts for a shorter or
  different time:
  - protein keeps post and comment facts 48 h, by createdUtc (`src/state.js:9,125-126`);
  - sneaker keeps post facts 7 d (`src/state.js:15,461`).
- So a comment on a protein post relayed 2.5 days ago arrives without context. Today protein fills that gap with an
  archive lookup (`src/pipeline.js:62-73`), which ends with the archive.
- **Fix:**
  - Send the post's ctx copy when `now − post.createdAt > ctxAfterSec[n]`, or when the post was never relayed.
    `ctxAfterSec` is 24 h for protein and 5 d for sneaker and perfume.
  - The same rule applies to a parent comment's ctx copy.
  - Everything relayed as ctx gets its own `rr:item:p` or `rr:pc` entry, so its deletion is checked like the rest.

**RC13 [must] Comment deletion checks don't fit the budget.**
- §1.11's 3,000–3,500 checks a day count posts only.
- Sneaker keeps comment-derived price records 400 days (`src/priceLog.js:11`). Those records are `k: 'c'` and carry
  `op` (`src/core/observe.js:1251,1410-1411`).
- So every post with relayed comments gets about 19 tree walks over 400 days, each of up to 4 requests. If 300 relayed
  comments a day fall on about 100 posts, that is about 1,900 walks a day. The recheck job does 10 per run, which is
  1,440 a day.
- The backlog would grow for a year, and deletions would be honoured later and later.
- **Fix (Q10):**
  - The app keeps sneaker's comment records for 30 days (`sneakerCommentKeepDays` 30). The sneaker PC drops `k: 'c'`
    records older than 30 d.
  - Post records keep their own horizon.
  - A post's comment tree is walked at +1 h, +1 d, +7 d and +28 d only.
  - These walks are counted in §1.5 and in RC9's bucket.
  - The heartbeat's `recheck` gains `oldestDueSec`, and the PCs note a backlog over a day.

**RC14 [should] Comment classification has gaps.**
- Comment.md has no `removedByCategory`. A non-moderator sees:
  - a removed comment as `[removed]`;
  - a deleted comment as `[deleted]`;
  - a comment whose author deleted the account with its body intact and `authorName` "[deleted]".
- §1.11 purges only the first two.
- **Fix:**
  - `author-deleted` applies to comments too, because `op` is derived from the author.
  - A relayed comment missing from a **complete** tree is confirmed with one `getCommentById`, since Reddit leaves
    deleted comments without replies out of the tree. A `[deleted]` or `[removed]` body purges it; "not found" goes
    through RC5's not-found rule.
  - A comment missing from an incomplete tree is confirmed the same way.

**RC15 [should] Comment reads have no health signal.**
- The heartbeat reports listing reads only.
- **Fix:** add `subs.<Name>.comments: { ok, err, errAt, relayed24h }`. Sneaker and protein raise their quiet-first note
  when comment reads have failed for 30 min.

### 8.3 Discord sending and the message format

**RC16 [must] Purge notices must never expire.**
- Purges wait in the outbox under the 48-h TTL of `rr:msg:<key>`, and `rr:out:<n>` expires after 3 days (§1.8). In
  Discord, the 14-day age-out deletes them (§1.11).
- So they are lost when Discord is out for over 48 h, or when a PC is off for more than 14 days (R16). The PC then keeps
  records of deleted posts for up to 400 days.
- The Rules: "Any retention of content and data that has been deleted, even if disassociated, de-identified or
  anonymized, is a violation" (devvit_rules.md).
- **Fix:**
  - Purge notices hold ids only, so they get no TTL in the outbox and are left out of the age-out.
  - The PC deletes each notice after applying it. One older than 14 days needs a single DELETE, because bulk delete
    refuses it.

**RC17 [must] Pace each webhook at 30 messages a minute or less.**
- Discord: "A webhook can only send 30 messages per minute" (https://discord.com/safety/using-webhooks-and-embeds).
  With one request a second across all webhooks, a backlog (catch-up, a comment burst) can send one channel 60 a
  minute.
- The egress IP is shared by every Devvit app, which makes this worse in two ways:
  - Every 429 counts toward that IP's limit of 10,000 invalid requests per 10 minutes ("401, 403, and 429 statuses",
    rate-limits).
  - Webhook calls carry no Authorization header, so Discord's global limit of 50 requests a second "is applied to the
    IP address".
- **Fix:**
  - Wait at least 2.2 s between sends on one webhook (at most 27 a minute), with the 1-a-second limit on top.
  - Every send takes the drain lock, heartbeats included: two jobs updating `rr:dc:pace` without a lock can both send.
  - Add to R8 as a fallback (not v1): a sender bot that calls with `Authorization: Bot` takes the global limit off the
    shared IP.

**RC18 [must] Purges, heartbeats and hellos must always fit inline.**
- A purge of 100 fullnames, at the 16-character comment ids planned in §1.7, is about 2,200 characters ("t1_" + 16,
  plus quotes and commas). That is over the 1,900 inline limit.
- It would then go as an attachment named `purge-….json`. The PC's filename rule `^(post|comment)-…` (§2.2) refuses
  that file, so the purge is lost.
- **Fix:**
  - Split `ids` so that an encoded purge stays at or under 1,800 characters (about 75 ids at worst).
  - `encodeRelay` throws if a `purge`, `hb` or `hello` would not fit inline.

**RC19 [should] Make inline payloads ASCII-only.**
- The signature covers the exact text, so any change Discord makes to `content` breaks it: Unicode normalisation,
  stripping invisible or bidi characters, U+2028. The item then fails its signature check and is lost.
- **Fix:**
  - Encode inline JSON with every character above U+007E written as `\uXXXX`. It is still valid JSON, and `JSON.parse`
    gives the same object.
  - Measure the 1,900 limit on that encoded text.
  - Attachments stay UTF-8.
  - P16 tests it.

**RC20 [should] Don't drop items that a filter blocked.**
- §1.10 drops every 400 as "a bug in our payload". But Discord also answers 400 when a filter blocks a message
  (opcodes-and-status-codes):
  - 240000 "Message blocked by harmful links filter";
  - 200000 "Message was blocked by automatic moderation".
- Reddit posts link to all sorts of shops, so this will happen.
- **Fix:**
  - On those codes, resend once as an attachment, then drop.
  - Count `rejected` in the heartbeat's `out` (it isn't there today), so the PC can say how many items never arrived.

**RC21 [should] Purge deletes need their own retry queue.**
- §1.11 runs "DELETE every Discord message of it" inside "one transaction per item". A fetch can't run inside a Redis
  transaction, and a 5xx or a timeout on that DELETE isn't handled.
- **Fix:**
  - A purge enqueues its notice (no TTL, RC16), and adds `ZADD rr:del <n>:<msgId>` for each copy.
  - The drain sends the deletes with the same pacing and back-off. A 404 counts as done.

### 8.4 Settings, config and secrets

**RC22 [must] Select defaults are strings.**
- settings-and-secrets.mdx: a select's default is a string (`"defaultValue": "production"`); a multiSelect's is an
  array (`["welcome"]`). §1.3 gives `sameAuthorLinks` the default `["same-sub"]`.
- **Fix:** `"defaultValue": "same-sub"`. `parseInstallSettings` accepts either a string or a one-element array (P18).

**RC23 [should] The docs settle two "(verify)" notes.**
- validationEndpoint (settings-and-secrets.mdx): the request is `{ "value": … }`, and the answer is
  `{ "success": true }` or `{ "success": false, "error": "…" }`.
- Number settings have no documented min or max, so add a validation endpoint for `<n>KeepDays` (14–400) and clamp it
  in `config.ts` too.

**RC24 [should] Locks and transactions.**
- redis.mdx documents `set(key, value)` and a separate `expire(key, seconds)`, with no `nx` or `expiration` option. It
  allows "20 concurrent transaction blocks; 5-second execution timeout".
- **Fix:**
  - Take locks with `watch(key)` → `get` → `multi()` → `set` + `expire` → `exec()`. A failed `exec` means another run
    holds the lock.
  - Keep at most 5 transactions in flight: enqueue a pass's items one after another, not one `multi` per item in
    parallel.

**RC25 [must] Port `authorFlairOf` together with the flair rule.**
- `sellerOfAuthor` (`src/core/sellers.js:285-295`) is given `it.authorFlair`, which `toItem` has already cleaned
  (`src/sources/http.js:221,242-256`):
  - "(…)" and "[…]" are dropped;
  - `:emoji:` codes are dropped;
  - whitespace is collapsed, and the text is cut at 80 characters.
- §1.6.4 ports `flairKey` and the regex, but matches the raw `authorFlair.text`. A flair like ":verified: Seller : Well
  Scented" fails the `^\s*seller` anchor. That breaks the flair path and §4.8's 100% seller target.
- **Fix:**
  - `sellers.ts` gets `authorFlairOf`, tested with made-up flair vectors that `scripts/relay-vectors.mjs` writes.
  - CommonFlair.md calls `text` the "Plain text representation of the flair"; P20 checks whether it keeps emoji codes.

**RC26 [should] Strip u/ handles from relayed text.**
- "No usernames … are ever sent" isn't true yet: titles, bodies and comments quote `u/name`.
- All three notifiers delete these tokens before reading anything, so removing them in the app changes no decision:
  - perfume: `HANDLE_TOKEN`, used by `cleanText` (`src/core/normalize.js:45-52`);
  - sneaker and protein: normalize.js, "u/ r/ tokens → spaces".
- **Fix:**
  - Before capping, `postToRelay` and `commentToRelay` remove the user forms of `HANDLE_TOKEN` (`u/`, `/u/`, `user/`,
    `/user/`) the way `stripHandles` does.
  - They turn `reddit.com/u/<name>` and `reddit.com/user/<name>` links into `reddit.com/user/[user]`.
  - `@handles` stay: perfume's seller detection reads Instagram handles (`src/core/sellers.js:22`).
  - Shadow mode runs the same function on archive text before hashing it.
  - The Privacy page can then say plainly that no usernames are forwarded.

**RC27 [should] Key the heartbeat's seller checksum.**
- `sellers` is the first 8 hex of sha256 over the hash lines, so a hash of handle hashes leaves Reddit.
- **Fix:** send `hmacSha256Hex(relaySigningKey, sortedLines).slice(0, 8)`; the PC computes the same with its key.

**RC28 [verify] Secrets in the platform's logs.**
- `redact()` covers the app's own log lines, but an error printed by the runtime may carry the fetch URL, which holds
  the webhook token.
- **Fix:**
  - Wrap every fetch, so that a thrown error is replaced by a new Error with only its kind and status.
  - P19 breaks a webhook and checks `npx devvit logs r/<relay-sub> --verbose` for `/webhooks/`.

### 8.5 Perfume PC: purges, retention, privacy

**RC29 [must] Purge the `.prev` copies too.**
- `savePriceLog` writes with `keepPrevious: true` (`src/priceLog.js:149`), and `writeJsonAtomic` first copies the
  current file to `<name>.prev.json` (`src/state.js:235-246`).
- So after a purge, the old records survive in the `.prev` copies:
  - `state/price-log.prev.json` keeps them until the next price-log save, which can be days away;
  - `state.prev.json` keeps them for one run.
- **Fix:**
  - After a purge, `applyRelayPurges` also rewrites `price-log.prev.json` from the purged log.
  - The run's state save does the same for `state.prev.json`.

**RC30 [must] Rewrite the shipped history; don't only filter it on load.**
- §2.6 leaves purged records in `data/price-history.json` on disk ("not edited on the PC").
- **Fix:**
  - When a purge hits the shipped file, rewrite it atomically without those records (`writePriceHistory`,
    `src/priceLog.js:100-105`).
  - When a new zip brings a changed file (`priceHistoryStamp`, `src/priceLog.js:73-75`), filter it with
    `state.purged` and rewrite it.
- **The Mac [should]:** the Mac's build reads the archive, which keeps deleted posts.
  - The app also posts each purge to a `#purge-log` channel: ids only, never deleted by a PC, aged out at keepDays.
  - `history-backfill.mjs` reads that channel with the bot token. Before it writes the file, it drops those ids and
    their `.replay-cache` pages.

**RC31 [must] Complete the purge table of §2.6.** It misses:
- `state.digest.pending`: quiet alerts that wait up to 12 h for the digest, with titles and links
  (`src/state.js:23,593-598`);
- `state.offers.keys`: loud entries whose channel is `post:P`, and offer keys whose party is `i:r<P>` (§2.5's
  `indHash`);
- the shadow records of §2.10 (`titleH`, `bodyH`, offers);
- the delivered alerts (RC32).

**RC32 [must] Delivered alerts are Reddit content, kept by Telegram and ntfy.**
- Q8 leaves this for later, but alerts carry offer titles, prices, Reddit links and image links
  (`src/core/format.js:316-347`).
- Image alerts go to Telegram by `sendPhoto` with the Reddit image (`src/notify/telegram.js:53-59`), so Telegram
  stores the picture itself. Telegram keeps alerts indefinitely, and the Rules require removal from "any external
  systems".
- What the services allow:
  - Telegram Bot API: "A message can only be deleted if it was sent less than 48 hours ago". `editMessageText`'s
    48-hour limit is only for business messages, so the bot can edit its own alerts at any age.
  - ntfy: a message published with `X-Sequence-ID` can be removed with `DELETE /<topic>/<sequence_id>`. That removes
    it from the server cache and from apps that support it: docs.ntfy.sh/publish lists Android, web and desktop, so
    the iPhone app may keep it.
- **Fix (v1; this closes Q8):**
  1. The outbox records `{ ref: 'post:P', telegramMessageId, ntfySeq }` for each alert it sends (ids only).
  2. On a purge, in Telegram:
     - delete the alert with `deleteMessage` if it is under 48 h old;
     - otherwise edit its text with `editMessageText` to "Removed: the Reddit post was deleted";
     - replace a photo alert with `editMessageMedia` (a placeholder picture), because a caption edit would leave the
       Reddit image;
     - edit a digest to drop that line.
  3. On a purge, in ntfy: `DELETE` by sequence id.
  4. When an item's deletion checks end (its keepDays), its alerts are edited the same way. So nothing that quotes an
     unchecked item is left.
- Neither `src/notify/telegram.js` nor `src/notify/ntfy.js` keeps message ids today. Sneaker and protein need the same
  change.

**RC33 [should] Give the relay step an overall time budget.**
- With 6-s timeouts, 3 pages and 20 attachments, a bad network can take about 2 minutes. The run's phases have fixed
  deadlines: a loud flush ≤ 15 s and Telegram under 95 s (`src/runner.js:2-7`).
- **Fix:**
  - `RELAY_RUN.budgetMs: 15_000`.
  - Attachments are fetched up to 4 at a time.
  - As planned, the cursor stops before the first unfinished message.

**RC34 [should] Match `same` to the offer gate.**
- `same` links posts up to 24 h apart (§1.6.1), but the "also on …" gate lasts 48 h (`POST_LIMITS.offerQuietSec`,
  `src/core/decidePost.js:25`). So a repost 30 h later can ring as a new party.
- **Fix:** use a 48-h window, still with at most 5 ids.
- A known quirk while both sources run: an archive copy's party is the author's hash, but a relay copy's is `r<id>`. So
  a cross-post read once from each source can ring twice. `mergeItems` already prefers the relay copy, so accept this
  for the shadow period.

### 8.6 Health (`src/health.js`)

**RC35 [must] sub-quiet must use arrival time, and never the heartbeat's `newest`.**
- §2.4 raises `lastPostAt` from relay items and from the heartbeat's `subs.<Name>.newest`. That fails in two ways.
- **`newest` hides lost items.** It is what the app saw, not what reached the PC. A relay can read a sub but lose its
  items (a sub-name mismatch, failed signatures, `rejected`), and the sub would never look quiet. That is the very case
  sub-quiet exists for (`src/health.js:183`: "The archive may be missing this sub, so its deals may not reach you").
- **IFM looks quiet almost every day once the archive closes.** IFM posts then arrive only when mods approve them, in
  batches, days after posting.
  - `lastPostAt` comes from createdUtc (`src/redditStep.js:308`), so it lags by days.
  - IFM's threshold is 12 h (`HEALTH.subQuietSec`, `src/health.js:25`).
  - The result: a quiet note most days, and a loud warning after any slow approval day.
- **Fix:**
  - In relay mode, `lastPostAt` is the newest `seen` (the time the app read it) among items that reached the PC.
  - The PC compares the heartbeat's per-sub `items24h` with what arrived, judged only when it has read the channel all
    day. If the app relayed some and the PC got none, the sub-quiet rule applies, and it may ring.
  - If the heartbeat shows the sub read fine (`ok` fresh) and `items24h` is 0, the sub itself is quiet: a quiet note
    only, never a loud warning.
  - In relay-only mode, IFM's threshold is 3 days.

**RC36 [should] Start the relay records afresh when `relay.source` changes.**
- A `relay-stalled` record opened and noted in shadow mode keeps its `since`. After a switch to `relay`, its loud time
  (`since` + 24 h) may already be past, so it rings at once (`step()`, `src/health.js:346-359`).
- **Fix:** a change of `relay.source` deletes the `relay-*` records.

**RC37 [should] Judge the heartbeat only once the read reaches the end of the channel.**
- While a PC catches up after sleep (oldest first, 300 messages a run), the newest heartbeat it has read is old. The
  30-min delay usually hides this, but a catch-up of more than 15 runs gives a false stall note.
- **Fix:** `relayStalled` is `null` (not judged) on any run whose read ended on a full page or a cap.

**RC38 [should] Paused is not stalled.** A heartbeat with `paused: true` gives a quiet `relay-paused` note (with
reminders), never `relay-stalled`.

### 8.7 Rules: GenAI, and the Terms and Privacy texts

**RC39 [must] Paid Gemini meets one GenAI condition, not all of them.**
- devvit_rules.md, "Generative AI/LLM rules", lets an app use approved LLMs when it:
  - "Provides significant and unique benefit to Reddit users and communities through Reddit";
  - uses an approved LLM;
  - doesn't let a third party train on Reddit data;
  - has Terms and a Privacy Policy;
  - keeps to the rate limits.
- §3.1 argues only the training condition.
- The rule is written for an app that calls an LLM itself. Here the PC makes the call, with data the app relayed, and
  a personal notifier hardly meets the first condition.
- **Fix:** before `relay.source: "relay"`, get the Devvit team's written OK (Q9) for two things:
  - relayed images and titles going from your PC to Gemini's paid tier;
  - the seller match: the app compares an author's hash with your registry and forwards the registry id.
- Until then, the AI stays off for relay items (archive items are unchanged): §3.6's guard gains
  `ai.relayImages: false`.

**RC40 [must] Make the Terms and Privacy drafts accurate.**
- The Rules: they "must completely and accurately describe how you and your app collects, uses, shares, and stores
  data".
- **Fix, Terms item 4:** it says Reddit data is "never sold, licensed, shared or commercialised", yet the data goes to
  Discord, Telegram and ntfy, and images go to Google. Say instead: "shared only with the services named in the Privacy
  Policy, to deliver your notifications".
- **Fix, Privacy:**
  - Name Telegram and ntfy.sh as places where alerts go, and how they are removed (RC32).
  - Say that Google keeps paid-tier requests "for a limited period of time, solely for detecting and preventing
    violations", and that those copies can't be deleted on request.
  - For usernames and user flair, replace "Never collected" with "read in memory only, to compare a hash and a flair,
    then dropped; never stored or forwarded".
  - Say that deletion notices (ids only) stay in Discord until read (RC16).
  - Add the reinstall rule (RC8) and, if you choose it, the 30-day comment horizon (RC13).

### 8.8 Tests and playtest additions

**App tests (Vitest):**
- `passes.ts`:
  - a seen set of 1,500 ids never re-relays (RC1);
  - a post older than 12 d, or a comment older than 48 h, is never relayed as new (RC2);
  - a gap of over 30 min sets `catchUpFrom` (RC3);
  - the deep pass stops at 10 days or 1,000 posts (RC4);
  - hot posts join the comment pass but are never relayed as posts (RC10);
  - changed posts go round robin (RC11);
  - ctx copies follow `ctxAfterSec` (RC12).
- `recheck.ts`:
  - the back-off; 3 "not found" answers over 24 h give `gone`; the sub breaker gives `unavailable` after 7 d (RC5);
  - every `removedByCategory` value, plus an unknown one (RC6);
  - paused and switched-off notifiers keep their checks (RC7);
  - a comment by a deleted account, and one missing from a complete tree (RC14).
- `outbox.ts` and `encode.ts`:
  - purges never expire and skip the age-out (RC16);
  - 2.2 s between sends on one webhook (RC17);
  - hb, hello and purge are always inline, and purges split at 1,800 characters (RC18);
  - the ASCII-only round trip (RC19);
  - codes 240000 and 200000 get one retry as an attachment (RC20).
- `sellers.ts`: the `authorFlairOf` vectors (RC25) and the keyed checksum (RC27).
- `item.ts`: u/ handles removed and `@handles` kept (RC26).

**Perfume tests (node:test):**
- a purge rewrites `price-log.prev.json`, `state.prev.json` and `data/price-history.json`, and clears
  `digest.pending`, `offers.keys` and the shadow records (RC29–RC31);
- alert removal, with a fake Telegram (alerts under and over 48 h) and a fake ntfy (RC32);
- sub-quiet by arrival time, and the `items24h` comparison (RC35);
- a source change resets the relay records (RC36);
- an `installId` change (RC8).

**Playtest:**

| # | Check | Pass when |
|---|---|---|
| P16 | ASCII-only inline | a made-up post with combining accents, a ZWJ emoji, U+202E, U+2028 and Devanagari passes the signature check of the perfume notifier's `--check-relay test …`, as in P3 (RC19) |
| P17 | errors | `getPostById` on a made-up id, and on a post in a sub the app account can't see: the thrown error's shape is noted, for RC5 |
| P18 | select value | `settings.get('sameAuthorLinks')` gives a string (RC22) |
| P19 | logs | a broken webhook's error (during P10) in `npx devvit logs r/<relay-sub> --verbose` shows no `/webhooks/` path (RC28) |
| P20 | flair text (after perfume is on) | Devvit's `authorFlair.text` for an emoji flair, next to the archive's `author_flair_text` (RC25) |

### 8.9 Questions added or changed

| # | Question | Default / recommendation | Fallback |
|---|---|---|---|
| Q8 | (changed) Delete the delivered alerts of purged items | **v1**, as in RC32 | — |
| Q9 | Ask the Devvit team in writing to confirm the Gemini step and the seller match (RC39)? | yes, before `source: "relay"` | AI off for relay items |
| Q10 | Keep sneaker's comment-derived records 30 days instead of 400, so their deletion checks fit (RC13)? | 30 d | 400 d, only once the playtest shows the re-check budget holds |
| Q11 | Records that no deletion check covers any more: archive-era Reddit records once the archive closes, and records from before a reinstall (RC8) | keep them until their 400 days end, and say so in the Privacy Policy | drop Reddit records older than 30 days at that point |
| Q2 | (note) If IFM's mods install a published version, that installation has its own Redis and its own Install Settings, which IFM's mods can see | keep the seller lines in your sub's installation only. Held posts aren't public, so relay them only with the mods' explicit agreement | — |
| Q7 | (note) The app name is public as u/<name>, and `hyd-deal-relay` names your city | pick a neutral name | — |

### 8.10 Third review (28 Oct 2026, as built)

The third review checked the second review's fixes. What changed in behaviour and setup (the app's README "Build
notes" and the perfume README say the same):
- **Comment walks:** past its 10 requests a shallow walk skips the listings that need one and still reads the
  pre-filled ones; only the comments it would relay count toward its 500; a thread too big for the fetch cap is taken
  as handled at its comment count, so it is walked again when the count grows, not every pass. Replies behind the
  stubs past the cap stay unread (threads with over 9 "more" stubs).
- **Comment subs' turns:** the subs due in a pass go least recently read first.
- **Usernames in text:** a markdown-escaped name (`made\_up`), a zero-width character next to a token, and DM links
  written `&amp;to=` are scrubbed too; modmail links (`to=/r/<sub>`) stay. The perfume PC's shadow copy of the rules
  (`relayShadow.js relayText`) is tested equal to the app's.
- **Status messages:** heartbeats and hellos are deleted after 3 days (`rr:sent:st`); a webhook secret set to `off`
  turns a channel off. The heartbeat adds `off` (the notifier is not on) next to `paused`, still `v: 1`. The PC's
  "Reddit relay paused" note names the setting as built ("Pause new relays", §1.3), or says the notifier is not in
  "Notifiers on". A replaced webhook's id goes into the PC's `relay.webhookId`; for 14 days the PC then reads and
  deletes what the old webhook left (the app can't delete those any more).
- **Setup order:** the PC's bot and `relay` block (shadow) first, then `relay-settings.mjs` on the Windows PC and its
  two settings pasted, then `notifiers` = perfume only, then `--check-relay`. Sneaker and protein, and their webhooks,
  wait for their PC readers (§2.13). After 14 days `testWebhookUrl` goes `off` and
  `#relay-test` is deleted.
- **Playtest P3 and P16:** read `#relay-test` with the perfume notifier's `--check-relay test <channel id> <webhook
  id>`, which checks `n: "test"` messages and their files (on the Mac: it saves and deletes nothing).
- **Privacy page:** what the app reads (the crosspost original, the test sub), how long Redis, logs and Discord keep
  things, the deletion delay (within hours in a post's first day, within about a month later), what status messages
  carry, and Google's wording for a future Gemini step.
- **Check after the third review (28 Oct 2026):**
  - The zero-width scrub is one pass over each run (`dropZeroWidth` in `src/core/text.js`) with the old regex's
    output; the regex was quadratic on a long run (a 40,000-character body took about 3 s). The perfume shadow copy
    (`relayShadow.js`) changes the same way.
  - Setup order (§4.1 to §4.3, the README): Discord (webhooks for `#perfume` and `#relay-test` only; @everyone
    without access at server level, so the bot reads `#relay-test` too), the upload (it writes `dev.subreddit` into
    devvit.json itself: check it), the private-sub check, the pages, the secrets, the Install Settings, the playtest.
  - Playtest: P2 reads the logs' `config.problems` and the hellos (`settings list` shows names only); P3, P11 and P16
    use the perfume notifier's `--check-relay test` on the Mac; P4, P7 and P20 come after perfume is on; P8 uses the
    test sub; P10 makes a new webhook (a deleted one can't be restored); P19 names the sub.
  - The moderators' agreement: the perfume subs' on 27 Oct 2026; the sneaker and protein subs' moderators agreed too
    (confirmed 28 Oct 2026). The sneaker and protein subs are read only once those notifiers are switched on.
  - The Privacy Policy says that images and titles, once they may go to Gemini at all, go with `store: false` (§3.6
    item 4); until then no forwarded content goes to an AI service. For Google's abuse log it says 55 days and no
    switch, as Google's terms, usage policies and ZDR page do; RC40's "can't be deleted on request" is gone, since
    none of those pages says it.
  - devvit.json's five secrets carry no `defaultValue`: @devvit/shared-types 0.15.1's schema refuses one on a secret,
    and the CLI parses devvit.json before every project command, so `npx devvit upload` stopped there
    (`test/devvit-config.test.js` runs the CLI's own parser once `npm install` has run). The `notifiers` help text
    says sneaker and protein go on once their PC readers exist.
