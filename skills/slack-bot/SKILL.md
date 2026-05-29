---
name: slack-bot
description: Send messages, add reactions, join channels, and search messages through Slack Web API using the local Slack bot token. Use when the user asks to interact with Slack via bot API calls. If Slack tools are available via `mcp__claude_ai_Slack__*`, prefer those, unless user asks for "Slack bot" interactions ("via bot").
---

# Slack bot interaction

Use Slack Web API directly with the bot token from the environment. Never print the token. Do not hardcode it.

Before calling Slack, verify the token is available without printing it:

```bash
if [ -z "${SLACK_BOT_XOXB_TOKEN:-}" ]; then
  echo "SLACK_BOT_XOXB_TOKEN is not set" >&2
  exit 1
fi
```

## Common response handling

Pipe Slack responses through `jq` and report either success or the Slack error:

```bash
| jq -r 'if .ok then "ok" else "error: " + .error end'
```

Example construction:

```bash
curl -sS -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer ${SLACK_BOT_XOXB_TOKEN}" \
  -H 'Content-type: application/json; charset=utf-8' \
  --data '{"channel":"C0123456789","text":"hello"}' \
  | jq -r 'if .ok then "ok ts=" + .ts else "error: " + .error end'
```

Safer JSON construction:

```bash
CHANNEL_ID="C0123456789"
MESSAGE="hi"

jq -n --arg channel "$CHANNEL_ID" --arg text "$MESSAGE" \
  '{channel: $channel, text: $text}' \
  | curl -sS -X POST https://slack.com/api/chat.postMessage \
      -H "Authorization: Bearer ${SLACK_BOT_XOXB_TOKEN}" \
      -H 'Content-type: application/json; charset=utf-8' \
      --data @- \
  | jq -r 'if .ok then "ok ts=" + .ts else "error: " + .error end'
```

## Methods

### chat.postMessage

Docs: <https://docs.slack.dev/reference/methods/chat.postMessage>

Input args:

- `channel`: channel, DM, or MPIM ID to send to.
- `text`: primary message text; required unless `blocks` or `attachments` are provided.
- `blocks`: structured Block Kit message content.
- `attachments`: legacy structured attachments.
- `thread_ts`: parent message timestamp when replying in a thread.
- `reply_broadcast`: also send a threaded reply to the channel.
- `mrkdwn`: enable or disable Slack markdown parsing.
- `unfurl_links`: enable or disable link unfurls.
- `unfurl_media`: enable or disable media unfurls.

Output params:

- `ok`: boolean success flag.
- `channel`: channel ID the message was posted to.
- `ts`: posted message timestamp; use this for threads or reactions.
- `message`: posted message object.
- `message.text`: posted message text.
- `error`: Slack error code when `ok` is false.

### reactions.add

Docs: <https://docs.slack.dev/reference/methods/reactions.add>

Input args:

- `channel`: channel ID containing the message.
- `timestamp`: target message `ts`.
- `name`: emoji name without colons, e.g. `eyes`.

Output params:

- `ok`: boolean success flag.
- `error`: Slack error code when `ok` is false.

### conversations.join

Docs: <https://docs.slack.dev/reference/methods/conversations.join>

Input args:

- `channel`: public channel ID to join.

Output params:

- `ok`: boolean success flag.
- `channel.id`: joined channel ID.
- `error`: Slack error code when `ok` is false.

### search.messages

Docs: <https://docs.slack.dev/reference/methods/search.messages>

Input args:

- `query`: Slack search query, e.g. `in:ask-enrichment enrichment after:2026-05-01`.
- `count`: max results to return.
- `page`: result page number.
- `sort`: `score` or `timestamp`.
- `sort_dir`: `asc` or `desc`.

Output params:

- `ok`: boolean success flag.
- `messages.matches[]`: matched messages.
- `messages.matches[].channel.name`: channel name.
- `messages.matches[].ts`: message timestamp.
- `messages.matches[].user`: author user ID.
- `messages.matches[].text`: message text.
- `error`: Slack error code when `ok` is false.

### conversations.history

Docs: <https://docs.slack.dev/reference/methods/conversations.history>

Input args:

- `channel`: channel ID to read.
- `limit`: max messages to return.
- `cursor`: pagination cursor from `response_metadata.next_cursor`.
- `oldest`: only messages after this timestamp.
- `latest`: only messages before this timestamp.
- `inclusive`: include messages matching `oldest` or `latest`.

