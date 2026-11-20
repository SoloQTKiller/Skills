# Operational Patterns Reference

Use this reference when a SoloQTKiller-base bot needs management panels, modal configuration, destructive actions, cooldowns, diagnostics, bot customization, or repeated API work. These patterns come from repeated requirements in the Moreira-Sensi bot.

## Components V2 Separators

- Prefer the shared `src/utils/componentsV2.js` helpers for new panels.
- `createContainer(color, content)` should split the first line into the title, add a large separator, then render the remaining body as a second text block.
- Action rows should be visually separated from content. A helper may patch or wrap `ContainerBuilder#addActionRowComponents` to insert one small separator before the first action row.
- Use semantic accent colors: `info`, `success`, `warning`, `danger`, `attention`, and `neutral`.
- Avoid dense panels with one long text block and immediate buttons. Use section lines, status lines, and separators so admins can scan quickly.
- Put Markdown heading markers before emojis (`## <:config:id> Configurações`), because Discord does not render the heading correctly when the emoji precedes `##`.
- When select menus and buttons coexist, add all select rows first. Action, navigation, and pagination buttons always follow them.
- The shared helper must be the single owner of the separator between current-state text and interactive components.

## Same-Message Configuration

- Keep a panel configuration flow in one message. Buttons may replace the panel with channel, role, user, bot, category, or mentionable selects; submit handlers rebuild the originating panel in that message.
- Prefer `interaction.update()` for navigation and mutation. Use `reply()` or `followUp()` only for private progress, transient confirmation, or safe errors that must coexist with the panel.
- After every mutation: persist, reload/invalidate guild cache, re-read canonical state, then render. Never render unsaved optimistic values.
- Every configuration panel lists all supported settings, including explicit empty states.
- List/select entries reflect canonical stored name, emoji, status, and description instead of generic replacements.

## Management Panel Contract

Every administrable system should have a management panel when it is configured repeatedly or has multiple states.

A management panel should show:

- Current enabled/disabled state.
- Configured channels, roles, users, categories, panel publication, destination, and logs when relevant.
- Last run, next run, cooldown, queue, active operation count, disabled reason, or failure state when the feature has automation.
- Actions for create, edit, remove, enable, disable, publish, repost, sync, refresh, diagnostics, and back navigation as applicable.
- Safe empty states such as "Nenhum painel criado", "Não configurado", "Indisponível", or "Pendente".

Render panels from `src/messages/**`, refresh them through `client.functions.refreshMessageData(...)`, and keep handlers focused on validation, persistence, and interaction acknowledgement.

## Modal Origin and Refresh

When a button opens a modal that changes configuration, preserve the originating panel before calling `showModal`.

Recommended behavior:

- Use a helper like `src/utils/panelOrigin.js`.
- Track origin by guild, user, modal custom id, source channel, source message, renderer, optional renderer data, and an expiry such as 15 minutes.
- After modal submit, persist the change, reload local config/cache, refresh the tracked panel, then send a success response.
- If the source message or channel no longer exists, keep the saved change and skip only the visual refresh.
- Use centralized modal success/error payloads so responses match the visual system.

Modal fields should use `LabelBuilder` and may include text inputs, string selects, user selects, role selects, mentionable selects, and channel selects. Prefill current values with `setValue(...)`, selected defaults, or descriptions explaining why a sensitive value is not displayed.

## Destructive Actions

Destructive or high-impact actions require a confirmation step.

Examples:

- Closing or deleting tickets.
- Deleting panels, options, coupons, rules, counters, or automations.
- Clearing stock or imports.
- Reposting a public panel after deleting the previous message.
- Restoring a backup.
- Removing access users/roles.
- Changing bot identity.

The confirmation message should explain impact, use Components V2, use danger/warning styling, be ephemeral when appropriate, and include clear confirm/cancel buttons. After confirmation, re-check permissions and current state because the target may have changed while the confirmation was open.

## Cooldowns, Locks, and Rate Limits

Button disable state is not enough. Store cooldowns or locks in persistent data when the operation affects shared state, calls Discord heavily, or calls an external API.

Recommended examples:

- Ticket notification cooldown: store `lastNotificationAt` and require a short wait such as 5 minutes.
- Counters and channel renames: enforce a minimum interval such as 5 minutes and store `nextUpdateAt`, `lastUpdatedAt`, and `failureCount`.
- Bot username changes: enforce a long cooldown such as 2 days.
- Bot avatar/banner changes: enforce a long cooldown such as 6 hours.
- Ticket/cart creation: use `TicketOperation` or `CartOperation` with a unique key, expiry, per-guild queue, global queue, and per-user failure cooldown.
- Follow-up automation: store `lastFollowUpAt`, `followUpCount`, `repeatMinutes`, and `maxFollowUps`.
- External APIs and payments: use idempotency keys, pending records, retry timestamps, and clear terminal states.

On `429`, do not loop blindly. Honor `retry_after`, use a bounded backoff helper, and avoid automatic retries for high-risk identity changes unless the project explicitly tracks a safe retry window.

