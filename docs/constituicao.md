# Constituição do Projeto

> Princípios inegociáveis. Toda spec e toda implementação é avaliada por eles.
> Versão 1.0 — 2026-10-11.

## Produto

### P1. Abrir chamado deve ser simples
Diante de duas soluções, vence a que dá menos trabalho a quem pede atendimento. Facilitar a abertura de chamados aumenta a adesão.

### P2. Funciona de cara, e personalizar é reversível
Todo recurso nasce com um padrão útil. Personalizações são armazenadas separadas dos valores de fábrica e sempre podem ser restauradas ao padrão.

### P3. O básico é obrigatório, o resto é acessório
O núcleo funciona completo sem WhatsApp, IA ou integrações externas. Nada no núcleo depende deles.

### P4. Resposta rápida
No cenário de referência — 100 mil chamados, 10 mil usuários, 200 organizações, 50 usuários simultâneos, servidor de 2 vCPUs e 4 GB de RAM — o p95 do tempo de resposta do servidor fica em:

| Operação | Meta (p95) |
|---|---|
| Leitura simples | ≤ 200 ms |
| Listagem paginada | ≤ 300 ms |
| Escrita | ≤ 300 ms |
| Métricas do dashboard | ≤ 1 s |

Regras que sustentam a meta:
- Listagens são sempre paginadas, com no máximo 100 itens por página.
- Consultas N+1 são proibidas.
- Trabalho pesado (ex.: envio de e-mail) roda em filas, nunca durante a requisição.

Verificação: testes automatizados de quantidade de consultas no CI e teste de carga antes de cada versão. Os números podem ser revisados após o primeiro teste de carga.

## Arquitetura

### P5. API-first
Toda funcionalidade existe na API. Nenhuma regra vive apenas em uma interface, e qualquer interface é apenas mais um cliente da API.

### P6. O servidor decide, o cliente sugere
Validação, regras de negócio e autorização acontecem sempre no backend. Nenhum cliente da API é confiável, seja um aplicativo, um agente de IA ou uma integração.

### P7. Autorização por permissão, não por papel
O código verifica permissões granulares (ex.: "pode atribuir chamado"), nunca papéis (ex.: "é técnico?"). Papéis apenas agrupam permissões.

### P8. O núcleo não depende de terceiros
Integrações externas ficam atrás de interfaces. A falha de um serviço externo (e-mail, ferramentas de gestão, IA) nunca impede abrir ou atender um chamado.

### P9. Auto-hospedado de verdade
O sistema roda inteiro na infraestrutura do cliente, sem precisar se comunicar com servidores do fornecedor para funcionar.

## Dados

### P10. Nada com histórico é apagado
Registros com vínculos usam soft delete. A exclusão definitiva só é permitida para itens sem vínculo.

### P11. Toda mudança deixa rastro
Toda alteração relevante registra quem fez, quando, o que mudou e por qual canal (web, API, agente, integração).

### P12. Privacidade por padrão
Segregação de dados por organização, suporte a chamados privados, coleta do mínimo de dados pessoais e nenhum dado sensível (senhas, chaves, conteúdo privado) em logs.

## Processo e qualidade

### P13. Spec antes de código
Nenhuma funcionalidade é implementada sem spec aprovada. Se a implementação precisar divergir, a spec é atualizada antes.

### P14. Todo critério de aceite vira teste
Testes automatizados fazem parte da entrega. Nenhum teste depende de serviço externo real.

### P15. Simplicidade técnica
Nenhuma dependência ou abstração sem necessidade presente. O código deve ser fácil de ler por quem chega ao repositório.

## Convenção de idioma

| Item | Idioma |
|---|---|
| Código (classes, métodos, variáveis, tabelas, rotas) | Inglês |
| Comentários no código | Português |
| Specs e documentação | Português |
| Mensagens de commit | Português |

## Alterações

Esta constituição só muda com aprovação explícita do responsável pelo produto. Cada alteração incrementa a versão e registra a data.
