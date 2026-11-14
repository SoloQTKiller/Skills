# Data Integrity, Permissions, and Retention Reference

Use this reference whenever a SoloQTKiller-base bot changes domain state, creates Discord resources, writes logs/history, stores temporary workflow data, reloads cached config, or changes schemas. These rules were extracted from repeated requirements in the Moreira-Sensi bot.

## Domain States

Every persistent workflow should expose a clear `status` and terminal states.

Recommended patterns:

- Tickets: `opening`, `open`, `closing`, `closed`, `orphaned`.
- Carts/orders: `open`, `awaiting_payment`, `paid`, `delivered`, `cancelled`, `expired`, `orphaned`.
- Operation locks: `processing`, `completed`, `failed`.
- Temporary workflows: `pending`, `submitted`, `expired`.
- Managed Discord resources: `active`, `disabled`, `missing`, `stale`, `orphaned`.

Avoid modeling lifecycle only with booleans such as `isOpen`, `isClosed`, `deleted`, and `finished`. Booleans become unclear when the bot crashes halfway through a flow, when a Discord resource is deleted manually, or when a migration needs to decide what should be cleaned.

Keep domain transitions in services when multiple entrypoints can change the same state. A button, modal submit, slash command, background job, and webhook should call the same domain helper rather than each one mutating `status` directly.

## Crash Recovery Plan

Every feature with multi-step side effects should define what happens if the bot restarts halfway through the operation.

Recovery candidates:

- `TicketOperation`, `CartOperation`, or other operation locks stuck in `processing`.
- Tickets or carts with Discord channels/threads created but database status not finalized.
- Orders with payment created but not expired, delivered, cancelled, or confirmed.
- Stock/coupon reservations past `expiresAt`.
- Pending workflows waiting for user upload or import.
- Scheduled jobs interrupted after partial work.
- Managed panel messages, counters, roles, webhooks, commands, AutoMod rules, and application emojis that may have changed while the bot was offline.

Recommended startup behavior:

- Run bounded reconciliation after `clientReady` and database connection.
- Expire or fail stale operation locks using their `expiresAt` or max age.
- Re-fetch Discord resources before trying to reuse stored ids.
- Release expired reservations and clear temporary workflows.
- Mark impossible states as `orphaned`, `failed`, `expired`, or `needs_review`.
- Write private/admin logs for recovered or suspicious states.
- Keep recovery idempotent so it can run safely more than once.

## Feature Flags and Gradual Configuration

Large or risky systems should be controlled by configuration flags.

Recommended behavior:

- Default new high-impact systems to disabled or admin-only when appropriate.
- Store flags per guild or per feature, such as `tickets.enabled`, `sales.enabled`, `counters.enabled`, `automations.enabled`, or `payments.providerEnabled`.
- Show enabled/disabled state, disabled reason, and rollout warnings in the management panel.
- Check the flag in every entrypoint, not only in the panel renderer.
- Allow partial rollout by guild, role, channel, or panel when a feature has risk or needs testing.
- Keep safe fallbacks while a migration/backfill is incomplete.

Do not hide a feature behind only a code constant when admins need to manage it. Persist the flag in config and refresh cache after changes.

## Scheduled Jobs With Locks

Jobs that can overlap or duplicate side effects need a lock.

Jobs that usually need locks:

- Counter/channel rename jobs.
- Automatic messages and follow-ups.
- Payment expiration and delivery.
- Ticket/cart lifecycle cleanup.
- Audit pruning and retention cleanup.
- Backfills and migrations.
- Emoji/command synchronization when it can be triggered manually and automatically.

Recommended lock fields:

- `jobName`.
- `guildId` when the job is guild-scoped.
- `lockedAt`.
- `lockedBy`.
- `expiresAt`.
- `lastRunAt`.
- `nextRunAt`.
- `failureCount`.
- `status`.

Locks should expire automatically or be recoverable after crash. A stale lock should not block maintenance forever.

## Friendly Error Pattern

Users should receive short, safe, actionable errors. Developers/admins should receive technical context privately.

Recommended behavior:

