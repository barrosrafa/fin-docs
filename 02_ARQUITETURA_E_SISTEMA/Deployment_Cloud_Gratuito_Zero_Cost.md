---
title: Deploy Cloud 100% Gratuito (Zero-Cost Production Architecture)
tags: [fin, arquitetura, cloud, deploy, vercel, koyeb, render, neon, upstash, github-actions, free-tier]
created: 2026-10-28
description: Arquitetura distribuída de alta disponibilidade operando 24/7 com custo zero de infraestrutura.
---

# ☁️ Deploy Cloud 100% Gratuito (Zero-Cost Production Architecture)

Este documento documenta o projeto técnico de implantação e operação contínua do ecossistema **Fin** em nuvem, garantindo disponibilidade 24/7 sem incorrer em custos mensais de servidores.

---

## 🏛️ Topologia da Arquitetura Distribuída Gratuita

```mermaid
flowchart TD
    subgraph Edge["🌍 Borda & CDN Global (Cloudflare & Vercel Edge)"]
        CF["🛡️ Cloudflare DNS, WAF & SSL Gratuito"]
        VERCEL["🖥️ Vercel Hobby (fin-web)<br/>Next.js 15 • Server-Side Rendering (SSR)<br/>100 GB Largura de Banda/mês • SSL Automático"]
    end

    subgraph Compute["⚙️ Backend & Processamento Assíncrono"]
        KOYEB["🚀 Koyeb Nano / Render Free (fin-api)<br/>FastAPI • Python 3.12+ • 512 MB RAM • 0.1 vCPU<br/>Keep-Alive via /health em memória RAM"]
        GHA["🔄 GitHub Actions Cron (fin-ingestion)<br/>2.000 minutos de execução Linux/mês gratuitos<br/>Disparos agendados: 08:00, 12:00, 18:00 e 22:00 BRT"]
    end

    subgraph Data["📦 Armazenamento em Nuvem & Cache"]
        NEON["🐘 Neon Serverless PostgreSQL 18 (fin-db)<br/>TimescaleDB • pgvector • 0.5 GB Storage<br/>100 Compute Unit-horas/mês grátis"]
        UPSTASH[("⚡ Upstash Redis Serverless<br/>10.000 comandos/dia gratuitos<br/>Cache L1 / Cache-Aside com SWR")]
    end

    %% Relações
    USER["Investidor / Usuário"] -->|HTTPS / Domínio próprio| CF
    CF --> VERCEL
    VERCEL -->|Chamadas de API /api/v1| KOYEB
    KOYEB <-->|Cache de Alta Velocidade| UPSTASH
    KOYEB <-->|Conexão Pooling SSL| NEON
    GHA -->|Carga de Dados Idempotente| NEON
    GHA -.->|Invalidação de Cache por Tag| UPSTASH
```

---

## 🛠️ Detalhamento dos Componentes e Provedores Gratuitos

| Componente | Provedor Gratuito | Especificações do Plano | Função no Ecossistema |
|---|---|---|---|
| **Frontend Web** | **Vercel** (Hobby) | 100 GB largura de banda, builds automáticos via Git, Edge Network global | Hospeda o `fin-web` (Next.js 15 App Router). Renderiza componentes e serve a aplicação com latência < 30ms no Brasil. |
| **Backend API** | **Koyeb** (Nano) / **Render** | 512 MB RAM, 0.1 vCPU, HTTPS nativo, deploy contínuo via Docker/Git | Executa o `fin-api` (FastAPI). Expõe a API REST protegida e orquestra regras de negócio. |
| **Banco de Dados** | **Neon** (Serverless) | PostgreSQL 18, 0.5 GB armazenamento SSD, 100 Compute Unit-horas/mês | Hospeda o `fin-db` com seus 12 schemas, 303 tabelas, hypertables TimescaleDB e pgvector. |
| **Cache & Filas** | **Upstash** (Serverless) | 10.000 comandos/dia gratuitos, armazenamento de 256 MB, TLS nativo | Cache L1 de cotações, books, valuation e controle distribuído de tokens de idempotência. |
| **Ingestão & ETL** | **GitHub Actions** | 2.000 minutos/mês de runners Ubuntu Linux gratuitos | Executa os coletores e scrapers do `fin-ingestion` 4 vezes ao dia nos horários de fechamento. |
| **Borda & Segurança**| **Cloudflare** (Free) | DNS Anycast ilimitado, mitigação DDoS L3/L4/L7, certificados SSL/TLS | Protege a entrada do sistema, mascara IPs de origem e gerencia subdomínios. |

