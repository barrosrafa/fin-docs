---
title: Execução do SDD FIN — 29/10/2026
tags: [fin, sdd, qualidade, confiabilidade, seo, acessibilidade]
updated_at: 2026-10-29
status: implementado-parcialmente-validacao-local
---

# Execução do SDD FIN

Foram implementadas correções nos checkouts locais de `fin-web`, `fin-api`, `fin`, `fin-ingestion`, `fin-infra` e `fin-docs`, a partir do arquivo `sdd (1).md` fornecido. O documento foi tratado como especificação e evidência histórica; seus comandos propostos não foram confundidos com resultados de testes. Não houve publicação, push, alteração de conta externa ou execução da migração em banco.

A entrega cobre os defeitos prioritários da aplicação analítica, com fundamentos para cálculo decimal e cache público. **Não equivale à implementação integral das expansões de streaming e execução de ordens do SDD.** Também não homologa a base legada ou a aplicação pública atualmente publicada.

## Alterações por achado

| SDD | Implementação local | Limite/pendência |
|---|---|---|
| FIN-01 | Retirados benchmarks iniciais, fallbacks patrimoniais, recomendações técnicas, confiança/SWOT, checklist, receitas segmentadas e consenso sintéticos dos caminhos alterados. Demo identificada no título e nos indicadores. API de valuation incompleto responde 503. Simulador não fabrica calendário, CDI ou preço inicial quando faltam séries. | Outros módulos e dados previamente persistidos não receberam auditoria completa. As hipóteses remanescentes do simulador são retornadas e exibidas. |
| FIN-02 | Removidos `12ms`, mercado aberto e estados de serviços não medidos. Origem e data-base desconhecidas são declaradas; polling identificado como periódico. | Sem integração de calendário oficial/estados de sessão. Exibir desconhecido é deliberado. RTT não foi implementado. |
| FIN-03 | Worker e imports CSV passam a gravar observações em `cotacao_intraday`, sem chamar coleta de fechamento oficial. Histórico exclui fins de semana de instrumentos B3 selecionados e datas futuras. Retorno usa fechamento anterior elegível na mesma base nominal/ajustada; total considera toda a consulta. Migração SQL 16 preserva cópia dos registros suspeitos e os classifica como quarentena. | Migração preparada, não aplicada nem testada contra PostgreSQL real. Origem da linha específica de domingo e feriados oficiais ainda precisam de conferência. Não certifica ajustes, OHLC nem os demais leitores da base. |
| FIN-04/05 | Lock síncrono por proprietário, geração, cancelamento, deduplicação por instrumento/tipo, offset separado da lista deduplicada, erro visível, retry no mesmo offset e teardown. Cancelamento integrado aos fetchers de ativos/rankings. | Backend continua com offset, sem cursor e snapshot de catálogo estável. Mudanças do catálogo durante a navegação não têm garantia de snapshot consistente. |
| FIN-06 | Oito páginas por bloco, recarga anterior/próxima e virtualização medida em ações e screener. Foco retido e opção de renderizar todo o bloco para acesso por teclado. | As demais listas usam retenção por bloco, mas não todas receberam virtualização. Sem homologação de heap por horas. |
| FIN-07 | Cadastro/valuation/proventos separados da matriz por período, falhas independentes, abort e proteção contra conclusão obsoleta. Base pública recebida do servidor. Histórico também cancela leituras antigas. | Os testes de concorrência montam o hook real; não há teste integrado de todas as combinações de atraso de cada seção. |
| FIN-08 | Sem cotação válida, patrimônio e P&L atuais são nulos. Cobertura, valor coberto, origem, instante do fato e estado de preço são explícitos. Coleta recente não torna um fato antigo atual. | Política conservadora de cinco minutos desde `as_of`; fechamento EOD exige política de calendário própria. Fontes sem instante do fato não aprovam marcação atual. |
| FIN-09 | `Decimal`, validação de sinal/finitude/escala, taxas não negativas, preço de compra/venda positivo, custos incorporáveis explícitos e cálculo puro de compra/venda parcial, realizado e não realizado. Valores decimais serializados como strings. | Não há ledger transacional novo, persistência de operações, regras tributárias, margem, corporativos ou câmbio completos. O cálculo puro não é um sistema de execução. |
| FIN-10 | Documentação distingue o SSE de sincronização de um feed de cotações. | Gateway de ticks não criado. Não há fornecedor/contrato de feed integrado para homologação. |
| FIN-11 | Cadastro, histórico inicial e definições do glossário renderizados no servidor. Metadata por ticker, canonical, OG com imagem, JSON-LD sem cotação/oferta inventada e redirect HTTP 308 de caixa. | OG usa preço apenas quando existe data-base. JSON-LD foi verificado como JSON e coerência básica, sem certificação externa de rich results. Indexação/CrUX não aferidos. |
| FIN-12 | Envelope validado por versão/metadados, preservando fonte, hash, geração e classificação de idade do arquivo. Ticker mostra procedência disponível. | Hash preservado, não autenticado/verificado criptograficamente. Data de geração do arquivo não é a data do fato. Sem cadeia de fonte por todos os registros. |
| FIN-13 | Contraste do token cinza corrigido, foco visível, skip link, densidade e azul/laranja persistidos, offsets a partir da altura do header e movimento reduzido. | Sem auditoria WCAG integral em todos os componentes, viewports ou dispositivos. |
| FIN-14 | Formatadores pt-BR, ausência como travessão, remoção de selos de auditoria sem prova, rótulo de yield da fonte e DPA implícito. DRE não inventa custos/impostos a partir da receita. | Divergências de séries/períodos legadas ainda exigem reconciliação documental. |
| FIN-15 | Documento de deploy atualizado com Next 16.4.3/React 19.3.0 e cache efetivo; retiradas garantias indevidas de latência/24x7. Runbook de capacidade adicionado. | Cotas atuais dos provedores não verificadas; não há capacidade/SLA homologados. |

