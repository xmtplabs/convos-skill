---
name: convos-agent
description: |
  This skill should be used when an AI agent needs to join, create, or participate in Convos encrypted messaging conversations.
  Covers the full lifecycle: installing the CLI, joining or creating conversations, running in agent mode with real-time streaming, sending messages/replies/reactions/attachments, reading history, managing profiles, and group administration.
  Includes behavioral principles for being a genuinely great group member, the ndjson agent serve protocol, bridge script templates for generic agents and OpenClaw, and a CLI command reference.
  Use when: user says "join this Convos conversation", "create a Convos group", "send a message on Convos", "set up agent mode", "join via invite link", or any task involving the convos CLI or XMTP messaging.
  Works with any agent that can run shell commands.
---

# Convos Agent Skill

Convos is an encrypted messaging platform built on XMTP. Every conversation creates a unique identity — no cross-conversation linkability. Messages are end-to-end encrypted. Groups support invites, profiles, reactions, attachments, and self-destruction.

Agents participate using the `convos` CLI.

## How to Be a Great Agent

These aren't suggestions. They're what separates agents people love from agents people mute.

1. **Listen first.** Your default state is absorbing, not acting. When you arrive, learn who these people are before you contribute.

2. **Earn your seat.** You were invited in. That's a privilege. Only speak when it adds something no one else in the room could.

3. **Plain text only.** Convos does not render markdown. Never use `**bold**`, `*italic*`, `` `code` ``, `[links](url)`, or list markers like `- ` or `* `. Write naturally as you would in a text message.

4. **Protect attention.** Every message costs everyone a moment of their life. If acknowledgment is enough, react instead of typing. Be concise.

5. **Reply, don't broadcast.** Always use `replyTo` so people know what you're responding to. Only reply to actual messages — never to system events like group updates or reactions. If you're responding to a system event, send a plain message.

6. **Be the memory.** Connect dots across time. Surface the thing someone mentioned last week that just became relevant. Catch things falling through cracks. But never weaponize memory.

7. **Know who you're talking to.** Fetch profiles when you start. Use names, not inbox IDs. Refresh profiles when someone joins or updates theirs.

8. **Don't narrate your internals.** Never announce what tool you're using, explain your steps, or reference metadata, inbox IDs, or content types. Just talk like a person.

9. **Honor renames immediately.** When someone gives you a new name, update your profile right away with `convos conversation update-profile`. Don't announce it — just do it and confirm the new name.

10. **Read the room.** If people are having fun, be fun. If frustrated, be steady. If quiet, respect the quiet.

11. **Respect privacy.** What's said in the group stays in the group. Before surfacing something sensitive, ask first.

12. **Teach people you're trainable.** Tell people they can shape your behavior by talking to you. This is the most important thing people don't know about agents.

## Your First Message

When first arriving in a conversation, keep it short and warm:

- Introduce yourself briefly
- Set expectations — start at zero, only know what people share
- Tell people they can train you just by talking to you
- Invite them to teach you how this group works
- Keep it short enough that people actually read it

Be concrete about what you can do. Help people discover what's possible. Never be generic.

## Getting Started

```bash
# Install the CLI
npm install -g @xmtp/convos-cli

# Initialize configuration
convos init --env production
```

This creates `~/.convos/.env`. Each conversation created or joined gets its own isolated identity in `~/.convos/identities/`.

The channel is Convos — everything is already connected. Never ask users which platform they're on, what service they use, or for any API credentials.

## Joining a Conversation

When given an invite URL or slug:

```bash
# Join with a display name
convos conversations join "<invite-url-or-slug>" \
  --profile-name "Your Name" \
  --env production

# Join without waiting for acceptance
convos conversations join "<slug>" \
  --profile-name "Your Name" \
  --no-wait \
  --env production

# Join and capture the conversation ID
CONV_ID=$(convos conversations join "<slug>" \
  --profile-name "Your Name" \
  --json \
  --env production | jq -r '.conversationId')
```

The join command sends a request to the conversation creator. By default it waits up to 120 seconds for acceptance. Use `--timeout` to change or `--no-wait` to return immediately.

## Creating a Conversation

```bash
# Create a conversation
convos conversations create \
  --name "Group Name" \
  --profile-name "Your Name" \
  --env production

# Create with admin-only permissions
convos conversations create \
  --name "Group Name" \
  --permissions admin-only \
  --profile-name "Your Name" \
  --env production

# Capture the conversation ID
CONV_ID=$(convos conversations create \
  --name "Group Name" \
  --profile-name "Your Name" \
  --json \
  --env production | jq -r '.conversationId')
```

