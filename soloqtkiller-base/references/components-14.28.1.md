# Components — discord.js 14.28.1

## Contrato e fontes

Use esta referência com `examples.md`, `architecture.md`, `discord-limits.md` e `operational-patterns.md`. Exemplos CommonJS para a base SoloQTKiller, sem listeners ou loaders paralelos. Versão executada na validação: `discord.js@14.28.1`.

Fontes oficiais verificadas em 14/11/2026:

- [File](https://docs.discord.com/developers/components/reference#file), [File Upload](https://docs.discord.com/developers/components/reference#file-upload), [Radio Group](https://docs.discord.com/developers/components/reference#radio-group), [Checkbox Group](https://docs.discord.com/developers/components/reference#checkbox-group), [Checkbox](https://docs.discord.com/developers/components/reference#checkbox).
- [Releases oficiais](https://github.com/discordjs/discord.js/releases?after=10.0.1): `file types` entrou em 14.28.0; 14.28.1 corrige permissões de BaseChannel, não introduz todos estes componentes.
- API e serialização conferidas também no pacote oficial instalado: `discord.js/src/structures/ModalSubmitFields.js`, `ModalSubmitInteraction.js`, `Attachment.js` e `@discordjs/builders/dist/index.js` (1.15.x). Esses contratos de implementação complementam a documentação HTTP do Discord.

## Builders, propriedades e diferenças

| Recurso | Contexto e dados | Métodos específicos no discord.js 14.28.1 |
| --- | --- | --- |
| `FileBuilder` (13) | Mensagem V2: `file.url`, `spoiler`. Exibe um anexo enviado pelo bot; não recebe resposta. `name` e `size` são dados devolvidos pela API. | `setURL('attachment://nome.ext')`, `setSpoiler(boolean)`, `toJSON()` |
| `FileUploadBuilder` (19) | Filho de Label em modal: `custom_id`, `min_values`, `max_values`, `required`, `file_types`. Recebe IDs resolvidos em anexos. | `setCustomId`, `setMinValues`, `clearMinValues`, `setMaxValues`, `clearMaxValues`, `setRequired`, `setFileTypes`, `addFileTypes`, `clearFileTypes`, `toJSON` |
| `RadioGroupBuilder` (21) | Filho de Label: `custom_id`, `options`, `required`. Um valor ou `null`. | `setCustomId`, `addOptions`, `setOptions`, `spliceOptions`, `setRequired`, `toJSON` |
| `CheckboxGroupBuilder` (22) | Filho de Label: `custom_id`, `options`, `min_values`, `max_values`, `required`. Lista de valores, inclusive `[]`. | `setCustomId`, `addOptions`, `setOptions`, `spliceOptions`, `setMinValues`, `setMaxValues`, `setRequired`, `toJSON` |
| `CheckboxBuilder` (23) | Filho de Label: `custom_id`, `default`. Booleano, inclusive `false`. Não tem `required`, opções, texto próprio nem `setValue`. | `setCustomId`, `setDefault`, `toJSON` |

Todos herdam identificação numérica `setId`, `clearId` e `data` do ComponentBuilder. Não confundir `id` numérico com `custom_id` de roteamento. Grupos expõem `options` como builders; prefira seus métodos a modificar arrays internos. `RadioGroupOptionBuilder` e `CheckboxGroupOptionBuilder` têm `setLabel`, `setValue`, `setDescription`, `setDefault`, `toJSON`; propriedades correspondentes: `label`, `value`, `description`, `default`. Não aceitam emoji de opção como StringSelect. Use no máximo um default no Radio; defaults do Checkbox Group precisam respeitar a cardinalidade.

`LabelBuilder`: `setLabel`, `setDescription`, `clearDescription` e respectivamente `setFileUploadComponent`, `setRadioGroupComponent`, `setCheckboxGroupComponent`, `setCheckboxComponent`. O texto de Checkbox fica no Label. Modal: `ModalBuilder#setCustomId`, `setTitle`, `addLabelComponents` e `toJSON`; não recebe flag `IsComponentsV2`. `ContainerBuilder#addFileComponents` posiciona File dentro de um container, ou File pode ser componente raiz de mensagem. `AttachmentBuilder` envia os bytes em `files`; FileBuilder apenas os referencia.

### Leitura em interaction.fields

- `getUploadedFiles(customId, required = false)` → `Collection<Snowflake, Attachment> | null`. Itere `values()`, use `.size`; não é array de URLs nem `interaction.attachments`. O getter obrigatório pode lançar erro em campo vazio. Verifique também coleção vazia.
- `getRadioGroup(customId, required = false)` → `string | null`.
- `getCheckboxGroup(customId)` → `string[]`.
- `getCheckbox(customId)` → `boolean`. `false` é resposta válida, não campo ausente.
- `getField(customId, ComponentType...)` permite inspeção tipada; campos ausentes/tipo incorreto lançam erro. Não usar `getTextInputValue` para estes componentes.

### Limites e compatibilidade

Os limites detalhados estão em [discord-limits.md](discord-limits.md#components-14281). Em particular, File aceita somente `attachment://`, uma referência por componente; os outros quatro são exclusivos de modal dentro de Label, nunca ActionRow ou mensagem. Não há evento separado de Radio/Checkbox/Upload: chegam juntos ao `ModalSubmit`.

`FileUploadBuilder.setFileTypes()` está disponível a partir de **14.28.0**, junto com `addFileTypes`/`clearFileTypes` no builder correspondente. Aceita argumentos variádicos ou array: `.setFileTypes('image', '.pdf')`. Filtros são categorias `image`, `video`, `audio` ou extensões iniciadas por ponto; não MIME como `application/pdf`. Esse filtro é conveniência do seletor, não validação de conteúdo. Em versões anteriores com FileUpload, omita o filtro e valide no serviço; não injete manualmente `file_types` como se houvesse suporte garantido.

Não trate todo o recurso como novidade de 14.28.1. Para projetos antigos, confira exports, getters e o lockfile efetivamente instalado: atualizar apenas `@discordjs/builders` pode deixar o parser de discord.js sem suporte. Se faltar builder/getter, atualize o conjunto ou preserve o fluxo compatível (StringSelect, botão de confirmação, upload por mensagem) e explique a mudança de UX. Não publique componentes novos com leitura antiga. Fixe `14.28.1` no ambiente de homologação e respeite o Node exigido pelo projeto/dependências.

## Exemplo completo, isolado e integrado

Este conjunto implementa cinco fluxos executáveis no mesmo painel: Upload, Radio, Checkbox Group, Checkbox e Todos. Cada botão isolado abre apenas o seu componente e altera apenas aquele campo. Todos abre quatro Labels no mesmo modal. O quinto componente, File, aparece no painel como um resumo JSON gerado pelo bot — File não pertence ao modal. Assim os mesmos handlers/serviços cobrem exemplos individuais e o integrado sem duplicar arquitetura.

Instalação na base existente: copie os arquivos com os caminhos abaixo, mantenha `componentsV2.js` e seu registro de emojis, e deixe os loaders existentes descobrirem renderers, handlers, comando e modelo. `src/mongoose/index.js` deve registrar o modelo como `client.database.ComponentProfile` e aguardar os índices únicos antes de publicar a funcionalidade (`await client.database.ComponentProfile.init()` no fluxo já existente). Os helpers utilizados aqui são o `createContainer(color, content)`, `payload(color, content, options)` e `emoji(client, name)` documentados em `examples.md`; adapte somente a assinatura se o projeto já tiver outra. O hub `/central` pode navegar a este renderer com o mecanismo de painel existente. `/componentes` é uma entrada pessoal de demonstração, não um hub administrativo concorrente.

O painel é pessoal e ephemeral. A autorização é propriedade do usuário, verificada novamente no submit, além dos guards de guild da base. Em um painel administrativo reutilizado, acrescente o helper de permissão específico do projeto tanto no Button como no ModalSubmit. Nenhum upload de evidência é republicado: persistimos somente metadados validados e digest por 30 dias. URLs CDN assinadas não são armazenamento durável; caso precise guardar o PDF, use o adapter de armazenamento privado já existente antes de confirmar a persistência e aplique sua retenção. O exemplo não promete retenção dos bytes originais.

### src/mongoose/schemas/ComponentProfile.js

```js
const mongoose = require('mongoose');

const attachmentSchema = new mongoose.Schema({
  id: { type: String, required: true },
  name: { type: String, required: true },
  size: { type: Number, required: true },
  contentType: { type: String, required: true },
  sha256: { type: String, required: true },
}, { _id: false });

const schema = new mongoose.Schema({
  guildId: { type: String, required: true },
  userId: { type: String, required: true },
  revision: { type: Number, default: 0, min: 0 },
  priority: { type: String, enum: ['normal', 'urgente'], default: 'normal' },
  notifications: { type: [String], enum: ['email', 'discord'], default: [] },
  consent: { type: Boolean, default: false },
  attachments: { type: [attachmentSchema], default: [] },
  expiresAt: { type: Date, required: true },
}, { timestamps: true });
schema.index({ guildId: 1, userId: 1 }, { unique: true });
schema.index({ expiresAt: 1 }, { expireAfterSeconds: 0 });

module.exports = mongoose.models.ComponentProfile
  || mongoose.model('ComponentProfile', schema);
```

### src/services/componentProfile.js

```js
const { createHash } = require('node:crypto');

const MODES = Object.freeze(['upload', 'radio', 'group', 'checkbox', 'all']);
const MAX_BYTES = 2 * 1024 * 1024; // Limite interno, não limite universal do Discord.
const expiresAt = () => new Date(Date.now() + 30 * 24 * 60 * 60 * 1000);
const expected = message => Object.assign(new Error(message), { userFacing: true });
const key = interaction => ({ guildId: interaction.guildId, userId: interaction.user.id });

function parse(interaction) {
  const match = /^components\/(?:open|submit)_(upload|radio|group|checkbox|all)_(\d+)_(\d{17,20})_([a-f0-9]{24})$/.exec(interaction.customId);
  if (!interaction.inGuild() || !match || match[3] !== interaction.user.id) {
    throw expected('Este painel pertence a outro usuário ou expirou. Abra /componentes.');
  }
  const revision = Number(match[2]);
  if (!Number.isSafeInteger(revision)) throw expected('Versão de painel inválida.');
  return { mode: match[1], revision, recordId: match[4] };
}

async function load(client, interaction) {
  const Model = client.database.ComponentProfile;
  // TTL é assíncrono: remova explicitamente o registro já expirado.
  await Model.deleteOne({ ...key(interaction), expiresAt: { $lte: new Date() } });
  try {
    return await Model.findOneAndUpdate(key(interaction), {
      $setOnInsert: { ...key(interaction), expiresAt: expiresAt() },
    }, { upsert: true, new: true, setDefaultsOnInsert: true }).lean();
  } catch (error) {
    if (error.code !== 11000) throw error;
    const current = await Model.findOne(key(interaction)).lean();
    if (!current) throw error;
    return current;
  }
}

async function validatePdf(attachment, fetcher = fetch) {
  if (!attachment.name?.toLowerCase().endsWith('.pdf')
      || attachment.contentType?.split(';')[0] !== 'application/pdf'
      || !Number.isInteger(attachment.size) || attachment.size < 5
      || attachment.size > MAX_BYTES) {
    throw expected('Envie um PDF de até 2 MiB.');
  }
  const url = new URL(attachment.url);
  if (url.protocol !== 'https:' || !['cdn.discordapp.com', 'media.discordapp.net'].includes(url.hostname)
      || url.username || url.password || url.port) {
    throw expected('Origem do arquivo inválida.');
  }
  const response = await fetcher(url, { redirect: 'error', signal: AbortSignal.timeout(10000) });
  if (!response.ok || !response.body) throw expected('Não foi possível ler o arquivo. Envie-o novamente.');
  const reader = response.body.getReader();
  const chunks = [];
  let size = 0;
  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      size += value.byteLength;
      if (size > MAX_BYTES) throw expected('O arquivo excede 2 MiB.');
      chunks.push(Buffer.from(value));
    }
  } finally {
    await reader.cancel().catch(() => {});
    reader.releaseLock();
  }
  const bytes = Buffer.concat(chunks);
  if (size !== attachment.size || bytes.subarray(0, 5).toString('ascii') !== '%PDF-') {
    throw expected('O conteúdo recebido não corresponde ao PDF informado.');
  }
  // Assinatura não é antivírus nem parser PDF. Não executar/interpretar o documento.
  return {
    id: attachment.id, name: 'documento.pdf', size,
    contentType: 'application/pdf', sha256: createHash('sha256').update(bytes).digest('hex'),
  };
}

async function readValues(fields, mode, fetcher) {
  if (!MODES.includes(mode)) throw expected('Formulário inválido.');
  const has = name => mode === name || mode === 'all';
  const changes = {};
  if (has('radio')) {
    const priority = fields.getRadioGroup('priority', true);
    if (!['normal', 'urgente'].includes(priority)) throw expected('Prioridade inválida.');
    changes.priority = priority;
  }
  if (has('group')) {
    const values = fields.getCheckboxGroup('notifications');
    if (!Array.isArray(values) || values.length > 2 || new Set(values).size !== values.length
        || values.some(value => !['email', 'discord'].includes(value))) {
      throw expected('Seleção de notificações inválida.');
    }
    changes.notifications = values;
  }
  if (has('checkbox')) {
    const consent = fields.getCheckbox('consent');
    if (typeof consent !== 'boolean') throw expected('Resposta de consentimento inválida.');
    changes.consent = consent; // false deve ser salvo; não usar `if (consent)`.
  }
  if (has('upload')) {
    const files = fields.getUploadedFiles('documents', false);
    if (!files || files.size !== 1) throw expected('Envie exatamente um PDF.');
    changes.attachments = [await validatePdf(files.first(), fetcher)];
  }
  return changes;
}

async function save(client, interaction, revision, changes, recordId) {
  const saved = await client.database.ComponentProfile.findOneAndUpdate({
    ...key(interaction), _id: recordId, revision, expiresAt: { $gt: new Date() },
  }, { $set: { ...changes, expiresAt: expiresAt() }, $inc: { revision: 1 } }, {
    new: true, runValidators: true,
  }).lean();
  if (!saved) throw expected('O painel mudou ou expirou. Abra /componentes e tente novamente.');
  // Sem cache duplicado: o renderer sempre relê MongoDB. Se adaptar para configuração
  // em client.configs, aguarde client.reloadConfigs(guildId) aqui antes do renderer.
  return saved;
}

module.exports = { MODES, MAX_BYTES, expected, parse, load, validatePdf, readValues, save };
```

### src/utils/componentProfileModal.js

```js
const {
  ModalBuilder, LabelBuilder, FileUploadBuilder, RadioGroupBuilder,
  RadioGroupOptionBuilder, CheckboxGroupBuilder, CheckboxGroupOptionBuilder,
  CheckboxBuilder,
} = require('discord.js');
const { MODES, expected } = require('../services/componentProfile');

module.exports = (mode, state, userId) => {
  if (!MODES.includes(mode)) throw expected('Formulário inválido.');
  const modal = new ModalBuilder()
    .setCustomId(`components/submit_${mode}_${state.revision}_${userId}_${state._id}`)
    .setTitle('Preferências e documento');
  const has = name => mode === name || mode === 'all';
  if (has('upload')) {
    modal.addLabelComponents(new LabelBuilder()
      .setLabel('Documento PDF')
      .setDescription('Um PDF de até 2 MiB. Um novo envio substitui os metadados anteriores.')
      .setFileUploadComponent(new FileUploadBuilder()
        .setCustomId('documents').setMinValues(1).setMaxValues(1)
        .setRequired(true).setFileTypes('.pdf')));
  }
  if (has('radio')) {
    modal.addLabelComponents(new LabelBuilder().setLabel('Prioridade')
      .setRadioGroupComponent(new RadioGroupBuilder().setCustomId('priority')
        .setRequired(true).addOptions(
          new RadioGroupOptionBuilder().setLabel('Normal').setValue('normal')
            .setDescription('Atendimento na fila comum').setDefault(state.priority === 'normal'),
          new RadioGroupOptionBuilder().setLabel('Urgente').setValue('urgente')
            .setDescription('Solicitar análise prioritária').setDefault(state.priority === 'urgente'),
        )));
  }
  if (has('group')) {
    modal.addLabelComponents(new LabelBuilder().setLabel('Notificações')
      .setCheckboxGroupComponent(new CheckboxGroupBuilder().setCustomId('notifications')
        .setRequired(false).setMinValues(0).setMaxValues(2).addOptions(
          new CheckboxGroupOptionBuilder().setLabel('E-mail').setValue('email')
            .setDefault(state.notifications.includes('email')),
          new CheckboxGroupOptionBuilder().setLabel('Discord').setValue('discord')
            .setDefault(state.notifications.includes('discord')),
        )));
  }
  if (has('checkbox')) {
    modal.addLabelComponents(new LabelBuilder().setLabel('Permitir contato sobre esta solicitação')
      .setCheckboxComponent(new CheckboxBuilder().setCustomId('consent').setDefault(state.consent)));
  }
  return modal;
};
```

### src/messages/components/panel.js

```js
const { ActionRowBuilder, ButtonBuilder, ButtonStyle, FileBuilder, AttachmentBuilder,
  MessageFlags } = require('discord.js');
const { createContainer, emoji } = require('../../utils/componentsV2');
const { load } = require('../../services/componentProfile');

module.exports = async (client, interaction) => {
  const state = await load(client, interaction);
  const mark = name => emoji(client, name) || '';
  const summary = {
    revision: state.revision, priority: state.priority,
    notifications: state.notifications, consent: state.consent,
    documents: state.attachments.length,
  };
  const card = createContainer('info', [
    `## ${mark('configuracoes')} Preferências pessoais`,
    `${mark('usuario')} Prioridade: ${state.priority === 'urgente' ? 'Urgente' : 'Normal'}`,
    `Notificações: ${state.notifications.join(', ') || 'Nenhuma'}`,
    `Contato: ${state.consent ? 'Permitido' : 'Desativado'}`,
    `Documentos: ${state.attachments.length || 'Nenhum'} — apenas metadados, retenção de 30 dias`,
    'O arquivo abaixo contém somente o resumo destas preferências.',
  ].join('\n'));
  card.addFileComponents(new FileBuilder().setURL('attachment://preferencias.json').setSpoiler(false));
  const row = new ActionRowBuilder();
  for (const [mode, label, icon] of [
    ['upload', 'Enviar PDF', 'adicionar'], ['radio', 'Prioridade', 'editar'],
    ['group', 'Notificações', 'configuracoes'], ['checkbox', 'Contato', 'usuario'],
    ['all', 'Editar tudo', 'editar'],
  ]) {
    const button = new ButtonBuilder().setStyle(ButtonStyle.Primary).setLabel(label)
      .setCustomId(`components/open_${mode}_${state.revision}_${interaction.user.id}_${state._id}`);
    const resolved = mark(icon);
    if (resolved) button.setEmoji(resolved);
    row.addComponents(button);
  }
  card.addActionRowComponents(row); // O helper é dono do separador antes da primeira row.
  // Painel pessoal raiz: /componentes permite reabrir. Ao usar como filho de /central,
  // acrescente a row de Voltar com o customId real do pai e preserve panelOrigin.
  return {
    flags: MessageFlags.IsComponentsV2,
    allowedMentions: { parse: [] },
    components: [card],
    attachments: [], // Em edit, remove o resumo anterior antes de anexar o novo.
    files: [new AttachmentBuilder(Buffer.from(JSON.stringify(summary, null, 2)), {
      name: 'preferencias.json', description: 'Resumo das preferências salvas',
    })],
  };
};
```

### src/utils/componentProfileError.js

```js
const { MessageFlags } = require('discord.js');
const { payload } = require('./componentsV2');

