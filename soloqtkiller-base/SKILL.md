---
name: soloqtkiller-base
description: Create Discord.js bots and reusable bot bases that follow the SoloQTKiller architecture extracted from the user's own base, using the Moreira-Contas and Ticket-Moreira projects as client examples. Use when Codex needs to scaffold a new Discord.js bot, explain the user's base, add or refactor features in this pattern, or generate files in the structure src/client/events, commands, SlashCommands, interactions, messages, mongoose/schemas, config, and utils.
---

# SoloQTKiller Base

Build on the user's established Discord.js base instead of inventing a new architecture. Keep folder names, loading flow, naming conventions, and the separation between prefix commands, slash commands, interaction handlers, dynamic message panels, MongoDB schemas, and utility helpers.

The complete reference structure may also include `src/services/` for business workflows and scheduled maintenance, `src/emojis/` for local custom-emoji assets, `src/config/application-emojis.json` for the synchronized emoji registry, and `test/` for automated tests. Add only the feature-specific subfolders needed by the bot, but preserve these layers when the project uses them.

## Quick Start

1. Inspect the target project and confirm whether it already follows the SoloQTKiller base.
2. Read [references/architecture.md](references/architecture.md) before changing bootstrap, routing, cache, config, database, or folder layout.
3. Read [references/examples.md](references/examples.md) when creating or adapting `events`, `commands`, `SlashCommands`, `interactions`, `messages`, or `schemas`.
4. Read [references/discord-limits.md](references/discord-limits.md) before implementing features that create or manage Discord channels, categories, roles, threads/topics, forum tags, application commands, emojis, webhooks, invites, or high-volume API operations.
5. Read [references/operational-patterns.md](references/operational-patterns.md) before implementing management panels, modal configuration flows, destructive actions, cooldowns, diagnostics, bot customization, or repeated API work.
6. Read [references/data-integrity.md](references/data-integrity.md) before changing domain states, permissions, logs, cache, migrations/backfills, orphan reconciliation, crash recovery, feature flags, scheduled jobs, audit/history records, or collections that can grow quickly.
7. Read [references/tickets.md](references/tickets.md) completely before creating or changing ticket panels, options, opening, permissions, participants, publication, transcripts, closing, retention, or recovery.
8. Read [references/turbofypay.md](references/turbofypay.md) completely before implementing, reviewing, migrating, or testing TurbofyPay charges, polling, webhooks, balance, receipts, withdrawals, or payouts. Use direct HTTP against the current public v2 contract; do not use `@turbofy/sdk`, whose published contract is materially outdated.
9. Preserve the central routing pattern:
   `index.js` loads collections and handlers.
   `src/client/events/interactionCreate.js` resolves handlers by interaction type and `customId`.
   `src/messages/**` stores dynamic panel renderers.
   `src/mongoose/index.js` loads models into `client.database`.
10. Keep `customId` values predictable, normally in the format `categoria/acao` or `categoria/acao_id`.
11. In an existing project, read `PROJECT_CONTEXT.md` when present before changing a workflow. Update it after a material architectural or operational decision, keeping it concise, factual, and free of secrets.

## Base Rules

- Keep `index.js` as the application bootstrap and loader.
- Keep `src/SlashCommands` with the capital `S`, because the existing base uses that exact folder name.
- Respect the existing handler base for commands and events. Before adding a new entrypoint, inspect how `index.js`, `src/client/events/`, `src/commands/`, `src/SlashCommands/`, and `src/interactions/` are loaded and routed, then follow the same export shape, arguments, registration flow, error handling, and naming convention. Do not create a parallel loader, manually register one-off handlers, or move a handler to a different architecture just to implement a feature.
- Preserve the project's indentation and formatting convention in every changed file. Match the surrounding code exactly, including spaces versus tabs, indentation width, brace style, trailing commas, quote style, semicolon usage, and line wrapping. Do not reformat unrelated code while adding a feature.
- Always write user-facing phrases with correct Portuguese accents in embeds, buttons, placeholders, modal labels, select options, replies, logs, and panel texts. Only omit accents in identifiers, file names, variable names, schema keys, and other technical tokens when needed.
- Support modern Discord modals. When a flow benefits from richer input, prefer modals with `Label` components wrapping `Text Input`, `String Select`, `User Select`, `Role Select`, `Mentionable Select`, or `Channel Select` instead of forcing every configuration step through message components only.
- Distinguish clearly between:
  `commands/` for prefix commands.
  `SlashCommands/` for Discord application commands.
  `interactions/` for buttons, modals, and select menus.
  `messages/` for reusable panel/message builders.
