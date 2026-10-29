# Plano de Implementação — Backfill Histórico de 30 Anos e Ingestão Diária às 18h

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implementar e executar a carga histórica completa de 30 anos (desde a data do IPO do ativo) de cotações diárias (OHLCV), proventos (dividendos/JCP/rendimentos) e desdobramentos no PostgreSQL 18/TimescaleDB (`fin-db`) em 2 etapas (Etapa 1: ~150 ativos principais para validação rápida; Etapa 2: universo completo de 1.200+ ativos), garantindo zero duplicação via constraints únicas e configurando a rotina autônoma de ingestão diária às 18:00 BRT via GitHub Actions no `fin-ingestion`.

**Architecture:** O módulo `fin_ingestion.pipelines.historical_backfill` consulta os ativos em `core.ativo`, extrai a série temporal completa desde o primeiro dia de negociação (`period1=0&period2=now`), identifica e atualiza a data do IPO (`core.ativo.data_inicio_negociacao`), e realiza upserts atômicos em lote com idempotência garantida via cláusulas `ON CONFLICT DO UPDATE/NOTHING` em `market.cotacao_diaria`, `market.provento` e `market.desdobramento_split`. O workflow do GitHub Actions em `fin-ingestion` é configurado com gatilhos manuais para as etapas 1 e 2 e agendamento diário pontual às 18:00 BRT (21:00 UTC).

**Tech Stack:** Python 3.10+, uv, asyncpg, httpx/urllib, PostgreSQL 18, TimescaleDB, dbt-core, GitHub Actions YAML.

**Spec:** Baseado na especificação REA consolidada e contratos de dados em `fin-docs/03_ENGENHARIA_REVERSA_REA/` e `fin-ingestion/docs/REA_SPECIFICATION.md`.

---

## Global Constraints

- **Zero Duplicação:** Nenhuma linha de cotação ou provento pode ser duplicada no banco. Utilizar exclusivamente constraints nativas (`ON CONFLICT (id_ativo, data_pregao)` em `cotacao_diaria`, `ON CONFLICT (id_ativo, tipo_provento, data_com, data_pagamento)` em `provento` e `ON CONFLICT (id_ativo, data_ex, tipo)` em `desdobramento_split`).
- **Respeito à Data do IPO:** A data inicial de cada ativo deve refletir a primeira observação de mercado real (`min(data_pregao)`), gravando em `core.ativo.data_inicio_negociacao` e `core.ativo.data_ipo`.
- **Ownership Exclusivo:** Toda a lógica de extração histórica e agendamento diário pertence ao repositório `fin-ingestion`.
- **Horário do Pregão:** Rotina diária agendada para 18:00 BRT (21:00 UTC nos dias úteis de pregão B3, Seg-Sex).
- **Cobertura de Testes:** Todas as rotinas de parsing, detecção de IPO e geração de queries de upsert devem possuir testes unitários automatizados passando via `uv run pytest`.

## Review Focus

1. **Idempotência de Proventos com mesma data e valores distintos:** Garantir que proventos de tipos diferentes (ex.: DIVIDENDO e JCP) na mesma `data_com` não se sobrescrevam.
2. **Tratamento de Tickers Delistados ou Inválidos:** Tratar respostas vazias ou erros 404 sem derrubar o lote em execução.
3. **Estabilidade de Conexão no Neon Pooler:** Utilizar lotes controlados com semáforo de concorrência (`asyncio.Semaphore(8)`) para não estourar conexões no pooler.
4. **Precisão Numérica de Preços e Proventos:** Converter valores de ponto flutuante para `Decimal(str(round(v, 4)))` para evitar dízimas periódicas.
5. **Ajuste de Timezone do Pregão B3:** Tratar timestamps UTC convertendo estritamente para `datetime.date` local do pregão.

---

## Estrutura de Arquivos

```
fin-ingestion/
├── fin_ingestion/
│   ├── pipelines/
│   │   ├── __init__.py
│   │   └── historical_backfill.py      # [Novo] Motor de backfill histórico de 30 anos e detecção de IPO
│   ├── cli.py                          # [Modificado] Novo comando typer 'backfill-history'
├── .github/
│   └── workflows/
│       └── scheduled-ingestion.yml     # [Modificado] Steps de backfill Stage 1/2 e cron diário das 18h BRT
├── tests/
│   └── test_historical_backfill.py     # [Novo] Testes unitários de parsing, detecção de IPO e idempotência
```

---

## Tarefas de Implementação

