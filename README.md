# ReativaLead

> **Reativação automatizada de leads imobiliários via WhatsApp.**
> Construído para corretores autônomos e pequenas imobiliárias que investem em tráfego pago e perdem a maior parte dos leads gerados por falta de follow-up consistente.

![Status](https://img.shields.io/badge/status-em%20produ%C3%A7%C3%A3o-success) ![Stack](https://img.shields.io/badge/stack-Next.js%20%C2%B7%20Supabase%20%C2%B7%20n8n-1f2937) ![Taxa de resposta](https://img.shields.io/badge/taxa%20de%20resposta-20%25-brightgreen)

> 📌 Este repositório é um **case study técnico** do ReativaLead. O código-fonte do produto é fechado. Aqui você encontra a arquitetura, decisões técnicas e estado atual do projeto.

---

## O problema

Imobiliárias brasileiras investem entre **R$ 2.000 e R$ 5.000 por mês** em tráfego pago (Meta Ads, Imoblead) e geram centenas de leads por mês. Mas a maior parte dessa base **nunca é recontactada**.

O motivo é estrutural, não falta de vontade:

- **Volume.** Um corretor com 500 leads na planilha não tem como ligar pra todos, muito menos fazer 4 toques de follow-up em cada.
- **Consistência.** Follow-up manual depende de disciplina. O corretor prioriza leads novos e a base antiga vira cemitério.
- **Personalização em escala.** Copiar e colar mensagem personalizada pra 300 leads no WhatsApp é inviável: demora horas e é propenso a erro.

Resultado: **a maior parte do investimento em tráfego é desperdiçada** porque o lead não recebe nem a primeira mensagem de retomada.

## A solução

O ReativaLead automatiza o reaquecimento dessa base parada e a qualificação inicial via WhatsApp, devolvendo ao corretor **apenas os leads que demonstraram interesse**.

### Funcionalidades principais

- **Importação inteligente de planilha.** Upload de CSV/XLSX com auto-detecção de cabeçalho, mapeamento flexível de colunas, normalização de telefone e merge de duplicatas.
- **Campanha com rodízio de templates.** 3 mensagens por tipo alternando automaticamente entre leads, com verificação prévia no WhatsApp antes do envio.
- **Follow-up automático de 4 toques.** Sistema avança o lead por 4 etapas (3, 5, 7, 7 dias) sem intervenção. Cada etapa tem templates próprios.
- **CRM inline editável.** Status, etapa, prioridade e observações editáveis direto na tabela, sem abrir formulários.
- **Dashboard com métricas em tempo real.** Funil de conversão, taxa de resposta por etapa de follow-up, controle de limites diários e mensais.

### Fluxo do corretor

1. Login no dashboard
2. Importa planilha de leads (drag-and-drop, mapeia colunas, confirma)
3. Cadastra templates de mensagem (3 por tipo)
4. Cria campanha aplicando filtros, vê prévia, confirma disparo
5. Sistema executa com delay aleatório (12–25s entre envios)
6. Follow-ups disparam nos dias seguintes sem ação manual
7. Corretor trata **apenas os leads que responderam**

### Fluxo do lead

Recebe mensagem personalizada no WhatsApp → silêncio aciona follow-up 1 após 3 dias → silêncio aciona follow-up 2 após 5 dias → mais 2 toques de 7 dias → se ainda em silêncio, marcado como finalizado. Resposta em qualquer etapa interrompe o ciclo e notifica o corretor.

---

## Arquitetura

```mermaid
flowchart TB
    Dashboard["Dashboard<br/>Next.js + Vercel"]

    subgraph n8n["n8n self-hosted · 5 workflows"]
        WF0[WF0 · Importação]
        WF1[WF1 · Agente de Campanhas]
        WF2[WF2 · Despachante de Disparos]
        WF3[WF3 · Resposta do Lead]
        WFFollow[WF Follow-up<br/>Cron diário 10h · 92 nodes]
    end

    Supabase[("Supabase<br/>PostgreSQL + RLS")]
    Redis[("Redis<br/>Fila de campanha")]
    Evo["Evolution API<br/>WhatsApp"]

    Dashboard -->|upload planilha| WF0
    Dashboard -->|criar campanha| WF1
    WF0 --> Supabase

    WF1 --> Supabase
    WF1 -->|salva fila| Redis
    WF1 --> WF2

    WF2 -->|lê fila| Redis
    WF2 -->|busca leads| Supabase
    WF2 -->|envia| Evo
    WF2 -->|atualiza status| Supabase

    Evo -->|webhook resposta| WF3
    WF3 -->|atualiza lead| Supabase

    WFFollow -->|busca elegíveis| Supabase
    WFFollow -->|envia| Evo
    WFFollow -->|avança etapa| Supabase
```

### Camadas

| Camada | Tecnologia | Responsabilidade |
|---|---|---|
| Frontend | Next.js 16 + TypeScript + Tailwind + shadcn/ui | Dashboard, CRM, campanhas, templates |
| Autenticação | Supabase Auth | Login, JWT, base do RLS |
| Banco | Supabase (PostgreSQL) | Leads, campanhas, templates, contadores |
| Automação | n8n self-hosted em Docker Swarm | 5 workflows orquestrando o backend |
| Fila | Redis | Fila de disparo por campanha |
| Mensageria | Evolution API v2 (MVP) | Envio, verificação de número, webhook de resposta |
| Deploy | Vercel (front) + VPS (backend) | Separação clara front/back |

### Multi-tenancy

Isolamento por **Row-Level Security (RLS) nativa do PostgreSQL**. Cada tabela tem policy `user_id = auth.uid()` — um cliente nunca enxerga dado de outro, mesmo que a aplicação tenha bug. Redis usa chaves prefixadas por `user_id` (`disparo:{user_id}:fila`). Onboarding de novo cliente leva ~15 minutos via dashboard admin.

---

## Decisões técnicas

> Esta seção documenta escolhas que foram **debatidas internamente** e os trade-offs assumidos.
> O objetivo não é convencer de que cada decisão é a melhor — é mostrar que foi consciente.

### 1. n8n self-hosted em vez de serverless functions

**Decisão.** Toda a lógica de backend roda em workflows n8n self-hosted (Docker Swarm), não em Lambda/Edge Functions/API server tradicional.

**Por quê.** O fluxo de follow-up tem **92 nodes em 5 ramos paralelos**. Gerenciar essa complexidade em código serverless seria impraticável — qualquer mudança exigiria deploy completo e debug remoto. Em n8n, o fluxo é visual, debugável node a node, e cada execução fica registrada em histórico.

**Trade-off.** Precisa manter servidor de pé (custo + manutenção). Em troca, ganho velocidade de iteração absurda — uma mudança no follow-up que levaria meio dia em código leva 15 minutos no n8n.

### 2. Evolution API self-hosted (MVP) em vez de WhatsApp Business API oficial

**Decisão.** Mensageria via Evolution API (Baileys/WhatsApp Web) em VPS própria, não via BSP comercial (Meta Cloud API, Zapster).

**Por quê.** Custo. API oficial cobra R$ 0,15–0,80 por mensagem; em campanha de 1.000 leads + 4 toques de follow-up, isso já é R$ 600–3.200. Inviável pro ticket-médio do cliente-alvo no MVP.

**Trade-off.** Maior risco de banimento, menor estabilidade, sem selo verificado. **Decisão consciente para MVP** — migração para Zapster (BSP comercial) já planejada quando receita justificar.

### 3. Redis como fila em vez de BullMQ ou SQS

**Decisão.** Fila de campanha em chaves Redis simples (`LPUSH`/`RPOP`), sem worker dedicado.

**Por quê.** O próprio n8n é o consumer — ele lê o Redis dentro do workflow. Adicionar BullMQ exigiria um worker Node.js separado só pra rodar a fila, dobrando a superfície de manutenção pra ganho marginal.

**Trade-off.** Sem retry automático, sem dead-letter queue. Aceito para o volume atual (<1.000 leads por campanha). Se a operação passar disso, troca pra BullMQ.

### 4. Rodízio de templates por hash de UUID em vez de round-robin com estado

**Decisão.** Para distribuir os 3 templates entre N leads, usa o **último caractere do UUID** do lead como seed (`hex % 3`).

**Por quê.** Stateless. Não precisa de contador global, não precisa de lock, não há race condition. Cada lead recebe seu template determinístico baseado no próprio UUID.

**Trade-off.** A distribuição não é exatamente 33/33/33 — a aleatoriedade dos UUIDs gera variação de poucos pontos. Aceito.

---

## Estado atual

**Em produção, com 2 pilotos ativos há ~2 meses** — primeira corretora piloto em Pelotas-RS (~1.800 leads na base) e uso interno da agência.

Números acumulados no período:

| Métrica | Valor |
|---|---|
| Disparos iniciais executados | **400** |
| Respostas recebidas | **80** (taxa de **20%**) |
| Leads em fase final de qualificação para assinatura | **2** |
| Clientes em produção | 2 |

A taxa de resposta de **20% está acima do benchmark típico de cold outreach via WhatsApp** (geralmente 5–10%), o que valida o efeito combinado de rodízio de templates + personalização por variável + verificação prévia de número.

### Capacidade técnica

- ~200 mensagens/hora por instância
- ~2.200/dia em janela comercial
- Limites de plano (100/300/500 por dia) são intencionalmente inferiores à capacidade técnica, como margem de segurança contra banimento

### Próxima validação

Primeira campanha de escala real (~1.800 leads da base da corretora piloto), em curso. O objetivo é medir a taxa de resposta no volume maior e validar a estabilidade da Evolution API nesse patamar — único gap relevante entre o estado atual e a operação em escala.

---

## Limitações conhecidas

- **Sem entrada automática de leads.** CRMs imobiliários (Imoblead etc.) não oferecem API aberta. Entrada via upload manual de planilha. Webhook de integração quando o CRM permitir.
- **Volume >5.000 leads/campanha** — o `SplitInBatches` do n8n processa sequencialmente; campanhas desse porte podem levar mais de 24h. Próximo passo: paralelização com múltiplas instâncias.
- **Observabilidade limitada.** Sem APM dedicado (Sentry/Datadog). Detecção de erros via logs do n8n e Supabase. Aceitável no volume atual, planejado pra escala.
- **Internacionalização.** Normalização de telefone assume formato brasileiro (DDD 2 dígitos + 9 dígitos). Adaptação necessária pra outros mercados.

---

## Roadmap (próximos 3–6 meses)

- Primeira campanha de escala (1.800+ leads) com métricas reais de taxa de resposta
- Migração de Evolution API para **Zapster** (BSP comercial — maior estabilidade, menor risco de banimento)
- Landing page do produto + funil de aquisição
- 5–10 clientes pagantes
- Notificação em tempo real para o corretor quando lead responde
- Integração com webhook do Imoblead (entrada automática)

---

## Stack

- **Frontend** — Next.js 16 · TypeScript · Tailwind CSS · shadcn/ui
- **Backend** — n8n 2.17.7 (self-hosted, Docker Swarm) · Edge Functions (Supabase)
- **Banco** — PostgreSQL (Supabase) com RLS nativo · 8 tabelas principais · índice único parcial em `(user_id, telefone)` para merge-duplicates
- **Cache / Fila** — Redis 7.4 Alpine
- **Mensageria** — Evolution API v2 (Baileys) · migração pra Zapster planejada
- **IA** — GPT-4o-mini (OpenAI) para o agente conversacional do Chatwoot pós-resposta
- **Auth** — Supabase Auth · JWT · RLS por `auth.uid()`
- **Hosting** — Vercel (frontend) · VPS dedicada (n8n + Redis + Evolution API)

---

## Sobre

Construído por [Guilherme Bosco](https://github.com/Guilherme-Bosco), co-founder da [Mind in Shift](https://mindinshift.com.br) — agência de automação e IA em Jacareí-SP.

Para contato sobre o produto ou consultoria técnica em automação: [contato@mindinshift.com.br](mailto:contato@mindinshift.com.br) · [LinkedIn](https://www.linkedin.com/in/guilherme-bosco-dos-santos-012bb620b/)