# NNN — Nome do módulo: Tarefas

| Status | Data | Aprovado por |
|---|---|---|
| Rascunho | AAAA-MM-DD | — |

Design: [design.md](design.md)

## Regras

- Cada tarefa é pequena, testável e vira **um commit**.
- O teste do critério de aceite é escrito antes da implementação.
- Uma tarefa só é marcada como feita com testes, Pint e Larastan passando.
- A ordem respeita as dependências: uma tarefa não depende de outra que vem depois dela.

## Tarefas

- [ ] **T-NNN-01** — [descrição] _(cobre CA-NNN-01)_
- [ ] **T-NNN-02** — [descrição] _(cobre CA-NNN-02, CA-NNN-03)_

## Cobertura

| Critério | Tarefas |
|---|---|
| CA-NNN-01 | T-NNN-01 |

> Todo critério de aceite da spec precisa aparecer nesta tabela.

## Histórico de alterações

| Data | Alteração | Motivo |
|---|---|---|
