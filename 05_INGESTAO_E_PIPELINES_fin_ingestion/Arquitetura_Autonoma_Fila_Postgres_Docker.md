# Arquitetura Autônoma do fin-ingestion: Fila PostgreSQL, Workers e Docker

Este documento descreve a arquitetura de produção do motor **`fin-ingestion`**, operando como um ecossistema autônomo, transacional e desacoplado de agendadores externos como crons estáticos do GitHub Actions.

---

## 1. Visão Geral da Arquitetura

O sistema opera com o padrão **Lease / Fila Persistente no PostgreSQL** via `SELECT ... FOR UPDATE SKIP LOCKED`, garantindo:
- **Reserva Atômica e Concorrente:** Workers retiram tarefas da fila sem colisão entre instâncias.
- **Tolerância a Falhas e Recuperação de Zumbis:** Leases com expiração temporal (`locked_until`). Se um worker crashar ou o container reiniciar, o reconciliador reabre a tarefa automaticamente.
- **Deduplicação e Idempotência:** Chave única (`task_key`) que impede enfileiramento redundante.
- **Idempotência no DW:** Inserções com `COALESCE` preservam histórico detalhado contra sobreposição por dados parciais.
- **Validação Pré-Publicação:** Rejeição de inconsistências geométricas de candles OHLC e quarentena de variações extremas (> 50%).

```
┌─────────────────────────────────┐
│     Scheduler & Reconciler      │
│  (Políticas, Calendários B3)    │
└────────────────┬────────────────┘
                 │ INSERT ON CONFLICT DO NOTHING
                 ▼
┌─────────────────────────────────┐
│      ingestion.ingestion_task   │
│   (Fila PostgreSQL Persistente) │
└────────────────┬────────────────┘
                 │ FOR UPDATE SKIP LOCKED
                 ▼
┌─────────────────────────────────┐
│     Worker Cluster (Docker)     │
│  (Extração, Raw, Validação, DW) │
└────────────────┬────────────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
 ┌──────────────┐  ┌──────────────┐
 │  PostgreSQL  │  │  Quarentena  │
 │  (Canonical) │  │   (Issues)   │
 └──────────────┘  └──────────────┘
```

---

## 2. Entidades de Controle no PostgreSQL (`schema ingestion`)

Criadas na migração `14_ingestion_control_and_data_quality_v4.2`:

1. **`ingestion_policy`:** Regras de frequência por dataset, calendário (`B3_TRADING_DAYS`, `BUSINESS_DAYS`), prioridade e SLA aceitável (`expected_delay_min`).
2. **`source_policy`:** Orçamento de requisições por minuto (`requests_per_minute`), concorrência máxima por fonte e estado do **Circuit Breaker** compartilhado (`CLOSED`, `OPEN`, `HALF_OPEN`).
3. **`ingestion_task`:** Fila persistente de tarefas com `task_key`, prioridade, status (`PENDING`, `RUNNING`, `COMPLETED`, `FAILED`), lease token (`lock_token`) e prazo de reserva (`locked_until`).
4. **`ingestion_run`:** Auditoria detalhada de cada execução (duração em ms, registros extraídos, registros gravados, status HTTP, stack trace).
5. **`ingestion_checkpoint`:** Marcas d'água confirmadas (`watermark_date`, `watermark_timestamp`) por dataset e partição.
6. **`ingestion_issue`:** Registro de dados em quarentena (anomalias OHLC, quebras de schema, variações atípicas).
7. **`v_pipeline_health_summary`:** View operacional em tempo real calculando SLAs e freshness da ingestão.

---

## 3. Correções de Confiabilidade Implementadas

1. **Eliminação de Fallbacks Fictícios (`status_invest.py` / `etfs_brasil.py`):**
   - Removido o retorno de cotações estáticas hardcoded (`37.85` e `37.50` para `PETR4`) em caso de exceção de rede.
   - Fixtures são isoladas estritamente nos testes unitários via mocks.
2. **Cotação Pontual vs. OHLC Diário Real (`daily_market.py`):**
   - Preços pontuais de scraping XHR são gravados como observações em `market.cotacao_intraday` e `canonical.price_observation`.
   - Não se fabrica mais candles falsos onde abertura, máxima, mínima e fechamento eram duplicados com volume 0.
   - O universo de ativos é consultado dinamicamente no banco (`core.ativo`) com ordenação por liquidez.
3. **Eliminação de CNPJ Sintético (`enrichment.py`):**
   - Removido o gerador pseudo-aleatório baseado em `hash(ticker)` que variava a cada execução de processo (`PYTHONHASHSEED`).
   - A coluna `cnpj` em `core.emissor_companhia` tornou-se anulável na migração 14, preservando emissores internacionais e cadastros em análise com status `PENDENTE_CONFIRMACAO`.
4. **Cálculos Analíticos Reais (`cli.py compute-analytics`):**
   - Substituídos valores simulados por consultas a balanços e séries históricas reais de `market.cotacao_diaria` e `analytics.indicador_fundamentalista`.
   - Resultados de valuation (Graham, Bazin, RSI, Pivots) são persistidos em `analytics.valuation` com `formula_version` e hash de inputs.
5. **Tratamento de Códigos HTTP Críticos (`http_client.py`):**
   - **HTTP 429 Too Many Requests:** Lê o cabeçalho `Retry-After`, aciona cooldown e pausa a fonte correspondente.
   - **HTTP 401/403:** Abre o circuito imediatamente para evitar banimentos de IP e spam inútil.

---

## 4. Empacotamento Docker e Comandos Operacionais

### Serviços no Docker Compose (`docker-compose.ingestion.yml`):
- **`ingestion-scheduler`:** Executa o agendador autônomo e reconciliador a cada 60 segundos.
- **`ingestion-worker`:** Executa workers em réplicas consumindo a fila persistente via `claim_next_task()`.

### Comandos da CLI (`fin-ingestion`):
```bash
# Verificar estado da fila e cumprimento de SLAs
uv run fin-ingestion queue-status

# Disparar reconciliação de tarefas zumbis e lacunas
uv run fin-ingestion queue-reconcile

# Iniciar agendador autônomo
uv run fin-ingestion scheduler-start --interval 60

# Iniciar worker consumidor
uv run fin-ingestion worker-start --concurrency 3

# Calcular indicadores com dados reais do DW
uv run fin-ingestion compute-analytics --tickers PETR4,VALE3
```
