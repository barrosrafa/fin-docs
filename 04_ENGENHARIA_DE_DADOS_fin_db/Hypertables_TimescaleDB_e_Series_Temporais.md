---
title: Hypertables TimescaleDB & Séries Temporais no fin-db
tags: [fin, banco-de-dados, timescaledb, hypertables, series-temporais, postgresql, performance]
created: 2026-10-28
description: As 36 hypertables provisionadas para dados de alta frequência, cotações, livros e métricas financeiras.
---

# ⚡ Hypertables TimescaleDB & Séries Temporais no fin-db

Para lidar com dezenas de milhões de registros de cotações, livros de ofertas e eventos de mercado sem degradação de performance, o **`fin-db`** utiliza a extensão **TimescaleDB** sobre o PostgreSQL 18.

---

## 🏛️ O Que é uma Hypertable?

Uma **Hypertable** é uma abstração do TimescaleDB que se apresenta para a aplicação como uma única tabela comum do PostgreSQL, mas nos bastidores particiona automaticamente os dados em **pedaços físicos (*chunks*)** com base em um eixo temporal (coluna `tempo TIMESTAMPTZ`).

### Vantagens Práticas:
1. **Velocidade de Escrita Constante:** As inserções caem apenas no *chunk* correspondente à data atual, mantendo os índices pequenos o suficiente para caber na memória RAM.
2. **Exclusão em Lote Sem Overhead:** A remoção de dados antigos é feita descartando chunks inteiros em milissegundos (`drop_chunks`), sem gerar *bloat* de vácuo no banco.
3. **Funções Analíticas Especializadas:** Suporte nativo a `time_bucket()`, `first()`, `last()` e interpolação linear (`locf()`).

---

## 📊 Inventário das 36 Hypertables por Domínio

| Domínio / Schema | Nome da Hypertable | Coluna de Partição | Intervalo de Chunk | Finalidade dos Dados |
|---|---|---|:---:|---|
| **Mercado** | `market.cotacao_diaria` | `tempo` | 1 mês | Cotações diárias OHLCV de ações, FIIs, BDRs e ETFs. |
| | `market.cotacao_intraday` | `tempo` | 1 dia | Cotações minuto a minuto durante o pregão da B3. |
| | `market.book_ordens` | `tempo` | 1 dia | Profundidade do livro de ofertas (Top 10 níveis de compra e venda). |
| | `market.negocio_intraday` | `tempo` | 1 dia | Negócios executados tick a tick (Tape Reading). |
| | `market.after_market` | `tempo` | 7 dias | Negociações realizadas na sessão após o pregão regular. |
| | `market.ajuste_intraday` | `tempo` | 1 mês | Ajustes de fechamento do mercado futuro e derivativos. |
| | `market.serie_macroeconomica`| `tempo` | 1 ano | Séries temporais de Selic, CDI, IPCA, IGP-M e Câmbio do BCB. |
| | `market.aluguel_acoes` | `tempo` | 1 mês | Histórico de taxa média e volume de aluguel de ações (BTC B3). |
| | `market.benchmark_serie` | `tempo` | 1 ano | Retorno acumulado diário do Ibovespa, IFIX, CDI e S&P 500. |
| **FIIs & Fundos** | `asset_details.historico_vacancia_fii` | `tempo` | 1 ano | Evolução da vacância física e financeira por fundo. |
| | `asset_details.historico_patrimonio_fii` | `tempo` | 1 ano | Variação mensal do patrimônio líquido contábil de FIIs. |
| | `asset_details.etf_nav` | `tempo` | 1 mês | Valor patrimonial da cota (NAV) de ETFs. |
| | `asset_details.etf_fluxo_cotas`| `tempo` | 1 mês | Criação e resgate de cotas de ETFs (fluxo institucional). |
| **Analytics** | `analytics.historico_indicador_fundamentalista` | `tempo` | 1 mês | Séries históricas de múltiplos (P/L, P/VP, ROE, Margem Líquida). |
| | `analytics.historico_indicador_tecnico` | `tempo` | 1 mês | Médias móveis (SMA 20, 50, 200), IFR (RSI), MACD e Bandas de Bollinger. |
| | `analytics.historico_risco` | `tempo` | 1 ano | Volatilidade anualizada de 21, 63 e 252 dias e Beta do ativo. |
| | `analytics.evento_usuario` | `tempo` | 1 mês | Cliques e interações analíticas na plataforma. |
| | `analytics.recomendacao` | `tempo` | 1 ano | Histórico de recomendações emitidas pela plataforma. |
| **Wealth** | `wealth.transacao_bancaria` | `tempo` | 1 ano | Extrato de transações bancárias do Open Finance. |
| **Data Warehouse**| `dw.historico_posicao` | `tempo` | 1 mês | Snapshots diários da posição de cada ativo por carteira. |
| | `dw.historico_carteira` | `tempo` | 1 mês | Snapshots consolidados de saldo e patrimônio por carteira. |
| | `dw.historico_patrimonio` | `tempo` | 1 ano | Patrimônio total consolidado do investidor. |
| | `dw.historico_rentabilidade` | `tempo` | 1 mês | Rentabilidade diária calculada por TWR e MWR. |
| | `dw.ranking` | `tempo` | 1 mês | Rankings diários de maiores altas, maiores dividendos e menor P/L. |
| | `dw.historico_indicador` | `tempo` | 1 mês | Agregações macro para backtesting quantitativo. |
| **Segurança & Log**| `system.uso_api` | `tempo` | 7 dias | Rastreio de tempo de resposta, endpoint e payload de chamadas REST. |
| | `system.auditoria_log` | `tempo` | 1 mês | Trilha de auditoria imutável de eventos do sistema. |
| | `system.log_acesso` | `tempo` | 7 dias | Registro de conexões e endereços IP. |
| | `system.tentativa_login` | `tempo` | 7 dias | Tentativas de autenticação (proteção contra força bruta). |
| | `system.api_key_uso` | `tempo` | 7 dias | Consumo de requisições por chave de API. |
| | `system.alerta_disparo` | `tempo` | 1 mês | Alertas de preço e eventos corporativos disparados para usuários. |
| | `system.evento_seguranca` | `tempo` | 1 mês | Alertas de fraude, token expirado e tentativas de injeção. |
| | `system.rate_limit` | `tempo` | 1 dia | Janelas deslizantes de rate limiting por IP e usuário. |
| | `system.inbox` | `tempo` | 1 mês | Notificações entregues no painel do usuário. |
| | `system.envio_campanha` | `tempo` | 1 mês | Relatórios e e-mails disparados aos usuários. |
| **Social** | `social.mensagem_direta` | `tempo` | 1 mês | Mensagens e teses compartilhadas entre investidores. |

