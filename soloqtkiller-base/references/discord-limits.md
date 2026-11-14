# Discord Limits and Capacity Reference

Use this reference when a SoloQTKiller-base bot creates or manages Discord resources, especially ticket channels, ticket threads, cart threads, calls, roles, application commands, emojis, webhooks, invites, forum posts, and repeated API writes.

Limits can change. For work that depends on an exact cap, verify the current Discord developer docs or support article before shipping. The numbers below reflect the official docs/support pages checked on 2026-09-12 and the operational guards used by the Moreira-Sensi bot.

## Server Structure Limits

- Guild channels: 500 total, including text channels, voice channels, and categories.
- Categories: 50 per server.
- Channels inside one category: 50.
- Roles: 250 per server.
- Role name length: 100 characters.
- Unique invite codes: 999 per server.
- Audit log retention visible in Discord: 45 days.
- Server emojis: 50 on base servers, 100 at boost level 1, 150 at boost level 2, and 250 at boost level 3.
- Application emoji upload size: 256 KiB per static or animated emoji file.
- Webhooks: 1000 per guild. Do not create one webhook per user, order, or ticket.
- Channel permission overwrites: 1000 per channel. Prefer category-level synced permissions, staff roles, and threads over accumulating user-specific overwrites on long-lived channels.

## Thread and Forum Limits

- Threads do not count toward the guild's 500-channel limit, but Discord still caps active threads per guild. The Moreira-Sensi base uses an internal guard of 1000 active threads/topics per guild before creating more.
- Thread members: 1000 members in a public or private thread.
- Private-thread role mentions: up to 250 roles.
- Thread and channel names: 1 to 100 characters.
- Thread `autoArchiveDuration`: use Discord-supported values such as 60, 1440, 4320, or 10080 minutes.
- Forum/media channel tags: up to 20 available tags.
- Applied tags on a forum/media thread: up to 5.
- Forum tag names: up to 20 characters.
- Channel topics: 1024 characters for most text/announcement channels; 4096 characters for forum/media channels.
- Slowmode/rate limit per user: 0 to 21600 seconds. On channels that support thread creation, this can also affect thread creation cadence.

## Command and Component Limits

- Global chat input slash commands: 100 per application.
- Global user commands: 15 per application.
- Global message commands: 15 per application.
- Guild-specific commands: same per-guild quantities for command types except primary entry point.
- Application command creates: 200 per day per guild.
- Slash command name: 1 to 32 characters.
- Slash command description: 1 to 100 characters.
- Options per command/subcommand/subcommand group level: 25.
- Choices per option: 25.
- Total application command payload text budget: 8000 characters across names, descriptions, values, options, choices, and longest localizations.
- Component `custom_id`: 1 to 100 characters and unique within the same message.
- Button label: up to 80 characters.
- Action row: up to 5 buttons, or one select menu.
- String select options: up to 25.
- Select placeholder: up to 150 characters.
- Select min/max values: maximum 25.

## API Rate Limits

- Do not hardcode most per-route limits. Discord expects clients to honor rate-limit headers and `retry_after`.
- Global bot REST limit: commonly 50 requests per second, with interaction endpoints treated separately by Discord.
- Invalid requests can trigger temporary API restrictions. Avoid repeated `401`, `403`, and `429` responses by validating token state, permissions, existing resources, and rate-limit buckets before retrying.
- Emoji routes have special rate limiting and may return inaccurate quota headers; keep emoji sync idempotent and avoid repeated full syncs during normal interaction handling.

## Required Tactics