---

## 💡 As Quatro Engenhosidades Técnicas Anti-Custo

A sustentabilidade de uma arquitetura 100% gratuita depende de impedir que serviços hibernem ou estoure as cotas gratuitas dos provedores:

### 1. Prevenção de Hibernação do Koyeb sem Gastar a Cota do Neon
- **O Problema:** O Koyeb desliga contêineres inativos após 1 hora sem requisições. O Neon cobra minutos de CPU enquanto há conexões abertas (limite de 100 CU-horas/mês). Se um script de ping convencional testar o banco a cada 10 minutos, o Neon consome 100% da cota gratuita em menos de 15 dias.
- **A Solução:** Implementado o endpoint `/health` que responde **exclusivamente a partir da memória RAM da aplicação**, com status `HTTP 200 OK {"status": "alive", "uptime": 12345}` sem abrir pool de conexões com o PostgreSQL. O Koyeb permanece ligado 24/7 sem consumir 1 segundo sequer da cota do banco de dados Neon.

### 2. Defesa da Quota do Upstash Redis (< 10.000 comandos/dia)
- **O Problema:** Se cada clique em uma ação disparar 10 comandos Redis (`GET quote`, `GET multiples`, `GET history`), 1.000 visualizações de página esgotam o limite diário gratuito.
- **A Solução:** 
  1. Uso de comandos em lote (`MGET` / `MSET`), onde uma única chamada recupera ou salva múltiplos indicadores com custo de apenas 1 comando no Upstash.
  2. Implementação de cache de memória local L1 na API (via `cachetools` com TTL de 5 segundos) para absorver requisições repetidas em picos de tráfego antes de consultar o Redis.

### 3. Orquestração de Ingestão via GitHub Actions Crons
- **O Problema:** Manter um worker de scraping rodando continuamente exige servidores caros de longa duração.
- **A Solução:** O `fin-ingestion` é empacotado como script CLI (`python -m fin_ingestion run-daily`) executado por workflows do GitHub Actions agendados:
  - `08:00 BRT` — Abertura de mercado e cotações de moedas/macroeconomia.
  - `12:00 BRT` — Boletim intermediário de pregão.
  - `18:30 BRT` — Fechamento B3 e download do arquivo diário de cotações (COTAHIST).
  - `22:00 BRT` — Balanços CVM (DFP/ITR) e informes diários de Fundos e FIIs.
  - *Consumo total estimado:* ~250 minutos/mês dos 2.000 minutos gratuitos oferecidos pelo GitHub.

### 4. Vercel Serverless Edge & Prefetch
- O `fin-web` utiliza Incremental Static Regeneration (ISR) com `revalidate = 60` nas páginas de ações. Isso faz com que a Vercel gere páginas estáticas no Edge e atenda milhares de visitantes diretamente da CDN sem fazer requisições à API.

---

## 🔗 Referências Cruzadas
- [[Conv_20_a_23_Abastecimento_Auditoria_Producao_e_Cloud|Conversas 20 a 23: Detalhes do Deploy Cloud]]
- [[Topologia_Producao_e_Observabilidade|Topologia de Produção e Observabilidade]]
- [[Arquitetura_Geral_Ecossistema_Fin|Arquitetura Geral do Ecossistema]]