### Task 1: Motor de Backfill Histórico de 30 Anos (`HistoricalBackfillPipeline`)

**Files:**
- Create: `fin-ingestion/fin_ingestion/pipelines/historical_backfill.py`
- Test: `fin-ingestion/tests/test_historical_backfill.py`

- [ ] **Step 1:** Escrever teste unitário `test_historical_backfill.py` validando o parsing de candles históricos, identificação da data do IPO (`min_date`), normalização de proventos (DIVIDENDO/JCP/RENDIMENTO), splits e construção dos registros para upsert.
- [ ] **Step 2:** Executar `uv run pytest tests/test_historical_backfill.py` e verificar que falha (TDD).
- [ ] **Step 3:** Implementar a classe `HistoricalBackfillPipeline` em `fin_ingestion/pipelines/historical_backfill.py`:
  - Método `get_stage_universe(stage: int) -> list[dict]`: Filtra Stage 1 (top ~150 ativos por liquidez e componentes IBOV/IFIX) e Stage 2 (restante dos ativos).
  - Método `fetch_complete_history(ticker: str, tipo_ativo: str)`: Consulta série máxima (`period1=0&period2=now&events=div|split`).
  - Método `persist_asset_history(conn, asset_data, candles, divs, splits)`: Executa queries atômicas com `ON CONFLICT DO UPDATE`.
  - Atualização cadastral em `core.ativo`: Grava `data_inicio_negociacao` com a data do primeiro pregão observado.
- [ ] **Step 4:** Rodar `uv run pytest tests/test_historical_backfill.py` e garantir 100% de aprovação.

---

### Task 2: Integração com o CLI do `fin-ingestion`

**Files:**
- Modify: `fin-ingestion/fin_ingestion/cli.py`

- [ ] **Step 1:** Adicionar o comando `@app.command() def backfill_history(...)` no `cli.py` suportando as flags `--stage` (1 ou 2), `--tickers` (opcional), `--concurrency` (default 8) e `--batch-size` (default 50).
- [ ] **Step 2:** Adicionar formatação de saída com `rich.table.Table` exibindo: Ticker, IPO Detectado, Período Coberto, Candles Inseridos, Proventos e Desdobramentos.
- [ ] **Step 3:** Testar o comando via CLI localmente para 2 ativos de controle (ex.: `PETR4` e `HGLG11`):
  `uv run fin-ingestion backfill-history --tickers PETR4,HGLG11`
- [ ] **Step 4:** Validar no banco Neon que as cotações de 2000 em diante foram inseridas sem duplicações:
  `SELECT min(data_pregao), count(*) FROM market.cotacao_diaria JOIN core.ativo USING(id_ativo) WHERE ticker = 'PETR4';`

---

### Task 3: Automação via GitHub Actions e Rotina Diária às 18h BRT

**Files:**
- Modify: `fin-ingestion/.github/workflows/scheduled-ingestion.yml`

- [ ] **Step 1:** Atualizar os gatilhos manuais (`workflow_dispatch.inputs.action`) para incluir as opções:
  - `backfill_stage1` (Validação rápida dos ~150 ativos principais)
  - `backfill_stage2` (Carga dos ~1.200 ativos restantes)
- [ ] **Step 2:** Configurar o agendamento `schedule` diário de fechamento para as **18:00 BRT** (que equivale a **21:00 UTC**):
  `- cron: '0 21 * * 1-5'`
- [ ] **Step 3:** Garantir que o step pós-fechamento diário execute:
  1. `uv run fin-ingestion collect --pipeline daily`
  2. `uv run fin-ingestion compute-analytics`
  3. `uv run fin-ingestion dbt-run --models staging intermediate marts`
- [ ] **Step 4:** Validar a sintaxe do workflow com linter e conferir steps de execução.

---

### Task 4: Execução da Etapa 1 e Auditoria de Qualidade

**Files:**
- Execution: Terminal / CLI em `fin-ingestion`

- [ ] **Step 1:** Disparar a Etapa 1 para o universo de validação rápida (~150 ativos IBOV/IFIX).
- [ ] **Step 2:** Executar script de auditoria SQL no banco Neon verificando:
  - Zero duplicidade em `market.cotacao_diaria` e `market.provento`.
  - Presença de dados históricos retroativos a partir da data de IPO de cada ativo.
  - Atualização dos campos `data_inicio_negociacao` e `data_ipo` em `core.ativo`.
- [ ] **Step 3:** Emitir relatório final com volumetria consolidada e instruções para disparo da Etapa 2 via GitHub Actions.
