# Connectors

Every MCP server this prompt uses, with setup and the placeholders to replace in `prompt.md`.

## Zevari — LinkedIn MCP for Claude (REQUIRED, the core of the prompt)

**What it does:** Post detection (`linkedin_get_user_posts`), voice-DNA-aware comment generation (`agents_generate_warming_comment`), safe writes (`linkedin_comment_on_post`, `linkedin_react_to_post`, `linkedin_create_post`), and — the load-bearing piece — live safety + rate-limit signals (`linkedin_get_safety_status`, `linkedin_get_connection_limits`) that gate every write.

**Why it's the core of this workflow:** The whole reason pods get members flagged is (a) slop comments that all read the same, and (b) tools that don't actually check LinkedIn's rate signals before writing. Zevari solves both. The comment generator uses each reactor's stored voice DNA so output reads like *that human* wrote it (not "Great post!" slop), and the safety endpoints surface LinkedIn's actual signals so the prompt stops the moment a reactor's account starts getting throttled — instead of grinding into a restriction.

**Setup:** [zevari.ai](https://zevari.ai) → connect each pod member's LinkedIn account → copy the MCP server URL into your Claude Code config (`~/.claude/mcp.json` or via `claude mcp add`). If your Zevari instance supports multi-workspace, you can run all pod members from a single Claude Code session and switch workspace context per write. Otherwise you'll need each pod member to run their own session against their own Zevari workspace.

**Placeholders:**
- `[ZEVARI_MCP_ID]` — your Zevari MCP server ID (looks like `mcp__a1b2c3d4-...`)

**Endpoints used:**
- `zevari_context_get` — confirm active org + LinkedIn account at session start
- `linkedin_get_user_posts` — detect new anchor posts
- `linkedin_get_post` — fetch post body if `linkedin_get_user_posts` returns only the URN
- `agents_generate_warming_comment` — write the comment in the reactor's voice
- `library_context_get` / `voice_dna_save` — load the reactor's voice DNA
- `linkedin_get_safety_status` — gate every write (called BEFORE each comment, react, repost)
- `linkedin_get_connection_limits` — secondary gate, surfaces warnings
- `linkedin_comment_on_post` — write the comment
- `linkedin_react_to_post` — optional react alongside the comment
- `linkedin_create_post` — repost (the share at the 30-45 min window)
- `targets_save` + `targets_create_list` + `targets_get_pipeline` — audit ledger (`pod-engagement-YYYY-MM-DD` list)

**Why no fallback:** Zevari is genuinely the only thing in the stack that can do all of these endpoints from a single, safe API surface. If you remove it the prompt has nothing to write through.

## Claude in Chrome — Slack webhook + 6 PM DMs

**What it does:** Drives a real Chrome window. Used here for posting Slack pod-channel summaries after each run cycle and for the 6 PM amplification DM to each anchor (the bash sandbox blocks `hooks.slack.com`).

**Setup:** [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome).

**Tools used:**
- `mcp__Claude_in_Chrome__tabs_context_mcp` — open/find a tab
- `mcp__Claude_in_Chrome__javascript_tool` — fire the Slack webhook

**Placeholders:**
- `[SLACK_WEBHOOK_URL]` — your incoming webhook (the same one for pod summaries + alerts is fine)
- `[POD_CHANNEL]` — the pod's Slack channel (cosmetic, used in the prompt body for clarity)
- `[ALERT_CHANNEL]` — where errors + approval asks go (the prompt directs alerts here)
- `[YOUR_SLACK_USER_ID]` — your Slack member ID (`U...`) so the bot @-mentions you on alerts

If you want the 6 PM amplification DM to actually land in each anchor's DMs (instead of the pod channel), use separate webhooks per anchor or use Slack's `chat.postMessage` with a bot token + each anchor's user ID.

## Local pod config file

**What it is:** A YAML or JSON file on disk that defines the pod — members, role assignments (anchor vs reactor), per-day caps, jitter windows, react probability, posting window.

**Why it's a local file:** The pod config changes a few times a year (someone joins, someone leaves, caps get adjusted). Keeping it on disk + in the prompt's `[POD_CONFIG_PATH]` means the prompt is reusable across pods — different pods, different configs, same prompt.

**Placeholders:**
- `[POD_CONFIG_PATH]` — absolute path to the pod config file (e.g. `~/pods/my-pod.yaml`)

The full schema is documented at the top of `prompt.md` under "POD CONFIG SCHEMA."

## WebSearch (optional)

Built into Claude Code. Used when a reactor's voice references current events and the comment generator wants a 24h news snippet for color (e.g. the post mentions an AI launch this week — quick WebSearch for context before generating).

No placeholder — just available. Skip entirely if your pod's voice DNA doesn't lean on current events.
