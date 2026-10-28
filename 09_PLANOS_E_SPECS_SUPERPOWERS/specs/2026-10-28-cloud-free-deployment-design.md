# Arquitetura e Deploy Distribuído Gratuito: fin-web, fin-api, fin-ingestion e fin-infra

## 1. Contexto & Intenção (The Why)
O ecossistema **fin** é uma plataforma analítica e transacional de investimentos brasileira e internacional composta por frontend Next.js (`fin-web`), API de alta performance FastAPI/Python (`fin-api`), pipeline de ingestão e analytics com dbt (`fin-ingestion`), banco de dados relacional e analítico (`fin-db`) e configurações de infraestrutura (`fin-infra`).

O objetivo deste projeto é disponibilizar o ecossistema completo para acesso público e irrestrito (desktop, mobile e qualquer dispositivo), operando 24 horas por dia, 7 dias por semana, com **custo zero permanente de infraestrutura** utilizando os melhores serviços gerenciados de camada gratuita (Free Tier).

---

## 2. Metas, Não-Metas e Anti-Metas

### Metas (Goals)
- **Custo Zero ($0/mês)**: Operar 100% dentro dos limites gratuitos definitivos dos provedores.
- **Disponibilidade 24/7**: Interface web e API respondendo sem cold starts ou hibernação por inatividade.
- **Ingestão Incremental & Idempotente**: Rotinas de coleta programadas no GitHub Actions para atualizar cotações, proventos e novos tickers sem duplicidade de registros (`ON CONFLICT DO UPDATE` / deduplicação).
- **Deploy Contínuo (CI/CD)**: Cada push nas branches `main` dos repositórios GitHub atualiza automaticamente os serviços em produção.
- **Segurança & SSL/HTTPS**: Certificados TLS gerenciados automaticamente de ponta a ponta.

### Não-Metas (Non-Goals)
- Provisionamento de infraestrutura paga em AWS (RDS Aurora, NAT Gateways, ALB, ECS Fargate), presente nos templates do `fin-infra/terraform`.
- Ingestão contínua em streaming tick-by-tick segundo a segundo (inviável em free tiers de banco de dados).

### Anti-Metas (Anti-Goals)
- Depender de máquinas locais ligadas 24h ou de túneis efêmeros (como URLs temporárias do `trycloudflare.com`).
- Aceitar plataformas que hibernem a API após 15 minutos sem visitas (como o plano grátis do Render sem workarounds).
- Ingestão descontrolada que gere registros duplicados no banco de dados Neon ou esgote o limite de armazenamento.

---

## 3. Matriz de Arquitetura e Componentes

| Componente | Repositório | Plataforma de Nuvem | Plano / Custo | Função Principal |
|---|---|---|---|---|
| **fin-web** | `barrosrafa/fin-web` | **Vercel** | Hobby ($0) | Frontend Next.js 16 / React 19, CDN Edge global, HTTPS nativo |
| **fin-api** | `barrosrafa/fin-api` | **Koyeb** | Free Nano ($0) | API FastAPI (Python 3.12), sem hibernação, endpoints públicos REST |
| **fin-db** | `barrosrafa/fin-db` | **Neon Serverless** | Free Tier ($0) | PostgreSQL 18 com TimescaleDB, pgvector, conexão via pooler seguro |
| **Redis** | Integrado | **Upstash Redis** | Free Tier ($0) | 10.000 comandos/dia, 256MB, SSL nativo (`rediss://`), sem expiração |
| **fin-ingestion** | `barrosrafa/fin-ingestion` | **GitHub Actions** | 2.000 min/mês ($0) | Cron workflows agendados para coleta de B3/CVM/StatusInvest e enriquecimento |
| **fin-infra** | `barrosrafa/fin-infra` | **GitHub & Koyeb CLI** | Repositório ($0) | Configurações de deploy (`koyeb.yaml`), variáveis de ambiente e documentação |

---

## 4. Detalhamento Técnico das Decisões

### 4.1. Frontend (`fin-web`) na Vercel
- **Build & Runtime**: `pnpm run build` (Next.js 16.4.3 Turbopack).
- **Variáveis de Ambiente**:
  - `NEXT_PUBLIC_API_URL`: URL HTTPS pública fornecida pelo Koyeb (ex: `https://fin-api-<username>.koyeb.app/api/v1`).
- **Deploy**: Conexão nativa do GitHub com o Vercel. Cada push dispara preview e production deploys.

### 4.2. Backend (`fin-api`) no Koyeb
- **Build**: Imagem Docker gerada a partir do `Dockerfile` oficial do repositório `fin-api`.
- **Configuração de Execução**:
  - Porta de escuta vinculada à variável `PORT` (porta 8000).
  - Health check HTTP configurado no endpoint `/health` ou `/docs`.
- **CORS**:
  - Adição do domínio Vercel (`*.vercel.app` e domínio de produção) nas origens permitidas do middleware FastAPI.
- **Variáveis de Ambiente**:
  - `ENVIRONMENT=production`
  - `DEBUG=false`
  - `PORT=8000`
  - `HOST=0.0.0.0`
  - `DATABASE_URL`: String pooler do Neon (`postgresql+asyncpg://...`)
  - `REDIS_URL`: String de conexão Upstash (`rediss://...`)

### 4.3. Cache e Fila no Upstash Redis
- Cluster Redis Serverless gerenciado.
- Conexão TLS (`rediss://default:<token>@<endpoint>:6379`).
- Atende ao cache de cotações, `TickerCatalog` O(1) e filas assíncronas de workers sem necessidade de manter uma VPS para o Redis.

### 4.4. Ingestão Automatizada (`fin-ingestion`) via GitHub Actions
- Workflow `.github/workflows/scheduled-ingestion.yml` configurado no repositório `fin-ingestion`.
- **Gatilhos**:
  - Cron diário às 18:30 BRT (21:30 UTC), após fechamento da B3.
  - Gatilho manual `workflow_dispatch` para disparar coletas sob demanda via interface web do GitHub.
- **Execução**:
  - `uv run fin-ingestion collect --pipeline daily`
  - `uv run fin-ingestion enrich-metadata --limit 100`
- **Idempotência**: Garantida pela chave composta primária e regras `ON CONFLICT DO UPDATE` do modelo Neon, evitando duplicidade de ativos e cotações históricas.

### 4.5. Orquestração e Repositório de Infraestrutura (`fin-infra`)
- Centraliza a documentação operacional e o manifesto declarativo do Koyeb (`koyeb.yaml`).
- Fornece scripts de sincronização e validação das variáveis de ambiente entre os serviços.

---

## 5. O que fica a cargo do Executor (Left to Builder)
1. Ajustar o CORS no `fin-api` para aceitar requisições originadas do domínio Vercel (`https://*.vercel.app`).
2. Garantir que a porta do `Dockerfile` do `fin-api` seja dinâmica respeitando a variável `$PORT` do Koyeb caso necessário.
3. Criar os arquivos de workflow do GitHub Actions no repositório `fin-ingestion` com as chamadas CLI de coleta idempotente.
4. Criar a especificação do `koyeb.yaml` dentro de `fin-infra`.
5. Elaborar o guia passo a passo executável para conexão das contas gratuitas (Vercel, Koyeb, Upstash e GitHub Secrets).
