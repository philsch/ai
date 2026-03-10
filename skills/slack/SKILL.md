---
name: slack
description: "Interact with Slack — read conversations, threads, and messages, and reply to messages. Use when the user shares a Slack permalink or asks to read a Slack message or thread, or wants to reply to a Slack message."
---

# Slack Skill

## Credentials

Read `SLACK_USER_TOKEN` and `SLACK_COOKIE` from the `.env` file in the same directory as this skill (`<path-to-this-skill>/.env`).

- `SLACK_USER_TOKEN`: session user token, starts with `xoxc-`
- `SLACK_COOKIE`: the value of the browser `d` cookie, starts with `xoxd-`

If the file is missing or either value is not set, show the user these setup instructions:

```
Slack credentials not found. To set up:
1. Copy skills/slack/.env.example to skills/slack/.env
2. Follow the instructions inside to obtain your xoxc- token and xoxd- cookie
3. Set SLACK_USER_TOKEN and SLACK_COOKIE in .env
```

Load the credentials with:
```bash
export $(grep -v '^#' <path-to-this-skill>/.env | xargs)
```

All API calls must include both headers:
```
-H "Authorization: Bearer ${SLACK_USER_TOKEN}"
-H "Cookie: d=${SLACK_COOKIE}"
```

---

## Utility: Resolve channel name to channel_id

Use this procedure when only a channel name (e.g. `#general`) is known and a `channel_id` is required.

### Step 1 — Fetch the first page

```bash
curl -s -X GET \
  -H "Authorization: Bearer ${SLACK_USER_TOKEN}" \
  -H "Cookie: d=${SLACK_COOKIE}" \
  "https://slack.com/api/conversations.list?exclude_archived=true&limit=1000" \
  | jq '{channels: [.channels[] | {id, name}], next_cursor: .response_metadata.next_cursor}'
```

### Step 2 — Search the results

Search the returned `channels` array for an item whose `name` matches the target channel name (without the leading `#`).

- If found, use its `id` as the `channel_id`.

### Step 3 — Paginate if needed

If the channel was not found, check `next_cursor`:

- If `next_cursor` is non-empty, repeat the call with `&cursor=<next_cursor>` appended to the URL and go back to Step 2.
- If `next_cursor` is empty, all pages are exhausted.

### Step 4 — Handle not found

If the channel is not found after all pages, surface an error to the user:

```
Channel "#<name>" not found. Verify the channel name and try again.
```

---

## Message Formatting

### Character escaping

The following characters **must always be escaped** in message text before sending:

| Character | Escaped form |
|-----------|-------------|
| `&`       | `&amp;`     |
| `<`       | `&lt;`      |
| `>`       | `&gt;`      |

No other characters require escaping.

### Markdown-style formatting

Slack uses its own formatting syntax (not standard Markdown):

| Syntax | Result |
|--------|--------|
| `_italic_` | italic text |
| `*bold*` | bold text |
| `~strike~` | strikethrough text |
| `\n` | line break (new line in multi-line text) |
| `> text` | block quote (leading `>` on one or more lines) |
| `` `code` `` | inline code |
| ` ```code block``` ` | multi-line code block (use `\n` inside for line breaks) |

---

## Capability: Read a conversation from a permalink

**Trigger:** The user provides a Slack permalink URL in the form:
```
https://<workspace>.slack.com/archives/C123ABC/p1712345678123456
```

### Step 1 — Parse the permalink

Extract two values from the URL path:

- `channel_id`: the segment after `/archives/` (e.g. `C123ABC`)
- `ts`: the `p`-prefixed segment with a `.` inserted after position 10
  - Example: `p1712345678123456` → `1712345678.123456`
  - Rule: strip the leading `p`, then insert `.` between character 10 and 11

### Step 2 — Fetch the message

