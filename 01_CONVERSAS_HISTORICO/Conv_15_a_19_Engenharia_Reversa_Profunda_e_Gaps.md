---
title: Conversas 15 a 19 — Engenharia Reversa Profunda (Fundamentus, Oceans14, Análise de Ações, ETFs Brasil)
tags: [fin, historico, rea, fundamentus, oceans14, analise-de-acoes, etfs-brasil, valuation]
created: 2026-10-28
conversations: [dd314499-9397-40f8-a63c-56c39ff51425, d2e28710-9181-45a8-9a0a-27383b7e0340, 36704987-c39b-4e93-8b6b-b8394c62ee99, 2409a472-eb6d-4bb8-9a25-cf2535493318, b1005280-f2f1-4d60-b15c-5eb1b017183b]
---

# 🔬 Conversas 15 a 19: Engenharia Reversa Profunda (Fundamentus, Oceans14, Análise de Ações, ETFs Brasil)

Este documento reúne a engenharia reversa especializada, a descoberta de rotas e dados de 4 grandes portais de mercado e a implementação de modelos quantitativos de valuation nas quatro camadas da stack Fin.

---

## 📌 Conversa 15: Engenharia Reversa do Fundamentus, Linhagem Upstream e Pentest
- **Conversation ID:** `dd314499-9397-40f8-a63c-56c39ff51425`
- **Data:** 28/10/2026
- **Artefatos Gerados:**
  - `fundamentus_rea_gap_analysis.md` (58.317 bytes)
  - `fundamentus_upstream_data_lineage_and_fin_ingestion_guide.md` (14.540 bytes)
  - `pentest_and_implementation_report.md` (12.099 bytes)
  - `ecosystem_features_gap_analysis.md` (13.162 bytes)

### 🎯 Objetivo do Usuário
Realizar engenharia reversa minuciosa de todas as rotas do portal **Fundamentus** (`fundamentus.com.br`), rastrear a linhagem upstream regulatória (de onde o Fundamentus obtém cada número) e realizar auditoria de pentest de segurança.

### ⚙️ Descobertas Técnicas da Engenharia Reversa
1. **Mapeamento Exaustivo de 13 Rotas:**
   - Detalhes de ações: `/detalhes.php?papel={ticker}`
   - Balanços históricos: `/balancos.php?papel={ticker}`
   - DRE completa: `/demonstracoes.php?papel={ticker}`
   - Proventos históricos: `/proventos.php?papel={ticker}`
   - Screener geral de ações: `/resultado.php`
   - Screener geral de FIIs: `/fii_resultado.php`
   - Portfólio de imóveis físicos de FIIs: `/fii_imoveis.php`
   - Tabela de cotações históricas diárias: `/cotacoes.php?papel={ticker}`
   - **Descoberta Crítica — Endpoint Oculto de Cotações:** Descoberto o script legado `/amline/cot_hist.php` que cospe séries históricas de cotações em JSON compactado, permitindo captura $O(1)$ sem scraping de HTML.
2. **Linhagem Oficial dos Dados (Upstream Lineage):**
   - **B3:** Cotações diárias, volume financeiro, negócios, número total de ações, free float.
   - **CVM (Comissão de Valores Mobiliários):** Balanços Patrimoniais (BPA/BPP), DREs, DFCs e DVA via arquivos padronizados `DFP` (anual) e `ITR` (trimestral).
   - **IBGE / BCB:** Índices de inflação para deflacionamento de séries históricas.
3. **Tratamento Especial: Bancos vs Empresas Industriais:**
   - Bancos e seguradoras não possuem Ebitda, Margem Bruta nem Estoques. Utilizam Margem Financeira Bruta, Resultado de Intermediação Financeira e PDD (Provisão para Devedores Duvidosos). O schema do `fin-db` foi adaptado para comportar essa bifurcação contábil via `analytics.demonstracao_financeira`.
4. **Pentest de Segurança Realizado:**
   - Teste contra injeção de SQL (`' OR '1'='1`), sanitização de tags HTML/XSS nos campos de busca, verificação de cabeçalhos de proteção (HSTS, CSP, X-Frame-Options, X-Content-Type-Options) na API e no Frontend.

---

## 📌 Conversa 16: Execução Integrada de Serviços e Túnel Cloudflare
- **Conversation ID:** `d2e28710-9181-45a8-9a0a-27383b7e0340`
- **Data:** 28/10/2026
- **Serviços Ativos:** `fin-web` (3000), `fin-api` (8000), `fin-infra` (Cloudflare Tunnels), Neon DB

### ⚙️ Execução e Testes
1. Subida simultânea da stack local integrada.
2. Conexão da API ao banco de produção Neon Serverless.
3. Abertura de túneis públicos seguros via Cloudflare:
   - Frontend web: `https://combine-special-reflects-blackjack.trycloudflare.com`
   - API REST: `https://cubic-dog-photographs-investigator.trycloudflare.com`
