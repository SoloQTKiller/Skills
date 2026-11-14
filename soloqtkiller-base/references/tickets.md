# Ticket System Reference

Read this reference completely before implementing or changing tickets in a SoloQTKiller-base bot. It documents the reusable system proven in RankedNexus-Salas: multiple public panels, independent options, channel/private-thread opening, staff controls, publication, transcripts, locking, recovery, and retention.

## Architecture

Keep Discord handlers thin and lifecycle rules in services:

```text
src/
  interactions/
    Button/tickets/              # configuration actions
    SelectMenu/tickets/          # panel/option selection and public opening
    ChannelSelectMenu/tickets/   # publication/destination
    RoleSelectMenu/tickets/      # staff roles when outside modals
    ModalSubmit/tickets/         # persist panel/option settings
    Button/ticket/               # actions on one open ticket
    UserSelectMenu/ticket/       # owner and participant changes
  messages/tickets/              # configuration renderers
  messages/ticket/               # open-ticket/staff renderers
  services/tickets.js            # lifecycle and permission rules
  mongoose/schemas/
    TicketPanel.js
    TicketOption.js
    Ticket.js
    TicketOperation.js
```

Use `tickets` for configuration/public-panel actions and `ticket` for one open ticket. Examples: `tickets/opcoes_<panelId>`, `tickets/publicar_<panelId>`, `tickets/abrir_<panelId>`, `ticket/admin_<ticketId>`, `ticket/encerrar_<ticketId>`.

## Persistent Models

### TicketPanel

```js
{
  guildId: String,
  name: String,
  description: String,
  imageUrl: String,              // optional HTTP/HTTPS
  enabled: Boolean,
  publishedChannelId: String,
  publishedMessageId: String,
  createdAt: Date,
  updatedAt: Date,
}
```

Validate the optional image as `http:`/`https:` and omit its media component when blank.

### TicketOption

Every option is independently configured:

```js
{
  guildId: String,
  panelId: ObjectId,
  name: String,                  // maximum 100
  description: String,           // maximum 100
  emoji: { id: String, name: String, animated: Boolean }, // optional
  enabled: Boolean,
  position: Number,
  mode: "channel" | "thread",
  destinationId: String,         // category for channel; parent for thread
  transcriptChannelId: String,
  teamRoleIds: [String],
  ticketMessage: String,
}
```

Do not create a separate “mentioned roles” setting: configured attendant roles are the staff roles. Emoji is optional and must not block activation. A public select supports at most 25 active options.

### Ticket

Freeze an option snapshot when opening so later edits do not mutate open tickets:

```js
{
  guildId: String,
  publicId: String,
  panelId: ObjectId,
  optionId: ObjectId,
  optionName: String,
  optionEmoji: Object,
  ownerId: String,
  participantIds: [String],
  assigneeId: String,
  teamRoleIds: [String],
  mode: "channel" | "thread",
  channelId: String,
  transcriptChannelId: String,
  ticketMessage: String,
  status: "opening" | "open" | "closing" | "closed" | "orphaned",
  openedAt: Date,
  closedAt: Date,
  cleanupAt: Date,
}
```

Use a partial unique key for active-ticket uniqueness when required. Never assign creation-time TTL to active tickets.

### TicketOperation

Use a short-lived unique record to absorb double clicks/repeated select submissions:

```js
{
  operationKey: String,          // guild + panel + option + user
  guildId: String,
  userId: String,
  status: "processing" | "completed" | "failed",
  expiresAt: Date,
}
```

Add unique `operationKey` and TTL `expiresAt` indexes. Finalize/release in the domain service, including failure paths.

## Management Panels

The Tickets central screen shows every current setting, including empty states: global status, panel count, page, each panel's name/status, and publication state. When both exist, select menus appear before buttons.

Put `Ativar/Desativar` before `Criar painel` on the Tickets root.

Panel-management layout:

```text
Ativar/Desativar | Editar painel | Gerenciar opções
Publicar/atualizar | Prévia | Repostar | Excluir
Voltar
```

Keep “Criar opção” only inside `Gerenciar opções`, not duplicated on the panel screen.

