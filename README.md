# FIRE Planner

[![Version](https://img.shields.io/github/package-json/v/nemorfol/fire20?label=version&color=blue)](https://github.com/nemorfol/fire20/releases)
[![License](https://img.shields.io/github/license/nemorfol/fire20?color=green)](LICENSE)
[![SvelteKit](https://img.shields.io/badge/SvelteKit-2.57-FF3E00?logo=svelte&logoColor=white)](https://kit.svelte.dev)
[![Svelte 5](https://img.shields.io/badge/Svelte-5-runes?logo=svelte&logoColor=white)](https://svelte.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-5-blue?logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-4-38B2AC?logo=tailwind-css&logoColor=white)](https://tailwindcss.com)
[![Tests](https://img.shields.io/badge/check-svelte--check-yellow)](https://github.com/nemorfol/fire20/actions)
[![Deploy](https://img.shields.io/badge/Deploy-Vercel-black?logo=vercel)](https://fire20.vercel.app)
[![YouTube](https://img.shields.io/badge/YouTube-Reference%20Video-red?logo=youtube)](https://www.youtube.com/watch?v=02JlWw1r6ug)

> **Applicazione web completa per la pianificazione F.I.R.E. (Financial Independence, Retire Early), pensata specificamente per il contesto fiscale e previdenziale italiano.**

🌐 **Live Demo:** [fire20.vercel.app](https://fire20.vercel.app)

Tutti i calcoli avvengono nel browser. I dati finanziari restano sul dispositivo dell'utente (IndexedDB) e non vengono mai inviati a server esterni. **Privacy by design.**

---

## 🎥 Ispirazione e Riferimento Video

Questo progetto è nato dall'ispirazione e dalla base tecnica condivisa nel video YouTube **"Simulazioni FIRE con Excel e Python: attenzione a inflazione e rendimenti iniziali"** di **Paolo Coletti** (@PaoloColetti), che dimostra l'approccio di simulazione Monte Carlo e analisi finanziaria con Excel e Python.

[![Simulazioni FIRE con Excel e Python](https://img.youtube.com/vi/02JlWw1r6ug/maxresdefault.jpg)](https://www.youtube.com/watch?v=02JlWw1r6ug)

**▶️ Guarda il video di riferimento:** https://www.youtube.com/watch?v=02JlWw1r6ug

> **Nota:** Il video di Paolo Coletti copre simulazioni finanziarie FIRE con Excel e Python (Monte Carlo, regola del 4%, guardrails, rischio sequenza rendimenti). FIRE Planner porta quell'approccio rigoroso in un'applicazione web completa, interattiva e **specifica per l'Italia** (IRPEF, INPS, TFR, fondi pensione, capital gains 26%/12.5%), con privacy totale (client-side only).

### Canale YouTube dell'autore
- **Canale:** [Paolo Coletti](https://www.youtube.com/@PaoloColetti)
- **Sito web:** [paolocoletti.com](https://www.paolocoletti.com)
- **Telegram:** [CanalePaoloCole](https://t.me/CanalePaoloCole)
- **GitHub:** [nemorfol](https://github.com/nemorfol)

---

## 📋 Funzionalità Principali

| Area | Descrizione |
|------|-------------|
| **Profilo Finanziario** | Multi-profilo, 6 sezioni, 9 classi asset, salvataggio IndexedDB |
| **Calcolatore FIRE** | FIRE Number, Coast FIRE, anni al FIRE, 4 strategie prelievo |
| **Monte Carlo** | 100k iterazioni, Web Worker, 3 modalità, fan chart P5-P95 |
| **Scenari Rischio** | 8 predefiniti + custom, stress test before/after |
| **Gestione Scenari** | Salvataggio, duplicazione, confronto side-by-side (max 4) |
| **Dati Storici** | 10 dataset (S&P 500 1928+, obbligazioni, oro, inflazione, CAPE) |
| **Fiscalità Italiana** | IRPEF 2026, capital gains, fondi pensione, TFR, INPS |
| **Guida Interattiva** | 20 capitoli A-Z con esempi reali, progress tracking |
| **Import/Export** | JSON, CSV, backup/ripristino, cancellazione selettiva |

---

## 🏗️ Tech Stack

| Tecnologia | Versione | Ruolo |
|---|---|---|
| [SvelteKit](https://kit.svelte.dev) | 2.57 | Framework full-stack |
| [Svelte 5](https://svelte.dev) | 5.55 | UI (runes mode: `$state`, `$derived`, `$effect`) |
| [Tailwind CSS](https://tailwindcss.com) | 4.2 | Styling utility-first |
| [Flowbite-Svelte](https://flowbite-svelte.com) | 1.33 | Componenti UI accessibili |
| [Apache ECharts](https://echarts.apache.org) | 6.0 | Grafici interattivi |
| [Dexie.js](https://dexie.org) | 4.4 | Wrapper IndexedDB type-safe |
| [jStat](https://jstat.github.io) | 1.9 | Distribuzioni statistiche |
| [PapaParse](https://www.papaparse.com) | 5.5 | Parsing CSV |
| [jsPDF](https://github.com/parallax/jsPDF) | latest | Export PDF report |
| [xlsx](https://github.com/SheetJS/sheetjs) | latest | Export Excel |
| [@vite-pwa/sveltekit](https://vite-pwa-org.netlify.app) | latest | PWA (service worker, offline) |

---

## 🔬 Fonti Dati e Riferimenti

### Fonti Dati Primarie (Dataset Storici Embedded)

| Dataset | Fonte | Periodo | Note |
|---------|-------|---------|------|
| **S&P 500 Total Return** | [Damodaran, NYU Stern](https://pages.stern.nyu.edu/~adamodar/New_Home_Page/datafile/histretSP.html) | 1928–2024 | Rendimenti nominali e reali |
| **CAPE Ratio (Shiller PE)** | [Robert Shiller, Yale](http://www.econ.yale.edu/~shiller/data.htm) | 1881–presente | Cyclically Adjusted PE |
| **Obbligazioni USA 10Y** | [Damodaran / FRED](https://fred.stlouisfed.org) | 1928–2024 | Rendimenti decennali |
| **Oro (USD/oz)** | [World Gold Council / FRED](https://www.gold.org) | 1968–2024 | Prezzo spot Londra |
| **Inflazione USA (CPI)** | [Bureau of Labor Statistics](https://www.bls.gov/cpi/) | 1913–2024 | CPI-U All Items |
| **Inflazione Italia (NIC/FOI)** | [ISTAT](https://www.istat.it) / [World Bank](https://data.worldbank.org) | 1955–2024 | Indice prezzi al consumo |
| **MSCI World** | [MSCI / Vanguard](https://www.vanguard.com) | 1970–2024 | Azionario globale developed |
| **Real Estate (REITs)** | [NAREIT / FRED](https://www.nareit.com) | 1972–2024 | Indice FTSE NAREIT |
| **Correlazioni Asset** | [Morningstar](https://www.morningstar.com) / [Vanguard](https://www.vanguard.com) | Variabile | Matrici rolling |
| **Risk-Free Rate** | [FRED - DGS10/DGS3MO](https://fred.stlouisfed.org) | 1962–2024 | Treasury USA |

### Riferimenti Metodologici

| Argomento | Fonte |
|-----------|-------|
| **Regola del 4% (Trinity Study)** | Bengen (1994), Cooley et al. (1998), Trinity University |
| **Safe Withdrawal Rate** | Pfau (2010-2024), Kitces, EarlyRetirementNow |
| **Guyton-Klinger Guardrails** | Guyton & Klinger (2006), *J. Financial Planning* |
| **VPW (Variable Percentage Withdrawal)** | Bogleheads Wiki, Longinvest |
| **CAPE-Based Withdrawal** | Pfau & Kitces, *Reducing Retirement Risk* |
| **Monte Carlo Finanziario** | Glasserman (2004), *Monte Carlo Methods in Financial Engineering* |
| **Bootstrap Storico** | Politis & Romano (1994), Block Bootstrap |
| **Sequenza Rendimenti (Sequence Risk)** | Kitces (2014), Pfau (2017) |

### Normativa Fiscale Italiana (2026)

| Elemento | Parametro | Fonte |
|----------|-----------|-------|
| **IRPEF** | 23% ≤28k, 33% 28k–50k, 43% >50k | [Agenzia Entrate](https://www.agenziaentrate.gov.it) |
| **Capital Gains** | 26% azioni/ETF, 12.5% titoli Stato | [D.Lgs. 461/1997](https://www.normattiva.it) |
| **Fondo Pensione** | Deduzione 5.300€, rendimenti 20%, prestazioni 9–15% | [COVIP](https://www.covip.it) |
| **TFR** | Rivalutazione 1.5% + 75% inflazione | [Art. 2120 C.C.](https://www.brocardi.it) |
| **INPS Contributiva** | Coefficienti trasformazione 2025 | [INPS Circolare](https://www.inps.it) |
| **Bollo/IVAFE** | 0.20% / 0.2% (estero) | [Agenzia Entrate](https://www.agenziaentrate.gov.it) |

---

## 🚀 Avvio Rapido

### Prerequisiti
- **Node.js** ≥ 20
- **npm** ≥ 10 (o pnpm/yarn)

```bash
# Clona il repository
git clone https://github.com/nemorfol/fire20.git
cd fire20

# Installa dipendenze
npm install

# Avvia in sviluppo (Vite + SvelteKit)
npm run dev

# Build per produzione (statico in build/)
npm run build

# Preview build locale
npm run preview

# Type check + diagnostica Svelte (gate obbligatorio pre-commit)
npm run check

# Rigenera icone PWA
npm run generate-icons
```

### Deploy su Vercel (Consigliato)
1. Importa il repository su [Vercel](https://vercel.com)
2. Framework rilevato automaticamente (SvelteKit + `@sveltejs/adapter-static`)
3. Build produce file statici in `build/` → deploy automatico

**Altri hosting statici supportati:** Netlify, GitHub Pages, Cloudflare Pages, Firebase Hosting, qualsiasi CDN.

---

## 📁 Struttura del Progetto

```
src/
├── routes/                    # 14 pagine SvelteKit (file-based routing)
│   ├── +page.svelte           # Dashboard
│   ├── +layout.svelte/.ts     # Layout root + config (prerender/ssr)
│   ├── profilo/               # Profilo finanziario (6 sezioni)
│   ├── calcolatore/           # Calcolatore FIRE + strategie prelievo
│   ├── simulazione/           # Monte Carlo (Web Worker)
│   ├── scenari/               # Scenari salvati + confronto
│   ├── rischi/                # Stress test 8 scenari predefiniti
│   ├── dati-storici/          # Esplorazione 10 dataset + grafici
│   ├── pensione/              # Pensione INPS + gap pensionistico
│   ├── fondo-pensione/        # Fondo pensione + TFR
│   ├── contenitori/           # Confronto contenitori fiscali
│   ├── confronto-profili/     # Confronto multi-profilo
│   ├── performance/           # TWR + analisi performance
│   ├── community/             # Condivisione anonima parametri
│   ├── guida/                 # Guida interattiva 20 step
│   └── impostazioni/          # Import/Export, backup, privacy
├── lib/
│   ├── engine/                # MOTORE DI CALCOLO (TS puro, no UI)
│   │   ├── fire-calculator.ts    # FIRE Number, Coast FIRE, saving rate
│   │   ├── monte-carlo.ts        # Simulazione MC (parametrica/bootstrap)
│   │   ├── statistics.ts         # Utilities statistiche
│   │   ├── withdrawal.ts         # 4 strategie: 4%, VPW, Guardrails, CAPE
│   │   ├── tax-italy.ts          # IRPEF, capital gains, bollo/IVAFE
│   │   ├── pension-italy.ts      # Pensione INPS contributiva
│   │   ├── inps-simulator.ts     # Simulatore INPS dettagliato
│   │   ├── pension-fund.ts       # Fondo pensione (deduzione, tassazione)
│   │   ├── tfr-rules.ts          # TFR rivalutazione
│   │   ├── risk-scenarios.ts     # 8 scenari stress predefiniti
│   │   ├── sensitivity.ts        # Analisi sensitività (tornado)
│   │   ├── container-comparison.ts # Confronto conti fiscali
│   │   ├── couple.ts / family.ts # Planner coppia/famiglia
│   │   ├── life-events.ts        # Eventi vita (bonus, spese una tantum)
│   │   ├── assumptions.ts        # Ipotesi condivise
│   │   └── twr-calculator.ts     # Time-weighted return
│   ├── db/                    # Dexie/IndexedDB (schema + CRUD)
│   │   ├── index.ts              # Database connection
│   │   ├── profiles/             # CRUD profili
│   │   ├── results/              # CRUD risultati simulazioni
│   │   └── scenarios/            # CRUD scenari rischio
│   ├── workers/               # Web Workers (off-main-thread)
│   │   └── monte-carlo.worker.ts # Simulazioni MC non bloccanti
│   ├── data/                  # Dataset storici embedded (TS/JSON)
│   │   ├── sp500.ts / bonds.ts / gold.ts / inflation.ts / cape.ts / msci.ts / reits.ts / correlations.ts
│   ├── guida/                 # Contenuti guida interattiva
│   │   └── steps.ts              # 20 capitoli + metadata
│   ├── components/            # 44+ componenti Svelte riusabili
│   ├── i18n/                  # Stringhe localizzate (IT)
│   ├── utils/                 # Formattazione, import/export, validazione
│   └── assets/                # Asset statici
```

---

## 🧮 Motore di Calcolo — Dettagli Tecnici

### Strategie di Prelievo Implementate

| Strategia | Descrizione | Parametri Chiave |
|-----------|-------------|------------------|
| **Regola del 4% (Fissa)** | Prelievo costante reale 4% anno 1, aggiustato inflazione | `withdrawal_rate: 0.04` |
| **VPW** | Percentuale variabile basata su aspettativa di vita residua | Tabella VPW per asset allocation |
| **Guyton-Klinger Guardrails** | Band ±20% da tasso iniziale, aggiustamento annuale | `initial_rate`, `guardrail_pct: 0.20`, `floor/ceiling` |
| **CAPE-Based** | Tasso prelievo funzione CAPE Shiller (inverso) | `cape_slope`, `cape_intercept`, `min/max_rate` |

### Monte Carlo — 3 Modalità

| Modalità | Descrizione | Uso Consigliato |
|----------|-------------|-----------------|
| **Parametrica (Log-normale)** | Campioni da distribuzione parametrica (μ, σ) | Analisi forward-looking, stress test |
| **Bootstrap Storico** | Ricampionamento con ripetizione dai rendimenti reali | Backtesting, preserva fat-tail/autocorrelazione |
| **Block Bootstrap** | Blocchi contigui (es. 5 anni) per preservare struttura temporale | Sequence risk, regime change |

**Output:** Fan chart (P5-P95), istogramma finale, tasso successo, scenari P10/P50/P90.

---

## 🔒 Privacy & Sicurezza

- **Zero backend** — Nessun server, database, API proprietarie
- **Dati solo locale** — IndexedDB nel browser dell'utente
- **Nessun tracking** — No analytics, no telemetria, no cookie non essenziali
- **PWA Offline** — Service worker per funzionamento offline completo
- **CSP Strict** — Content Security Policy restrittiva
- **HTTPS Only** — Deploy su Vercel/Netlify forza HTTPS

---

## 🤝 Contribuire

I contributi sono benvenuti! Il progetto **accetta Pull Request su GitHub**.

### Workflow Consigliato

1. **Fork** → crea branch feature (`git checkout -b feat/nome-funzionalita`)
2. **Sviluppa** → mantieni stile esistente (italiano, Svelte 5 runes mode)
3. **Verifica** → `npm run check` (deve passare senza errori)
4. **Documenta** → Ogni nuova feature utente richiede capitolo in `src/lib/guida/steps.ts`
5. **PR** → Apri Pull Request con descrizione chiara

### Convenzioni

- **Lingua:** Italiano (UI, commenti, commit messages)
- **Commit:** Conventional commits in italiano → `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`
- **Fiscale:** Verifica sempre aliquote/soglie/coefficienti prima di modificare; cita la fonte
- **Architettura:** Logica dominio in `src/lib/engine/` (TS puro), UI in `src/routes/` + `src/lib/components/`

---

## 📄 Licenza

Distribuito con licenza **MIT** — vedi [LICENSE](LICENSE).

Sei libero di usare, modificare e distribuire il codice, mantenendo l'avviso di copyright.

---

## ⚠️ Disclaimer

> **Questa applicazione è uno strumento educativo e di simulazione. Non costituisce consulenza finanziaria, fiscale o legale. I risultati sono basati su assunzioni e dati storici che non garantiscono performance future. Consulta un professionista qualificato (consulente finanziario, commercialista, avvocato) prima di prendere decisioni di investimento o previdenziali.**

---

## 🔍 Visibilità e Scopribilità (SEO GitHub)

**Parole chiave per ricerca:**
```
FIRE, Financial Independence Retire Early, Monte Carlo simulation,
withdrawal strategies, 4% rule, VPW, Guyton-Klinger, CAPE-based,
Italian tax, IRPEF, INPS, pension fund, TFR, capital gains,
SvelteKit, Svelte 5, TypeScript, Tailwind CSS, ECharts,
IndexedDB, Dexie, Web Worker, PWA, privacy-first
```

**Topic GitHub suggeriti:**
`fire`, `financial-independence`, `retire-early`, `monte-carlo`, `withdrawal-strategies`, `italian-tax`, `pension-planning`, `sveltekit`, `svelte-5`, `typescript`, `echarts`, `indexeddb`, `privacy-first`, `pwa`, `financial-planning`

---

## 📞 Supporto e Contatti

- **Issues:** [GitHub Issues](https://github.com/nemorfol/fire20/issues)
- **Discussioni:** [GitHub Discussions](https://github.com/nemorfol/fire20/discussions)
- **Demo Live:** [fire20.vercel.app](https://fire20.vercel.app)
- **Autore:** [Paolo Coletti](https://www.paolocoletti.com) / [@PaoloColetti](https://www.youtube.com/@PaoloColetti)

---

## 🙏 Ringraziamenti

- **Paolo Coletti** — Per il video ["Simulazioni FIRE con Excel e Python"](https://www.youtube.com/watch?v=02JlWw1r6ug) che ha ispirato l'approccio computazionale rigoroso
- **Comunità Bogleheads / EarlyRetirementNow / Kitces** — Per la ricerca sui safe withdrawal rate
- **Autori dataset:** Damodaran (NYU), Shiller (Yale), BLS, ISTAT, World Bank, FRED, MSCI, Vanguard, Morningstar
- **Open Source:** Svelte/SvelteKit team, Tailwind CSS, Apache ECharts, Dexie.js, jStat, PapaParse, jsPDF, SheetJS, Vite PWA

---

## 📈 Roadmap

- [ ] **Coast FIRE / Barista FIRE** — Calcolatori dedicati
- [ ] **Eredità / Donazioni** — Pianificazione successoria
- [ ] **Multi-valuta** — Supporto EUR/USD/CHF per expat
- [ ] **API Pubbliche** — Integrazione opzionale quotazioni real-time (opt-in)
- [ ] **Mobile App** — Capacitor/TAURI per app native iOS/Android
- [ ] **Collaborazione** — Condivisione profili tra coniugi/famiglia (P2P WebRTC)

---

> **FIRE Planner** — *Pianificazione finanziaria rigorosa, privacy totale, fatta per l'Italia.*

**Se questo strumento ti è utile, lascia una ⭐ su GitHub e condividi con chi pianifica l'indipendenza finanziaria!**