- Load handlers dynamically from folders instead of hardcoding each file in `index.js`.
- Store runtime configuration in `client.config` and database-derived live config in `client.configs`.
- Use `client.database` as the registry for Mongoose models.
- Keep `src/services/` as the business-logic layer when the project has workflows such as payments, carts, delivery, reservations, tickets, reports, automations, transcripts, backups, or external integrations. Handlers should orchestrate inputs and responses while services own domain operations and scheduled jobs.
- Keep `src/emojis/` as the source of local custom-emoji assets and maintain `src/config/application-emojis.json` as the application-emoji registry when custom emojis are used. Synchronize assets during startup before panels that reference them are published.
- Keep automated tests in `test/` and preserve the project's lint and test scripts when changing the base. New pure helpers and critical business rules should have focused tests when practical.
- Prefer slash commands over prefix commands for new user-facing functionality. Use prefix commands mainly for legacy compatibility, fallback administration, or when a project requirement explicitly calls for them.
- Use slash-command subcommands and subcommand groups to organize related operations under one command. Prefer `/vendas criar`, `/vendas editar`, and `/vendas deletar` over separate top-level commands such as `/vendas-criar`, `/vendas-editar`, and `/vendas-deletar`, unless Discord's command constraints or the project's requirements make the grouped design unsuitable.
- Prefer utility functions in `src/utils/functions.js` for shared logic such as permission checks, panel refreshes, ticket checks, thread creation, and schedule validation.
- When a flow uses panels, include a clear way to return to the previous panel so navigation stays fluid. Prefer a dedicated back button such as `categoria/backToPainelAnterior` or another `customId` in the same `/`-based routing pattern.
- If a message contains select menus and buttons, render every select-menu row before every button row. Keep action, navigation, and pagination buttons below selects.
- Configuration flows opened from a panel should normally update that same message with `interaction.update()`. Use separate replies only for user-specific progress, confirmations, or errors that must coexist with the panel.
- Every configuration panel must show every supported field, including unset values as `Não configurado`, `Nenhum`, or `Desativado`.

## Commands, Presence, and Intents

### Project-derived operational defaults

- Treat `/central` as the default administrative hub for guild configuration. Avoid creating one command per setting when the feature belongs to the central control surface.
- Keep access management centralized: only the guild owner or the configured fixed administrator may add or remove authorized users and roles; authorized users and roles may operate protected systems but must not change the access list.
- Separate configuration channels by domain. Use distinct fields for sales logs, sales transcripts, ticket logs, and ticket transcripts; keep MED logs independent. Legacy fields require an idempotent migration and must not remain the active write target.
- When a project uses application emojis, validate local assets before startup synchronization: stable lowercase names, image dimensions suitable for Discord, and the application emoji size limit. Missing emojis must not cause `setEmoji(null)` or `setEmoji(undefined)` validation failures.
- For bot customization, use upload components for avatar/banner files when supported by the installed Discord.js version. Validate image MIME types, GIF support, file size, resizing, cooldowns, and API `429` responses before attempting the Discord request.

- Keep the minimum required command contracts compatible with the dynamic loaders, but do not treat them as closed schemas. A prefix command may expose additional properties on `exports.run` behavior and `exports.help` metadata when the project requires them, such as category, usage, examples, permissions, cooldown, owner-only flags, guild-only flags, required bot permissions, aliases, subcommands, or custom validation metadata. The loader and handler should preserve and use supported metadata instead of deleting or rejecting it.
- For prefix commands, `exports.run` must continue receiving the arguments expected by the existing project loader, while additional context may be provided when the base supports it. Do not force every command into only `(client, message, args)` if the project handler requires more context; inspect the current command dispatcher first.
- For slash commands, preserve the project's command object shape while allowing additional Discord command metadata and options. Implement related operations as subcommands or subcommand groups under one root command whenever that produces a clearer command tree.
- Every bot created from this base must publish a Twitch streaming presence containing `https://www.twitch.tv/soloqtkiller/`. Set it after `clientReady` and schedule it to be applied again at least once every hour so reconnects or Discord presence changes do not remove the link. The activity name and streaming text may describe the bot's current purpose, but the Twitch URL must remain the required URL.
- Use only the Discord gateway intents required by the bot's implemented features. Start by inventorying which events and data the bot actually consumes, declare those intents explicitly in the client, and do not enable broad or privileged intents merely because the base example used them. If a privileged intent is required, document the feature that needs it and ensure it is enabled in the Discord developer portal.
- When adding or removing a feature, re-evaluate the intent list and remove intents that are no longer used. The goal is to avoid receiving events and updates that the bot never processes.

## Discord Limits and Capacity

