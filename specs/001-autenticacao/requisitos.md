# 001 — Autenticação: Requisitos

| Status | Data | Aprovado por |
|---|---|---|
| Requisitos aprovados | 2026-10-12 | Dêvisson Souza |

## Contexto

Todo acesso ao sistema passa por este módulo (M1 da [visão do produto](../../docs/visao-do-produto.md)). Ele atende a dois tipos de cliente da mesma API (P5):

- **Pessoas**, que usam uma interface (web ou aplicativo) e se autenticam com e-mail e senha, mantendo uma **sessão** por cookie HttpOnly.
- **Integrações** (ex.: o Zabbix abrindo chamados por alerta, o agente do WhatsApp na fase 2), que usam **chaves de API** de uma **conta de integração**: um usuário próprio, sem senha, criado pelo Administrador.

Chaves de API pertencem a integrações, nunca a pessoas. Assim, o controle de quem acessa o sistema por fora da interface fica concentrado no Administrador, e trocar a senha ou desativar uma pessoa não derruba uma integração.

A autenticação diz **quem** está fazendo a requisição. O que essa pessoa ou integração **pode fazer** é decidido pelos papéis e permissões ([permissoes.md](../../docs/permissoes.md)).

O login precisa ser simples (P1): e-mail e senha, sem passos extras na fase 1.

## Histórias de usuário

- Como **usuário**, quero entrar com meu e-mail e senha para acessar meus chamados.
- Como **usuário**, quero continuar conectado no meu computador para não precisar entrar toda vez.
- Como **usuário**, quero sair do sistema para que ninguém use minha sessão num computador compartilhado.
- Como **usuário**, quero recuperar minha senha por e-mail quando esquecê-la, sem depender da TI.
- Como **usuário**, quero trocar minha senha quando desejar.
- Como **Administrador**, quero cadastrar uma conta de integração com papéis e canal de origem próprios para conectar uma ferramenta (ex.: Zabbix) com acesso limitado ao que ela precisa.
- Como **Administrador**, quero gerar, trocar e revogar as chaves de uma integração para controlar e cortar imediatamente seu acesso.
- Como **integração**, quero me autenticar com uma chave de API para usar a API sem depender da senha de uma pessoa.
- Como **cliente da API** (interface ou integração), quero consultar quem sou e quais permissões tenho para adaptar o que ofereço.

## Regras de negócio

### Login e sessão

| ID | Regra |
|---|---|
| RN-001-01 | O login é feito com e-mail e senha, apenas por pessoas. O e-mail é único no sistema e comparado sem diferenciar maiúsculas de minúsculas |
| RN-001-02 | Falhas de login respondem sempre com a mesma mensagem genérica, sem revelar se o e-mail existe, se a senha está errada ou se o usuário está desativado (P12) |
| RN-001-03 | Tentativas de login são limitadas por combinação de e-mail e IP: após 3 falhas em 1 minuto, novas tentativas são recusadas até o fim do bloqueio |
| RN-001-04 | A sessão expira após 60 minutos sem atividade. Com a opção "manter conectado", dura 7 dias. Os dois valores são configuráveis, com padrão de fábrica restaurável (P2) |
| RN-001-05 | Usuário desativado, pessoa ou conta de integração, perde o acesso imediatamente: não entra, e suas sessões e chaves deixam de funcionar |

### Senha

| ID | Regra |
|---|---|
| RN-001-06 | A senha é forte: no mínimo 8 caracteres, com pelo menos uma letra maiúscula, uma letra minúscula, um número e um caractere especial. Nenhuma verificação depende de serviço externo (P8, P9) |
| RN-001-07 | Senhas são armazenadas apenas como hash. Senhas, chaves de API e links de recuperação nunca aparecem em logs, respostas de erro ou auditoria (P12) |
| RN-001-08 | A recuperação de senha envia por e-mail um link de uso único, válido por 60 minutos. A resposta ao pedido é sempre a mesma, exista o e-mail ou não (P12) |
| RN-001-09 | Ao redefinir ou trocar a senha, todas as outras sessões da pessoa são encerradas |
| RN-001-10 | Trocar a própria senha exige informar a senha atual |

### Contas de integração e chaves de API

