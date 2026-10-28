---
title: REA Análise de Ações — Consenso de Mercado, Projeção de Dividendos e Score de Saúde
tags: [fin, rea, analise-de-acoes, consenso-analistas, preco-alvo, dividendos-futuros, score-saude]
created: 2026-10-28
description: Engenharia reversa de 22 rotas do analisedeacoes.com e implementação dos algoritmos de consenso e projeção.
---

# 🎯 REA Análise de Ações: Consenso de Mercado, Projeção de Dividendos e Score de Saúde

O portal **Análise de Ações** (`analisedeacoes.com`) possui como diferencial o foco no sentimento de mercado dos analistas institucionais (Sell-Side) e modelos de projeção de proventos futuros. Esta documentação consolida o REA realizado na Conversa 18.

---

## 🗺️ Mapeamento das 22 Rotas Alvo

O portal estrutura suas URLs em torno de objetivos analíticos claros:
- `/acoes/{ticker}/preco-alvo` — Consenso consolidado de corretoras e bancos de investimento.
- `/acoes/{ticker}/dividendos-futuros` — Projeção probabilística de proventos para os próximos 12 a 24 meses.
- `/acoes/{ticker}/score` — Nota de 0 a 100 baseada em 5 dimensões fundamentalistas.
- `/rankings/maiores-altas-potenciais` — Lista de ativos com maior *upside* em relação ao preço-alvo médio.
- `/rankings/melhores-pagadoras-dividendos` — Projeção de Dividend Yield futuro.

---

## 1. 📊 Consenso de Mercado dos Analistas

O Fin unifica e pondera as recomendações de analistas certificados (CNPI) das principais casas de análise brasileiras (BTG Pactual, Itaú BBA, XP Investimentos, Santander Corretora, Bradesco BBI, Genial, Ágora):

### A. Categorização das Recomendações:
- **Compra Forte (Strong Buy):** Potencial de valorização $> 25\%$ com baixo risco percebido.
- **Compra (Buy):** Potencial de valorização $> 15\%$.
- **Manter (Hold / Neutral):** Retorno esperado em linha com o CDI / custo de oportunidade.
- **Venda (Sell):** Ativo sobreavaliado com risco de perda de capital.
- **Venda Forte (Strong Sell):** Risco iminente de colapso de lucros ou solvência.

### B. Cálculo do Preço-Alvo Médio e Upside:
$$\text{Preço-Alvo Médio} = \frac{\sum_{i=1}^{k} \text{Target Price}_i}{k}$$

$$\text{Upside Potencial (\%)} = \left(\frac{\text{Preço-Alvo Médio} - \text{Preço Atual}}{\text{Preço Atual}}\right) \times 100\%$$

---

## 2. 🔮 Modelo de Projeção de Proventos Futuros

Em vez de olhar apenas para o retrovisor, o modelo projeta o dividendo futuro ($D_{\text{proj}}$) com base na estimativa de lucro líquido do consenso de mercado e na taxa histórica de distribuição (*Payout*):

$$D_{\text{proj}} = \frac{\text{Lucro Líquido Projetado (12m)} \times \overline{\text{Payout (3 anos)}}}{\text{Número de Ações em Circulação}}$$

$$\text{Dividend Yield Projetado (\%)} = \left(\frac{D_{\text{proj}}}{\text{Preço Atual da Ação}}\right) \times 100\%$$

---

## 3. 🛡️ Algoritmo do Score de Saúde Financeira (0 a 100)

O algoritmo avalia 5 pilares com pesos equilibrados (20 pontos por pilar):

| Pilar | Métricas Avaliadas | Critério de Pontuação Máxima (20 pts) |
|---|---|---|
| **1. Rentabilidade** | ROE (Return on Equity) e ROIC | $\text{ROE} \ge 15\%$ e $\text{ROIC} \ge 12\%$ |
| **2. Endividamento** | Dívida Líquida / Ebitda | $\text{Dívida Líquida / Ebitda} \le 1.5\text{x}$ (ou caixa líquido) |
| **3. Eficiência** | Margem Líquida e Margem Ebitda | $\text{Margem Líquida} \ge 10\%$ e $\text{Margem Ebitda} \ge 20\%$ |
| **4. Crescimento** | CAGR Receita (5a) e CAGR Lucros (5a) | $\text{CAGR} \ge 10\%$ ao ano |
| **5. Governança & Liquidez** | Tag Along, Segmento B3 e Volume Diário | Tag Along de 100%, Novo Mercado e Vol Diário > R$ 5 milhões |

---

## 🗄️ Estrutura de Banco de Dados (Migration 11)

```sql
CREATE TABLE IF NOT EXISTS analytics.consenso_mercado (
    consenso_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instrument_id UUID NOT NULL REFERENCES canonical.instrument(instrument_id),
    data_consenso DATE NOT NULL DEFAULT CURRENT_DATE,
    preco_alvo_medio NUMERIC(15, 4),
    preco_alvo_mediano NUMERIC(15, 4),
    preco_alvo_maximo NUMERIC(15, 4),
    preco_alvo_minimo NUMERIC(15, 4),
    upside_medio_pct NUMERIC(8, 2),
    qtd_compra_forte INT NOT NULL DEFAULT 0,
    qtd_compra INT NOT NULL DEFAULT 0,
    qtd_manter INT NOT NULL DEFAULT 0,
    qtd_venda INT NOT NULL DEFAULT 0,
    qtd_venda_forte INT NOT NULL DEFAULT 0,
    score_saude_financeira INT CHECK (score_saude_financeira BETWEEN 0 AND 100),
    dy_projetado_12m_pct NUMERIC(8, 2),
    criado_em TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp(),
    UNIQUE (instrument_id, data_consenso)
);
```

---

## 🔗 Referências Cruzadas
- [[Matriz_Unificada_12_Portais_e_Gaps|Matriz Unificada dos 12 Portais]]
- [[Conv_15_a_19_Engenharia_Reversa_Profunda_e_Gaps|Conversa 18: REA Análise de Ações]]
- [[Modelagem_Relacional_12_Schemas|Esquemas Relacionais do fin-db]]
