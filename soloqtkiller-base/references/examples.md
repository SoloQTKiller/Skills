# Examples Reference

## Table of Contents

1. Bootstrap example
2. Index example
3. Config examples
4. Client example
5. Event examples
6. Prefix command example
7. Slash command example
8. Interaction examples
9. Modal examples
10. Message panel example
11. Components V2 separators and management panels
12. Modal origin refresh example
13. Cooldown and diagnostics examples
14. Data integrity and retention examples
15. Counter and placeholder examples
16. Schema example
17. Utility patterns
18. Application emoji registry example
19. Scaffold checklist

## Bootstrap Example

The reference bots use `index.js` to attach shared registries to `client` before loading files dynamically.

Write all user-facing texts with proper Portuguese accents by default. Preserve ASCII only for technical identifiers such as file names, variables, model names, database keys, and `customId` values.

Typical fields:

```js
client.config = require('./src/config/config.json');
client.localCache = {};
client.messages = new Discord.Collection();
client.slashCommands = new Discord.Collection();
client.commands = new Discord.Collection();
client.aliases = new Discord.Collection();
client.database = {};
client.logger = logger;
client.functions = require('./src/utils/functions');
```

Use this same pattern when creating a new base.

## Index Example

Use `index.js` as the main bootstrap file that wires config, client state, database, messages, commands, slash commands, events, cache reload, crash protection, and login. This file should represent the full base entrypoint.

Example:

```js
const fs = require('fs');
const path = require('path');
const Discord = require('discord.js');
const { Client, GatewayIntentBits, Options } = require('discord.js');

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    // Add only the intents required by the bot's implemented features.
  ],
  makeCache: Options.cacheWithLimits({
    ...Options.DefaultMakeCacheSettings,
    MessageManager: 50,
    GuildMessageManager: 50,
    ThreadManager: 100,
    GuildTextThreadManager: 100,
    GuildForumThreadManager: 100,
    ThreadMemberManager: 25,
    UserManager: 300,
    GuildMemberManager: {
      maxSize: 200,
      keepOverLimit: member => member.id === member.client.user?.id,
    },
    PresenceManager: 0,
    ReactionManager: 0,
    ReactionUserManager: 0,
    VoiceStateManager: 0,
    StageInstanceManager: 0,
  }),
  sweepers: {
    ...Options.DefaultSweeperSettings,
    messages: {
      interval: 1800,
      lifetime: 3600,
    },
    users: {
      interval: 3600,
      filter: () => user => user.id !== client.user?.id && !user.bot,
    },
    guildMembers: {
      interval: 3600,
      filter: () => member => member.id !== member.client.user.id && !member.voice.channelId,
    },
  },
});

const logger = require('./src/utils/logger');
const clientConfig = require('./src/config/client.js');

client.config = require('./src/config/config.js');
client.localCache = {};

client.messages = new Discord.Collection();
client.slashCommands = new Discord.Collection();
client.commands = new Discord.Collection();
client.aliases = new Discord.Collection();
client.database = {};
client.logger = logger;
client.functions = require('./src/utils/functions');

require('./src/utils/antiCrash')(process, client);
require('./src/utils/reloadLocalCache')(client);
require('./src/mongoose/index').start(client);

const messagesPath = path.join(__dirname, 'src', 'messages');
const slashCommandsPath = path.join(__dirname, 'src', 'SlashCommands');
const commandsPath = path.join(__dirname, 'src', 'commands');
const eventsPath = path.join(__dirname, 'src', 'client', 'events');

if (!fs.existsSync(messagesPath)) {
  fs.mkdirSync(messagesPath, { recursive: true });
}

fs.readdir(messagesPath, (error, categories) => {
  if (error) return logger.error(`(MENSAGENS): ${error}`);

  categories.forEach(category => {
    const categoryPath = path.join(messagesPath, category);

    fs.readdir(categoryPath, (innerError, files) => {
      if (innerError) return logger.error(`(MENSAGENS): ${innerError}`);

      logger.success(`(MENSAGENS): Foram carregados ${files.length} handlers na pasta ${category}`);

      files.forEach(file => {
        if (!file.endsWith('.js')) return;

        const filePath = path.join(categoryPath, file);
        const handler = require(filePath);
        const handlerName = file.split('.')[0];

        if (typeof handler !== 'function') {
          logger.warn(`(MENSAGENS): O ficheiro ${file} na pasta ${category} nao exporta uma funcao.`);
          return;
        }

        client.messages.set(`${category}/${handlerName}`, handler);
      });
    });
  });
});

const slashArray = [];

fs.readdir(slashCommandsPath, (error, categories) => {
  if (error) return logger.error(`(SLASHCOMMANDS): ${error}`);

  categories.forEach(category => {
    const categoryPath = path.join(slashCommandsPath, category);

    fs.readdir(categoryPath, (innerError, files) => {
      if (innerError) return logger.error(`(SLASHCOMMANDS): ${innerError}`);

      logger.success(`(SLASHCOMMANDS): Foram carregados ${files.length} SlashCommands na pasta ${category}`);

      files.forEach(file => {
        if (!file.endsWith('.js')) return;

        const command = require(path.join(categoryPath, file));
        if (!command?.name) return;

        client.slashCommands.set(command.name, command);
        slashArray.push(command);
      });
    });
  });
});

client.on('clientReady', async () => {
  client.guilds.cache.forEach(guild => {
    guild.commands.set(slashArray).catch(console.error);
  });
});

fs.readdir(commandsPath, (error, categories) => {
  if (error) return logger.error(`(COMANDOS): ${error}`);

  categories.forEach(category => {
    const categoryPath = path.join(commandsPath, category);

    fs.readdir(categoryPath, (innerError, files) => {
      if (innerError) return logger.error(`(COMANDOS): ${innerError}`);

      logger.success(`(COMANDOS): Foram carregados ${files.length} comandos na pasta ${category}`);

      files.forEach(file => {
        if (!file.endsWith('.js')) return;

        const command = require(path.join(categoryPath, file));
        if (!command?.help?.name) return;

        client.commands.set(command.help.name, command);

        if (Array.isArray(command.help.aliases)) {
          command.help.aliases.forEach(alias => {
            client.aliases.set(alias, command.help.name);
          });
        }
      });
    });
  });
});

fs.readdir(eventsPath, (error, files) => {
  if (error) return logger.error(`(EVENTOS): ${error}`);

  files.forEach(file => {
    if (!file.endsWith('.js')) return;

    const event = require(path.join(eventsPath, file));
    const eventName = file.split('.')[0];

    client.on(eventName, event.bind(null, client));
    delete require.cache[require.resolve(path.join(eventsPath, file))];
  });

  logger.success(`(EVENTOS): Foram carregados ${files.length} eventos.`);
});

client.login(clientConfig.token).then(() => {
  logger.success('(BOT): Index iniciada com successo.');
});
```

This example shows a full base-ready `index.js` with:

- `client`
- `config`
- `utils`
- `mongoose`
- `messages`
- `commands`
- `SlashCommands`
- `events`

If the project still uses `client.json` instead of `client.js`, change only the login/config imports and keep the same loading flow.

## Config Examples

The reference projects use JSON files in `src/config/`, but when the user asks for `config.js` or wants a JS-based config module, keep the same data split used in the SoloQTKiller base.

### `src/config/client.json`

Use this file for token and client identity details.

Example:

