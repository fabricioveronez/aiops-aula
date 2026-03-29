---
prd_number: "001"
status: rascunho
priority: alta
created: 2026-03-29
issue: ""
depends_on: []
references: []
---

# PRD 001: Sistema Interno de Gerenciamento de Projetos (Kanban)

## 1. Contexto

- **Sistema/produto**: Sistema web interno para gerenciamento de projetos e tarefas, substituindo o controle atual feito via Google Sheets. Stack: Python + Django (backend e frontend com templates Django). Infraestrutura disponível em cluster Kubernetes interno.

- **Estado atual**: Hoje os 6 times da empresa (~40 funcionários) controlam projetos, tarefas, responsáveis e prazos em planilhas no Google Sheets. Cada time toca entre 3 e 5 projetos simultâneos. Não há sistema de notificações, dashboards ou relatórios automatizados. A comunicação acontece via Slack e o time de desenvolvimento usa GitHub.

- **Problema**: A planilha não escala. Os principais sintomas são:
  - **Falta de visibilidade**: gestores não conseguem ver rapidamente o que está atrasado, bloqueado ou onde precisam intervir.
  - **Informação desatualizada**: as pessoas esquecem de atualizar a planilha, tornando o dado não confiável.
  - **Desperdício de tempo**: toda segunda-feira a empresa perde ~1 hora de reunião de status por time, apenas para alinhar o que cada pessoa está fazendo.
  - **Falta de priorização clara**: os próprios desenvolvedores reclamam que não sabem a prioridade das tarefas.
  - **Tentativa anterior fracassada**: uma implantação de Jira foi abandonada por excesso de complexidade. O novo sistema precisa priorizar simplicidade acima de tudo.

## 2. Solução Proposta

### Visão geral

- Sistema web com autenticação própria (email/senha), sem dependência de SSO externo
- Organização hierárquica: **Times → Projetos → Tarefas**, com isolamento de visibilidade por time
- Board kanban com colunas fixas (Backlog → Em progresso → Em revisão → Concluído) para cada projeto
- Dashboard gerencial com visão consolidada de status, atrasos e bloqueios por time/projeto
- Notificações via Slack (atribuição de tarefa, prazo próximo) e relatório semanal automático por canal do time
- Integração com GitHub para vincular PRs/commits a tarefas

### Decisões-chave

1. **Colunas fixas no kanban** — Elimina decisão de configuração por time e mantém a experiência uniforme. Reduz complexidade, alinhado com a diretriz de simplicidade.
2. **Isolamento por time, não por projeto** — Um membro vê todos os projetos do seu time. Simplifica o modelo de permissão e reflete a estrutura organizacional real.
3. **Três papéis de acesso (Admin, Gestor, Membro)** — Granularidade suficiente sem overhead de configuração. Admin gerencia usuários e times; Gestor tem visão consolidada e gerencia projetos; Membro interage com tarefas dos projetos do seu time.
4. **Notificações via Slack, não email** — A empresa já usa Slack como canal principal de comunicação. Evita mais um canal de notificação que seria ignorado.
5. **Django com templates (server-side rendering)** — Evita complexidade de SPA + API separada. Para um sistema interno de 40 usuários, server-side rendering é mais que suficiente e acelera a entrega.

### Fora do escopo

- **Gráfico de Gantt** — Adiciona complexidade visual sem resolver o problema central de visibilidade de status
- **Gestão de orçamento** — Não é uma dor reportada e adicionaria campos que ninguém preencheria
- **Timesheet / controle de horas** — Mesmo motivo acima
- **App mobile nativo** — O sistema web responsivo atende. App nativo pode ser avaliado no futuro se houver demanda
- **SSO / integração com identity provider externo** — Não há SSO corporativo hoje. Pode ser adicionado futuramente se a empresa adotar um
- **Customização de colunas do kanban** — Decisão deliberada por simplicidade e uniformidade
- **Subtarefas ou dependências entre tarefas** — Aumenta complexidade significativamente. Tarefas devem ser quebradas em itens atômicos

## 3. Funcionalidades

### US01: Cadastro e autenticação de usuários

Como administrador, quero cadastrar usuários com email e senha, para que cada funcionário tenha acesso individual ao sistema.

**Rules:**
- Login via email corporativo + senha
- Senha deve ter no mínimo 8 caracteres, com pelo menos uma letra e um número
- Sessão expira após 8 horas de inatividade
- Admin pode desativar um usuário (soft delete — não exclui dados, apenas impede login)
- Cada usuário tem um papel global: Admin, Gestor ou Membro
- Admin pode resetar a senha de qualquer usuário

