---
title: REA Fundamentus — Engenharia Reversa, Linhagem Upstream e Endpoints Ocultos
tags: [fin, rea, fundamentus, linhagem-upstream, scrapers, cvm, b3, pentest]
created: 2026-10-28
description: Descoberta de rotas legadas, endpoints ocultos em JSON, rastreabilidade regulatória e pentest do Fundamentus.
---

# 🕵️ REA Fundamentus: Engenharia Reversa, Linhagem Upstream e Endpoints Ocultos

O **Fundamentus** (`fundamentus.com.br`) é o mais tradicional portal de análise fundamentalista de ações e FIIs do Brasil. Esta documentação consolida o relatório da Engenharia Reversa de Aplicação (REA) conduzida na Conversa 15.

---

## 🗺️ Mapeamento Exaustivo das 13 Rotas do Fundamentus

| Rota / URL | Parâmetros Chave | Finalidade Original | Oportunidade de Extração para o `fin-ingestion` |
|---|---|---|---|
| `detalhes.php` | `?papel={ticker}` | Visão consolidada do ativo, cotação, múltiplos de valuation e dados da empresa. | Extração por parsing HTML via BeautifulSoup4 com seletores CSS de tabelas padronizadas. |
| `balancos.php` | `?papel={ticker}` | Balanço Patrimonial Ativo e Passivo dos últimos 10 a 20 trimestres. | Ingestão em lote de séries temporais de Ativo Total, Dívida Bruta e Patrimônio Líquido. |
| `demonstracoes.php` | `?papel={ticker}` | Demonstração do Resultado do Exercício (DRE) detalhada por trimestre. | Extração de Receita Líquida, Custos, Ebitda, Lucro Líquido e Margens. |
| `proventos.php` | `?papel={ticker}` | Histórico de Dividendos, JCP, Bonificações e Desdobramentos. | Carga de eventos corporativos com data com, data ex e valor por ação. |
| `cotacoes.php` | `?papel={ticker}` | Tabela HTML com cotações diárias de pregões passados. | Carga de histórico de fechamento, máxima, mínima e volume. |
| **`resultado.php`** | *(POST / GET com filtros)* | **Screener Geral de Ações** com todos os papéis e indicadores em uma única tabela HTML. | **Salto Quântico $O(1)$:** Uma única requisição HTTP captura mais de 450 empresas com P/L, P/VP, ROE e Margens simultaneamente. |
| **`fii_resultado.php`** | *(POST / GET com filtros)* | **Screener Geral de FIIs** listando todos os fundos com DY, P/VP e Vacância. | **Salto Quântico $O(1)$:** Extração massiva de todo o mercado de FIIs em segundos. |
| **`fii_imoveis.php`** | Sem parâmetros | Tabela com todos os imóveis físicos de todos os FIIs de tijolo do Brasil. | Carga direta de endereços, cidades, estados e área locável sem navegar fundo a fundo. |
| **`/amline/cot_hist.php`** | `?papel={ticker}` | **Endpoint Oculto Legado** que alimenta gráficos em Flash/JavaScript. | **Endpoint Oculto:** Retorna array de cotações em JSON puro, eliminando a necessidade de parsear tabelas HTML. |
| `fr.php` | Sem parâmetros | Feed cronológico de Comunicados ao Mercado e Fatos Relevantes. | Coleta contínua de notícias regulatórias entregues à CVM. |
| `ultimos-resultados.php`| Sem parâmetros | Lista das empresas que divulgaram balanços nos últimos dias. | Gatilho (*trigger*) para disparo imediato de pipelines de atualização contábil. |
| `graficos.php` | `?papel={ticker}` | Interface legada de gráficos de indicadores fundamentalistas. | Usado para validação cruzada de fórmulas matemáticas de múltiplos. |
| `contato.php` | N/A | Formulário institucional de contato. | Verificado durante o pentest. |

---

## 🏛️ Linhagem Upstream dos Dados: De Onde Realmente Vêm os Números?

O Fundamentus não cria dados financeiros do nada. Ele atua como um consolidador de fontes governamentais e regulatórias oficiais. O Fin substitui a dependência de scraping por conexões diretas às fontes originais:

```mermaid
flowchart LR
    subgraph Regulators["Fontes Primárias Oficiais (Upstream)"]
        B3["B3 S.A.<br/>Cotações, BVBG.028, Tickers, Negócios"]
        CVM["CVM Dados Abertos<br/>DFP Anual, ITR Trimestral, FRE, CVM 175"]
        BCB["Banco Central (SGS)<br/>Taxas de Juros, Inflação, Câmbio"]
        IBGE["IBGE<br/>Classificação Setorial CNAE, Censo"]
    end

    subgraph FundamentusSite["Portal Fundamentus"]
        FND["fundamentus.com.br<br/>Parseia CVM/B3 e gera tabelas HTML"]
    end

    subgraph FinEngine["Ecossistema Fin (fin-ingestion)"]
        DIR["Ingestão Direta das Fontes Primárias<br/>(Sem risco de bloqueio ou dependência)"]
        FALL["Fallback Inteligente<br/>(Endpoints Ocultos do Fundamentus)"]
    end

    B3 --> FND
    CVM --> FND
    BCB --> FND
    IBGE --> FND

    B3 ==> DIR
    CVM ==> DIR
    BCB ==> DIR
    IBGE ==> DIR
    FND -.->|Backup O(1)| FALL
```

---

## 🏦 Tratamento Contábil Especial: Instituições Financeiras vs Empresas Industriais

Durante a engenharia reversa do Fundamentus, evidenciou-se que o portal divide suas demonstrações financeiras em dois esquemas contábeis totalmente distintos:

### 1. Empresas Industriais e Comerciais (Plano Padrão)
- **Estrutura de Resultados:** Receita Líquida $\rightarrow$ Custos dos Produtos Vendidos (CPV) $\rightarrow$ Lucro Bruto $\rightarrow$ Ebitda $\rightarrow$ Ebit $\rightarrow$ Resultado Financeiro Líquido $\rightarrow$ Lucro Líquido.
- **Estrutura Patrimonial:** Ativo Circulante (Caixa, Clientes, Estoques) e Dívida Líquida (Dívida Bruta - Caixa).

### 2. Bancos e Instituições Financeiras (COSIF - Banco Central)
- Bancos **não possuem Ebitda, Estoques nem Dívida Líquida tradicional**, pois o dinheiro é a matéria-prima do negócio.
- **Estrutura de Resultados:**
  - *Receitas de Intermediação Financeira* (juros de empréstimos, financiamentos, títulos).
  - *Despesas de Intermediação Financeira* (captações em CDB, poupança, depósitos interfinanceiros).
  - *Resultado Bruto da Intermediação Financeira*.
  - *Provisão para Créditos de Liquidação Duvidosa (PDD)* — Indicador crítico de inadimplência.
  - *Receitas de Prestação de Serviços & Tarifas Bancárias*.
  - *Lucro Líquido*.

### Modelagem Adotada no `fin-db`
O schema `analytics.demonstracao_financeira` suporta ambas as estruturas através de uma coluna identificadora `tipo_plano_contabil VARCHAR(16) CHECK (tipo_plano_contabil IN ('PADRAO_INDUSTRIA', 'BANCO_COSIF', 'SEGURADORA_SUSEP'))`.

---

## 🛡️ Pentest e Auditoria de Segurança

Como parte da análise do Fundamentus e do ecossistema Fin, foram executados testes de segurança nas rotas expostas:

1. **SQL Injection:**
   - Teste de injeção nos parâmetros `?papel=' OR '1'='1` e `?papel=PETR4; DROP TABLE--`.
   - **No Fin:** Mitigação 100% garantida pelo uso de queries parametrizadas no SQLAlchemy assíncrono e validação rigorosa de regex com Pydantic (`ticker: str = Field(regex=r"^[A-Z0-9]{4,6}$")`).
2. **Cross-Site Scripting (XSS):**
   - Tentativa de injeção de payload `<script>alert(1)</script>` no campo de busca.
   - **No Fin:** O React 19 / Next.js 15 escapa automaticamente qualquer entidade HTML renderizada no DOM, prevenindo XSS refletido e persistido.
3. **Cabeçalhos de Segurança HTTP:**
   - Adicionados via middleware no FastAPI e no Gateway Traefik:
     - `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`
     - `X-Content-Type-Options: nosniff`
     - `X-Frame-Options: DENY`
     - `Content-Security-Policy: default-src 'self'`

---

## 🔗 Referências Cruzadas
- [[Matriz_Unificada_12_Portais_e_Gaps|Matriz Unificada dos 12 Portais]]
- [[Conv_15_a_19_Engenharia_Reversa_Profunda_e_Gaps|Conversas 15 a 19: Detalhes do REA Fundamentus]]
- [[Modelagem_Relacional_12_Schemas|Modelagem Relacional no fin-db]]
