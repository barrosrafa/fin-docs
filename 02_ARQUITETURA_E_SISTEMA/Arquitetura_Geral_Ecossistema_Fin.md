---
title: Arquitetura Geral do Ecossistema Fin
tags: [fin, arquitetura, ecossistema, backend, frontend, banco-de-dados, ingestao, infraestrutura]
created: 2026-10-28
description: Visão macroarquitetural do ecossistema Fin, seus 5 módulos, fluxo de dados e limites de fronteira.
---

# 🏛️ Arquitetura Geral do Ecossistema Fin

A plataforma **Fin** foi projetada como um ecossistema desacoplado de alta performance, unindo a velocidade de um frontend moderno em Next.js 15, a robustez de um backend assíncrono em FastAPI, a precisão matemática de um banco relacional com séries temporais (PostgreSQL 18 + TimescaleDB) e uma malha de coleta assíncrona com evasão anti-bloqueio.

---

## 🧭 Diagrama Arquitetural de Alto Nível

```mermaid
flowchart TD
    subgraph ClientLayer["🖥️ Camada de Interface & Usuário"]
        WEB["fin-web<br/>Next.js 15 • React 19 • Tailwind CSS • shadcn/ui<br/>Porta 3000 / Vercel Edge"]
    end

    subgraph IngressGateway["🛡️ Camada de Gateway & Borda"]
        GW["fin-infra (Gateway)<br/>Traefik / Caddy • Cloudflare Tunnel • SSL Automático<br/>Rate Limiting: 60 rpm anônimo / 300 rpm autenticado"]
    end

    subgraph ServiceLayer["⚙️ Camada de Aplicação & APIs"]
        API["fin-api<br/>FastAPI • Python 3.12+ • SQLAlchemy 2.0 Async • Uvicorn<br/>Porta 8000 / Koyeb Serverless"]
        REDIS[("Redis Cache & Filas<br/>Redis 7.2 / Upstash<br/>Cache-Aside • SWR • Distributed Lock")]
        WORKER["Worker em Background<br/>ARQ Worker • Python Assíncrono"]
    end

    subgraph IngestionLayer["🔄 Camada de Ingestão & ETL"]
        ING["fin-ingestion<br/>Coletores Assíncronos • Scrapers HTTPX/Playwright • dbt<br/>GitHub Actions / Cron agendado"]
        SOURCES["Fontes Oficiais & Portais de Mercado<br/>B3 • CVM • BCB • ANBIMA • 12 Portais Financeiros"]
        SQUID["Proxy Egress & Evasão<br/>Squid • Proxies Residenciais Rotativos"]
    end

    subgraph DataLayer["📦 Camada de Persistência (fin-db)"]
        NEON[("PostgreSQL 18 (Neon Serverless)<br/>12 Schemas • 303 Tabelas • 36 Hypertables TimescaleDB<br/>pgvector • pg_trgm • pgcrypto")]
    end

    %% Fluxos de Usuário
    WEB -->|HTTPS / REST / SSE| GW
    GW -->|Reverse Proxy /api/v1| API
    API <-->|Cache L1 / SWR| REDIS
    API -->|Enfileirar tarefas pesadas| WORKER
    API <-->|Consultas ACID & Timescale| NEON

    %% Fluxos de Ingestão
    ING -->|Requisições com rotação de IP| SQUID
    SQUID --> SOURCES
    SOURCES --> SQUID
    SQUID --> ING
    ING -->|Pouso bruto em raw.payload| NEON
    ING -->|Transformações dbt| NEON
```

---

## 📦 Matriz de Responsabilidades dos 5 Módulos