**Edge cases:**
- Usuário tenta login com email não cadastrado → mensagem genérica "Credenciais inválidas" (sem revelar se o email existe)
- Usuário desativado tenta login → mensagem "Conta desativada. Contate o administrador."
- Dois admins tentam desativar o mesmo usuário ao mesmo tempo → o segundo recebe confirmação idempotente (sem erro)
- Último admin tenta se desativar → sistema bloqueia a ação com mensagem "Não é possível desativar o último administrador"

### US02: Gestão de times

Como administrador, quero criar e gerenciar times, para organizar os funcionários conforme a estrutura da empresa.

**Rules:**
- Um time tem nome (único) e descrição (opcional)
- Um usuário pode pertencer a mais de um time
- Cada time tem pelo menos um Gestor atribuído
- Admin pode adicionar e remover membros de qualquer time

**Edge cases:**
- Remover o último Gestor de um time → sistema bloqueia e exibe "O time precisa ter pelo menos um gestor"
- Remover um membro que tem tarefas atribuídas → membro é removido do time, mas as tarefas permanecem atribuídas a ele com um indicador visual "Membro removido" para que o Gestor reatribua
- Tentar criar time com nome duplicado → erro "Já existe um time com este nome"

### US03: Gestão de projetos

Como gestor de time, quero criar e gerenciar projetos dentro do meu time, para organizar o trabalho em andamento.

**Rules:**
- Um projeto pertence a exatamente um time
- Projeto tem: nome, descrição (opcional) e status (Ativo / Arquivado)
- Apenas Admin e Gestores do time podem criar, editar e arquivar projetos
- Projetos arquivados saem da visualização padrão mas podem ser consultados via filtro
- Membros do time veem todos os projetos do time
- Usuários fora do time não veem os projetos daquele time (exceto Admin, que vê tudo)

**Edge cases:**
- Arquivar um projeto com tarefas em andamento → permitido, mas exibe confirmação "Este projeto tem X tarefas não concluídas. Deseja arquivar mesmo assim?"
- Gestor tenta acessar projeto de outro time → retorna 403

### US04: Board kanban com gestão de tarefas

Como membro de um time, quero visualizar e gerenciar tarefas em um board kanban, para saber o que tenho que fazer e acompanhar o progresso.

**Rules:**
- Colunas fixas: **Backlog** → **Em progresso** → **Em revisão** → **Concluído**
- Card da tarefa exibe: título, responsável (avatar), prioridade (indicador visual) e prazo
- Campos da tarefa: título (obrigatório), descrição (opcional, com suporte a Markdown), responsável (opcional — pode ser "não atribuída"), prioridade (Alta / Média / Baixa, padrão: Média), prazo (opcional), labels/tags (opcionais, texto livre), comentários
- Arrastar e soltar (drag and drop) para mover tarefas entre colunas
- Qualquer membro do time pode criar, editar e mover tarefas dentro dos projetos do time
- Tarefas movidas para "Concluído" registram automaticamente a data de conclusão
- Filtros no board: por responsável, por prioridade, por prazo (atrasadas, vencendo esta semana)
- Tarefas atrasadas (prazo ultrapassado e não concluídas) recebem indicador visual vermelho

**Edge cases:**
- Dois usuários arrastam a mesma tarefa ao mesmo tempo → o segundo recebe mensagem "Esta tarefa foi movida para [coluna] por [usuário]. Atualize a página."
- Tarefa sem prazo → não aparece em filtros de prazo, sem indicador de atraso
- Tarefa sem responsável → aparece no board mas sem avatar, filtrável como "Não atribuída"
- Usuário tenta mover tarefa de "Concluído" de volta para outra coluna → permitido (a data de conclusão é removida)

### US05: Dashboard gerencial

Como gestor, quero ver um dashboard consolidado, para identificar rapidamente o que está atrasado, bloqueado e onde preciso intervir.

**Rules:**
- Gestor vê dashboard dos times que gerencia; Admin vê dashboard de todos os times
- Visão consolidada por time, com drill-down para projeto
- Métricas exibidas:
  - Total de tarefas por coluna (por projeto e consolidado por time)
  - Tarefas atrasadas (prazo vencido, não concluídas) com destaque
  - Tarefas sem responsável
  - Tarefas concluídas na última semana
- Indicadores visuais: vermelho para projetos com mais de 3 tarefas atrasadas, amarelo para 1-3 atrasadas, verde para nenhuma
- Dashboard atualiza ao carregar a página (sem necessidade de real-time/websocket)

