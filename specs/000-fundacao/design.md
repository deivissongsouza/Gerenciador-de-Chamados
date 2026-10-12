# 000 — Fundação: Design

| Status | Data | Aprovado por |
|---|---|---|
| Não iniciado | — | — |

Requisitos: [requisitos.md](requisitos.md)

## Visão geral

A solução em poucas linhas.

## Modelo de dados

Tabelas e colunas novas ou alteradas, tipos, índices, relacionamentos e soft delete. Referencie o modelo de dados geral em vez de repeti-lo.

## Endpoints

### `MÉTODO /api/v1/recurso`

| Item | Valor |
|---|---|
| Permissão | `recurso.acao` |
| Autenticação | Sessão, chave de API ou ambas |
| Critérios cobertos | CA-000-01, CA-000-02 |

**Entrada:** campos, tipos e validações.

**Saída:** formato da resposta de sucesso.

**Erros:** códigos HTTP e quando ocorrem.

> Filtros, ordenação e paginação de listagens seguem as convenções da API.

## Eventos, filas e tarefas agendadas

O que é disparado, o que roda em fila e o que é agendado.

## Auditoria

O que é registrado no log de auditoria (P11).

## Decisões

| Decisão | Alternativas consideradas | Motivo |
|---|---|---|

## Princípios da constituição

Princípios mais relevantes para este módulo e como o design os atende. Exceções exigem justificativa explícita.

## Histórico de alterações

| Data | Alteração | Motivo |
|---|---|---|
