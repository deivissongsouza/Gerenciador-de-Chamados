# 001 — Autenticação: Design

| Status | Data | Aprovado por |
|---|---|---|
| Design aprovado | 2026-10-12 | Dêvisson Souza |

Requisitos: [requisitos.md](requisitos.md)

## Visão geral

- **Pessoas** se autenticam por **sessão** (cookie HttpOnly + CSRF), no fluxo SPA do **Laravel Sanctum**. Login, logout e recuperação de senha usam o **Laravel Fortify** em modo headless (sem telas), com rotas sob `/api/v1/auth`.
- **Integrações** se autenticam por **chave de API** (tokens do Sanctum). A chave pertence a um usuário do tipo `integration`; o acesso é decidido pelas permissões desse usuário (spatie/laravel-permission), sem escopos.
- Um **resolvedor de canal** define o canal de origem a partir de quem está autenticado. Ele é usado pela auditoria agora e pela criação de chamados na spec 006.
- O gerenciamento de contas de integração e de chaves fica em `/api/v1/integrations` e só aceita sessão.

```
Pessoa ──cookie──► guard sanctum ──► sessão (tabela sessions) ──► User type=person
Integração ─Bearer─► guard sanctum ──► personal_access_tokens ──► User type=integration
                                                     │
                       permissões (spatie) ◄─────────┘   canal: web | conta.channel
```

## Dependências

| Depende de | O que é usado | Como fica nesta spec |
|---|---|---|
| [Spec 000](../000-fundacao/requisitos.md) (Fundação) | Docker, Makefile, CI, handler de erros RFC 9457, Pest, Pint, Larastan, activity log, serviço `Settings` | Pré-requisito: concluída antes da primeira tarefa desta spec |
| Spec 004 (Estrutura) | Tabelas `organizations` e `departments` | Esta spec cria as migrations **mínimas** (colunas do [modelo de dados](../../docs/modelo-de-dados.md)), sem endpoints. A spec 004 acrescenta regras e endpoints |
| Spec 002 (Controle de acesso) | Tabelas do spatie, papéis de fábrica | Esta spec instala o pacote, cria a permissão `integrations.manage` e o papel `Admin` no seeder de fábrica. A spec 002 completa os demais papéis e permissões |

## Modelo de dados

### `users` (criada aqui)

| Coluna | Tipo | Regra |
|---|---|---|
| `id` | ulid | PK |
| `type` | string(20) | `person` ou `integration` (enum `App\Enums\UserType`) |
| `organization_id` | ulid FK | Obrigatória |
| `department_id` | ulid FK, nula | Só pessoas |
| `name` | string(150) | Obrigatório |
| `email` | string(255), nula | Só pessoas. Gravado em minúsculas; índice único |
| `password` | string, nula | Só pessoas. Hash bcrypt |
| `remember_token` | string(100), nula | "Manter conectado" |
| `channel` | string(20), nula | Só integrações: `api`, `agent` ou `monitoring` (enum `App\Enums\Channel`) |
| `phone` | string(30), nula | Opcional |
| `created_at`, `updated_at`, `deleted_at` | timestamp | Soft delete = desativado (P10) |

**Restrição no banco** (`CHECK`), para que nenhum caminho do código crie uma conta incoerente (P6):
- `person` → `email` e `password` preenchidos, `channel` nulo;
- `integration` → `email` e `password` nulos, `channel` preenchido e diferente de `web`.

### Tabelas de pacotes (ajustadas para ULID)

| Tabela | Origem | Ajuste |
|---|---|---|
| `personal_access_tokens` | Sanctum | `ulidMorphs('tokenable')`. `abilities` sempre `["*"]`. Usa `expires_at` e `last_used_at` nativos |
| `sessions` | Laravel | `user_id` como `foreignUlid`, indexado |
| `password_reset_tokens` | Laravel | Sem ajuste. Expiração de 60 min (`auth.passwords.users.expire`) |
| `activity_log` | spatie/laravel-activitylog | `nullableUlidMorphs` para `subject` e `causer` |
| tabelas do spatie/laravel-permission | spatie | `model_morph_key` como ULID |

