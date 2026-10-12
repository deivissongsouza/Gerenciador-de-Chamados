# 000 — Fundação: Requisitos

| Status | Data | Aprovado por |
|---|---|---|
| Requisitos aprovados | 2026-10-12 | Dêvisson Souza |

## Contexto

Antes de qualquer funcionalidade, o projeto precisa de uma base que todas as specs usam: o ambiente Docker, o projeto Laravel, as ferramentas de qualidade, a integração contínua, os comportamentos transversais da API (erros, idioma, documentação) e o harness do Claude Code que aplica o processo de specs.

Esta spec não entrega funcionalidade ao usuário final. Ela garante três coisas:
- **Qualquer pessoa sobe o projeto com um comando**, tendo apenas Docker e Make na máquina ([stack](../../docs/stack.md)).
- **Nada entra no repositório sem passar pelos checks**: testes, Pint e Larastan, localmente e no CI (P14).
- **O Claude Code segue o processo** descrito em [specs/README.md](../README.md), em vez de apenas conhecê-lo ([harness](../../docs/harness.md)).

## Histórias de usuário

- Como **desenvolvedor**, quero subir todo o ambiente com um comando para começar a trabalhar sem instalar PHP, PostgreSQL ou Redis na máquina.
- Como **desenvolvedor**, quero rodar todos os checks com um comando para saber, antes do commit, se a entrega está pronta.
- Como **responsável pelo produto**, quero que nenhum commit com teste quebrado ou fora do padrão entre no repositório.
- Como **responsável pelo produto**, quero que o Claude Code siga o processo de specs (uma tarefa, um teste, um commit) sem que eu precise repetir as regras a cada sessão.
- Como **cliente da API**, quero erros num formato padrão e estável para tratá-los programaticamente.
- Como **integrador**, quero a documentação da API sempre atualizada, gerada a partir do código.
- Como **quem hospeda o sistema**, quero verificar se a aplicação está saudável para monitorá-la.

## Regras de negócio

### Ambiente

| ID | Regra |
|---|---|
| RN-000-01 | O ambiente roda inteiro em Docker, com os serviços `app`, `web`, `horizon`, `scheduler`, `postgres`, `redis` e `mailpit` ([stack](../../docs/stack.md)). Nada além de Docker e Make é exigido na máquina |
| RN-000-02 | Todas as imagens e versões principais (Laravel, PHP, PostgreSQL, Redis, Nginx, Node se houver, pacotes e ferramentas) são fixadas e **registradas** no design desta spec e na [stack](../../docs/stack.md). Nenhuma usa `latest` |
| RN-000-03 | Os containers rodam com usuário não-root e têm healthcheck |
| RN-000-04 | Um único Dockerfile com estágios. Esta spec entrega o estágio de desenvolvimento; o de produção fica para depois |
| RN-000-05 | O Makefile é a única porta de entrada dos comandos (`make help` lista todos). CI, hooks e harness chamam os mesmos alvos |
| RN-000-06 | O `.env.example` traz todas as variáveis necessárias, com valores seguros para desenvolvimento e nenhum segredo real. O `.env` nunca é versionado |

### Qualidade

| ID | Regra |
|---|---|
| RN-000-07 | `make check` executa testes (Pest), estilo (Pint, modo verificação) e análise estática (Larastan) e falha se qualquer um falhar. É a definição de pronto |
| RN-000-08 | Testes não dependem de serviço externo real (P14). E-mails, filas e armazenamento usam *fakes* nos testes |
| RN-000-09 | Existe um teste que detecta consultas N+1 (Laravel `preventLazyLoading` ativo em desenvolvimento e testes) (P4) |
| RN-000-10 | O CI roda `make check` na mesma imagem Docker do desenvolvimento, a cada push e pull request, no GitHub Actions. A `main` é protegida: merge só por pull request com o CI aprovado |
| RN-000-11 | Um git hook de pré-commit roda os checks e bloqueia o commit se algo falhar, para pessoas e para o Claude Code |
| RN-000-27 | Cada push na `main` é espelhado no GitLab por um job do GitHub Actions. O GitLab CI existe, chama os mesmos alvos do Makefile e só roda manualmente, para não consumir a cota gratuita em duplicidade |
| RN-000-28 | O fluxo de trabalho é uma branch por spec (`spec/NNN-nome`), um commit por tarefa e pull request ao concluir a spec. A spec 000 é a exceção: nasce na `main`, antes de existir CI |

