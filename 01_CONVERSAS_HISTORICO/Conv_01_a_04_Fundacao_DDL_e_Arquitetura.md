---
title: Conversas 01 a 04 — Fundação de Dados, DDL Neon e Arquitetura Macro
tags: [fin, historico, fin-db, postgresql, timescaledb, ddl, arquitetura]
created: 2026-10-28
conversations: [79731967-b954-41e2-b2f6-29e5509b4a3b, 8db7512b-60d3-47e2-b384-bca9322be4f9, 032ae62b-7737-4a09-9c26-4c9071245c9d, fa1c8cee-0c06-4015-85ec-e8dd4aba50d0]
---

# 🏛️ Conversas 01 a 04: Fundação de Dados, DDL Neon e Arquitetura Macro

Este documento registra as decisões, códigos, esquemas e modelos implementados durante a fase fundacional do ecossistema Fin.

---

## 📌 Conversa 01: Execução do DDL Inicial no Neon PostgreSQL e Organização dos Scripts
- **Conversation ID:** `79731967-b954-41e2-b2f6-29e5509b4a3b`
- **Data:** 26/10/2026
- **Ambiente:** PostgreSQL 18.6 (Neon Serverless) • TimescaleDB (Apache 2.0) • `pgvector` • `pg_trgm` • `pgcrypto`
- **Repositório:** `fin-db` (inicialmente `fin`)

### 🎯 Objetivo do Usuário
Executar os arquivos DDL anexados (`diyfin.md`, `melhorias_db.md` v3.1 e `melhorias_db2.md` v3.2) diretamente no banco de dados Neon, adaptando-os para compatibilidade com o PostgreSQL Serverless, e organizar os scripts SQL no repositório.

### ⚙️ Execução Técnica e Resultados
1. **Provisionamento de Schemas:** Criação e estruturação de **12 schemas modulares**:
   - `canonical`, `raw`, `core`, `market`, `asset_details`, `analytics`, `wealth`, `social`, `dw`, `system`, `audit`, `public`.
2. **Volumetria de Objetos:** Foram criadas **273 tabelas e views**.
3. **Séries Temporais (Hypertables):** Provisionamento de **36 hypertables** TimescaleDB, particionadas por tempo (intervalos de 7 dias a 1 mês):
   - Cotações diárias/intraday (`market.cotacao_diaria`, `market.cotacao_intraday`), livros de ofertas (`market.book_ordens`), macroeconomia (`market.serie_macroeconomica`), vacância e patrimônio de FIIs (`asset_details.historico_vacancia_fii`, `asset_details.historico_patrimonio_fii`), logs de auditoria e rate limiting.
4. **Views Materializadas:** `analytics.busca_ativo`, `analytics.cagg_cotacao_mensal`, `analytics.cagg_provento_mensal`.
5. **Adaptações Neon Serverless:**
   - Como o Neon não permite continuous aggregates proprietários do TimescaleDB corporativo, foram criadas **Materialized Views nativas do PostgreSQL** com triggers e rotinas de refresh incremental.

---

## 📌 Conversa 02: Expansão do Schema Fin-DB e Plano de Migração
- **Conversation ID:** `8db7512b-60d3-47e2-b384-bca9322be4f9`
- **Data:** 26/10/2026
- **Repositório:** `fin-db`

### 🎯 Objetivo do Usuário
Criar um plano formal de migração incremental e executar os arquivos de extensão no Neon DB, garantindo rastreabilidade, integridade referencial e idempotência estrita.

### ⚙️ Execução Técnica e Resultados
1. **Criação do Plano de Migração:** Registro em artefato estruturado (`task.md`).
2. **Evolução de Objetos:** A contagem de objetos subiu de **273** para **282** (+9 objetos: 7 novas tabelas, 1 associativa N:N, 1 view analítica).
3. **Idempotência Estrita:** Todos os scripts foram revisados para conter `CREATE TABLE IF NOT EXISTS`, `ADD COLUMN IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS` e blocos `DO $$ ... $$` com tratamento de exceções.
4. **Tax Lots & Controle Fiscal:**
   - Criação da tabela `core.tax_lot` para cálculo de preço médio ponderado e regras de isenção de IR (R$ 20.000 para ações no swing trade; 0% de isenção em FIIs e day trade).