- Throw expected/domain errors with friendly messages for validation, permission, cooldown, capacity, and missing resource cases.
- Log unexpected errors privately with context such as `guildId`, `userId`, `channelId`, `messageId`, `orderId`, `ticketId`, `operationKey`, `jobName`, and `customId`.
- Do not expose stack traces, database documents, provider payloads, tokens, or internal paths in user-facing replies.
- Use the shared Components V2/status payload style for error replies.
- Treat expired interactions (`10062`) and deleted resources (`10003`) as non-actionable when the flow itself may delete the channel/message.

## Counters and Placeholders

Counter and placeholder helpers are critical because they touch visible channel names and reusable messages.

Counter rules:

- Validate the counter type against an allowlist.
- Require at least one valid placeholder in the template.
- Cap rendered channel names to Discord's 100-character channel-name limit.
- Enforce a minimum update interval such as 5 minutes.
- Store `lastValue`, `lastUpdatedAt`, `nextUpdateAt`, `failureCount`, `disabledReason`, and `lastManualUpdateAt`.
- Disable counters after repeated failures instead of retrying forever.
- Prevent two counters from managing the same channel.
- Use a job lock if the maintenance job can overlap.

Placeholder rules:

- Keep placeholders explicit and documented, such as `@user`, `<user.id>`, `<user>`, `<server>`, `{valor}`, `{membros}`, `{call}`, `{clientes}`, `{vendas}`, `{itens}`, `{faturamento}`, `{avaliacoes}`, and `{servidor}`.
- Do not evaluate arbitrary code, templates, or user-provided expressions.
- Use focused tests for placeholder replacement.
- Keep user-facing placeholders stable to avoid breaking existing configured messages.

## Orphan Reconciliation

Any feature that stores a Discord id must be able to handle that resource disappearing outside the bot.

Resources to reconcile:

- Channels and categories.
- Threads/topics and forum posts.
- Published panel messages.
- Roles.
- Webhooks.
- AutoMod rules.
- Application commands.
- Application emojis.
- Log channels.

Recommended reconciliation behavior:

- Re-fetch the Discord resource before editing/deleting it.
- If the resource is missing, mark the local record as `orphaned`, `missing`, `stale`, or `disabled`.
- Clear fields that would make future operations unsafe, such as active `channelId`, `messageId`, `threadId`, or `webhookId`.
- Preserve enough metadata for admin diagnosis, such as `orphanedAt`, `orphanReason`, `lastKnownChannelId`, or `lastKnownMessageId`.
- Set `cleanupAt` for temporary records that should disappear after the admin has had time to inspect them.
- Write a private/admin audit entry; do not spam public sales/support logs with internal reconciliation noise.
- Refresh the central or child management panel so admins see the real state.

## Layered Permissions

Permission checks should be centralized and layered.

Recommended layers:

- Discord platform permission, such as Administrator, ManageGuild, ManageChannels, ManageRoles, or ManageMessages.
- Configured admin/staff role or user allowlist from guild config.
- Feature-specific permission, such as ticket staff, sales manager, backup manager, fraud reviewer, or bot identity manager.
- Resource ownership, such as the user who owns a ticket/cart or the panel that owns a message.
- Confirmation-stage permission for destructive or high-impact actions.

Re-check permissions on modal submit and confirmation buttons. A user can open a modal while authorized and lose the role before submitting it.

Keep helpers separate by meaning. A fixed administrator, guild owner, Discord administrator, configured staff member, and resource owner are not interchangeable:

```js
const isFixedOwner = (client, userId) => userId === client.config.fixedOwnerId;
const isGuildOwnerOrAdministrator = (interaction) =>
  interaction.guild?.ownerId === interaction.user.id ||
  interaction.member?.permissions?.has(PermissionFlagsBits.Administrator);
```

Compose feature access deliberately. A fixed administrator may manage the central panel without guild admin while purchasing may remain guild-owner/admin-only. Diagnostic or manual-balance commands may be fixed-administrator-only even for a guild owner. Test these negative cases explicitly.

## Public Logs vs Private Logs

Use different log surfaces for different audiences.

Public/customer-facing logs may include:

- Purchase completed summaries.
- Ticket opened/closed summaries.
- Delivery completed summaries.
- Safe status changes.

Private/admin logs may include:

