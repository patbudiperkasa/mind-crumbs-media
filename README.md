# Mind Crumbs: media + how to build your own animated explainer channel

This repo does two jobs:

1. **It hosts the finished Mind Crumbs videos** (`videos/`) at public links, so the scheduling tool (Buffer) can pick them up and post them to YouTube Shorts and TikTok automatically.
2. **It documents the whole system**, step by step, so you can build your own channel the same way: Claude researches, writes, animates, voices and renders each video, then schedules it, and the human only reviews and replies to comments.

Mind Crumbs (@themindcrumb) posts one 40–50 second animated science explainer every day: one curious question per video ("Why do onions make you cry?", "Could you hear anything on Mars?"), English voiceover, original drawings and music.

> Use this as a method, not a template. Your niche, name, look, mascot and voice should be your own.

---

## Contents

1. [How the whole system fits together](#1-how-the-whole-system-fits-together)
2. [What you need and what it costs](#2-what-you-need-and-what-it-costs)
3. [Step 1: Decide your channel](#3-step-1-decide-your-channel)
4. [Step 2: Create the accounts (safely)](#4-step-2-create-the-accounts-safely)
5. [Step 3: Build the video engine with Claude](#5-step-3-build-the-video-engine-with-claude)
6. [Step 4: The episode formula and rules](#6-step-4-the-episode-formula-and-rules)
7. [Step 5: Make the look rich, not "stick figures"](#7-step-5-make-the-look-rich-not-stick-figures)
8. [Step 6: Automate the batches](#8-step-6-automate-the-batches)
9. [Step 7: Automate the posting (Buffer + this repo)](#9-step-7-automate-the-posting-buffer--this-repo)
10. [Step 8: A dashboard to run it all (optional)](#10-step-8-a-dashboard-to-run-it-all-optional)
11. [Growing the channel](#11-growing-the-channel)
12. [Lessons learned the hard way](#12-lessons-learned-the-hard-way)
13. [Troubleshooting](#13-troubleshooting)
14. [This repo's layout and rules](#14-this-repos-layout-and-rules)
15. [Copy-paste prompts](#15-copy-paste-prompts)

---

## 1. How the whole system fits together

```
            ┌───────────────────────────── twice a week ─────────────────────────────┐
            │                                                                        │
  topics.json ──► Claude scheduled batch ──► research ──► script ──► scenes (HTML/SVG)  │
  (the queue)        (cloud session)                                   │               │
                                                                        ▼               │
                                      voice (Kokoro TTS) ──► frames (Playwright) ──► ffmpeg
                                                                        │
                                       video.mp4 + cover.png + captions ▼
                                                                        │
                    ┌───────────────────────────────┬──────────────────┴─────────────┐
                    ▼                               ▼                                ▼
       private production repo          public media repo (this one)       dashboard (HQ)
       (code, episodes, queue)          videos/NNN-slug.mp4                files, captions,
                                                    │                      ticks, numbers
                                                    ▼
                                         Buffer (via Claude connector)
                                          ├─► YouTube Shorts, 7–8 PM
                                          └─► TikTok, 7–8 PM
```

The human's job: glance at each batch, reply to comments in the first hour after a post, and log the numbers once a week.

---

## 2. What you need and what it costs

| Thing | What for | Cost |
|---|---|---|
| Claude (paid plan with scheduled tasks, connectors and a cloud workspace) | Research, writing, animation, rendering, scheduling | Your Claude plan |
| GitHub account | One **private** repo for the engine and episodes, one **public** repo for finished videos | Free |
| YouTube channel + TikTok account | Where the videos go | Free |
| Buffer account, connected to Claude | Scheduling posts to YouTube and TikTok | Free plan works (see limits below); paid removes the "#PoweredByBuffer" tag |
| Rendering tools: Python, Node, Playwright, ffmpeg, Kokoro TTS | Making the videos | Free, installed in Claude's workspace |

Buffer free plan limits that matter here: **3 channels**, **10 queued posts per channel at once**, and it adds **#PoweredByBuffer** to captions. Paid Essentials is about $6/month per channel.

---

## 3. Step 1: Decide your channel

Decide these once and never change them. Consistency is what makes people recognise you in a fast-scrolling feed.

| Decision | Guidance |
|---|---|
| Niche | Something people genuinely wonder about. One question per video works best: "Why do…?", "What happens if…?", "Where did … come from?" |
| Name and handle | Short, memorable, and the **same handle free on every platform**. |
| Content pillars | 4–6 themes you rotate (Mind Crumbs uses: brain/mind, body, sleep, space, animals, everyday science, history and words). Rotating stops the feed getting samey. |
| Brand look | One background mood, 3 accent colours, 1 font family, 1 logo mark. |
| Mascot | One original character that plays every role (see section 7). Draw your own; never borrow a known character. |
| Voice | One narrator voice and one speed for every episode. |
| Language and market | Make videos in your audience's language. TikTok locks your account region at signup and shows new accounts to local viewers first, so sign up in the country you're targeting. YouTube lets you set the channel country in settings. |

---

## 4. Step 2: Create the accounts (safely)

New accounts are watched closely by spam filters. Create them like a real person would:

1. **Profile first:** profile picture (your logo), display name, one-line bio, and on YouTube a 2560×1440 banner (keep text inside the middle 1546×423 safe area).
2. **Warm up before posting:** spend 15–20 minutes a day for 2–3 days using each app normally: watch, like, comment, follow a few creators in your niche.
3. **Start slow:** one video on day one, then one a day. Do not upload a week of content in the first 48 hours.
4. **One device, one location** for the first weeks. Logins from several networks or countries close together look like a hijacked account.
5. **TikTok:** a Personal account is fine; Buffer can post to it. Allow Duet, Stitch and Downloads.
6. **Label AI content:** turn on "AI-generated content" on TikTok, and use YouTube's altered-content setting honestly (animated explainers with a synthetic narrator are usually "No" for realistic altered content; Buffer exposes an `isAiGenerated` flag for both).

See section 12 for why this matters.

---

## 5. Step 3: Build the video engine with Claude

Create a **private** GitHub repo (for example `my-channel`) and give Claude access to it. Then ask Claude to build the engine (prompt in section 15). The finished layout looks like this:

```
engine/
  base.css     brand look: background, colours, fonts, logo bar, topic tag, progress bar
  base.js      builds the brand chrome, runs scenes on a timeline, exposes window.render(t)
  rich.css/js  the illustration layer: mascot builder, panels, camera, bokeh (section 7)
  vo.py        voiceover per scene with Kokoro TTS, writes timing.json
  render.js    Playwright screenshots every frame (1080x1920, 30fps) -> ffmpeg clip per scene
  audio.py     original music bed + sound effects + voice, with ducking
  cover.js     renders cover.html to a PNG
  build.sh     one command: voice -> frames -> audio -> mux + loudness -> cover -> caption
  setup.sh     installs the voice model and checks tools
episodes/
  002-deja-vu/            the reference episode every new one copies
    episode.json          scenes, voice lines, sound cues, sources, captions
    episode.html          the scenes, drawn in SVG + JS
    cover.html            the thumbnail
topics.json               the queue: number, date, title, status (todo/ready/posted), backlog
README.md                 the runbook the scheduled Claude follows every batch
out/                      finished files (not committed)
```

**Why HTML and SVG instead of a video editor?** Every frame is code. Claude can write it, fix it, and re-render with one command, and the style stays identical across hundreds of episodes.

**How timing works**

- `episode.json` has one entry per scene, each with a `vo` (voice) line and sound `cues`.
- `vo.py` makes each scene's voice clip and sets the scene's length to the voice length plus a short lead-in and tail, rounded to whole frames. The picture always waits for the narrator.
- `episode.html` defines `window.SCENES = { s1: (t, dur) => {...}, ... }` where `t` is seconds since the scene started. The renderer calls `render(t)` for every frame.
- Small animation helpers (`pop`, `rise`, `floatIn`, `slideIn`, `count`, `type`, easing functions) keep motion consistent between episodes.

**Render settings that work:** H.264, CRF 18, 30fps, `-movflags +faststart`; audio AAC 192k at 48 kHz, loudness-normalised to about −14 LUFS so you're not quieter than the videos around you.

**Before every full render**, render one still per shot and check:

- All text sits between y≈420 and y≈1600 (the platforms' buttons and captions cover the rest).
- No overlaps, no cut-off words, no empty dead areas.
- Readable on a phone at arm's length: headlines 84px+, body 44px+.
- The cover's middle 1080×1440 area works on its own (profile grids crop to it).

---

## 6. Step 4: The episode formula and rules

**The six beats (40–50 seconds total)**

1. **Hook (0–3 s):** a relatable moment or surprising statement, with something moving in the first second. No logo intro.
2. **Name it:** the term, big, with a one-line definition.
3. **Surprise number:** one sourced statistic, shown big with an animated counter.
4. **Why it happens:** one simple visual of the mechanism.
5. **Twist or proof:** a study, a weird detail, or "scientists still argue…". Show the source small on screen.
6. **Question + teaser:** a comment question people can answer in a few words, and "Next: …" teasing the next episode.

**Rules that keep you credible and safe**

- Every number and claim comes from a page Claude actually opened. Prefer studies, universities, encyclopedias and trusted science or health sites. Say "about", "one theory", "studies suggest" when science isn't settled.
- Original drawings, music and voice only. No photos of real people, no movie or TV clips, no brand logos, no copied characters or artwork.
- Never quote song lyrics or poems, and never recreate a real melody, even in an episode about music.
- Health topics stay general. No medical advice; add "check with a doctor" where it matters. Never encourage dangerous stunts (the "no sleep for days" episode says "please don't try this").
- No politics, religion or real private people.
- Text on screen: one idea per line, 12 words max.

**Caption format**

```
Hook question with one emoji 🧅

2–3 plain sentences with the facts.

Comment question ending with 👇

Sources: Publication (year); University; ...

Follow for a new crumb every day 🧠

#specific #specific #specific #specific #yourhandle
```

YouTube gets the same text as the description (with "Subscribe" instead of "Follow"), plus a title under 100 characters ending in `#shorts`.

---

## 7. Step 5: Make the look rich, not "stick figures"

Flat shapes feel thin next to big creators. The rich layer keeps the brand identical and makes every scene feel illustrated and alive. It has six parts:

1. **Shared paint:** one block of SVG `<defs>` in every page with gradients (lit-to-shaded) for the mascot, panels and props, a soft drop-shadow filter and a glow filter. Top-left to bottom-right shading gives everything volume at almost no cost.
2. **A mascot builder:** `mascot({mouth, cap, glasses, cane})` returns SVG for your character standing at (0,0). Arms are groups that remember their pivot, so `pose(el, {armL, armR})` just rotates them; `look(el, dx, dy)` moves the pupils; `blink(el, t)` closes the eyes for 0.15 s every few seconds.
3. **Framed panels:** each scene's picture sits in a rounded "window" with its own sky, hills, room or desk, on top of the constant brand background.
4. **A camera:** wrap each scene in a `.cam` layer that settles in from a 3% zoom, then pushes in about 5% and drifts about 14 px sideways, alternating direction per scene.
5. **Sub-shots:** cut inside a scene when the narrator starts a new sentence, so something changes every 1–2 seconds. Keep secondary motion going: blinking, ticking counters, swinging, drifting particles. This matters more for watch time than detail does.
6. **Atmosphere:** a few large blurred dots drifting upward (bokeh) and a soft vignette, both under 30% opacity.

**Mascot rules that worked for us**

- The mascot is the **only** character. It plays "you", kids (smaller, with a cap), older people (glasses and a cane), scientists, a viewer reacting.
- Use it **only when a scene needs a character** to represent someone. Scenes about objects, diagrams, numbers or places don't need it; don't add it just to fill space.

**Gotchas**

- Never draw limbs as zero-width `<line>` elements with a gradient stroke; they render invisible. Use rounded `<rect>` shapes.
- Draw arms **after** the body so raised arms aren't hidden behind it.
- Use `gradientUnits="userSpaceOnUse"` on shared gradients, or overlapping shapes show seams.
- A `filter` (blur, shadow) on a group also blurs anything you later put inside it. Keep characters outside blurred groups.

---

## 8. Step 6: Automate the batches

Create a **scheduled task** in Claude (for example Wednesday and Saturday mornings, in your time zone) whose prompt tells it to:

1. Clone the private repo and read its `README.md` runbook.
2. Run `engine/setup.sh`.
3. Take the next 3–4 `todo` topics from `topics.json` in order, check none repeats an earlier episode or post, research each one, write `episode.json`, `episode.html` and `cover.html`.
4. Render stills, check them (section 5), fix problems, then build each episode.
5. Publish the videos to the media repo and schedule them in Buffer (section 9).
6. Update the dashboard (section 10), mark the episodes `ready` in `topics.json`, keep at least 12 topics queued, commit and push.
7. Send a short summary: what was made, post dates, sources, anything that needs a human.

Keep the runbook in the private repo's README, not in the scheduled prompt. Then improving the process is just editing one file.

---

## 9. Step 7: Automate the posting (Buffer + this repo)

### Why Buffer instead of uploading straight to YouTube

YouTube's own upload API puts every video uploaded through a **new, unaudited API project** into **private, and locks it there**; lifting that needs Google's compliance audit. Buffer is already an approved app, so videos it posts go public normally. Buffer also handles TikTok, which has a similar review process for its posting API.

### Why this public repo exists

Buffer only accepts media as a **public, direct download link**. Claude's workspace can push to GitHub, so finished videos go into `videos/` here, and their raw links are handed to Buffer:

```
https://raw.githubusercontent.com/<owner>/<media-repo>/main/videos/016-grey-hair.mp4
```

Keep this repo **public** and keep **only finished videos** in it. Code, plans and anything private live in the private production repo.

### One-time setup

1. **Create the public media repo** on GitHub (empty, no README) and give your Claude session push access to it.
2. **Create a Buffer account just for the channel.** Don't mix it with client or personal accounts: the free plan only has 3 channel slots, and the batch should never be able to see other channels.
3. In Buffer, **connect your YouTube channel and your TikTok account**.
4. In Claude, **connect the Buffer connector** (Settings → Connectors → Buffer) and log in with that Buffer account.
5. Ask Claude to **list the Buffer channels** and confirm only your channel's YouTube and TikTok appear. Note the organization and channel IDs in your private runbook.
6. **Test with one video** (prompt in section 15) and check it appears scheduled in Buffer with the right title, caption and time before switching the batches over.

### What the batch does for each episode

1. Copy `out/NNN-slug.mp4` to `videos/NNN-slug.mp4` in the media repo, commit and push.
2. Check the raw link downloads (HTTP 200, `video/mp4` or `application/octet-stream`, correct size).
3. Create **two Buffer posts** with `schedulingType: automatic`, `mode: customScheduled` and `dueAt` in the account's time zone with its UTC offset (for example `2026-11-26T19:30:00+07:00`):
   - **YouTube:** video asset = raw link; `text` = YouTube description; `metadata.youtube` = `{ type: "short", title: "<title under 100 chars> #shorts", categoryId: "27" (Education), privacy: "public", madeForKids: false, notifySubscribers: true }`.
   - **TikTok:** video asset = raw link; `text` = TikTok caption; `metadata.tiktok` = `{ isAiGenerated: true }`.
4. Read each post back and confirm status `scheduled`; record the post IDs in the dashboard.
5. Respect the **10 queued posts per channel** limit on the free plan: schedule up to 10 days ahead, and let the next batch top the queue up as posts go out.
6. **Clean up:** delete videos from `videos/` about 7 days after they've been published, so the repo stays small.

### Things Buffer can't do (free plan)

- No custom thumbnail for YouTube Shorts (YouTube picks a frame).
- "#PoweredByBuffer" is added to captions.
- YouTube category can't be changed for Shorts posted through Buffer.

---

## 10. Step 8: A dashboard to run it all (optional)

A single web page (we use a Claude artifact with a small built-in database) with four views:

- **Today:** today's video with download and copy buttons, plus anything not yet ticked as posted.
- **Schedule:** the next two weeks, with a dot per platform once it's posted.
- **Library:** every episode, searchable, with covers.
- **Numbers:** views, likes, comments, shares and saves per video per platform; top videos ranked; a weekly follower log.

Each batch uploads its files and writes one record per episode, without overwriting the human's ticks and numbers.

---

## 11. Growing the channel

- **One video a day, every day,** at the same time (we use 7–8 PM local). Consistency beats volume.
- **Reply to every comment in the first hour.** Ask questions that are easy to answer in a word or two.
- **YouTube Shorts usually moves first** for brand-new animated channels. Don't judge a platform on its first week.
- **Measure weekly** (YouTube Studio → Content → Shorts; TikTok Studio → Analytics):
  - "Viewed vs swiped away" (YouTube) or early drop-off: a weak number means the first 2 seconds need work.
  - Average watch time vs length: people leaving mid-way means pacing or visuals.
  - Shares and saves: strong ones mean the topic itself works. Make more like it.
- **Double down on what works.** Our early winner was sleep paralysis; spooky-but-relatable body and sleep topics outperformed. Move similar topics earlier in the queue.
- **Test one change at a time.** Make a batch in a new style or hook, compare against the previous batch, keep whichever holds viewers longer.

---

## 12. Lessons learned the hard way

- **Our first Instagram account was permanently disabled within a week**, by automated systems, with no specific reason given. The likely triggers were all "this looks like a bot" signals: posting six videos within the first days of a brand-new account, near-identical caption patterns, logins from the phone app and the web on different networks, and no normal activity (no scrolling, following or commenting). The content itself was not the problem: the same videos did thousands of views on YouTube. Follow section 4. Don't open a replacement account after a ban; that breaks Meta's rules and can take down other accounts linked to you.
- **TikTok showed zero views for days** on a new account that only posted scheduled uploads from a computer. Check each video's privacy is "Everyone" and that it isn't marked "not eligible for For You", then warm the account up on your phone.
- **A find-and-replace on captions can break facts.** We once changed "every week" to "every day" in a follow line and accidentally changed a statistic too. Edit captions per field, never with a global replace.
- **Topic repeats sneak in.** A feed post and a planned video were both "why is the sky blue". Check new topics against everything already made.
- **Keep secrets out of public repos.** This repo only ever holds finished videos and this README.

---

## 13. Troubleshooting

| Problem | Fix |
|---|---|
| Buffer rejects the video link | Open the raw link in a browser: it must download the file directly. Make sure the repo is public and the path and file name match exactly (case matters). |
| Buffer post shows "error" | Read the error on the post. Common causes: title over 100 characters, video over 3 minutes or not vertical for `type: short`, `dueAt` in the past or without a time-zone offset. |
| Buffer says the queue is full | Free plan holds 10 queued posts per channel. Wait for posts to go out, or upgrade. |
| Video uploads to YouTube as private | You're using your own YouTube API project, which is unaudited. Post through Buffer instead. |
| Characters' arms invisible | Limbs drawn as zero-width lines with gradient strokes. Use rounded rects. |
| Raised arms hidden | Draw arm groups after the body. |
| Text cut off on the phone | Keep text between y≈420 and y≈1600; render stills and check before the full build. |
| Video quieter than others in the feed | Loudness-normalise to about −14 LUFS in the final ffmpeg step. |
| Render times out | Build episodes one at a time in the background and check the log, rather than in one long foreground command. |

---

## 14. This repo's layout and rules

```
videos/
  NNN-slug.mp4    finished episode video, 1080x1920, H.264 + AAC
README.md         this guide
```

- File names: three-digit episode number, a short slug, `.mp4` (`016-grey-hair.mp4`).
- Only finished, public-ready videos. No code, captions with private notes, account names, emails or IDs.
- Videos are removed about a week after they're published.
- Everything here is original: drawings, music, voice and scripts. Please don't re-upload the videos as your own; build your own channel with the method above.

---

## 15. Copy-paste prompts

**Build the engine and a reference episode**

> I want to start a short-form channel of animated explainer videos about **[niche]**, called **[name]**. Build me an HTML/SVG + Playwright + ffmpeg rendering engine with Kokoro TTS voiceover and original numpy-generated music, in my private GitHub repo **[repo]**, following the Mind Crumbs README method. Use the rich layer: one original mascot from a reusable builder, framed illustrated panels, a slow camera push, sub-shots so something changes every 1–2 seconds. My brand: background **[mood]**, colours **[3 colours]**, font **[font]**, logo **[describe]**. Make one reference episode on **[topic]**, show me a still of every shot, and iterate with me until it's right. Then write the README runbook.

**Set up the batches**

> Create a scheduled task that runs every **[days]** at **[time, time zone]**. Each run: clone **[repo]**, follow its README, make the next 4 todo episodes, publish the videos to **[media repo]**, schedule each one on YouTube and TikTok through Buffer at **[time]** on its date, update the dashboard, mark them ready, keep 12 topics queued, push, and send me a summary.

**Test the Buffer pipeline once**

> Using the Buffer connector, list my organizations and channels and confirm only **[channel]**'s YouTube and TikTok are there. Push episode **[NNN]** to the media repo, check its raw link downloads, then schedule it on both channels for **[date] [time]** with its YouTube title and description and its TikTok caption, marked as AI-generated on TikTok. Read the posts back and show me their status.
