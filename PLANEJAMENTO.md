# Planejamento técnico: Sistema de Atendimento Imobiliário Self-Hosted

**Projeto:** atendimento inteligente e CRM imobiliário para Letícia
**Versão do documento:** 1.1 — 24/09/2026
**Base:** revisão da versão 1.0 (22/09/2026)

Este documento substitui [Planejamento_imobiliario_atualizado.txt](Planejamento_imobiliario_atualizado.txt) como referência de implantação. A versão anterior permanece apenas como histórico.

---

## 1. Objetivo do projeto

Desenvolver e implantar um sistema próprio, hospedado em uma VPS, que automatize o atendimento comercial de Letícia pelo WhatsApp e organize seus clientes em um CRM.

Em produção, o sistema deverá usar o número de WhatsApp que ela já usa profissionalmente, preservar o atendimento manual e executar automaticamente as tarefas repetitivas da rotina. **A homologação usará um número de teste.** O número profissional só entra depois dos go/no-go da Fase 2.

O resultado esperado é que Letícia deixe de gastar tempo com apresentações, envio de materiais, respostas padronizadas e organização manual de contatos, concentrando o trabalho nos clientes que precisam de atendimento pessoal.

### Premissas estabelecidas

- Hospedagem própria em VPS
- Ferramentas open source
- Evolution API (Node) como integração com WhatsApp — não Evolution Go
- Evo CRM Community `v1.1.0` como base do CRM, usando a arquitetura oficial
- Python próprio somente como extensão, depois de homologar o que o CRM já oferece
- API da OpenAI somente na Fase 4, para mensagens fora do roteiro
- Atendimento automático e intervenção humana
- Implementação incremental
- Sem n8n, Meta Cloud API ou CRM comercial de assinatura neste momento

O sistema não será um SaaS comercial. Inicialmente, será uma instalação dedicada à Letícia, com possibilidade de expansão futura.

---

## 2. Arquitetura proposta

A arquitetura parte da stack oficial do Evo CRM Community. Não construiremos um FastAPI paralelo no caminho crítico do atendimento enquanto os componentes nativos não tiverem sido homologados.

A Evolution API é um serviço companheiro, com versionamento próprio. Ela não faz parte do tag `v1.1.0` do CRM.

### 2.1. Visão dos serviços

```text
WhatsApp (número de teste na Fase 2; número da Letícia só depois do go/no-go)
        │
        ▼
Evolution API (Node, versão pinada)
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  Evo CRM Community v1.1.0                                   │
│                                                             │
│  evo-frontend  →  evo-auth + sidekiq                        │
│               →  evo-crm + sidekiq  →  evo-processor        │
│                                     →  evo-bot-runtime      │
│                                     →  evo-core             │
│                                     →  evo-flow             │
│                                                             │
│  Persistência: PostgreSQL (pgvector) + Redis                │
│                + RabbitMQ + ClickHouse                      │
└─────────────────────────────────────────────────────────────┘
        │
        ▼  somente se o nativo for insuficiente
Camada Python própria (catálogo imobiliário e, se preciso, mute de takeover)
        │
        ▼  Fase 4
OpenAI API (interpretação fora do roteiro; nunca inventa preço)
```

O CRM continua sendo a interface principal de trabalho da Letícia.

A camada Python **não** substitui o `evo-processor`, o `evo-bot-runtime` nem o `evo-flow`. Ela só entra para regras imobiliárias que o nativo não cobrir — em especial o catálogo de empreendimentos, unidades e valores.

### 2.2. O que já é nativo e não deve ser reimplementado

| Necessidade do projeto | Recurso nativo |
|---|---|
| Conversas, contatos, histórico | `evo-crm` |
| Login e papéis | `evo-auth` |
| Funil comercial | Pipeline kanban + estágios + regras por estágio |
| Tarefas / tickets | Tarefas de item de pipeline |
| Apresentação, vídeo, áudios, valores padronizados | Canned responses com anexo |
| Jornadas determinísticas | Automação do CRM e evo-flow (`send_canned_response`, mover conversa) |
| Agente de IA | `evo-processor` + `evo-bot-runtime` + `evo-core` (Fase 4) |