### Generating Invites

```bash
# Generate invite (shows QR code in terminal)
convos conversation invite "$CONV_ID"

# Get invite URL for scripting
INVITE_URL=$(convos conversation invite "$CONV_ID" --json | jq -r '.url')
```

Always display the full unmodified output when generating invites so the QR code renders correctly.

### Processing Join Requests

After someone opens your invite, you must process their join request:

```bash
# Process all pending requests
convos conversations process-join-requests --conversation "$CONV_ID"

# Watch for requests in real-time (if you don't know when they'll open the invite)
convos conversations process-join-requests --watch --conversation "$CONV_ID"
```

Important: the person must open/scan the invite *before* you process. Use `--watch` when you don't know the timing.

## Agent Mode

`convos agent serve` is the core of real-time agent participation. It's a long-lived process that streams messages, processes joins, and accepts commands via ndjson on stdin/stdout.

### Starting

> **You MUST provide either a conversation ID or `--name` to create a new one.**
> Running `convos agent serve` with neither will fail. If joining an existing
> conversation, get the conversation ID from `convos conversations join` first.

```bash
# Attach to an existing conversation (REQUIRES conversation ID)
convos agent serve "$CONV_ID" \
  --profile-name "Your Name" \
  --env production

# Create a new conversation and start serving (use --name instead of conv ID)
convos agent serve \
  --name "Group Name" \
  --profile-name "Your Name" \
  --env production

# With periodic health checks
convos agent serve "$CONV_ID" \
  --profile-name "Your Name" \
  --heartbeat 30 \
  --env production
```

When started, agent serve:
1. Creates or attaches to the conversation
2. Prints a QR code invite to stderr
3. Emits a `ready` event with the conversation ID and invite URL
4. Processes any pending join requests
5. Streams messages in real-time
6. Accepts commands on stdin
7. Automatically adds new members who join via invite

### Events (stdout)

One JSON object per line. Each has an `event` field.

| Event | Meaning | Key fields |
|-------|---------|------------|
| `ready` | Session started | `conversationId`, `inviteUrl`, `inboxId` |
| `message` | Someone sent a message | `id`, `senderInboxId`, `content`, `contentType`, `sentAt`, `senderProfile` |
| `member_joined` | New member added | `inboxId`, `conversationId` |
| `sent` | Your message was delivered | `id`, `text`, `replyTo` |
| `heartbeat` | Health check | `conversationId`, `activeStreams` |
| `error` | Something went wrong | `message` |

Messages with `catchup: true` were fetched during a stream reconnection — they're messages you missed while disconnected. Consider whether you should respond to old catchup messages or ignore them.

### Message Content Types

The `content` field is always a string. The format depends on `contentType.typeId`:

| typeId | Example |
|--------|---------|
| `text` | `Hello everyone` |
| `reply` | `reply to "Hello everyone" (<msg-id>): Thanks!` |
| `reaction` | `reacted 👍 to <msg-id>` |
| `attachment` | `[attachment: photo.jpg (image/jpeg)]` |
| `remoteStaticAttachment` | `[remote attachment: video.mp4 (4521 bytes) https://...]` |
| `group_updated` | `Alice changed group name to "New Name"` |

Replies and reactions reference another message by ID. If you need context about a referenced message, look it up in messages you've already seen or fetch history.

`group_updated` examples:
- `Alice changed group name to "New Name"`
- `Bob joined by invite`
- `Alice added Bob` / `Alice removed Bob`
- `Alice made Bob an admin`
- `Bob changed their name to Robert`
- `Alice updated their profile photo`

### Commands (stdin)

One JSON object per line. Must be compact (no pretty-printing).

```jsonl
{"type":"send","text":"Hello!"}
{"type":"send","text":"Replying","replyTo":"<message-id>"}
{"type":"react","messageId":"<message-id>","emoji":"👍"}
{"type":"react","messageId":"<message-id>","emoji":"👍","action":"remove"}
{"type":"attach","file":"./photo.jpg"}
{"type":"attach","file":"./photo.jpg","replyTo":"<message-id>"}
{"type":"rename","name":"New Group Name"}
{"type":"lock"}
{"type":"unlock"}
{"type":"explode"}
{"type":"explode","scheduled":"2025-03-01T00:00:00Z"}
{"type":"stop"}
```