### Configurações (RN-001-04)

| Chave | Padrão de fábrica |
|---|---|
| `auth.session_idle_minutes` | 60 |
| `auth.remember_days` | 7 |

Os padrões ficam em `config/` e são lidos por um serviço `Settings` que, quando houver personalização em `settings`, a usa no lugar do padrão. O gerenciamento dessas personalizações pela API é a decisão em aberto D1.

## Endpoints

Todos sob `/api/v1`. Erros no formato RFC 9457 ([convenções da API](../../docs/convencoes-api.md)).

### Resumo

| Método e rota | Autenticação | Permissão | Critérios |
|---|---|---|---|
| `GET /auth/csrf-cookie` | Nenhuma | — | CA-001-01 |
| `POST /auth/login` | Nenhuma | — | CA-001-01 a 04, 07 |
| `POST /auth/logout` | Sessão | — | CA-001-08 |
| `POST /auth/forgot-password` | Nenhuma | — | CA-001-10, 11 |
| `POST /auth/reset-password` | Nenhuma | — | CA-001-12 a 14 |
| `POST /me/password` | Sessão | — | CA-001-14 a 16 |
| `GET /me` | Sessão ou chave | — | CA-001-05, 06, 09, 30, 31 |
| `GET /integrations` | Sessão | `integrations.manage` | CA-001-27, 28 |
| `POST /integrations` | Sessão | `integrations.manage` | CA-001-17, 18, 27, 28 |
| `GET /integrations/{id}` | Sessão | `integrations.manage` | — |
| `PATCH /integrations/{id}` | Sessão | `integrations.manage` | CA-001-18, 22 |
| `DELETE /integrations/{id}` | Sessão | `integrations.manage` | CA-001-09 |
| `GET /integrations/{id}/tokens` | Sessão | `integrations.manage` | CA-001-23, 27, 28 |
| `POST /integrations/{id}/tokens` | Sessão | `integrations.manage` | CA-001-19, 26, 27, 28, 29 |
| `GET /integrations/{id}/tokens/{tokenId}` | Sessão | `integrations.manage` | — |
| `DELETE /integrations/{id}/tokens/{tokenId}` | Sessão | `integrations.manage` | CA-001-24, 26 |

Os critérios de uso da chave (CA-001-20, 21, 22, 25) valem para qualquer endpoint protegido. Os testes usam `GET /me` e uma rota de teste protegida por permissão.

### Autenticação de pessoas

#### `GET /auth/csrf-cookie`
Rota do Sanctum com o prefixo reconfigurado (`sanctum.prefix`). Devolve `204` e o cookie `XSRF-TOKEN`. A interface chama antes do login, e envia o cabeçalho `X-XSRF-TOKEN` nas requisições que alteram dados.

#### `POST /auth/login`
**Entrada:** `email` (obrigatório, e-mail), `password` (obrigatório), `remember` (booleano, opcional).

**Regras:**
- `Fortify::authenticateUsing` busca por e-mail em minúsculas, **apenas** `type = person` e não desativado. Qualquer falha devolve `null`, e o Fortify responde com a mesma mensagem (`auth.failed`) (RN-001-02).
- Limitador `login`: 3 por minuto, chave `email|ip` (RN-001-03).
- A sessão é regenerada após o login (proteção contra fixação de sessão).

**Saída:** `200` com o mesmo corpo de `GET /me`, para que a interface não precise de uma segunda chamada.

**Erros:** `422` credenciais inválidas (mensagem única) · `429` bloqueio, com cabeçalho `Retry-After`.

#### `POST /auth/logout`
Encerra e invalida a sessão, apaga o cookie de "manter conectado" e regenera o token CSRF. **Saída:** `204`.

#### `POST /auth/forgot-password`
**Entrada:** `email`.