### Comportamentos transversais da API

| ID | Regra |
|---|---|
| RN-000-12 | Todo erro da API sai no formato RFC 9457 (`application/problem+json`), com `type`, `title`, `status`, `detail` e, em validação, `errors` ([convenções §6](../../docs/convencoes-api.md)) |
| RN-000-13 | Fora do ambiente de desenvolvimento, nenhum erro expõe detalhes internos (stack trace, SQL, caminhos de arquivo) (P12) |
| RN-000-14 | O idioma das mensagens vem do cabeçalho `Accept-Language`, dentro dos idiomas suportados. Na fase 1, `pt_BR` é o padrão e o único idioma ([convenções §8](../../docs/convencoes-api.md)) |
| RN-000-15 | Datas são gravadas e devolvidas em UTC, no formato ISO 8601 ([convenções §3](../../docs/convencoes-api.md)) |
| RN-000-16 | A documentação OpenAPI é gerada a partir do código (Scramble) e publicada em `/docs/api` |
| RN-000-17 | Existe um endpoint de saúde que indica se a aplicação, o banco e o Redis respondem, sem exigir autenticação e sem expor detalhes internos |
| RN-000-18 | Chaves primárias de domínio são ULID, e entidades de domínio usam soft delete. A base de modelos já nasce com esses padrões ([modelo de dados](../../docs/modelo-de-dados.md)) |
| RN-000-19 | O pacote de auditoria (activity log) é instalado e configurado com ULID, pronto para as specs registrarem eventos (P11) |
| RN-000-20 | Existe o serviço de configurações (`Settings`) com padrão de fábrica e personalização opcional, apenas para leitura. O gerenciamento pela API é da spec 013 |

### Harness do Claude Code

| ID | Regra |
|---|---|
| RN-000-21 | O `CLAUDE.md` da raiz é curto e **aponta** para `docs/` e `specs/` em vez de copiá-los. Contém visão em poucas linhas, comandos do Makefile, definição de pronto, resumo do processo e convenção de idioma |
| RN-000-22 | Os comandos do processo existem como skills: `/spec-nova`, `/spec-requisitos`, `/spec-design`, `/spec-tarefas`, `/tarefa-proxima` e `/spec-status`. Cada etapa recusa começar sem a aprovação da anterior |
| RN-000-23 | Existem os subagentes **revisor de spec** e **revisor de código**, que revisam com contexto limpo |
| RN-000-24 | Permissões do Claude Code: checks e leitura liberados; `git commit` e edição da constituição perguntam antes; ler `.env`, `git push`, remover volumes Docker e `migrate:fresh` são proibidos |
| RN-000-25 | Hooks do Claude Code: formata com Pint após editar arquivo PHP; roda os checks antes de `git commit` e bloqueia se falharem |
| RN-000-26 | O MCP Laravel Boost roda no container `app` e está configurado para o Claude Code |

## Critérios de aceite

### Ambiente

#### CA-000-01: Subir o ambiente do zero
- **Dado** um clone novo do repositório, numa máquina só com Docker e Make
- **Quando** se executa `make setup`
- **Então** as imagens são construídas, o `.env` é criado a partir do `.env.example`, as dependências são instaladas, as migrations rodam e todos os serviços ficam saudáveis

#### CA-000-02: Comandos documentados
- **Dado** o ambiente instalado
- **Quando** se executa `make help`
- **Então** são listados, com descrição, ao menos: `setup`, `up`, `down`, `shell`, `test`, `lint`, `fix`, `analyse`, `check`, `migrate` e `logs`

