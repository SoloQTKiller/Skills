# Architecture Reference

## Table of Contents

1. Source projects
2. Shared folder structure
3. Bootstrap lifecycle
4. Routing conventions
5. Command design, presence, and intents
6. Discord limits and capacity
7. Operational UX and management panels
8. Data integrity, permissions, and retention
9. Data and configuration flow
10. Folder responsibilities
11. Naming conventions
12. UI and panel architecture
13. When to mirror each layer

## Source Projects

This skill documents the SoloQTKiller base using these client projects as extraction references:

- `D:\r028\Moreira-Contas`
- `D:\r028\Ticket-Moreira\Ticket-Moreira`

Both projects share the same SoloQTKiller architectural spine and differ mainly in business rules, interaction categories, and schemas.

## Shared Folder Structure

```text
index.js
src/
  client/
    events/
      clientReady.js
      interactionCreate.js
      messageCreate.js
  commands/
    admin/
    general/
  config/
    client.json
    config.json
    application-emojis.json
  interactions/
    Button/
    ChannelSelectMenu/
    ModalSubmit/
    RoleSelectMenu/
    SelectMenu/
  messages/
    <categoria>/
  services/
    <dominio>.js
  mongoose/
    index.js
    schemas/
  SlashCommands/
    <grupo>/
  emojis/
    <nome>.png
  utils/
    antiCrash.js
    functions.js
    logger.js
    reloadLocalCache.js
test/
  <feature>.test.js
```

## Bootstrap Lifecycle

The base starts in `index.js` and normally does this in order:

1. Create the Discord client with explicit `GatewayIntentBits`.
2. Configure cache limits and sweepers.
3. Attach shared state to `client`:
   `config`, `messages`, `slashCommands`, `commands`, `aliases`, `database`, `logger`, `functions`, `localCache`.
4. Load crash protection from `src/utils/antiCrash.js`.
5. Start MongoDB through `src/mongoose/index.js`.
6. Load message renderers from `src/messages/**`.
7. Load slash commands from `src/SlashCommands/**`.
8. Load prefix commands from `src/commands/**`.
9. Load Discord event handlers from `src/client/events/**`.
10. Login with `src/config/client.json`.

After login or during `clientReady`, fetch the application's real remote emoji collection and synchronize every eligible asset from `src/emojis/` before rendering/publishing panels or starting dependent services. Treat `src/config/application-emojis.json` only as cached ids/hashes, not as remote truth. Upload using the exact validated local basename, recreate untrusted/missing/mismatched remote entries, persist confirmed ids, and fail the dependent startup path when required synchronization cannot be completed.

Restricted deployments use `ALLOWED_GUILDS_ID`, a comma-separated allowlist normalized into `client.config.allowedGuildIds`. Startup configuration loading, guild command registration, cache hydration, event guards, and scheduled maintenance must iterate/query the entire list. Do not retain a single `allowedGuildId` shortcut that silently processes only the first guild.

Startup logs should provide a safe operational inventory: renderers by category, slash commands by category, prefix commands by category, events by name/category, emoji synchronization result, MongoDB state, and the bot invite URL with `permissions=8`. Never log tokens, database URIs, gateway secrets, PIX codes, or webhook credentials.

Message-driven parsers that consume other bots' panels should handle both `messageCreate` and `messageUpdate`, fetch partial messages before parsing, accept threads whose `parentId` is a configured parent channel, and deduplicate by source message id. A new source message in the same channel is a distinct event (for example, a rematch); an edited source message already accepted must never create a duplicate.

## Routing Conventions

### Events

Every file in `src/client/events/` becomes an event listener. The filename becomes the event name.

Examples:

- `clientReady.js` -> `client.on("clientReady", ...)`
- `interactionCreate.js` -> `client.on("interactionCreate", ...)`
- `messageCreate.js` -> `client.on("messageCreate", ...)`

### Prefix commands

Prefix commands live in `src/commands/<grupo>/arquivo.js`.

Minimum expected shape:

- `exports.run = async (client, message, args) => {}`
- `exports.help = { name, aliases }`

`messageCreate.js` parses prefix input and resolves commands through `client.commands` and `client.aliases`.

### Slash commands

Slash commands live in `src/SlashCommands/<grupo>/arquivo.js`.

Expected shape:

- `module.exports = { name, description, type, run }`