- Failed Discord API calls.
- Orphan reconciliation.
- Payment diagnostics.
- Fraud and restriction evidence.
- Queue saturation.
- Backoff/rate-limit events.
- Stack summaries and job failures.

Never expose tokens, secrets, raw payment payloads, private evidence attachments, full customer payloads, database dumps, or bot credentials in public channels, embeds, transcripts, panels, or backups.

## Critical Helper Tests

Focused automated tests should be added whenever practical for helpers that affect:

- Money formatting and totals.
- Stock reservation and release.
- Coupons and usage limits.
- Permission decisions.
- Domain state transitions.
- Ticket/cart duplicate prevention.
- Discord limit/capacity guards.
- Placeholder replacement.
- Cache reload/invalidation behavior.
- TTL cleanup and retention cutoffs.
- Orphan reconciliation decisions.

Prefer testing pure helpers directly. If a helper needs Discord or MongoDB, isolate the external boundary with lightweight fakes so the test validates business behavior without needing a live guild.

## Migrations and Backfills

Schema changes that affect existing production records need a migration/backfill strategy.

Before creating a migration, verify that legacy production data exists. If a bot has not launched and has no records to transform, change schemas/defaults/indexes directly and do not create migration-history noise.

Recommended migration rules:

- Make new fields backwards-compatible with defaults.
- Write panels so they can render old records while backfill is still running.
- Make backfills idempotent; running the same script/job twice should not duplicate records or corrupt state.
- Process in batches for large collections.
- Log progress and failures privately.
- Avoid deleting old fields until every handler and renderer no longer depends on them.
- Rebuild or invalidate affected caches after backfill.
- Add temporary compatibility reads when old and new schemas must coexist during rollout.

## Cache Policy

Any configurable system needs a clear cache policy.

Recommended behavior:

- Read panel state from MongoDB-backed config through `client.configs` or the project's local cache layer.
- After a mutation, call `client.reloadConfigs(guildId)` or invalidate the specific cache key before rendering the next panel.
- Do not reply "configuração salva" until the database mutation and cache refresh both succeed.
- Keep short-lived resource counts, such as active thread totals, in memory only with a small TTL such as 15 seconds.
- Cache expensive reads, but never let cache hide a destructive or permission-sensitive state change.
- Clear or refresh cache after migrations/backfills that modify cached fields.

## TTL and Retention Policy

Collections that can grow quickly must define a retention policy before the feature is considered complete.

High-volume or temporary records should normally use one of these patterns:

- `expiresAt` plus a TTL index for short-lived locks, drafts, pending workflows, imports, and idempotency records.
- `cleanupAt` plus a TTL index for business records that should remain until a terminal state, such as closed tickets, expired carts, released reservations, old transcript metadata, or temporary evidence.
- Scheduled pruning for audit logs that need retention by event type instead of one global TTL.
- Compact permanent summaries for accounting, security, or delivery records that must outlive bulky transient data.

Common candidates for TTL/retention:

- `AuditLog` and event history.
- `TicketOperation` and `CartOperation`.
- `PendingWorkflow`.
- Stock/coupon reservations.
- Temporary backup/import previews.
- Automatic message deliveries.
- Report deliveries.
- Review prompts.
- Fraud evidence and restriction attempts.
- Transcript metadata after the retention period ends.
- Payment confirmations, delivery proofs, order summaries, invoice-like records, and accounting/security evidence.

TTL rules:

- Never let audit/history collections grow forever by default.
- Pick retention per business value: seconds/minutes for locks, hours/days for drafts/imports, months for support/debug logs, years only for accounting/security summaries that truly need it.
- Treat proofs differently from noisy logs. Payment confirmations, delivery confirmations, delivered order summaries, and invoice-like records can require retention such as 5 years; keep a compact record with ids, user, order, amount, timestamps, delivery result, and provider reference, while pruning raw payloads, screenshots, Discord transcript HTML, and temporary diagnostics earlier when they are not required.
- Use partial TTL indexes for optional `cleanupAt` fields so active records without cleanup date are not removed accidentally.
- Set `cleanupAt` only after the record reaches a safe terminal state.
- Keep a maintenance job for records that need event-type retention or data slimming before deletion.
- Avoid storing large raw payloads in audit logs. Store ids, status, reason, and compact summaries instead.
- Give match/room/game data a bounded project-defined retention window; 60 days is a useful maximum when older match data has no support or accounting value.
- Permanent event-deduplication keys are different from operational records. If one source message must never be processed twice, its unique lock must outlive the related operation record.

