# FIN·WEB — Terminal Institucional de Análise & Valuation Financeiro

> **Frontend do ecossistema FIN**: Interface futurista de alta precisão construída em **Next.js 16 (App Router)**, **React 19**, **TypeScript**, **Zod** e **CSS Puro / Tailwind v4**.

[![Next.js](https://img.shields.io/badge/Next.js-16.4.3-black?style=flat&logo=next.js)](https://nextjs.org/)
[![React](https://img.shields.io/badge/React-19.3.0-blue?style=flat&logo=react)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.9.3-3178C6?style=flat&logo=typescript)](https://www.typescriptlang.org/)
[![Zod](https://img.shields.io/badge/Zod-4.6.7-3E67B1?style=flat&logo=zod)](https://zod.dev/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## 1. Visão Geral & Engenharia Reversa (REA) dos 12 Portais

O `fin-web` foi concebido através da **Engenharia Reversa (REA)** dos 12 principais portais brasileiros de inteligência e dados de mercado, sintetizando as melhores ideias e superpoderes de cada um em uma experiência futurista, ultra-rápida e sem ruído visual de anúncios:

| # | Portal Analisado | Superpoderes Extraídos e Aperfeiçoados no `fin-web` |
|---|---|---|
| 1 | **Status Invest** | Séries históricas de 5 a 10 anos de DRE/DFC, monitor de Aluguel de Ações (BTC) e gregas de opções com preços teóricos Black-Scholes. |
| 2 | **Investidor10** | Checklist de critérios objetivos para Buy & Hold, decomposição de faturamento por segmento e mapa anual de dividendos. |
| 3 | **Fundamentei** | Visão de ultra-longo prazo (15–20 anos), KPIs operacionais setoriais (BOE/dia, celulose/ton, SSS) e governança CVM com Reclame Aqui. |
| 4 | **Fundamentus** | Velocidade de screening instantânea, transparência contábil tabular, programas de recompra e negociações de insiders. |
| 5 | **Clube FII** | Cockpit imobiliário com dados de padrão construtivo SiiLA, vendas/locações, cronograma de contratos e concentração de inquilinos. |
| 6 | **Funds Explorer** | Ranking consolidado de FIIs com P/VP, rentabilidade acumulada e vacância física/financeira histórica. |
| 7 | **FIIs.com.br** | Agenda de dividendos por mês de competência (`mes_referencia`) e calculadora do Magic Number (bola de neve de cotas). |
| 8 | **Mais Retorno** | Matriz Quantitativa de Risco 10x10 (Sharpe, Sortino, Treynor, Alfa de Jensen, Drawdown e recuperação) e carteira CDA de fundos CVM 175. |
| 9 | **Análise de Ações** | Modelos matemáticos de valuation automatizados (Preço Justo de Graham, Preço Teto de Bazin e DCF) com sliders interativos. |
| 10 | **Brick360** | Brick Score proprietário em 6 eixos (spider chart SVG), DRE CVM trimestral e parecer de Comitê de IA com análise SWOT. |
| 11 | **ADVFN Brasil** | Painel técnico com 13 indicadores em múltiplos timeframes, 4 suportes / 3 resistências e projeções de analistas sell-side. |
| 12 | **ETFs Brasil** | Custo Total Efetivo (TER primária + secundária), Spread NAV (ágio/deságio) e fluxo de criação/destruição de cotas em 30 dias. |

---

## 2. Princípios de Design & Arquitetura de Software

* **Sem Bibliotecas Externas para Layout:** Layout construído exclusivamente em **CSS puro moderno**, CSS Grid, Flexbox e variáveis semânticas. Sem Radix, Chakra, MUI, Mantine ou bloatware.
* **Double-Bezel Hardware Architecture:** Containers e cartões utilizam a estética de hardware usinado (outer shell com borda translúcida e padding milimétrico + inner core com fundo obsidian OLED e inset shadow).
* **Button-in-Button CTA Architecture:** Botões em pílula com ícone destacado em círculo individual animado por física de mola (`cubic-bezier(0.32, 0.72, 0, 1)`).
* **Floating Glass Island Navigation:** Barra de navegação flutuante com blur vítreo, desacoplada do topo e com drawer retrátil para mobile.
* **Gráficos em SVG Nativo Puro:**
  - `Sparkline.tsx`: Linhas suavizadas com gradiente de área e IDs estáticos via `useId()`.
  - `RadarChart.tsx`: Spider charts poligonais interativos para o Brick Score (6 eixos) e Checklist.
  - `BarChart.tsx`: Gráficos de barras interativos com tooltips e física de elevação para DRE e proventos.
* **Validação Rígida com Zod:** Todos os modelos de dados de ativos, cotações, demonstrativos contábeis, rankings e pareceres neurais são validados e tipados através de `src/schemas/financial.ts`.

---

## 3. Catálogo Completo de Rotas

```
fin-web/src/app/
├── layout.tsx                     # Shell Global (Ticker Tape, Floating Glass Nav, Footer)
├── page.tsx                       # Cockpit Principal (Bento Grid com radar macro, top ativos e Comitê IA)
├── acoes/
│   ├── page.tsx                   # Screener e Grid de Ações B3 com busca e múltiplos
│   └── [ticker]/
│       ├── page.tsx               # Server Component com SSG e generateStaticParams()
│       └── AcaoDetailClient.tsx   # Deep-dive (DRE 5 anos, Graham/Bazin com sliders, Checklist, Técnico)
├── fiis/
│   ├── page.tsx                   # Screener e Ranking de FIIs e Fiagros
│   └── [ticker]/
│       ├── page.tsx               # Server Component com SSG
│       └── FIIDetailClient.tsx    # Cockpit FII (Imóveis SiiLA, Brick Score 6 eixos, Magic Number)
├── fundos/
│   └── page.tsx                   # Fundos CVM 175, CDA Aberto e Matriz de Risco 10x10
├── etfs/
│   └── page.tsx                   # ETFs Locais & Globais, Custo Efetivo TER e Spread NAV
├── renda-fixa/
│   └── page.tsx                   # Tesouro Direto, PUs, Taxas Indicativas ANBIMA e Curva Zero ETTJ
├── screener/
│   └── page.tsx                   # Super Screener Universal Multiativo Ultra-Rápido
├── comparador/
│   └── page.tsx                   # Comparador Head-to-Head com Radar Chart SVG e retornos
├── carteira/
│   └── page.tsx                   # Gestão de Carteira, Projeção de Renda Passiva e Proventos
└── radar-ia/
    └── page.tsx                   # Central do Comitê de IA, NLP CVM e Análise de Sentimento
```

---

## 4. Instalação & Execução Local

```bash
# Navegar até o repositório
cd /Users/rafaelaparecido/fin-web

# Instalar dependências com pnpm
pnpm install

# Executar em modo desenvolvimento (Hot Reload)
pnpm dev

# Executar verificação e build de produção
pnpm build

# Iniciar servidor de produção
pnpm start
```

---

## 5. Ecossistema de Repositórios Relacionados

* **`fin-web`**: Frontend Next.js 16 + React 19 (`https://github.com/barrosrafa/fin-web`)
* **`fin-db`**: Banco de Dados PostgreSQL 18 / TimescaleDB no Neon Serverless (`https://github.com/barrosrafa/fin-db`)
* **`fin-api`**: Backend FastAPI com endpoints REST e WebSockets (`https://github.com/barrosrafa/fin-api`)
* **`fin-ingestion`**: Pipeline de coleta assíncrona com Prefect e scrapers regulatórios (`https://github.com/barrosrafa/fin-ingestion`)