Scheduled jobs need locks when overlap can duplicate side effects. Counter renames, lifecycle cleanup, delivery jobs, payment expiration, audit pruning, backfills, command sync, and emoji sync should store a lock or operation record with `lockedAt`, `lockedBy`, `expiresAt`, `lastRunAt`, `nextRunAt`, `failureCount`, and `status` when duplicate runs would be unsafe.

## Crash Recovery and Feature Flags

Startup maintenance should recover interrupted work before accepting expensive new work.

Recommended recovery:

- Expire stale operation locks stuck in `processing`.
- Release expired reservations.
- Reconcile missing channels, threads, panel messages, webhooks, roles, commands, and emojis.
- Resume or fail pending payments, deliveries, ticket/cart openings, imports, and backfills based on explicit state.
- Log recovered or suspicious state privately.

Risky systems should use feature flags or gradual configuration. Store flags per guild/feature, check them in every entrypoint, show state in management panels, and default incomplete or risky features to disabled/admin-only until configured.

## Friendly Errors

Interaction handlers should separate safe user messages from private technical logs.

- Use short Components V2/status payload messages for users and admins.
- Log unexpected errors with context, not just the message.
- Do not expose stack traces, provider payloads, tokens, file paths, raw database documents, or private evidence.
- Treat deleted resources and expired interactions as expected when the flow can legitimately remove them.
- Prefer domain errors for validation, permission, cooldown, capacity, missing config, and missing resource cases.

## Retention and TTL for Operational Data

Operational data must not grow forever. Any feature that writes frequent audit entries, operation attempts, pending workflows, temporary imports, reports, reviews, delivery attempts, fraud evidence, or transcript metadata should define retention before launch.

Recommended patterns:

- Short-lived operation locks and drafts use `expiresAt` with a TTL index.
- Records that need a final state first use `cleanupAt` with a partial TTL index after they become closed, expired, cancelled, delivered, failed, or orphaned.
- Audit logs use event-specific pruning when different event types need different retention windows.
- Payment confirmations, delivery proofs, delivered order summaries, refunds, chargebacks, and invoice-like records may need long retention such as 5 years; keep compact proof records for that window and prune bulky transient payloads separately.
- Public/business records keep compact summaries; bulky raw payloads, private evidence, temporary attachments, and repeated diagnostics get their own retention.
- Maintenance jobs should prune disabled/noisy audit types and slim or delete expired evidence before the database becomes heavy.

When adding a new high-volume collection, document the retention window, the field that controls cleanup, and the maintenance job or TTL index responsible for enforcing it.

## Machine Health and Diagnostics

For large bots, add a health service and central diagnostics panel.

Health signals may include:

- CPU usage.
- Memory pressure.
- Event-loop lag.
- MongoDB ping latency.
- Active operations.
- Queue depth.
- Current degraded/critical state.

Use degraded state to reduce internal concurrency. Use critical state to reject new expensive operations with a friendly message. Notify a configured log channel when the state changes if the bot has operational logs configured.

## Bot Customization

When the bot allows administrators to change username, nickname, avatar, or banner:

- Validate cooldown before showing or submitting the change.
- Validate image type and size before uploading.
- Normalize images through a library such as `sharp` when the project already uses it.
- Do not retry identity changes automatically after Discord returns a rate limit.
- Persist `lastFailureAt` and the last successful update timestamp.
- Never expose tokens or secret values in panels, modals, logs, or backups.

## Protected Resources

When the bot manages native Discord resources such as AutoMod rules, webhooks, commands, or channels, track whether the resource is managed by the bot.

- Bot-managed resources can be edited, toggled, synced, or deleted through the panel.
- External resources should be displayed as protected or read-only unless the project explicitly imports ownership.
- Handlers should re-fetch the native Discord resource before editing and handle missing resources by marking the local record as stale or unavailable.

## Pagination and Select Limits

Discord selects support at most 25 options. Panels with unbounded records should paginate or provide search/filter flows.

- Use `.limit(25)` for select-backed pages unless the UI intentionally uses fewer options.
- Use previous/next buttons with page numbers in the `customId`.
- Keep `customId` suffixes predictable, such as `categoria/lista_2` or `categoria/item_<id>`.
- Show an empty state instead of rendering invalid selects with no options.
- Keep labels and descriptions within Discord's component limits and truncate safely.
- Select option descriptions have a 100-character maximum. Enforce it in modal input and schema validation, then truncate defensively for legacy records.

## Published Panel Lifecycle

Treat publish/update and repost as distinct operations:

- `Publicar/atualizar` edits the stored message directly when its channel/message ids still resolve; do not ask for a channel again.
- If the stored publication cannot be fetched, clear stale pointers, persist the unpublished/orphaned state, and then ask where to publish.
- First publication asks for a channel.
- `Repostar` always asks for a destination. Send and persist the new message first; only then delete the old one.
- Re-fetch native Discord resources before editing or deleting. A missing publication is an expected reconciliation case.

```js
if (panel.publishedChannelId && panel.publishedMessageId) {
  const updated = await refreshPublishedPanel(client, panel);
  if (updated) return interaction.update(await renderManagement(panel));
  panel.publishedChannelId = undefined;
  panel.publishedMessageId = undefined;
  await panel.save();
}
return interaction.update(await renderChannelPicker(panel));
```