module.exports = async (client, interaction, error, saved = false) => {
  if (!error.userFacing) {
    // Não registrar URL assinada, bytes ou mensagem de erro contendo URL/token.
    client.logger.error('Falha no formulário de componentes.', {
      guildId: interaction.guildId, userId: interaction.user.id,
      code: error.code || error.name, saved,
    });
  }
  const content = saved
    ? 'Dados salvos. Não foi possível atualizar a mensagem; abra /componentes para consultar.'
    : error.userFacing ? error.message : 'Não foi possível concluir. Tente novamente.';
  const data = payload('warning', `## Formulário\n${content}`, { flags: MessageFlags.Ephemeral });
  if (interaction.deferred || interaction.replied) {
    return interaction.followUp(data).catch(() => null);
  }
  return interaction.reply(data).catch(() => null);
};
```

### src/interactions/Button/components/open.js

```js
const service = require('../../../services/componentProfile');
const buildModal = require('../../../utils/componentProfileModal');
const onError = require('../../../utils/componentProfileError');

module.exports = async (client, interaction) => {
  try {
    const { mode, revision, recordId } = service.parse(interaction);
    const state = await service.load(client, interaction);
    if (state.revision !== revision || String(state._id) !== recordId) throw service.expected('Painel desatualizado. Abra /componentes.');
    // showModal é a resposta inicial: não deferir/reply antes dela.
    await interaction.showModal(buildModal(mode, state, interaction.user.id));
  } catch (error) {
    await onError(client, interaction, error);
  }
};
```

### src/interactions/ModalSubmit/components/submit.js

```js
const service = require('../../../services/componentProfile');
const onError = require('../../../utils/componentProfileError');

