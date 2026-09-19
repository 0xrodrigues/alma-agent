# alma-agent

Código do agente **ALMA — Applied Learning & Memory Assistant**: agente de conhecimento
institucional que aprende sobre os processos, produtos e serviços de um negócio e ajuda no
planejamento, desenvolvimento e documentação contínua.

Este repo guarda o **agente** (skill, config). O **conhecimento** em si (o que o ALMA aprendeu)
mora em outro repo — hoje [`alma-kb`](https://github.com/0xrodrigues/alma-kb). Separação
proposital: conhecimento muda toda hora sem precisar de review de engenharia, código do agente
precisa de versionamento e review normal.

## O que o ALMA faz hoje (v1)

Roda como **skill do Claude Code** — comando `/alma <ação> <alvo>` dentro de qualquer sessão
Claude Code, em qualquer repo aberto.

| Ação | O que faz |
|---|---|
| `/alma document <caminho/descrição>` | Lê um componente do repo atual (endpoint, scheduled job, consumer), entende o que faz (técnica e negócio), escreve/atualiza entrada em `alma-kb` e deixa um ponteiro curto (`docs/ALMA.md`) no repo do serviço. |
| `/alma ask <pergunta>` | Responde sobre processo/produto do negócio, buscando primeiro na `alma-kb` e conferindo (drift-check) contra o código atual do repo aberto. |
| `/alma refine <história/solução>` | Cruza uma história ou solução rascunhada com a KB e o código atual, devolve critérios de aceite, perguntas em aberto e riscos. |

Regras de comportamento fixas na skill: nunca inventa regra de negócio ausente da KB/código
(sinaliza "não encontrado, confirmar com o time" em vez de arriscar), sempre cita a fonte, e
achado suspeito no código entra como "achado, não confirmado" — nunca como afirmação de bug.

Publicação automática de PR (GitHub) e página no Confluence ainda **não está integrada** — a
skill para nesse ponto e avisa explicitamente.

## Estrutura deste repo

```
alma/
├── SKILL.md      # comportamento do agente ("soul") — o que cada ação faz, regras gerais
└── config.json   # kb_path, kb_remote, sync_enabled (ver Configuração abaixo)
```

Pasta se chama `alma/` (não `skill/`) de propósito — Claude Code exige que o nome da pasta
bata com o `name:` do frontmatter do `SKILL.md`.

`SKILL.md` e `config.json` aqui são a **fonte de verdade**, versionada. O Claude Code lê a skill
de `~/.claude/skills/alma/` — os arquivos lá são **symlinks** pra esses dois arquivos deste repo,
então editar aqui já reflete em qualquer sessão, sem precisar reinstalar nada.

## Configuração inicial

1. Clone os dois repos: este (`alma-agent`) e o de conhecimento (`alma-kb`).
2. Crie a pasta da skill e symlinke os dois arquivos:

   ```bash
   mkdir -p ~/.claude/skills/alma
   ln -s /caminho/para/alma-agent/alma/SKILL.md    ~/.claude/skills/alma/SKILL.md
   ln -s /caminho/para/alma-agent/alma/config.json ~/.claude/skills/alma/config.json
   ```

3. Edite `alma/config.json` com os caminhos/URLs do seu ambiente:

   ```json
   {
     "kb_path": "/caminho/local/para/alma-kb",
     "kb_remote": "https://github.com/<org>/alma-kb.git",
     "sync_enabled": false
   }
   ```

   - `kb_path` — pasta local onde o `alma-kb` está clonado (ou onde vai ficar, modo local puro).
   - `kb_remote` — URL do repo remoto de conhecimento. Pode deixar preenchido mesmo sem usar
     ainda (fica guardado pra quando `sync_enabled` virar `true`).
   - `sync_enabled` — `false`: ALMA opera 100% local, nunca faz `git pull`/`push` na KB (só
     commita local). `true`: antes de cada ação a skill sincroniza (`pull`, ou `clone` se
     `kb_path` ainda não existir) e, depois de escrever, faz commit + push pro remoto — assim o
     time todo vê a atualização, não só quem rodou o comando.

4. Se o diretório da KB ficar fora do repo em que você normalmente trabalha, adicione-o em
   `permissions.additionalDirectories` no `~/.claude/settings.json` global (senão o Claude Code
   bloqueia leitura/escrita fora do working directory da sessão):

   ```json
   {
     "permissions": {
       "additionalDirectories": ["/caminho/local/para/alma-kb", "/caminho/local/para/alma-agent"]
     }
   }
   ```

5. Pronto — abra o Claude Code em qualquer repo e use `/alma ask`, `/alma document` ou
   `/alma refine`.

## Contexto do projeto

Ver [`CONTEXT.md`](./CONTEXT.md) — histórico completo de decisões de arquitetura, trade-offs
discutidos, estado atual e próximos passos. Útil pra retomar o projeto em outra conversa sem
perder contexto.