### 2.3. Comunicação

A comunicação entre os componentes ocorrerá pelas APIs e eventos oficiais. Não haverá dois respondedores ativos na mesma conversa: ou o fluxo nativo (canned response / automação / agente) responde, ou a camada própria responde. Nunca os dois ao mesmo tempo.

---

## 3. Stack tecnológica

| Componente | Tecnologia | Responsabilidade |
|---|---|---|
| Hospedagem | VPS Linux (Ubuntu Server) | Execução isolada do sistema |
| Orquestração | Docker Compose oficial do tag `v1.1.0` | Implantação dos serviços do CRM |
| Proxy reverso | Traefik | HTTPS, HTTP e WebSocket/ActionCable por cima do compose oficial — sem reescrevê-lo |
| WhatsApp | Evolution API (Node), versão pinada | Integração e mensageria |
| Frontend | `evo-ai-frontend-community` | Interface de atendimento |
| Autenticação | `evo-auth-service-community` + Sidekiq | Login, RBAC, OAuth |
| CRM | `evo-ai-crm-community` + Sidekiq | Contatos, conversas, inbox, pipeline, tarefas |
| Processador de IA | `evo-ai-processor-community` (Python/FastAPI oficial) | Agentes, sessões, ferramentas |
| Core de agentes | `evo-ai-core-service-community` | Gestão de agentes e chaves |
| Runtime de bot | `evo-bot-runtime` | Pipeline do bot, debounce, dispatch |
| Jornadas | `evo-flow-community` | Fluxos e campanhas |
| Extensão própria | Python + FastAPI **somente se o nativo faltar** | Catálogo imobiliário e mute de takeover |
| Inteligência artificial | OpenAI API, a partir da Fase 4 | Interpretação fora do roteiro |
| Banco de dados | PostgreSQL 16 + pgvector | Persistência do CRM (`evo_community`) e, se o Flow estiver ativo, `evo_campaign` |
| Cache e filas | Redis | Estado, Sidekiq e cache |
| Broker do Flow | RabbitMQ | Exigido pelo evo-flow mesmo em modo direto |
| Eventos de campanha | ClickHouse | Analytics do evo-flow |
| Armazenamento | Volumes persistentes / ActiveStorage do CRM | Vídeos, imagens, áudios e documentos |

**Importante:** o Evo CRM Community não é uma aplicação monolítica. O compose oficial `v1.1.0` sobe, no mínimo, postgres, redis, rabbitmq, clickhouse, mailhog, evo-auth, evo-auth-sidekiq, evo-crm, evo-crm-sidekiq, evo-core, evo-processor, evo-bot-runtime, evo-flow e evo-frontend. A implantação aproveita essa arquitetura; não a substitui por uma aplicação Python única.

### 3.1. Versão de referência

