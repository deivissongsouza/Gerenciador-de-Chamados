# Specs

Cada funcionalidade é especificada antes de ser implementada (P13). Este arquivo explica o processo e mostra o status de todas as specs.

Documentos de referência: [visão do produto](../docs/visao-do-produto.md) · [constituição](../docs/constituicao.md) · [stack](../docs/stack.md) · [glossário](../docs/glossario.md) · [modelo de dados](../docs/modelo-de-dados.md) · [convenções da API](../docs/convencoes-api.md) · [permissões](../docs/permissoes.md)

## Índice — Fase 1

| Spec | Módulo | Status |
|---|---|---|
| [000](000-fundacao/) | Fundação (ambiente, qualidade, API transversal, harness) | Design aprovado |
| [001](001-autenticacao/) | Autenticação | Design aprovado |
| 002 | Controle de acesso | Não iniciada |
| 003 | Usuários | Não iniciada |
| 004 | Estrutura organizacional | Não iniciada |
| 005 | Catálogo | Não iniciada |
| 006 | Chamados | Não iniciada |
| 007 | Interações | Não iniciada |
| 008 | Anexos | Não iniciada |
| 009 | SLA | Não iniciada |
| 010 | Avaliação | Não iniciada |
| 011 | Notificações por e-mail | Não iniciada |
| 012 | Dashboard | Não iniciada |
| 013 | Configurações | Não iniciada |

## Estrutura

Cada spec tem uma pasta `NNN-nome-curto/` com três arquivos, criados a partir de [`_modelos/`](_modelos/):

| Arquivo | Responde | Conteúdo |
|---|---|---|
| `requisitos.md` | **O quê** e **por quê** | Contexto, histórias de usuário, regras de negócio, critérios de aceite, fora de escopo, dúvidas |
| `design.md` | **Como** | Endpoints, modelo de dados, eventos e filas, decisões, princípios afetados |
| `tarefas.md` | **Em que ordem** | Tarefas pequenas, cada uma ligada aos critérios que cobre |

## Ciclo de vida

```
Não iniciada → Rascunho → Requisitos aprovados → Design aprovado → Tarefas aprovadas → Em implementação → Concluída
```

Cada etapa só começa depois que a anterior é aprovada pelo responsável pelo produto. O status fica no cabeçalho de cada arquivo e neste índice.

## Identificadores

| Tipo | Formato | Exemplo |
|---|---|---|
| Regra de negócio | `RN-NNN-XX` | RN-006-02 |
| Critério de aceite | `CA-NNN-XX` | CA-006-03 |
| Tarefa | `T-NNN-XX` | T-006-05 |

`NNN` é o número da spec; `XX` é sequencial dentro dela. Identificadores nunca são reaproveitados: um item removido fica marcado como removido.

## Rastreabilidade

```
CA-006-03  →  T-006-05  →  teste Pest "CA-006-03 …"  →  commit "feat(tickets): … [T-006-05]"
```

- Todo critério de aceite é coberto por pelo menos uma tarefa e um teste (P14).
- O nome do teste começa com o identificador do critério.
- O commit cita o identificador da tarefa.

## Regras do processo

1. O Claude Code **propõe**; o responsável pelo produto **aprova**. Nenhuma etapa avança sem aprovação.
2. Uma tarefa por vez. Ao concluir, ela é marcada como feita em `tarefas.md`.
3. **Uma tarefa = um commit**, no padrão Conventional Commits, com o identificador da tarefa.
4. O commit é **bloqueado** se testes, Pint ou Larastan falharem.
5. Spec aprovada que precise mudar é atualizada **antes** do código, com registro na seção "Histórico de alterações" do arquivo.
6. Termos seguem o [glossário](../docs/glossario.md). Termo novo entra no glossário antes de ser usado na spec.