---

## 🛠️ Exemplo DDL de Criação de Hypertable

```sql
-- 1. Criação da tabela padrão com coluna de tempo
CREATE TABLE IF NOT EXISTS market.cotacao_diaria (
    tempo TIMESTAMPTZ NOT NULL,
    instrument_id UUID NOT NULL REFERENCES canonical.instrument(instrument_id),
    preco_abertura NUMERIC(15, 4) NOT NULL,
    preco_maximo NUMERIC(15, 4) NOT NULL,
    preco_minimo NUMERIC(15, 4) NOT NULL,
    preco_fechamento NUMERIC(15, 4) NOT NULL,
    preco_medio NUMERIC(15, 4),
    volume_financeiro NUMERIC(18, 2) NOT NULL,
    quantidade_negocios INT NOT NULL,
    quantidade_titulos BIGINT NOT NULL,
    PRIMARY KEY (instrument_id, tempo)
);

-- 2. Transformação em Hypertable particionada por mês (intervalo de 30 dias)
SELECT create_hypertable(
    'market.cotacao_diaria',
    'tempo',
    chunk_time_interval => INTERVAL '30 days',
    if_not_exists => TRUE
);

-- 3. Criação de índice composto para busca de séries temporais
CREATE INDEX IF NOT EXISTS idx_cotacao_diaria_inst_tempo 
ON market.cotacao_diaria (instrument_id, tempo DESC);
```

---

## 📈 Consultas Temporais Avançadas no Fin

Graças ao TimescaleDB, a API consegue agrupar cotações diárias em barras semanais ou mensais (*candlesticks*) em pouquíssimos milissegundos:

```sql
-- Agrupamento de cotações em velas semanais (Candlestick Semanal)
SELECT 
    time_bucket('1 week', tempo) AS semana,
    first(preco_abertura, tempo) AS abertura,
    max(preco_maximo) AS maxima,
    min(preco_minimo) AS minima,
    last(preco_fechamento, tempo) AS fechamento,
    sum(volume_financeiro) AS volume_total
FROM market.cotacao_diaria
WHERE instrument_id = :instrument_id
  AND tempo >= NOW() - INTERVAL '1 year'
GROUP BY semana
ORDER BY semana ASC;
```

---

## 🔗 Referências Cruzadas
- [[Modelagem_Relacional_12_Schemas|Dicionário dos 12 Schemas]]
- [[Catalogo_1242_Ativos_e_Chaves_Naturais|Catálogo de Ativos B3]]
- [[Especificacao_APIs_FastAPI_e_Routers|Endpoints de Séries Temporais na API]]