- Treat Discord resource limits as part of feature design. Before creating channels, categories, roles, threads/topics, forum tags, commands, emojis, webhooks, or invites, inspect the current guild state, compare it with the limits in [references/discord-limits.md](references/discord-limits.md), and fail with a clear user-facing message before making an API request that is known to exceed a limit.
- For ticket, cart, support, purchase, onboarding, or moderation flows that create channels or threads/topics, use a capacity guard like `assertCapacity` before creation. Check total guild channels, category child count, active ticket/cart records, active threads/topics, permissions, and configured destination validity.
- Ticket creation and similar high-impact flows must use an operation lock and queue to absorb double-clicks, repeated select submissions, flood attempts, Discord latency, and bot overload. Prefer a short-lived operation model such as `TicketOperation` or `CartOperation` with unique operation keys, TTL cleanup, per-guild queue limits, global queue limits, per-user cooldown after failures, and a friendly "already processing" response.
- Wrap Discord write operations that can hit route limits, such as creating channels, creating threads, sending repeated messages, registering commands, or syncing emojis, with a backoff helper that honors Discord `429` retry timing. Do not implement tight retry loops.
- Prefer threads/topics for temporary tickets or carts when they fit the product, because they do not count toward the guild's channel limit. Still enforce an internal active-thread cap, auto-archive short-lived threads, and avoid unarchiving old threads just to reuse them unless the flow explicitly requires it.
- Avoid creating one channel, role, webhook, or command per user/product/order when a reusable panel, database record, thread, select option, or subcommand can represent the same state. Resource conservation is part of the SoloQTKiller base.
- When a limit is reached, do not partially create resources. Stop early, keep database records consistent, clean up temporary operation locks, and tell the user/admin which configured category, channel, role, or queue needs attention.

### Sales and ticket concurrency

- A user may have at most one active cart and one active stock reservation. Reserve stock only when the PIX is generated, never when an item is merely added to a cart.
- A reservation must have a fixed expiry, must not be silently renewed, and must be released on cancellation, payment expiry, terminal delivery, orphan reconciliation, or manual Discord resource deletion.
- Never allow a second PIX for the same cart. Reconcile reservations whose cart, order, or payment is already terminal before rejecting a new payment attempt.
- Use per-user and per-guild operation keys for cart/ticket creation, short-lived locks, bounded queues, failure cooldowns, and TTL cleanup. Public panels remain enabled; only the current user's interaction should be rejected while an operation is in progress or capacity is exhausted.
- Before creating a ticket, cart thread, call, or channel, validate Discord capacity, destination type, permissions, category child limits, active records, and the existence of the configured resource. Do not leave a database record claiming an active resource when creation failed.

## Handler and Formatting Contract

- Treat `src/client/events/` as the event-handler layer. Event files receive the same `client` instance used by the base, are loaded dynamically from their filenames, and should coordinate the event rather than duplicate command or panel business logic.
- Treat `src/commands/` and `src/SlashCommands/` as command-handler layers. Keep their established minimum export contracts, allow project-specific metadata on command objects, and let the existing loaders populate `client.commands`, `client.aliases`, and `client.slashCommands`.
- Treat `src/interactions/` as the component-handler layer. Buttons, selects, and modal submits must be resolved through the existing `interactionCreate.js` routing strategy and must not be handled by ad hoc listeners inside feature files.
- When a handler needs reusable behavior, add or reuse a helper in `src/utils/` and keep the handler responsible for validation, orchestration, and the interaction response. Do not duplicate the same routing, permission, refresh, or configuration logic across handlers.
- Before finishing a change, verify that every new command, event, interaction, and message renderer is discoverable by the existing dynamic loader and that its `customId`, filename, folder, export shape, and invocation arguments agree.
- Treat expected user/configuration validation failures as normal interaction outcomes: reply with a short localized message and avoid logging a full technical stack trace. Reserve error logs for unexpected failures, API failures, reconciliation failures, and data-integrity problems.
- Preserve indentation in generated examples and source edits. Formatting is part of compatibility with this base: use the local file as the authority when the reference examples differ from the target project's style.

## Visual Identity and Custom Emojis