```json
{
  "token": "SEU_TOKEN_AQUI"
}
```

### `src/config/config.json`

Use this file for runtime configuration such as prefix, guild id, database URI, and static defaults.

Example:

```json
{
  "prefix": "!",
  "server": "123456789012345678",
  "mongoose": "mongodb://127.0.0.1:27017/moreira-bot"
}
```

### `src/config/client.js`

If the project should use JavaScript instead of JSON for client config, export the same information in object form.

Example:

```js
module.exports = {
  token: process.env.DISCORD_TOKEN || 'SEU_TOKEN_AQUI',
};
```

### `src/config/config.js`

If the project should use JavaScript instead of JSON for runtime config, keep the same keys used by the SoloQTKiller base so the rest of the loaders do not change.

Example:

```js
module.exports = {
  prefix: '!',
  server: '123456789012345678',
  mongoose: process.env.MONGO_URI || 'mongodb://127.0.0.1:27017/moreira-bot',
};
```

When converting the base from JSON to JS config files, update imports in `index.js` from:

```js
client.config = require('./src/config/config.json');
const token = require('./src/config/client.json').token;
```

to:

```js
client.config = require('./src/config/config.js');
const token = require('./src/config/client.js').token;
```

## Client Example

If the user asks for a `client.js`, treat it as the module responsible for constructing and exporting the configured Discord client instance. Declare only the intents required by the bot's actual events and features. Do not use all intents as a default; inventory the implementation first and document any privileged intent that is necessary.

Example:

```js
const { Client, GatewayIntentBits, Options } = require('discord.js');

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    // Add only the intents required by the bot's implemented features.
  ],
  makeCache: Options.cacheWithLimits({
    ...Options.DefaultMakeCacheSettings,
    MessageManager: 50,
    ThreadManager: 100,
    ThreadMemberManager: 25,
    UserManager: 300,
    PresenceManager: 0,
    ReactionManager: 0,
    ReactionUserManager: 0,
    VoiceStateManager: 0,
    StageInstanceManager: 0,
    GuildMemberManager: {
      maxSize: 200,
      keepOverLimit: member => member.id === member.client.user?.id,
    },
  }),
  sweepers: {
    ...Options.DefaultSweeperSettings,
    messages: {
      interval: 1800,
      lifetime: 3600,
    },
    users: {
      interval: 3600,
      filter: () => user => user.id !== client.user?.id && !user.bot,
    },
  },
});

module.exports = client;
```

If this module exists, `index.js` should import the client instead of instantiating it inline. Re-evaluate the explicit intent list whenever features are added or removed.

## Event Examples

### `messageCreate.js`

Responsibilities:

- ignore bots
- optionally restrict the bot to one guild
- resolve prefix command names and aliases

Pattern:

```js
module.exports = async (client, message) => {
  if (message.author.bot) return;
  if (message.guild && message.guild.id !== client.config.server) return;

  const prefix = client.config.prefix;
  if (!message.content.startsWith(prefix)) return;

  const parts = message.content.toLowerCase().split(' ');
  const name = parts[0].slice(prefix.length);
  const args = parts.slice(1);
  const command =
    client.commands.get(name) ||
    client.commands.get(client.aliases.get(name));

  if (command) await command.run(client, message, args);
};
```

### `interactionCreate.js`

Responsibilities:

- detect interaction type
- resolve handler by folder and `customId`
- execute slash commands from `client.slashCommands`

Pattern:

```js
const handlerPath = path.join(
  __dirname,
  `../../interactions/${type}/${category}/${action}.js`
);
```

This means the generated `customId` and the file path must always agree.

Always use `/` as the separator in `customId` values.

Correct:

```text
tickets/abrirTicket
config/setLogs
ticket/selecionarMotivo
```

Avoid `:` as a separator in this base.

When creating panel navigation, also keep the same `/` pattern for back actions.

Examples:

```text
central/backToPainel
config/backToCentral
ticketsConfig/backToPainel
```

## Prefix Command Example

Based on `commands/admin/central.js`, prefix commands usually wrap existing message/panel flows:

```js
exports.run = async (client, message) => {
  if (!message.member.permissions.has(Discord.PermissionsBitField.Flags.Administrator)) {
    return message.reply({ content: 'Sem permissao.' });
  }

  const mockInteraction = {
    guild: message.guild,
    member: message.member,
    user: message.author,
    editReply: async options => message.reply(options),
    deferReply: async () => Promise.resolve(),
  };

  return client.functions.refreshMessageData(client, mockInteraction, 'central/painelCentral');
};

exports.help = {
  name: 'central',
  aliases: ['painel'],
};
```

Use this pattern when a prefix command should reuse an existing panel renderer instead of duplicating embed code.

The example shows the minimum compatibility behavior, not a closed command schema. `exports.run` may receive additional context supported by the project's dispatcher, and `exports.help` may include metadata such as `category`, `usage`, `examples`, `permissions`, `cooldown`, `guildOnly`, or other project-specific requirements. Inspect the existing loader before changing the signature or metadata contract.

## Slash Command Example

Based on the references, slash commands normally publish a panel or trigger a controlled action:

The snippet below demonstrates the command entrypoint and routing shape. For new panels in projects that provide `src/utils/componentsV2.js`, use the project's Components V2 payload helpers, include `MessageFlags.IsComponentsV2`, and apply the registered application emoji to every actionable option.

```js
module.exports = {
  name: 'ticket-painel',
  description: 'Envia o painel de abertura de tickets no canal atual.',
  type: Discord.ApplicationCommandType.ChatInput,

  run: async (client, interaction) => {
    await interaction.deferReply({ flags: Discord.MessageFlags.Ephemeral });

    const row = new Discord.ActionRowBuilder().addComponents(
      new Discord.StringSelectMenuBuilder()
        .setCustomId('tickets/abrirTicket')
        .setPlaceholder('Selecione uma opcao...')
    );

    await interaction.channel.send({ components: [row] });
    await interaction.editReply({ content: 'Painel enviado.' });
  }
};
```

Use slash commands as entrypoints for panels, admin actions, or setup flows.

When several operations belong to the same domain, prefer one root command with subcommands, for example `/vendas criar`, `/vendas editar`, and `/vendas deletar`. Add separate top-level commands only when the operations are unrelated or a concrete Discord/project constraint requires it.

## Interaction Examples

### Select menu opening a ticket flow

Based on `SelectMenu/tickets/abrirTicket.js` and `SelectMenu/ticket/selecionarMotivo.js`, select menus usually:

1. validate config or schedule
2. check whether the user already has an open ticket
3. create or continue the next UI step
4. persist the result in MongoDB when needed

Common pattern:

```js
const tipo = interaction.values[0];
const existing = await client.database.tickets.findOne({
  guildId: interaction.guild.id,
  userId: interaction.user.id,
  tipo,
  status: 'aberto',
});
```

### Button closing a ticket

Based on `Button/ticket/fecharTicket.js`, destructive actions should:

1. validate permission
2. validate log destination
3. generate transcript
4. confirm transcript delivery
5. only then lock or delete the thread

This is an important base rule when implementing close/archive flows.

### Fallback for dynamic ids

The contas base supports custom ids like `tickets/fechar_<uuid>` by falling back from the exact filename to the base action name. Reuse that behavior whenever the action embeds an id or suffix.

### Back button between panels

When a panel opens another panel, include a button that returns to the previous screen instead of forcing the user to restart the entire flow.

