# 🏥 Sprint Health Agent

**Dashboard inteligente de saúde de sprint** — monitoramento contínuo de métricas ágeis com recomendações baseadas em IA.

---

## 📋 Visão Geral

O **Sprint Health Agent** extrai dados em tempo real de quadros Jira, calcula métricas de saúde da sprint (velocidade, burndown, taxa de conclusão, lead time) e gera recomendações acionáveis via LLM. O resultado é um dashboard JSON estruturado, pronto para consumo por ferramentas de visualização ou pipelines de notificação.

### Fluxo de Dados

```
Jira REST API v3  →  Python (cálculo de métricas)  →  Langflow (orquestração)  →  LLM (recomendações)  →  JSON Output
```

### Stack Tecnológica

| Componente         | Tecnologia                        |
|---------------------|-----------------------------------|
| Entrada de dados    | Jira REST API v3                  |
| Processamento       | Python 3.11+                      |
| Orquestração        | Langflow 1.10                     |
| IA / Recomendações  | LLM (via Langflow)                |
| Automação opcional  | n8n                               |
| Saída               | JSON estruturado                  |

---

## 🚀 Funcionalidades

- 📊 **Extração automatizada** de issues Jira por sprint
- 📈 **Cálculo de métricas**: velocidade, burndown, throughput, lead time, cycle time
- 🧠 **Recomendações inteligentes** geradas por LLM com base nas anomalias detectadas
- 🔄 **Pipeline reutilizável** via Langflow (arrastar e soltar no canvas)
- 📤 **Saída JSON padronizada** pronta para dashboards, n8n ou webhooks
- 🔁 **Integração com n8n** para alertas automáticos (Slack, Email, Teams)

---

## 📁 Estrutura do Projeto

```
sprint-health-agent/
├── assets/
│   └── sprint-health-flow.json    # Flow Langflow exportado
├── docs/
│   └── ARCHITECTURE.md             # Documentação de arquitetura
├── flows/                          # Definições de flow Langflow
├── metrics/                        # Módulos Python de métricas
├── .env.example                    # Template de variáveis de ambiente
├── .gitignore
├── README.md
└── requirements.txt
```

---

## ⚙️ Configuração

### 1. Pré-requisitos

- Python 3.11+
- Langflow 1.10 instalado
- Acesso a uma instância Jira (Cloud ou Data Center)

### 2. Variáveis de Ambiente

Copie o arquivo de exemplo e preencha com suas credenciais:

```bash
cp .env.example .env
```

| Variável                     | Descrição                                |
|------------------------------|------------------------------------------|
| `JIRA_BASE_URL`              | URL base da instância Jira               |
| `JIRA_EMAIL`                 | Email da conta de serviço Jira          |
| `JIRA_API_TOKEN`             | Token de API Jira                        |
| `JIRA_PROJECT_KEY`           | Chave do projeto (ex: `PROJ`)            |
| `LLM_API_KEY`                | Chave de API do provedor LLM             |
| `LLM_MODEL`                  | Modelo LLM a ser utilizado               |

### 3. Instalação

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 4. Importar Flow no Langflow

1. Acesse o Langflow (`http://localhost:7860`)
2. Importe o arquivo `assets/sprint-health-flow.json`
3. Configure as credenciais nos nós de API
4. Execute o flow

---

## 📊 Métricas Calculadas

| Métrica              | Descrição                                                   |
|----------------------|-------------------------------------------------------------|
| **Velocidade**       | Story points concluídos por sprint                         |
| **Burndown**         | Progressão diária vs. planejada                             |
| **Throughput**       | Número de issues concluídas por sprint                      |
| **Lead Time**        | Tempo total da criação até a conclusão                      |
| **Cycle Time**       | Tempo em que a issue esteve ativamente em desenvolvimento   |
| **SLA Compliance**   | Percentual de issues entregues dentro do prazo              |
| **Scope Creep**      | Itens adicionados após início da sprint                     |

---

## 🔒 Segurança

⚠️ **Nunca versione credenciais.** Todas as secrets (tokens Jira, chaves de API LLM, credenciais n8n) devem ser armazenadas exclusivamente no arquivo `.env`, que está listado no `.gitignore`. Consulte `.env.example` para o template das variáveis necessárias.

- `.env` e `.env.*` → ignorados pelo Git
- `credentials*.json` e `*service-account*.json` → ignorados
- `config.yaml` → ignorado (pode conter segredos)

---

## 📄 Licença

MIT © Alan Costa · [alancosta.dev](https://alancosta.dev)
