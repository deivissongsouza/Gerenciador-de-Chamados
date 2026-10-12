# Glossário do Domínio

> Um nome para cada conceito: português nas specs e na documentação, inglês no código.
> Última revisão: 2026-10-12.

## Pessoas e acesso

| Termo | Código | Definição |
|---|---|---|
| Usuário | `User` | Qualquer pessoa com conta no sistema, independentemente do papel |
| Papel | `Role` | Conjunto nomeado de permissões |
| Permissão | `Permission` | Autorização para uma ação específica (ex.: `tickets.assign`) |
| Solicitante | `Requester` | Papel de quem pede atendimento. No chamado, a pessoa que precisa do atendimento (`requester`) |
| Técnico | `Technician` | Papel de quem atende chamados. Não usar `Agent`, reservado para agentes de IA |
| Gestor da Organização | `OrganizationManager` | Papel de quem acompanha os chamados e indicadores da própria organização (ex.: gerente do hospital) |
| Gestor | `Manager` | Papel de quem administra o sistema e acompanha os indicadores |
| Alcance | `.own`, `.organization`, `.linked`, `.all` | Conjunto de registros sobre os quais uma permissão vale (ver [permissoes.md](permissoes.md)) |
| Chave de API | `ApiToken` | Credencial de acesso à API, com escopos |
| Escopo | `ability` | O que uma chave de API pode fazer |

> "Usuário" sempre significa a conta. Para o papel de quem pede atendimento, use "Solicitante".

## Estrutura

| Termo | Código | Definição |
|---|---|---|
| Organização | `Organization` | Empresa atendida (ex.: um cliente da empresa de suporte) |
| Organização principal | `is_main` | A organização da empresa que opera o sistema (ex.: a empresa de suporte). Existe exatamente uma |
| Departamento | `Department` | Setor onde o usuário trabalha, dentro de uma organização (ex.: "Recepção") |
| Equipe | `Team` | Grupo técnico que recebe e atende chamados (ex.: "Infraestrutura") |
| Categoria | `Category` | Classificação principal do chamado (ex.: "Impressoras") |
| Subcategoria | `Category` com `parent_id` | Categoria que pertence a outra (ex.: "Sem impressão" dentro de "Impressoras"). A árvore não tem limite de níveis |

## Chamado

| Termo | Código | Definição |
|---|---|---|
| Chamado | `Ticket` | Registro de uma solicitação de atendimento |
| Registrante | `creator` | Quem registrou o chamado. Pode ser diferente do solicitante (abertura em nome de outra pessoa) |
| Responsável | `assignee` | Técnico atribuído ao chamado |
| Canal de origem | `channel` | Por onde o chamado entrou: `web`, `api`, `agent`, `monitoring` |
| Chamado privado | `is_private` | Chamado de conteúdo sensível, com visibilidade restrita |
| Chamado anterior | `previous_ticket_id` | Chamado fechado ao qual um novo chamado está vinculado, quando o problema volta a acontecer |
| Reabrir | `reopen` | Devolver um chamado Resolvido para Em atendimento, com um comentário obrigatório. Chamados Fechados não são reabertos |
| Confirmar solução | `close` | O solicitante aceita a solução e o chamado passa de Resolvido para Fechado |
| Fechamento automático | `auto_close` | Fechamento feito pelo sistema quando o chamado fica 3 dias úteis em Resolvido sem confirmação nem reabertura |
| Interação | `TicketComment` | Mensagem dentro do chamado |
| Comentário público | `visibility: public` | Interação visível ao solicitante |
| Nota interna | `visibility: internal` | Interação visível apenas à equipe técnica |
| Avaliação | `Rating` | Nota de 1 a 5, com comentário opcional, dada pelo solicitante ao atendimento. Opcional |
| Anexo | `Attachment` | Arquivo enviado em um chamado ou em uma interação. Herda a visibilidade de onde está |

### Status

| Termo | Código | Significado |
|---|---|---|
| Aberto | `open` | Registrado, aguardando atendimento |
| Em atendimento | `in_progress` | Um técnico está trabalhando no chamado |
| Pendente | `pending` | Aguardando resposta do solicitante. O relógio do SLA fica pausado |
| Resolvido | `resolved` | O técnico concluiu o atendimento |
| Fechado | `closed` | Encerrado definitivamente. Não pode ser reaberto |
| Cancelado | `canceled` | Encerrado sem atendimento |

### Prioridade

| Termo | Código |
|---|---|
| Baixa | `low` |
| Média | `medium` |
| Alta | `high` |
| Urgente | `urgent` |

## SLA e métricas

| Termo | Código | Definição |
|---|---|---|
| Política de SLA | `SlaPolicy` | Prazos de primeira resposta e de solução, por prioridade. Global ou por organização |
| Calendário | `BusinessCalendar` | Conjunto de horários de atendimento, com fuso horário (ex.: "24×7", "Comercial 5×2") |
| Horário útil | `BusinessCalendarHour` | Intervalo de atendimento em um dia da semana (ex.: segunda, 8h–12h). Só o horário útil conta para os prazos |
| Prazo de primeira resposta | `first_response_due_at` | Até quando um técnico deve responder pela primeira vez |
| Prazo de solução | `resolution_due_at` | Até quando o chamado deve ser resolvido |
| Prazo excedido | `breached` | Condição de um chamado cujo prazo passou. Não é um status |
| Pausa de SLA | — | Período em que o chamado está Pendente; não conta para os prazos |
| Padrão de fábrica | `default` | Valor original de uma configuração |
| Personalização | `override` | Valor definido pelo cliente, que substitui o padrão e pode ser removido |
| Tempo médio de primeira resposta | — | Média do tempo entre a abertura e a primeira resposta de um técnico |
| Tempo médio de solução | — | Média do tempo entre a abertura e a resolução, **descontadas as pausas**. Mede o desempenho da equipe técnica e é a base do SLA |
| Tempo médio corrido | — | Média do tempo total entre a abertura e a resolução, **incluindo as pausas**. Mede a experiência real do solicitante |
| Tempo médio em pendência | — | Média do tempo que os chamados passam em Pendente (corrido − solução). Indica quanto da demora vem da espera pelo solicitante |
| Taxa de confirmação | — | Percentual dos chamados fechados por confirmação do solicitante, e não automaticamente. Medida por organização e departamento, nunca por pessoa |
| Taxa de avaliação | — | Percentual dos chamados fechados que receberam avaliação. Medida por organização e departamento, nunca por pessoa |

> Comparar o tempo de solução com o tempo corrido aponta o gargalo: se a diferença é grande, a demora está na espera pelo solicitante; se é pequena, está no atendimento técnico.

## Dados

| Termo | Código | Definição |
|---|---|---|
| Arquivar | soft delete | Retirar de uso sem apagar. Para usuários, aparece como "desativar" |
| Auditoria | `activity_log` | Registro automático de alterações (quem, quando, o quê, canal) |