Example:

```js
const row = new Discord.ActionRowBuilder().addComponents(
  new Discord.ButtonBuilder()
    .setCustomId('central/backToPainel')
    .setLabel('Voltar')
    .setStyle(Discord.ButtonStyle.Secondary)
);
```

Pair that button with a handler such as:

```js
module.exports = async (client, interaction) => {
  return client.functions.refreshMessageData(client, interaction, 'central/painelCentral');
};
```

## Modal Examples

Discord modals now support more than text inputs. In this base, prefer `LabelBuilder` to wrap every modal field and keep `customId` values on the same `/` routing pattern.

### Modal with pre-filled text input

Use this when the modal should open with the current config already filled in.

```js
const {
  ActionRowBuilder,
  LabelBuilder,
  ModalBuilder,
  TextInputBuilder,
  TextInputStyle,
} = require('discord.js');

const modal = new ModalBuilder()
  .setCustomId('config/editarMensagemBoasVindas')
  .setTitle('Editar mensagem de boas-vindas');

const mensagemAtual = client.configs?.welcome?.message || 'Olá, seja bem-vindo ao servidor!';

const mensagemInput = new TextInputBuilder()
  .setCustomId('config/mensagemBoasVindas')
  .setStyle(TextInputStyle.Paragraph)
  .setRequired(true)
  .setValue(mensagemAtual)
  .setPlaceholder('Digite a mensagem de boas-vindas');

const mensagemLabel = new LabelBuilder()
  .setLabel('Mensagem de boas-vindas')
  .setDescription('O modal já abre com o valor atual preenchido.')
  .setTextInputComponent(mensagemInput);

modal.addComponents(mensagemLabel);
await interaction.showModal(modal);
```

### Modal with custom string select

Use this when the user should choose one of your own predefined options inside the modal.

```js
const {
  LabelBuilder,
  ModalBuilder,
  StringSelectMenuBuilder,
  StringSelectMenuOptionBuilder,
} = require('discord.js');

const sistemaAtual = client.configs?.tickets?.modo || 'publico';

const modoSelect = new StringSelectMenuBuilder()
  .setCustomId('ticketsConfig/selecionarModo')
  .setPlaceholder('Selecione o modo do sistema')
  .setRequired(true)
  .addOptions(
    new StringSelectMenuOptionBuilder()
      .setLabel('Público')
      .setDescription('Painéis visíveis no canal configurado')
      .setValue('publico')
      .setDefault(sistemaAtual === 'publico'),
    new StringSelectMenuOptionBuilder()
      .setLabel('Privado')
      .setDescription('Abertura focada em interações privadas')
      .setValue('privado')
      .setDefault(sistemaAtual === 'privado'),
  );

const modoLabel = new LabelBuilder()
  .setLabel('Modo do sistema')
  .setDescription('Escolha como os tickets devem ser abertos.')
  .setStringSelectMenuComponent(modoSelect);

const modal = new ModalBuilder()
  .setCustomId('ticketsConfig/editarModo')
  .setTitle('Configurar modo dos tickets')
  .addComponents(modoLabel);

await interaction.showModal(modal);
```

### Modal with pre-selected user

Use this when the config already stores a user id.

```js
const { LabelBuilder, ModalBuilder, UserSelectMenuBuilder } = require('discord.js');

const responsavelAtual = client.configs?.tickets?.responsavel;

const userSelect = new UserSelectMenuBuilder()
  .setCustomId('ticketsConfig/selecionarResponsavel')
  .setPlaceholder('Selecione o responsável')
  .setMinValues(1)
  .setMaxValues(1)
  .setRequired(true);

if (responsavelAtual) {
  userSelect.addDefaultUsers(responsavelAtual);
}

const modal = new ModalBuilder()
  .setCustomId('ticketsConfig/editarResponsavel')
  .setTitle('Responsável pelo sistema')
  .addComponents(
    new LabelBuilder()
      .setLabel('Responsável principal')
      .setDescription('Escolha quem ficará responsável pelo fluxo.')
      .setUserSelectMenuComponent(userSelect)
  );

await interaction.showModal(modal);
```

### Modal with pre-selected role

```js
const { LabelBuilder, ModalBuilder, RoleSelectMenuBuilder } = require('discord.js');

const cargoAtual = client.configs?.tickets?.cargos?.suporte;

const roleSelect = new RoleSelectMenuBuilder()
  .setCustomId('ticketsConfig/selecionarCargoSuporte')
  .setPlaceholder('Selecione o cargo de suporte')
  .setMinValues(1)
  .setMaxValues(1)
  .setRequired(true);

if (cargoAtual) {
  roleSelect.addDefaultRoles(cargoAtual);
}

const modal = new ModalBuilder()
  .setCustomId('ticketsConfig/editarCargoSuporte')
  .setTitle('Cargo de suporte')
  .addComponents(
    new LabelBuilder()
      .setLabel('Cargo de suporte')
      .setDescription('Defina o cargo que será mencionado nos tickets.')
      .setRoleSelectMenuComponent(roleSelect)
  );

await interaction.showModal(modal);
```

### Modal with pre-selected channel

```js
const {
  ChannelSelectMenuBuilder,
  ChannelType,
  LabelBuilder,
  ModalBuilder,
} = require('discord.js');

const canalAtual = client.configs?.tickets?.canal;

const channelSelect = new ChannelSelectMenuBuilder()
  .setCustomId('ticketsConfig/selecionarCanalPainel')
  .setPlaceholder('Selecione o canal do painel')
  .setMinValues(1)
  .setMaxValues(1)
  .setRequired(true)
  .addChannelTypes(ChannelType.GuildText);

if (canalAtual) {
  channelSelect.addDefaultChannels(canalAtual);
}

const modal = new ModalBuilder()
  .setCustomId('ticketsConfig/editarCanalPainel')
  .setTitle('Canal do painel')
  .addComponents(
    new LabelBuilder()
      .setLabel('Canal do painel')
      .setDescription('Defina onde o painel principal será enviado.')
      .setChannelSelectMenuComponent(channelSelect)
  );

await interaction.showModal(modal);
```

### Modal with mentionable select

Use this when the config may accept either users or cargos.

```js
const { LabelBuilder, MentionableSelectMenuBuilder, ModalBuilder } = require('discord.js');

const mentionableSelect = new MentionableSelectMenuBuilder()
  .setCustomId('ticketsConfig/selecionarDestino')
  .setPlaceholder('Selecione usuários ou cargos')
  .setMinValues(1)
  .setMaxValues(5)
  .setRequired(false);

const modal = new ModalBuilder()
  .setCustomId('ticketsConfig/editarDestino')
  .setTitle('Destino do sistema')
  .addComponents(
    new LabelBuilder()
      .setLabel('Usuários ou cargos permitidos')
      .setDescription('Aceita mistura de usuários e cargos.')
      .setMentionableSelectMenuComponent(mentionableSelect)
  );

await interaction.showModal(modal);
```

### Reading modal submit data

In `ModalSubmit` handlers, read the submitted values from `interaction.fields`. Use text and string getters for custom text/options, and the resolved selection getters for users, cargos, canais, and mentionables.