Output params:

- `ok`: boolean success flag.
- `messages[]`: channel messages.
- `messages[].ts`: message timestamp.
- `messages[].user`: author user ID.
- `messages[].text`: message text.
- `has_more`: whether more results exist.
- `response_metadata.next_cursor`: cursor for next page.
- `error`: Slack error code when `ok` is false.

### conversations.replies

Docs: <https://docs.slack.dev/reference/methods/conversations.replies>

Input args:

- `channel`: channel ID containing the thread.
- `ts`: parent message timestamp.
- `limit`: max messages to return.
- `cursor`: pagination cursor from `response_metadata.next_cursor`.
- `oldest`: only replies after this timestamp.
- `latest`: only replies before this timestamp.
- `inclusive`: include replies matching `oldest` or `latest`.

Output params:

- `ok`: boolean success flag.
- `messages[]`: parent message and replies.
- `messages[].ts`: message timestamp.
- `messages[].user`: author user ID.
- `messages[].text`: message text.
- `has_more`: whether more results exist.
- `response_metadata.next_cursor`: cursor for next page.
- `error`: Slack error code when `ok` is false.

### auth.test

Docs: <https://docs.slack.dev/reference/methods/auth.test>

Use to verify the token and resolve workspace/bot identity before debugging auth issues.

Input args: none besides token.

Output params:

- `ok`: boolean success flag.
- `url`: workspace URL.
- `team`: workspace name.
- `team_id`: workspace ID.
- `user`: authenticated user or bot name.
- `user_id`: authenticated user or bot user ID.
- `bot_id`: bot ID for bot tokens.
- `error`: Slack error code when `ok` is false.

### conversations.list

Docs: <https://docs.slack.dev/reference/methods/conversations.list>

Use to discover public channels, private channels, DMs, and MPIMs visible to the bot.

Input args:

- `types`: comma-separated conversation types, e.g. `public_channel,private_channel,mpim,im`.
- `exclude_archived`: exclude archived conversations.
- `limit`: max conversations to return.
- `cursor`: pagination cursor from `response_metadata.next_cursor`.
- `team_id`: workspace ID for org-level tokens.

Output params:

- `ok`: boolean success flag.
- `channels[]`: visible conversations.
- `channels[].id`: conversation ID.
- `channels[].name`: channel or MPIM name.
- `channels[].user`: DM user ID for IM conversations.
- `channels[].is_member`: whether the bot is a member, when present.
- `response_metadata.next_cursor`: cursor for next page.
- `error`: Slack error code when `ok` is false.

### conversations.info

Docs: <https://docs.slack.dev/reference/methods/conversations.info>

Use to resolve channel metadata from a conversation ID.

Input args:

- `channel`: conversation ID.
- `include_num_members`: include member count.
- `include_locale`: include locale.

Output params:

- `ok`: boolean success flag.
- `channel`: conversation object.
- `channel.id`: conversation ID.
- `channel.name`: channel name, when applicable.
- `channel.user`: DM user ID, when applicable.
- `channel.topic.value`: topic text.
- `channel.purpose.value`: purpose text.
- `channel.num_members`: member count, when requested and available.
- `error`: Slack error code when `ok` is false.

### conversations.members

Docs: <https://docs.slack.dev/reference/methods/conversations.members>

Use to list user IDs in a conversation.

Input args:

- `channel`: conversation ID.
- `limit`: max users to return.
- `cursor`: pagination cursor from `response_metadata.next_cursor`.

Output params:

- `ok`: boolean success flag.
- `members[]`: user IDs in the conversation.
- `response_metadata.next_cursor`: cursor for next page.
- `error`: Slack error code when `ok` is false.

### users.conversations

Docs: <https://docs.slack.dev/reference/methods/users.conversations>

Use to list conversations the bot can access, optionally filtered by shared membership with a user.

Input args:

- `user`: user ID to filter by shared conversations.
- `types`: comma-separated conversation types, e.g. `public_channel,private_channel,mpim,im`.
- `exclude_archived`: exclude archived conversations.
- `limit`: max conversations to return.
- `cursor`: pagination cursor from `response_metadata.next_cursor`.
- `team_id`: workspace ID for org-level tokens.

Output params:

- `ok`: boolean success flag.
- `channels[]`: visible conversations.
- `channels[].id`: conversation ID.
- `channels[].name`: channel or MPIM name.
- `channels[].user`: DM user ID for IM conversations.
- `response_metadata.next_cursor`: cursor for next page.
- `error`: Slack error code when `ok` is false.