| ID | Regra |
|---|---|
| RN-001-11 | Conta de integração é um usuário do tipo integração. Tem nome, organização, papéis e canal de origem. Não tem senha nem e-mail de login, e nunca se autentica por sessão |
| RN-001-12 | Apenas quem tem a permissão `integrations.manage` (Administrador) cria, edita e desativa contas de integração, e gera e revoga suas chaves |
| RN-001-13 | Pessoas não têm chaves de API |
| RN-001-14 | A chave dá exatamente o acesso dos papéis da conta de integração, avaliado a cada requisição: se os papéis da conta mudam, o acesso da chave muda junto. Chaves não têm escopos próprios |
| RN-001-15 | O valor da chave é exibido uma única vez, na geração. Depois disso, só o nome, a data de criação, o último uso e a expiração ficam visíveis |
| RN-001-16 | A chave pode ter data de expiração opcional. Chave expirada é recusada |
| RN-001-17 | Uma conta de integração pode ter várias chaves ativas, para permitir a troca sem interromper a integração |
| RN-001-18 | Contas de integração e chaves são gerenciadas apenas com sessão. Uma chave não gerencia contas nem chaves, para que uma chave vazada não consiga se perpetuar |
| RN-001-19 | O canal de origem é definido pelo sistema a partir de quem está autenticado, nunca informado pelo cliente (P6): sessão de pessoa é `web`; chave de integração é o canal cadastrado na conta (`api`, `agent` ou `monitoring`) |

### Identidade e auditoria

| ID | Regra |
|---|---|
| RN-001-20 | Login, falha de login, logout, recuperação e troca de senha, criação, edição e desativação de conta de integração, geração e revogação de chave são registrados na auditoria com usuário, data, IP e canal (P11) |
| RN-001-21 | A consulta "quem sou eu" devolve os dados do usuário autenticado, seu tipo (pessoa ou integração), seus papéis e permissões e, para integrações, o canal de origem |

## Critérios de aceite

### Login e sessão

#### CA-001-01: Login com credenciais válidas
- **Dado** uma pessoa ativa com e-mail `ana@exemplo.com` e senha correta
- **Quando** ela faz login
- **Então** recebe uma sessão por cookie HttpOnly e acessa endpoints protegidos

#### CA-001-02: E-mail sem diferenciar maiúsculas
- **Dado** uma pessoa cadastrada como `ana@exemplo.com`
- **Quando** ela faz login com `Ana@Exemplo.COM` e a senha correta
- **Então** o login é aceito

#### CA-001-03: Falha de login com mensagem genérica
- **Dado** três tentativas: e-mail inexistente, senha errada e pessoa desativada com senha correta
- **Quando** cada uma é enviada
- **Então** as três recebem a mesma resposta `422` com a mesma mensagem, e nenhuma cria sessão

#### CA-001-04: Bloqueio por excesso de tentativas
- **Dado** 3 falhas de login para o mesmo e-mail e IP no último minuto
- **Quando** uma quarta tentativa é feita, mesmo com a senha correta
- **Então** ela recebe `429`, com a informação de quando tentar novamente

#### CA-001-05: Requisição sem autenticação
- **Dado** um cliente sem sessão nem chave de API
- **Quando** acessa um endpoint protegido
- **Então** recebe `401` no formato de erro padrão (RFC 9457)

#### CA-001-06: Expiração por inatividade
- **Dado** uma sessão sem "manter conectado", sem atividade há mais de 60 minutos
- **Quando** a pessoa faz uma requisição
- **Então** recebe `401`

#### CA-001-07: Manter conectado
- **Dado** um login feito com "manter conectado"
- **Quando** a pessoa volta após 3 dias sem atividade
- **Então** continua autenticada
- **E quando** volta após mais de 7 dias
- **Então** recebe `401`

#### CA-001-08: Logout
- **Dado** uma pessoa autenticada por sessão
- **Quando** ela faz logout
- **Então** a sessão é encerrada e a próxima requisição com o mesmo cookie recebe `401`

#### CA-001-09: Desativação corta o acesso na hora
- **Dado** uma pessoa com sessão ativa e uma conta de integração com chave válida
- **Quando** ambas são desativadas
- **Então** a próxima requisição pela sessão e pela chave recebe `401`

### Senha

#### CA-001-10: Pedido de recuperação de senha
- **Dado** uma pessoa ativa
- **Quando** ela pede a recuperação informando o e-mail
- **Então** um e-mail com link de redefinição entra na fila de envio, e a resposta é `200` com mensagem genérica

#### CA-001-11: Recuperação para e-mail inexistente
- **Dado** um e-mail que não existe no sistema
- **Quando** é pedida a recuperação
- **Então** a resposta é idêntica à do CA-001-10 e nenhum e-mail é enviado