module.exports = async (client, interaction) => {
  let saved = false;
  try {
    const { mode, revision, recordId } = service.parse(interaction);
    if (!interaction.isFromMessage()) throw service.expected('Abra o formulário pelo painel.');
    // Revalidar aqui qualquer permissão administrativa adicionada ao Button.
    await interaction.deferUpdate(); // Confirma antes de rede/validação de arquivos/MongoDB.
    const changes = await service.readValues(interaction.fields, mode);
    await service.save(client, interaction, revision, changes, recordId);
    saved = true;
    const renderer = client.messages.get('components/panel');
    if (typeof renderer !== 'function') throw new Error('Renderer indisponível');
    // O token do ModalSubmit originado de mensagem edita exatamente aquela mensagem,
    // inclusive ephemeral. Não usar channel.messages.fetch para painéis ephemeral.
    await interaction.editReply(await renderer(client, interaction));
  } catch (error) {
    await onError(client, interaction, error, saved);
  }
};
```

### src/SlashCommands/general/componentes.js

```js
const { ApplicationCommandType, MessageFlags } = require('discord.js');
const onError = require('../../utils/componentProfileError');

module.exports = {
  name: 'componentes',
  description: 'Abre suas preferências e o formulário de demonstração.',
  type: ApplicationCommandType.ChatInput,
  run: async (client, interaction) => {
    try {
      if (!interaction.inGuild()) return interaction.reply({
        content: 'Use este comando em um servidor.', flags: MessageFlags.Ephemeral,
      });
      await interaction.deferReply({ flags: MessageFlags.Ephemeral });
      const renderer = client.messages.get('components/panel');
      if (typeof renderer !== 'function') throw new Error('Renderer indisponível');
      await interaction.editReply(await renderer(client, interaction));
    } catch (error) {
      await onError(client, interaction, error);
    }
  },
};
```

### Operação e reutilização

O roteador existente resolve `components/open_upload_0_<userId>_<recordId>` como `Button/components/open.js` e `components/submit_all_0_<userId>_<recordId>` como `ModalSubmit/components/submit.js`, por fallback do sufixo `_`. Os IDs dos campos são locais ao modal e não precisam de handlers próprios. Ajuste apenas a assinatura/export se o projeto exigir `(client, interaction, args)`; não adicione listeners.

Para usar só File, o renderer completo já envia AttachmentBuilder + FileBuilder, sem dependência de upload de usuário. Os quatro modos individuais usam exatamente os mesmos arquivos, com apenas um Label e leitura específica em `readValues`. No modo `all`, os quatro campos são validados antes de qualquer gravação; falha no PDF não grava parcialmente Radio/Checkbox. O update com filtro `revision` é atômico: duas submissões da mesma versão não se sobrescrevem. O `_id` do registro também faz parte do customId/filtro, impedindo que um modal antigo grave num registro recriado após TTL. Não há retry de gravação em caso de timeout incerto; reabra o painel para consultar o estado canônico. Se a gravação funcionar e o edit falhar, o usuário recebe aviso de dados salvos e pode reabrir; isso não é apresentado como rollback.

O fechamento do modal por cancelar/X não gera ModalSubmit nem evento de cancelamento. O painel anterior permanece válido; não se deve prometer refresh em evento inexistente. Para origens públicas ou navegação com múltiplas mensagens, reutilize `panelOrigin.js` conforme `operational-patterns.md`. Para gravações em configuração existente, use o esquema e `reloadConfigs` da base; este exemplo é registro pessoal de domínio, sem cache duplicado. Na expiração, o usuário reabre /componentes; um modal de registro expirado não recria o documento no submit.

Logs/transcripts/exportações operacionais com anexos continuam sendo `{ embeds, files }` enviados diretamente, sem V2/decoradores. O File deste exemplo é conteúdo do painel pessoal, não um log. Não passe o payload legado de auditoria por este renderer ou por `payload`.
