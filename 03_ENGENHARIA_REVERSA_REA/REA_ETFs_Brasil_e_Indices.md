---
title: REA ETFs Brasil — Decomposição de Cestas, Tracking Error e Índices Subjacentes
tags: [fin, rea, etfs, indices, b3, tracking-error, composicao-carteira]
created: 2026-10-28
description: Engenharia reversa de fundos de índice (ETFs), carteiras teóricas, métricas de aderência e custos operacionais.
---

# 🌐 REA ETFs Brasil: Decomposição de Cestas, Tracking Error e Índices

Os fundos de índice (ETFs - *Exchange Traded Funds*) revolucionaram os investimentos passivos no Brasil e no mundo. A análise do portal **ETFs Brasil** (`etfsbrasil.com.br`) na Conversa 19 resultou na modelagem completa desse mercado no Fin.

---

## 🏛️ Desafios na Modelagem de ETFs

Diferente de uma ação individual (que representa a participação em uma empresa), um ETF é um condomínio aberto cujas cotas refletem uma cesta de ativos com pesos dinâmicos ajustados periodicamente (rebalanceamento trimestral/quadrimestral).

Os três elementos críticos modelados pelo Fin são:
1. **Composição da Cesta Teórica (Holdings):** Conhecer exatamente quais ativos compõem a cota e o peso percentual de cada um.
2. **Aderência ao Índice (Tracking Error & Tracking Difference):** Medir com rigor se o gestor do ETF está conseguindo replicar fielmente o índice de referência.
3. **Custos Invisíveis:** Taxa de administração, spread de compra/venda na B3 e eficiência tributária (ex: tributação de dividendos em ETFs internacionais).

---

## 📐 Métricas Matemáticas de Rastreio de ETFs

### 1. Tracking Difference (Diferença de Rastreio)
Mede a diferença absoluta entre o retorno acumulado do ETF ($R_{\text{ETF}}$) e o retorno acumulado do seu Benchmark ($R_{\text{Index}}$) em um período $T$:

$$\text{Tracking Difference}_T = R_{\text{ETF}, T} - R_{\text{Index}, T}$$

> *Nota:* O Tracking Difference é tipicamente negativo devido à dedução da taxa de administração e custos de transação do fundo.

### 2. Tracking Error (Erro de Rastreio)
Mede a volatilidade (desvio padrão amostral) das diferenças diárias de retorno entre o ETF e o índice. Representa a consistência da réplica:

$$\text{Tracking Error} = \sqrt{\frac{1}{N - 1} \sum_{t=1}^{N} \left(\Delta_t - \bar{\Delta}\right)^2} \times \sqrt{252}$$

Onde:
- $\Delta_t = R_{\text{ETF}, t} - R_{\text{Index}, t}$ é a diferença de retorno no dia $t$.
- $\bar{\Delta}$ é a média das diferenças diárias.
- $\sqrt{252}$ é o fator de anualização para dias úteis de pregão.
- Quanto menor o Tracking Error, melhor a qualidade técnica da gestão do ETF.

---

## 🗄️ Estrutura de Banco de Dados (Migration 12)

```sql
-- Tabela de Metadados do ETF
CREATE TABLE IF NOT EXISTS asset_details.detalhe_etf (
    instrument_id UUID PRIMARY KEY REFERENCES canonical.instrument(instrument_id),
    indice_referencia VARCHAR(32) NOT NULL, -- IBOVESPA, IFIX, SP500, SMALL, SMLL
    gestor VARCHAR(128) NOT NULL,          -- BlackRock, Itaú, BTG Pactual, Investo
    administrador VARCHAR(128) NOT NULL,
    taxa_administracao_pct NUMERIC(5, 2) NOT NULL,
    taxa_performance_pct NUMERIC(5, 2) DEFAULT 0.0,
    frequencia_rebalanceamento VARCHAR(32) DEFAULT 'QUADRIMESTRAL',
    moeda_exposicao CHAR(3) NOT NULL DEFAULT 'BRL',
    possui_hedge_cambial BOOLEAN NOT NULL DEFAULT FALSE,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp()
);

-- Tabela Associativa N:N de Composição da Carteira Teórica
CREATE TABLE IF NOT EXISTS asset_details.composicao_etf (
    composicao_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    etf_instrument_id UUID NOT NULL REFERENCES canonical.instrument(instrument_id),
    ativo_componente_id UUID NOT NULL REFERENCES canonical.instrument(instrument_id),
    data_posicao DATE NOT NULL,
    peso_percentual NUMERIC(7, 4) NOT NULL CHECK (peso_percentual > 0 AND peso_percentual <= 100),
    quantidade_acoes NUMERIC(18, 4),
    criado_em TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp(),
    UNIQUE (etf_instrument_id, ativo_componente_id, data_posicao)
);

-- Tabela de Histórico de Tracking Error
CREATE TABLE IF NOT EXISTS asset_details.tracking_etf (
    tracking_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instrument_id UUID NOT NULL REFERENCES canonical.instrument(instrument_id),
    data_calculo DATE NOT NULL,
    periodo_dias INT NOT NULL DEFAULT 252,
    tracking_difference_pct NUMERIC(7, 4),
    tracking_error_anualizado_pct NUMERIC(7, 4),
    criado_em TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp(),
    UNIQUE (instrument_id, data_calculo, periodo_dias)
);
```

---

## 🔗 Referências Cruzadas
- [[Matriz_Unificada_12_Portais_e_Gaps|Matriz Unificada dos 12 Portais]]
- [[Conv_15_a_19_Engenharia_Reversa_Profunda_e_Gaps|Conversa 19: REA ETFs Brasil]]
- [[Catalogo_1242_Ativos_e_Chaves_Naturais|Catálogo de Ativos B3]]
