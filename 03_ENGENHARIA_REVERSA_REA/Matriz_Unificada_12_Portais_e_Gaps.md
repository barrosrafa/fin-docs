---
title: Matriz Unificada dos 12 Portais Financeiros & Análise de Gaps
tags: [fin, rea, matriz-funcional, 12-portais, comparativo, gap-analysis]
created: 2026-10-28
description: Levantamento exaustivo, desduplicação e mapeamento de funcionalidades dos 12 principais sites de investimentos do Brasil.
---

# 📊 Matriz Unificada dos 12 Portais Financeiros & Análise de Gaps

Este documento consolida a Engenharia Reversa de Aplicação (REA) conduzida sobre as **12 maiores plataformas de análise de investimentos do Brasil**, desduplicando suas funcionalidades em 6 domínios de negócio e estabelecendo como o **Fin** atinge paridade e superioridade técnica.

---

## 🏛️ Os 12 Portais Analisados

1. **Status Invest** (`statusinvest.com.br`) — Líder em volume de dados, gregas de opções, proventos e remuneração CVM.
2. **Investidor10** (`investidor10.com.br`) — Destaque em receitas por segmento/geografia, checklists Buy & Hold e projeções.
3. **Fundamentei** (`fundamentei.com`) — Referência em UX, histórico contábil consolidado de 15+ anos e estado GraphQL (102 KPIs).
4. **Fundamentus** (`fundamentus.com.br`) — O pioneiro do mercado brasileiro; tabelas leves, screeners rápidos e endpoints históricos.
5. **Clube FII** (`clubefii.com.br`) — O maior portal especializado em FIIs; vacância, imóveis, relatórios gerenciais e fórum.
6. **Funds Explorer** (`fundsexplorer.com.br`) — Líder em rankings e métricas visuais de Fundos Imobiliários.
7. **FIIs.com.br** (`fiis.com.br`) — Foco em emissões de cotas, subscrições, fatos relevantes e proventos de FIIs.
8. **Mais Retorno** (`maisretorno.com`) — Referência em Fundos de Investimento (CVM 175), comparador de rentabilidade e lâminas.
9. **Oceans14** (`oceans14.com.br`) — Foco em valuation quantitativo clássico (Graham, Bazin, DCF) e radares de 10 anos.
10. **Análise de Ações** (`analisedeacoes.com`) — Consenso de mercado de analistas, preços-alvo e projeções de dividendos futuros.
11. **ADVFN Brasil** (`br.advfn.com`) — Livro de ofertas em tempo real, fluxo de ordens (tape reading) e fórum comunitário.
12. **ETFs Brasil** (`etfsbrasil.com.br`) — Foco exclusivo em fundos de índice, carteiras teóricas, taxas e tracking difference.

---

## 🧭 Matriz Exaustiva de Cobertura Funcional por Domínio

### Legenda da Matriz
- ✅ **Nativo no Fin:** Modelado no banco `fin-db`, ingerido no `fin-ingestion`, exposto no `fin-api` e exibido no `fin-web`.
- 🟡 **Em Ingestão Programada:** Schema e endpoints prontos, aguardando execução de lote pelo coletor.
- ❌ **Ausente nos Portais Tradicionais:** Funcionalidade exclusiva do ecossistema Fin.