```js
const Discord = require('discord.js');

module.exports = async (client, interaction) => {
  const mensagem = interaction.fields.getTextInputValue('config/mensagemBoasVindas');
  const modo = interaction.fields.getStringSelectValues('ticketsConfig/selecionarModo');
  const responsaveis = interaction.fields.getSelectedUsers('ticketsConfig/selecionarResponsavel', false);
  const cargos = interaction.fields.getSelectedRoles('ticketsConfig/selecionarCargoSuporte', false);
  const canais = interaction.fields.getSelectedChannels('ticketsConfig/selecionarCanalPainel', false);
  const destino = interaction.fields.getSelectedMentionables('ticketsConfig/selecionarDestino', false);

  const responsavelIds = responsaveis ? [...responsaveis.keys()] : [];
  const cargoIds = cargos ? [...cargos.keys()] : [];
  const canalIds = canais ? [...canais.keys()] : [];
  const userIdsDestino = destino?.users ? [...destino.users.keys()] : [];
  const roleIdsDestino = destino?.roles ? [...destino.roles.keys()] : [];

  await interaction.reply({
    content: 'Configurações recebidas com sucesso.',
    flags: Discord.MessageFlags.Ephemeral,
  });
};
```

Prefer saving only the ids you need in MongoDB, then reopening future modals with those same values prefilled or preselected.

### Notes for modal selects

- In modals, `required` is available for select menus and defaults to `true`.
- In modals, disabled select menus are not supported.
- For auto-populated selects, the current config can be shown as selected by default.
- For text inputs, the current config can be shown with `setValue(...)`.
- For modal fields, use `LabelBuilder` text and description to explain the current setting and what the user is changing.

## Message Panel Example

Panels in `src/messages/**` are refreshable renderers, not generic helpers.

For current projects, these renderers should use the shared Components V2 helpers and the centralized application-emoji registry. Any legacy `embeds` or message-component snippet in this reference is about the renderer lifecycle and routing contract only; do not copy its visual implementation when Components V2 is available.

Pattern:

```js
module.exports = async (client, interaction) => {
  await client.reloadConfigs();
  const cfg = client.configs.tickets || {};

  await interaction.editReply({
    embeds: [embedBuiltFromCfg],
    components: [row1, row2, row3, backRow],
  });
};
```

Use message files when multiple entrypoints must render the same screen. In child panels, include a back button whenever returning to the previous panel improves navigation flow.

## Components V2 Separators and Management Panels

Current panels should use a shared Components V2 helper that standardizes title, separators, body, action rows, accent colors, and application emojis.

Example helper:

```js
const {
  ContainerBuilder,
  MessageFlags,
  SeparatorBuilder,
  SeparatorSpacingSize,
  TextDisplayBuilder,
} = require("discord.js");
const { get: getApplicationEmoji } = require("./applicationEmojis");

const originalAddActionRowComponents =
  ContainerBuilder.prototype.addActionRowComponents;

ContainerBuilder.prototype.addActionRowComponents = function (...components) {
  if (!this.__separatorAdded) {
    this.addSeparatorComponents(
      new SeparatorBuilder().setSpacing(SeparatorSpacingSize.Small),
    );
    this.__separatorAdded = true;
  }
  return originalAddActionRowComponents.apply(this, components);
};

const colors = Object.freeze({
  info: 0x5865f2,
  success: 0x57f287,
  warning: 0xfee75c,
  danger: 0xed4245,
  attention: 0xf0a14a,
  neutral: 0x99aab5,
});

exports.createContainer = (color, content) => {
  const lines = String(content ?? "").split("\n");
  const title = lines.shift() || "";
  const body = lines.join("\n").trim();
  const container = new ContainerBuilder()
    .setAccentColor(colors[color] || colors.info)
    .addTextDisplayComponents(new TextDisplayBuilder().setContent(title))
    .addSeparatorComponents(
      new SeparatorBuilder().setSpacing(SeparatorSpacingSize.Large),
    );

  if (body) {
    container.addTextDisplayComponents(
      new TextDisplayBuilder().setContent(body),
    );
  }

  return Object.assign(container, { __separatorAdded: true });
};

exports.payload = (color, content, options = {}) => ({
  ...options,
  components: [exports.createContainer(color, content)],
  flags: (options.flags || 0) | MessageFlags.IsComponentsV2,
});

exports.emoji = (client, name, fallback = "") =>
  getApplicationEmoji(client, name, fallback);
```

Example management panel shape:

```js
const {
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  MessageFlags,
} = require("discord.js");
const { createContainer, emoji } = require("../../utils/componentsV2");

module.exports = async (client, interaction, panelId) => {
  const panel = await client.database.SalesPanel.findById(panelId);
  if (!panel) {
    return {
      components: [
        createContainer("danger", `${emoji(client, "alerta")} Painel indisponível\nEste painel foi removido.`),
      ],
      flags: MessageFlags.IsComponentsV2,
    };
  }

  const card = createContainer(
    panel.salesEnabled ? "info" : "warning",
    `## ${emoji(client, "painel")} Gerenciando Painel: ${panel.name}\n` +
      `${emoji(client, "canal")} **Canal:** ${panel.channelId ? `<#${panel.channelId}>` : "Não configurado"}\n` +
      `${emoji(client, "marcador")} **Status:** ${panel.salesEnabled ? "Vendas abertas" : "Vendas bloqueadas"}\n` +
      `${emoji(client, "painel")} **Publicação:** ${panel.messageId ? "Ativa" : "Pendente"}`,
  ).addActionRowComponents(
    new ActionRowBuilder().addComponents(
      new ButtonBuilder()
        .setCustomId(`vendasCentral/editar_${panel.id}`)
        .setLabel("Editar")
        .setEmoji(emoji(client, "editar"))
        .setStyle(ButtonStyle.Primary),
      new ButtonBuilder()
        .setCustomId(`vendasCentral/publicar_${panel.id}`)
        .setLabel("Publicar/atualizar")
        .setEmoji(emoji(client, "painel"))
        .setStyle(ButtonStyle.Primary),
      new ButtonBuilder()
        .setCustomId(`vendasCentral/toggle_${panel.id}`)
        .setLabel(panel.salesEnabled ? "Pausar" : "Ativar")
        .setEmoji(emoji(client, panel.salesEnabled ? "pausar" : "ativar"))
        .setStyle(ButtonStyle.Secondary),
      new ButtonBuilder()
        .setCustomId("configuracoes/paineisVendas")
        .setLabel("Voltar")
        .setEmoji(emoji(client, "voltar"))
        .setStyle(ButtonStyle.Secondary),
    ),
  );

  return { components: [card], flags: MessageFlags.IsComponentsV2 };
};
```

## Modal Origin Refresh Example

When a modal changes a setting from a panel, remember the source panel and refresh it after saving.

Button opening the modal:

```js
const {
  LabelBuilder,
  ModalBuilder,
  TextInputBuilder,
  TextInputStyle,
} = require("discord.js");
const origin = require("../../../utils/panelOrigin");

module.exports = async (client, interaction) => {
  const config = client.configs.get(interaction.guild.id);
  origin.remember(client, "relatorios", interaction, "configuracoes/relatoriosConfig");

  await interaction.showModal(
    new ModalBuilder()
      .setCustomId("configuracoes/salvarRelatorios")
      .setTitle("Relatórios automáticos")
      .addComponents(
        new LabelBuilder()
          .setLabel("Limite de estoque baixo")
          .setDescription("Use um número inteiro igual ou maior que zero.")
          .setTextInputComponent(
            new TextInputBuilder()
              .setCustomId("estoque")
              .setStyle(TextInputStyle.Short)
              .setRequired(true)
              .setValue(String(config.reports?.lowStockThreshold ?? 3)),
          ),
      ),
  );
};
```

Modal submit:

```js
const origin = require("../../../utils/panelOrigin");