**Regras:** sempre responde com a mesma mensagem genérica, inclusive para e-mail inexistente, conta de integração ou pedido limitado pelo throttle do broker (RN-001-08). O throttle também revelaria se o e-mail existe, por isso é mascarado. O e-mail sai **em fila** (notificação própria com `ShouldQueue`). O link aponta para a interface: `FRONTEND_URL/reset-password?token=…&email=…`.

**Saída:** `200` `{ "message": "Se o e-mail estiver cadastrado, você receberá um link para redefinir a senha." }`

#### `POST /auth/reset-password`
**Entrada:** `token`, `email`, `password`, `password_confirmation`.

**Regras:** broker de senhas do Laravel (token de uso único, 60 min); regra de senha forte; ao concluir, apaga **todas** as sessões da pessoa (RN-001-09).

**Saída:** `200` com mensagem. **Erros:** `422` token inválido, expirado ou usado; senha fora da regra.

#### `POST /me/password`
**Entrada:** `current_password`, `password`, `password_confirmation`.

**Regras:** valida a senha atual (RN-001-10) e a regra de senha forte; troca o hash; apaga as **outras** sessões da pessoa (`sessions` com o mesmo `user_id` e `id` diferente do atual) e renova o `remember_token`.

**Saída:** `204`. **Erros:** `422`.

> O Fortify usa `PUT /user/password`, e as convenções não usam `PUT`. Por isso esse recurso do Fortify fica desligado e a ação é um endpoint próprio, que reaproveita a classe de ação do Fortify.

### Identidade

#### `GET /me`
**Saída `200`:**

```json
{
  "data": {
    "id": "01J…",
    "type": "person",
    "name": "Ana Souza",
    "email": "ana@exemplo.com",
    "organization": { "id": "01J…", "name": "Hospital X" },
    "department": { "id": "01J…", "name": "Recepção" },
    "channel": "web",
    "roles": ["Requester", "DepartmentManager"],
    "permissions": ["tickets.create", "tickets.view.organization", "…"]
  }
}
```

Para integrações: `type` é `integration`, `email` e `department` são `null`, e `channel` é o da conta (CA-001-31). Papéis e permissões são carregados de uma vez, sem N+1 (P4).

### Contas de integração

Recurso próprio. Contas de integração **não aparecem** em `/users` (spec 003), que lista apenas pessoas.

#### `GET /integrations`
Filtros: `filter[name]` (parcial), `filter[organization_id]`, `filter[channel]`, `filter[trashed]`. Ordenação: `name`, `created_at`. Inclusões: `organization`, `roles`. Paginação padrão.

#### `POST /integrations`
**Entrada:**

| Campo | Regra |
|---|---|
| `name` | Obrigatório, até 150, único entre integrações ativas |
| `organization_id` | Obrigatório, organização existente e ativa |
| `channel` | Obrigatório, `api`, `agent` ou `monitoring` (RN-001-19) |
| `role_ids` | Lista opcional de papéis existentes |

**Saída:** `201` + `Location`, com a integração (`id`, `name`, `organization`, `channel`, `roles`, `created_at`).

**Erros:** `403` sem permissão ou autenticado por chave · `422` validação.

#### `GET /integrations/{id}` · `PATCH /integrations/{id}`
Leitura e edição de `name`, `organization_id`, `channel` e `role_ids`, com as mesmas regras. Mudança de papéis vale na próxima requisição da chave (CA-001-22): as permissões são consultadas a cada requisição, e o cache do spatie é invalidado na alteração.

#### `DELETE /integrations/{id}`
Desativa (soft delete). As chaves continuam no banco, mas deixam de autenticar porque a conta está desativada (CA-001-09). **Saída:** `204`.

### Chaves de API

#### `GET /integrations/{id}/tokens`
Lista `id`, `name`, `created_at`, `last_used_at`, `expires_at`. Nunca o valor.

#### `POST /integrations/{id}/tokens`
**Entrada:** `name` (obrigatório, até 100, único na conta), `expires_at` (opcional, data futura).

**Saída:** `201` + `Location`:

```json
{
  "data": {
    "id": "01J…",
    "name": "servidor 01",
    "expires_at": null,
    "created_at": "2026-10-12T15:00:00Z",
    "token": "chm_1|aB3…"
  }
}
```

`token` só existe nesta resposta (RN-001-15). O prefixo `chm_` (`sanctum.token_prefix`) permite que scanners de segredos, como o do GitHub, reconheçam uma chave vazada.

#### `GET /integrations/{id}/tokens/{tokenId}` · `DELETE /integrations/{id}/tokens/{tokenId}`
Leitura dos metadados e revogação. A revogação apaga a chave (`204`); o registro permanece na auditoria.

### Uso da chave

Validação no guard do Sanctum, via `Sanctum::authenticateAccessTokensUsing`. A chave é aceita somente se:
1. é válida e não expirou (`expires_at`) (RN-001-16);
2. o dono existe, **não está desativado** e é do tipo `integration` (RN-001-05, RN-001-13).

Recusas devolvem `401`. A autorização segue por permissões (`can`), como para pessoas (P7).

### Restrição "só sessão"

Middleware `session.only` nas rotas de `/integrations`: se a requisição foi autenticada por chave, responde `403` (RN-001-18, CA-001-28). Na autenticação por sessão, o Sanctum usa um `TransientToken`, o que permite distinguir os dois casos.

## Canal de origem

Serviço `ChannelResolver`:
- requisição autenticada por sessão → `web`;
- requisição autenticada por chave → `user.channel`;
- sem autenticação (ex.: recuperação de senha) → `web`.

Campos `channel` enviados pelo cliente são ignorados (RN-001-19). Usado na auditoria (esta spec) e na criação de chamados (spec 006).

## Eventos, filas e tarefas agendadas

| Item | Tipo | Detalhe |
|---|---|---|
| E-mail de recuperação de senha | Fila | Notificação com `ShouldQueue`. O conteúdo final é da spec 011 |
| Limpeza de tokens de recuperação expirados | Agendada | `auth:clear-resets`, diária |
| Limpeza de sessões expiradas | Automática | Coleta de lixo do driver `database` do Laravel |

## Auditoria

Via spatie/laravel-activitylog, `log_name = auth`. Cada registro guarda `causer` (quando identificado), `properties.ip` e `properties.channel` (RN-001-20).

| Evento | `description` | Observação |
|---|---|---|
| Login | `login` | |
| Falha de login | `login_failed` | Guarda o e-mail tentado (necessário para investigar ataques); nunca a senha |
| Logout | `logout` | |
| Pedido de recuperação | `password_reset_requested` | Só quando o e-mail existe; o registro não é exposto ao solicitante |
| Senha redefinida | `password_reset` | |
| Senha trocada | `password_changed` | |
| Integração criada, editada, desativada | `integration_created`, `integration_updated`, `integration_deactivated` | Edição guarda campos antes e depois |
| Chave gerada, revogada | `token_created`, `token_revoked` | Guarda id e nome da chave; nunca o valor |

**Segredos (RN-001-07, CA-001-33):** `password`, `password_confirmation`, `current_password` e `token` ficam na lista `dontFlash` do handler de exceções, e nenhum log registra corpo de requisição. O teste do CA-001-33 procura os valores usados nos testes dentro de `activity_log` e do arquivo de log.

## Decisões

