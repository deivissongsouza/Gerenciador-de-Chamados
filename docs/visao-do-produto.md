# Visão do Produto

> Documento vivo. Registra o entendimento do negócio e serve de base para todas as specs.
> Última revisão: 2026-10-12.

## 1. O problema

Pequenos incidentes de tecnologia (um sistema fora do ar, uma impressora parada) viram tropeços na operação das empresas. Hoje eles são tratados assim:

- **Pelo usuário:** liga para a TI ou manda mensagem num grupo de WhatsApp, porque abrir chamado é mais difícil que reclamar.
- **Pelo técnico:** recebe demandas espalhadas em telefone, grupos e conversas, sem registro, prioridade ou histórico.
- **Pela gestão e diretoria:** não tem visão do que acontece na operação.

As ferramentas de help desk existentes não resolvem porque têm interfaces pouco amigáveis e lentas, e configuração longa e técnica. O resultado é implementação cara e demorada, e **baixa adesão**.

## 2. Filosofia

> Abrir um chamado não deve ser complexo para o usuário, atender não deve ser árduo para o técnico, e entender a operação não deve ser confuso para a gestão.

Princípios de produto:

1. **Facilitar a abertura de chamados aumenta a adesão.** Toda decisão de produto é avaliada por esse critério.
2. **Funciona de cara.** As configurações padrão bastam para começar. Personalizar é opcional e sempre reversível ("restaurar padrão").
3. **O básico é obrigatório, o resto é acessório.** O sistema é um help desk completo por si só. Agentes, aplicativos e automações são complementos.
4. **API-first.** Tudo o que o sistema faz, a API faz. Nenhum recurso existe só na interface.
5. **O cliente é dono dos seus dados e da sua infraestrutura.** Sem dependência do fornecedor para funcionar.

## 3. Diferenciais

| Diferencial | O que significa |
|---|---|
| **Auto-hospedado** | Roda na infraestrutura do cliente. Ele controla dados e servidor, e não fica refém de licença ou fornecedor. |
| **UX amigável** | Pessoas não técnicas usam e entendem o que estão fazendo. |
| **Resposta rápida e previsibilidade** | Integra-se a ferramentas de monitoramento já existentes (Zabbix, Grafana…) via API, para que alertas e relatórios ajudem a detectar e mitigar falhas. O monitoramento **não** fica dentro do sistema. |
| **Aberto a extensões** | Por ser API-first, qualquer empresa pode criar seus próprios agentes, apps e automações. |

## 4. Pessoas e papéis

| Papel | Quem é | O que faz |
|---|---|---|
| **Solicitante** | Quem precisa de atendimento | Abre e acompanha chamados, interage com o técnico |
| **Técnico** | Quem atende | Recebe, assume, resolve chamados e registra notas internas |
| **Gestor de Departamento** | Gestor de um setor que pede atendimento (ex.: Almoxarifado) | Além do que o Solicitante faz, acompanha chamados e indicadores do próprio departamento |
| **Gestor da Organização** | Gerente do lado do cliente | Além do que o Solicitante faz, acompanha chamados e indicadores da própria organização |
| **Gestor** | Quem coordena o atendimento e acompanha o negócio | Vê todos os chamados e indicadores, coordena a equipe técnica. Não administra o sistema |
| **Administrador** | Quem configura o sistema | Cadastra usuários, papéis, organizações, catálogo, SLA e configurações; consulta a auditoria; gerencia chaves de API |

As permissões são **granulares por baixo dos papéis**, e os **papéis são cumulativos**. Administrar é uma função, não um cargo: o gestor da TI costuma ter os papéis Gestor e Administrador; o gestor do Almoxarifado, Solicitante e Gestor de Departamento.

### Foco da fase 1: TI

O produto é posicionado como **service desk de TI**: sistemas fora do ar, periféricos, instalações, acessos. É onde estão os diferenciais (monitoramento, auto-hospedagem). O exemplo de referência é uma empresa com TI interna, em que os demais setores (Almoxarifado, Recepção, Faturamento) são **clientes** da TI.

O núcleo, porém, é genérico no mecanismo (equipes, categorias, SLA), e não fecha a porta para atender outras áreas no futuro (manutenção predial, RH, rotinas administrativas), o chamado *Enterprise Service Management*.

## 5. Modelo de organização

- **Uma instalação atende uma empresa. Não é multi-tenant.**
- Dentro da instalação existem **organizações** (ex.: os clientes de uma empresa de suporte). Usuários pertencem a uma organização, e a organização segrega a visibilidade dos chamados.
- **Departamento:** onde o usuário trabalha dentro de uma organização (ex.: "Recepção" do hospital). O departamento do solicitante é gravado no chamado na abertura.
- **Equipe:** grupo técnico que recebe e atende chamados (ex.: "Infraestrutura", "Sistemas").

### Regras de visibilidade

