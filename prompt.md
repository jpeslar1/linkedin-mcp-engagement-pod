# Engagement Pod — Safe Run

A Claude Code prompt that runs a small LinkedIn engagement pod (3-8 members) safely. Detects new posts from pod anchors, generates contextual comments in each reactor's voice via the LinkedIn MCP (Zevari), gates every write on live LinkedIn safety status, and posts a Slack summary at the end of each run + a 6 PM amplification stats DM.

The reason this prompt exists: every existing pod tool I've tried (Lempod, Engage AI, Podawaa, Aware) gets pod members flagged eventually. They write slop comments that all read the same, and they don't actually read LinkedIn's rate signals before writing. Zevari fixes both — comments are written using each reactor's voice DNA, and every write is gated on `linkedin_get_safety_status` so the prompt never grinds into a restriction.

---

## Overview

You are running the engagement pod for `[POD_NAME]`. For every poll cycle:

1. Load the pod config from `[POD_CONFIG_PATH]` (members, roles, jitter windows, per-day caps).
2. For each anchor, call Zevari `linkedin_get_user_posts` and look for posts published in the last 5-15 minutes that the pod hasn't engaged on yet.
3. For each new post, schedule comment + react actions across the reactors with jittered timing, randomized reactor order, and per-day cap enforcement.
4. Before EVERY write (`linkedin_comment_on_post`, `linkedin_react_to_post`, `linkedin_create_post`), call `linkedin_get_safety_status` and `linkedin_get_connection_limits` for the acting reactor — if anything is yellow/red, skip the action and log it.
5. Log every action (fired + skipped) to a Zevari list `pod-engagement-YYYY-MM-DD`.
6. Post a Slack summary to `[POD_CHANNEL]` at the end of each run.
7. At 6 PM local time, send each anchor a Slack DM with the day's amplification stats (comments received, reactions, reposts).

Polling cadence: every 5 minutes during posting hours (default 8 AM - 6 PM local). Wrap-up at 6 PM.

---

## POD CONFIG SCHEMA

The pod config lives at `[POD_CONFIG_PATH]` (YAML or JSON). Schema:

```yaml
pod_name: "[POD_NAME]"
timezone: "America/New_York"
posting_window:
  start: "08:00"
  end: "18:00"
members:
  - id: "member-1"
    linkedin_url: "[LINKEDIN_URL_1]"
    zevari_workspace_id: "[WORKSPACE_ID_1]"
    role: "anchor"                  # anchor = main publisher, reactor = engagement only
    daily_cap_comments: 3           # max comments/day this account will post
    daily_cap_reposts: 1            # max reposts/day
    reciprocation_rate: 0.3         # for anchors, % of pod posts they engage back on
    voice_dna_ref: "[VOICE_DNA_ID]" # optional override; default pulls from voice_dna_save history
  - id: "member-2"
    linkedin_url: "[LINKEDIN_URL_2]"
    zevari_workspace_id: "[WORKSPACE_ID_2]"
    role: "reactor"
    daily_cap_comments: 3
    daily_cap_reposts: 1
  # ... up to 8 members total. Pods of 3-5 are recommended.
jitter:
  comment_delay_min_min: 10         # min minutes after publish before commenting
  comment_delay_max_min: 15
  reactor_stagger_min_sec: 90       # min seconds between consecutive reactor comments
  reactor_stagger_max_sec: 300
  repost_delay_min_min: 30
  repost_delay_max_min: 45
react_probability: 0.5              # % of comments that also fire a react
```

Pods of 3-5 members are recommended. Anything over 8 creates engagement clusters that LinkedIn's anomaly detection picks up.

---

## ERROR & APPROVAL NOTIFICATIONS

In two situations, send a Slack notification to `[ALERT_CHANNEL]`:

