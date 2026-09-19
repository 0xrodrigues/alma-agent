# ALMA — Contexto do Projeto

> Documento de contexto acumulado. Cole isso em qualquer novo chat pra retomar o projeto sem
> reexplicar tudo. Atualizado em 2026-09-18.

## O que é o ALMA

**ALMA — Applied Learning & Memory Assistant.** Agente de conhecimento institucional. Objetivos,
em ordem de maturidade:

1. Criar e manter base de conhecimento/documentação sozinho, sem trabalho manual do usuário.
2. Dar contexto em tempo de planejamento/desenvolvimento, direto no Claude Code.
3. Responder perguntas sobre processo/produto do negócio.
4. Ajudar a refinar histórias e soluções.
5. **(futuro)** Participar de reuniões, opinar, propor soluções.

Primeiro caso de uso concreto: documentar serviços existentes no GitHub. ALMA lê um
endpoint/scheduler/consumer, entende o que faz (técnica e negócio), documenta, e mantém
atualizado quando o fluxo muda.

## Decisões de arquitetura (nessa ordem, cada uma revisando/refinando a anterior)

### 1. Interface — como o ALMA é chamado

- **v1 (hoje): skill do Claude Code**, comando `/alma <ação> <alvo>`. Roda dentro de qualquer
  sessão Claude Code, em qualquer repo aberto.
- Motivo: reusa sessão, tools e auths já ativos (GitHub, Atlassian/Confluence plugins), zero
  infra nova, ship rápido.
- **Futuro: CLI própria** (`alma`, tipo "open alma"), via Claude Agent SDK — mesmo SDK que roda
  o Claude Code. Faz sentido quando ALMA precisar existir fora do Claude Code (reunião, rodar
  headless). Não é caminho fechado: mesmo system prompt/KB pode alimentar os dois.
- Ações da skill: `document <caminho/descrição>`, `ask <pergunta>`, `refine <história>`.

### 2. Onde mora o conhecimento — teve uma correção importante no meio do caminho

Primeira tentativa: KB local (`kb/` numa pasta do meu disco). **Errado** — conhecimento precisa
ser institucional (acessível pro time todo, não só minha máquina).

Modelo final, **3 lugares, dono claro cada um**:

1. **`alma-kb`** (repo git, remoto, compartilhado) — **fonte única da verdade**. Não é doc
   "bonita" pra humano ler, é memória de trabalho do ALMA: entradas curtas, com frontmatter,
   cross-linkadas entre conceitos de produto que atravessam serviços.
2. **Repo do serviço** — recebe só um **ponteiro curto** (`docs/ALMA.md`) linkando pra entrada
   canônica em `alma-kb`. Não recebe cópia completa da doc.
3. **Confluence** — espelho **gerado** a partir do `alma-kb`, não fonte paralela mantida na mão.

Motivo da revisão pra ponteiro-em-vez-de-cópia: duplicar conteúdo = 2+ lugares pra sincronizar
toda vez que um fluxo muda — exatamente o problema que o usuário queria evitar ao centralizar.

### 3. Separação de repositórios

- **`alma-kb`** — conhecimento. Muda toda hora (ALMA documentando, time perguntando coisa nova),
  sem precisar de CI/code review.
- **`alma`** (nome do repo do agente, a definir) — código do agente: skill hoje, futuramente MCP
  server / app Agent SDK. Precisa versionamento, teste, PR revisado por dev.
- Motivo: ritmos de mudança diferentes, histórico de commit limpo, controle de acesso
  configurável separado.
- **Split já feito localmente**: `/Users/batman/workspace/alma-kb` (conhecimento) e
  `/Users/batman/workspace/alma-agent` (código do agente, skill/CONTEXT.md). Falta só virar
  remoto no GitHub da empresa quando org/repo forem definidos (ainda não criado, de propósito —
  só rascunho local por enquanto).

### 4. Roteamento rápido (agente ↔ KB)

- `alma-kb/INDEX.md` — índice mestre, uma linha por entrada (`- [Título](caminho) — resumo.
  tags: a, b, c`), organizado por seção (`services`, `processes`, `decisions`, `glossary`).
- ALMA lê o índice **primeiro**, só abre arquivo completo se bater — mesmo padrão que o Claude
  usa pra própria memória entre sessões. Evita carregar a KB inteira em contexto toda vez.
- Hoje: skill aponta direto pra pasta local. Quando `alma-kb` virar repo remoto de verdade, troca
  por `git pull` antes de cada uso (incremental, rápido).
- Busca semântica/embedding: **não construir agora** (YAGNI). Só vale quando a KB crescer pra
  milhares de docs.

### 5. Estrutura de cada doc na KB

```
kb/
├── services/    # doc por serviço/componente (endpoint, scheduler, consumer)
├── processes/   # regra de negócio cross-serviço (ex: "parcelado loja" ⟷ "parcelado cliente")
├── decisions/   # histórias/soluções refinadas que valeu guardar
└── glossary/    # termos do domínio
```

Frontmatter padrão (`_templates/*.md`): `title, type, tags, services, confluence, source_repo,
owner, created, updated`.

### 6. Fluxo de `/alma document`

1. Localiza componente no repo atual, classifica tipo (endpoint/scheduler/consumer), traça
   dependências.