| Quem | Vê |
|---|---|
| **Solicitante** | Os chamados da própria organização, **exceto os privados** de outras pessoas. O Administrador pode restringir ao departamento ou aos próprios chamados |
| **Gestor de Departamento** | Chamados e indicadores do departamento |
| **Gestor da Organização** | Chamados e indicadores da organização |
| **Técnico** | **Configurável:** todos os chamados, ou apenas os das organizações/equipes às quais está vinculado |
| **Gestor** | Todos os chamados e indicadores |
| **Administrador** | Configurações, cadastros e auditoria. Não vê chamados por este papel |

**Chamado privado:** marcado como sensível (ex.: um gestor pede a criação ou remoção de um usuário). Visível apenas para o solicitante, o registrante, o responsável, os membros da equipe atribuída e os Gestores. O Gestor de Departamento e o Gestor da Organização só o veem se forem o solicitante ou o registrante.

## 6. Chamados

- **Fluxo padrão** de status e prioridades, pronto para uso. Fluxos personalizáveis ficam para uma fase posterior.

  ```
  Aberto → Em atendimento ⇄ Pendente (aguardando usuário) → Resolvido → Fechado
                        ↘ Cancelado                         ↺ Reaberto volta a Em atendimento
  ```

  Chamado Fechado não é reaberto: abre-se um novo, vinculado ao anterior. Transições e permissões em [permissoes.md](permissoes.md).

- **Prazo excedido** é uma condição de SLA, não um status: um chamado pode estar "Em atendimento" e com o prazo excedido ao mesmo tempo.
- **Pendente:** quando o solicitante comenta um chamado Pendente, o chamado volta automaticamente para Em atendimento.
- **Reabertura:** reabrir um chamado Resolvido exige um comentário explicando o motivo.
- **Fechamento:** o solicitante pode confirmar a solução, e o chamado fecha na hora. Sem confirmação nem reabertura, o chamado fecha automaticamente após **3 dias úteis** em Resolvido (horário útil da organização do chamado). O prazo é configurável e pode ser restaurado ao padrão. O chamado registra a forma de fechamento (confirmado ou automático).
- **Engajamento:** taxas de confirmação e de avaliação são medidas **por organização e departamento, nunca por pessoa**. Servem para orientar, não para punir: o solicitante nunca é pontuado negativamente (P1, P12).
- **Avaliação:** nota de **1 a 5** com comentário opcional, dada apenas pelo solicitante, uma vez por chamado, enquanto o chamado está Resolvido ou até **7 dias** após o fechamento. Nunca é obrigatória para fechar (P1).
- **Remoção:** registros com histórico (usuários, chamados) são desativados ou arquivados (soft delete), nunca apagados.
- **Interações:** comentários públicos (o solicitante vê) e notas internas (só a equipe técnica vê).
- **Abertura em nome de outra pessoa:** um usuário autorizado (incluindo integrações e agentes) pode abrir chamado em nome de outro. O sistema registra o solicitante, quem registrou e o **canal de origem** (web, API, agente, monitoramento), definido pelo próprio sistema a partir de quem está autenticado.
- **SLA:** prazos padrão por prioridade, que o cliente pode sobrescrever globalmente ou **por organização** (ex.: contratos diferentes por cliente). Os valores de fábrica ficam preservados separados dos personalizados, permitindo **restaurar o padrão** a qualquer momento.
- **Horário útil:** cada organização usa um calendário (ex.: 24×7 ou comercial 5×2, cada um com seu fuso horário). Os prazos contam apenas o horário útil da organização do chamado.
- **Anexos:** chamados e interações aceitam anexos (prints, fotos), que herdam a visibilidade de onde estão.
- **E-mail:** somente para notificações. Não abre nem responde chamados.

## 7. Produto e modelo de negócio

```
┌────────────────────────────────┐        ┌───────────────────────────────┐
│ NÚCLEO (auto-hospedado)        │  API   │ HUB DE AGENTES (serviço pago) │
│ servidor do cliente            │◄──────►│ servidor do fornecedor        │
│ chamados, usuários, SLA,       │        │ agente de IA + WhatsApp,      │
│ notificações, relatórios, API  │        │ chaves do cliente (BYOK),     │
└────────────────────────────────┘        │ painel de consumo             │
               ▲                          └───────────────────────────────┘
               │ API / MCP
   Agentes, apps e automações do próprio cliente;
   monitoramento (Zabbix, Grafana…) abrindo chamados por eventos
```

- **Núcleo:** auto-hospedado, completo por si só.
- **Recursos premium:** a definir. Diretriz: o que garante a **adoção** (abrir e acompanhar chamados) nunca é pago.
- **Hub de agentes:** serviço contratado à parte. O agente acessa o núcleo pela API com uma **conta de integração** própria e permissões limitadas, criada pelo Administrador do cliente. O cliente fornece as próprias chaves de IA e acompanha o consumo. Futuramente, LLM própria hospedada pelo fornecedor.
- **Isolamento total de dados:** quem não quiser o hub constrói o próprio agente sobre a API.
- **Serviços sob contrato:** desenvolvimento de agentes personalizados (ex.: governo).
- **Licença:** a definir.