- Treat visual identity as part of the feature contract, not as decoration added later. Panels, buttons, select-menu options, modal labels where supported, embeds, status messages, and other user-facing actions should communicate their purpose consistently.
- Every panel must use custom emojis in its title, section headings, status indicators, descriptive blocks, buttons, and select-menu options wherever the component supports them and a suitable registered emoji exists. The panel itself must have a visual identity, not only its action buttons.
- Register each custom emoji once in the central application-emoji registry, keep its semantic name stable, and reuse that name across panels, logs, embeds, tickets, modals, and services. Do not create feature-local emoji maps that can diverge from the global registry.
- Treat `src/emojis/` as the authoritative source of application-emoji assets and `src/config/application-emojis.json` only as a cache of confirmed ids/hashes. On every startup, fetch the real application emojis from Discord and reconcile every eligible local asset before any dependent panel, command publication, or service starts; never trust the JSON alone as proof that an emoji still exists.
- Upload each application emoji using exactly the local file basename as its Discord name. Require valid stable lowercase basenames instead of silently lowercasing or renaming files. For example, `configuracoes.png` must register as `configuracoes`.
- Reuse a remote emoji only when the registry id exists remotely, its remote name matches the exact local basename, and the stored local hash matches the current asset hash. A remote emoji found only by name, or an entry with missing/mismatched id, name, or hash, is untrusted and must be deleted/recreated from the local file when exact asset identity is required.
- After reconciliation, rewrite the local registry with the confirmed remote id, exact filename, current hash, and animation state, then build the runtime mention map from those confirmed remote objects. A required synchronization failure must prevent emoji-dependent startup flows from continuing silently.
- Every actionable button and every select-menu option should use a custom application emoji that represents its action whenever a suitable emoji exists. Examples include configuration, return, save, edit, delete, users, roles, channels, logs, sales, tickets, reports, security, and status actions.
- Use the same emoji for the same action everywhere in the bot. A child panel, modal, ticket message, and central panel that expose the same operation must share the registered emoji instead of inventing a new visual symbol.
- Resolve emojis through the project's centralized application-emoji registry or helper, such as `src/utils/applicationEmojis.js` and `emoji(client, name)`, when that infrastructure exists. Do not scatter raw emoji ids, hardcode image URLs, or mix arbitrary Unicode emoji into components that should use application emojis.
- Reusable button and option builders should apply the registered emoji automatically when possible. A component should only omit an emoji when no appropriate registered emoji exists or when Discord's component constraints make it impossible.
- Choose an emoji based on the semantic action, not merely the text label. Avoid using the same generic emoji for unrelated operations, and keep destructive actions visually distinct from neutral or navigation actions.
- When adding a new action that has no suitable emoji, register or document the required application emoji following the project's existing synchronization process before using it in panels. Keep the local emoji name stable so future handlers can reference it consistently.
- When adding a new panel section or feature, first check whether a registered emoji already represents it; if not, add the local asset and registry entry through the existing synchronization flow before publishing the panel.
- If the project requires fully custom emoji UI, do not substitute Unicode emoji in titles, buttons, select options, status labels, or log headings. Prefer omitting an unavailable emoji safely over passing a null value to a Discord builder.

## Components V2 and Central Control

- Use Discord Components V2 for every new panel, administration screen, ticket panel, configuration screen, and other rich interactive interface whenever the target project supports it. Build these screens through the project's shared Components V2 helpers and include the required `MessageFlags.IsComponentsV2` flag.
- Do not mix legacy panel construction and Components V2 arbitrarily. If an existing flow is being changed, preserve compatibility with its current renderer or migrate the complete renderer through the project's established Components V2 abstraction.
- Prefer shared builders such as `createContainer`, `payload`, and `emoji` from `src/utils/componentsV2.js` when they exist. Keep layout, colors, separators, text blocks, action rows, and emoji treatment consistent through those helpers rather than rebuilding the visual system in each feature.
- Components V2 panels should use structured text sections and separators, not a single dense block. Prefer the shared `createContainer` pattern that splits the first line into the title, adds a large separator before the body, and inserts a small separator before the first action row.
- Markdown heading markers must precede custom emojis: use `## <:emoji:id> Título`, never `<:emoji:id> ## Título`.
- Components V2 panel builders should centralize the placement of panel emojis, section emojis, state emojis, and action emojis so new panels inherit the visual language automatically.
- Configuration and administration features should be reachable from a central control panel whenever the feature belongs to the bot's global or guild management surface. The central panel should expose the available areas, current state when useful, and navigation to child panels without requiring separate commands for every setting.
- The central panel is a management hub, not just a landing message: it should allow administrators to access, publish, edit, refresh, enable, disable, and otherwise manage the supported panels and systems through the existing permission and persistence layers.
- Keep permission helpers semantically separate. A fixed administrator, guild owner, Discord `Administrator`, configured staff role, and resource owner are different principals. Never reuse a broad helper for a fixed-administrator-only command, and test negative cases where guild owners/admins must still be rejected.
- New settings that require administration should be represented in the central navigation or in a clearly linked child panel. Do not hide a manageable feature behind an undocumented `customId` or an isolated command when it belongs in the central control surface.
- Large bots should expose operational panels for diagnostics, backups, resource usage, queues, automation status, and bot customization when those systems exist. These panels should show current state and safe actions rather than requiring hidden commands.
- Every log message that includes one or more file attachments must use a conventional/legacy Discord message with `embeds` and `files`. This is mandatory for ticket transcripts and applies equally to exported reports, evidence files, backups, receipts, or any other attachment-bearing log. Such payloads must not contain Components V2, must not set `MessageFlags.IsComponentsV2`, and must never pass through `createContainer`, `refreshMessageData`, `decoratePayload`, an interaction decorator, or any helper that converts/decorates payloads as Components V2. Send the legacy payload directly, re-fetch it when delivery confirmation matters, and confirm its attachments before deleting or finalizing the source resource.
- A Components V2 helper that injects separators must be the single owner of the separator before the first action row. Feature renderers must not add another separator at that boundary.

## Panel Lifecycle and Navigation