2. Checa `INDEX.md` — se já existe entrada, **atualiza**, não duplica.
3. Escreve/atualiza entrada completa em `alma-kb/kb/services/`.
4. Cria/atualiza só o ponteiro em `<repo-do-serviço>/docs/ALMA.md`.
5. Publicação real de PR (GitHub) e Confluence: **ainda não automatizada** — ALMA para nesse
   ponto e avisa explicitamente, não simula que foi feito.
6. Se o componente tocar processo de negócio maior, cria/atualiza entrada em `kb/processes/`
   linkando os dois.

Atualização quando o fluxo muda: v1 é manual — rodar `/alma document` de novo no componente.

### 7. Persona / regras de comportamento (validadas no primeiro teste real)

- Nunca inventa regra de negócio ausente da KB/código — diz "não encontrado, confirmar com o
  time" em vez de arriscar.
- Achado suspeito no código (ex: bug em potencial) entra como **"achado, não confirmado"**, não
  como afirmação de bug.
- Sempre cita a fonte (arquivo da KB, link Confluence se existir).
- Ao responder `ask`, faz **drift-check**: compara o que a KB diz com o código atual antes de
  responder.

## Infra / permissões (resolvido)

- `alma-kb`/`alma-agent` locais e o skill (`~/.claude/skills/alma/`) ficavam fora do working
  directory de qualquer outro repo aberto → bloqueado por
  `permissions.blockReadsOutsideWorkingDirectories`.
- Fix: `/Users/batman/workspace/alma-kb` e `/Users/batman/workspace/alma-agent` adicionados em
  `permissions.additionalDirectories` no `~/.claude/settings.json` global — liberado
  permanentemente, em qualquer repo, sem precisar `/add-dir` de novo.
- Auto Mode: não é "aprova tudo" — classifica ação por ação. Ações irreversíveis/visíveis
  (git push, abrir PR, publicar Confluence) sempre pedem confirmação, mesmo com auto mode
  ligado. Isso é esperado, não bug.

## O que já existe (estado atual, concreto)

- `/Users/batman/workspace/alma-kb/`: `README.md`, `INDEX.md`,
  `kb/{services,processes,decisions,glossary}/`, `_templates/{service,process,decision,glossary}.md`.
- `/Users/batman/workspace/alma-agent/`: hoje só este `CONTEXT.md` — futuro lar do código do
  agente (skill/MCP server/Agent SDK app).
- `~/.claude/skills/alma/SKILL.md` — o "soul", implementa `document`/`ask`/`refine`.
- `~/.claude/skills/alma/config.json` — `kb_path` aponta pra `alma-kb`, `kb_remote` (null,
  pendente).
- Primeiro teste real: documentado `transfer-funds-endpoint` do serviço `voidbank`. Entrada
  criada em `kb/services/transfer-funds-endpoint.md`, índice atualizado, ponteiro criado em
  `voidbank/docs/ALMA.md`. Achados sinalizados (não confirmados): `ValidateAmount` sem
  `@Component` (validação de amount > 0 não roda), falha de validação engolida sem interromper
  fluxo, tópico Kafka `TRANSACTION_FAILED_VALIDATION` sem consumidor.
- Duas perguntas `/alma ask` sobre o mesmo fluxo respondidas corretamente, reusando contexto
  dentro da mesma sessão, com drift-check contra o código atual.

## Fase futura — generalização pra qualquer negócio

Ideia levantada em 2026-09-18: ALMA nasceu pra Cielo, mas o núcleo (KB com `INDEX.md`,
frontmatter, persona "cita fonte, não inventa") é agnóstico de negócio — generaliza fácil. O que
é Cielo/software-específico é a ação `document` (lê endpoint/scheduler/consumer, publica em
GitHub+Confluence) — não se aplica a negócio não-técnico.

Direção proposta (**não iniciar antes do piloto Cielo terminar**): um **onboarding/setup
inicial** que pergunta tipo de negócio e ferramentas usadas (GitHub? Notion? Confluence? Slack?),
e a partir disso gera:

- a taxonomia do `kb/` (pode não ser `services/processes/decisions/glossary` pra toda empresa);
- quais ações da skill fazem sentido ativar (`document` só cabe se o negócio tem código/serviços
  pra documentar).

Trade-off já discutido: isso muda o projeto de "ferramenta interna pra Cielo" pra "template
reusável" — mais design upfront, mas evita reescrever tudo se a intenção for produtizar depois.

## Pendente / próximos passos

- [ ] Definir org + nomes exatos dos repos no GitHub da empresa (`alma-kb`, `alma-agent`) —
      split local já feito, falta só o remoto.
- [ ] Automatizar publicação real: abrir PR no repo do serviço (via plugin GitHub) e publicar no
      Confluence (via plugin Atlassian).
- [ ] Confluence: espaço/hierarquia já existe na empresa, mas ainda não foi passado pro ALMA.
- [ ] Validar template final de doc (seções) com o time.
- [ ] Segundo componente piloto — idealmente um com dependência cross-serviço (ex: exemplo
      original do usuário, "parcelado loja" ⟷ "parcelado cliente"), pra validar de verdade a
      parte de `kb/processes/` e cross-link, que ainda não foi exercitada.
- [ ] Considerar rodar skill `fewer-permission-prompts` pra reduzir prompt em comandos
      read-only repetidos.
- [ ] Fase futura: CLI própria via Claude Agent SDK, pra ALMA existir fora do Claude Code.