module.exports = async (client, interaction) => {
  const threshold = Number(interaction.fields.getTextInputValue("estoque"));
  if (!Number.isInteger(threshold) || threshold < 0) {
    return interaction.reply(
      origin.modalError(client, "Limite inválido", "Informe um número inteiro igual ou maior que zero."),
    );
  }

  await client.database.GuildConfig.updateOne(
    { guildId: interaction.guild.id },
    { $set: { "reports.lowStockThreshold": threshold } },
    { upsert: true },
  );
  await client.reloadConfigs(interaction.guild.id);
  await origin.refresh(client, "relatorios", interaction);
  await interaction.reply(
    origin.success(client, "Relatórios atualizados", "O limite de estoque baixo foi salvo."),
  );
};
```

## Cooldown and Diagnostics Examples

Use persistent timestamps for sensitive or expensive actions.

Example ticket notification cooldown:

```js
module.exports = async (client, interaction) => {
  const ticket = await client.database.Ticket.findById(
    interaction.customId.split("_")[1],
  );
  if (!ticket) throw new Error("Ticket não encontrado.");

  if (
    ticket.lastNotificationAt &&
    Date.now() - ticket.lastNotificationAt.getTime() < 5 * 60_000
  ) {
    throw new Error("Aguarde alguns minutos antes de notificar novamente.");
  }

  const user = await client.users.fetch(ticket.ownerId);
  await user.send({ content: `Atualização no ticket #${ticket.publicId}.` });
  ticket.lastNotificationAt = new Date();
  await ticket.save();
};
```

Example diagnostic panel:

```js
const { ActionRowBuilder, ButtonBuilder, ButtonStyle, MessageFlags } = require("discord.js");
const { createContainer, emoji } = require("../../../utils/componentsV2");
const capacity = require("../../../services/operationCapacity");
const health = require("../../../services/machineHealth");

module.exports = async (client, interaction) => {
  const snapshot = await capacity.snapshot(client);
  const machine = health.snapshot();

  const card = createContainer(
    machine.state === "normal" ? "success" : "warning",
    `## ${emoji(client, "estatisticas")} Diagnóstico do sistema\n` +
      `${emoji(client, "marcador")} **Estado:** ${machine.state}\n` +
      `${emoji(client, "relogio")} **Operações ativas:** ${snapshot.activeGlobal}\n` +
      `${emoji(client, "estoque")} **Fila em memória:** ${snapshot.queuedGlobal}\n` +
      `${emoji(client, "configuracoes")} **CPU:** ${machine.cpu.toFixed(1)}%\n` +
      `${emoji(client, "configuracoes")} **Memória:** ${machine.memory.toFixed(1)}%\n` +
      `${emoji(client, "relogio")} **Event loop:** ${machine.lag}ms\n` +
      `${emoji(client, "mongodb")} **MongoDB:** ${machine.mongoMs}ms`,
  ).addActionRowComponents(
    new ActionRowBuilder().addComponents(
      new ButtonBuilder()
        .setCustomId("central/backToPainel")
        .setLabel("Voltar")
        .setEmoji(emoji(client, "voltar"))
        .setStyle(ButtonStyle.Secondary),
    ),
  );

  await interaction.reply({
    components: [card],
    flags: MessageFlags.Ephemeral | MessageFlags.IsComponentsV2,
  });
};
```

## Data Integrity and Retention Examples

Use these examples when a feature stores workflow state, creates Discord resources, writes audit/history records, or needs cache-safe configuration updates.

### Domain state with TTL cleanup

Use explicit states plus `expiresAt` or `cleanupAt` instead of relying only on booleans.

```js
const mongoose = require("mongoose");
const { randomUUID } = require("crypto");

const ticketOperationSchema = new mongoose.Schema(
  {
    _id: { type: String, default: () => randomUUID() },
    operationKey: { type: String, required: true, unique: true },
    guildId: { type: String, required: true, index: true },
    panelId: { type: String, required: true, index: true },
    userId: { type: String, required: true, index: true },
    status: {
      type: String,
      enum: ["processing", "completed", "failed"],
      default: "processing",
      index: true,
    },
    ticketId: String,
    channelId: String,
    expiresAt: { type: Date, required: true },
  },
  { timestamps: true, _id: false },
);

ticketOperationSchema.index({ expiresAt: 1 }, { expireAfterSeconds: 0 });

module.exports = mongoose.model("TicketOperation", ticketOperationSchema);
```

For business records that should only be removed after reaching a terminal state, prefer optional `cleanupAt` with a partial TTL index:

```js
ticketSchema.index(
  { cleanupAt: 1 },
  {
    expireAfterSeconds: 0,
    partialFilterExpression: { cleanupAt: { $type: "date" } },
  },
);
```

### Event-specific audit retention

When audit events need different retention durations, use a pruning helper instead of one global TTL.

```js
const RETENTION_MS = Object.freeze({
  ticket_aberto: 90 * 86_400_000,
  ticket_fechado: 90 * 86_400_000,
  pagamento_confirmado: 5 * 365 * 86_400_000,
  comprovante_entrega: 5 * 365 * 86_400_000,
  compra_realizada: 5 * 365 * 86_400_000,
  bloqueio_manual: 5 * 365 * 86_400_000,
});

const DISABLED_AUDIT_TYPES = Object.freeze([
  "automod_executado",
  "autorole_aplicado",
]);

exports.prune = async (client) => {
  await client.database.AuditLog.deleteMany({
    type: { $in: DISABLED_AUDIT_TYPES },
  });

  const now = Date.now();
  for (const [type, retention] of Object.entries(RETENTION_MS)) {
    await client.database.AuditLog.deleteMany({
      type,
      createdAt: { $lte: new Date(now - retention) },
    });
  }
};
```

Do not store large raw payloads in audit logs by default. Store ids, status, reason, and compact summaries, then keep detailed private evidence in a bounded collection with its own `cleanupAt`.

### Five-year payment and delivery proof

Payment confirmations and delivery proofs should be treated as compact commercial evidence, not as noisy operational logs. Keep the normalized proof for a long window such as 5 years, and prune heavy payloads earlier unless the project explicitly requires them.

```js
const mongoose = require("mongoose");
const { randomUUID } = require("crypto");

const proofRetentionMs = 5 * 365 * 86_400_000;

const orderProofSchema = new mongoose.Schema(
  {
    _id: { type: String, default: () => randomUUID() },
    guildId: { type: String, required: true, index: true },
    orderId: { type: String, required: true, index: true },
    userId: { type: String, required: true, index: true },
    type: {
      type: String,
      enum: ["payment_confirmed", "delivery_completed", "refund", "chargeback"],
      required: true,
      index: true,
    },
    provider: String,
    providerReference: String,
    amount: Number,
    currency: { type: String, default: "BRL" },
    deliveredRoleIds: [String],
    deliveredProductIds: [String],
    summary: String,
    occurredAt: { type: Date, required: true },
    cleanupAt: { type: Date, required: true },
  },
  { timestamps: true, _id: false },
);

orderProofSchema.index({ cleanupAt: 1 }, { expireAfterSeconds: 0 });