### conversations.open

Docs: <https://docs.slack.dev/reference/methods/conversations.open>

Use to open or resume a DM or MPIM before posting.

Input args:

- `users`: comma-separated user IDs; one user opens a DM, multiple users open an MPIM.
- `channel`: existing IM or MPIM ID to resume instead of `users`.
- `return_im`: include full IM conversation object.
- `prevent_creation`: only return existing conversations.

Output params:

- `ok`: boolean success flag.
- `channel.id`: DM or MPIM conversation ID.
- `channel`: full conversation object when `return_im` is true.
- `already_open`: whether the conversation already existed.
- `no_op`: whether no new conversation was created.
- `error`: Slack error code when `ok` is false.

### users.list

Docs: <https://docs.slack.dev/reference/methods/users.list>

Use to resolve Slack users by name, display name, or profile fields.

Input args:

- `limit`: max users to return.
- `cursor`: pagination cursor from `response_metadata.next_cursor`.
- `include_locale`: include locale.
- `team_id`: workspace ID for org-level tokens.

Output params:

- `ok`: boolean success flag.
- `members[]`: user objects.
- `members[].id`: user ID.
- `members[].name`: username.
- `members[].real_name`: real name.
- `members[].profile.display_name`: display name.
- `members[].deleted`: whether user is deactivated.
- `response_metadata.next_cursor`: cursor for next page.
- `error`: Slack error code when `ok` is false.

Note: `profile.email` requires the extra `users:read.email` scope.

### users.info

Docs: <https://docs.slack.dev/reference/methods/users.info>

Use to resolve details for a known user ID.

Input args:

- `user`: user ID.
- `include_locale`: include locale.

Output params:

- `ok`: boolean success flag.
- `user`: user object.
- `user.id`: user ID.
- `user.name`: username.
- `user.real_name`: real name.
- `user.profile.display_name`: display name.
- `user.deleted`: whether user is deactivated.
- `error`: Slack error code when `ok` is false.

Note: `profile.email` requires the extra `users:read.email` scope.

### usergroups.list

Docs: <https://docs.slack.dev/reference/methods/usergroups.list>

Use to list Slack user groups.

Input args:

- `include_users`: include user IDs for each group.
- `include_count`: include user count.
- `include_disabled`: include disabled user groups.
- `team_id`: workspace ID for org-level tokens.

Output params:

- `ok`: boolean success flag.
- `usergroups[]`: user group objects.
- `usergroups[].id`: user group ID.
- `usergroups[].handle`: mention handle.
- `usergroups[].name`: display name.
- `usergroups[].users`: user IDs when `include_users` is true.
- `usergroups[].user_count`: user count when requested.
- `error`: Slack error code when `ok` is false.

### usergroups.users.list

Docs: <https://docs.slack.dev/reference/methods/usergroups.users.list>

Use to list users in a Slack user group.

Input args:

- `usergroup`: user group ID.
- `include_disabled`: include disabled users.
- `team_id`: workspace ID for org-level tokens.

Output params:

- `ok`: boolean success flag.
- `users[]`: user IDs in the user group.
- `error`: Slack error code when `ok` is false.

### chat.getPermalink

Docs: <https://docs.slack.dev/reference/methods/chat.getPermalink>

Use to turn a `channel` + message timestamp into a Slack URL.

Input args:

- `channel`: conversation ID containing the message.
- `message_ts`: message timestamp.

Output params:

- `ok`: boolean success flag.
- `channel`: conversation ID.
- `permalink`: Slack URL for the message.
- `error`: Slack error code when `ok` is false.

### chat.update

Docs: <https://docs.slack.dev/reference/methods/chat.update>

Use to edit messages posted by the bot.

Input args:

- `channel`: conversation ID containing the message.
- `ts`: timestamp of the message to update.
- `text`: replacement message text.
- `blocks`: replacement Block Kit content.
- `attachments`: replacement attachments.
- `reply_broadcast`: broadcast an existing thread reply.

Output params:

- `ok`: boolean success flag.
- `channel`: conversation ID.
- `ts`: updated message timestamp.
- `text`: updated text.
- `message`: updated message object.
- `error`: Slack error code when `ok` is false.

### chat.delete

Docs: <https://docs.slack.dev/reference/methods/chat.delete>

Use to delete messages posted by the bot.