Option-management layout:

```text
Ativar/Desativar | Editar texto | Modos e canais | Configurar cargos
Subir | Descer | Excluir | Voltar
```

Ordinary actions use primary/blue, delete uses danger/red, activate uses success/green, deactivate uses danger/red. Movement uses registered `cima` and `baixo` emojis. Every back action returns to the exact panel/option context.

Panel create/edit requests name, description, and optional image URL. Option create/edit supports name, optional guild emoji (`<:name:id>`), description up to 100 characters, internal ticket message, mode, destination, transcript channel, and staff roles. Prefill existing values.

Before activation validate staff roles, destination type, transcript channel, and thread-parent access. Blank emoji clears it; a supplied emoji must belong to the current guild.

## Public Panel and Publication

Use Components V2, the shared `ticket` title emoji, optional configured image, and one string select of active options sorted by `position`.

```js
const optionData = {
  label: option.name.slice(0, 100),
  description: (option.description || "Abrir atendimento").slice(0, 100),
  value: option.id,
  ...(option.emoji?.id ? { emoji: option.emoji } : {}),
};
```

When no option is active, render an unavailable state without an empty select. Disable the select when the panel/global ticket system is disabled.

`Publicar/atualizar` edits the existing stored message when found and asks for a channel only on first publication or after missing-resource reconciliation. Clear stale ids first. `Repostar` always asks for a destination, saves the new message, and only then deletes the old one.

## Opening Workflow and Concurrency

Use this sequence:

1. Reset/update the public select immediately to remove Discord's cached selection.
2. Send ephemeral progress: “Criando seu ticket…”.
3. Validate feature, panel, option, user, destination, permissions, and capacity.
4. Acquire a unique `TicketOperation`.
5. Re-check active-ticket uniqueness after the lock.
6. Persist an `opening` ticket with the option snapshot.
7. Create the Discord channel/private thread.
8. Apply permissions and participants.
9. Send the ticket-local panel.
10. Persist `channelId`, transition to `open`, and complete the operation.
11. On failure, compensate partial resources, make database state truthful, and release the lock.

Use bounded guild/global queues. Check guild channel count, category children, active private threads, active tickets, destination validity, and Discord permissions before creation. Apply bounded `429` backoff to Discord writes.

## Channel Mode Permissions

The destination is a category. Create a private text channel with explicit overwrites:

- deny `ViewChannel` to everyone;
- allow owner and attendant roles to view;
- allow `SendMessages`, `ReadMessageHistory`, `AttachFiles`, `EmbedLinks`, and `SendVoiceMessages` where supported;
- give the bot required management permissions.

Do not depend only on inherited category permissions when privacy is contractual.

## Private-Thread Mode Permissions

The destination is a text/announcement parent channel:

- create a private thread with an appropriate short auto-archive duration;
- add the owner explicitly;
- do not allow the customer to invite arbitrary members;
- ensure bot view/manage access;
- ensure all attendant roles have `ViewChannel` and `ManageThreads` on the parent.

First attempt to repair missing attendant-role overwrites automatically. Only show “staff roles need View channel and Manage threads” when automatic correction fails. Convert Discord `50001 Missing Access` into a friendly error and private technical log.

## Ticket-Local and Staff Panels

The ticket panel uses custom emojis and displays public number, selected option, owner, assigned staff/“Aguardando equipe”, and the configured message with placeholders such as `{user}`, `{ticket}`, and `{option}`.

Main actions:

```text
Painel staff | Fechar ticket
```

Only staff may close. The staff panel supports assume/claim, change owner, add participant, remove participant when supported, and return. Authorize using the frozen `teamRoleIds` snapshot plus explicitly configured administrator policy, not the mutable option record.

For channel participants, update overwrites with the customer permission set. For private threads, use `thread.members.add(userId)` and handle missing access. Do not add duplicates or remove the current owner; transfer first. Add new owner access before removing old access. Persist after Discord succeeds or compensate on persistence failure. Refresh the local panel after each change.

## Closing and Transcripts

Closing is staff-only and requires explicit confirmation. Use `better-discord-transcripts`. The transcript destination is the option's frozen/configured transcript channel.