```bash
curl -s -X GET \
  -H "Authorization: Bearer ${SLACK_USER_TOKEN}" \
  -H "Cookie: d=${SLACK_COOKIE}" \
  -H "Content-Type: application/json" \
  "https://slack.com/api/conversations.history?channel=<channel_id>&oldest=<ts>&latest=<ts>&inclusive=true&limit=1" \
  | jq .
```

Check the response:
- If `.ok` is `false`, surface the `.error` field to the user and stop.
- Extract the message from `.messages[0]`.

### Step 3 — Check for thread replies

If the message has `.reply_count > 0`, fetch the full thread:

```bash
curl -s -X GET \
  -H "Authorization: Bearer ${SLACK_USER_TOKEN}" \
  -H "Cookie: d=${SLACK_COOKIE}" \
  -H "Content-Type: application/json" \
  "https://slack.com/api/conversations.replies?channel=<channel_id>&ts=<ts>" \
  | jq .
```

Use `.messages[]` for the full thread (first message is the parent).

### Step 4 — Resolve user display names

For each unique `user` ID found across all messages, call `users.info` to get a human-readable name. Cache results to avoid duplicate calls.

```bash
curl -s -X GET \
  -H "Authorization: Bearer ${SLACK_USER_TOKEN}" \
  -H "Cookie: d=${SLACK_COOKIE}" \
  -H "Content-Type: application/json" \
  "https://slack.com/api/users.info?user=<user_id>" \
  | jq '.user.real_name // .user.name'
```

### Step 5 — Present to the agent

Before presenting, pre-process each message body:
- Replace `<@USERID>` with `@<resolved_name>` (use the cache from Step 4)
- Replace `<URL|display>` with `display (URL)`
- Replace `<!channel>` with `@channel` and `<!here>` with `@here`

Then output structured context, not a visual layout. Use this format for each message:

```
---
message_id: <ts>
thread_ts: <thread_ts>          # same as message_id if this is the root
role: root | reply
channel_id: <channel_id>
channel_name: <resolved name, if available>
channel_type: public_channel | private_channel | im | mpim
author: <resolved_name> (<user_id>)
timestamp_unix: <ts as float>
timestamp_human: <ISO 8601, e.g. 2024-04-05T14:23:11Z>
edited: true | false
text: |
  <resolved message body, multi-line if needed>
reactions: <emoji>(N), <emoji>(N)   # omit line if none
reply_count: <N>                    # root message only; omit for replies
---
```

Output the root message first, followed by replies in chronological order. If there are no replies, omit the `reply_count` line. If a field has no value, omit the line entirely rather than leaving it blank.

---

## Capability: Reply to a message in a thread

**Trigger:** The user wants to reply to a Slack message, optionally providing a permalink and reply text.

This capability always posts into the thread of the original (root) message. If the target message already has replies (i.e. is part of an ongoing thread), the new reply is appended to that same thread.

### Step 1 — Identify the target message

If the user provides a permalink, parse `channel_id` and `ts` as described in the "Read a conversation" capability.

If the user does not provide a permalink but has already read a message in this session, reuse the `channel_id` and `ts` (= `thread_ts`) from that context.

### Step 2 — Compose the reply text

Use the reply text provided by the user exactly as given. Do not paraphrase or modify it.

**Before sending, always show the user a preview of the message and ask for confirmation.** Only proceed after explicit approval.

### Step 3 — Post the reply

Use `chat.postMessage` with `thread_ts` set to the root message's `ts`. This ensures the reply is always attached to the thread, regardless of whether replies already exist.

```bash
curl -s -X POST \
  -H "Authorization: Bearer ${SLACK_USER_TOKEN}" \
  -H "Cookie: d=${SLACK_COOKIE}" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "channel=<channel_id>" \
  --data-urlencode "thread_ts=<root_ts>" \
  --data-urlencode "text=<reply_text>" \
  "https://slack.com/api/chat.postMessage" \
  | jq .
```

Check the response:
- If `.ok` is `false`, surface the `.error` field to the user and stop.
- On success, confirm to the user that the reply was posted, including the resolved timestamp from `.message.ts`.
