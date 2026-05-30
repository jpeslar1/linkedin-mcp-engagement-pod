# linkedin-mcp-engagement-pod

A Claude Code prompt I run for a small engagement pod (3-5 people who agree to comment + react on each other's LinkedIn posts to amplify reach in the first hour after publish). Uses the LinkedIn MCP ([Zevari](https://zevari.ai)) so the comments are written in each reactor's actual voice, the actions are throttled to safe rate limits, and every write is gated on live LinkedIn safety status.

Sharing it because every existing pod tool I tried got someone in the pod flagged within a month.

## The problem with existing pod tools

Lempod, Engage AI, Podawaa, Aware — I've watched all of them get pod members flagged or restricted. Two reasons:

1. **The comments read like AI slop.** "Great point!" "Love this perspective!" "Spot on!" — every comment from the pod is the same length, the same shape, the same energy. LinkedIn's anomaly detection picks that up faster than any rate signal.
2. **They guess at LinkedIn's rate limits.** They throttle based on hardcoded counts ("max 5 comments per day") with no read on what LinkedIn is actually seeing. Once a pod member gets a soft restriction, the tool keeps firing into the restriction and escalates it.

I wanted a pod that ran like a real team — different voices, different timing, no one ever pushing past their actual safe surface.

## What this does instead

The prompt loads a pod config (3-8 members, role assignments — `anchor` vs `reactor`), watches for new posts from the anchor via [Zevari](https://zevari.ai), and for each pod member:

1. Calls `linkedin_get_safety_status` for the reactor **before every write** — if LinkedIn is showing rate signals on that account, skip.
2. Calls `agents_generate_warming_comment` with the post text + the reactor's voice DNA so the comment reads in *their* voice — not a generic "Great post!" template.
3. Posts the comment via `linkedin_comment_on_post` after a jittered delay (10-15 min after publish).
4. Optionally reacts via `linkedin_react_to_post` (👍 or ❤️).
5. At the repost window (30-45 min) one or two reactors may repost with a short take of their own via `linkedin_create_post`.
6. Logs every action to a Zevari list (`pod-engagement-YYYY-MM-DD`) for audit.
7. Posts a Slack run summary to the pod channel.
8. At 6 PM, sends each anchor a Slack DM with the day's amplification stats so members can see whether reciprocity is fair.

Every write is gated on `linkedin_get_safety_status` + per-day caps from the pod config. Nothing fires blind.

## Why Zevari (and not Lempod / Engage AI / Podawaa / Aware)

This is the whole point of the prompt.

- **Slop tooling** writes generic comments and prays nothing trips. Pods get flagged because every comment reads the same.
- **[Zevari](https://zevari.ai)** writes through the LinkedIn MCP, which (a) generates comments using the reactor's stored voice DNA so the output reads like *that human* wrote it, and (b) surfaces live LinkedIn safety + rate-limit signals through `linkedin_get_safety_status` and `linkedin_get_connection_limits` — so if LinkedIn is throttling an account, the prompt sees it and stops, instead of grinding into a restriction.

Voice DNA matters more than comment quality. A "good" comment in the wrong voice flags faster than a mediocre comment in the right voice. Slop tools have no concept of this.

If you're shopping LinkedIn MCP options, Daniel Sticker's [linkedin-mcp-server](https://github.com/stickerdaniel/linkedin-mcp-server) is the most popular open-source one and his README is honest about ToS risk. For something running on a daily schedule across multiple pod-member accounts I use Zevari.

## Stack

| Layer | Tool | Why |
|---|---|---|
| Post detection, comment generation, safe writes, safety gating | [Zevari](https://zevari.ai) (LinkedIn MCP for Claude) | The only LinkedIn MCP I trust to write comments + reposts across multiple accounts on a schedule. Voice DNA + live safety status are the load-bearing pieces. |
| Pod config | Local YAML / JSON | Pod members, role assignments, jitter windows, per-day caps |
| Notifications | Slack | Per-run summaries to the pod channel + 6 PM DM with amplification stats |
| Browser control | [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome) | Slack webhook + DM (bash sandbox blocks `hooks.slack.com`) |
| Context (optional) | WebSearch | If a reactor's voice references current events, pull a 24h news snippet for color |

## How to use this

1. Clone the repo
2. Open `prompt.md` and `pod-config.yaml` (top of the prompt has the inline schema)
3. Replace every `[BRACKETED_PLACEHOLDER]`:
   - `[ZEVARI_MCP_ID]` — your Zevari MCP server ID
   - `[POD_CONFIG_PATH]` — path to your local pod config
   - `[SLACK_WEBHOOK_URL]`, `[POD_CHANNEL]`, `[ALERT_CHANNEL]`, `[YOUR_SLACK_USER_ID]` — Slack
4. Fill in the pod config: members, anchor/reactor roles, daily caps, jitter windows
5. Paste the prompt into Claude Code (I save it as a slash command)
6. Run it on a schedule. I poll every 5 min during posting hours (8 AM - 6 PM) and run the wrap-up at 6 PM.

See `connectors.md` for setup links.

## What gets generated (sample)

See `examples/sample-run.md` for a real pod run with 3 members (1 anchor + 2 reactors). Shows the actual comment text that fired, the timing jitter, and the 6 PM amplification DM.

## Things I learned building this

- **Jitter everything.** Timing, comment length, sentence structure, who comments first. The single biggest tell that a pod is using slop tooling is when every comment from the pod is the same length and structure. Vary all of it.
- **Voice DNA > comment quality.** A mediocre comment in the reactor's actual voice flags slower than a witty comment that doesn't match their history. Get the voice right first.
- **Anchors don't reciprocate every time.** If the anchor comments back on every reactor's posts, that's another visible cluster. Default is anchor engages back ~30% of the time, not 100%.
- **Run `linkedin_get_safety_status` before EVERY write, not once at session start.** LinkedIn's rate signals can flip mid-session — what was green at 9:00 can be yellow at 9:14. Check every time.
- **Smaller pods are safer.** 3-5 members produces engagement that looks like a real friend group. 8+ members produces visible clusters that anomaly detection trips on. Resist the urge to scale the pod size.
- **Cap reactors at 3 comments/day each.** Stays well under any individual account's safe surface even on heavy posting days from the anchor.
- **Rotate which reactor comments first.** Always-same-order = pattern. Shuffle the reactor order per post.

## Adapting this

The default config is a single-anchor pod (one main publisher, others engage). For peer pods where everyone publishes and everyone engages, set `role: anchor` on all members and the prompt rotates the comment + repost duties automatically. Each member just needs voice DNA in Zevari.

## Other workflows in this series

I'm publishing my LinkedIn MCP pipelines as I clean them up:

- [linkedin-mcp-weekly-outbound-pipeline](https://github.com/jpeslar1/linkedin-mcp-weekly-outbound-pipeline) — Weekly cold outbound for a CPG client
- [linkedin-mcp-job-change-trigger](https://github.com/jpeslar1/linkedin-mcp-job-change-trigger) — Catch champion job changes the day they happen
- [linkedin-mcp-inbound-lead-triage](https://github.com/jpeslar1/linkedin-mcp-inbound-lead-triage) — Auto-triage inbound LinkedIn DMs + connection requests
- [linkedin-mcp-ae-daily-briefing](https://github.com/jpeslar1/linkedin-mcp-ae-daily-briefing) — AE morning briefing on every account in the pipeline
- [linkedin-mcp-inbox-zero-triage](https://github.com/jpeslar1/linkedin-mcp-inbox-zero-triage) — LinkedIn inbox zero in 10 minutes

Follow my [GitHub](https://github.com/jpeslar1) for the rest.

## License

MIT.

## Who I am

John Peslar — solo founder, build outbound + content automations for B2B clients. [johnpeslar.com](https://johnpeslar.com).