## Configurable Guild Emojis

- Optional emoji input left blank means omit the `emoji` property entirely; never pass `null`/`undefined` to a Discord builder.
- For `<:name:id>`, validate the complete format, fetch from the current guild, reject foreign emojis, and persist canonical `id`, `name`, and `animated`.
- Reject `<a:name:id>` clearly when a field supports static emojis only.
- Application emoji synchronization uploads each asset with exactly the PNG base filename; never silently translate or rename it.
- The local registry is only a cache. Fetch the application's real remote emoji collection on every startup and reconcile it against every local asset before rendering dependent UI.
- A remote emoji discovered only by matching name is not proof of asset identity. Reuse only a remotely present registry id whose exact name and stored file/hash match the current local file; otherwise recreate it from the local asset.
- Do not swallow required startup synchronization errors and then start emoji-dependent panels/services with stale ids.
- Test valid, blank, malformed, foreign, animated/static, and no-emoji rendering paths.

## Backups and Sensitive Data

Backup/export features should avoid tokens, payment secrets, user private data, customer stock payloads, and credentials unless the user explicitly requests a secure migration artifact and the project has a safe transport.

Restore flows should require confirmation, validate the JSON shape, report what will change, and refresh the relevant central panels after completion.

## Logs With File Attachments

Any log that carries a file is a mandatory legacy-message path, even when the rest of the bot uses Components V2.

- Send it directly with `channel.send({ embeds: [...], files: [...] })`.
- Do not add `components` or `MessageFlags.IsComponentsV2`.
- Do not pass it through `createContainer`, `refreshMessageData`, `decoratePayload`, automatic interaction decorators, or any helper that converts payloads to Components V2.
- This applies to ticket/cart transcripts, report exports, evidence, backup archives, receipts, CSV/PDF/HTML exports, and similar log deliveries.
- Custom emojis may appear as text inside the conventional embed; this does not turn the message into Components V2.
- When the file is a safety boundary, fetch the sent message and confirm the expected attachment before closing, deleting, or marking the source workflow complete.
- Add a focused test asserting `files` and `embeds` are present while `components` and the `IsComponentsV2` flag are absent.
- Ordinary interactive panels that display an attachment-backed image are not automatically logs; this exception is specifically mandatory for log/delivery messages whose purpose includes delivering the file.

## File, uploads e escolhas em modais (14.28.1)

Exemplos completos de cada fluxo e do conjunto integrado: [components-14.28.1.md](components-14.28.1.md). Use `createContainer`, `payload`, `emoji` e os loaders existentes. FileBuilder é exibição de bytes anexados pelo bot, não upload de usuário; FileUploadBuilder recebe anexos resolvidos via `interaction.fields.getUploadedFiles`. Radio e Checkbox chegam pelo ModalSubmit, com getters próprios, sem handlers/listeners adicionais.

- Antes de showModal, validar origem, guild, propriedade/permissão e versão. showModal precisa ser a primeira resposta; não deferir o Button. No ModalSubmit, revalidar permissão e usar deferUpdate para a origem em mensagem antes de download/MongoDB. Para modal sem origem, deferReply ephemeral e usar panelOrigin em vez de presumir que existe mensagem.
- Checkbox false, CheckboxGroup [] e Radio opcional null têm semânticas diferentes. Preservar false/[] ao salvar; validar allowlists e cardinalidade novamente no serviço. Aceite obrigatório não é propriedade de CheckboxBuilder.
- Prefill com setDefault nas opções/grupos/Checkbox; não tentar preencher FileUpload com uma URL antiga. Mostrar o estado anterior no painel e explicar a substituição.
- File types filtra extensões no cliente; não é fronteira de validação. Validar quantidade, tamanho, MIME, origem CDN, bytes limitados e assinatura; não executar arquivos. Usar timeout, limitar redirects/hosts, evitar logs de URLs assinadas e não fazer download sem limite. Aplicar parser seguro/antivírus quando o domínio exigir.
- Para persistência, salvar todos os campos juntos após validação, com índice único e atualização condicional por revisão/ID do registro. Não sobrescrever silenciosamente uma submissão mais nova nem recriar o registro expirado no submit. Não armazenar URLs assinadas como prova durável. Definir armazenamento privado/retencão antes de prometer retenção dos bytes.
- Após gravar, invalidar/recarregar somente o cache usado e reconstruir o renderer de origem. Modal de mensagem permite editReply inclusive para origem ephemeral. Se o edit falhar após persistência, avisar que os dados foram salvos e permitir reabrir; não mandar a submissão repetir o efeito como se a gravação tivesse falhado.
- Cancelar/fechar modal não gera submit nem evento de cancelamento. Deixar o painel válido e expirar contextos temporários de origem; não implementar collector ou promessa de evento inexistente.
- File em painel V2 não altera a regra de logs: transcript, backup, recibo, evidência e outros logs com anexos devem sair diretamente como payload legado, sem flags/components/decoradores V2. Não transformar automaticamente todo payload com files em V2.