They are collected in `index.js`, pushed into an array, and registered in `clientReady`. The object may include additional Discord command metadata and options required by the project. Prefer subcommands and subcommand groups for related operations.

### Interactions

Interactions live in:

- `src/interactions/Button/<categoria>/<acao>.js`
- `src/interactions/ModalSubmit/<categoria>/<acao>.js`
- `src/interactions/SelectMenu/<categoria>/<acao>.js`
- `src/interactions/ChannelSelectMenu/<categoria>/<acao>.js`
- `src/interactions/RoleSelectMenu/<categoria>/<acao>.js`

The routing key is the `customId`.

Examples:

- `tickets/abrirTicket` -> `src/interactions/SelectMenu/tickets/abrirTicket.js`
- `tickets/fechar_<id>` can fall back to `src/interactions/Button/tickets/fechar.js` in the contas base
- `ticket/selecionarMotivo` -> `src/interactions/SelectMenu/ticket/selecionarMotivo.js`

The handler category is the first segment before `/`. The action is the second segment.

Modern modals can now collect more than plain text. In the SoloQTKiller base, modal submit handlers in `src/interactions/ModalSubmit/**` should be ready to process:

- text inputs
- string selects
- user selects
- role selects
- mentionable selects
- channel selects

For modal construction, wrap each modal field in a `Label` component and keep the same `/` routing pattern in every modal `customId`.

## Command Design, Presence, and Intents

- Prefer slash commands for new functionality. Use prefix commands for compatibility or explicit project requirements.
- Group related operations under one root slash command, such as `/vendas criar`, `/vendas editar`, and `/vendas deletar`, instead of multiplying top-level commands.
- Prefix command files keep the loader's minimum `exports.run` and `exports.help` contract, but both may contain additional project-specific metadata. Do not discard fields such as category, permissions, usage, cooldown, aliases, or custom command configuration.
- Every bot must expose a streaming presence with the Twitch URL `https://www.twitch.tv/soloqtkiller/`. Set it during `clientReady` and refresh it at least hourly.
- Declare only the gateway intents required by the events and features actually used. Revisit the list whenever features change and document any required privileged intent.

## Discord Limits and Capacity

Resource limits are part of the architecture, not only runtime error handling. Before a feature creates channels, categories, roles, threads/topics, forum tags, application commands, emojis, webhooks, invites, or repeated API writes, read `references/discord-limits.md` and design the flow around those caps.

Ticket and cart systems should use service-level capacity guards before creating Discord resources. The Moreira-Sensi pattern is: short-lived operation records for double-click protection, per-guild/global queue limits, per-user cooldowns after failed attempts, active ticket/cart checks, active-thread counting with a short cache, category child-count checks, and `429` backoff around Discord write calls.

Prefer threads/topics for temporary user-specific flows when appropriate because they preserve the guild channel budget. Still count active threads/topics, apply short auto-archive durations for temporary work, and stop early with a clear message when the internal active-thread cap or Discord resource caps are reached.

Do not create one permanent channel, role, command, webhook, or invite for every user/order/product when a database row, select option, panel section, subcommand, or temporary thread can represent the state. The central panel and services should make resource usage visible enough for admins to fix full categories, full queues, or invalid destinations.

## Operational UX and Management Panels

The base treats operational UX as part of the architecture. Management features should be reachable through panels that show current state, expose safe actions, and refresh after mutations. Avoid hidden-only admin commands for systems that are configured repeatedly.

Components V2 panels should follow the shared visual system: title text, separator, body/status blocks, another separator before action rows, semantic accent color, and registered custom emojis. Use the project helper in `src/utils/componentsV2.js` when available instead of hand-building every separator and payload.

Modal-driven configuration must preserve origin. When a button opens a modal that changes guild state, remember the originating panel and refresh it after submit. If the original message was deleted, the persisted change remains valid and only the visual refresh is skipped.

Destructive and high-impact operations need a confirmation step and an operation strategy proportional to their risk. Deleting panels/options, clearing stock, restoring backups, closing tickets, changing bot identity, reposting public panels, creating channels/threads, syncing emojis, and external API operations should use confirmations, cooldowns, queues, locks, or backoff as appropriate.

Large bots should expose diagnostics in the central panel or a child panel. Useful state includes active operations, queue depth, CPU, memory, event-loop lag, MongoDB latency, automation status, failed jobs, and configured log channels.