1. An error occurs that stops the run — send notification, then stop.
2. Approval is required (e.g. a reactor's safety status is yellow and you want a human call on whether to skip the whole day) — send notification.

How to send (bash sandbox blocks `hooks.slack.com` — use Chrome only):

1. Call `mcp__Claude_in_Chrome__tabs_context_mcp` with `createIfEmpty: true`
2. If the tab is on a `chrome://` URL, navigate to `https://google.com` first
3. Run via `mcp__Claude_in_Chrome__javascript_tool`:

```js
fetch('[SLACK_WEBHOOK_URL]', {
  method: 'POST',
  mode: 'no-cors',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    text: "<@[YOUR_SLACK_USER_ID]> ⚠️ *Engagement Pod — Action Needed*\n\n<describe the error or what approval is needed>"
  })
}).then(r => ({status: r.status, type: r.type})).catch(e => ({error: e.message}))
```

`{status: 0, type: "opaque"}` = success.

---

## CONNECTORS

- **Zevari (LinkedIn MCP):** `mcp__[ZEVARI_MCP_ID]` — post detection, comment generation, safe writes, safety + rate-limit gating, audit list
- **Chrome:** `mcp__Claude_in_Chrome__` — Slack webhook + DM
- **WebSearch:** optional — for current-events color in a reactor's comment
- **Local file:** `[POD_CONFIG_PATH]` — the pod config

See `connectors.md` for setup links.

---

## STEP 0 — CONFIRM ZEVARI CONTEXT

Call `zevari_context_get` to confirm the active organization and LinkedIn account. The pod runs across multiple member accounts via their `zevari_workspace_id` — each reactor's write needs to be made from their own workspace context.

Before any write for a reactor, switch context to that reactor's workspace if needed (or run in multi-tenant mode if your Zevari setup supports it).

---

## STEP 1 — LOAD POD CONFIG + DETERMINE WHAT TO DO

Read `[POD_CONFIG_PATH]` (YAML or JSON). Validate:

- Every member has `linkedin_url`, `zevari_workspace_id`, `role`
- At least one member is `role: anchor`
- Pod size is between 3 and 8 (warn if more — Slack note, not a hard stop)
- Time is within `posting_window` (otherwise skip to wrap-up if past `end`, or exit if before `start`)

Load today's action ledger from the Zevari list `pod-engagement-YYYY-MM-DD` via `targets_get_pipeline` (or whatever list-fetch endpoint your Zevari instance exposes). This is the source of truth for what's already fired today — used for per-day cap enforcement.

---

## STEP 2 — DETECT NEW ANCHOR POSTS

For each member with `role: anchor`:

1. Call Zevari `linkedin_get_user_posts` with the anchor's `linkedin_url`. Limit results to the last 24h.
2. For each returned post, check whether the pod has already engaged on it (lookup in today's Zevari list ledger via the post URN).
3. If the post is < 5 minutes old → too fresh, wait for next poll cycle (avoids commenting before the post is actually out).
4. If the post is 5-15 minutes old AND no pod actions logged yet → this is a candidate. Schedule reactor actions.
5. If the post is > 15 minutes old AND no pod actions logged → catch-up case. Schedule with a shorter jitter (immediate comments — but still jittered between reactors).

Also: for each anchor, check the OTHER anchors' recent posts. With `reciprocation_rate: 0.3`, the anchor engages on ~30% of the other anchors' posts (sampled at random). Don't reciprocate on every one — that's a pattern.

---

## STEP 3 — SCHEDULE REACTOR ACTIONS PER POST

For each candidate post by an anchor:

1. Shuffle the list of reactors (random order — never the same reactor first every time).
2. For each reactor (in shuffled order):
   - Check `daily_cap_comments` — if they've already commented `cap` times today (per the ledger), skip this reactor for this post.
   - Pick a jittered timestamp: `now + uniform(reactor_stagger_min_sec, reactor_stagger_max_sec)` from the previous reactor's action time (so reactors don't stack within 9 minutes of each other — they spread across the comment window).
   - At that timestamp, run STEP 4 (safety gate) then STEP 5 (write the comment).
3. At the repost window (30-45 min after publish), pick 1-2 reactors at random (under their `daily_cap_reposts`) to repost. Same safety gate → then `linkedin_create_post` with a short take of their own.

For long polls / batched scheduling: if the prompt is running synchronously (not via a scheduler), use `time.sleep()` between actions with the jittered delays. Otherwise enqueue to your scheduler of choice.

---

## STEP 4 — SAFETY GATE (run before EVERY write)

For the acting reactor, call:

- `linkedin_get_safety_status` — returns the current rate-limit / restriction status for the account
- `linkedin_get_connection_limits` — returns the connection request limits + any warnings

Decision:

- **Status green, no warnings** → proceed to write.
- **Status yellow OR warning present** → skip this action. Log to the ledger as `skipped: safety_yellow`. Do NOT write. Notify `[ALERT_CHANNEL]` if this is the third skip for the same reactor today.
- **Status red OR restriction active** → skip ALL remaining actions for this reactor today. Log as `skipped: safety_red`. Send an immediate alert to `[ALERT_CHANNEL]`.

This runs before every write. LinkedIn's rate signals can flip mid-session — a green status at 9:00 can be yellow at 9:14. Never trust a previous check.

---

## STEP 5 — GENERATE + POST THE COMMENT

For each (reactor, post) pair that passed the safety gate:

1. Fetch the post text via `linkedin_get_post` if `linkedin_get_user_posts` returned only the URN.
2. Load the reactor's voice DNA. Either:
   - From the reactor's stored voice DNA (default — call `library_context_get` or the equivalent voice-DNA fetch), OR
   - From `voice_dna_ref` in the pod config if overridden
3. Call Zevari `agents_generate_warming_comment` with:
   - `post_text`: the anchor's post
   - `actor_voice_dna`: the reactor's voice DNA
   - `relationship_context`: "[same team / coworker / friend — engaging on a teammate's post]"
   - `length_hint`: vary across reactors (one short, one medium — never all the same length)
4. Review the generated comment:
   - Must not start with "Great", "Love", "Spot on", or any other top-3 slop opener.
   - Must read like a reply from a teammate, not a fan.
   - Should reference *something specific* in the post — a number, a phrase, a contradiction.
   - If the comment fails any of these checks, regenerate with a stricter prompt or drop the action.
5. Call `linkedin_comment_on_post` with the reactor's workspace + the post URN + the generated comment.
6. With probability `react_probability` (default 0.5), also call `linkedin_react_to_post` (👍 default, ❤️ if the post is celebratory).
7. Log to the ledger via `targets_save` into list `pod-engagement-YYYY-MM-DD`:
   - `post_urn`, `anchor_id`, `reactor_id`, `action_type` (comment / react / repost), `comment_text`, `fired_at`

---

## STEP 6 — REPOST WINDOW

30-45 minutes after the anchor publishes, pick 1-2 reactors (random, under their `daily_cap_reposts`) to repost.

For each chosen reactor:

1. Run the safety gate (Step 4) again.
2. Generate a short take of their own — 1-2 sentences in their voice — that builds on the anchor's post. Use `agents_generate_warming_comment` with `output_format: "repost_caption"` if your Zevari version supports it, otherwise hand-craft the prompt:
   - "Write a 1-2 sentence repost caption in [reactor]'s voice that adds one of: a contrasting angle, a related stat, a question. Do not summarize the original post."
3. Call `linkedin_create_post` with the reactor's workspace, `share_urn` of the anchor's post, and the caption.
4. Log to the ledger.

---

## STEP 7 — END-OF-RUN SLACK SUMMARY

After each poll cycle (not just end of day), send a brief summary to `[POD_CHANNEL]` via Chrome `javascript_tool` (not curl/python).

Format:

```
🤝 *Engagement Pod — Run Update [HH:MM local]*

📝 New posts detected this cycle: [N]
💬 Comments fired: [N]
   • [anchor → reactor]: "[comment preview, ~60 chars]"
   • ...
🔁 Reposts fired: [N]
   • [anchor → reactor]: "[caption preview]"
⏭️ Skips: [N]
   • [reactor]: safety_yellow / daily_cap_hit / etc.
🛑 Hard stops: [N] (red safety status)
```

Skip the summary post entirely if the cycle had zero activity (no new posts, no fires, no skips) — avoids spamming the pod channel with empty updates.

---

## STEP 8 — 6 PM AMPLIFICATION DM

At 6 PM local time, for each anchor, send a Slack DM (Chrome `javascript_tool` calling Slack's `chat.postMessage` via the webhook with `channel: [anchor's Slack ID]` if your webhook supports DMs, else a separate webhook per anchor):

```
📊 *Daily Pod Amplification — [anchor name]*

Posts published today: [N]
For each post:
• "[post first 50 chars]..."
  ↳ Pod comments received: [N] (of [pod_size - 1] possible)
  ↳ Pod reactions received: [N]
  ↳ Pod reposts received: [N]

Reciprocity check: you engaged on [N] of [M] other-anchor posts today.
```

The point: anchors can see whether the pod is showing up for them fairly, and whether they're carrying their share. Reciprocity is the thing that kills pods over time — surfacing the stat keeps it healthy.

---

## STEP 9 — REFLECTION (end of day)

After the 6 PM DM, briefly note in the run log:

- Any reactors with > 2 safety_yellow skips → suggest taking that account out of the pod tomorrow.
- Any anchor with reciprocity < 0.2 → flag (they're free-riding).
- Any comments that got regenerated 2+ times before passing the slop check → review the voice DNA, it may be stale.
- Any 8 PM-or-later posts the pod missed because the posting window ended at 6 PM → consider extending the window.

---

## HARD RULES (non-negotiable)

- **Never write without running `linkedin_get_safety_status` first.** Not "every session" — every write.
- **Never use the same reactor first on consecutive anchor posts.** Always shuffle.
- **Never let all reactors comment with the same length.** Vary `length_hint` across reactors per post.
- **Never have the anchor reciprocate on 100% of other anchors' posts.** Cap at `reciprocation_rate` (default 30%).
- **Never run pods larger than 8 members.** 3-5 is the safe sweet spot.
- **Never push past a yellow safety status hoping it clears.** It doesn't. It escalates.

---

## Schedule

Runs every 5 minutes during the posting window (default 8 AM - 6 PM local). Wrap-up + 6 PM DM at the end. I run this via Claude Code's cron.