- Preflight before creation: before creating a channel, category, role, thread, webhook, command, invite, or emoji, check whether the target guild/category/application can still accept it.
- Fail early: if a category already has 50 children, if the guild is near 500 channels, if the bot cannot manage the target role/channel, or if the queue is full, reply with a clear message instead of attempting the API call.
- Avoid overwrite growth: private ticket channels should keep overwrites minimal. When many users need temporary access, prefer private threads, staff roles, or cleanup on close instead of leaving user overwrites behind.
- Use operation locks for double-click protection: create a short-lived database record with a unique operation key such as `guildId:panelId:userId` for tickets or `guildId:userId` for carts before expensive work starts.
- Use queues for high-impact flows: ticket and cart creation should enforce per-guild queue limits, global queue limits, and active-operation limits so bursts do not flood Discord or overload the bot.
- Use per-user cooldowns after failed operations: a user repeatedly clicking the same button/select should receive a friendly wait message instead of creating repeated database/API work.
- Keep operation TTLs short: stale `TicketOperation` or `CartOperation` records should expire automatically and be cleaned by maintenance.
- Count active records and active Discord resources: compare database counts for open/opening/closing tickets or carts with current guild channel/thread state, then reconcile ghost records when Discord resources disappeared.
- Cache expensive counts briefly: active-thread counting can be cached for a few seconds to reduce REST calls during bursts, but the cache must be short-lived enough not to hide a full server for long.
- Wrap Discord writes with backoff: channel/thread creation, command registration, panel publishing, and emoji sync should use a helper that waits for `retry_after` on `429` instead of retrying immediately.
- Keep writes idempotent: if a retry happens after a partial success, use saved message/channel/thread IDs and database operation records to continue or clean up without duplicating resources.
- Prefer reuse over resource creation: use slash subcommands, select options, database rows, Components V2 panels, and threads instead of creating many top-level commands, channels, or roles.

## Internal Cooldown Recommendations

These are not all Discord hard limits; they are base defaults that prevent repeated clicks, API bursts, and expensive repeated work.

- Ticket/customer notification: at least 5 minutes per ticket.
- Counter/channel-name updates: at least 5 minutes between updates, with `nextUpdateAt` and `failureCount`.
- Follow-up automation: store `lastFollowUpAt`, `repeatMinutes`, and `maxFollowUps`.
- Ticket/cart creation: operation lock with a short TTL, per-guild queue cap, global queue cap, and per-user failure cooldown.
- Bot username changes: at least 2 days between successful changes.
- Bot avatar/banner changes: at least 6 hours between successful changes.
- Emoji sync: run at startup or explicit admin action, not on every interaction.
- Public panel repost/publish: avoid repeated immediate reposts; keep `messageId` and edit existing messages when possible.
- External APIs and payments: use idempotency keys, pending records, retry timestamps, and terminal states.

If a feature can be triggered by repeated button clicks or select submissions, store the cooldown/lock server-side. Do not rely only on disabled components in Discord messages.

## Ticket and Cart Capacity Pattern

For tickets, carts, purchase flows, and support flows, mirror the Moreira-Sensi shape:

- `operationCapacity.assertAvailable(client, guildId, userId)` before opening a ticket/cart.
- `TicketOperation` or `CartOperation` with a unique key and `expiresAt` before creating the Discord resource.
- `assertNoOpenTicket` or equivalent active-record check to stop duplicates.
- `assertCapacity` before creating a channel/thread, including active tickets, active threads, configured parent channel/category, category child count, and guild-level resource limits.
- `countGuildActiveThreads(guild)` with a short cache before creating private threads.
- `withBackoff(() => guild.channels.create(...))` or `withBackoff(() => parent.threads.create(...))` for Discord write operations.
- Cleanup on failure: mark operation failed or delete it, remove incomplete ticket/cart records, and delete any Discord resource created during a failed operation when safe.

The user-facing behavior should be calm: "Seu ticket já está sendo criado", "A fila deste servidor está cheia", "O limite de tópicos ativos foi atingido", or "A categoria configurada já atingiu o limite de canais do Discord." Do not expose raw Discord errors when the bot can explain the limit.

## Source Pointers

- Discord support server caps: `https://support.discord.com/hc/en-us/articles/33694251638295-Discord-Account-Caps-Server-Caps-and-More`
- Discord channel resource: `https://docs.discord.com/developers/resources/channel`
- Discord threads topic: `https://docs.discord.com/developers/topics/threads`
- Discord application commands: `https://docs.discord.com/developers/docs/interactions/slash-commands`
- Discord components reference: `https://docs.discord.com/developers/components/reference`
- Discord emoji resource: `https://docs.discord.com/developers/resources/emoji`
- Discord rate limits: `https://docs.discord.com/developers/topics/rate-limits`