**Edge cases:**
- Time sem projetos → exibe mensagem "Nenhum projeto ativo neste time"
- Projeto sem tarefas → exibe no dashboard com todos os contadores zerados
- Gestor gerencia múltiplos times → vê todos consolidados com filtro por time

**Notas de implementação:**
- Queries devem ser otimizadas com agregação no banco. Para 40 usuários e ~100 projetos, a carga é baixa, mas evitar N+1 queries desde o início.

### US06: Notificações via Slack

Como membro de um time, quero receber notificações no Slack, para saber quando fui atribuído a uma tarefa ou quando um prazo está próximo.

**Rules:**
- Notificações enviadas via Slack Bot (Slack App com Bot Token)
- Eventos que disparam notificação:
  - Tarefa atribuída a um usuário → DM para o usuário no Slack
  - Prazo em 24 horas → DM para o responsável
  - Prazo vencido → DM para o responsável + mensagem no canal do time
- Para vincular Slack ao usuário: campo "Slack User ID" no perfil do usuário (preenchido pelo Admin ou pelo próprio usuário)
- Notificações de prazo executam via job agendado (cron/celery beat), rodando 1x por dia às 9h

**Edge cases:**
- Usuário sem Slack User ID configurado → notificação não é enviada, mas fica registrada em log
- Slack API fora do ar → notificação falha silenciosamente com retry automático (3 tentativas com backoff). Se todas falharem, registra em log
- Tarefa sem responsável → nenhuma notificação de prazo é enviada
- Tarefa atribuída e removida rapidamente (< 1 min) → notificação já foi enviada, sem recall

### US07: Relatório semanal automático

Como gestor, quero receber um relatório semanal automático, para acompanhar o progresso sem precisar de reunião de status.

**Rules:**
- Relatório gerado toda segunda-feira às 8h, enviado ao canal do Slack de cada time
- Conteúdo do relatório:
  - Resumo por projeto: tarefas concluídas na semana, tarefas em andamento, tarefas atrasadas
  - Top 5 tarefas atrasadas do time (ordenadas por dias de atraso)
  - Tarefas sem responsável
- Formato: mensagem formatada no Slack (usando Block Kit para legibilidade)
- Período do relatório: segunda anterior até domingo

**Edge cases:**
- Time sem atividade na semana → relatório enviado com mensagem "Nenhuma movimentação de tarefas nesta semana"
- Canal do Slack do time não configurado → relatório não é enviado, erro registrado em log. Dashboard exibe alerta para o Admin
- Segunda-feira é feriado → relatório é enviado normalmente (não há lógica de feriado)

### US08: Integração com GitHub

Como desenvolvedor, quero vincular commits e PRs às tarefas, para que o progresso técnico seja visível no board sem atualização manual.

**Rules:**
- Vínculo por menção do ID da tarefa na mensagem do commit ou no título/corpo do PR (formato: `#TASK-123`)
- Webhook do GitHub notifica o sistema quando:
  - PR é aberto mencionando uma tarefa → tarefa recebe link para o PR e badge "PR aberto"
  - PR é mergeado mencionando uma tarefa → tarefa move automaticamente para "Em revisão" (se estava em "Em progresso") ou "Concluído" (se estava em "Em revisão")
- A movimentação automática pode ser revertida manualmente (drag and drop)
- Histórico de commits/PRs vinculados aparece nos comentários da tarefa automaticamente

**Edge cases:**
- PR menciona tarefa que não existe → webhook ignora silenciosamente
- PR menciona múltiplas tarefas → todas são atualizadas
- PR é mergeado mas a tarefa já está em "Concluído" → nenhuma ação, apenas registra o vínculo
- Webhook do GitHub falha (timeout, erro de rede) → GitHub faz retry automático (comportamento padrão do GitHub webhooks)
- Repositório não está configurado no sistema → webhook é ignorado

**Notas de implementação:**
- Implementar via GitHub Webhooks (push e pull_request events)
- Validar assinatura do webhook (secret) para segurança

## 4. Visão de Arquitetura

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Browser    │────▶│    Django     │────▶│   PostgreSQL    │
│  (Templates) │◀────│  (Gunicorn)  │◀────│                 │
└─────────────┘     └──────┬───────┘     └─────────────────┘
                           │
                    ┌──────┴───────┐
                    │  Celery +    │
                    │  Redis       │
                    │  (async jobs)│
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Slack   │ │  GitHub  │ │  Cron    │
        │  API     │ │ Webhooks │ │  Jobs    │
        └──────────┘ └──────────┘ └──────────┘