#### CA-001-12: Redefinição com link válido
- **Dado** um link de recuperação gerado há menos de 60 minutos e ainda não usado
- **Quando** a pessoa define uma nova senha válida
- **Então** a senha é trocada, o link deixa de valer e as outras sessões da pessoa são encerradas

#### CA-001-13: Link expirado ou já usado
- **Dado** um link gerado há mais de 60 minutos, ou já usado
- **Quando** a pessoa tenta redefinir a senha
- **Então** recebe `422` e a senha não muda

#### CA-001-14: Senha fora da regra
- **Dado** uma pessoa redefinindo ou trocando a senha
- **Quando** informa uma senha que viola a regra: `Ab1!xyz` (7 caracteres), `abcd123!` (sem maiúscula), `ABCD123!` (sem minúscula), `Abcdefg!` (sem número) ou `Abcd1234` (sem caractere especial)
- **Então** cada uma recebe `422` com o erro no campo da senha, indicando o que falta
- **E** `Abcd123!` é aceita

#### CA-001-15: Troca da própria senha
- **Dado** uma pessoa autenticada por sessão, com outra sessão aberta em outro dispositivo
- **Quando** troca a senha informando a senha atual correta
- **Então** a senha é trocada, a sessão atual continua válida e a outra sessão é encerrada

#### CA-001-16: Troca de senha com senha atual errada
- **Dado** uma pessoa autenticada
- **Quando** tenta trocar a senha informando a senha atual errada
- **Então** recebe `422` e a senha não muda

### Contas de integração e chaves de API

#### CA-001-17: Cadastrar conta de integração
- **Dado** um Administrador autenticado por sessão
- **Quando** cadastra a conta "Zabbix produção", na organização principal, com um papel que só permite abrir chamados e canal `monitoring`
- **Então** recebe `201` com a conta criada, do tipo integração, sem senha e sem e-mail

#### CA-001-18: Canal inválido para integração
- **Dado** um Administrador cadastrando ou editando uma conta de integração
- **Quando** informa o canal `web`, ou um valor fora de `api`, `agent` e `monitoring`
- **Então** recebe `422` com o erro no campo do canal

#### CA-001-19: Gerar chave
- **Dado** a conta "Zabbix produção"
- **Quando** o Administrador gera a chave "servidor 01", sem expiração
- **Então** recebe `201` com o valor da chave, exibido somente nesta resposta

#### CA-001-20: Usar chave
- **Dado** a chave "servidor 01"
- **Quando** uma requisição é feita com `Authorization: Bearer <chave>`
- **Então** ela é autenticada como a conta "Zabbix produção", e o último uso da chave é atualizado

#### CA-001-21: A chave só pode o que a conta pode
- **Dado** a chave "servidor 01", cuja conta só tem permissão para abrir chamados
- **Quando** é usada para listar chamados
- **Então** recebe `403`

#### CA-001-22: Mudança de papel vale na hora
- **Dado** a chave "servidor 01"
- **Quando** o Administrador acrescenta à conta um papel que permite ver chamados
- **Então** a próxima requisição da chave para listar chamados é aceita, sem gerar nova chave

#### CA-001-23: Listar chaves de uma conta
- **Dado** uma conta de integração com duas chaves
- **Quando** o Administrador lista as chaves da conta
- **Então** vê nome, criação, último uso e expiração de cada uma, sem o valor

#### CA-001-24: Revogar chave
- **Dado** uma chave válida
- **Quando** o Administrador a revoga
- **Então** a chave passa a receber `401` imediatamente

#### CA-001-25: Chave expirada
- **Dado** uma chave cuja data de expiração já passou
- **Quando** é usada
- **Então** recebe `401`

#### CA-001-26: Troca de chave sem interrupção
- **Dado** a conta "Zabbix produção" com a chave "servidor 01"
- **Quando** o Administrador gera a chave "servidor 01 v2" e depois revoga a "servidor 01"
- **Então** a "servidor 01 v2" continua funcionando e a "servidor 01" recebe `401`

#### CA-001-27: Sem permissão para gerenciar integrações
- **Dado** uma pessoa com o papel Gestor e sem o papel Administrador
- **Quando** tenta cadastrar uma conta de integração, gerar, listar ou revogar chaves
- **Então** recebe `403`

#### CA-001-28: Chave não gerencia integrações
- **Dado** uma conta de integração com a permissão `integrations.manage`, autenticada por chave
- **Quando** tenta cadastrar contas, gerar, listar ou revogar chaves
- **Então** recebe `403`