- Implement panels as refreshable renderers in `src/messages/**`, so commands and interactions can reuse the same screen instead of duplicating layout and state logic.
- Every child panel must provide a visible and working way to return to its parent or the central panel. Use the same `/`-based `customId` routing convention for back actions and preserve the user's navigation context.
- When a panel opens a modal or another child screen, preserve its origin and refresh the originating panel after submission, cancellation, or a successful configuration change whenever the project has origin tracking such as `src/utils/panelOrigin.js`.
- Read the current state from `client.configs` and MongoDB-backed configuration before rendering. After a mutation, persist the change and refresh the local configuration cache before rebuilding the panel so the interface reflects the real state.
- Prefill modal fields and default-select existing users, roles, channels, mentionables, and options whenever the current configuration is known. A management panel should make the current state visible and editable rather than forcing administrators to re-enter unchanged values.
- Keep permission checks, persistence, panel rendering, and interaction acknowledgement separated into the existing handlers and utilities. A panel must not claim a setting changed until the database/cache update succeeds.
- Destructive actions such as deleting panels, removing options, clearing stock, closing tickets, restoring backups, deleting rules, or removing configured access must use an explicit confirmation step, explain the impact, and refresh the relevant panel after completion.
- Functions with Discord cooldowns, expensive API calls, external payment/API calls, or high fan-out effects must expose a domain cooldown, operation lock, or queue. Store timestamps such as `lastNotificationAt`, `lastUpdatedAt`, `nextUpdateAt`, or dedicated operation documents instead of trusting button disable state alone.
- Every modal/select mutation must refresh the exact originating panel, including after success, cancellation, deletion, or return navigation. A generic fallback panel must not redirect a child flow to an unrelated section.

## Data Integrity, Permissions, Logs, and Retention

- Restricted bots must use the plural environment variable `ALLOWED_GUILDS_ID` as a comma-separated allowlist, normalized into an immutable `client.config.allowedGuildIds` array. Validate every configured snowflake, apply guards to every guild entrypoint, register/load every allowed guild, remove commands from foreign guilds, and query maintenance data for the full list rather than a single guild. Do not use the legacy singular `ALLOWED_GUILD_ID` in new or updated bots.
- Ao conectar com Mongoose, use `serverSelectionTimeoutMS: 60000` e `socketTimeoutMS: 45000`. Mantenha esses valores em uma configuração central/testável e combine-os com jobs single-flight e backoff, para tolerar eleições ou oscilações breves do MongoDB sem sobrepor rotinas de manutenção.
- Model every important workflow with explicit domain states and terminal states. Prefer `status` enums such as `processing`, `open`, `closed`, `failed`, `expired`, and `orphaned` over scattered booleans that make panels, cleanup jobs, and backfills ambiguous.
- Keep allowed state transitions in a service/helper when the same domain is touched by commands, buttons, modals, jobs, or webhooks. A handler should ask the domain layer to transition state instead of mutating status fields freely.
- Every feature that creates Discord resources must have an orphan reconciliation path. If a managed channel, thread, message, role, webhook, AutoMod rule, command, or emoji disappears outside the bot, mark the database record as `orphaned`, `missing`, `stale`, or `disabled`, clear unsafe active pointers, set `cleanupAt` when appropriate, write a private admin audit, and refresh the related panel.
- Validate permissions in layers: Discord permissions, configured admin/staff roles, resource ownership, domain-specific permission, and destructive confirmation permission. Re-check permissions on the final submit/confirm step because roles and ownership may change while a modal or confirmation is open.
- Separate public logs from private/admin logs. Public logs should contain only safe operational summaries, while private logs may include moderation, fraud, payment diagnostics, stack/error summaries, queue failures, and reconciliation details. Never expose tokens, secrets, raw payment payloads, private evidence, or full sensitive database documents in public panels or channels.
- Helpers that affect money, stock, coupons, permissions, tickets/carts, state transitions, placeholders, cache reload, limits, TTL cleanup, and orphan reconciliation are critical helpers. Add focused automated tests in `test/` whenever practical before or alongside changes to those helpers.
- Schema changes that affect existing records need a migration/backfill plan. Prefer backwards-compatible defaults, idempotent backfills, progress logging, and safe fallback rendering while old records are being migrated.
- Cache policy is mandatory for configurable systems. After a database mutation, reload the affected guild cache such as `client.reloadConfigs(guildId)` or invalidate the specific local cache entry before refreshing panels or replying that the change is active.
- Features with side effects should define a crash recovery plan. On startup or scheduled maintenance, reconcile operations stuck in `processing`, pending payments, half-open tickets/carts, expired reservations, missing Discord resources, and jobs that were interrupted before writing their terminal state.
- Large or risky features should be controlled by feature flags or gradual configuration. Default new systems to disabled or admin-only when appropriate, expose enabled/disabled state in management panels, and use per-guild config to roll out without activating incomplete behavior globally.
- Scheduled jobs must use locks when duplicate runs can create duplicate messages, double delivery, repeated renames, repeated cleanup, or conflicting backfills. Store `lockedAt`, `lockedBy`, `expiresAt`, `lastRunAt`, `nextRunAt`, `failureCount`, and a terminal status when the job affects shared state.
- Use friendly error handling for interactions. Show the user/admin a short safe message through the shared Components V2/status payload style, while logging the technical error privately with useful context such as `guildId`, `userId`, `orderId`, `ticketId`, `jobName`, or `operationKey`.
- Counter systems and placeholder renderers are critical helpers. Validate placeholder allowlists, cap channel names to Discord limits, enforce minimum update intervals, track `lastUpdatedAt`, `nextUpdateAt`, `failureCount`, and disable broken counters after repeated failures instead of retrying forever.
- Collections that can grow quickly must define a retention policy before launch. Audit logs, operation locks, pending workflows, rate-limit/cooldown attempts, temporary imports, transcript metadata, automation deliveries, report deliveries, review requests, fraud evidence, and repeated event history should use `expiresAt`, `cleanupAt`, TTL indexes, or scheduled pruning so MongoDB does not keep unbounded operational noise forever.
- Use TTL carefully: short-lived locks and drafts can expire directly with `expiresAt`; business records that need a close/final state should set `cleanupAt` only after reaching a terminal state; payment confirmations, delivery proofs, order summaries, invoice-like records, and other commercial/accounting evidence may require long retention such as 5 years, so store compact proof records and prune only bulky payloads or transient evidence according to the project's retention requirement.