Transcript delivery is a safety boundary:

1. Atomically transition `open -> closing`.
2. Generate HTML.
3. Send a conventional Discord message directly with an embed plus the HTML attachment. Transcript logs are never Components V2: do not set `MessageFlags.IsComponentsV2`, do not include `components`, and do not pass the payload through `createContainer`, `refreshMessageData`, `decoratePayload`, interaction decorators, or any Components V2 conversion helper.
4. Re-fetch that log message by ID.
5. Confirm the expected attachment exists.
6. Retry a bounded number of times when delivery is uncertain.
7. Only after confirmation transition to `closed`, store compact metadata/set retention, and delete the ticket resource.
8. If confirmation fails, return the ticket to `open` and preserve its channel/thread.

Store compact metadata (log channel/message ids, attachment name/URL when safe, closedBy, closedAt, reason). Do not store transcript HTML in MongoDB unless explicitly required.

Correct transcript-log payload:

```js
const sent = await transcriptChannel.send({
  embeds: [new EmbedBuilder()
    .setTitle("Transcript de ticket")
    .setDescription(`Ticket: ${ticket.publicId}`)],
  files: [transcriptAttachment],
});
```

Forbidden transcript-log patterns:

```js
// Never do any of these for a log containing files:
await transcriptChannel.send({ components, files, flags: MessageFlags.IsComponentsV2 });
await transcriptChannel.send(decoratePayload(client, { embeds, files }));
await transcriptChannel.send(await client.functions.refreshMessageData(/* ... */));
```

```js
const claimed = await Ticket.findOneAndUpdate(
  { _id: ticketId, status: "open" },
  { $set: { status: "closing", closingBy: actorId } },
  { new: true },
);
if (!claimed) throw new Error("Este ticket já está sendo fechado.");

try {
  const receipt = await sendAndConfirmTranscript(claimed);
  await closeTicketAtomically(claimed.id, receipt);
  await deleteTicketResource(claimed);
} catch (error) {
  await Ticket.updateOne(
    { _id: claimed.id, status: "closing" },
    { $set: { status: "open" }, $unset: { closingBy: 1 } },
  );
  throw error;
}
```

## Recovery and Retention

Startup/maintenance reconciliation must expire stale operations, resolve `opening` tickets without resources, orphan `open` tickets whose resource disappeared, recover stale `closing` tickets safely, refresh published panels after option changes, and clear missing publication pointers.

Set `cleanupAt` only after terminal states. Active tickets have no cleanup date. Release active-ticket uniqueness when a ticket becomes closed/orphaned.

## Required Tests

- options remain independently configured;
- tickets freeze option/staff snapshots;
- duplicate opening creates one ticket;
- capacity failure creates no partial Discord resource;
- channel/thread permission contracts;
- automatic thread permission repair and failure path;
- optional/foreign emoji and 100-character description behavior;
- enabled, disabled, and no-options public panels;
- publish/update, missing-message republication, and repost safety;
- ownership/participant rules;
- only staff closes;
- transcript is confirmed before deletion;
- transcript failure preserves/reopens the ticket;
- orphan reconciliation releases uniqueness;
- navigation returns to the exact parent panel.

## Optional Guild Emoji Example

```js
async function resolveGuildEmoji(guild, input) {
  const value = String(input || "").trim();
  if (!value) return null;
  const match = /^<:([a-zA-Z0-9_]{2,32}):(\d{17,20})>$/.exec(value);
  if (!match) throw new Error("Use um emoji deste servidor no formato <:nome:id>.");
  const emoji = await guild.emojis.fetch(match[2]).catch(() => null);
  if (!emoji || emoji.guild.id !== guild.id)
    throw new Error("O emoji informado não pertence a este servidor.");
  return { id: emoji.id, name: emoji.name, animated: emoji.animated };
}
```

## Same-Message Refresh Example

```js
await option.save();
await refreshPublishedPanel(client, panel);
await interaction.update(
  await client.functions.refreshMessageData(
    client,
    interaction,
    "tickets/opcaoConfiguracao",
    option,
  ),
);
```