| Domínio de Negócio | Funcionalidade Específica | Portais de Origem | Cobertura no Ecossistema Fin | Tabela / Schema no `fin-db` |
|---|---|---|:---:|---|
| **1. Ações & Contábil** | Balanços Padronizados (BPA/BPP 10-20 anos) | Fundamentus, Fundamentei, Oceans14 | ✅ | `analytics.demonstracao_financeira` |
| | DRE Analítica Trimestral e Anual | Status Invest, Fundamentei | ✅ | `analytics.demonstracao_financeira` |
| | Fluxo de Caixa Livre e Operacional (DFC) | Investidor10, Oceans14 | ✅ | `analytics.demonstracao_financeira` |
| | Demonstração do Valor Adicionado (DVA) | Status Invest, Fundamentus | ✅ | `analytics.demonstracao_financeira` |
| | Receita por Segmento de Negócio e Geografia | Investidor10, Fundamentei | ✅ | `analytics.receita_segmento` |
| | Remuneração da Diretoria & Conselho (FRE CVM) | Status Invest | ✅ | `core.remuneracao_orgao` |
| | Histórico de Recompras de Ações (Buybacks) | Status Invest | ✅ | `market.programa_recompra` |
| | Diferenciação Bancos vs Indústrias | Fundamentus, Status Invest | ✅ | `analytics.demonstracao_financeira` |
| **2. Valuation & Métricas** | Preço Justo de Benjamin Graham | Oceans14, Investidor10 | ✅ | `analytics.valuation_ativo` |
| | Preço Teto de Décio Bazin (DY 6%) | Oceans14, Análise de Ações | ✅ | `analytics.valuation_ativo` |
| | Fluxo de Caixa Descontado (DCF com WACC) | Oceans14 | ✅ | `analytics.valuation_ativo` |
| | Modelo de Dividendos Descontados (Gordon) | Oceans14 | ✅ | `analytics.valuation_ativo` |
| | Consenso de Mercado & Recomendações | Análise de Ações, Status Invest | ✅ | `analytics.consenso_mercado` |
| | Checklist Automatizado de Buy & Hold | Investidor10, Fundamentei | ✅ | `analytics.checklist_criterio` |
| | Radar Fundamentalista Multidimensional | Oceans14 | ✅ | `analytics.score_eixo` |
| **3. Fundos Imobiliários** | Vacância Física vs Vacância Financeira | Clube FII, Funds Explorer | ✅ | `asset_details.historico_vacancia_fii` |
| | Portfólio de Imóveis Físicos (Endereço e ABL) | Clube FII, SiiLA, Fundamentus | ✅ | `asset_details.imovel_fii` |
| | Vencimento de Contratos (Atípicos vs Típicos) | Clube FII, Status Invest | ✅ | `asset_details.contrato_locacao_fii` |
| | Alavancagem e Obrigações por CRIs Emitidos | Clube FII | ✅ | `asset_details.alavancagem_fii` |
| | DRE Trimestral de Rendimentos e Caixa | FIIs.com.br, Clube FII | ✅ | `asset_details.dre_fii` |
| **4. Fundos de Investimento** | Lâminas e Classificação Anbima (CVM 175) | Mais Retorno, Status Invest | ✅ | `asset_details.perfil_fundo` |
| | Carteira Aberta de Ativos (CDA CVM 90 dias) | Mais Retorno | ✅ | `asset_details.composicao_carteira_fundo` |
| | Métricas de Risco (Sharpe, Sortino, VaR) | Mais Retorno | ✅ | `analytics.metrica_risco_fundo` |
| | Drawdown Histórico & Período de Recuperação | Mais Retorno | ✅ | `analytics.drawdown_fundo` |
| **5. ETFs & Globais** | Carteira Teórica e Pesos de Componentes | ETFs Brasil, Status Invest | ✅ | `asset_details.composicao_etf` |
| | Tracking Error e Tracking Difference Diário | ETFs Brasil | ✅ | `asset_details.tracking_etf` |
| | Taxa de Administração e Spread Médio | ETFs Brasil | ✅ | `asset_details.detalhe_etf` |
| **6. Renda Fixa & Títulos** | Curvas de Juros ETTJ ANBIMA (Prefixado, IPCA) | Mais Retorno, Status Invest | ✅ | `market.curva_juros_ettj` |
| | Tesouro Direto com Marcação a Mercado (MaM) | Status Invest | ✅ | `canonical.government_bond` |
| | Spreads Indicativos de Debêntures ANBIMA | Status Invest | ✅ | `canonical.debenture` |
| **7. Diferenciais Fin** | **Look-Through de Devedor Consolidado** | ❌ *Nenhum portal possui* | ✅ | `core.fn_lookthrough_devedor()` |
| | **Ledger de Partidas Dobradas (Soma Zero)** | ❌ *Nenhum portal possui* | ✅ | `core.lancamento_contabil` |
| | **Risco de Crédito Cruzado (Ação + FII + Fundo)**| ❌ *Nenhum portal possui* | ✅ | `dw.grafo_risco_credito` |
| | **Idempotência Transacional SHA-256** | ❌ *Nenhum portal possui* | ✅ | `system.idempotency_keys` |

---

## 🔗 Referências Cruzadas
- [[REA_Fundamentus_Linhagem_e_Endpoints_Ocultos|REA do Fundamentus]]
- [[REA_Oceans14_e_Modelos_Valuation|REA do Oceans14]]
- [[REA_Status_Invest_e_Investidor10|REA do Status Invest e Investidor10]]
- [[Tese_Carteira_Invisivel_Dual_Engine|A Tese da Carteira Invisível]]
