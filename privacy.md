---
title: deal-feed-relay — Privacy Policy
---

# deal-feed-relay — Privacy Policy

Last updated 29 October 2026.

<!-- The public copy of this page is in the reddit-relay-site repo (GitHub Pages): change both together. Keep this
page in step with the app and the notifiers: the Devvit Rules require it to describe completely and accurately what
happens to the data. -->

**Who.** u/CipherFaze (the developer). Contact: modmail to r/deal_feed_relay_dev.

**What the app reads.** Public posts and comments in the subreddits listed in the [Terms](terms), and, while the
developer tests the app, the posts and comments of r/deal_feed_relay_dev, the developer's own private subreddit the app is
installed in. For a crosspost without an image of its own, it also reads the original post, wherever it was posted (a
user's profile too), to take its first image link. Nothing private: no messages, no votes, nothing behind a login.

**What it forwards, per post or comment.**
- the post or comment id, the subreddit, the time it was posted and the time the app read it; for a comment, the id of
  the post and of the comment it answers;
- the title, and the text up to 8,000 characters (4,000 for comments), without Reddit usernames: u/… mentions and a
  profile's own r/u_… subreddit are removed, and in profile links and message/compose?to=… links to a user the name is
  replaced by [user], also when the author's formatting escapes the name or splits it with an invisible character
  (links that write to a subreddit's moderators stay as they are);
- the post flair text and its template id, the link (a username in it replaced by [user]), the first image link (for a
  crosspost without one, the original post's), the number of images and the crosspost id;
- simple flags: edited, locked, NSFW, spoiler, pinned, distinguished, read late, and the moderators' approval time when
  Reddit gives one; whether the author's account still exists or is a bot, AutoModerator or a mod team;
- whether a comment is by the post's author (yes or no);
- the ids of the same author's other posts among the posts just read, from the previous 48 hours (to spot reposts);
- for the perfume list only: the id of a seller in the developer's public seller registry, when the author is one of
  that registry's sellers.
- Status messages carry no Reddit content: times, counts, subreddit names, error kinds, whether relaying is paused or
  off, the app's version and installation id, the notifier's settings and a keyed checksum of the seller list. They stay in Discord until a
  computer reads them, or 3 days at the most.

**Usernames and user flair** are read in memory only, to compare a one-way hash of the username with the seller
registry and to read a seller flair, then dropped. They are never stored or forwarded, and neither is the hash.
Never collected at all: account ids beyond that moment, profile details, avatars, karma, votes, private messages.

**Where data goes, and for how long.**
- **Reddit's app storage (Redis):** the send queue: posts and comments at most 48 hours; deletion notices (ids only)
  wait until they are sent. Post and comment ids for de-duplication, comment counts and the day's counters, at most
  31 days. Ids and check dates for the deletion checks below, with no text, until a day after the checks end: 400 days
  after posting plus a day (perfume and sneaker posts), 30 days plus a day (sneaker comments) or 14 days plus a day
  (protein); with them the ids of the Discord messages the app sent, until those are deleted.
- **Reddit's app logs:** job counts, item ids, status codes and subreddit names, never a text or a username; Devvit
  keeps them for up to 7 days (or its last 5,000 events).
- **Discord:** the developer's private server, one channel per notifier; a deal notifier is switched on only once one of
  the developer's computers reads its channel. The developer's computers delete each message after reading it
  (normally within minutes), and the app deletes its copies after 14 days at the latest (status messages after 3).
  Deletion notices (ids only) stay until a computer has read them.
- **The developer's computers:** prices, product names and ids derived from posts, for up to 400 days (comments of the
  sneaker list: 30 days). Texts are kept only while they are being processed.
- **Notifications:** short alerts (offer title, price, seller and a link) go to the developer's own Telegram bot and
  ntfy topic, and a daily summary lists the best prices of the perfumes the developer watches. When the Reddit content
  is deleted, the developer's computer deletes every alert quoting it, the daily summary included (Telegram allows this
  within 48 hours; older alerts are edited to remove the content), and deletes the ntfy message. Some apps (for
  example ntfy on iPhone) may keep a copy already shown on the phone.
- **AI services:** none. No forwarded content is sent to an AI service. If that ever changes (for example images of
  sale posts and their titles read by Google Gemini on a paid plan, which Google does not use to improve its
  products), it will happen only after Reddit's Developer Platform team has agreed in writing, and this page will say
  so first, dated. Images and titles would then be sent with Gemini's request storage turned off (`store: false`, which
  the developer's notifier sends with every Gemini request), so they would not be kept in the project's Gemini API logs.
  For paid services Google still logs prompts and responses for a limited period of time, solely for detecting and
  preventing violations of the Prohibited Use Policy to maintain the safety and security of the Services, and any
  required legal or regulatory disclosures: 55 days by its usage policies, a log the developer can't switch off or
  shorten.

**Deletions.** The app re-checks forwarded posts after 1 hour, 6 hours, 1, 3, 7, 14 and 28 days, then every 30 days
while records derived from them are kept, and a post's forwarded comments 1 hour, 1, 7 and 28 days after the newest of
them. Content that has been deleted or removed, or whose author deleted the account, is removed from Discord, and the
developer's computers delete every record derived from it. A subreddit that becomes unreadable (private or banned) for
7 days is treated the same way. Checks happen by polling on this schedule, never instantly: a deletion is honoured
within hours in a post's first day, within days in its first weeks and within about a month after that (a comment's
within about three weeks); a check that fails is tried again, at the latest 3 days later. If the app is ever
uninstalled and installed again, posts forwarded before that are no longer re-checked; their derived records are kept
until their time above runs out.

**Sharing.** None beyond the services above, which deliver the developer's own notifications. No sale, no profiling,
no tracking, no advertising, no training of AI models.

**Security.** Webhook addresses and the signing key are kept in Reddit's encrypted app secrets and in the developer's
private configuration. Every forwarded message is signed, and the computers ignore unsigned ones.

**Changes.** Updates are dated on this page.