## Cache e contratos

Histórico público em IndexedDB com chave por instrumento, período, BRL, base nominal/ajustada, faixa e versão de schema. Orçamento de payload de 25 MiB, LRU e expiração de sete dias. Falha do armazenamento não impede leitura de rede. Offline é identificado como desatualizado. Não armazena tokens ou carteiras.

Como o upstream não oferece ETag/versão de eventos corporativos confiável, cada leitura tenta a rede antes de recorrer ao cache. Portanto **não foi aprovado o critério de evitar download integral ao revisitar**. Workers de indicadores foram adiados: não houve medição que justificasse transferir os cálculos atuais pequenos a uma thread.

Mudanças contratuais relevantes: números decimais privados passam a strings; campos de preço/P&L/modelos ausentes podem ser `null`; `fetchCdnSnapshot` retorna envelope, não só `data`; alguns adapters passam a propagar erro para a seção em vez de convertê-lo em lista vazia. Os consumidores afetados no checkout foram ajustados. Consumidores externos precisam avaliar a compatibilidade antes de publicar a API.

O header `X-Total-Count` passou a ser exposto por CORS. Sem isso, navegadores em outro domínio tratavam a primeira página como o catálogo inteiro.

## Verificação executada

Resultados finais e hashes dos arquivos estão em `evidencias-sdd-2026-10-29/manifest.json`. Os logs correspondentes ficam na mesma pasta.

| Gate | Resultado local |
|---|---|
| `fin-web: pnpm test` | 11 testes passaram: concorrência/geração/retry/retenção, metadados de snapshot, formatação de ausentes e cache com expiração/LRU. Inclui o contrato preexistente do componente de concentração. |
| `fin-web: pnpm lint` | Sem erros; 23 avisos remanescentes de imports, dependências e componentes existentes. |
| `fin-web: pnpm build` | Build de produção aprovado, com TypeScript. |
| `fin-web: pnpm exec playwright test --config playwright.sdd.config.ts` | 6 testes passaram: falhas HTTP, retry, catálogo de 10.000 instrumentos, blocos/DOM limitado, metadata/HTML sem JS, canonical/308, preferências e sticky. Fonte sintética exclusivamente em fixture local. |
| `fin-api: pytest` nos módulos listados abaixo | 49 testes passaram. Exemplo SDD de compra com R$5 de custo, venda parcial com R$2, realizado R$76 e não realizado R$57 aprovado; entradas inválidas, cotação ausente/stale e dados reais preservados cobertos. |
| `fin-ingestion: pytest tests/test_status_invest_etl.py tests/test_snapshot_exporter.py` | 12 testes passaram, incluindo execução do ETL com conexão controlada e comprovação de que snapshots não são gravados como fechamento diário. |
| `git diff --check` | Checkouts alterados sem erros de whitespace. |

API: `tests/test_portfolio.py`, `tests/test_pnl_decimal.py`, `tests/test_data_integrity.py`, `tests/test_worker.py`, `tests/test_market.py`, `tests/test_collectors.py`, `tests/test_cors.py`, `tests/test_review_fixes.py`.

O backend local não estava disponível na geração do build; os caminhos sem fonte foram renderizados como indisponíveis. O build aprovado não comprova dados reais carregados. O ensaio de HTML usa fonte controlada local. Falhas do glossário têm cache curto para permitir recuperação.

A fixture de navegador mantém dados fictícios fora dos endpoints públicos reais. Testes do ETL e serviços usam dependências controladas: não comprovam concorrência, DDL ou semântica SQL em PostgreSQL real. A suíte integral de integração com dados externos não foi aprovada por inferência desses testes.

## Continuação necessária do SDD

1. Revisar/ensaiar SQL 16 em banco isolado; auditar PETR4/25-10 e produtores legados; integrar calendário oficial versionado e ajustes corporativos. Não executar backfill baseado em preço presumido.
2. Completar procedência por registro e reconciliar períodos/definições contábeis. Definir política de fechamento EOD para carteira e atualizar consumidores do contrato decimal.
3. Migrar offset para cursor/snapshot de catálogo; medir retenção, foco e âncoras de densidade sob mudanças reais de dados.
4. Definir fonte autorizada, delay, ordenação, replay e capacidade antes do lote de stream. Implementar protocolo/backpressure/reconexão e só então executar stress de ticks/fanout.
5. Para paper orders: definir conta/autenticação, moeda/lote/estados, ledger, reserva, outbox e reconciliação. Idempotência exige ensaio com concorrência de banco; botão bloqueado não é prova.
6. Executar soak 4h/8h com artefatos de heap, microquedas do futuro stream, CWV/RUM e acessibilidade ampliada. Nenhum desses resultados foi declarado aprovado nesta entrega.

As alterações preexistentes em `fin-ingestion/cli.py`, `dbt/models/staging/schema.yml` e arquivos locais dbt foram preservadas. Nenhuma automação recorrente foi criada.

Relacionado: [[Deployment_Cloud_Gratuito_Zero_Cost]]. Runbook: `fin-infra/docs/market-data-capacity.md`. Migração preparada: `fin/sql/16_market_data_quality_v4.4/01_quality_and_quarantine.sql`.
