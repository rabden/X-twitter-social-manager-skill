# Hermes Tweet Backend Adapter

Use this optional backend when the host agent has Hermes Agent with the
Hermes Tweet plugin enabled. Keep `twitter-cli` as the default backend. Switch
to Hermes Tweet when the user wants structured X/Twitter reads, safer action
gating, monitor/export workflows, or a Hermes-native toolset.

Do not require Hermes Tweet for normal use. This adapter is additive.

## When to Use This Backend

| Need | Preferred Backend |
|------|-------------------|
| Simple local posting with existing browser cookies | `twitter-cli` |
| Structured tweet search with query parameters | Hermes Tweet |
| Read tweet details, replies, quotes, threads, likers, retweeters | Hermes Tweet |
| User lookup, recent tweets, mentions, followers | Hermes Tweet |
| Monitor tweets, keywords, or accounts | Hermes Tweet |
| Export followers or run extraction jobs | Hermes Tweet |
| Approval-gated tweet, reply, DM, follow, like, retweet | Hermes Tweet or `twitter-cli`, whichever is verified |
| Video media upload | Manual posting unless the selected backend supports it |

## Install

In Hermes Agent:

```bash
hermes plugins install Xquik-dev/hermes-tweet --enable
hermes tools list
```

Interactive install can prompt for `XQUIK_API_KEY`. Non-interactive sessions
must set it in the Hermes runtime environment or `~/.hermes/.env`.

Never ask the user to paste API keys into chat. Only ask them to configure the
runtime:

```bash
export XQUIK_API_KEY="..."
export HERMES_TWEET_ENABLE_ACTIONS="false"
```

Set actions to true only for interactive sessions where the user will approve
each write:

```bash
export HERMES_TWEET_ENABLE_ACTIONS="true"
```

## Backend Status

Track this in `references/memory.md` under Environment Status:

```markdown
- **X backend:** twitter-cli / Hermes Tweet / manual
- **Hermes Tweet:** unavailable / explore-only / read-ready / actions-ready
```

Status meanings:

- `unavailable`: toolset is not visible.
- `explore-only`: `tweet_explore` is available, but no API key is configured.
- `read-ready`: `tweet_read` can call read-only endpoints.
- `actions-ready`: `tweet_action` is enabled for approval-gated actions.

## Tool Rules

Hermes Tweet exposes three tools:

1. `tweet_explore` - search the endpoint catalog. Does not call X.
2. `tweet_read` - call catalog-listed read-only endpoints.
3. `tweet_action` - call write-like or private endpoints when actions are enabled.

Rules:

- Always call `tweet_explore` before using an endpoint. Do not guess paths.
- Use `tweet_read` only for GET endpoints that the catalog marks as read-only.
- Use `tweet_action` for posting, replies, DMs, follows, likes, retweets,
  monitors, exports, media jobs, giveaway draws, private reads, and deletes.
- Before `tweet_action`, show the exact action summary and final payload to the
  user, then wait for explicit approval.
- If `tweet_action` is disabled, stop and tell the user how to enable actions.
  Do not route around the gate.
- Never pass credentials in tool arguments.

## Command Mapping

Use this table when a workflow references `twitter-cli`.

| Existing CLI Step | Hermes Tweet Flow |
|-------------------|-------------------|
| `twitter user HANDLE --json` | Explore `user lookup`, then `tweet_read` |
| `twitter user-posts HANDLE --max 20 -c --json` | Explore `user tweets`, then `tweet_read` |
| `twitter search "query" --json` | Explore `tweet search`, then `tweet_read` |
| `twitter tweet TWEET_ID --json` | Explore `tweet details`, `tweet replies`, or `tweet thread`, then `tweet_read` |
| `twitter followers HANDLE --json` | Explore `followers export` or `followers`, then use catalog route |
| `twitter post "text" --json` | Explore `post tweet`, then `tweet_action` after approval |
| `twitter reply TWEET_ID "text" --json` | Explore `reply tweet`, then `tweet_action` after approval |
| `twitter like TWEET_ID` | Explore `like tweet`, then `tweet_action` after approval |
| `twitter retweet TWEET_ID` | Explore `retweet`, then `tweet_action` after approval |
| `twitter follow HANDLE` | Explore `follow user`, then `tweet_action` after approval |
| DM outreach | Explore `send DM`, then `tweet_action` after approval |

