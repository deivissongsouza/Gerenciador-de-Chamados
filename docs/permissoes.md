# Matriz de Permissões — Fase 1

> Quem pode fazer o quê. Contrato que todas as specs seguem.
> Princípios em [constituicao.md](constituicao.md), termos em [glossario.md](glossario.md). Última revisão: 2026-10-12.

## Estrutura

- O código verifica **permissões**, nunca papéis (P7). Papéis são apenas conjuntos de permissões configuráveis.
- Nome das permissões: `recurso.ação`, em inglês (ex.: `tickets.assign`).
- Permissões que dependem de **alcance** levam o alcance no nome:

| Alcance | Sufixo | Significado |
|---|---|---|
| Próprios | `.own` | Apenas registros do próprio usuário (ex.: chamados que solicitou) |
| Organização | `.organization` | Registros da organização do usuário |
| Vinculados | `.linked` | Organizações vinculadas ao usuário ou às suas equipes |
| Todos | `.all` | Todos os registros |

A visibilidade configurável do técnico é definida por qual permissão o papel Técnico recebe: `.linked` ou `.all`.

## Papéis de fábrica

| Papel | Código | Resumo |
|---|---|---|
| Solicitante | `Requester` | Abre e acompanha chamados |
| Técnico | `Technician` | Atende chamados |
| Gestor da Organização | `OrganizationManager` | Acompanha chamados e indicadores da própria organização, e também abre e comenta chamados |
| Gestor | `Manager` | Administra o sistema e acompanha todos os indicadores |

## Matriz

**S** = Solicitante · **T** = Técnico · **GO** = Gestor da Organização · **G** = Gestor

### Chamados

| Ação | Permissão | S | T | GO | G |
|---|---|---|---|---|---|
| Abrir chamado | `tickets.create` | ✔ | ✔ | ✔ | ✔ |
| Abrir em nome de outra pessoa | `tickets.create-on-behalf` | | ✔ | | ✔ |
| Ver chamados | `tickets.view.*` | organização¹ | vinculados ou todos² | organização | todos |
| Editar dados do chamado | `tickets.update` | | ✔ | | ✔ |
| Atribuir | `tickets.assign` | | ✔ | | ✔ |
| Transições de atendimento | `tickets.transition` | | ✔ | | ✔ |
| Reabrir chamado resolvido | `tickets.reopen.*` | próprios | ✔ | | ✔ |
| Confirmar solução (fechar) | `tickets.close.*` | próprios | | | ✔ |
| Cancelar | `tickets.cancel.*` | próprios, só em Aberto | ✔ | | ✔ |
| Marcar como privado | `tickets.set-private.*` | próprios | ✔ | | ✔ |
| Arquivar e restaurar | `tickets.archive` | | | | ✔ |

¹ Exceto chamados privados de outras pessoas. ² Configurável.

O solicitante **não edita** título nem descrição após a abertura; complementa por comentários, para não alterar o que o técnico já leu.

### Interações e anexos

| Ação | Permissão | S | T | GO | G |
|---|---|---|---|---|---|
| Comentário público | `comments.create-public` | nos chamados que vê | ✔ | nos chamados que vê | ✔ |
| Criar e ver notas internas | `comments.internal` | | ✔ | | ✔ |
| Enviar anexo | `attachments.upload` | ✔ | ✔ | ✔ | ✔ |

Anexos não têm permissão de leitura própria: quem vê o chamado ou a interação vê o anexo.

### Avaliação e dashboard

| Ação | Permissão | S | T | GO | G |
|---|---|---|---|---|---|
| Avaliar atendimento | `ratings.create.own` | próprios⁴ | | | |
| Ver dashboard | `dashboard.view.*` | | vinculados ou todos² | organização | todos |

⁴ Uma vez por chamado, enquanto ele está Resolvido ou até 7 dias após o fechamento. Opcional.

### Administração

| Ação | Permissão | S | T | GO | G |
|---|---|---|---|---|---|
| Ver usuários | `users.view.*` | | vinculados ou todos³ | organização | todos |
| Criar, editar, desativar usuários | `users.manage` | | | | ✔ |
| Gerenciar papéis e permissões | `roles.manage` | | | | ✔ |
| Gerenciar organizações, departamentos e equipes | `organizations.manage` | | | | ✔ |
| Gerenciar categorias | `categories.manage` | | | | ✔ |
| Gerenciar SLA e calendários | `sla.manage` | | | | ✔ |
| Gerenciar configurações | `settings.manage` | | | | ✔ |
| Ver auditoria | `audit.view` | | | | ✔ |
| Usar chave de API | `api.access` | | | | |

³ Necessário para abrir chamado em nome de outra pessoa.

**Chave de API:** nenhum papel recebe `api.access` por padrão. O Gestor concede a usuários específicos (ex.: o usuário de um agente ou integração).

**Leituras básicas** (categorias, prioridades, status) são liberadas a qualquer usuário autenticado, pois são necessárias para abrir um chamado.

## Transições de status

| De | Para | Ação | Quem pode |
|---|---|---|---|
| Aberto | Em atendimento | Atender | T, G |
| Em atendimento | Pendente | Pendenciar | T, G |
| Pendente | Em atendimento | Retomar | T, G |
| Pendente | Em atendimento | Comentário do solicitante | Automático |
| Em atendimento | Resolvido | Resolver | T, G |
| Resolvido | Em atendimento | Reabrir (exige comentário) | S (próprios), T, G |
| Resolvido | Fechado | Confirmar solução | S (próprios), G |
| Resolvido | Fechado | Fechamento automático após 3 dias úteis | Sistema |
| Aberto | Cancelado | Cancelar | S (próprios), T, G |
| Em atendimento, Pendente | Cancelado | Cancelar | T, G |

Chamado **Fechado não é reaberto**: abre-se um novo chamado, vinculado ao anterior (`previous_ticket_id`).

## Regras gerais

| Regra | Detalhe |
|---|---|
| Recurso invisível | Quem não pode ver um recurso recebe `404`, nunca `403` (P12) |
| Chave de API | Acesso efetivo = interseção entre os escopos da chave e as permissões do usuário |
| Visibilidade antes da consulta | Filtros de alcance são aplicados antes de filtros, ordenação e paginação |
| Chamado privado | Visível ao solicitante, ao registrante, ao responsável, aos membros da equipe atribuída e aos Gestores. O Gestor da Organização só o vê se for o solicitante ou o registrante. Prevalece sobre `tickets.view.organization` e `tickets.view.linked` |