## Reliable External Workflows

- Treat every external integration as an adapter with an explicit input/output contract. Keep provider-specific URLs, authentication, payloads, response normalization, status mapping, timeout, retry, and rate-limit behavior inside the adapter; handlers and domain services should consume the normalized result.
- Distinguish an internal gateway from the upstream provider it calls. When an internal API owns balance, permissions, audit records, Discord writes, or recovery, call the internal endpoint instead of bypassing it with a direct provider request.
- Configuration precedence must be explicit and tested, normally environment variables over local JSON/config files. Never put secrets in query strings, logs, fixtures, documentation, or user-visible responses. Prefer headers for internal credentials unless an existing contract requires another transport.
- Every external call needs a bounded timeout and a safe error code. Retry only idempotent operations or operations with a proven idempotency key; honor `Retry-After` for `429` and never blindly repeat a resource-creation request after an unknown timeout.
- Normalize provider identifiers before entering the domain flow. An external event id is not globally unique unless its scope is defined; compose identity from the source/provider plus account or device, server/tenant, actor/resource, and provider event id when those dimensions can collide.

### Provider-backed resources and reservations

- When an upstream provider exposes technical mode, product, queue, or resource identifiers, keep a central catalog with immutable internal IDs and translated display names. Aliases are optional and should exist only for real legacy/compatibility inputs. All commands, panels, API endpoints, and provider payloads must resolve through that catalog.
- Treat provider migrations as a bounded adapter change: map the old and new contracts, define whether compatibility is required, normalize the new response, move mode/resource mappings into the catalog, remove obsolete provider paths only after callers and tests are migrated, and retain a rollback or reconciliation plan when the provider owns live resources.
- For a finite internal credit, stock item, seat, or quota that creates an external resource, model one operation with a generated `operationId`: atomically reserve the internal unit, call the provider, attach its request/resource id, then atomically complete or compensate. Track terminal states such as `completed`, `failed`, and `needs_review`; set TTL cleanup only after a terminal state.
- Do not implement concurrent balances, reservations, counters, or histories as read-modify-`save()` flows. Use conditional atomic updates (`$inc`, `$push`, `$pull`, state filters, and a reservation key) so concurrent commands, API requests, webhooks, and recovery jobs cannot double-consume or double-refund.
- Completion and refund must be idempotent. The reservation record is the source of truth: remove it exactly once on completion or refund, and only finalize the customer-facing debit after a valid resource is available. If the provider resource cannot be released after a failure, retain the reservation, mark `needs_review`, and create a private operational alert rather than issuing an unsafe refund.
- A rate gate for slow creation APIs should control when requests start, not wait for an earlier request to finish. Release the next queued creation after the current request has begun, while still enforcing a bounded minimum start interval and provider-specific concurrency/rate limits.
- Recovery jobs must reconcile stale reserved/processing operations on startup and periodically. Process bounded batches, release or query the external resource before refunding, preserve unresolvable cases for review, and make repeated recovery safe.
- When completing a shared-wallet or multi-document workflow, use a database transaction where the deployment supports it. Verify transaction support in the intended MongoDB topology and keep an idempotent compensation path for failures before or outside the transaction.

### Payment and event concurrency