## MongoDB Connection Recovery

- Use `serverSelectionTimeoutMS: 60000` and `socketTimeoutMS: 45000` from central configuration.
- Treat `ECONNRESET`, TLS handshake resets, and transient server-selection failures as recoverable during startup/reconnect.
- Log each reconnect attempt without exposing the URI, use progressive bounded delay, and stop after at most five attempts unless the project explicitly chooses another bound.
- Keep maintenance jobs single-flight while MongoDB is degraded.
- After the final failure, terminate in a controlled way with a concise reason instead of an unhandled silent crash.

## Allowed-Guild List

Use `ALLOWED_GUILDS_ID` for bots restricted to a known set of guilds. The value is a comma-separated list of Discord guild IDs:

```env
ALLOWED_GUILDS_ID=1343014969851121756,123456789012345678
```

Rules:

- Do not use the singular legacy name `ALLOWED_GUILD_ID` in new or updated bots.
- Split on commas, trim whitespace, remove empty entries, remove duplicates, and validate every item as a 17–20 digit snowflake.
- Startup must fail clearly when the variable is absent, empty, or contains any invalid ID. Never silently discard an invalid configured entry.
- Store the normalized result as an immutable array such as `client.config.allowedGuildIds`.
- Register guild commands and load guild configuration for every allowed guild where the bot is present.
- Remove guild-scoped commands from connected guilds not present in the allowlist.
- Reject or ignore interactions, messages, cache loads, API operations, and guild-scoped jobs for foreign guilds.
- Maintenance queries must cover the complete list, normally with `{ guildId: { $in: allowedGuildIds } }`; never keep using only the first configured guild.
- If the bot is missing from one configured guild, report the exact missing guild and follow the project's explicit startup policy; do not accidentally authorize a different guild.
- Test missing, empty, malformed, duplicate, whitespace, multiple-valid, matching, and foreign guild cases, plus command registration and maintenance across more than one allowed guild.

Example parser:

```js
const snowflake = /^\d{17,20}$/;

function parseAllowedGuildIds(raw) {
  const ids = [...new Set(String(raw || "").split(",").map((id) => id.trim()).filter(Boolean))];
  if (!ids.length || ids.some((id) => !snowflake.test(id)))
    throw new Error("ALLOWED_GUILDS_ID deve conter IDs de servidores válidos separados por vírgula.");
  return Object.freeze(ids);
}
```

Example TTL indexes:

```js
operationSchema.index({ expiresAt: 1 }, { expireAfterSeconds: 0 });

ticketSchema.index(
  { cleanupAt: 1 },
  {
    expireAfterSeconds: 0,
    partialFilterExpression: { cleanupAt: { $type: "date" } },
  },
);
```

For audit logs with different retention per event type, prefer a pruning helper similar to the Moreira-Sensi audit service: keep a map of event type to retention duration, delete disabled/noisy event types, and run the prune job from maintenance/lifecycle code.

## Five-Year Proof Retention

Some records are not just operational logs. Payment and delivery proofs may be needed later for disputes, customer support, accounting, anti-fraud analysis, or legal/commercial requirements.

Recommended 5-year candidates:

- Confirmed payment proof.
- Delivered order proof.
- Delivery failure after payment.
- Refund/chargeback proof.
- Customer restriction tied to fraud or abuse.
- Final order summary with items, total, user id, order id, provider reference, and timestamps.

Recommended shape:

- Keep the compact proof for 5 years.
- Keep bulky raw provider payloads only if truly needed; otherwise store provider ids and normalized status.
- Keep screenshots, transcript HTML, and temporary evidence on a shorter `cleanupAt` unless the project explicitly requires them for the same 5-year window.
- Set `cleanupAt` only after the order reaches a terminal state such as `delivered`, `refunded`, `cancelled`, `chargeback`, or `failed_after_payment`.
- If using a TTL index, use `cleanupAt` with `expireAfterSeconds: 0` and calculate `cleanupAt` from the terminal timestamp.