| Módulo | Repositório GitHub | Stack Tecnológica | Papel no Ecossistema |
|---|---|---|---|
| **`fin-db`** | [`barrosrafa/fin-db`](https://github.com/barrosrafa/fin-db) | PostgreSQL 18, TimescaleDB, pgvector, pg_trgm | Repositório central de verdade. Modela os 12 schemas relacionais, 303 tabelas, 36 hypertables, regras de integridade referencial, constraints fiscais e partidas dobradas. |
| **`fin-api`** | [`barrosrafa/fin-api`](https://github.com/barrosrafa/fin-api) | FastAPI, Python 3.12+, SQLAlchemy 2.0, Redis | Backend REST assíncrono. Serve dados de alta performance para a interface, gerencia autenticação, calcula métricas de valuation quantitativo e consolidação de carteira. |
| **`fin-infra`** | [`barrosrafa/fin-infra`](https://github.com/barrosrafa/fin-infra) | Docker Compose, Traefik, Caddy, Cloudflare, Prometheus | Gateway de borda, malha de proxies residenciais para raspagem, observabilidade (Prometheus/Grafana/Loki) e túneis seguros de exposição. |
| **`fin-ingestion`**| [`barrosrafa/fin-ingestion`](https://github.com/barrosrafa/fin-ingestion) | Python, httpx, Playwright, dbt, BeautifulSoup4 | Motor assíncrono de coleta e extração. Povoa a camada `raw.payload` e orquestra a limpeza e estruturação dos dados de 12 sites e 4 órgãos reguladores. |
| **`fin-web`** | [`barrosrafa/fin-web`](https://github.com/barrosrafa/fin-web) | Next.js 15, React 19, TypeScript, Tailwind CSS | Frontend responsivo de alta densidade visual (estilo Bloomberg). Totalmente conectado à API viva, sem nenhum dado mockado. |

---

## 🛡️ Invariantes Arquiteturais e Regras de Isolamento

Para garantir escalabilidade, testabilidade e manutenibilidade sem degradação operacional, o ecossistema obedece a **5 invariantes invioláveis**:

1. **A API Nunca Faz Scraping em Requisição de Usuário:**
   Nenhum endpoint de `fin-api` pode acionar scrapers externos de forma síncrona. Se um usuário solicitar um ativo cujos dados contábeis não estão no banco, a API responde imediatamente com o que existe e registra uma solicitação assíncrona na fila de ingestão.
2. **Padrão de Pouso em Camada Bronze (`raw.payload`):**
   Todo dado capturado da internet pelo `fin-ingestion` é obrigatoriamente persistido primeiro na tabela `raw.payload` em formato JSONB imutável com hash SHA-256. Isso permite reprocessar qualquer transformação analítica do passado sem refazer scraping na fonte original.
3. **Consistência Contábil de Partidas Dobradas (Soma Zero):**
   Toda movimentação patrimonial, aporte, retirada ou cobrança de taxa na carteira obedece à equação contábil $\sum \text{Débitos} - \sum \text{Créditos} = 0$, validada por trigger em banco de dados.
4. **Isolamento de Acesso a Dados:**
   O frontend `fin-web` jamais conecta diretamente ao banco de dados PostgreSQL. Toda comunicação ocorre estritamente via HTTPS/REST através de `fin-api`.
5. **Idempotência Estrita em Toda a Cadeia:**
   Tanto a ingestão quanto as APIs de transação operam sob chaves naturais de conflito (`ON CONFLICT DO NOTHING / UPDATE`) e tokens de idempotência SHA-256, garantindo que retentativas de rede não gerem registros duplicados.

---

## 🔗 Referências Cruzadas
- [[00_Linha_do_Tempo_e_Matriz_23_Conversas|Histórico Completo das 23 Conversas]]
- [[Arquitetura_Financeira_Missao_Critica|Especificação de Missão Crítica & Ledger de Partidas Dobradas]]
- [[Tese_Carteira_Invisivel_Dual_Engine|A Tese da Carteira Invisível & Dual Engine]]
- [[Deployment_Cloud_Gratuito_Zero_Cost|Guia de Deploy Cloud 100% Gratuito]]