- Payment confirmation must be serialized at the domain boundary with a lock or database lease keyed by the match and player. Use conditional atomic updates so two workers cannot confirm, consume, refund, or complete the same workflow twice.
- Model payment and verification transitions explicitly (`processing`, `confirmed`, `expired`, `cancelled`, `failed`, or equivalent). Keep the transition in a service shared by commands, webhooks, listeners, and recovery jobs.
- Recovery after restart is part of the feature: claim pending records with a short lease, re-read the authoritative external status, reconcile terminal states, release the lease in `finally`, and make the recovery idempotent.
- If the product has multiple payment sources (for example app notifications, IMAP, or marketplace webhooks), normalize each source into one contract and define which source is authoritative when events race. A fallback source must not create a second verification for the same player/match.
- Persist a payment idempotency key before creating PIX. An uncertain timeout/response must trigger reconciliation of the original operation, never blind creation of a second charge.
- For TurbofyPay, follow [references/turbofypay.md](references/turbofypay.md) as the mandatory provider contract. Never copy routes or payloads from `@turbofy/sdk`; verify the public documentation and OpenAPI first, use the canonical `/sellers/pix` routes, and treat `/rifeiro/pix` only as a legacy alias.
- Administrative money inputs may accept Brazilian forms such as `1`, `1.00`, `1,00`, and `0,10`; normalize once to integer cents and keep all domain/database arithmetic in cents.
- Document every provider credential, base URL, timeout, polling interval, and safe limit in `.env.example`. Validate presence at startup without logging values.

### IMAP and long-lived listeners

- Persistent IMAP listeners should persist a cursor per account/mailbox, handle `UIDVALIDITY` changes, use a renewable lease, bound message size and sync batch size, cap concurrency, and retain failed UIDs for retry.
- Add watchdog checks for providers that stop emitting mailbox events, reconnect with progressive backoff, and expose operational health without exposing credentials or raw message bodies.
- Encrypt stored IMAP credentials at rest and provide an idempotent migration for legacy plaintext records. Debug logs may include redacted identifiers, domains, field presence, lengths, latency, and outcome, but never passwords, tokens, full e-mail content, or sensitive payment data.

### Webhooks and observability

- Internal webhooks need a body-size limit, payload validation, explicit authentication, safe `401/400` responses, health endpoints, bounded processing, and a clear bind policy. Do not let a secondary action make the webhook appear unavailable after the main event is safely stored.
- Observability is non-critical side work. Use short timeouts, redaction, correlation ids, and an asynchronous queue/outbox or circuit breaker so a telemetry database/network failure cannot block payment, delivery, or room creation. Record useful latency and failure context privately.
- External HTTP APIs must validate method, body, identifiers, authorization, readiness, maintenance state, and resource ownership before invoking a domain service. Use bounded JSON parsing, rate limiting appropriate to the caller, stable safe error responses, and a health endpoint. Do not put API credentials in query strings or record credentials, raw payment/webhook payloads, personal documents, or full request headers in logs.

### Discord message parsers

- Parsers that consume another bot's panels must handle both `messageCreate` and `messageUpdate`, fetch partial messages, accept child threads via configured `parentId`, and deduplicate accepted input by source message id.
- A new message in the same channel is a new event (for example a rematch); an already accepted message id can never become valid a second time.
- During parser homologation, log redacted configuration, full non-sensitive source text, extracted fields, extraction method, missing fields, and exact rejection reason. Make high-volume informational diagnostics easy to disable after homologation.

## Safe Admin Analytics and Data-Heavy Panels

- Analytics endpoints must require a bounded time range or apply a documented small default range. Enforce a maximum range, maximum page size, validated filters, and a hard cap for expensive aggregations.
- Use explicit projections and aggregation fields; never return full operational documents merely to render a table. Paginate in the backend, return `hasNext`/total metadata as appropriate, and keep count queries bounded or approximate when exact counts are not required.
- Opening a tab or changing filters must not launch duplicate requests. Use one in-flight load per view, cancel or ignore stale responses, and cap simultaneous analytics queries.
- Every async list, chart, or table must have a visible title/context plus distinct loading, success, empty, and error states. A blank table is not a loading state.
- Add performance tests for large datasets, missing-period requests, pagination boundaries, projection size, concurrent tab loads, and heap/RAM usage. Record the query shape and memory budget used by the test.

## Resilience Testing Checklist

- Critical integrations require contract tests for success, malformed responses, authentication failures, timeouts, `429`, upstream `5xx`, and duplicate delivery.
- Critical workflows require race tests: repeated confirmation, the same user in nearby matches, process restart during verification, lost lease, recovery after timeout, and provider response arriving twice.
- Parsers for Discord embeds, bank e-mails, and webhook payloads should use anonymized fixtures and test formatting variations, Brazilian currency formats, missing fields, truncation limits, and unsupported versions.
- Test TTL/cleanup behavior so active or processing records are never removed by a creation-time TTL. Test migrations twice to confirm idempotency.
- Run a focused memory/latency test for high-volume listeners and admin analytics before increasing limits or concurrency.
- For provider-backed resource creation, add focused tests for catalog resolution/translation, timeout and retry policy, `429` behavior, malformed provider responses, concurrent reservations, repeated completion/refund, failed external release, stale-operation recovery, and transaction/compensation boundaries. Use anonymized fixtures and never place credentials in tests.

### Payment, transcript, and external API invariants