## Data Integrity, Permissions, and Retention

The base treats domain integrity as architecture. Any workflow that can be opened, closed, paid, delivered, expired, cancelled, or deleted should have explicit states and a service-level transition path. Avoid spreading status changes across handlers because commands, buttons, modals, jobs, and webhooks can race each other.

Managed Discord resources need reconciliation. If a stored channel, thread, panel message, role, webhook, command, AutoMod rule, or application emoji is missing, the bot should mark the record as `orphaned`, `missing`, `stale`, or `disabled`, clear unsafe pointers, log privately, and refresh the admin panel instead of crashing or silently recreating duplicates.

Permissions should be checked in layers: Discord permission, configured admin/staff access, feature-specific role or owner, and final confirmation for destructive actions. Re-check permissions when a modal or confirmation is submitted.

Logs should be audience-aware. Public logs are for safe summaries; private/admin logs are for failures, diagnostics, fraud evidence, queue saturation, rate limits, and reconciliation. Sensitive payloads, secrets, tokens, raw payment data, and private evidence should not appear in public panels, logs, transcripts, or backups.

High-volume collections must include a retention policy. Use `expiresAt` TTL for short-lived locks and drafts, `cleanupAt` TTL for records that can only be removed after a terminal state, and scheduled pruning for audit logs that need event-specific retention. Store compact permanent summaries instead of keeping bulky operational history forever.

Schema changes should include migration/backfill planning. Prefer backwards-compatible defaults, idempotent batch backfills, temporary compatibility reads, private progress logs, and cache invalidation after the backfill changes cached fields.

### Messages

`src/messages/**` stores reusable embed/panel builders used to refresh or render rich UI flows.

Typical usage:

- admin command triggers a panel refresh
- interaction changes the active panel
- utility helper resolves `messages/<categoria>/<arquivo>.js`
- child panels should expose a back action that returns to the previous panel for smoother navigation
- child panels can open modals for focused configuration while still preserving a return path to the previous panel after submit or cancel

## Data and Configuration Flow

There are two config layers:

- `client.config`: static JSON config from `src/config/config.json`
- `client.configs`: live guild config loaded from MongoDB

`reloadLocalCache.js` usually loads the guild document from MongoDB and maps selected sections into:

- `client.configs.logs`
- `client.configs.tickets`
- `client.configs.vendas`

This allows panels and interactions to use database-driven switches, channels, roles, and flags without reading raw JSON every time.

## Folder Responsibilities

### `src/client/events`

Handle Discord entrypoints only. Delegate business logic to commands, interactions, messages, or utilities.

### `src/commands`

Store traditional prefix commands for admins or quick legacy triggers.

### `src/SlashCommands`

Store Discord slash commands used to publish panels, run utilities, or expose admin actions.

### `src/interactions`

Store the actual UI flow handlers for buttons, selects, and modals. This is the main engine for ticket systems and configuration panels.

### `src/messages`

Store reusable panel renderers and embed builders. These are useful when the same screen must be refreshed from commands or interactions.

Panels should use the centralized custom application-emoji registry in their title, sections, statuses, descriptions, buttons, and select options wherever supported. The panel's visual identity should remain consistent with related panels and logs.

### `src/services`

Store reusable business workflows and background maintenance. Services may own payments, carts, stock reservations, delivery, tickets, reports, automations, transcripts, backups, external APIs, and expiration/retry jobs. Keep Discord handler concerns outside the service when the operation can be reused or tested independently.

Services should also own lifecycle transitions, orphan reconciliation, capacity guards, maintenance pruning, event-specific audit retention, and idempotent backfills when those rules are shared across multiple handlers.

### `src/emojis` and application registry

Store local custom-emoji image assets in `src/emojis/` using stable semantic names. Synchronize them into application emojis at startup and persist the mapping in `src/config/application-emojis.json`. Components and logs resolve emojis through this registry instead of hardcoded IDs or scattered Unicode symbols.

### `test`

Store focused automated tests for pure utilities and high-risk domain behavior such as money, placeholders, reservations, payments, coupon usage, and lifecycle transitions.

Tests are expected for critical helpers when practical, especially permission decisions, TTL/retention cutoffs, orphan reconciliation, cache invalidation, state transitions, money totals, stock/coupon reservations, and Discord capacity guards.

### `src/mongoose/schemas`

