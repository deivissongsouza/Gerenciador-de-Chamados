# 000 — Fundação: Design

| Status | Data | Aprovado por |
|---|---|---|
| Design aprovado | 2026-10-12 | Dêvisson Souza |

Requisitos: [requisitos.md](requisitos.md)

## Visão geral

- Projeto **Laravel 13** apenas de API (sem Vite, sem telas), em **PHP 8.5**, rodando em Docker com sete serviços. O Makefile é a única porta de entrada.
- Qualidade: **Pest**, **Pint** e **Larastan**, reunidos em `make check`. Rodam no git hook de pré-commit e no **GitHub Actions**, sempre dentro da mesma imagem.
- Base transversal da API: erros RFC 9457, idioma por cabeçalho, datas em UTC sem microssegundos, documentação Scramble, endpoint de saúde, modelo base com ULID e soft delete, activity log e serviço `Settings`.
- Harness do Claude Code: `CLAUDE.md`, seis skills, dois subagentes, permissões, um hook de formatação e o MCP Laravel Boost.

## Versões

Consultadas em 2026-10-12 nas fontes oficiais ([endoflife.date](https://endoflife.date), [Packagist](https://packagist.org), [Docker Hub](https://hub.docker.com) e GitHub Releases). Também ficam registradas na [stack](../../docs/stack.md) (RN-000-02).

### Plataforma e imagens

| Peça | Versão | Imagem fixada | Suporte até |
|---|---|---|---|
| PHP | 8.5.11 | `php:8.5.11-fpm-trixie` | Segurança até 2029-12-31 |
| Laravel (framework) | 13.35.0 | — | Correções até 2027-09-30, segurança até 2028-03-17 |
| Esqueleto Laravel | 13.11.0 | — | — |
| Composer | 2.10.3 | `composer:2.10.3` (só o binário, copiado na imagem) | — |
| PostgreSQL | 18.6 | `postgres:18.6-trixie` | 2030-11-14 |
| Redis | 8.10.2 | `redis:8.10.2-trixie` | Versão atual |
| Nginx (estável) | 1.30.5 | `nginxinc/nginx-unprivileged:1.30.5-trixie` | Ramo estável |
| Mailpit | 1.31.5 | `axllent/mailpit:v1.31.5` | Só desenvolvimento |
| Debian (base) | 13 "trixie" | — | — |
| Extensão phpredis (PECL) | 6.3.0 | — | — |

### Pacotes PHP

| Pacote | Versão | Uso |
|---|---|---|
| `laravel/horizon` | 5.50.0 | Filas |
| `laravel/pint` | 1.32.1 | Estilo (dev) |
| `laravel/boost` | 2.10.3 | MCP do Claude Code (dev) |
| `spatie/laravel-activitylog` | 5.1.1 | Auditoria |
| `spatie/laravel-query-builder` | 7.3.5 | Listagens (usado a partir da spec 001) |
| `dedoc/scramble` | 0.13.47 | Documentação OpenAPI |
| `pestphp/pest` | 5.3.2 | Testes (dev) |
| `larastan/larastan` | 3.13.0 | Análise estática (dev) |

Instalados pelas specs seguintes, já verificados como compatíveis com Laravel 13: `laravel/sanctum` 4.3.3, `laravel/fortify` 1.41.0 e `spatie/laravel-permission` 8.3.0.

### Ações do GitHub

| Ação | Versão |
|---|---|
| `actions/checkout` | v7.0.1 |
| `docker/setup-buildx-action` | v4.4.1 |
| `docker/build-push-action` | v7.4.0 |

### Política de versões

- **Imagens Docker:** fixadas na versão exata (patch), para builds reproduzíveis.
- **Composer:** restrição `^` no `composer.json` e versões exatas travadas no `composer.lock`, que é versionado.
- **Atualizações:** o **Dependabot** do GitHub (gratuito) abre pull requests semanais para Composer, imagens Docker e ações do GitHub. Cada atualização passa pelo CI antes do merge e atualiza a tabela acima e a stack.

## Estrutura do repositório

```
.
├── .claude/
│   ├── agents/           revisor-spec.md, revisor-codigo.md
│   ├── skills/           spec-nova/, spec-requisitos/, spec-design/,
│   │                     spec-tarefas/, tarefa-proxima/, spec-status/
│   └── settings.json     permissões e hooks
├── .github/
│   ├── workflows/        ci.yml, mirror.yml
│   └── dependabot.yml
├── .githooks/pre-commit
├── .gitlab-ci.yml
├── .mcp.json             MCP Laravel Boost
├── docker/
│   ├── php/              Dockerfile, php.ini, www.conf
│   └── nginx/            default.conf
├── docker-compose.yml
├── Makefile
├── CLAUDE.md
├── docs/  specs/         (já existem)
└── app/ bootstrap/ config/ database/ lang/ routes/ tests/ …   (Laravel)
```

O Laravel fica na **raiz** do repositório, sem subpasta, porque é a única aplicação.

## Ambiente Docker

### Dockerfile (`docker/php/Dockerfile`)

| Estágio | Conteúdo |
|---|---|
| `base` | `php:8.5.11-fpm-trixie` + extensões `pdo_pgsql`, `intl`, `zip`, `bcmath`, `pcntl` (Horizon), `opcache` e `redis` (PECL 6.3.0). Usuário `app` não-root, com UID e GID iguais aos do usuário do WSL (argumentos de build), para evitar problemas de permissão nos arquivos montados |
| `dev` | `base` + binário do Composer + `php.ini` de desenvolvimento |
| `prod` | Fora do escopo (RN-000-04); o estágio fica reservado |

### Serviços (`docker-compose.yml`)

| Serviço | Imagem | Porta no host | Healthcheck | Usuário |
|---|---|---|---|---|
| `app` | estágio `dev` | — | `php-fpm-healthcheck` (status do FPM) | `app` |
| `web` | `nginx-unprivileged` | `8000` → 8080 | `GET /up` | `nginx` (não-root) |
| `horizon` | estágio `dev` | — | `php artisan horizon:status` | `app` |
| `scheduler` | estágio `dev` | — | processo `schedule:work` ativo | `app` |
| `postgres` | `postgres:18.6-trixie` | `5432` (só local) | `pg_isready` | `postgres` |
| `redis` | `redis:8.10.2-trixie` | — | `redis-cli ping` | `redis` |
| `mailpit` | `axllent/mailpit` | `8025` (tela) | `/livez` | padrão da imagem |

- Volumes nomeados para os dados do PostgreSQL e do Redis. O código é montado em `app`, `horizon` e `scheduler`.
- Um banco separado, `chamados_test`, é criado no primeiro start, para que os testes nunca toquem o banco de desenvolvimento.
- A rede é interna; só `web`, `postgres` e a tela do `mailpit` expõem portas, e apenas em `127.0.0.1`.

### Makefile

| Alvo | Ação |
|---|---|
| `help` | Lista os alvos com descrição (padrão quando se roda só `make`) |
| `setup` | Cria o `.env` a partir do `.env.example` (se não existir), constrói as imagens, sobe os serviços, instala dependências, gera a chave da aplicação, roda as migrations e ativa o git hook (`core.hooksPath`) |
| `up` · `down` · `restart` | Sobe, derruba (sem apagar volumes) e reinicia |
| `shell` | Terminal no container `app` |
| `logs` | Logs dos serviços |
| `migrate` | Roda as migrations |
| `test` | Pest, no banco de testes |
| `lint` | Pint em modo verificação |
| `fix` | Pint corrigindo os arquivos |
| `analyse` | Larastan |
| `check` | `lint` + `analyse` + `test`; falha no primeiro erro. É a definição de pronto (RN-000-07) |
| `ci` | Igual ao `check`, sem TTY, usado pelo CI |

Não existe alvo que apague volumes: isso fica manual e explícito, de propósito.

## Qualidade

| Ferramenta | Configuração |
|---|---|
| Pest | `tests/Feature` e `tests/Unit`. Banco `chamados_test` com `RefreshDatabase`. `Mail::fake`, `Queue::fake` e `Storage::fake` disponíveis (RN-000-08) |
| Pint | Preset `laravel` |
| Larastan | **Nível 6** em `app/`, `config/`, `database/` e `routes/`. Sem *baseline*: o projeto nasce limpo |
| N+1 | `Model::shouldBeStrict()` fora de produção: impede *lazy loading* e acesso a atributos inexistentes (RN-000-09) |

### Git hook de pré-commit (`.githooks/pre-commit`)

Executa `make check`. Se os containers não estiverem rodando, falha com a mensagem "suba o ambiente com `make up`". Vale para pessoas e para o Claude Code, que faz commits pelo mesmo git (RN-000-11).

### CI (`.github/workflows/ci.yml`)

1. Gatilhos: push em qualquer branch e pull request para `main`.
2. Constrói a imagem `dev` com cache do GitHub Actions (buildx).
3. Sobe `postgres` e `redis` pelo mesmo `docker-compose.yml`.
4. Executa `make ci`.

### Espelho (`.github/workflows/mirror.yml`)

Em push na `main`: `git push --mirror` para o GitLab, autenticado pelo segredo `GITLAB_TOKEN` (token de projeto com permissão de escrita no repositório) e pela variável `GITLAB_REPO`. Sem esses dados configurados, o job é ignorado em vez de falhar.

### GitLab CI (`.gitlab-ci.yml`)

Um job `check` com `when: manual`, que constrói a imagem e roda `make ci` (RN-000-27).

### Dependabot (`.github/dependabot.yml`)

Semanal, para `composer`, `docker` e `github-actions`, com no máximo 5 pull requests abertos.

## Comportamentos transversais da API

### Erros RFC 9457 (RN-000-12, RN-000-13)

Renderização centralizada em `bootstrap/app.php` (`withExceptions`), para toda rota sob `/api`. Toda resposta tem `Content-Type: application/problem+json`.

| Exceção | Status | `type` (sufixo) |
|---|---|---|
| `ValidationException` | 422 | `validation` (com `errors` por campo) |
| `AuthenticationException` | 401 | `unauthenticated` |
| `AuthorizationException`, `AccessDeniedHttpException` | 403 | `forbidden` |
| `ModelNotFoundException`, `NotFoundHttpException` | 404 | `not-found` |
| `ConflictHttpException` | 409 | `conflict` |
| `ThrottleRequestsException` | 429 | `too-many-requests` (com `Retry-After`) |
| Outras `HttpException` | o próprio | `http-error` |
| Qualquer outra | 500 | `internal-error` |

- `type` = `{APP_URL}/errors/{sufixo}`. `title` e `detail` vêm dos arquivos de tradução.
- Com `APP_DEBUG=true` (somente em desenvolvimento), a resposta ganha um campo `debug` com exceção e trilha. Em qualquer outro caso, nunca.
- Um middleware força `Accept: application/json` nas rotas da API, para que nenhum erro vire página HTML ou redirecionamento.

### Idioma (RN-000-14)

- Middleware `SetLocaleFromHeader`: lê `Accept-Language`, escolhe o primeiro idioma da lista `config('app.supported_locales')` (`['pt_BR']`) e, sem correspondência, usa `pt_BR`.
- As traduções `pt_BR` do Laravel (validação, autenticação, senhas, paginação) são **copiadas uma vez** para `lang/pt_BR` e versionadas, sem dependência permanente (P15). As mensagens de erro da API ficam em `lang/pt_BR/errors.php`.

### Datas (RN-000-15)

`APP_TIMEZONE=UTC`. A serialização de datas é configurada uma única vez (`Date::serializeUsing`) para o formato `Y-m-d\TH:i:s\Z`. O padrão do Laravel inclui microssegundos (`.000000Z`), fora da convenção.

### Documentação (RN-000-16)

Scramble em `/docs/api` (tela) e `/docs/api.json` (especificação), acessível sem autenticação em todos os ambientes, pois não contém dados.

### Saúde (RN-000-17)

`GET /up`, fora de `/api/v1`, sem autenticação e sem limite de requisições.

```json
{ "status": "ok", "checks": { "app": "ok", "database": "ok", "redis": "ok" } }
```

Responde `200` quando tudo está `ok`; `503` quando algo falha, com o item marcado como `fail` e sem mensagem de erro (P12). O healthcheck do `web` usa esse endpoint.

### Modelo base (RN-000-18)

Classe abstrata `App\Models\Model`, com os traits `HasUlids` e `SoftDeletes`, que todas as entidades de domínio estendem. Modelos que precisam estender outra classe (ex.: `User` estende `Authenticatable`) usam os mesmos traits diretamente.

### Auditoria (RN-000-19)

`spatie/laravel-activitylog` instalado. A migration usa `nullableUlidMorphs` em `subject` e `causer`. O canal de origem (`properties.channel`) é acrescentado pela spec 001, que cria o `ChannelResolver`.

### Configurações (RN-000-20)

| Peça | Detalhe |
|---|---|
| Padrões de fábrica | `config/settings.php`: chave → valor padrão. Esta spec não define nenhuma chave real; a spec 001 adiciona as de sessão |
| Tabela `settings` | `key` (string, PK), `value` (jsonb), `updated_by` (ulid, nulo), `updated_at`. Só existe linha para valores personalizados |
| Serviço `App\Support\Settings` | `get(key)`: personalização ou padrão. `set(key, value)` e `reset(key)`: usados pelos testes e, depois, pela spec 013. Chave desconhecida lança exceção. Leitura em cache (Redis), invalidado em `set` e `reset` |

## Harness do Claude Code

O Claude Code roda **dentro do WSL**, no diretório do projeto, como definido no [harness](../../docs/harness.md).

### `CLAUDE.md`

No máximo uma página: visão em cinco linhas; mapa de `docs/` e `specs/` (com links, sem copiar conteúdo); comandos do Makefile; definição de pronto (`make check`); resumo do processo SDD e da rastreabilidade (CA → tarefa → teste → commit); convenção de idioma; fluxo de branches.

### Skills (`.claude/skills/`)

| Skill | Faz | Recusa quando |
|---|---|---|
| `/spec-nova` | Cria `specs/NNN-nome/` a partir de `_modelos/`, registra no índice e cria a branch `spec/NNN-nome` | O número já existe |
| `/spec-requisitos` | Conduz `requisitos.md`: propõe, lista as dúvidas com proposta e só marca aprovado com o "aprovado" explícito do responsável | — |
| `/spec-design` | Conduz `design.md` | Requisitos não estão "aprovados" (CA-000-23) |
| `/spec-tarefas` | Conduz `tarefas.md`, com a tabela de cobertura | Design não está "aprovado" |
| `/tarefa-proxima` | Lê a próxima tarefa pendente; escreve o teste do critério (falhando); implementa; roda `make check`; aciona o revisor de código; marca a tarefa e propõe o commit | Tarefas não estão "aprovadas" ou há alterações não commitadas |
| `/spec-status` | Mostra o status de cada spec e a próxima ação | — |

### Subagentes (`.claude/agents/`)

| Subagente | Lê | Verifica |
|---|---|---|
| `revisor-spec` | A spec, a constituição, o glossário, as permissões e as convenções | Termos fora do glossário, conflito com princípios, critério sem caso de erro ou de permissão, dúvida sem resposta |
| `revisor-codigo` | O diff, a spec e a tarefa | Critério sem teste, regra da spec não implementada, N+1, autorização por papel em vez de permissão (P7), segredo em log (P12), código além do escopo da tarefa |

Os dois só leem e relatam; não editam.

### Permissões (`.claude/settings.json`)

| Nível | Regras |
|---|---|
| `allow` | `Bash(make help)`, `Bash(make test*)`, `Bash(make lint)`, `Bash(make fix)`, `Bash(make analyse)`, `Bash(make check)`, `Bash(git status*)`, `Bash(git diff*)`, `Bash(git log*)`, leitura de arquivos |
| `ask` | `Bash(git commit*)`, `Edit(docs/constituicao.md)` |
| `deny` | `Read(.env)`, `Read(.env.*)` exceto `.env.example`, `Bash(git push*)`, `Bash(docker compose down -v*)`, `Bash(docker volume rm*)`, `Bash(*migrate:fresh*)`, `Bash(*db:wipe*)` |

### Hook (`.claude/settings.json`)

| Momento | Ação |
|---|---|
| `PostToolUse` em `Edit` e `Write` de arquivo `.php` | `docker compose exec -T app vendor/bin/pint <arquivo>` |

A verificação antes do commit fica **só no git hook** (ver Decisões), que vale para o Claude Code e para pessoas.

### MCP (`.mcp.json`)

Servidor `laravel-boost`: `docker compose exec -T app php artisan boost:mcp`.

### Verificação manual (CA-000-22 a 25)

A tarefa do harness traz um roteiro: abrir uma sessão nova, pedir `/spec-design` numa spec em rascunho, pedir leitura do `.env`, `git push` e `migrate:fresh`, editar um PHP desalinhado. O resultado de cada passo fica registrado na própria tarefa.

## Ações manuais do responsável pelo produto

Itens que dependem da sua conta e de credenciais, e que o Claude Code não faz:

| Quando | Ação |
|---|---|
| Após o primeiro push com o CI | GitHub → Settings → Branches: proteger a `main` (exigir pull request e o check `ci` aprovado) |
| Para ativar o espelho | Criar o projeto vazio no GitLab; criar um token de projeto com escrita no repositório; no GitHub, cadastrar o segredo `GITLAB_TOKEN` e a variável `GITLAB_REPO` |
| Para ativar o Dependabot | GitHub → Settings → Code security: ligar as atualizações de versão (o arquivo de configuração já vem no repositório) |

## Decisões

| Decisão | Alternativas consideradas | Motivo |
|---|---|---|
| PHP 8.5 | PHP 8.4 | Mais recente suportado pelo Laravel 13 (RN-000-02), com suporte de segurança até 2029 |
| Imagens Debian (`trixie`) | Alpine | Evita incompatibilidades da biblioteca musl com extensões; diferença de tamanho irrelevante para o projeto |
| `nginx-unprivileged` | Imagem oficial `nginx` | A oficial inicia como root; a variante oficial sem privilégios atende ao RN-000-03 sem configuração manual |
| Imagens fixadas no patch, com Dependabot | Fixar só o número principal (ex.: `php:8.5`) | Builds reproduzíveis e toda versão registrada (RN-000-02); o Dependabot traz as correções de segurança por pull request |
| Verificação antes do commit só no git hook | Git hook e também hook do Claude Code | Um mecanismo só, que vale para todos (P15). O hook do Claude Code duplicaria a execução dos testes |
| Traduções `pt_BR` copiadas e versionadas | Pacote de tradução como dependência | Os arquivos mudam raramente; sem dependência permanente (P15) |
| Laravel sem Vite e sem `package.json` | Manter o esqueleto padrão | O projeto é só API (P5); Node seria uma dependência sem uso |
| Larastan nível 6 | Nível máximo (10) | Equilíbrio entre rigor e produtividade num projeto que nasce agora. Subir de nível é uma tarefa futura, sem retrabalho |
| Banco de testes separado | Mesmo banco com transações | Os testes nunca apagam dados de desenvolvimento |
| Scramble público | Restrito por login | A especificação não contém dados; quem integra precisa dela antes de ter acesso |
| Saúde em `/up`, fora de `/api/v1` | `/api/v1/health` | É do ambiente, não da API de negócio; monitores e o healthcheck do Docker usam uma URL estável |

**Ressalva sobre o Scramble:** a tela de `/docs/api` carrega componentes visuais de uma CDN pública. Em instalações sem acesso à internet, a tela não abre, mas a especificação em `/docs/api.json` continua disponível. A documentação não é funcionalidade de negócio, então o P9 continua atendido.

## Princípios da constituição

| Princípio | Como o design atende |
|---|---|
| P4 Resposta rápida | `shouldBeStrict` impede N+1 desde o início; OPcache na imagem; cache das configurações |
| P5 API-first | Projeto só de API; documentação gerada do código |
| P6 O servidor decide | Erros padronizados e idioma decididos no servidor |
| P8, P9 Sem terceiros | Tudo roda nos containers; testes com *fakes*; Mailpit em desenvolvimento |
| P10 Nada é apagado | Modelo base com soft delete |
| P11 Rastro | Activity log instalado e configurado |
| P12 Privacidade | Nenhum detalhe interno em erros fora de desenvolvimento; `.env` fora do git e negado ao Claude Code |
| P13 Spec antes de código | Skills recusam etapas sem aprovação |
| P14 Critério vira teste | `make check` no hook e no CI; testes sem serviços externos |
| P15 Simplicidade | Um Makefile, um mecanismo de bloqueio de commit, sem Node, sem pacote de tradução |

## Ajustes feitos nos requisitos

| Item | Ajuste | Motivo |
|---|---|---|
| RN-000-25 | O hook do Claude Code **só formata com o Pint**. A verificação antes do commit fica no git hook (RN-000-11), que já bloqueia commits do Claude Code. O [harness](../../docs/harness.md) é atualizado igual | Hoje a regra pede dois mecanismos para a mesma coisa, e os testes rodariam duas vezes a cada commit do Claude Code (P15) |

## Histórico de alterações

| Data | Alteração | Motivo |
|---|---|---|
| 2026-10-12 | Rascunho inicial | — |
| 2026-10-12 | Ajuste no RN-000-25 e ressalva do Scramble aceitos; design aprovado | Decisões do responsável pelo produto |