5. **Integridade Referencial Validada:**
   - Chaves estrangeiras ligadas a `canonical.instrument(instrument_id)`, `canonical.government_bond(instrument_id)`, `canonical.legal_entity(legal_entity_id)` e `raw.payload(payload_id)`.

---

## 📌 Conversa 03: Engenharia Reversa (REA) dos 12 Portais e Expansão v3.4
- **Conversation ID:** `032ae62b-7737-4a09-9c26-4c9071245c9d`
- **Data:** 26/10/2026 – 27/10/2026
- **Repositório:** `fin-db`

### 🎯 Objetivo do Usuário
Realizar a Engenharia Reversa de Aplicação (REA) ao vivo dos 12 principais portais de investimentos do Brasil:
1. Status Invest
2. Investidor10
3. Fundamentei
4. Fundamentus
5. Clube FII
6. Funds Explorer
7. FIIs.com.br
8. Mais Retorno
9. Oceans14
10. Análise de Ações
11. ADVFN Brasil
12. ETFs Brasil

### ⚙️ Execução Técnica e Descobertas
1. **Fundamentei:** Inspeção do estado global do Apollo Client (GraphQL). Descoberta e catalogação de **102 indicadores financeiros e operacionais**, histórico de 15 anos de dividendos e DRE padronizada.
2. **Investidor10:** Mapeamento de receitas por segmento de negócio e distribuição geográfica (Brasil vs. Exterior), métricas de endividamento e rankings setoriais.
3. **Status Invest:** Extração de contratos de derivativos, gregas de opções (Delta, Gamma, Vega, Theta), remuneração de diretoria e conselho (FRE CVM) e histórico de recompra de ações (*buybacks*).
4. **Clube FII & Funds Explorer:** Métricas especializadas de FIIs: vacância física vs financeira, cap rate médio, área bruta locável (ABL), contratos atípicos vs típicos e dados de imóveis físicos da SiiLA.
5. **Criação do Estágio `06_rea_coverage_v3.4`:**
   - Geração de 6 scripts SQL adicionando tabelas especializadas no `asset_details` e `analytics`.
   - Documentado em `docs/05_rea_12_sites_coverage_v3.4.md`.

---

## 📌 Conversa 04: Arquitetura de APIs e Estrutura dos Repositórios
- **Conversation ID:** `fa1c8cee-0c06-4015-85ec-e8dd4aba50d0`
- **Data:** 26/10/2026 – 27/10/2026
- **Artefato Gerado:** `arquitetura_apis_fin.md` (13.858 bytes)

### 🎯 Objetivo do Usuário
Definir a topologia de repositórios e serviços do ecossistema Fin: monólito vs microsserviços, divisão de responsabilidades, contratos entre módulos e stack de APIs.

### 🏛️ Decisões Arquiteturais Fundamentais
1. **Decisão Monólito Modular vs Microserviços:**
   - Adotou-se o modelo de **Monólito Modular** para a API (`fin-api`) e separação estrita por repositórios de responsabilidade única para o ecossistema.
   - Justificativa: elimina latência de rede entre microsserviços internos, compartilha pooling de conexões do Neon PostgreSQL e simplifica transações ACID sem necessidade inicial de coordenação de duas fases (2PC).
2. **Topologia dos 5 Repositórios Oficiais:**
   - `fin-db`: PostgreSQL 18, TimescaleDB, DDLs, migrações e contratos relacionais.
   - `fin-api`: Backend FastAPI assíncrono, cache-aside no Redis, autenticação Supabase.
   - `fin-infra`: Gateway de borda, proxies reversos (Traefik/Caddy), túneis Cloudflare e IaC.
   - `fin-ingestion`: Coletores assíncronos, scrapers, pipelines Python e transformações dbt.
   - `fin-web`: Frontend Next.js 15 (App Router), React 19, Tailwind CSS e shadcn/ui.
3. **Invariante de Isolamento:**
   - A camada de ingestão (`fin-ingestion`) nunca fala com o frontend (`fin-web`).
   - A API (`fin-api`) nunca executa scrapers nem chama fontes externas durante requisições de clientes HTTP.
   - Todo dado entra via `raw.payload` no `fin-db`, é transformado e depois lido pela `fin-api`.
