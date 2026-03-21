# 🛰️ AI-Ops Watchdog

> **Observabilidade de infraestrutura autônoma com n8n + GPT-4o**  
> Monitora sistemas, detecta anomalias, gera RCA e postmortems — automaticamente.

![Status](https://img.shields.io/badge/status-ativo-brightgreen?style=flat-square)
![n8n](https://img.shields.io/badge/n8n-1.x-EA4B71?style=flat-square&logo=n8n)
![OpenAI](https://img.shields.io/badge/GPT--4o-powered-412991?style=flat-square&logo=openai)
![License](https://img.shields.io/badge/licença-MIT-blue?style=flat-square)

---

## 📌 O que é isso?

**AI-Ops Watchdog** é um pipeline de observabilidade totalmente automatizado que substitui a triagem manual de alertas e a documentação de incidentes. Ele coleta métricas e logs da sua infraestrutura, usa o GPT-4o para classificar anomalias e identificar causas raiz, e então entrega RCAs e postmortems acionáveis — sem nenhuma intervenção humana no loop de detecção.

Construído inteiramente sobre o **n8n** como camada de orquestração, integra-se às ferramentas que seu time já usa: Prometheus, Slack, Notion, GitHub e e-mail.

---

## 🏗️ Arquitetura

```
┌─────────────────────────────────────────────────────┐
│                  FONTES DE DADOS                     │
│  Prometheus · Loki/ELK · AWS/GCP · GitHub · Uptime  │
└────────────────────┬────────────────────────────────┘
                     │ poll / webhook
                     ▼
┌─────────────────────────────────────────────────────┐
│            CAMADA DE ORQUESTRAÇÃO n8n                │
│   Agendamento · Normalização · Roteamento · Storage  │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│           ENGINE DE ANÁLISE GPT-4o                   │
│   Classificação · Geração de RCA · Narrativa         │
└──┬──────────────┬──────────────┬───────────────┬───┘
   │              │              │               │
   ▼              ▼              ▼               ▼
Alerta Slack  Issue GitHub  Página Notion  Relatório E-mail
```

---

## ⚙️ Visão Geral dos Workflows

| # | Workflow | Gatilho | Descrição |
|---|----------|---------|-----------|
| 01 | `health-monitor` | A cada 5 min | Consulta o Prometheus para CPU, memória e latência. Dispara o pipeline de anomalia se os limites forem excedidos. |
| 02 | `log-anomaly-detector` | Webhook (ELK/Loki) | Recebe streams de log, filtra entradas críticas e usa o GPT-4o para classificar e suprimir falsos positivos. |
| 03 | `incident-rca-agent` | Webhook (interno) | Disparado pelos workflows 01 ou 02. Busca 30 min de histórico de métricas e pede ao GPT-4o a análise de causa raiz. |
| 04 | `postmortem-writer` | Webhook (interno) | Cria postmortem estruturado em Markdown. Gera página no Notion + issue no GitHub automaticamente. |
| 05 | `weekly-ops-report` | Todo domingo às 8h | Consulta médias semanais, calcula health score, gera narrativa com IA e envia para Slack + e-mail. |

---

## 🚀 Primeiros Passos

### Pré-requisitos

- [n8n](https://n8n.io/) self-hosted ou cloud (v1.x+)
- Chave de API da OpenAI (acesso ao GPT-4o)
- Instância do Prometheus (ou API de métricas compatível)
- Workspace do Slack com um Incoming Webhook configurado

### 1. Clone o repositório

```bash
git clone https://github.com/Paula-Tech007/ai-ops-watchdog-n8n.git
cd ai-ops-watchdog-n8n
```

### 2. Configure as variáveis de ambiente

No seu n8n, vá em **Settings → Environment Variables** e adicione:

```env
# Infraestrutura
PROMETHEUS_URL=http://seu-prometheus:9090

# Notificações
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/SEU/WEBHOOK/URL

# URLs internas de webhook (copie de cada workflow após importar)
WEBHOOK_ANOMALY_URL=https://seu-n8n/webhook/log-ingest
WEBHOOK_INCIDENT_URL=https://seu-n8n/webhook/incident-rca
WEBHOOK_POSTMORTEM_URL=https://seu-n8n/webhook/postmortem-writer

# Destinos do postmortem
NOTION_DB_POSTMORTEMS_ID=id-do-seu-database-no-notion
GITHUB_OWNER=Paula-Tech007
GITHUB_REPO=ai-ops-watchdog-n8n

# Relatório semanal
REPORT_FROM_EMAIL=watchdog@seudominio.com
REPORT_TO_EMAIL=time-de-engenharia@seudominio.com
```

### 3. Importe os workflows

No n8n: **Settings → Import from file**, nesta ordem:

```
04_postmortem-writer.json    ← importe primeiro (copie a URL do webhook)
03_incident-rca-agent.json   ← importe segundo (copie a URL do webhook)
02_log-anomaly-detector.json ← importe terceiro (copie a URL do webhook)
01_health-monitor.json       ← importe por último (ative para começar o monitoramento)
05_weekly-ops-report.json    ← independente (ative separadamente)
```

> ⚠️ Importe de dentro para fora — a URL de webhook de cada workflow é referenciada pelo anterior. Copie cada URL para a variável de ambiente correspondente antes de importar o próximo.

### 4. Configure as credenciais no n8n

Vá em **Credentials** e adicione:
- **OpenAI API** — para os nós de análise de IA
- **Notion API** — para as páginas de postmortem (requer integração com seu workspace)
- **GitHub API** — para criação de issues (token clássico com escopo `repo`)
- **SMTP** — para o relatório semanal por e-mail

### 5. Ative e teste

Ative os workflows 02, 03, 04 primeiro (os webhooks precisam estar ouvindo). Depois ative o 01.

Para testar o pipeline de logs manualmente:

```bash
curl -X POST https://seu-n8n/webhook/log-ingest \
  -H "Content-Type: application/json" \
  -d '{
    "logs": [
      {
        "timestamp": "2024-01-15T10:30:00Z",
        "level": "ERROR",
        "service": "payment-api",
        "message": "Pool de conexões do banco de dados esgotado após timeout de 30s",
        "host": "prod-worker-01"
      }
    ]
  }'
```

---

## 📊 O que é gerado automaticamente

### Alerta no Slack (workflow 02)
```
🚨 AI-OPS WATCHDOG — Anomalia Detectada

Serviço:         payment-api
Classificação:   database_error
Resumo:          Pool de conexões esgotado em prod-worker-01
Causa Provável:  Pico de tráfego sobrecarregando as conexões ociosas
Ação Recomendada: Aumentar tamanho do pool ou reiniciar o serviço imediatamente
```

### RCA no Slack (workflow 03)
```
🔍 RCA Pronto — INC-1705316400000

Causa Raiz: Pool de conexões do PostgreSQL esgotado devido a pico de 3x
nas requisições de checkout combinado com uma query lenta introduzida
no deploy #482.

Ações Imediatas:
1. Reiniciar os pods do payment-api para liberar conexões
2. Reverter o deploy #482
3. Definir pool_size=50 na config do banco

MTTR Estimado: 12 minutos
```

### Issue no GitHub (workflow 04)
Postmortem completo com timeline, fatores contribuintes, passos de remediação e checklist de prevenção — criado automaticamente com labels e categorização.

### E-mail / Slack Semanal (workflow 05)
```
📊 Relatório Semanal — 14/01/2024 | Health Score: 87/100

Resumo Executivo: A infraestrutura teve bom desempenho essa semana com
CPU média de 42% e memória em 61%. Um incidente na terça foi resolvido
em menos de 15 minutos. Recomendamos revisar o tamanho do pool de
conexões antes do próximo pico de tráfego.
```

---

## 🧩 Stack

| Ferramenta | Papel |
|------------|-------|
| [n8n](https://n8n.io) | Orquestração dos workflows |
| [GPT-4o](https://platform.openai.com) | Classificação de anomalias, RCA, relatórios |
| [Prometheus](https://prometheus.io) | Fonte de métricas de infraestrutura |
| [Slack](https://slack.com) | Alertas e relatórios em tempo real |
| [Notion](https://notion.so) | Base de conhecimento de postmortems |
| [GitHub](https://github.com) | Rastreamento de issues e histórico de incidentes |

---

## 🔧 Customização

### Alterar os limites de alerta

No arquivo `01_health-monitor.json`, encontre o nó de código `Evaluate Metrics` e ajuste:

```javascript
if (cpuValue > 85)    // Limite de CPU (%)
if (memValue > 90)    // Limite de memória (%)
if (latValue > 2)     // Limite de latência p95 (segundos)
```

### Adicionar uma nova fonte de métrica

1. Adicione um nó HTTP Request consultando sua nova fonte
2. Conecte ao nó de código `Evaluate Metrics`
3. Adicione a nova métrica à lógica de avaliação

### Trocar o modelo de IA

Em qualquer nó da OpenAI, atualize o campo `model`. Testado com:
- `gpt-4o` (recomendado — melhor qualidade de RCA)
- `gpt-4o-mini` (mais rápido e barato — ótimo para classificação de logs)

---

## 📁 Estrutura do Repositório

```
ai-ops-watchdog-n8n/
├── workflows/
│   ├── 01_health-monitor.json
│   ├── 02_log-anomaly-detector.json
│   ├── 03_incident-rca-agent.json
│   ├── 04_postmortem-writer.json
│   └── 05_weekly-ops-report.json
├── dashboard/
│   └── index.html               ← dashboard de observabilidade standalone
├── docs/
│   └── architecture.png
└── README.md
```

---

## 🛡️ Boas Práticas de Segurança

- Em produção, proteja os endpoints de webhook com o header de autenticação nativo do n8n
- Guarde as chaves de API apenas no cofre de credenciais criptografado do n8n — nunca no JSON dos workflows
- Limite o escopo do token do GitHub apenas a `repo`
- Considere VPN ou allowlist de IPs para o acesso ao Prometheus

---

## 🤝 Contribuindo

Pull requests são bem-vindos. Para mudanças maiores, abra uma issue primeiro para discutirmos.

---

## 📄 Licença

MIT — feito por [Paula-Tech007](https://github.com/Paula-Tech007)

---

<div align="center">

</div>