Store the persistent data model. Register every schema in `src/mongoose/index.js` and expose it via `client.database`.

Schemas that store temporary or high-volume data should include `expiresAt`, `cleanupAt`, TTL indexes, or a documented scheduled-pruning strategy. Long-lived business schemas should keep explicit `status`, terminal timestamps, and compact summaries rather than unbounded raw payloads.

### `src/utils`

Store cross-cutting helpers:

- `antiCrash.js` for process safety
- `functions.js` for domain utilities
- `logger.js` for standard logs
- `reloadLocalCache.js` for cached guild configuration

## Naming Conventions

- Keep business categories short and folder-driven: `tickets`, `ticket`, `config`, `central`, `pagamento`, `analise`.
- Keep interaction names aligned with intent: `abrirTicket`, `selecionarMotivo`, `setLogs`, `toggle`.
- For panel navigation, prefer explicit back names such as `backToPainel`, `backToCentral`, or `backTo<NomeDoPainel>` while keeping the `categoria/acao` pattern.
- Keep slash command groups semantic: `miscellaneous`, `ticket`, `tickets`, `config`, `vendas`.
- Keep schemas singular and PascalCase: `Guild`, `Ticket`, `Sale`, `User`, `Divida`.

## UI and Panel Architecture

- New panels and interactive administration screens should use Components V2 through the project's shared builders when available, including `MessageFlags.IsComponentsV2`.
- The central panel is the management entrypoint for guild configuration and supported administrative systems. New manageable settings should be reachable from it or from a clearly linked child panel.
- Buttons and select-menu options should use the centralized custom application-emoji registry, with one stable emoji per semantic action across the bot.
- Panel renderers remain in `src/messages/**` and should be refreshable after mutations. Child panels should preserve origin/context and expose a visible back action to the parent or central panel.
- Keep visual layout, colors, separators, action rows, labels, and emoji assignment in shared Components V2 helpers when those helpers exist; do not rebuild the visual system independently in each feature.
- Management panels should include enough state for an admin to understand the feature without opening the database: enabled/disabled state, configured channels/roles, publication status, last update, cooldown/next run when relevant, and any disabled reason.

## When to Mirror Each Layer

- Create `commands/` when the bot needs prefix compatibility.
- Create `SlashCommands/` when the bot needs deployable Discord application commands.
- Create `messages/` when there are panels that must be regenerated or refreshed.
- Create `mongoose/schemas/` when feature state needs persistence.
- Create `interactions/` whenever buttons, modals, or selects are part of the UX.
- Create modal-driven configuration flows when selecting users, cargos, canais, mentionables, or predefined options inside a focused form is clearer than stacking multiple message components.

## Components 14.28.1: fluxo de arquivos e formulários

Leia [components-14.28.1.md](components-14.28.1.md) para o conjunto completo reutilizável. Preserve o fluxo `SlashCommands → messages → Button → modal → ModalSubmit → services → client.database → messages`. File (13) é apresentação de anexo em mensagem V2; FileUpload (19), RadioGroup (21), CheckboxGroup (22) e Checkbox (23) são entradas de modal dentro de Label. Não criar pastas de interação RadioGroup/Checkbox: a entrada é sempre `ModalSubmit`.

O renderer `src/messages/components/panel.js` utiliza `createContainer` e `emoji`; modal builders ficam em `src/utils/componentProfileModal.js`; validação, leitura dos campos, download limitado, digest e persistência ficam em `src/services/componentProfile.js`; o schema é carregado pelo `src/mongoose/index.js` existente. Os handlers exportam `(client, interaction)` e usam fallback de customId com `_` do dispatcher central. Não registrar listeners em arquivos da funcionalidade.

Uma submissão deve validar todos os campos antes da gravação atômica, reler o estado canônico e editar a mensagem de origem. Para registros pessoais sem cache, reler o MongoDB é suficiente; ao alterar configuração em `client.configs`, recarregar o cache da guild antes do renderer. Índice único e controle de versão protegem concorrência; o ID do documento distingue recriações após TTL. Campos pessoais têm retenção explícita, sem aplicar TTL a evidência comercial ou operações em andamento.

Mantenha a separação existente entre painel com File e log com anexos: somente o primeiro pode ser V2. Logs com anexos permanecem diretamente em `{ embeds, files }`, fora de helpers/decoradores V2.