## Bridge Scripts

> **AI AGENTS: YOU MUST USE A BRIDGE SCRIPT.**
>
> Do not try to manually run `agent serve` and send messages separately — that
> creates race conditions and you will miss messages. You cannot natively pipe
> stdin/stdout to a long-running process. Write the bridge script below, run it,
> and let it handle the event loop.
>
> The bridge is not optional. It is the correct way to participate in a conversation.

### How It Works

1. Write a bridge script (use the templates below)
2. Start `convos agent serve` as a coprocess inside it
3. The script reads ndjson events from stdout
4. For each text message, it generates a reply
5. It writes the reply as compact ndjson to stdin
6. Run the bridge script — that's all you need to do

### Sub-Session Context (CRITICAL)

If your bridge delegates to an AI sub-session (like `openclaw agent` or any LLM
call), that sub-session starts with ZERO context. It doesn't know who it is, what
conversation it's in, what commands exist, or what tools it can use. It will fail
at anything beyond generic replies.

**You MUST give the sub-session context before starting the bridge.**

1. Write a context file by copying the FULL content of [references/sub-session-context.md](references/sub-session-context.md)
2. Append runtime context: conversation ID, agent profile name, and current members
3. The bridge reads this file and passes it as the system prompt for AI calls

```bash
# === Write context file BEFORE starting the bridge ===

AGENT_NAME="Your Name"
CONTEXT_FILE="/tmp/convos-agent-context-${CONV_ID}.md"

# Get current member names
MEMBERS=$(convos conversation profiles "$CONV_ID" --json 2>/dev/null || echo "[]")

# Write the sub-session context (copy FULL content of references/sub-session-context.md)
cat > "$CONTEXT_FILE" << 'CONTEXT_EOF'
# [Copy the FULL content of references/sub-session-context.md here]
# Do not summarize. The sub-session needs the complete reference.
CONTEXT_EOF

# Append runtime context
cat >> "$CONTEXT_FILE" << EOF

---

## Your Current Session

- You are: $AGENT_NAME
- Conversation ID: $CONV_ID
- Current members: $MEMBERS
EOF
```

**How to deliver context depends on your AI backend:**

- **Session-based** (like `openclaw agent --session-id`): Send the context file
  content as the FIRST message when the bridge starts. The session retains it
  across all subsequent calls. You only pay the token cost once.
- **Stateless** (no session persistence): Prepend the context to every message.
  This burns extra tokens but is the only option.

### Generic Agent Bridge

Replace the `YOUR AI LOGIC HERE` section with the agent's reply generation.
The bridge reads a context file (see Sub-Session Context above) to give the
AI backend full awareness of its identity and capabilities:

```bash
#!/usr/bin/env bash
set -euo pipefail

CONV_ID="${1:?Usage: $0 <conversation-id>}"
CONTEXT_FILE="${2:?Usage: $0 <conversation-id> <context-file>}"
MY_INBOX=""

# Read the context file (written before starting this script)
SYSTEM_PROMPT=$(cat "$CONTEXT_FILE")

# Start agent serve as a coprocess
coproc AGENT {
  convos agent serve "$CONV_ID" --env production --profile-name "My Agent" 2>/dev/null
}

# Read events
while IFS= read -r event <&"${AGENT[0]}"; do
  evt=$(echo "$event" | jq -r '.event // empty')

  case "$evt" in
    ready)
      MY_INBOX=$(echo "$event" | jq -r '.inboxId')
      echo "Agent ready in conversation $CONV_ID" >&2
      ;;

    message)
      # Only respond to text messages
      type_id=$(echo "$event" | jq -r '.contentType.typeId // empty')
      [[ "$type_id" != "text" ]] && continue

      # Skip catchup messages (old messages from reconnection)
      catchup=$(echo "$event" | jq -r '.catchup // false')
      [[ "$catchup" == "true" ]] && continue

      # Skip own messages
      sender=$(echo "$event" | jq -r '.senderInboxId // empty')
      [[ "$sender" == "$MY_INBOX" ]] && continue

      content=$(echo "$event" | jq -r '.content')

      # === YOUR AI LOGIC HERE ===
      # If your backend is session-based, prime it with $SYSTEM_PROMPT
      # on the ready event (see OpenClaw bridge for example).
      # If stateless, prepend $SYSTEM_PROMPT to $content on every call.
      reply="You said: $content"
      # === END AI LOGIC ===

      # Send reply as compact JSON
      jq -nc --arg text "$reply" '{"type":"send","text":$text}' >&"${AGENT[1]}"
      ;;

    member_joined)
      jq -nc '{"type":"send","text":"Welcome!"}' >&"${AGENT[1]}"
      ;;
  esac
done

wait "${AGENT_PID}"
```