- CRM: tag git `v1.1.0` (02/09/2026). Imagens Docker sem o prefixo `v` (exemplo: `evoapicloud/evo-ai-crm-community:1.1.0`).
- Clonar com `git clone --recurse-submodules --branch v1.1.0`.
- **Não** executar `git submodule update --remote` — isso sai do pin e pode puxar `develop`.
- Pin de **todas** as imagens, inclusive `evo-core`. O compose oficial usa `:latest` nesse serviço; isso será sobrescrito.
- Evolution API: versão pinada de release estável, independente do tag do CRM.
- Não usar Evolution Go. O inbound de mídia no CRM já falhou nesse provedor ([issue #119](https://github.com/evolution-foundation/evo-crm-community/issues/119)).

O README e a documentação pública do CRM ainda podem citar `v1.0.0-rc2` e “cinco serviços”. A referência operacional é o release `v1.1.0` e o `docker-compose.yml` desse tag.

### 3.2. Repositórios de referência

- [Evo CRM Community](https://github.com/evolution-foundation/evo-crm-community) — orquestração e submódulos
- [Evolution API](https://github.com/evolution-foundation/evolution-api) — WhatsApp (Node)

### 3.3. Licença

Evo CRM Community e Evolution API usam Apache 2.0 com condições extras:

- Não remover logo nem copyright do frontend.
- Exibir, na área administrativa ou na documentação de operação, que o sistema utiliza Evo CRM Community e Evolution API.

Sem esse aviso, o produtor pode exigir licença comercial. O aviso entra na Fase 2, junto com a primeira instalação.

---

## 4. Funcionalidades do sistema

### Módulo 1. Atendimento automático no WhatsApp

Prioridade: essencial. Implementação na Fase 3, **depois** dos go/no-go da Fase 2.

Quando um novo cliente entrar em contato, o sistema inicia o atendimento conforme sequência configurada:

1. Novo cliente envia mensagem — o sistema identifica o contato e verifica se já existe atendimento em andamento.
2. Envio da apresentação pessoal.
3. Apresentação do empreendimento — vídeo e os dois áudios gravados por Letícia.
4. Identificação do interesse.
5. Atualização do CRM — contato e estágio do pipeline.

O fluxo não deve repetir a apresentação a cada nova mensagem. O estado fica no contato/conversa/pipeline do CRM, não em uma máquina de estados paralela.

A primeira implementação tenta canned responses + regras nativas. Sequências diferentes por empreendimento só entram se o nativo permitir; caso contrário, isso vira candidato à extensão Python.

### Módulo 2. Apresentação de empreendimentos e valores

Prioridade: essencial.

Catálogo com nome, localização, descrição, tipologias, unidades, valores, condições de pagamento e materiais. Quando o cliente perguntar preço, o sistema identifica o empreendimento, consulta o cadastro e envia a mensagem de valores.

A IA não inventa nem estima preços. Letícia atualiza o cadastro quando houver alteração.

Este é o principal candidato a tabelas próprias, se o modelo nativo (atributos customizados + canned responses) for insuficiente.

### Módulo 3. CRM e classificação automática

Prioridade: essencial. Usar o pipeline nativo.

Funil inicial proposto, a ser criado como estágios do kanban:

1. Novo contato
2. Apresentação enviada
3. Interesse identificado
4. Visita a agendar
5. Visita agendada
6. Visita realizada
7. Em negociação
8. Venda ou locação concluída

Situações paralelas via status da conversa e labels: atendimento humano, aguardando resposta, sem interesse, atendimento encerrado.

Não criar funil paralelo em tabelas próprias.

### Módulo 4. Atendimento inteligente

Prioridade: após o fluxo básico (Fase 4). Usar o processador oficial, não um segundo agente.

A IA interpreta mensagens fora do roteiro, consulta o cadastro e responde ou encaminha para Letícia. Descontos, promessa de disponibilidade e alteração de condições comerciais ficam com ela.

Mensagens enviadas à OpenAI saem do ambiente próprio. Isso precisa constar no registro de tratamento da Fase 0/4 (LGPD).

### Módulo 5. Atendimento humano

Prioridade: essencial. **Go/no-go da Fase 2** — ver seção 13.

Estados operacionais:

| Estado | Comportamento nativo aproximado | Quem controla |
|---|---|---|
| Automático | Bot/automação responde | conversa `pending` + bot ativo |
| Humano | Letícia assume; bot suspenso | conversa `open` + label `human-takeover` |
| Pausado | Automação desligada até retomada explícita | label/configuração de inbox; sem retomada implícita |

A retomada automática só ocorre se estiver explicitamente configurada. O comportamento nativo atual **não** atende o takeover pelo celular — ver seção 13.

### Módulo 6. Agendamento de visitas

Prioridade: segunda etapa (Fase 5). Primeira versão com confirmação manual de Letícia, sem calendário externo.

### Módulo 7. Convites para cafés da manhã

Prioridade: segunda etapa (Fase 5), com aprovação manual de Letícia antes do disparo. Disparos semanais aumentam o risco de restrição no WhatsApp não oficial; não entram no MVP.

---

## 5. Estrutura de dados

O Evo CRM é a fonte principal de clientes, conversas, funil e tarefas.

Tabelas próprias só para o que o CRM não representar bem. Não duplicar contato, conversa, estágio nem ticket.

| Entidade | Onde vive | Informações principais |
|---|---|---|
| Cliente | CRM (contato) | Identificador nativo, nome, telefone, dados comerciais |
| Conversa / atendimento | CRM (conversa) | Status, labels, responsável, histórico |
| Funil | CRM (pipeline + estágios) | Etapa comercial |
| Tarefa / ticket | CRM (tarefa de item de pipeline) | Ver seção 12 |
| Empreendimento | Nativo (atributos/canned) ou tabela própria, se faltar | Nome, localização, descrição, materiais |
| Unidade | Idem | Tipologia, valor, disponibilidade |
| Interesse | Pipeline item + campos do contato | Cliente, empreendimento, status |
| Visita | Fase 5 — nativo (tarefa `meeting`) ou tabela própria | Data, horário, confirmação |
| Evento / convite | Fase 5 | Fora do MVP |

A estrutura definitiva depende da homologação da Fase 2 e das respostas da Fase 0.

---

## 6. Infraestrutura self-hosted

### 6.1. Servidor

Implantação inicial em VPS Linux com Docker. Isolamento próprio: redes, bancos, volumes, credenciais e backups.

Dimensionamento inicial para homologação — estimativa, não garantia:

- 4–6 vCPU
- 12–16 GB de RAM (pode ficar justo com ClickHouse + RabbitMQ + Flow + Evolution API + Traefik)
- 80–100 GB de disco
- Ubuntu Server

Medir CPU, RAM, disco e banco com a stack efetivamente no ar antes de dimensionar produção. Se o evo-flow não for necessário no MVP, avaliar desligá-lo (e ClickHouse/RabbitMQ) só depois de confirmar que canned responses e regras do CRM cobrem o fluxo inicial.

### 6.2. Serviços e isolamento

| Grupo | Componentes |
|---|---|
| Entrada | Traefik (HTTPS + WebSocket), envelopando o compose oficial |
| WhatsApp | Evolution API, compose/serviço separado, versão pinada |
| CRM | Compose oficial `v1.1.0`, sem remover serviços sem medir o efeito |
| Extensão | Serviço Python próprio — ausente até existir gap comprovado |
| Persistência | PostgreSQL, Redis; RabbitMQ e ClickHouse se o Flow estiver ativo |
| Armazenamento | Volumes persistentes e ActiveStorage do CRM |

Produção exige `BACKEND_URL` e `FRONTEND_URL` públicos. Defaults `localhost` quebram webhook, OAuth e URL de mídia.

Senhas e chaves default do compose oficial são de desenvolvimento e serão trocadas na primeira instalação.

### 6.3. Segurança e disponibilidade

- Autenticação individual, HTTPS, credenciais segregadas, APIs internas restritas
- Backups automatizados de banco e mídia, fora da VPS, com teste periódico de restauração
- Persistência da sessão da Evolution API
- Watchdog de conectividade WhatsApp (instância pode ficar `open` sem entregar webhook)
- Logs sem telefone, mensagem ou credencial em claro
- Aviso de licença visível para o administrador

---

## 7. Planejamento de desenvolvimento

A implantação é incremental. Cada fase só avança depois da validação da anterior.

### Fase 0. Coleta e premissas

Objetivo: reunir o que falta da Letícia e fechar as premissas que mudam o desenho, **antes de qualquer compose**.

A lista completa está na seção 14. Sem questionário, materiais e número de teste, a Fase 2 não começa com o número profissional e a Fase 3 não fecha especificação.

Entrega: respostas da Fase 0, pasta de materiais e número de teste definido.

### Fase 1. Levantamento e especificação

Objetivo: transformar as respostas da Fase 0 em requisitos e regras de atendimento.

- Analisar o questionário
- Catalogar mensagens, áudios, vídeos e tabelas de preços
- Identificar empreendimentos e diferenças de fluxo
- Documentar os fluxos reais
- Mapear campos e estágios para o pipeline nativo
- Estabelecer quando o humano entra e como retoma

Entrega: especificação funcional do MVP e catálogo inicial de mensagens (mapeado para canned responses).

### Fase 2. Infraestrutura e CRM

Objetivo: levantar a stack oficial e validar operação com número de teste.

- Preparar VPS, Docker e Traefik por cima do compose oficial
- Instalar Evo CRM Community `v1.1.0` com submódulos pinados
- Trocar senhas default e configurar `BACKEND_URL` / `FRONTEND_URL`
- Incluir o aviso de licença
- Instalar Evolution API pinada e conectar **número de teste**
- Validar texto, imagem, vídeo e áudio nos dois sentidos
- Testar atendimento no painel e sincronização
- Executar os go/no-go da seção 13 (takeover pelo celular, sessão após reboot, pipeline e tarefa nativos)

Entrega: CRM acessível, WhatsApp de teste funcional e parecer go/no-go. Sem Python próprio nesta fase.

### Fase 3. Automação determinística (MVP)

Objetivo: automatizar o repetitivo sem IA, usando nativo primeiro.

- Canned responses da apresentação, vídeo, áudios e valores
- Regras/jornadas nativas para novo contato e avanço de estágio
- Controle de estado no pipeline e nas labels (sem reiniciar apresentação)
- Transferência para humano conforme o workaround homologado na Fase 2
- Extensão Python somente se o catálogo ou o mute de takeover não couberem no nativo

Entrega: atendimento inicial autônomo + continuidade manual, sem dois bots na mesma conversa.

### Fase 4. Inteligência artificial

Objetivo: interpretação contextual com o processador oficial e OpenAI.

Somente depois do MVP real em uso. Registrar o fluxo de dados pessoais para a OpenAI.

### Fase 5. Agendamentos e eventos

Objetivo: visitas e convites, com confirmação manual. Fora do MVP.

### Fase 6. Homologação e entrega

Objetivo: número profissional, backups, documentação de uso e acompanhamento dos primeiros atendimentos reais.

O número da Letícia só é conectado aqui, depois de sessão persistida, mídia ok, takeover ok e watchdog no ar.

---

## 8. Critérios de aceite do MVP

O sistema estará apto à primeira utilização real quando estes cenários forem válidos no **número profissional**, após passarem no número de teste:

- Um novo contato é registrado automaticamente no CRM.
- A apresentação, o vídeo e os áudios corretos são enviados sem duplicação.
- O sistema identifica o empreendimento e envia os valores cadastrados.
- Uma nova mensagem de cliente existente não reinicia a apresentação.
- Letícia assume pelo WhatsApp do celular sem resposta concorrente do bot.
- Uma mensagem recebida durante atendimento humano não reativa a automação.
- O histórico permanece no CRM.
- O sistema não gera resposta comercial sem informação suficiente.
- A indisponibilidade temporária de um serviço não perde nem duplica mensagem.
- O ambiente recupera dados e sessão após reinício.
- A restauração de um backup conclui com sucesso.
- Uma solicitação de visita ou proposta vira tarefa nativa vinculada ao contato/conversa corretos, sem duplicar.

---

## 9. Custos previstos

Não haverá assinatura de CRM, n8n ou plataforma comercial de automação. Self-hosted não é custo zero.

| Item | Modelo de custo |
|---|---|
| Evo CRM Community | Sem cobrança por usuário na Community; obrigações de logo e aviso |
| Evolution API | Sem assinatura na integração escolhida; mesmas obrigações de aviso |
| Extensão Python | Sem licença comercial |
| PostgreSQL, Redis, RabbitMQ, ClickHouse | Sem assinatura de software |
| Hospedagem | Custo da VPS |
| OpenAI API | Cobrança por uso, só a partir da Fase 4 |
| Domínio | Registro ou renovação, se necessário |
| Backup externo | Conforme provedor |
| Risco do WhatsApp Web | Indireto: restrição ou perda do número profissional |

A OpenAI fica limitada às mensagens que realmente exigirem interpretação. Apresentação, materiais, valores padronizados e atualização determinística do CRM não geram chamada à IA.

---

## 10. Decisões pendentes

A especificação definitiva depende da Fase 0. As decisões que ainda mudam o desenho:

| Decisão | Informação necessária |
|---|---|
| Canal de resposta humana | Celular, painel do CRM ou os dois — define o workaround de takeover |
| Número | Um número compartilhado versus número só do bot |
| Fluxo inicial | Ordem exata das mensagens, vídeos e áudios |
| Empreendimentos | Quantidade, materiais e diferenças de atendimento |
| Valores | Origem dos dados, dono e frequência de atualização |
| Retomada | Quando o sistema para e como volta |
| Classificação | Critérios para avançar cada estágio do pipeline nativo |
| Visitas | Disponibilidade e confirmação (Fase 5) |
| Eventos | Destinatários e regras de convite (Fase 5) |
| IA | Assuntos que poderá responder sozinha (Fase 4) |

A integração Evolution API + CRM, o takeover pelo WhatsApp e a adequação do funil nativo são homologação técnica da Fase 2, não customização prévia.

---

## 11. Escopo da primeira entrega

**MVP:** atendimento inicial + CRM nativo + intervenção humana.

Incluído: WhatsApp de teste e depois o profissional, cadastro de contatos, apresentações automáticas, envio de vídeo e áudios, respostas padronizadas de valores, histórico, pipeline inicial e atendimento humano sem concorrência do bot.

Fora do MVP: agente de IA contextual, agendamento inteligente, convites semanais, relatórios avançados e qualquer FastAPI próprio que não tenha gap comprovado.

A execução começa pela Fase 0. Sem material da Letícia e sem número de teste, não se sobe compose nem se conecta o número profissional.

---

## 12. Gestão de tarefas pelo pipeline nativo

### 12.1. Objetivo

Registrar solicitações e pendências vinculadas ao cliente e à conversa, sem um gerenciador de tickets paralelo.

O Evo CRM já expõe tarefas em item de pipeline (`POST/GET /api/v1/pipelines/{id}/pipeline_items/{id}/tasks`), com título, tipo, status, prioridade, prazo, responsável, subtarefas e metadados. A operação acontece nessa interface.

### 12.2. Mapeamento dos estados pedidos

O planejamento original pedia A fazer, Em andamento, Aguardando cliente, Concluída e Cancelada. O enum nativo é outro. Usar o nativo e, se preciso, complementar com tipo, label ou metadata — não criar tabela de tickets.

| Estado desejado | Equivalente nativo |
|---|---|
| A fazer | `pending` |
| Em andamento | `pending` + metadata/label `in_progress` |
| Aguardando cliente | `pending` + metadata/label `waiting_customer` |
| Concluída | `completed` |
| Cancelada | `cancelled` |
| Atrasada | `overdue` (nativo) |

Tipos nativos úteis: `call`, `email`, `meeting`, `follow_up`, `note`, `other`. Visita tende a `meeting`; envio de proposta a `follow_up` ou `other`.

O estado da tarefa **não** substitui o estágio do pipeline nem o estado Automático/Humano/Pausado da conversa.

### 12.3. Requisitos

- Criação manual no CRM no MVP. Criação automática por regra ou IA fica para evolução.
- Identificador único nativo, sem reutilização.
- Vínculo com contato, item de pipeline e, por extensão, conversa. Empreendimento em campo/metadata quando existir.
- Campos mínimos: identificador, título, solicitante/responsável, status, criação e histórico. Prazo opcional.
- Consultar, atualizar, concluir e reabrir sem perder a origem.
- Notificação no CRM. Confirmação ao cliente no WhatsApp só se configurada e sem expor dado interno.
- Mensagens repetidas não criam tarefa duplicada.
- Existência de tarefa não reativa o bot em modo Humano ou Pausado.

### 12.4. Fluxo ilustrativo

1. Cliente pede visita ou proposta.
2. O sistema (ou Letícia, no MVP) verifica se já existe tarefa aberta da mesma solicitação.
3. Se não existir, cria tarefa no item de pipeline do contato, status `pending`.
4. Letícia executa pelo CRM.
5. Status e histórico atualizam até `completed`. Confirmação ao cliente só quando configurada.

### 12.5. Implantação incremental

**MVP:** homologar tarefas nativas na Fase 2; cadastrar e atualizar manualmente; vincular ao pipeline. Sem módulo paralelo.

**Evolução:** criação automática a partir de eventos, classificação com IA, prazos, lembretes e relatórios.

### 12.6. Critérios de aceite adicionais

- Solicitação de visita ou proposta gera tarefa nativa no item correto.
- A tarefa tem identificador estável e é localizável no CRM.
- Reprocessar a mesma mensagem não duplica a tarefa.
- Mudança de status mantém histórico, responsável e vínculo.
- Tarefa concluída não altera sozinha o estágio comercial nem retoma conversa humana.
- O recurso sobrevive a reinício e restore.

---

## 13. Go/no-go da Fase 2: takeover humano e número de teste

Esta seção é critério de passagem da Fase 2. Sem parecer escrito, a Fase 3 não começa e o número profissional não é conectado.

### 13.1. Por que é blocker

O critério de aceite do MVP exige que Letícia responda pelo WhatsApp do celular e o bot não dispute a conversa.

A issue [evolution-foundation/evo-crm-community#82](https://github.com/evolution-foundation/evo-crm-community/issues/82) descreve exatamente isso e segue aberta, sem PR. O bot ignora a mensagem `fromMe`, mas **retoma na próxima mensagem do cliente**. O parâmetro `stopBotFromMe` existe na Evolution API standalone; no CRM, não.

Evolution API emula WhatsApp Web (Baileys). Há relatos de `401 device_removed`, restrição de 24 horas e instância `open` sem webhook. O primeiro experimento **não** pode ser o número com o qual Letícia já atende clientes.

### 13.2. Regras da homologação

- Fase 2 usa somente número de teste.
- Não usar Evolution Go.
- Não ligar apresentação automática até o takeover estar homologado ou ter workaround aceito.
- Um único respondedor por conversa.

### 13.3. Workarounds, em ordem de preferência

1. **Atendimento só pelo painel do CRM** — o bot nativo costuma ceder quando o agente responde pelo dashboard. Quebra a premissa do celular; só serve como fallback temporário se Letícia aceitar.
2. **Patch/workaround no CRM após ver o webhook `fromMe`** — ao detectar outbound originado no cliente WhatsApp (não no painel), mover a conversa para `open`, aplicar label `human-takeover` e só retomar com ação explícita ou janela configurada. Caminho preferido se o teste confirmar o payload.
3. **Desligar o bot nativo e orquestrar o mute na extensão Python** — somente depois de provar que não há segundo respondedor (automação/flow/agente) na mesma inbox.

Se nenhum caminho passar no número de teste, o projeto não avança para automação no número profissional. Reavaliar premissa (atender só no CRM) ou canal (Meta Cloud API, fora do escopo atual).

### 13.4. Checklist go/no-go

Todos os itens abaixo no **número de teste**:

- [ ] Texto, imagem, vídeo e áudio fluem nos dois sentidos
- [ ] Sessão da Evolution API sobrevive a reboot da VPS sem novo QR, ou o procedimento de re-pareamento está documentado
- [ ] Letícia (ou o operador de teste) responde pelo celular; o bot não fala por cima
- [ ] A próxima mensagem do cliente **não** reativa o bot
- [ ] Retomada do automático só ocorre de forma explícita (ou na janela combinada)
- [ ] Pipeline nativo comporta o funil da seção 4
- [ ] Tarefa nativa cria, atualiza e localiza pendência sem tabela paralela
- [ ] Watchdog detecta instância `open` sem entrega de mensagem
- [ ] Aviso de licença visível para o administrador
- [ ] Backup de banco e de sessão tem teste de restore

Parecer possível: **go**, **go com ressalvas** (workaround aceito e documentado) ou **no-go**.

### 13.5. Quando o número da Letícia entra

Somente na Fase 6, se o parecer da Fase 2 for go ou go com ressalvas, e depois de repetir o checklist de takeover e sessão no número profissional em janela controlada.

---

## 14. Fase 0 — o que falta coletar da Letícia

Nada de compose, Traefik ou conexão de WhatsApp até esta lista ter dono e prazo. Itens técnicos (servidor, DNS) podem avançar em paralelo; itens de produto bloqueiam a especificação da Fase 3.

### 14.1. Bloqueiam qualquer conexão de WhatsApp

| Item | Por que importa | Status |
|---|---|---|
| Número de teste (chip ou linha descartável) | Homologação sem arriscar o número profissional | Pendente |
| Quem responde no dia a dia: celular, CRM ou os dois | Define o workaround de takeover | Pendente |
| Um número compartilhado ou número só do bot | Muda risco de conflito e de restrição | Pendente |
| Confirmação de que o número profissional só entra após o go/no-go | Premissa operacional | Pendente |

### 14.2. Bloqueiam a especificação do MVP (Fase 1/3)

| Item | Por que importa | Status |
|---|---|---|
| Questionário respondido | Regras reais de atendimento | Pendente |
| Texto da apresentação pessoal | Canned response inicial | Pendente |
| Vídeo de cada empreendimento | Mídia da sequência | Pendente |
| Os dois áudios gravados, por empreendimento | Mídia da sequência | Pendente |
| Ordem exata da sequência (texto → vídeo → áudios, ou outra) | Fluxo determinístico | Pendente |
| Lista de empreendimentos atendidos | Catálogo e diferenças de fluxo | Pendente |
| Tabela de valores e condições, com data de validade | Resposta de preço sem IA | Pendente |
| Quem atualiza valores e com que frequência | Evita preço velho no automático | Pendente |
| Mensagens prontas que ela já envia hoje | Base das canned responses | Pendente |
| Quando ela assume na mão (exemplos reais) | Regras de humano / pausa | Pendente |
| Quando o automático pode voltar | Retomada explícita vs. janela | Pendente |
| Critérios para avançar cada estágio do funil | Pipeline nativo | Pendente |

### 14.3. Não bloqueiam a Fase 2, mas devem ser anotados

| Item | Quando entra |
|---|---|
| Disponibilidade para visitas e regras de horário | Fase 5 |
| Lista e regras dos cafés da manhã / convites | Fase 5 |
| Assuntos que a IA poderá responder sozinha | Fase 4 |
| Consentimento / aviso sobre mensagens irem à OpenAI | Antes da Fase 4 |
| Acesso ao número profissional (QR, aparelho, 2FA) | Fase 6 |
| Preferência de domínio e e-mail de aviso da Letícia | Fase 2 (DNS) se já houver VPS |

### 14.4. Como receber o material

Preferir uma pasta compartilhada com nomes estáveis, por exemplo:

```text
fase-0/
  questionario.md
  mensagens/
    apresentacao.txt
    valores-<empreendimento>.txt
  midias/
    <empreendimento>/apresentacao.mp4
    <empreendimento>/audio-1.ogg
    <empreendimento>/audio-2.ogg
  tabelas/
    valores-<empreendimento>.csv
```

Não versionar mídia pesada neste repositório sem acordo. Telefones e preços reais não entram em commit público.

---

## 15. O que não muda em relação à versão 1.0

- MVP = apresentação + CRM + humano
- Sem IA no começo
- Sem n8n
- Sem Meta Cloud API neste momento
- PostgreSQL próprio
- Backup externo
- O sistema não inventa preço