exports.createPaymentProof = async (client, order, payment) => {
  const occurredAt = payment.confirmedAt || new Date();
  return client.database.OrderProof.create({
    guildId: order.guildId,
    orderId: order.id,
    userId: order.userId,
    type: "payment_confirmed",
    provider: payment.provider,
    providerReference: payment.id,
    amount: order.total,
    occurredAt,
    cleanupAt: new Date(occurredAt.getTime() + proofRetentionMs),
    summary: `Pagamento confirmado para o pedido ${order.publicId}.`,
  });
};
```

If the project also stores transcript HTML, screenshots, or raw payment payloads, put those heavy artifacts in a separate collection or storage path with its own shorter `cleanupAt` unless the retention requirement says they must follow the same 5-year window.

### Orphan reconciliation

When a ticket, cart, panel, or automation points to a Discord resource, re-fetch the resource before editing it and mark missing resources explicitly.

```js
exports.reconcileTicketChannel = async (client, ticket) => {
  if (!ticket.channelId || ticket.status === "closed") return ticket;

  const channel = await client.channels
    .fetch(ticket.channelId)
    .catch(() => null);

  if (channel) return ticket;

  ticket.status = "orphaned";
  ticket.orphanedAt = new Date();
  ticket.orphanReason = "channel_missing";
  ticket.cleanupAt = new Date(Date.now() + 90 * 86_400_000);
  await ticket.save();

  await client.functions.writeAudit?.(client, {
    guildId: ticket.guildId,
    type: "ticket_fantasma_removido",
    userId: ticket.userId,
    data: {
      ticketId: ticket.id,
      channelId: ticket.channelId,
      reason: "O canal salvo não existe mais no Discord.",
    },
    notify: false,
  });

  return ticket;
};
```

### Layered permission helper

Centralize permission decisions so commands, buttons, modals, and confirmations agree.

```js
const { PermissionFlagsBits } = require("discord.js");

exports.canManageTickets = (client, interaction, ticket) => {
  if (!interaction.inGuild()) return false;
  if (interaction.memberPermissions?.has(PermissionFlagsBits.Administrator)) {
    return true;
  }

  const config = client.configs.get(interaction.guild.id);
  const allowedRoles = config?.tickets?.staffRoles || [];
  const hasStaffRole = allowedRoles.some((roleId) =>
    interaction.member.roles.cache.has(roleId),
  );

  if (hasStaffRole) return true;
  return ticket?.userId === interaction.user.id && ticket?.status === "open";
};
```

Always call the helper again on modal submit and destructive confirmation.

### Public vs private logs

Separate safe public messages from private diagnostics.

```js
exports.logPurchasePublic = async (client, order) => {
  await client.functions.writeAudit(client, {
    guildId: order.guildId,
    type: "compra_realizada",
    userId: order.userId,
    orderId: order.id,
    data: {
      total: order.totalText,
      itemCount: order.items.length,
    },
  });
};

exports.logPaymentFailurePrivate = async (client, order, error) => {
  await client.functions.writeAudit(client, {
    guildId: order.guildId,
    type: "falha_pagamento_privada",
    userId: order.userId,
    orderId: order.id,
    data: {
      provider: order.payment?.provider,
      error: error.message,
      code: error.code,
    },
    notify: false,
  });
};
```

Never include tokens, secrets, raw payment payloads, private evidence, or full database documents in public logs.

### Cache-safe config mutation

After writing config, reload the affected cache before refreshing the panel.

```js
module.exports = async (client, interaction) => {
  await client.database.GuildConfig.updateOne(
    { guildId: interaction.guild.id },
    { $set: { "tickets.enabled": true } },
    { upsert: true },
  );

  await client.reloadConfigs(interaction.guild.id);
  return client.functions.refreshMessageData(
    client,
    interaction,
    "configuracoes/tickets",
  );
};
```

Do not tell the admin a setting is active until both the database update and cache refresh succeeded.

### Migration/backfill shape

Backfills should be idempotent and safe to resume.

```js
exports.backfillTicketStatuses = async (client, { limit = 100 } = {}) => {
  const tickets = await client.database.Ticket.find({
    status: { $exists: false },
  })
    .sort({ createdAt: 1 })
    .limit(limit);

  for (const ticket of tickets) {
    const status = ticket.closedAt ? "closed" : "open";
    await client.database.Ticket.updateOne(
      { _id: ticket.id, status: { $exists: false } },
      { $set: { status } },
    );
  }

  return tickets.length;
};
```

Panels should tolerate both old and new shapes while the backfill runs.

### Crash recovery after startup

Run bounded, idempotent recovery after `clientReady` and MongoDB are available.

```js
exports.recoverAfterCrash = async (client) => {
  const now = new Date();

  await client.database.TicketOperation.updateMany(
    {
      status: "processing",
      expiresAt: { $lte: now },
    },
    {
      $set: {
        status: "failed",
        failureReason: "stale_after_restart",
      },
    },
  );

  await client.database.StockReservation.updateMany(
    {
      expiresAt: { $lte: now },
      releasedAt: { $exists: false },
    },
    {
      $set: {
        releasedAt: now,
        cleanupAt: new Date(now.getTime() + 2 * 86_400_000),
      },
    },
  );
};
```

Keep recovery safe to run more than once and log suspicious states privately.

### Feature flag check

Check feature flags in handlers and services, not only in panel renderers.

```js
exports.assertFeatureEnabled = (config, path, label) => {
  const enabled = path
    .split(".")
    .reduce((value, key) => value?.[key], config);

  if (enabled === false) {
    const error = new Error(`${label} está desativado neste servidor.`);
    error.expected = true;
    throw error;
  }
};

module.exports = async (client, interaction) => {
  const config =
    client.configs.get(interaction.guild.id) ||
    (await client.reloadConfigs(interaction.guild.id));

  exports.assertFeatureEnabled(config, "tickets.enabled", "O sistema de tickets");
  return client.functions.refreshMessageData(client, interaction, "tickets/painel");
};
```

### Scheduled job lock

Use a lock when a scheduled job can overlap or duplicate side effects.

```js
exports.withJobLock = async (client, jobName, ttlMs, run) => {
  const now = new Date();
  const lockId = `${jobName}:global`;
  const lockedBy = `${process.pid}:${Date.now()}`;
  const lock = await client.database.JobLock.findOneAndUpdate(
    {
      _id: lockId,
      $or: [
        { expiresAt: { $lte: now } },
        { lockedAt: { $exists: false } },
      ],
    },
    {
      $set: {
        jobName,
        lockedBy,
        lockedAt: now,
        expiresAt: new Date(now.getTime() + ttlMs),
        status: "running",
      },
    },
    { upsert: true, new: true },
  );

  if (lock.lockedBy !== lockedBy) return false;

  try {
    await run();
    await client.database.JobLock.updateOne(
      { _id: lockId, lockedBy },
      {
        $set: {
          status: "completed",
          lastRunAt: new Date(),
          failureCount: 0,
        },
        $unset: { lockedAt: "", expiresAt: "" },
      },
    );
    return true;
  } catch (error) {
    await client.database.JobLock.updateOne(
      { _id: lockId, lockedBy },
      {
        $set: {
          status: "failed",
          lastError: error.message,
          expiresAt: new Date(Date.now() + 60_000),
        },
        $inc: { failureCount: 1 },
      },
    );
    throw error;
  }
};
```

### Friendly interaction error

Reply with a safe message and log the technical details privately.

```js
exports.toUserError = (error) => {
  if (error?.expected) return error.message;
  return "Não foi possível concluir esta ação agora. Tente novamente em alguns instantes.";
};

