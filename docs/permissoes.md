# Matriz de Permissões — Fase 1

> Quem pode fazer o quê. Contrato que todas as specs seguem.
> Princípios em [constituicao.md](constituicao.md), termos em [glossario.md](glossario.md). Última revisão: 2026-10-12.

## Estrutura

- O código verifica **permissões**, nunca papéis (P7). Papéis são apenas conjuntos de permissões configuráveis.
- **Papéis são cumulativos:** um usuário pode ter vários, e seu acesso é a soma deles (ex.: o Gestor da TI costuma ter Gestor + Administrador).
- Nome das permissões: `recurso.ação`, em inglês (ex.: `tickets.assign`).
- Permissões que dependem de **alcance** levam o alcance no nome:

| Alcance | Sufixo | Significado |
|---|---|---|
| Próprios | `.own` | Apenas registros do próprio usuário (ex.: chamados que solicitou) |
| Departamento | `.department` | Registros do departamento do usuário. Para chamados, o departamento do solicitante gravado na abertura |
| Organização | `.organization` | Registros da organização do usuário |
| Vinculados | `.linked` | Organizações vinculadas ao usuário ou às suas equipes |
| Todos | `.all` | Todos os registros |

A visibilidade configurável do técnico é definida por qual permissão o papel Técnico recebe: `.linked` ou `.all`.

## Papéis de fábrica

| Papel | Código | Resumo |
|---|---|---|
| Solicitante | `Requester` | Abre e acompanha chamados |
| Gestor de Departamento | `DepartmentManager` | Somado ao Solicitante: acompanha os chamados e indicadores do próprio departamento (ex.: o gestor do Almoxarifado) |
| Gestor da Organização | `OrganizationManager` | Somado ao Solicitante: acompanha os chamados e indicadores da própria organização (ex.: o gerente do hospital cliente) |
| Técnico | `Technician` | Atende chamados |
| Gestor | `Manager` | Coordena o atendimento e acompanha todos os chamados e indicadores. Não administra o sistema |
| Administrador | `Admin` | Configura o sistema: usuários, papéis, estrutura, catálogo, SLA, configurações, auditoria e chaves de API. Não vê chamados por este papel |

O Administrador é uma **função**, não um cargo: pode ser o gestor da TI, um analista designado ou ambos.

## Matriz

**S** = Solicitante · **GD** = Gestor de Departamento · **GO** = Gestor da Organização · **T** = Técnico · **G** = Gestor · **A** = Administrador

As colunas GD e GO mostram apenas o que esses papéis **acrescentam** ao Solicitante, que todo usuário pessoa recebe.

### Chamados

| Ação | Permissão | S | GD | GO | T | G | A |
|---|---|---|---|---|---|---|---|
| Abrir chamado | `tickets.create` | ✔ | | | ✔ | ✔ | |
| Abrir em nome de outra pessoa | `tickets.create-on-behalf` | | | | ✔ | ✔ | |
| Ver chamados | `tickets.view.*` | organização¹ ⁵ | departamento¹ | organização¹ | vinculados ou todos² | todos | |
| Editar dados do chamado | `tickets.update` | | | | ✔ | ✔ | |
| Atribuir | `tickets.assign` | | | | ✔ | ✔ | |
| Transições de atendimento | `tickets.transition` | | | | ✔ | ✔ | |
| Reabrir chamado resolvido | `tickets.reopen.*` | próprios | | | ✔ | ✔ | |
| Confirmar solução (fechar) | `tickets.close.*` | próprios | | | | ✔ | |
| Cancelar | `tickets.cancel.*` | próprios, só em Aberto | | | ✔ | ✔ | |
| Marcar como privado | `tickets.set-private.*` | próprios | | | ✔ | ✔ | |
| Arquivar e restaurar | `tickets.archive` | | | | | | ✔ |

¹ Exceto chamados privados de outras pessoas. ² Configurável. ⁵ Em organizações grandes (ex.: TI interna de um hospital), o Administrador pode restringir o Solicitante a `tickets.view.department` ou `tickets.view.own` editando o papel, sem mudar código.

Com o padrão de fábrica (Solicitante vê a organização), o alcance de chamados do GD e do GO não acrescenta nada; o que os diferencia são os **indicadores** (dashboard) e a lista de usuários. O alcance de chamados passa a fazer diferença quando o Solicitante é restringido (⁵).

O solicitante **não edita** título nem descrição após a abertura; complementa por comentários, para não alterar o que o técnico já leu.

### Interações e anexos

| Ação | Permissão | S | GD | GO | T | G | A |
|---|---|---|---|---|---|---|---|
| Comentário público | `comments.create-public` | nos chamados que vê | | | ✔ | ✔ | |
| Criar e ver notas internas | `comments.internal` | | | | ✔ | ✔ | |
| Enviar anexo | `attachments.upload` | ✔ | | | ✔ | ✔ | |

Anexos não têm permissão de leitura própria: quem vê o chamado ou a interação vê o anexo.

### Avaliação e dashboard

| Ação | Permissão | S | GD | GO | T | G | A |
|---|---|---|---|---|---|---|---|
| Avaliar atendimento | `ratings.create.own` | próprios⁴ | | | | | |
| Ver dashboard | `dashboard.view.*` | | departamento | organização | vinculados ou todos² | todos | |

⁴ Uma vez por chamado, enquanto ele está Resolvido ou até 7 dias após o fechamento. Opcional.

### Administração

| Ação | Permissão | S | GD | GO | T | G | A |
|---|---|---|---|---|---|---|---|
| Ver usuários | `users.view.*` | | departamento | organização | vinculados ou todos³ | todos | todos |
| Criar, editar, desativar usuários | `users.manage` | | | | | | ✔ |
| Gerenciar papéis e permissões | `roles.manage` | | | | | | ✔ |
| Gerenciar organizações, departamentos e equipes | `organizations.manage` | | | | | | ✔ |
| Gerenciar categorias | `categories.manage` | | | | | | ✔ |
| Gerenciar SLA e calendários | `sla.manage` | | | | | | ✔ |
| Gerenciar configurações | `settings.manage` | | | | | | ✔ |
| Ver auditoria | `audit.view` | | | | | | ✔ |
| Gerenciar contas de integração e chaves de API | `integrations.manage` | | | | | | ✔ |

³ Necessário para abrir chamado em nome de outra pessoa.

**Chaves de API:** pertencem apenas a **contas de integração** (ex.: "Zabbix produção"), nunca a pessoas. A conta recebe papéis como qualquer usuário, e a chave pode exatamente o que esses papéis permitem. Detalhes na [spec 001](../specs/001-autenticacao/requisitos.md).

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
| Chave de API | Acesso efetivo = permissões atuais da conta de integração dona da chave. Chaves não têm escopos próprios |
| Visibilidade antes da consulta | Filtros de alcance são aplicados antes de filtros, ordenação e paginação |
| Chamado privado | Visível ao solicitante, ao registrante, ao responsável, aos membros da equipe atribuída e aos Gestores. O Gestor de Departamento e o Gestor da Organização só o veem se forem o solicitante ou o registrante. Prevalece sobre `tickets.view.department`, `tickets.view.organization` e `tickets.view.linked` |