#### CA-000-03: Versões fixadas e registradas
- **Dado** o Dockerfile, o `docker-compose.yml`, o `composer.json` e a [stack](../../docs/stack.md)
- **Quando** são inspecionados
- **Então** nenhuma imagem usa `latest` nem versão sem número principal, e cada versão usada aparece registrada na stack

#### CA-000-04: Containers sem root e saudáveis
- **Dado** o ambiente em execução
- **Quando** se verifica o usuário e o estado de cada container
- **Então** nenhum processo da aplicação roda como root e todos os containers com healthcheck estão `healthy`

#### CA-000-05: Filas funcionando
- **Dado** o ambiente em execução
- **Quando** um job de teste é enviado para a fila
- **Então** o Horizon o processa

#### CA-000-06: E-mails capturados
- **Dado** o ambiente de desenvolvimento
- **Quando** a aplicação envia um e-mail
- **Então** ele aparece no Mailpit e não sai para a internet

### Qualidade

#### CA-000-07: Checks passam no projeto limpo
- **Dado** o projeto recém-criado
- **Quando** se executa `make check`
- **Então** testes, Pint e Larastan passam

#### CA-000-08: Checks falham quando devem
- **Dado** três alterações isoladas: um teste falhando, um arquivo fora do padrão do Pint e um erro de tipo detectável pelo Larastan
- **Quando** se executa `make check` com cada uma
- **Então** o comando falha nas três

#### CA-000-09: N+1 detectado
- **Dado** um teste que acessa um relacionamento não carregado dentro de um laço
- **Quando** o teste é executado
- **Então** ele falha com erro de carregamento preguiçoso (*lazy loading*)

#### CA-000-10: CI no GitHub
- **Dado** um push para o repositório
- **Quando** o GitHub Actions executa
- **Então** roda `make check` na imagem do projeto e marca o commit como aprovado ou reprovado

#### CA-000-26: Espelho no GitLab
- **Dado** um push na `main` do GitHub
- **Quando** o job de espelho termina
- **Então** o repositório do GitLab tem o mesmo commit na `main`

#### CA-000-11: Commit bloqueado
- **Dado** uma alteração que faz `make check` falhar
- **Quando** se tenta fazer `git commit`
- **Então** o hook de pré-commit bloqueia o commit

### Comportamentos transversais da API

#### CA-000-12: Erro 404 no formato padrão
- **Dado** uma rota inexistente em `/api/v1`
- **Quando** é chamada
- **Então** responde `404` com `Content-Type: application/problem+json` e os campos `type`, `title`, `status` e `detail`

#### CA-000-13: Erro de validação no formato padrão
- **Dado** um endpoint de teste com validação
- **Quando** recebe dados inválidos
- **Então** responde `422` no formato RFC 9457, com `errors` por campo e mensagens em português

#### CA-000-14: Erro interno sem detalhes
- **Dado** o ambiente configurado como produção e um endpoint de teste que lança exceção
- **Quando** é chamado
- **Então** responde `500` no formato RFC 9457, sem stack trace, SQL ou caminhos de arquivo

#### CA-000-15: Idioma pelo cabeçalho
- **Dado** um erro de validação
- **Quando** a requisição envia `Accept-Language: en` (não suportado) ou nenhum cabeçalho
- **Então** a mensagem sai em `pt_BR`

#### CA-000-16: Datas em UTC
- **Dado** um registro criado com o fuso da aplicação em UTC
- **Quando** é devolvido pela API
- **Então** as datas estão no formato `AAAA-MM-DDTHH:MM:SSZ`

#### CA-000-17: Documentação da API
- **Dado** o ambiente em execução
- **Quando** se acessa `/docs/api`
- **Então** a documentação OpenAPI é exibida, e o JSON da especificação está disponível