exports.logInteractionError = (client, interaction, error) => {
  client.logger.error("Falha em interação.", {
    message: error.message,
    stack: error.stack,
    guildId: interaction.guildId,
    userId: interaction.user?.id,
    channelId: interaction.channelId,
    customId: interaction.customId,
    commandName: interaction.commandName,
  });
};
```

## Counter and Placeholder Examples

Counters and placeholders should be implemented as explicit helpers with tests.

### Counter schema

```js
const mongoose = require("mongoose");
const { randomUUID } = require("crypto");

const counterConfigSchema = new mongoose.Schema(
  {
    _id: { type: String, default: () => randomUUID() },
    guildId: { type: String, required: true, index: true },
    channelId: { type: String, required: true },
    type: {
      type: String,
      enum: [
        "membros",
        "membros_call",
        "clientes",
        "vendas_unicas",
        "vendas_total",
        "faturamento",
        "avaliacoes",
      ],
      required: true,
    },
    template: { type: String, required: true, maxlength: 100 },
    intervalMinutes: { type: Number, min: 5, default: 5 },
    enabled: { type: Boolean, default: true },
    lastValue: String,
    lastUpdatedAt: Date,
    nextUpdateAt: Date,
    failureCount: { type: Number, default: 0 },
    disabledReason: String,
    lastManualUpdateAt: Date,
  },
  { timestamps: true, _id: false },
);

counterConfigSchema.index({ guildId: 1, channelId: 1 }, { unique: true });

module.exports = mongoose.model("CounterConfig", counterConfigSchema);
```

### Counter service

```js
const MIN_INTERVAL_MINUTES = 5;
const TYPES = new Set([
  "membros",
  "membros_call",
  "clientes",
  "vendas_unicas",
  "vendas_total",
  "faturamento",
  "avaliacoes",
]);

exports.validate = ({ type, template, intervalMinutes }) => {
  if (!TYPES.has(type)) throw new Error("Tipo de contador inválido.");
  if (
    !template ||
    template.length > 100 ||
    !/\{(?:valor|membros|call|clientes|vendas|itens|faturamento|avaliacoes|servidor)\}/.test(template)
  ) {
    throw new Error("Use um template de até 100 caracteres com pelo menos um placeholder válido.");
  }
  if (!Number.isInteger(intervalMinutes) || intervalMinutes < MIN_INTERVAL_MINUTES) {
    throw new Error(`O intervalo mínimo é de ${MIN_INTERVAL_MINUTES} minutos.`);
  }
};

exports.render = (template, value, guild) =>
  template.replace(
    /\{(valor|membros|call|clientes|vendas|itens|faturamento|avaliacoes|servidor)\}/g,
    (match, key) => (key === "servidor" ? guild.name : String(value)),
  ).slice(0, 100);
```

Counter maintenance should set `nextUpdateAt`, reset `failureCount` on success, increment `failureCount` on failure, and disable the counter with `disabledReason` after repeated failures.

### Placeholder helper

```js
exports.renderWelcome = (content, member) =>
  String(content || "")
    .replaceAll("@user", `<@${member.id}>`)
    .replaceAll("<user.id>", member.id)
    .replaceAll("<user>", member.displayName)
    .replaceAll("<server>", member.guild.name);

exports.renderAutomatic = (content, guild) =>
  String(content || "").replaceAll("<server>", guild.name);
```

### Placeholder tests

```js
const test = require("node:test");
const assert = require("node:assert/strict");
const { renderAutomatic, renderWelcome } = require("../src/utils/placeholders");

test("substitui placeholders de boas-vindas", () => {
  const member = {
    id: "123",
    displayName: "Felipe",
    guild: { name: "Moreira" },
  };

  assert.equal(
    renderWelcome("@user <user> <user.id> <server>", member),
    "<@123> Felipe 123 Moreira",
  );
});

test("substitui o nome do servidor em mensagens automáticas", () => {
  assert.equal(
    renderAutomatic("Novidades em <server>", { name: "Moreira" }),
    "Novidades em Moreira",
  );
});
```

## Schema Example

`src/mongoose/schemas/Ticket.js` in the references shows the expected style:

```js
const ticketSchema = new mongoose.Schema({
  _id: { type: String, default: () => randomUUID() },
  guildId: { type: String, required: true },
  userId: { type: String, required: true },
  tipo: { type: String, required: true },
  status: { type: String, default: 'aberto' },
}, { timestamps: true, _id: false });
```

Guidelines:

- use singular model names
- prefer explicit ids and required guild/user relations
- keep feature config in `Guild`
- keep operational records in dedicated schemas like `Ticket`, `Sale`, `User`, `Divida`

## Utility Patterns

Useful utilities extracted from the references:

- `refreshMessageData(client, interaction, 'categoria/arquivo')`
- `checkTicket(client, interaction)`
- `createTicketThread(channel, options)`
- `hasPermission(client, interaction)`
- `isInsideOperatingHours(category, guildData)`
- `formatOperatingHours(category, guildData)`

Prefer adding shared behavior to `src/utils/functions.js` when more than one handler needs it.

## Application Emoji Registry Example

Use this pattern when a bot must register and reuse custom application emojis from inside the bot itself. The Moreira-Sensi base keeps local assets in `src/emojis/`, persists the synchronized registry in `src/config/application-emojis.json`, syncs the assets during `clientReady`, and exposes a helper for panels and buttons.

### Registry file

Start with an empty registry and let the startup sync fill ids and hashes:

```json
{}
```

After synchronization, entries follow this shape:

```json
{
  "configuracoes": {
    "file": "configuracoes.png",
    "hash": "sha256-do-arquivo-local",
    "emojiId": "123456789012345678",
    "animated": false
  }
}
```

Keep each local file name semantic, lowercase, and stable, such as `configuracoes.png`, `voltar.png`, `remover.png`, `ticket.png`, and `estatisticas.png`. The exact file basename becomes the emoji name used by panels and handlers. Reject an invalid uppercase/mixed-case basename instead of silently normalizing or renaming it.

### `src/utils/applicationEmojis.js`

This utility is responsible for validating local files, creating or updating application emojis, writing the registry, and making mention strings available through `client.applicationEmojis`.

```js
const crypto = require("crypto");
const fs = require("fs");
const path = require("path");

const emojiDirectory = path.join(__dirname, "..", "emojis");
const registryPath = path.join(__dirname, "..", "config", "application-emojis.json");
const supportedExtensions = new Set([".png", ".jpg", ".jpeg", ".webp", ".gif"]);
const maxBytes = 256 * 1024;

const hashFile = (file) =>
  crypto.createHash("sha256").update(fs.readFileSync(file)).digest("hex");
const readRegistry = () => JSON.parse(fs.readFileSync(registryPath, "utf8"));
const writeRegistry = (registry) =>
  fs.writeFileSync(registryPath, `${JSON.stringify(registry, null, 2)}\n`);