#### CA-001-29: Pessoa não tem chave
- **Dado** um Administrador
- **Quando** tenta gerar uma chave para um usuário do tipo pessoa (inclusive para si mesmo)
- **Então** recebe `404`, pois a pessoa não é uma conta de integração, e nenhuma chave é criada

### Identidade e auditoria

#### CA-001-30: Quem sou eu (pessoa)
- **Dado** uma pessoa autenticada por sessão
- **Quando** consulta "quem sou eu"
- **Então** recebe seus dados (nome, e-mail, organização, departamento), o tipo `person`, seus papéis e permissões

#### CA-001-31: Quem sou eu (integração)
- **Dado** a chave "servidor 01"
- **Quando** consulta "quem sou eu"
- **Então** recebe os dados da conta "Zabbix produção", o tipo `integration`, o canal `monitoring`, seus papéis e permissões

#### CA-001-32: Eventos de autenticação na auditoria
- **Dado** um login bem-sucedido, uma falha de login, um logout, uma troca de senha, o cadastro de uma conta de integração, a geração e a revogação de uma chave
- **Quando** a auditoria é consultada
- **Então** cada evento aparece com usuário (quando identificado), data, IP e canal

#### CA-001-33: Segredos fora dos registros
- **Dado** os eventos do CA-001-32 e uma recuperação de senha
- **Quando** a auditoria e os logs da aplicação são inspecionados
- **Então** nenhum contém senha, valor de chave de API ou token de recuperação

## Fora de escopo

| Item | Onde fica |
|---|---|
| Cadastro, edição e desativação de pessoas | Spec 003 (Usuários) |
| Definição da senha inicial de uma pessoa nova (convite) | Spec 003 (Usuários), reaproveitando o mecanismo de recuperação desta spec |
| Papéis e permissões, inclusive os atribuídos a contas de integração | Spec 002 (Controle de acesso) |
| Gravação do canal de origem no chamado | Spec 006 (Chamados), aplicando a RN-001-19 |
| Conteúdo e layout dos e-mails | Spec 011 (Notificações por e-mail). Esta spec apenas dispara o envio em fila. Contas de integração não recebem e-mail |
| Consulta da auditoria | Spec 002 (`audit.view`) |
| Categoria de fábrica "Acessos > Integração / chave de API", para pedir integrações por chamado | Spec 005 (Catálogo) |
| Chaves de API para pessoas (ex.: scripts de técnicos) | Fora da fase 1 |
| Autenticação em dois fatores | Fase futura (relevante para vendas ao governo) |
| Login único (SSO, LDAP, Active Directory, OAuth) | Fase futura |
| Gerenciar as próprias sessões abertas (ver e encerrar dispositivos) | Fase futura |
| Limite de requisições da API em geral | Convenções da API §9; aplicado de forma transversal |

## Dúvidas em aberto

A spec só é aprovada depois que todas forem respondidas.

| # | Dúvida | Resposta |
|---|---|---|
| 1 | Tempo de sessão | 60 min de inatividade; 7 dias com "manter conectado". Configuráveis (RN-001-04) |
| 2 | Regra de senha | Senha forte: mínimo 8, com maiúscula, minúscula, número e caractere especial (RN-001-06) |
| 3 | Chave de API gerencia contas ou chaves? | Não, apenas com sessão (RN-001-18) |
| 4 | Quem gerencia chaves? | Apenas o Administrador, papel separado do Gestor (RN-001-12) |
| 5 | Chave de API deve ter expiração obrigatória? | Não, expiração opcional (RN-001-16) |
| 6 | A quem pertencem as chaves de API? | A contas de integração, criadas pelo Administrador, sem escopos próprios e com canal de origem fixo. Pessoas não têm chaves na fase 1. O pedido de integração é feito por chamado (RN-001-11 a RN-001-19) |
| 7 | Bloqueio de login | 3 falhas por minuto, por e-mail e IP (RN-001-03) |

## Histórico de alterações

| Data | Alteração | Motivo |
|---|---|---|
| 2026-10-12 | Rascunho inicial | — |
| 2026-10-12 | Requisitos aprovados | — |
| 2026-10-12 | CA-001-29: resposta muda de `422` para `404` | Proposto no design e aprovado: a rota de chaves é da conta de integração, e o id de uma pessoa não existe nesse endereço (convenções da API §5) |
| 2026-10-12 | Respostas às dúvidas 1 a 7. Chaves de API passam a pertencer a contas de integração; regras e critérios renumerados | Decisões do responsável pelo produto. A renumeração foi feita antes da aprovação, sem nenhuma referência aos identificadores antigos |