#### CA-000-18: Saúde da aplicação
- **Dado** o ambiente em execução
- **Quando** se chama o endpoint de saúde, sem autenticação
- **Então** responde `200` com o estado da aplicação, do banco e do Redis
- **E** com o banco fora do ar, responde `503`, sem detalhes internos

#### CA-000-19: Modelo base com ULID e soft delete
- **Dado** um modelo de domínio criado a partir da base
- **Quando** um registro é criado e depois removido
- **Então** o id é um ULID e o registro continua no banco com `deleted_at` preenchido

#### CA-000-20: Auditoria pronta
- **Dado** um modelo de teste configurado para auditoria
- **Quando** é alterado por um usuário autenticado
- **Então** o activity log registra o evento com o autor e os campos alterados

#### CA-000-21: Configuração com padrão de fábrica
- **Dado** uma configuração com padrão de fábrica `60`
- **Quando** é lida sem personalização, depois com personalização `30`, e depois com a personalização removida
- **Então** o serviço devolve `60`, `30` e `60`

### Harness do Claude Code

#### CA-000-22: Arquivos do harness presentes
- **Dado** o repositório
- **Quando** é inspecionado
- **Então** existem o `CLAUDE.md`, as seis skills do processo, os dois subagentes, o `.claude/settings.json` com permissões e hooks, e a configuração do MCP Laravel Boost

#### CA-000-23: Etapas respeitam aprovações
- **Dado** uma spec com requisitos em "Rascunho"
- **Quando** se executa `/spec-design` para ela
- **Então** o Claude Code recusa e informa que os requisitos precisam ser aprovados antes

#### CA-000-24: Permissões aplicadas
- **Dado** uma sessão do Claude Code no projeto
- **Quando** ele tenta ler o `.env`, executar `git push` ou `migrate:fresh`
- **Então** a ação é bloqueada

#### CA-000-25: Pint automático
- **Dado** uma sessão do Claude Code
- **Quando** ele edita um arquivo PHP fora do padrão
- **Então** o arquivo é formatado pelo Pint logo após a edição

> Os critérios do harness (CA-000-22 a 25) são verificados manualmente, com um roteiro registrado na tarefa, pois testam o comportamento do Claude Code, não da aplicação.

## Fora de escopo

| Item | Onde fica |
|---|---|
| Estágio de produção do Dockerfile e guia de instalação | Antes da primeira versão para clientes |
| Teste de carga com k6 | Antes de cada versão (P4); a estrutura pode ser criada quando houver endpoints reais |
| Gerenciamento de configurações pela API | Spec 013 |
| Autenticação e qualquer endpoint de negócio | Spec 001 em diante |
| Frontend | Questão Q2 da visão |

## Dúvidas em aberto

| # | Dúvida | Resposta |
|---|---|---|
| 1 | Fluxo de branches | Uma branch por spec, um commit por tarefa, pull request ao concluir a spec com CI obrigatório. A spec 000 nasce na `main` (RN-000-28) |
| 2 | Espelho no GitLab | Job do GitHub Actions espelha a `main` no GitLab; GitLab CI só manual. Funciona nas contas gratuitas: o repositório é público, com Actions ilimitado e proteção de branch disponível (RN-000-10, RN-000-27) |
| 3 | Versões | Laravel estável mais recente no dia da criação, com o PHP mais recente que ele suporta (mínimo 8.4). **Todas** as versões ficam registradas no design e na stack (RN-000-02) |
| 4 | Onde fica o serviço `Settings` | Nesta spec, só a leitura (RN-000-20). O gerenciamento pela API fica na spec 013 |
| 5 | Endpoint de saúde | `GET /up`, ampliado para checar banco e Redis (RN-000-17) |

## Histórico de alterações

| Data | Alteração | Motivo |
|---|---|---|
| 2026-10-12 | Rascunho inicial | — |
| 2026-10-12 | Respostas às dúvidas 1 a 5; RN-000-27, RN-000-28 e CA-000-26 adicionados | Decisões do responsável pelo produto |
| 2026-10-12 | Requisitos aprovados | — |