4. Validação de chamadas HTTP reais entre frontend e backend sem bloqueio por CORS.

---

## 📌 Conversa 17: Engenharia Reversa do Oceans14 e Modelos de Valuation Quantitativo
- **Conversation ID:** `36704987-c39b-4e93-8b6b-b8394c62ee99`
- **Data:** 28/10/2026
- **Repositórios:** Todos os 4 repositórios (`fin-db`, `fin-ingestion`, `fin-api`, `fin-web`)

### 🎯 Objetivo do Usuário
Realizar o REA do portal Oceans14 (`oceans14.com.br`), famoso por seus balanços históricos de 10 anos, radar fundamentalista e modelos matemáticos de valuation.

### 📐 Modelos Matemáticos Implementados
1. **Fórmula de Benjamin Graham:**
   $$V = \sqrt{22.5 \times \text{LPA} \times \text{VPA}}$$
   Onde LPA é o Lucro por Ação e VPA é o Valor Patrimonial por Ação. Margem de segurança de Graham: $\text{Margem} = \frac{V - P_{\text{atual}}}{V} \times 100\%$.
2. **Método Décio Bazin (Preço Teto de Dividendos):**
   $$P_{\text{teto}} = \frac{\text{Dividendo Médio dos Últimos 3 Anos}}{0.06}$$
   Garantindo um Dividend Yield mínimo esperado de 6% a.a.
3. **Fluxo de Caixa Descontado (DCF):**
   - Projeção de Fluxo de Caixa Livre da Empresa (FCFF) para 5 anos.
   - Cálculo da taxa de desconto através do WACC (Custo Médio Ponderado de Capital):
     $$\text{WACC} = \left(\frac{E}{V} \times K_e\right) + \left(\frac{D}{V} \times K_d \times (1 - T)\right)$$
   - Valor Terminal calculado pelo modelo de perpetuidade de Gordon com taxa de crescimento perpétuo $g$ entre 2% e 3.5%.
4. **Fechamento de Gaps em Código:**
   - **`fin-db`:** Migration 10 (`10_oceans14_coverage_v3.8.sql`) adicionando tabelas de valuation quantitativo e balanço de 10 anos.
   - **`fin-api`:** Endpoints em `/api/v1/valuation/{ticker}` retornando o valor intrínseco calculado pelos diferentes modelos.
   - **`fin-web`:** Tela interativa de simulação de valuation permitindo ao investidor alterar as premissas (taxa de desconto, crescimento e margem de segurança).

---

## 📌 Conversa 18: Engenharia Reversa do Análise de Ações (`analisedeacoes.com`)
- **Conversation ID:** `2409a472-eb6d-4bb8-9a25-cf2535493318`
- **Data:** 28/10/2026
- **Artefato Gerado:** `analise_de_acoes_full_coverage_report.md` (10.990 bytes)

### ⚙️ Recursos Implementados
1. **Consenso de Mercado de Analistas:**
   - Coleta e agregação de recomendações de bancos e corretoras (Compra Forte, Compra, Manter, Venda), preço-alvo médio (*target price*) e upside projetado.
2. **Projeções Futuras de Dividendos:**
   - Algoritmo de projeção de proventos futuros com base no payout médio histórico e nas estimativas de lucro líquido do consenso de mercado.
3. **Score de Saúde Financeira do Ativo:**
   - Algoritmo ponderado (0 a 100) avaliando 5 pilares: Rentabilidade (ROE, ROIC), Endividamento (Dívida Líquida / Ebitda), Crescimento (CAGR Receita/Lucro), Eficiência (Margem Líquida) e Valuation (P/L, P/VP).
4. **Banco de Dados:** Criação da Migration 11 (`11_analise_de_acoes_coverage_v3.9.sql`).

---

## 📌 Conversa 19: Engenharia Reversa e Cobertura Completa de ETFs Brasil
- **Conversation ID:** `b1005280-f2f1-4d60-b15c-5eb1b017183b`
- **Data:** 28/10/2026
- **Artefato Gerado:** `plan_etfs_brasil_full_implementation.md` (5.809 bytes)

### ⚙️ Engenharia de ETFs Implementada
1. **Composição e Pesos de Índices:**
   - Mapeamento da carteira teórica de todos os ETFs listados na B3 (ex: `BOVA11`, `SMAL11`, `IVVB11`, `HASH11`, `XFIX11`).
   - Tabela de decomposição relacional para exibir o percentual de cada ação ou criptoativo que compõe a cota do ETF.
2. **Métricas de Rastreio (Tracking Error & Tracking Difference):**
   - Cálculo diário do desvio entre a variação patrimonial do ETF e a variação do seu índice de referência (Benchmark).
3. **Banco de Dados:** Provisionamento da Migration 12 (`12_etfs_brasil_coverage_v4.0.sql`).
4. **Git Push:** Sincronização e envio das alterações para os 5 repositórios remotos no GitHub.