## Research Queries

Pre-post research usually starts with:

```json
{"query":"tweet search","method":"GET"}
```

Then read with the endpoint returned by `tweet_explore`:

```json
{"path":"/api/v1/x/tweets/search","query":{"q":"topic keywords","limit":20}}
```

For competitor analysis:

```json
{"query":"user tweets","method":"GET"}
```

For voice extraction:

```json
{"query":"tweet search","method":"GET"}
```

Then search:

```json
{"path":"/api/v1/x/tweets/search","query":{"q":"from:HANDLE filter:replies","limit":30}}
```

Use endpoint and parameter names returned by `tweet_explore`; examples show
intent, not a reason to skip catalog lookup.

## Posting Flow

Use this when the direct posting section selects Hermes Tweet:

1. Validate character count.
2. Confirm media support. If the post requires video and the catalog route does
   not support it, prepare a manual posting package instead.
3. Present final post text, first reply text, account, and action summary.
4. Wait for explicit approval.
5. Explore `post tweet` with actions included.
6. Call `tweet_action` with the catalog path, method, body, and reason.
7. Capture tweet URL or id from the response.
8. If first reply exists, repeat the approval summary for the reply or include
   it only when the user approved both actions together.
9. Log backend, endpoint, timestamp, and tweet id in `memory.md`.

Payload example:

```json
{
  "path": "/api/v1/x/tweets",
  "method": "POST",
  "body": {
    "account": "@handle",
    "text": "Final approved post text"
  },
  "reason": "Post the user-approved X Social Media Manager draft."
}
```

## Reply Flow

Use this for reply crafting:

1. Read the target tweet.
2. Read thread/replies if needed for context.
3. Analyze media if available and supported.
4. Draft 2-3 replies with `[N/280]`.
5. Wait for approval of one specific reply.
6. Explore `reply tweet`.
7. Call `tweet_action`.
8. Log engaged account and reply id in `memory.md`.

## DM Outreach Flow

DMs require the strictest gate:

- User must identify the recipient.
- User must approve the exact message.
- Do not bulk DM.
- Do not auto-follow then DM.
- Do not invent relationships or claim prior contact.
- Do not include private profile notes unless the user explicitly includes them.

Explore `send DM` only after the draft is approved.

## Monitoring and Intelligence Update

Hermes Tweet is useful for the self-evolving parts of this skill:

- monitor a posted tweet for replies and metrics
- monitor a keyword for lead opportunities
- monitor Tier 1 accounts
- export followers for manual segmentation
- fetch quote tweets or retweeters for audience analysis

Before creating a monitor:

1. State target, cadence, stop condition, and memory fields to update.
2. Get approval.
3. Use `tweet_action` only if actions are enabled.
4. For unattended follow-up reads, prefer read-only calls after setup.

Suggested memory entry:

```markdown
### [Date] - Hermes Tweet Monitor
- Target: [tweet/account/query]
- Backend: Hermes Tweet
- Cadence: [cadence]
- Stop: [condition]
- Fields: [metrics or audience segments]
```

## Failure Handling

| Failure | Action |
|---------|--------|
| `tweet_explore` unavailable | Use `twitter-cli` or manual package. |
| API key missing | Ask user to configure the Hermes runtime. Do not request the key value. |
| `tweet_action` disabled | Stop and explain `HERMES_TWEET_ENABLE_ACTIONS=true`. |
| Endpoint not found | Try a clearer `tweet_explore` query. Do not guess a path. |
| Auth, policy, account-state, or rate-limit error | Stop, report the error category, and wait for user direction. |
| Write succeeds but reply fails | Report the posted tweet id and failed follow-up. Do not keep posting. |

## Output Standards

For read workflows, summarize:

- what endpoint was used
- time range or query
- key findings
- confidence and limitations
- 3-5 actionable content or engagement recommendations

For write workflows, summarize:

- account
- action
- final text
- endpoint
- approval phrase
- response id or URL

Do not include raw secrets, cookies, API keys, or full private logs in outputs.
