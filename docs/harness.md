# Harness do Claude Code

> Definição da configuração que faz o Claude Code seguir o processo de forma consistente.
> **Nenhuma peça foi criada ainda.** A criação acontece junto com o esqueleto do repositório, logo antes da primeira tarefa de código.
> Última revisão: 2026-10-12.

## Princípio

O que não está no repositório não existe para o Claude Code. Toda decisão vira documento; o harness garante que o processo descrito em [specs/README.md](../specs/README.md) seja aplicado, não apenas descrito.

## Ambiente

- O Claude Code roda **dentro do WSL** (Ubuntu 24.04), no diretório do projeto.
- Comandos do projeto são executados pelo **Makefile**, que delega aos containers Docker.

## Peças

### CLAUDE.md (raiz)
Curto. Contém:
- Visão do produto em poucas linhas.
- Onde estão os documentos (`docs/`, `specs/`) — **aponta** para eles em vez de copiá-los, para não ocupar contexto.
- Comandos do Makefile.
- Definição de pronto: testes, Pint e Larastan passando.
- Resumo do processo SDD e da convenção de idioma.

### Comandos do processo (`.claude/skills/`)
Cada etapa do SDD é um comando:

| Comando | Função |
|---|---|
| `/spec-nova` | Cria a pasta da spec a partir de `specs/_modelos/` e registra no índice |
| `/spec-requisitos` | Conduz a escrita de `requisitos.md` |
| `/spec-design` | Conduz a escrita de `design.md` (exige requisitos aprovados) |
| `/spec-tarefas` | Conduz a escrita de `tarefas.md` (exige design aprovado) |
| `/tarefa-proxima` | Implementa a próxima tarefa pendente (fluxo abaixo) |
| `/spec-status` | Mostra o status das specs e a próxima ação |

Fluxo de `/tarefa-proxima`:
1. Lê a spec e a próxima tarefa pendente.
2. Escreve o teste do critério de aceite (falhando).
3. Implementa até o teste passar.
4. Roda `make check` (testes, Pint, Larastan).
5. Aciona o revisor de código.
6. Marca a tarefa como feita e propõe o commit para aprovação.

### Subagentes (`.claude/agents/`)

| Subagente | Função |
|---|---|
| Revisor de spec | Confere a spec contra a constituição, o glossário e os contratos transversais |
| Revisor de código | Confere a implementação contra a spec e a constituição antes do commit |

Revisam com contexto limpo, sem o viés de quem escreveu.

### Permissões (`.claude/settings.json`)

| Nível | Exemplos |
|---|---|
| Liberado | `make test`, `make lint`, `make check`, `git status`, `git diff`, leitura de arquivos |
| Pergunta antes | `git commit`, edição de `docs/constituicao.md` |
| Proibido | Ler `.env`, `git push`, remover volumes Docker, `migrate:fresh` |

### Hooks (`.claude/settings.json`)

| Momento | Ação |
|---|---|
| Após editar arquivo PHP | Formata com Pint |
| Antes de `git commit` | Roda testes, Pint e Larastan; **bloqueia o commit** se algo falhar |

### MCP
- **Laravel Boost**, executando no container `app`: documentação da versão exata do Laravel e dos pacotes, schema do banco, logs e Artisan.

### Git hooks
Os mesmos checks no commit para contribuidores humanos, independentes do Claude Code.

## Repositório

- **Principal: GitHub.** Pull requests, issues e revisão acontecem lá.
- **Espelho: GitLab**, sincronizado a partir do GitHub, com o GitLab CI executando os mesmos comandos do Makefile.
