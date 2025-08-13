# Desafio — Carteira de Ações & Fundos

**Stack obrigatória:** React (Next.js) + Node (API Routes) + Supabase (Postgres/Auth) + Deploy na Vercel\
**Escopo de ativos:** somente **Ações (B3)** e **Fundos (ANBIMA)**\
**Atualização de preços:** **manual e incremental** (botão)

---

## Objetivo

Construir um app web que permita ao usuário:

1. **configurar a carteira** com **data de início** e **composição atual** (ativos + quantidades);
2. **validar** ativos ao cadastrar (ação deve existir na **B3**; fundo deve existir na **ANBIMA**);
3. **consultar preços diários** desde a data de início e calcular **retornos**:
   - **Últimos 30 dias (30d)**,
   - **Últimos 12 meses (12m)**,
   - **Desde o início (SI)**;
4. **atualizar dados manualmente** com um **botão** que busca **apenas o delta** desde a **última atualização** registrada.

---

## Telas (MVP)

1. **Configurar Carteira**

   - Campos: `start_date` (obrigatório), lista de **ativos** (símbolo, nome, classe `stock|fund`, **quantidade**).
   - **Validação de ativo** na adição:
     - **Ação (B3)**: checar existência do ticker na B3.
     - **Fundo (ANBIMA)**: checar existência do fundo nos dados públicos ANBIMA.
   - Mostrar status de validação e mensagem de erro amigável.

2. **Dashboard**

   - KPIs: **Retorno 30d**, **Retorno 12m**, **Retorno SI**.
   - **Posição atual por ativo**: quantidade × último preço (MTM).
   - **Gráfico** de evolução diária do valor da carteira.
   - **Botão “Atualizar cotações”**: dispara **integração incremental** (desde a última atualização → hoje) e exibe **progresso** + **timestamp** da última execução.

---

## Regras de Negócio

- **Série de preços**: buscar **preço de fechamento diário** desde `start_date` até hoje.
- **Incremental** (botão): para cada símbolo, defina `start_dt = (maior dt salva) + 1 dia` ou `start_date` se vazio; `end_dt = hoje`.
- **Retornos**:
  - **30d** e **12m**: janelas móveis de 30/365 dias corridos.
  - **Desde o início (SI)**: \((V_{hoje} / V_{start}) - 1\), onde \(V_t = \sum (qty_i \times preco_{i,t})\).
  - Se quiser, use **TWR**; caso contrário, documente a metodologia adotada no README.
- **Dias sem preço**: usar **último preço disponível anterior** na composição do valor diário.
- **Moeda**: BRL.

---

## Endpoints mínimos

- `POST /api/portfolio` — cria/atualiza `start_date` e composição (ativos + quantidades).
- `POST /api/asset/validate` — valida um ativo (B3/ANBIMA) antes de salvar.
- `POST /api/prices/refresh` — atualiza **delta** de preços (desde última atualização → hoje).
- `GET /api/portfolio/returns` — retorna `{ last_30d, last_12m, since_inception }`.
- `GET /api/positions` — retorna posição atual por ativo (qty, último preço, MTM).
- `GET /api/health` — saúde do serviço: `{ status: "ok" }`.

---

## Tabelas mínimas

- `portfolios` — carteira.
- `instruments` — cadastro de ativos (ação/fundo) e metadados.
- `holdings` — **composição atual** (instrumento + quantidade).
- `prices_daily` — preços diários por instrumento e data.

---

## Integrações de Dados (sugestões)

- **Ações / FIIs / ETFs (B3)**: arquivos históricos **COTAHIST** (oficial) ou APIs comunitárias (ex.: brapi) como **fallback**.
- **Fundos (ANBIMA/CVM)**: uso de cadastros públicos (CVM) e bases ANBIMA para **validação** e, quando disponível, **cota/NAV** (ex.: INF\_DIARIO via CVM).

> Trate **timeouts**, **retries** e, opcionalmente, um **fallback** se a fonte principal estiver indisponível. Faça **cache** no banco para evitar reconsultas no mesmo período.

---

## Avaliação

- **Corretude**: cálculos, dias sem preço, janelas.
- **Integração & Resiliência**: incremental correto, cache, tratamento de erros.
- **Arquitetura & Código**: organização FE/BE, tipagem, limpeza.
- **UX & UI**: responsivo, feedback claro, estados de erro/carregamento.
- **Deploy**: README conciso, .env seguro, CI opcional.

---

## Estrutura sugerida

```
.
├─ README.md
├─ .gitignore
├─ LICENSE
├─ app/                # Next.js (App Router)
│  ├─ page.tsx         # Dashboard
│  ├─ settings/page.tsx# Configurar Carteira
│  ├─ api/
│  │  ├─ health/route.ts
│  │  ├─ portfolio/route.ts
│  │  ├─ asset/validate/route.ts
│  │  ├─ prices/refresh/route.ts
│  │  └─ portfolio/returns/route.ts
├─ lib/                # integração, validação, cálculos
│  ├─ prices.ts
│  ├─ returns.ts
│  └─ validate.ts
└─ package.json
```

---

## Variáveis de ambiente (exemplo)

Crie um arquivo `.env.local` na raiz do projeto e configure:

```
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=   # usado apenas no backend se necessário

# APIs externas (se aplicável)
BRAPI_API_KEY=               # opcional, se usar brapi com chave
YF_RAPIDAPI_KEY=             # exemplo, se usar um proxy/serviço
```

> **Nunca** comite chaves reais. Forneça \`\` com placeholders.

---

## Como calcular os retornos (simples)

1. Construa a série diária de **valor da carteira**:\
   \(V_t = \sum_i (qty_i \times preco_{i,t})\).
2. **30d / 12m / SI**:
   - 30d: \(R_{30d} = V_{hoje}/V_{hoje-30} - 1\)
   - 12m: \(R_{12m} = V_{hoje}/V_{hoje-365} - 1\)
   - SI: \(R_{SI} = V_{hoje}/V_{start} - 1\)
3. **Dias sem preço**: preencha usando o **último preço conhecido anterior**.

> Você pode optar por **TWR**; se fizer isso, explique no README.

---

## Dicas de implementação

- Comece pelo **MVP**: telas, validação de ativos, endpoint de refresh, cálculo básico e dashboard.
- No botão de atualização, exiba **loading** e um **resumo** ao final (ativos processados, dias inseridos, falhas).
- Faça logs simples em JSON e trate erros com mensagens amigáveis.

---

## Bônus (opcionais)

- **Relatório CSV** da atualização com os dias inseridos por ativo.
- **Autocomplete** de ativos usando uma lista validada em cache.
- **Indicadores extras**: máxima/mínima do período, drawdown.

---

## .gitignore sugerido (Node/Next.js)

```gitignore
# dependencies
node_modules

# builds
.next
out

# env
.env*
!.env.example

# misc
.DS_Store
npm-debug.log*
yarn-debug.log*
yarn-error.log*
```

---

## Entrega

- Publique na **Vercel** (frontend + APIs) e use **Supabase** como banco/Auth.
- Inclua no README as **URLs públicas** e **passos de setup**.
- Abra PRs/commits de forma legível (mensagens claras).
- Por fim, envie o link do seu projeto para seu contato Manchester Investimentos com cópia para rh@manchesterinvest.com.br.

