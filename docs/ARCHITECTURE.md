# 🏗️ Arquitetura — Sprint Health Agent

Documentação técnica da arquitetura do sistema de saúde de sprint.

---

## 1. Visão Arquitetural

O Sprint Health Agent segue uma **arquitetura de pipeline ETL com IA acoplada**, onde os dados fluem unidirecionalmente da fonte (Jira) até a saída (JSON) passando por camadas de processamento e enriquecimento com LLM.

```
┌─────────────┐     ┌──────────────────┐     ┌────────────┐     ┌───────────┐
│  Jira API   │────▶│  Python Metrics  │────▶│  Langflow   │────▶│   JSON    │
│   (v3)      │     │  Processor       │     │  + LLM      │     │  Output   │
└─────────────┘     └──────────────────┘     └────────────┘     └───────────┘
                           │                        │
                           ▼                        ▼
                    ┌──────────────┐         ┌───────────┐
                    │  Cache Layer │         │  n8n      │
                    │  (opcional)  │         │  Webhook  │
                    └──────────────┘         └───────────┘
```

---

## 2. Componentes

### 2.1 Jira Input Module (`jira_client.py`)

**Responsabilidade:** Extrair issues de sprints ativas/finalizadas via Jira REST API v3.

- Autenticação: Basic Auth (email + API token)
- Endpoints utilizados:
  - `GET /rest/agile/1.0/board/{boardId}/sprint` — listar sprints
  - `GET /rest/agile/1.0/sprint/{sprintId}/issue` — issues da sprint
  - `GET /rest/api/3/search` — busca JQL customizada
- Paginação: cursors com `startAt` e `maxResults`
- Rate limiting: delay adaptativo (429 retry com `Retry-After`)
- Cache: TTL de 5 minutos para reduzir chamadas repetidas

```
┌─────────────────────────────────────────┐
│            JiraClient                    │
├─────────────────────────────────────────┤
│ + fetch_sprints(board_id) → List[Sprint]│
│ + fetch_issues(sprint_id) → List[Issue] │
│ + fetch_changelog(issue_key) → History  │
│ - _handle_rate_limit() → void           │
│ - _cache_key(endpoint) → str            │
└─────────────────────────────────────────┘
```

### 2.2 Metrics Processor (`metrics/`)

**Responsabilidade:** Calcular métricas de saúde a partir dos dados brutos do Jira.

Módulos independentes:

| Módulo                | Métricas                                 |
|-----------------------|------------------------------------------|
| `velocity.py`         | Velocidade (story points/sprint)         |
| `burndown.py`         | Burndown diário, ideal vs. real          |
| `throughput.py`       | Issues concluídas, throughput acumulado  |
| `lead_cycle_time.py`  | Lead time, cycle time, histograma        |
| `sla.py`              | SLA compliance por tipo de issue         |
| `scope_creep.py`      | Itens adicionados pós-início da sprint   |

Cada módulo expõe uma interface comum:

```python
def compute(sprint_data: dict) -> MetricResult:
    ...
```

### 2.3 Langflow Orchestrator

**Responsabilidade:** Orquestrar o pipeline e invocar o LLM para recomendações.

- Flow principal: `sprint-health-flow.json` (importável no Langflow 1.10)
- Nós do flow:
  1. **API Request Node** → Chama o Metrics Processor
  2. **Python Function Node** → Formata métricas para o prompt
  3. **LLM Node** → Gera recomendações textuais
  4. **JSON Builder Node** → Estrutura a saída final
- Configuração: parâmetros do LLM (temperatura, max_tokens) ajustáveis no canvas

### 2.4 Output Schema

```json
{
  "sprint": {
    "id": 42,
    "name": "Sprint 42",
    "start_date": "2026-06-15",
    "end_date": "2026-06-29",
    "status": "active"
  },
  "metrics": {
    "velocity": { "committed": 34, "completed": 28, "completion_rate": 0.82 },
    "burndown": { "days_elapsed": 10, "ideal_remaining": 14, "actual_remaining": 18 },
    "throughput": { "total_issues": 15, "completed": 12 },
    "lead_time_avg_hours": 72.5,
    "cycle_time_avg_hours": 18.3,
    "sla_compliance_pct": 85.0,
    "scope_creep_count": 3
  },
  "recommendations": [
    "O scope creep de 3 itens está impactando a velocidade...",
    "Sugere-se revisar a estimativa das issues de alta complexidade..."
  ],
  "generated_at": "2026-06-25T10:30:00Z"
}
```

---

## 3. Integração com n8n (opcional)

O JSON de saída pode ser enviado para um webhook do n8n que dispara notificações:

```
JSON Output → n8n Webhook → [Condição: SLA < 80%] → Slack Alert
                          → [Sempre] → Google Sheets (log histórico)
                          → [Sexta-feira] → Email Report
```

---

## 4. Fluxo de Execução (Diagrama de Sequência)

```
Jira API        Metrics Proc.       Langflow          LLM           n8n
  │                  │                  │               │             │
  │──fetch_sprints──▶│                  │               │             │
  │───[issues]──────▶│                  │               │             │
  │                  │──compute()──┐    │               │             │
  │                  │◀──metrics───┘    │               │             │
  │                  │──POST /metrics──▶│               │             │
  │                  │                  │──prompt──────▶│             │
  │                  │                  │◀─recommend────│             │
  │                  │                  │──assemble─────│             │
  │                  │◀───JSON──────────│               │             │
  │                  │                  │──POST /webhook─────────────▶│
  │                  │                  │               │  ──notify──▶│
```

---

## 5. Considerações Técnicas

### 5.1 Tratamento de Erros

- Jira API indisponível → retry exponencial (3 tentativas, backoff 1s/2s/4s)
- LLM timeout → fallback para recomendações baseadas em regras (sem IA)
- Dados insuficientes → métricas retornam `null` com `status: "insufficient_data"`

### 5.2 Performance

- Cache em memória (TTL 5 min) para reduzir chamadas ao Jira
- Processamento de métricas paralelizável por sprint (ThreadPoolExecutor)
- JSON de saída compactado se > 100KB

### 5.3 Extensibilidade

- Adicionar nova métrica: criar módulo em `metrics/` e registrá-lo no flow Langflow
- Trocar provedor LLM: alterar nó LLM no canvas do Langflow
- Nova fonte de dados: criar novo `Input Module` seguindo a interface de `jira_client.py`

---

## 6. Segurança

| Camada              | Medida                                    |
|---------------------|-------------------------------------------|
| Transporte          | HTTPS para Jira API e LLM API             |
| Autenticação        | API tokens, nunca credenciais no código   |
| Secrets             | `.env` (não versionado)                   |
| Logs                | Sanitizados — sem tokens ou PII           |
| Langflow            | Roda em rede interna, não exposto         |