```

- **Django + Gunicorn**: Aplicação web com server-side rendering (templates Django)
- **PostgreSQL**: Banco de dados relacional
- **Celery + Redis**: Processamento assíncrono para notificações Slack e relatórios semanais
- **Kubernetes**: Deploy no cluster interno existente

Todos os componentes são novos. A infraestrutura Kubernetes já existe.

## 5. Critérios de Aceite

### Técnicos

| Critério | Método de verificação |
|----------|----------------------|
| Tempo de carregamento do dashboard < 2s com 100 projetos e 2000 tarefas | Teste de carga com dados seed + medição no browser |
| Drag and drop no kanban funciona nos navegadores Chrome e Firefox (últimas 2 versões) | Teste manual em ambos os navegadores |
| Webhook do GitHub processa evento em < 5s | Log de timestamp entre recebimento e processamento |
| Notificações Slack entregues em < 30s após o evento | Log de timestamp entre evento e confirmação do Slack API |
| Senhas armazenadas com hash seguro (bcrypt/argon2 via Django) | Code review + verificação no banco |
| Endpoints protegidos por autenticação e autorização por papel | Testes automatizados tentando acesso não autorizado |

### De negócio

| Métrica | Baseline (fonte) | Meta | Prazo | Mín. aceitável | Responsável |
|---------|-------------------|------|-------|-----------------|-------------|
| Tempo da reunião de status semanal | 1h/time/semana (relato do gestor) | 15 min/time/semana | 8 semanas após go-live | 30 min/time/semana | Gestor de cada time |
| Adoção do sistema | 0% (não existe) | 100% dos times usando ativamente | 4 semanas após go-live | 80% dos times (5 de 6) | Admin do sistema |
| % de tarefas com status atualizado | A levantar — medir nas 2 primeiras semanas de uso | 90% das tarefas atualizadas em < 24h | 6 semanas após go-live | 70% | Gestor de cada time |

## 6. Milestones

### Milestone 1: Estabelecer fundação e gestão de acesso

**Objetivo:** Sistema acessível com autenticação, gestão de times e estrutura base de projetos.

**Funcionalidades:** US01, US02, US03

- [ ] Configurar projeto Django, banco PostgreSQL e estrutura de deploy no Kubernetes (US01)
- [ ] Implementar cadastro, login, logout e gestão de sessão (US01)
- [ ] Implementar CRUD de times com atribuição de membros e gestores (US02)
- [ ] Implementar CRUD de projetos vinculados a times com controle de visibilidade (US03)
- [ ] Implementar sistema de permissões por papel (Admin/Gestor/Membro) (US01, US02, US03)

**Critério de conclusão:**
- Condição: Usuários conseguem fazer login, ver seus times e projetos, com isolamento de acesso funcionando
- Verificação: Testes automatizados de autenticação e autorização + deploy em staging
- Aprovador: Tech Lead

### Milestone 2: Implementar board kanban

**Objetivo:** Times conseguem gerenciar tarefas visualmente no dia a dia.

**Funcionalidades:** US04

- [ ] Implementar modelo de tarefas com todos os campos (US04)
- [ ] Implementar board kanban com colunas fixas e drag and drop (US04)
- [ ] Implementar filtros por responsável, prioridade e prazo (US04)
- [ ] Implementar indicadores visuais de atraso e prioridade (US04)
- [ ] Implementar comentários em tarefas (US04)

**Critério de conclusão:**
- Condição: Membros conseguem criar, mover e filtrar tarefas no board kanban de seus projetos
- Verificação: Teste funcional com dados reais de 1 time piloto em staging
- Aprovador: Gestor do time piloto

### Milestone 3: Entregar dashboard gerencial

**Objetivo:** Gestores têm visibilidade consolidada do status de seus times sem precisar abrir cada projeto.

**Funcionalidades:** US05

- [ ] Implementar dashboard com métricas agregadas por time e projeto (US05)
- [ ] Implementar indicadores visuais de saúde dos projetos (US05)
- [ ] Implementar drill-down de time para projeto (US05)
- [ ] Otimizar queries de agregação (US05)

**Critério de conclusão:**
- Condição: Dashboard carrega em < 2s e exibe métricas corretas para cada gestor, respeitando permissões
- Verificação: Teste com dados seed representativos + validação manual pelos gestores
- Aprovador: Gestores

### Milestone 4: Integrar notificações Slack e relatório semanal

**Objetivo:** Comunicação automatizada elimina a necessidade de reuniões de status para alinhamento básico.

**Funcionalidades:** US06, US07

- [ ] Configurar Slack App com Bot Token (US06)
- [ ] Implementar Celery + Redis para jobs assíncronos (US06, US07)
- [ ] Implementar notificações de atribuição e prazo via Slack DM (US06)
- [ ] Implementar relatório semanal automático por canal do time (US07)
- [ ] Implementar campo Slack User ID no perfil e configuração de canal por time (US06, US07)

**Critério de conclusão:**
- Condição: Notificações chegam no Slack em < 30s e relatório semanal é enviado às segundas às 8h
- Verificação: Teste end-to-end com Slack workspace de staging + execução manual do job de relatório
- Aprovador: Tech Lead + Gestor

### Milestone 5: Integrar com GitHub

**Objetivo:** Progresso técnico reflete automaticamente no board, reduzindo atualização manual pelos devs.

**Funcionalidades:** US08

- [ ] Implementar endpoint de webhook do GitHub com validação de assinatura (US08)
- [ ] Implementar parser de menção de tarefa em commits e PRs (US08)
- [ ] Implementar movimentação automática de tarefas com base em eventos de PR (US08)
- [ ] Implementar exibição de PRs/commits vinculados nos comentários da tarefa (US08)

**Critério de conclusão:**
- Condição: PR mergeado no GitHub movimenta tarefa automaticamente e exibe vínculo no card
- Verificação: Teste com repositório real do time de dev em staging
- Aprovador: Tech Lead

## 7. Riscos e Dependências

| Risco | Impacto | Mitigação | Status |
|-------|---------|-----------|--------|
| Baixa adoção por resistência dos times (repetição do caso Jira) | Alto | Priorizar simplicidade radical. Envolver 1 time piloto desde o Milestone 2 para feedback. Evitar campos obrigatórios desnecessários | Pendente |
| Dados da planilha atual precisam ser migrados | Médio | Criar script de migração CSV → banco. Priorizar migração de projetos ativos apenas | Pendente |
| Slack API com rate limiting em times com muitas notificações | Baixo | Para 40 usuários o volume é baixo. Celery com rate limiting configurável como precaução | Pendente |
| Complexidade do drag and drop cross-browser | Médio | Usar biblioteca JavaScript madura (SortableJS ou similar) em vez de implementação custom | Pendente |

**Dependências:**

| Dependência | Tipo | Status | Impacto se bloqueado |
|-------------|------|--------|----------------------|
| Cluster Kubernetes interno | Interna | Disponível | Milestones 1-5 (deploy) |
| Criação de Slack App no workspace da empresa | Interna | Pendente | Milestone 4 |
| Configuração de webhooks nos repositórios GitHub | Interna | Pendente | Milestone 5 |
| Definição de qual time será piloto | Interna | Pendente | Milestone 2 (validação) |

## 8. Referências

- [Django Documentation](https://docs.djangoproject.com/) — Framework principal do projeto
- [Slack API - Bot Tokens](https://api.slack.com/authentication/token-types#bot) — Referência para integração de notificações
- [GitHub Webhooks](https://docs.github.com/en/webhooks) — Referência para integração de PRs/commits
- [SortableJS](https://sortablejs.github.io/Sortable/) — Candidata para implementação de drag and drop no kanban
- [Celery Documentation](https://docs.celeryq.dev/) — Referência para jobs assíncronos e agendados

## 9. Registro de Decisões

- **2026-03-29:** Stack definida como Python + Django com server-side rendering. Motivo: familiaridade do time e simplicidade para sistema interno de ~40 usuários.
- **2026-03-29:** Autenticação própria (email/senha) sem SSO. Motivo: empresa não possui identity provider corporativo.
- **2026-03-29:** Três papéis de acesso: Admin, Gestor, Membro. Motivo: granularidade suficiente sem complexidade de permissões customizáveis.
- **2026-03-29:** Isolamento de visibilidade por time, não por projeto. Motivo: reflete a estrutura organizacional e simplifica o modelo de permissões.
- **2026-03-29:** Colunas fixas no kanban (Backlog, Em progresso, Em revisão, Concluído). Motivo: simplicidade e uniformidade entre times — prioridade explícita após fracasso do Jira.
- **2026-03-29:** Notificações exclusivamente via Slack (sem email). Motivo: Slack já é o canal principal de comunicação da empresa.
- **2026-03-29:** GitHub integration via webhooks com menção de task ID. Motivo: padrão amplamente adotado, sem necessidade de GitHub App complexo.
