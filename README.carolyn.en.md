# YouTube Digest — Carolyn's build

> Branch `carolyn` · forked from [zarazhangrui/youtube-digest](https://github.com/zarazhangrui/youtube-digest) v1.2.0 · MIT
> `main` is an untouched upstream mirror. Every change lives on `carolyn`.

English | [简体中文](README.carolyn.md)

![Executive Summary demo](YouTube%20Digest%20demo%20carolyn.png)

## Why this build exists

Upstream is good at watching a video: transcript, bilingual translation, chapters and notes in one side panel. It assumes you already decided to watch.

This build works on the step before that: **whether to watch at all, and how deep to go.**

Attention is a budget, for an ADHD brain more than anyone (the rules come from [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd)). The expensive part of a long video is not its length, it is the judgment call. Open a 40-minute video, realize at minute 10 that you knew all of it, and those 10 minutes do not come back. Upstream's Overview is Chapters plus Key Quotes, both laid out along the timeline, so making that call still means scanning the chapters top to bottom. Pay first, inspect later.

So the Overview now opens with an Executive Summary. Four lines, two questions:

1. **What does this video argue?** The claim, not the topic. "How to hire" is a topic; "hiring should run like executive search, not a funnel" is a claim.
2. **What do I walk away with?** Something you can do or decide differently. Every line must carry a number, a named method, or an assumption the video overturns. Nothing qualifies, nothing gets written.

Read those four lines, then choose: close the tab, spend 3 minutes on the Summary, or watch the whole thing. `verdict` states the suggested minutes and one timestamp worth jumping to.

The anti-filler prompt rules, the navigation, and the cache fix below all serve those two questions.

## What changed

### 1. Executive Summary (new)

A block at the top of Overview with four fixed fields:

| Field | Limit | What it answers |
|---|---|---|
| `oneLine` | 1 sentence, ≤25 words | Question 1, "what does it argue": the claim, not the topic |
| `framework` | 2–4 items, ≤12 words each | The author's scaffolding: mental models / stages / axes of comparison, not a play-by-play retelling |
| `takeaway` | 1–3 items, ≤15 words each | Question 2, "what do I walk away with": what you can do or decide differently |
| `verdict` | ≤25 words | Effort allocation: how many minutes, which timestamp to jump to |

**Anti-padding rule for takeaways**: each item must carry one of "a number / a named method / an overturned assumption".
An item with none of the three is filler. **Writing fewer lines beats making them up.** If the video genuinely offers nothing actionable, one honest line saying so.

### 2. Anti-filler prompt rules

Hard constraints added to `prompts/analysis.md`:

- **No throat-clearing openers**: `This video discusses`, `In this video the speaker`, `The video covers`
- **No filler connectors**: `It's worth noting that`, `Overall`, `In today's world`, `At the end of the day`
- **No empty intensifiers**: `very`, `really`, `incredibly`, `a lot of`
- **No vague praise**: `valuable insights`, `great advice`, `deep dive`, `practical tips`
- **No restating the title**, and no hedging when the author gives an exact number

**Pre-send check** (taken straight from [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd)):
delete adverbs that carry no information, delete idioms and metaphors, delete preview-style openers.

**Final test**: reading only `oneLine` and `verdict`, can you tell (a) what the video argues and (b) whether it deserves your time? If either answer is no, rewrite both.

**Chapter summaries changed tone too**: say what a section concluded, never what it "discussed".

`verdict` must give **specific minutes** and **one specific timestamp** (rules 6 and 3 of i-have-adhd):
not "worth a skim" but "6-minute skim; jump to 12:30 for the sourcing method".

### 3. Overview navigation

Chapters get long, so:

- **Sticky jump nav at the top**: `Summary / Chapters / Quotes`, one click smooth-scrolls there.
  If the target section is collapsed it **expands automatically**; jumping to an empty header looks like a dead click otherwise.
- **Chapters / Key Quotes collapse on click**, with an arrow indicator. Collapsed state is stored in `chrome.storage.local` and survives reopening.
- Executive Summary does not collapse. It is four lines, and the four lines you should read first.

### 4. Versioned analysis cache (a bug I hit)

**Symptom**: change the Overview structure, reload the extension, the interface does not move.

**Root cause**: `analysis` is cached in `chrome.storage.local`. Reloading swaps the code, the cache stays. The new render function draws from the **old structure**, finds no `execSummary` field, treats the block as empty and hides it. It looks exactly like "the update did nothing".

**Fix**:

```js
const ANALYSIS_SCHEMA_VERSION = 2;   // bump on every Overview structure change
```

`saveToCache` writes it; `loadFromCache` nulls `analysis` on a version mismatch.

**The trade-off that matters: only analysis is dropped, transcript and translation caches stay.**
Transcripts cost Supadata credits (100 per month on the free tier, 1 credit per video); recomputing analysis with DeepSeek costs a fraction of a cent. Dropping the expensive one to protect the cheap one would be backwards.

> ⚠️ **Next time the Overview structure changes, bump the version number**, or this bug returns.

## Files touched

| File | Change |
|---|---|
| `prompts/analysis.md` | Executive Summary output spec + word limits + banned-word list + pre-send check |
| `background.js` | `execSummary` added to the whitelist, rebuilt via `safeString`/`safeList`; schema version added |
| `sidepanel.html` | Executive Summary container + jump nav + collapsible headers |
| `sidepanel.js` | Executive Summary rendering, jump/collapse logic, cache version check |
| `sidepanel.css` | Matching styles, reusing upstream's existing CSS variables |

5 files, 376 added lines. **Every changed block carries a `/* CAROLYN */` comment**; `grep -rn "CAROLYN"` finds them all.

## Language: source text stays English

The Executive Summary is not generated in Chinese directly. It goes through the same translation layer as Chapters and Quotes, so the three buttons at the top (Original / 中文 / 双语) apply to it equally and share one cache. Its segments also sit **first in the translation queue**: during progressive translation the summary arrives before the chapters.

## Install

```bash
git clone https://github.com/ddcarolyn/youtube-digest.git
cd youtube-digest && git checkout carolyn
```

Then `chrome://extensions` → enable Developer mode → Load unpacked → pick this folder.
Getting and entering API keys is covered in the upstream [README.md](README.md).

> API keys live in `chrome.storage.local`. **They do not sync across Chrome profiles or devices**; every device and profile needs its own entry.

## After changing code

1. `chrome://extensions` → YouTube Digest → **Reload**
2. Back in the YouTube tab, refresh the page (without it the side panel keeps running old code)
3. Open Overview

## Staying in sync with upstream

```bash
git checkout main && git pull origin main   # origin = upstream
git checkout carolyn && git rebase main
npm test && npm run check
```

Conflicts, if any, land in `prompts/analysis.md` and one return statement in `background.js`.

Remote layout:

```
origin = zarazhangrui/youtube-digest   # upstream, pull only
mine   = ddcarolyn/youtube-digest      # push here
```

## Tests

```bash
npm test     # upstream's 62 tests, still 62/62 after the changes
npm run check # release file whitelist check
```

Upstream tests do not assert on the analysis return shape, so adding `execSummary` breaks none of them.

## Credits

- Upstream: [zarazhangrui/youtube-digest](https://github.com/zarazhangrui/youtube-digest) (MIT)
- Design starting point and anti-filler rules: [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd) (MIT)
