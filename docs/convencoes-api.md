# Convenções da API

> Regras que todos os endpoints seguem. Quem aprende a usar um endpoint sabe usar todos.
> Princípios em [constituicao.md](constituicao.md), termos em [glossario.md](glossario.md). Última revisão: 2026-10-11.

## 1. Endereços e recursos

| Item | Regra | Exemplo |
|---|---|---|
| Base e versão | Prefixo com versão. Mudança incompatível gera nova versão | `/api/v1/...` |
| Recursos | Substantivos no plural, em `kebab-case` | `/tickets`, `/business-calendars` |
| Sub-recursos | No máximo um nível de aninhamento | `/tickets/{id}/comments` |
| Identificadores | ULID na URL. O número amigável do chamado é buscável por filtro | `/tickets/01HZX...`, `?filter[number]=1542` |

## 2. Métodos

| Método | Uso | Resposta de sucesso |
|---|---|---|
| `GET` | Ler e listar | `200` |
| `POST` | Criar | `201` + cabeçalho `Location` |
| `PATCH` | Atualizar parcialmente | `200` |
| `DELETE` | Arquivar (soft delete) | `204` |

`PUT` não é usado.

### Ações de negócio

Ações que mudam estado ou têm regra própria usam endpoints dedicados, sempre com `POST`:

```
POST /api/v1/tickets/{id}/assign
POST /api/v1/tickets/{id}/resolve
POST /api/v1/tickets/{id}/restore
POST /api/v1/sla-policies/{id}/reset
```

Campos controlados por ações (ex.: `status`, `assignee_id`) **não** são aceitos no `PATCH`.

## 3. Formato dos dados

| Item | Regra |
|---|---|
| Envelope | `data` com o recurso; `meta` e `links` em listagens paginadas |
| Nomes dos campos | `snake_case` |
| Datas | ISO 8601 em UTC (`2026-10-11T14:30:00Z`) |
| Valores fixos | Sempre o código (`"status": "in_progress"`), nunca o texto traduzido |
| Relacionamentos | Retornados apenas quando solicitados via `include` |

```json
{
  "data": {
    "id": "01HZX3K8...",
    "number": 1542,
    "title": "Sistema de recepção fora do ar",
    "status": "in_progress",
    "priority": "urgent",
    "created_at": "2026-10-11T14:30:00Z"
  }
}
```

## 4. Listagens

| Item | Regra |
|---|---|
| Filtros | `?filter[status]=open&filter[priority]=high` |
| Ordenação | `?sort=-created_at` (`-` = decrescente) |
| Relacionamentos | `?include=requester,team` |
| Paginação | `?page[number]=2&page[size]=25`. Padrão 25, máximo 100 (P4) |
| Arquivados | Excluídos por padrão. `?filter[trashed]=with` ou `only` para incluí-los |

Apenas filtros, ordenações e relacionamentos declarados no endpoint são aceitos; qualquer outro gera erro `400`. Implementação: spatie/laravel-query-builder. As regras de visibilidade são aplicadas antes do query builder.

## 5. Códigos de resposta

| Código | Quando |
|---|---|
| `200`, `201`, `204` | Sucesso |
| `400` | Parâmetro de consulta não permitido (filtro, ordenação ou include) |
| `401` | Não autenticado |
| `403` | Autenticado, mas sem permissão para a ação |
| `404` | Recurso inexistente **ou que o usuário não pode ver** |
| `409` | Conflito de estado (ex.: resolver um chamado cancelado) |
| `422` | Erro de validação |
| `429` | Limite de requisições excedido |

Um recurso que o usuário não pode ver responde `404`, nunca `403`, para não revelar que ele existe (P12).

## 6. Erros

Formato **RFC 9457 (Problem Details)**, com `Content-Type: application/problem+json`:

```json
{
  "type": "https://<host>/errors/validation",
  "title": "Dados inválidos",
  "status": 422,
  "detail": "Um ou mais campos são inválidos.",
  "errors": {
    "priority": ["O valor informado não é uma prioridade válida."]
  }
}
```

- `type` identifica o tipo de erro e é estável; clientes podem tratá-lo programaticamente.
- `errors` aparece apenas em erros de validação.
- Nenhum erro expõe detalhes internos (stack trace, SQL) fora do ambiente de desenvolvimento (P12).

## 7. Autenticação

| Modo | Funcionamento |
|---|---|
| Sessão | Cookie HttpOnly com proteção CSRF (fluxo do Sanctum) |
| Chave de API | `Authorization: Bearer <chave>` |

O acesso efetivo de uma chave de API é a **interseção** entre os escopos da chave e as permissões do usuário dono dela. Uma chave nunca tem mais poder que seu usuário.

## 8. Idioma

- Mensagens (erros, validação, e-mails) usam o sistema de tradução do Laravel (`lang/`).
- Um middleware define o idioma a partir do cabeçalho `Accept-Language`, dentro da lista de idiomas suportados.
- **Fase 1:** `pt_BR` como padrão e único idioma suportado. Novos idiomas exigem apenas arquivos de tradução.
- Códigos (`in_progress`, `urgent`) e nomes de campos nunca são traduzidos.

## 9. Robustez

| Item | Regra |
|---|---|
| Limite de requisições | Por usuário ou chave de API, com cabeçalhos informando limite e saldo |
| Idempotência | `POST /api/v1/tickets` aceita o cabeçalho `Idempotency-Key`. Uma requisição repetida com a mesma chave devolve o resultado da primeira, sem criar outro chamado |
| Documentação | OpenAPI gerado pelo Scramble, publicado em `/docs/api` |

## 10. Decisões

| Decisão | Alternativas consideradas | Motivo |
|---|---|---|
| **Ações de negócio em endpoints próprios** (`POST /tickets/{id}/resolve`) | Alterar campos de estado via `PATCH /tickets/{id}` com `{"status": "resolved"}` | Cada ação tem permissão própria (P7), validação própria (ex.: só resolve o que está em atendimento), registro de auditoria claro (P11) e efeitos colaterais explícitos (calcular SLA, notificar). Com `PATCH`, uma única rota concentraria todas essas regras, o fluxo de status ficaria exposto a transições inválidas e as permissões ficariam presas a campos, não a ações. Também deixa a API autoexplicativa para quem constrói agentes e integrações |
| Campos em `snake_case` | `camelCase` | Padrão do Laravel e igual ao banco, sem camada de conversão (P15) |
| Erros em RFC 9457 | Formato padrão do Laravel (`message` + `errors`) | Padrão oficial, com tipo de erro estável para tratamento programático; mais adequado a uma API pública |
| `Idempotency-Key` na criação de chamados | Sem idempotência | Agentes e integrações reenviam requisições após falhas de rede; sem a chave, cada reenvio criaria um chamado duplicado |
| `404` para recurso sem permissão de leitura | `403` | `403` confirmaria a existência do recurso (ex.: um chamado privado) |
| Paginação por página | Paginação por cursor | Mais simples de usar e permite total de páginas; o volume da fase 1 não exige cursor |
| Sem `PUT` | `PUT` para substituição completa | Substituir o recurso inteiro raramente é a intenção; `PATCH` cobre os casos reais com menos risco |