Input args:

- `channel`: conversation ID containing the message.
- `ts`: timestamp of the message to delete.

Output params:

- `ok`: boolean success flag.
- `channel`: conversation ID.
- `ts`: deleted message timestamp.
- `error`: Slack error code when `ok` is false.

### chat.postEphemeral

Docs: <https://docs.slack.dev/reference/methods/chat.postEphemeral>

Use to send a message visible only to one user in a conversation. Delivery is not guaranteed; the user must be active and in the conversation.

Input args:

- `channel`: conversation ID.
- `user`: recipient user ID.
- `text`: message text.
- `blocks`: Block Kit content.
- `attachments`: attachments.
- `thread_ts`: parent message timestamp for an ephemeral thread reply.

Output params:

- `ok`: boolean success flag.
- `message_ts`: ephemeral message timestamp; cannot be used with `chat.update`.
- `error`: Slack error code when `ok` is false.

### chat.scheduleMessage

Docs: <https://docs.slack.dev/reference/methods/chat.scheduleMessage>

Use to schedule a future message. Do not use `metadata`; Slack notes scheduled messages with metadata may not post.

Input args:

- `channel`: conversation ID, channel name, or DM ID.
- `post_at`: Unix timestamp for future delivery.
- `text`: message text.
- `blocks`: Block Kit content.
- `attachments`: attachments.
- `thread_ts`: parent message timestamp for a scheduled thread reply.
- `reply_broadcast`: broadcast a scheduled thread reply.
- `unfurl_links`: enable link unfurls.
- `unfurl_media`: enable media unfurls.

Output params:

- `ok`: boolean success flag.
- `channel`: conversation ID.
- `scheduled_message_id`: ID used for deletion.
- `post_at`: scheduled Unix timestamp.
- `message`: scheduled message object.
- `error`: Slack error code when `ok` is false.

### chat.scheduledMessages.list

Docs: <https://docs.slack.dev/reference/methods/chat.scheduledMessages.list>

Use to inspect pending scheduled messages.

Input args:

- `channel`: filter by channel.
- `oldest`: earliest Unix timestamp.
- `latest`: latest Unix timestamp.
- `limit`: max scheduled messages to return.
- `cursor`: pagination cursor from `response_metadata.next_cursor`.
- `team_id`: workspace ID for org-level tokens.

Output params:

- `ok`: boolean success flag.
- `scheduled_messages[]`: pending scheduled messages.
- `scheduled_messages[].id`: scheduled message ID.
- `scheduled_messages[].channel_id`: destination channel ID.
- `scheduled_messages[].post_at`: scheduled Unix timestamp.
- `scheduled_messages[].text`: scheduled message text.
- `response_metadata.next_cursor`: cursor for next page.
- `error`: Slack error code when `ok` is false.

### chat.deleteScheduledMessage

Docs: <https://docs.slack.dev/reference/methods/chat.deleteScheduledMessage>

Use to cancel a pending scheduled message before it is sent. Slack does not allow deletion within 60 seconds of post time.

Input args:

- `channel`: conversation ID where the message is scheduled.
- `scheduled_message_id`: ID returned by `chat.scheduleMessage` or `chat.scheduledMessages.list`.

Output params:

- `ok`: boolean success flag.
- `error`: Slack error code when `ok` is false.

### Other useful methods

Docs: <https://docs.slack.dev/reference/methods>

Use the Slack methods index when a task needs an endpoint not listed above. Check the method docs for required scopes and bot-token support before calling it.

Potentially useful, depending on task and scopes:

- `reactions.get`: inspect reactions on a message; requires `reactions:read`.
- `users.lookupByEmail`: resolve user by email; requires `users:read.email` and may not support custom bot users.
- `users.profile.get`: read a user's profile fields.
- `conversations.create`: create public/private channels; admin/destructive, ask first.
- `conversations.invite`: invite users to a channel; ask first.
- `conversations.rename`: rename a channel; destructive, ask first.
- `conversations.setTopic`: set channel topic; ask first.
- `conversations.setPurpose`: set channel purpose; ask first.
- `conversations.archive` / `conversations.unarchive`: archive state changes; destructive, ask first.
- `usergroups.create` / `usergroups.update` / `usergroups.disable` / `usergroups.enable`: manage user groups; admin/destructive, ask first.
- `usergroups.users.update`: replace user group membership; destructive, ask first.
- Pagination patterns with `cursor` / `next_cursor`.