- Payment-gateway polling must use a bounded interval and initial delay, stop on terminal states, stop or disable after invalid credentials, honor `Retry-After` exactly on `429`, prevent duplicate pollers, and never retry in a tight loop. Keep TurbofyPay, PushinPay, or other provider details inside their adapters/references.
- Payment and delivery state transitions must be idempotent. A paid order must not reserve, deliver, grant roles, or write completion logs twice after a restart or duplicate gateway observation.
- Transcript closure is a safety boundary: generate the transcript, send the embed and attachment together, refetch the message by ID, confirm the attachment, retry a bounded number of times, and only then mark the ticket/cart closed and delete its Discord resource.
- Attachment-bearing logs are a hard exception to the Components V2 panel standard: send them directly as legacy `{ embeds, files }` payloads with no `components` and no `IsComponentsV2` flag, bypassing all Components V2 conversion/decorator functions. Store compact transcript/export metadata in MongoDB; do not store bulky attachment contents unless explicitly required.
- MED monitoring must associate an infraction with the matching order, keep evidence and defense data under an explicit retention policy, send a dedicated administrative alert, and prevent destructive order changes while the case is active.
- Backup restore must be owner/fixed-admin only, exclude secrets, validate the imported schema, show a preview, require an explicit confirmation button, apply atomically where possible, and reload the guild cache afterward.

### Feature-specific safety rules

- Automatic messages must keep channel IDs and category IDs in separate lists, expand categories to current child channels at send time, cap select menus at Discord's limit, and back off instead of flooding logs when no valid destination exists.
- Counter configurations must validate an allowlisted type and placeholder set, accept text/voice channels only where the rename API supports them, enforce a minimum five-minute interval, cap channel names, track `nextUpdateAt`/`failureCount`, and disable repeatedly failing counters.
- Coupon discounts must be recalculated after every cart item or quantity mutation. Coupon usage and reservations must be idempotent and expire independently from the cart when appropriate.
- Destructive ticket, stock, backup, access, AutoMod, and panel operations require an explicit confirmation and must refresh the parent list after completion.

## Recommended Workflow

### Create a new bot

Replicate the base folders first, then wire `index.js`, `src/client/events`, `src/utils`, `src/config`, and `src/mongoose` before implementing feature folders.

### Add a new feature

Define the entrypoint first:
- prefix command in `src/commands/<grupo>/`
- slash command in `src/SlashCommands/<grupo>/`
- component interaction in `src/interactions/<Type>/<categoria>/`
- reusable panel in `src/messages/<categoria>/`
- persistence in `src/mongoose/schemas/`

Then connect the feature to the existing routing format instead of bypassing it.

For a command with multiple related operations, first model the slash-command tree. Prefer one root command with subcommands or subcommand groups, and only create separate top-level commands when the operations are unrelated or a concrete Discord/project constraint requires it.

### Explain or refactor the base

Map the request onto these layers:
- bootstrap and collections
- event routing
- command routing
- interaction routing by `customId`
- dynamic message panels
- database and cached config

## Implementation Notes

- When generating a new interaction, make the `customId` match the file path that should handle it.
- When generating a panel message, prefer calling `client.functions.refreshMessageData(...)` or storing the renderer in `client.messages`.
- When building multi-step panels, keep the navigation reversible. If a user enters a child panel, provide a visible back action that returns to the parent panel instead of forcing the flow to restart from zero.
- When building modals for configuration, prefer pre-filling the current state:
  use `TextInputBuilder#setValue(...)` for existing text values;
  use default selected values for `User Select`, `Role Select`, `Mentionable Select`, and `Channel Select` when the current config already has ids saved.
- In modals, keep `customId` values on the SoloQTKiller `/` pattern just like buttons and select menus.
- When a feature depends on guild configuration, load it from MongoDB and refresh `client.configs` rather than duplicating literals.
- When a feature opens tickets or threads, keep validation, persistence, panel sending, and closing/logging separated into their own handlers or helpers.

## References

- Read [references/architecture.md](references/architecture.md) for the extracted structure and execution flow.
- Read [references/examples.md](references/examples.md) for concrete examples of `index.js`, events, commands, slash commands, interactions, messages, mongoose models, utility helpers, and the application-emoji registration system used by the base.
- Read [references/discord-limits.md](references/discord-limits.md) before implementing features that create Discord resources or can generate heavy API traffic.
- Read [references/operational-patterns.md](references/operational-patterns.md) for recurring rules around Components V2 separators, management panels, modal origin refresh, destructive confirmations, diagnostics, cooldowns, and bot customization.
- Read [references/data-integrity.md](references/data-integrity.md) for domain states, orphan reconciliation, layered permissions, public/private logs, helper tests, migrations/backfills, cache policy, crash recovery, feature flags, job locks, friendly errors, and TTL/retention rules.
- Read [references/tickets.md](references/tickets.md) for the complete ticket architecture, schemas, configuration, channel/thread permissions, staff controls, publication, transcripts, closing, recovery, and examples.
- Read [references/turbofypay.md](references/turbofypay.md) for the current TurbofyPay v2 routes, direct-HTTP requirement, authentication, HMAC, idempotency, PIX, webhook, balance, receipt, withdrawal, payout, polling, security, and testing contracts.