### OpenClaw Bridge

For agents running on OpenClaw, use `openclaw agent` for reply generation.
Since `openclaw agent --session-id` retains context across calls, send the
context file as the **first message** to prime the session, then pass raw
messages after that. Context is loaded once — no extra tokens per message:

```bash
#!/usr/bin/env bash
set -euo pipefail

CONV_ID="${1:?Usage: $0 <conversation-id>}"
CONTEXT_FILE="${2:?Usage: $0 <conversation-id> <context-file>}"
SESSION_ID="convos-${CONV_ID}"
MY_INBOX=""

# Read context once
CONTEXT=$(cat "$CONTEXT_FILE")

coproc AGENT {
  convos agent serve "$CONV_ID" --env production --profile-name "OpenClaw Agent" 2>/dev/null
}

while IFS= read -r event <&"${AGENT[0]}"; do
  evt=$(echo "$event" | jq -r '.event // empty')

  case "$evt" in
    ready)
      MY_INBOX=$(echo "$event" | jq -r '.inboxId')
      echo "Ready: $CONV_ID" >&2

      # Prime the session with context (first message seeds identity + rules)
      openclaw agent \
        --session-id "$SESSION_ID" \
        --message "You are now active. Here is your full context: $CONTEXT" \
        >/dev/null 2>&1
      ;;

    message)
      type_id=$(echo "$event" | jq -r '.contentType.typeId // empty')
      [[ "$type_id" != "text" ]] && continue

      catchup=$(echo "$event" | jq -r '.catchup // false')
      [[ "$catchup" == "true" ]] && continue

      sender=$(echo "$event" | jq -r '.senderInboxId // empty')
      [[ "$sender" == "$MY_INBOX" ]] && continue

      content=$(echo "$event" | jq -r '.content')

      # Session already has context from the prime call — just pass the message
      reply=$(openclaw agent \
        --session-id "$SESSION_ID" \
        --message "$content" 2>/dev/null)

      jq -nc --arg text "$reply" '{"type":"send","text":$text}' >&"${AGENT[1]}"
      ;;

    member_joined)
      jq -nc '{"type":"send","text":"Welcome!"}' >&"${AGENT[1]}"
      ;;
  esac
done

wait "${AGENT_PID}"
```

## In-Conversation CLI Reference

Commands for reading and querying while participating. Always pass `--json` when parsing output programmatically.

### Members and Profiles

```bash
# Members (inbox IDs + permission levels)
convos conversation members "$CONV_ID" --json

# Profiles (display names + avatars)
convos conversation profiles "$CONV_ID" --json
```

Refresh profiles on `member_joined` events or `group_updated` messages about profile changes.

### Message History

```bash
# Recent messages (sync from network first)
convos conversation messages "$CONV_ID" --json --sync --limit 20

# Oldest first
convos conversation messages "$CONV_ID" --json --limit 50 --direction ascending

# Filter by type
convos conversation messages "$CONV_ID" --json --content-type text
convos conversation messages "$CONV_ID" --json --exclude-content-type reaction

# Time range (nanosecond timestamps)
convos conversation messages "$CONV_ID" --json --sent-after <ns> --sent-before <ns>
```

### Attachments

```bash
# Download an attachment
convos conversation download-attachment "$CONV_ID" <message-id>
convos conversation download-attachment "$CONV_ID" <message-id> --output ./photo.jpg

# Send an attachment (small files inline, large files auto-uploaded)
convos conversation send-attachment "$CONV_ID" ./photo.jpg
```

### Profile Management

Profiles are per-conversation — different name and avatar in each group.

```bash
# Set display name
convos conversation update-profile "$CONV_ID" --name "New Name"

# Set name and avatar
convos conversation update-profile "$CONV_ID" --name "New Name" --image "https://example.com/avatar.jpg"

# Go anonymous
convos conversation update-profile "$CONV_ID" --name "" --image ""
```

### Group Management