## 8. Roadmap

O primeiro objetivo é um **projeto de apresentação** sólido. O produto evolui por fases até estar pronto para venda, inclusive ao governo (compliance).

| Fase | Escopo |
|---|---|
| **1. Núcleo (MVP de apresentação)** | Sistema de chamados enxuto e API-first. Módulos detalhados abaixo |
| **2. Agente e WhatsApp** | Hub de agentes, integração com WhatsApp, servidor MCP |
| _A ordenar_ | Integrações com monitoramento · Integrações com Jira, ClickUp e Monday · Gestão de projetos · Fluxos personalizáveis · Ativos e equipamentos · Recursos premium · Gamificação (apenas reconhecimento positivo, nunca punitiva ao solicitante, de preferência voltada à equipe técnica) · Atendimento a outras áreas (manutenção predial, RH, administrativo): aprovações, campos personalizados por categoria, chamados recorrentes, cadastro de locais, gestor por equipe |

### Fase 1: módulos

| # | Módulo | Escopo |
|---|---|---|
| M1 | **Autenticação** | Sessão por cookie HttpOnly para pessoas. Contas de integração com chaves de API, gerenciadas pelo Administrador. Logout, revogação de chaves, troca e recuperação de senha |
| M2 | **Usuários** | Cadastrar, editar, listar, visualizar e desativar (soft delete) |
| M3 | **Controle de acesso** | Papéis (Solicitante, Técnico, Gestor) compostos por permissões granulares. Criar, listar, editar, ver, atribuir e remover |
| M4 | **Estrutura organizacional** | Organizações, departamentos e equipes |
| M5 | **Catálogo** | Categorias e subcategorias de chamados |
| M6 | **Chamados** | Criar, listar, editar, ver, atribuir e arquivar. Fluxo padrão de status, prioridade, chamados privados, abertura em nome de outra pessoa, canal de origem |
| M7 | **Interações** | Comentários públicos e notas internas |
| M12 | **Anexos** | Anexos em chamados e interações, com download autenticado e visibilidade herdada |
| M8 | **SLA** | Prazos por prioridade (globais e por organização), override e restaurar padrão, calendários de horário útil por organização |
| M9 | **Avaliação** | Nota do atendimento |
| M10 | **Dashboard** | Endpoints de métricas: chamados por status, chamados com prazo excedido, tempo médio de primeira resposta, tempo médio de solução (sem pausas), tempo médio corrido (com pausas), tempo médio em pendência, SLA cumprido/excedido, média das notas, taxa de confirmação e taxa de avaliação (por organização e departamento) |
| M11 | **Notificações por e-mail** | Eventos do chamado (aberto, atribuído, respondido, resolvido…) e recuperação de senha |
| M13 | **Configurações** | Consultar, personalizar e restaurar ao padrão as configurações gerais (ex.: tempo de sessão, prazo do fechamento automático, visibilidade do técnico) |
| — | **Transversal** | Soft delete, histórico de alterações, documentação da API |

### Fora do escopo da fase 1
WhatsApp, agente de IA, MCP, integrações externas, gestão de projetos, fluxos personalizáveis, ativos, feriados nos calendários de horário útil e frontend (a ser discutido em momento próprio).

## 9. Questões em aberto

| # | Questão | Quando decidir |
|---|---|---|
| Q1 | Licença do núcleo e dos recursos premium | Após fechar o negócio |
| Q2 | Frontend | Quando o responsável pelo produto decidir |
| Q3 | Como o hub alcança núcleos atrás de firewall | Fase 2 |
| Q4 | Relação entre gestão de projetos própria e Jira/ClickUp/Monday (fonte da verdade, chamado virando tarefa) | Antes da fase de projetos |
| Q5 | Quais recursos são premium | Antes da primeira venda |

### Questões resolvidas

| # | Questão | Decisão (2026-10-12) |
|---|---|---|
| Q6 | Regra exata de visibilidade dos chamados privados | Solicitante, registrante, responsável, equipe atribuída e Gestores (§5) |
| Q7 | Escala da nota de avaliação | 1 a 5, com comentário opcional (§6) |
| Q8 | Quem avalia, quando, e se a avaliação é obrigatória para fechar | Só o solicitante, enquanto Resolvido ou até 7 dias após fechar; nunca obrigatória (§6) |
| Q9 | Fechamento: confirmação do usuário ou automático | Confirmação do solicitante ou automático após 3 dias úteis em Resolvido (§6) |

Referência usada nas decisões: o comportamento do GLPI (aprovação ou recusa da solução, fechamento automático por prazo, pesquisa de satisfação opcional, retomada automática de pendentes).
