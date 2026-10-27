# MAL Actions

A small personal addon that lets me update my MyAnimeList entries from inside Nuvio, the app I watch
anime in. I can set a status, mark episodes watched, rate a show and read the episode's discussion
thread without leaving the episode I'm on.

> The source code is private. This repo is a write-up of what it does and how it's built.

<img src="assets/nuvio-menu.png" alt="The MAL Actions group in Nuvio's source list for Cyberpunk: Edgerunners episode 10: a status line reading 'Currently: Completed · Ep 0/10', an episode discussion row, 'Mark watched up to Ep 10 (finishes it)', the other statuses, and rating rows" width="720">

## Why I built it

MyAnimeList is where I keep track of everything I watch, and Nuvio is where I actually watch it. Keeping
the two in sync meant leaving the app, finding the show on MAL, and updating it by hand. I usually
forgot, and my list was full of shows stuck at "Watching · Ep 0" that I'd finished months ago.

Nuvio speaks the Stremio addon protocol, and that protocol has no way for an addon to add a button.
Addons can only answer a few kinds of questions, like "what streams exist for this episode?" So this
addon answers that question, but with a menu instead of streams.

## What it does

When I open an episode, a **MAL Actions** group shows up next to the normal sources, like in the
screenshot above.

- **The status line** reads my list live from MAL.
- **The menu only shows rows that would change something.** The current status is hidden, episode
  progress only moves forward, and the rating rows (★10 down to ★6) only appear once a show is
  finished or dropped. That keeps it to about six rows instead of twelve.
- **Clicking an action** opens a small confirmation page ("Done ✓, Gachiakuta set to Completed").
  MAL is already updated by the time the page loads.
- **The 💬 row** opens that episode's discussion thread from MAL's forum, on a page built for
  reading it: spoilers blurred until tapped, long posts collapsed, oldest or newest first.

<img src="assets/done-page.png" alt="Confirmation page: 'Done ✓ Cyberpunk: Edgerunners 2 watched up to ep 10 of 10 · Completed.' with a link to MyAnimeList" width="520">

<p>
  <img src="assets/discussion-page.jpg" alt="Episode discussion page on a phone: header with the anime title, oldest/newest toggle, and a list of comments" width="300">
  <img src="assets/discussion-spoiler.png" alt="A comment whose images are hidden behind a blurred box labelled 'Spoiler · tap to reveal'" width="300">
</p>

## How it works

```mermaid
sequenceDiagram
    participant N as Nuvio
    participant A as Addon (Vercel)
    participant R as Upstash Redis
    participant M as MyAnimeList API
    participant B as Browser

    N->>A: streams for kitsu:1:5 / tt…:3:15 / mal:…
    A->>R: cached ID mapping + my list status
    A->>M: (on a cache miss) anime + my_list_status
    A-->>N: menu rows, each with a signed externalUrl
    Note over N: I click "🏁 Set: Completed"
    N->>B: open externalUrl
    B->>A: /action?mal=…&op=completed&sig=…
    A->>M: PATCH my_list_status
    A->>R: clear cached status
    A-->>B: "Done ✓" page
```

All of it is one serverless function on Vercel. Upstash Redis stores the MAL login and a few caches,
and a daily cron job keeps the login from expiring.

## The interesting problems

### Using the source list as a menu

A Stremio stream can carry an `externalUrl` instead of a video, and the app opens it in the browser.
Each menu row is a stream like that, pointing at my server. I didn't want to guess how Nuvio would
treat these rows, so I read Nuvio Desktop's source code. It showed me which fields become the label
and the subtitle, that a row with no link is silently dropped (so even the status line needs one), and
most importantly that auto-play never picks an `externalUrl`-only stream. Without that last part,
auto-play could have changed my list on its own.

I also considered sending the action as a fake "video" so no browser tab would open. Reading the
auto-play code ruled that out for the same reason: a video row can be auto-played.

### One show on IMDb, eight entries on MAL

Some apps identify episodes by IMDb ID, season and episode: `tt2560140:3:15`. MAL, on the other hand,
splits seasons into parts. Attack on Titan's season 3 is two separate MAL entries. The addon looks up
every MAL entry linked to the IMDb ID, keeps the ones for that season, and counts episodes across
the parts. So S3E15 becomes episode 3 of "Season 3 Part 2".

### MAL's OAuth quirks

MAL uses OAuth2 with PKCE, but only the `plain` method, where the challenge is just the verifier. I
checked that in the docs instead of assuming the usual `S256`. Refresh tokens last about a month, so a
quiet month would have logged me out. A daily cron job refreshes them, at most once every three days.

### The hole I found in my own design

Every action link is signed with an HMAC, so nobody can make one up. Halfway through, I realised the
signature didn't help much: the stream endpoint is public, and its whole job is to hand out signed
links. Anyone who knew the domain could install the addon and change my list.

The fix was a secret install path. The manifest and menu now only answer at
`/<key>/manifest.json`, where the key is derived from the same secret as the signatures. Other paths
return 404.

### Episode discussions from an undocumented API

MAL's forum endpoints are barely documented, so I called them first and wrote the types from the
real responses. What I found:

- **Search is loose.** A search for "Gachiakuta Episode 24 Discussion" returns every Gachiakuta
  thread. The addon matches the exact title `"{MAL title} Episode {n} Discussion"`. Because every
  season has its own MAL title, that also picks the right season.
- **The first page is slow.** MAL takes about 5 seconds to serve the first page of a thread. The menu
  starts finding the thread and loading that page in the background (with `waitUntil`), so it's
  usually already cached when I tap the row. The comment count appears in the menu after that.
- **Posts are BBCode inside HTML.** The renderer reduces every post to plain text and escapes it
  first. It then turns back on only the BBCode it recognises: quotes, spoilers, images, links and
  formatting. Nothing from a post is rendered as-is.
- **There are no likes.** MAL's API doesn't return likes or scores for forum posts.

### Staying under two seconds

| Data | Cached for |
| --- | --- |
| Kitsu / IMDb → MAL ID mapping | 7 days (misses for 1 day) |
| My list status for a show | 2 minutes, cleared after every action |
| Forum thread ID | Permanently (the comment count refreshes every 6 hours) |
| A page of comments | 10 minutes |

With warm caches the menu answers in under a second.

## Stack

TypeScript · Vercel Functions · Upstash Redis · MyAnimeList API v2 ·
[ARM](https://arm.haglund.dev) for anime ID mapping

## Limits

- **It's built for one user (me).** It links one MAL account.
- **No Reddit tab.** I wanted r/anime's episode threads alongside MAL's, but Reddit now requires a
  manually approved API app, so that waits.
- **No likes on the discussion page.** MAL's API doesn't provide them.