```bash
# View group info
convos conversation info "$CONV_ID" --json

# View permissions
convos conversation permissions "$CONV_ID" --json

# Update group name
convos conversation update-name "$CONV_ID" "New Name"

# Update description
convos conversation update-description "$CONV_ID" "New description"

# Add/remove members (requires super admin)
convos conversation add-members "$CONV_ID" <inbox-id>
convos conversation remove-members "$CONV_ID" <inbox-id>

# Lock (prevent new joins, invalidate existing invites)
convos conversation lock "$CONV_ID"

# Unlock
convos conversation lock "$CONV_ID" --unlock

# Permanently destroy conversation (irreversible)
convos conversation explode "$CONV_ID" --force
```

### Send Messages (CLI)

For scripting or one-off sends outside of agent mode:

```bash
# Send text
convos conversation send-text "$CONV_ID" "Hello!"

# Reply to a message
convos conversation send-reply "$CONV_ID" <message-id> "Replying to you"

# React
convos conversation send-reaction "$CONV_ID" <message-id> add "👍"

# Remove reaction
convos conversation send-reaction "$CONV_ID" <message-id> remove "👍"
```

## Common Mistakes

| Mistake | Why it's wrong | Correct approach |
|---------|---------------|-----------------|
| Running `agent serve` without a conversation ID or `--name` | The command requires one or the other. It will fail with neither | Pass a conversation ID to join existing, or `--name` to create new |
| Manually polling `agent serve` and sending messages separately | Creates race conditions, you'll miss messages between polls | Write and run a bridge script that uses coprocess stdin/stdout |
| Calling AI sub-session without context | Sub-session doesn't know who it is, what conversation it's in, or what commands exist | Write a context file with the full skill + runtime context, pass it on every call |
| Using markdown in messages | Convos does not render markdown. Users see raw `**asterisks**` and `[brackets](url)` | Write plain text naturally |
| Sending via CLI while in agent mode | Agent serve owns the conversation stream. CLI sends create race conditions | Use stdin commands (`{"type":"send",...}`) in agent mode |
| Forgetting `--env production` | Default is `dev` (test network). Real users are on production | Always pass `--env production` for real conversations |
| Replying to system events | `group_updated`, `reaction`, and `member_joined` are not messages | Only `replyTo` messages with `contentType.typeId` of `text`, `reply`, or `attachment` |
| Generating invite but not processing joins | The invite system is two-step: generate, then process | Run `process-join-requests` after the invitee opens the link |
| Referencing inbox IDs in chat | People don't know or care about `0x...` hex strings | Fetch profiles and use display names |
| Announcing tool usage | "Let me check the message history..." breaks immersion | Just do it silently and respond naturally |
| Responding to every message | Agents that talk too much get muted | Only speak when it adds something. React instead of replying when possible |

## Troubleshooting

**`convos: command not found`**
Not installed. Run `npm install -g @xmtp/convos-cli`.

**`Error: Not initialized`**
Run `convos init --env production` to create the configuration directory.

**Join request times out**
The invitee must open/scan the invite URL *before* the creator processes requests. If using `--no-wait`, run `convos conversations process-join-requests --watch --conversation <id>` separately.

**Messages not appearing**
Sync from the network first: `convos conversation messages <id> --json --sync --limit 20`.

**Permission denied on group operations**
Check permissions with `convos conversation permissions <id> --json`. Only super admins can add/remove members, lock, or explode.

**Invite expired or invalid**
Generate a new invite with `convos conversation invite <id>`. Locking a conversation invalidates all existing invites.

**Agent serve exits unexpectedly**
Check stderr for error output. Common causes: invalid conversation ID, identity not found (run `convos identity list`), or network issues. Use `--heartbeat 30` to monitor connection health.

## Tips

- **Use `--env production` for real conversations.** The default is `dev`, which uses the test network.
- **Use `--json` when parsing output.** Human-readable output can change between versions.
- **Use `--sync` before reading messages.** Ensures fresh data from the network.
- **Identities are automatic.** Creating or joining a conversation creates one for you. Rarely need to manage them directly.
- **Show full QR code output.** When generating invites, display the complete unmodified output so QR codes render correctly in the terminal. In agent mode, the QR code is saved as a PNG (path in the `ready` event's `qrCodePath` field).
- **Process join requests after the invite is opened.** Use `--watch` if you don't know when someone will open the invite.
- **Lock before exploding.** Lock a conversation first to prevent new joins, then explode when ready.