| Decisão | Alternativas consideradas | Motivo |
|---|---|---|
| Contas de integração na tabela `users`, com `type` | Tabela própria `integrations` | Chamados, comentários e auditoria referenciam `users`; uma tabela separada exigiria relacionamentos polimórficos em todo o sistema. Papéis e permissões do spatie funcionam sem adaptação |
| Restrição `CHECK` no banco para `type` | Só validação na aplicação | Garante coerência mesmo em seeders, comandos e futuros caminhos (P6) |
| E-mail gravado em minúsculas, com índice único simples | Coluna `citext` ou índice em `lower(email)` | Mais simples (P15) e portável; a comparação sem diferenciar maiúsculas vem da normalização |
| Sessões no driver `database` | Redis | Permite encerrar as sessões de uma pessoa por `user_id` (RN-001-09) de forma explícita e testável, e abre caminho para "ver minhas sessões" no futuro. Reavaliar se o teste de carga apontar gargalo (P4) |
| Rotas do Fortify sob `/api/v1/auth` | Rotas padrão (`/login`) | Uma única base de URL para toda a API (convenções §1) |
| Troca de senha em `POST /me/password` | `PUT /user/password` do Fortify | As convenções não usam `PUT`; trocar senha é uma ação de negócio |
| `forgot-password` mascara também o throttle | Devolver o erro do throttle | O throttle só acontece para e-mails existentes, então revelaria o cadastro (P12) |
| Login devolve o mesmo corpo de `/me` | `204` sem corpo | Economiza uma requisição na entrada (P1, P4) |
| Revogar chave apaga o registro | Soft delete do token | A chave não tem vínculos (P10 permite apagar); o rastro fica na auditoria (P11) e a listagem mostra só chaves válidas |
| Prefixo `chm_` nas chaves | Sem prefixo | Detecção automática de vazamento por scanners de segredos |
| `last_used_at` atualizado pelo Sanctum a cada requisição | Atualização amostrada | Comportamento nativo, exigido pelo CA-001-20. Custo de uma escrita por requisição de integração; reavaliar no teste de carga |

## Princípios da constituição

| Princípio | Como o design atende |
|---|---|
| P1 Simplicidade para quem pede | Login só com e-mail e senha; o login já devolve os dados do usuário |
| P4 Resposta rápida | Papéis e permissões carregados de uma vez em `/me`; cache de permissões do spatie; listagens paginadas |
| P5 API-first | Tudo, inclusive o gerenciamento de integrações, é endpoint. "Só sessão" não cria função exclusiva da interface |
| P6 O servidor decide | Canal definido pelo servidor; restrição `CHECK` por tipo; senha forte validada no backend |
| P7 Permissão, não papel | `integrations.manage` é verificada; o código nunca pergunta "é Admin?" |
| P8, P9 Sem terceiros | Nenhuma verificação externa de senha; e-mail em fila, sem bloquear a requisição |
| P10 Nada com histórico é apagado | Pessoas e integrações usam soft delete; só a chave revogada, sem vínculos, é apagada |
| P11 Rastro | Todos os eventos de autenticação e de integrações na auditoria, com canal |
| P12 Privacidade | Mensagens genéricas; `404` para recurso invisível; segredos fora de logs e auditoria |

## Ajustes feitos nos requisitos

| Item | Ajuste | Motivo |
|---|---|---|
| CA-001-29 | Gerar chave para uma pessoa responde **`404`**, não `422` | A rota é `/integrations/{id}/tokens`: o `id` de uma pessoa não é uma integração, então o recurso não existe nesse endereço. É o comportamento padrão das convenções e não revela nada sobre a pessoa |

## Decisões em aberto

| # | Decisão | Resposta |
|---|---|---|
| D1 | Nenhuma spec da fase 1 tratava o **gerenciamento de configurações** (`settings.manage`) | Criada a **spec 013 — Configurações** (`GET /settings`, `PATCH /settings`, `POST /settings/{key}/reset`). Nesta spec, os valores vêm do serviço `Settings`, com o padrão de fábrica em `config/` |
| D2 | Endereço da interface para o link de recuperação de senha, já que o frontend ainda não foi decidido (Q2) Variável `FRONTEND_URL` no `.env`, preenchida por quem instala. É configuração da instalação, não do negócio, por isso não fica na spec 013 |

## Histórico de alterações

| Data | Alteração | Motivo |
|---|---|---|
| 2026-10-12 | Rascunho inicial | — |
| 2026-10-12 | D1 e D2 resolvidas; design aprovado | Decisões do responsável pelo produto |
| 2026-10-12 | Dependência "esqueleto do repositório" passa a apontar para a spec 000; activity log e serviço `Settings` vêm de lá | Criação da spec 000 (Fundação) |
