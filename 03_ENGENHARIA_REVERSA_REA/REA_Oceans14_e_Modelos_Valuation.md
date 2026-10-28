---
title: REA Oceans14 & Modelos Matemáticos de Valuation Quantitativo
tags: [fin, rea, oceans14, valuation, graham, bazin, dcf, gordon, financas-quantitativas]
created: 2026-10-28
description: Engenharia reversa dos modelos quantitativos do Oceans14 e implementação dos algoritmos de valuation.
---

# 📐 REA Oceans14 & Modelos Matemáticos de Valuation Quantitativo

O **Oceans14** (`oceans14.com.br`) é o portal brasileiro mais respeitado pela comunidade de investidores fundamentalistas adeptos do *Value Investing*. Esta documentação detalha a engenharia reversa de seus modelos e a implementação matemática realizada na Conversa 17.

---

## 🏛️ Os Quatro Modelos Matemáticos de Valuation

O ecossistema Fin implementou nativamente no `fin-api` e no `fin-db` (Migration 10) os 4 grandes pilares clássicos de precificação de ativos:

### 1. Fórmula de Valor Intrínseco de Benjamin Graham

Desenvolvida pelo pai do *Value Investing*, a fórmula estabelece o teto que um investidor defensivo deve pagar por uma ação que combine lucros consistentes e base patrimonial sólida:

$$V = \sqrt{22.5 \times \text{LPA} \times \text{VPA}}$$

Onde:
- $\text{LPA}$ = Lucro Líquido por Ação dos últimos 12 meses (ou média dos últimos 3 anos para suavizar ciclos).
- $\text{VPA}$ = Valor Patrimonial por Ação (Patrimônio Líquido dividido pelo número total de ações).
- A constante $22.5$ deriva da recomendação clássica de Graham: um índice P/L máximo de $15.0$ multiplicado por um índice P/VP máximo de $1.5$ ($15 \times 1.5 = 22.5$).

#### Margem de Segurança (*Margin of Safety*):
$$\text{Margem de Segurança (\%)} = \left(\frac{V - P_{\text{atual}}}{V}\right) \times 100\%$$
- Se a Margem de Segurança for superior a $20\%$, o ativo é considerado descontado.

---

### 2. Método de Décio Bazin (Preço Teto de Dividendos)

Criado pelo autor de *"Faça Fortuna com Ações"*, o método é a principal bússola para investidores focados em renda passiva no Brasil:

$$P_{\text{teto}} = \frac{\bar{D}}{0.06}$$

Onde:
- $\bar{D}$ = Média aritmética dos dividendos e JCP líquidos por ação pagos nos últimos 3 anos (ou 5 anos).
- $0.06$ = Dividend Yield mínimo esperado de $6.0\%$ ao ano.

#### Critérios Complementares de Bazin:
1. Rejeitar empresas com endividamento excessivo ($\text{Dívida Líquida} / \text{Patrimônio Líquido} > 0.50$ ou $\text{Dívida Líquida} / \text{Ebitda} > 2.0$).
2. Evitar empresas envolvidas em escândalos de corrupção ou que operem sem transparência com acionistas minoritários.

---

### 3. Modelo de Crescimento de Dividendos (Gordon Growth Model)

Adequado para empresas maduras com taxa previsível de crescimento de proventos (concessionárias de energia, saneamento, bancos):

$$P_0 = \frac{D_1}{r - g} = \frac{D_0 \times (1 + g)}{r - g}$$

Onde:
- $D_0$ = Último dividendo anual por ação pago.
- $D_1$ = Dividendo por ação projetado para o próximo ano.
- $r$ = Taxa de retorno exigida pelo acionista (*Cost of Equity* - $K_e$, tipicamente calculada via CAPM: $K_e = R_f + \beta \times (R_m - R_f)$).
- $g$ = Taxa de crescimento sustentável perpétuo dos dividendos ($g = \text{ROE} \times (1 - \text{Payout})$).
- *Condição de Validade Matemática:* $r > g$.

---

### 4. Fluxo de Caixa Descontado (Discounted Cash Flow - DCF)

O modelo analítico mais sofisticado de finanças corporativas, projetando a geração de caixa livre para a empresa (*Free Cash Flow to Firm* - FCFF):

$$\text{Enterprise Value (EV)} = \sum_{t=1}^{N} \frac{\text{FCFF}_t}{(1 + \text{WACC})^t} + \frac{\text{Valor Terminal}}{(1 + \text{WACC})^N}$$

#### A. Custo Médio Ponderado de Capital (WACC):
$$\text{WACC} = \left(\frac{E}{V} \times K_e\right) + \left(\frac{D}{V} \times K_d \times (1 - T)\right)$$
Onde:
- $E$ = Valor de mercado do patrimônio líquido (*Equity*).
- $D$ = Valor de mercado da dívida bruta (*Debt*).
- $V = E + D$ = Valor total da firma.
- $K_e$ = Custo de capital próprio.
- $K_d$ = Custo médio da dívida (taxa média de juros de empréstimos/debêntures).
- $T$ = Alíquota efetiva de imposto de renda da pessoa jurídica (IRPJ/CSLL, nominal de $34\%$).

#### B. Valor Terminal na Perpetuidade:
$$\text{Valor Terminal} = \frac{\text{FCFF}_N \times (1 + g_{\text{perp}})}{\text{WACC} - g_{\text{perp}}}$$

#### C. Valor Justo por Ação:
$$\text{Valor Intrínseco por Ação} = \frac{\text{EV} - \text{Dívida Líquida} - \text{Participação Minoritários}}{\text{Número Total de Ações}}$$

---

## 🗄️ Implementação no Banco de Dados (Migration 10)

Os cálculos são persistidos na tabela `analytics.valuation_ativo`:

```sql
CREATE TABLE IF NOT EXISTS analytics.valuation_ativo (
    valuation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instrument_id UUID NOT NULL REFERENCES canonical.instrument(instrument_id),
    data_calculo DATE NOT NULL DEFAULT CURRENT_DATE,
    preco_mercado NUMERIC(15, 4) NOT NULL,
    -- Modelo Graham
    preco_justo_graham NUMERIC(15, 4),
    margem_seguranca_graham_pct NUMERIC(8, 2),
    -- Modelo Bazin
    preco_teto_bazin NUMERIC(15, 4),
    margem_seguranca_bazin_pct NUMERIC(8, 2),
    -- Modelo Gordon
    preco_justo_gordon NUMERIC(15, 4),
    taxa_desconto_gordon_pct NUMERIC(6, 2),
    taxa_crescimento_gordon_pct NUMERIC(6, 2),
    -- Modelo DCF
    preco_justo_dcf NUMERIC(15, 4),
    wacc_utilizado_pct NUMERIC(6, 2),
    crescimento_perpetuo_pct NUMERIC(6, 2),
    criado_em TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp(),
    UNIQUE (instrument_id, data_calculo)
);
```

---

## 🔗 Referências Cruzadas
- [[Matriz_Unificada_12_Portais_e_Gaps|Matriz Unificada dos 12 Portais]]
- [[Conv_15_a_19_Engenharia_Reversa_Profunda_e_Gaps|Conversas 15 a 19: Implementação do Valuation]]
- [[Motores_Matematicos_e_Valuation_Quant|Motores Quantitativos do fin-api]]