exports.sync = async (client) => {
  if (!fs.existsSync(emojiDirectory)) return;

  const registry = readRegistry();
  const localFiles = fs
    .readdirSync(emojiDirectory, { withFileTypes: true })
    .filter((entry) =>
      entry.isFile() &&
      supportedExtensions.has(path.extname(entry.name).toLowerCase()))
    .map((entry) => ({
      file: entry.name,
      name: path.basename(entry.name, path.extname(entry.name)),
      path: path.join(emojiDirectory, entry.name),
    }));

  const names = new Set();
  const eligibleFiles = [];

  for (const item of localFiles) {
    if (!/^[a-z0-9_]{2,32}$/.test(item.name)) {
      throw new Error(`Nome de emoji inválido: ${item.file}`);
    }
    if (names.has(item.name)) throw new Error(`Emoji duplicado: ${item.name}`);
    if (fs.statSync(item.path).size > maxBytes) {
      client.logger.warn(`Emoji ignorado por exceder 256 KiB: ${item.file}.`);
      continue;
    }
    names.add(item.name);
    eligibleFiles.push(item);
  }

  const remote = await client.application.emojis.fetch();
  const resolved = new Map();

  for (const item of eligibleFiles) {
    const hash = hashFile(item.path);
    const known = registry[item.name];
    const registered = known?.emojiId ? remote.get(known.emojiId) : null;
    const trusted = Boolean(
      registered &&
      registered.name === item.name &&
      known?.hash === hash &&
      known?.file === item.file,
    );
    let emoji = trusted ? registered : null;

    if (!trusted) {
      const stale = new Map();
      if (registered) stale.set(registered.id, registered);
      for (const value of remote.values())
        if (value.name === item.name) stale.set(value.id, value);
      for (const value of stale.values())
        await client.application.emojis.delete(value.id);
      emoji = await client.application.emojis.create({
        attachment: item.path,
        name: item.name,
      });
    }

    registry[item.name] = {
      file: item.file,
      hash,
      emojiId: emoji.id,
      animated: emoji.animated || false,
    };
    resolved.set(item.name, `<${emoji.animated ? "a" : ""}:${emoji.name}:${emoji.id}>`);
  }

  for (const name of Object.keys(registry)) {
    if (!names.has(name)) registry[name].orphaned = true;
  }

  writeRegistry(registry);
  client.applicationEmojis = resolved;
  client.logger.success(`${resolved.size} application emoji(s) sincronizado(s).`);
};

exports.get = (client, name, fallback = "") =>
  client.applicationEmojis?.get(name) || fallback;
```

The JSON registry is a cache, never the authority. The exact implementation may be adapted, but keep these invariants: fetch the real remote collection on every startup; validate exact basenames; reject duplicates and oversize assets; hash local files; reuse only a remote id whose name/file/hash registry data all match; recreate remote emojis found only by name or with untrusted metadata; persist confirmed remote ids; mark missing local entries as orphaned; and build `client.applicationEmojis` only from confirmed remote objects.

Add focused tests for: empty registry with empty remote application; stale JSON id; deleted remote emoji; same-name remote emoji without trusted registry metadata; changed local hash; invalid/mixed-case basename; duplicate basename; oversize asset; confirmed id/hash reuse without upload; and startup failure before dependent services begin.

### Startup sync in `src/client/events/clientReady.js`

Synchronize emojis after configs are loaded and before publishing or registering flows that render panels:

```js
try {
  await require("../../utils/applicationEmojis").sync(client);
} catch (error) {
  client.logger.error("Falha ao sincronizar application emojis.", error);
  throw error; // do not start emoji-dependent panels/services with an unverified registry
}
```

If the bot also sets the required SoloQTKiller Twitch streaming presence in `clientReady`, keep both responsibilities there: presence first, config/cache loading, emoji sync, slash command registration, then service startup.

### Components V2 helper

Expose application emojis through the shared Components V2 helper so panel files do not import the registry directly:

```js
const { get: getApplicationEmoji } = require("./applicationEmojis");

exports.emoji = (client, name, fallback = "") =>
  getApplicationEmoji(client, name, fallback);
```

Then use semantic names in every panel:

```js
const { ButtonBuilder, ButtonStyle, MessageFlags } = require("discord.js");
const { createContainer, emoji } = require("../../utils/componentsV2");

module.exports = async (client, interaction) => {
  const container = createContainer(
    "info",
    `## ${emoji(client, "configuracoes")} Central de configurações\n` +
      `${emoji(client, "ticket")} Tickets\n` +
      `${emoji(client, "estatisticas")} Relatórios`,
  ).addActionRowComponents((row) =>
    row.addComponents(
      new ButtonBuilder()
        .setCustomId("central/tickets")
        .setLabel("Tickets")
        .setEmoji(emoji(client, "ticket"))
        .setStyle(ButtonStyle.Primary),
      new ButtonBuilder()
        .setCustomId("central/voltar")
        .setLabel("Voltar")
        .setEmoji(emoji(client, "voltar"))
        .setStyle(ButtonStyle.Secondary),
    ),
  );

  await interaction.editReply({
    components: [container],
    flags: MessageFlags.IsComponentsV2,
  });
};
```

### Automatic emojis for legacy buttons

When the project has many legacy button builders, activate a small compatibility utility early in `index.js`:

```js
require("./src/utils/buttonApplicationEmojis");
```

The utility can map labels to semantic emoji names and apply registered application emojis automatically when a button does not already define one:

```js
const registry = require("../config/application-emojis.json");
const { ButtonBuilder } = require("discord.js");

const originalSetStyle = ButtonBuilder.prototype.setStyle;
const available = (name) =>
  registry[name]?.emojiId ? { id: registry[name].emojiId, name } : null;

const emojiNameForLabel = (label = "") => {
  const text = String(label).toLowerCase();
  if (/confirm/.test(text)) return "confirmar";
  if (/cancel/.test(text)) return "cancelar";
  if (/exclu|remov/.test(text)) return "remover";
  if (/voltar/.test(text)) return "voltar";
  if (/editar|renomear/.test(text)) return "editar";
  if (/adicionar|criar|novo/.test(text)) return "adicionar";
  if (/ticket|atendimento/.test(text)) return "ticket";
  if (/relat|estat.stic/.test(text)) return "estatisticas";
  if (/configur|gerenciar/.test(text)) return "configuracoes";
  return "marcador";
};

ButtonBuilder.prototype.setStyle = function setStyleWithApplicationEmoji(style) {
  if (!this.data.emoji) {
    const emoji = available(emojiNameForLabel(this.data.label));
    if (emoji) this.setEmoji(emoji);
  }
  return originalSetStyle.call(this, style);
};
```

Prefer explicit `setEmoji(emoji(client, "nome"))` in new Components V2 panels. Use automatic legacy mapping as a compatibility layer, not as the only source of visual identity.

## Scaffold Checklist

When asked to create a new bot in this base, generate at least:

```text
index.js
src/client/events/clientReady.js
src/client/events/interactionCreate.js
src/client/events/messageCreate.js
src/config/client.json
src/config/config.json
src/mongoose/index.js
src/utils/antiCrash.js
src/utils/functions.js
src/utils/logger.js
src/utils/reloadLocalCache.js
```

Then add only the feature folders that the request needs:

- `src/commands/...`
- `src/SlashCommands/...`
- `src/interactions/...`
- `src/messages/...`
- `src/services/...` for reusable business workflows or scheduled maintenance
- `src/emojis/...` and `src/config/application-emojis.json` when the bot uses custom application emojis
- `src/mongoose/schemas/...`
- `test/...` for focused tests of utilities and critical business rules
