# Stack do Backend

> Decisões técnicas da fase 1 e seus motivos. Princípios referenciados em [constituicao.md](constituicao.md).
> Última revisão: 2026-10-12.

## Base

| Peça | Escolha | Motivo | Princípios |
|---|---|---|---|
| Linguagem | PHP 8.4 ou superior | Versão atual, desempenho e tipagem | P4 |
| Framework | Laravel, versão estável mais recente na criação do projeto | Filas, eventos, agendamento e autenticação prontos | P15 |
| Banco de dados | PostgreSQL | Robusto para auto-hospedagem, bom com JSON e busca textual | P9 |
| Cache e filas | Redis | E-mails em fila e métricas em cache | P4 |
| Monitor de filas | Laravel Horizon | Visibilidade de filas, falhas e retentativas | P8 |

## Autenticação e acesso

| Peça | Escolha | Motivo | Princípios |
|---|---|---|---|
| Sessão e chaves de API | Laravel Sanctum | Login por cookie HttpOnly (pessoas) e chaves de API (contas de integração) no mesmo pacote | P5, P6 |
| Fluxos de login | Laravel Fortify | Login, logout e recuperação de senha apenas como endpoints | P5 |
| Papéis e permissões | spatie/laravel-permission | Papéis agrupam permissões e o código verifica permissões | P7 |

## API

| Peça | Escolha | Motivo | Princípios |
|---|---|---|---|
| Formato das respostas | API Resources (nativo) | Respostas padronizadas e desacopladas do banco | P5 |
| Filtros, ordenação, paginação | spatie/laravel-query-builder | Padrão único (estilo JSON:API) em todas as listagens; só o que é declarado é aceito | P4, P5, P6 |
| Documentação | Scramble | OpenAPI gerado a partir do código, sempre atualizado | P5 |

Observação: o query builder não aplica permissões. Regras de visibilidade (organização, chamados privados) são aplicadas antes dele.

## Dados

| Peça | Escolha | Motivo | Princípios |
|---|---|---|---|
| Remoção | Soft delete nativo | Registros com histórico nunca são apagados | P10 |
| Auditoria | Pacote de activity log (ex.: spatie/laravel-activitylog) | Registro automático de alterações | P11 |
| Métricas de chamado | Momentos-chave (primeira resposta, resolução, fechamento) gravados no próprio chamado | Métricas rápidas, sem varrer o log de auditoria | P4 |

## Qualidade

| Peça | Escolha | Motivo | Princípios |
|---|---|---|---|
| Testes | Pest | Critérios de aceite como testes legíveis | P14 |
| Estilo | Laravel Pint | Formatação automática | P15 |
| Análise estática | Larastan | Erros encontrados sem executar o código | P14 |
| Teste de carga | k6 | Medição do p95 no cenário de referência | P4 |
| CI | GitHub Actions + GitLab CI | Arquivos finos que chamam os mesmos comandos do Makefile, na mesma imagem Docker. Repositório principal no GitHub, espelho no GitLab. Detalhes definidos na criação do esqueleto | P4, P14 |

## Ambiente

Docker próprio (sem Laravel Sail). Nada instalado na máquina além do Docker.

| Serviço | Função |
|---|---|
| app | PHP-FPM com as extensões necessárias; executa Laravel, Composer e Artisan |
| web | Nginx, repassando para o PHP-FPM |
| horizon | Processamento das filas |
| scheduler | Tarefas agendadas (ex.: verificação de SLA, fechamento automático) |
| postgres | Banco de dados |
| redis | Cache e filas |
| mailpit | Captura de e-mails em desenvolvimento |

Regras:
- Um Dockerfile com estágios (desenvolvimento agora, produção depois), compartilhando a mesma base.
- Versões principais sempre fixadas, nunca `latest`. Os números exatos são definidos na criação dos arquivos.
- Containers com usuário não-root e healthchecks.
- Atalhos de comando via **Makefile** (`make help` lista os comandos).
- Projeto mantido no sistema de arquivos do WSL, por desempenho.
- Em produção, nenhum proxy externo é obrigatório; o Nginx interno basta. Proxies como Traefik ficam a critério de quem hospeda.

## Fora da fase 1

| Item | Motivo |
|---|---|
| Laravel Octane | Complexidade desnecessária agora. Reavaliar se o teste de carga não cumprir o P4 |
| Frontend | A ser discutido em momento próprio |
