# FIRE GUI Restyle + Shortcut Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Modernizzare la GUI di FIRE Planner con un restyle leggero e creare un collegamento desktop che serve la build locale aggiornata.

**Architecture:** Solo CSS + markup locale alla dashboard; nessun cambio di logica, rotte o componenti condivisi. Lo shortcut è un `.bat` sul Desktop che avvia `vite preview` su `build/` e apre il browser.

**Tech Stack:** SvelteKit 2 + Svelte 5 runes, Tailwind CSS 4, Flowbite-Svelte, Apache ECharts 6, `vite preview`.

## Global Constraints

- Lingua UI/contenuti/commit: italiano (convenzione repo).
- Gate obbligatorio prima di ogni commit: `npm run check` deve passare senza errori.
- `npm run build` deve riuscire e includere le modifiche.
- Niente nuove dipendenze npm.
- Niente richieste di rete esterne (privacy-by-design: niente Google Fonts; `Inter` resta solo come preferenza se installato).
- Non modificare logica di calcolo, traduzioni i18n, guida interattiva, layout/sidebar/navbar.
- Commit convenzionali in italiano (`style:`, `feat:`, `fix:`, `docs:`).

---

## File Structure

- Modify: `fire20_repo/src/app.css` (intero file, 38 righe) — token di polish globale.
- Modify: `fire20_repo/src/routes/+page.svelte` (sezioni: welcome ~riga 286, description card ~riga 297, charts row ~riga 398) — hero + polish + tema ECharts dashboard.
- Create: `%Desktop%\FIRE Planner (locale).bat` — launcher (fuori repo, sul Desktop risolto via `[Environment]::GetFolderPath('Desktop')`).

---

### Task 1: Polish globale in `app.css`

**Files:**
- Modify: `fire20_repo/src/app.css:34-38` (blocco `@layer base`, estendere sotto `body`)

**Interfaces:**
- Consumes: token `@theme` esistenti (`--color-primary-*`, `--font-sans`).
- Produces: stili base globali usati da tutte le pagine (nessuna firma da ricordare).

- [ ] **Step 1: Aggiungere il blocco polish in `app.css`**

```css
@layer base {
	body {
		@apply bg-gray-50 text-gray-900 dark:bg-gray-900 dark:text-gray-100;
		@apply antialiased;
		text-rendering: optimizeLegibility;
	}
	::selection {
		@apply bg-primary-200 text-primary-900 dark:bg-primary-800 dark:text-primary-100;
	}
	:focus-visible {
		@apply outline-2 outline-offset-2 outline-primary-500;
	}
	* {
		scrollbar-width: thin;
		scrollbar-color: theme(--color-primary-300) transparent;
	}
}
```

- [ ] **Step 2: Verificare che il gate passi**

Run: `npm run check`
Expected: PASS (0 errori)

- [ ] **Step 3: Commit**

```bash
git add src/app.css
git commit -m "style: polish globale (smoothing, selezione, focus, scrollbar)"
```

---

### Task 2: Hero dashboard + polish card in `+page.svelte`

**Files:**
- Modify: `fire20_repo/src/routes/+page.svelte:286-322` (welcome + program description card)

**Interfaces:**
- Consumes: `FireSolid`, `Card`, `Heading`, `Badge` già importati nel file.
- Produces: nessun cambio di interfacce (solo classi/markup locali).

- [ ] **Step 1: Trasformare la description card in hero con gradiente**

Sostituire le classi della `Card` descrizione programma:
da `bg-blue-50 dark:bg-blue-900/20 border-blue-200 dark:border-blue-800`
a `bg-gradient-to-br from-primary-600 via-primary-700 to-primary-900 border-0 text-white shadow-lg`,
e adattare i testi interni da `text-gray-700 dark:text-gray-300` a `text-primary-50`,
il titolo da `text-blue-800 dark:text-blue-200` a `text-white`,
la nota privacy da `text-blue-700 dark:text-blue-300` a `text-primary-100`.
Aggiungere `hover:-translate-y-0.5 transition-transform` ai wrapper delle 6 `StatCard`
(la grid `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3`, ~riga 303): avvolgere ogni
`<StatCard ... />` in `<div class="transition-transform hover:-translate-y-1">...</div>`
senza toccare il componente condiviso.

- [ ] **Step 2: Verificare che il gate passi**

Run: `npm run check`
Expected: PASS (0 errori)

- [ ] **Step 3: Commit**

```bash
git add src/routes/+page.svelte
git commit -m "style: hero gradiente dashboard e hover sulle card riepilogo"
```

---

### Task 3: Tema ECharts coerente nei 2 grafici dashboard

**Files:**
- Modify: `fire20_repo/src/routes/+page.svelte:82-128` (`projectionChartOptions`) e `:130-182` (`allocationChartOptions`)

**Interfaces:**
- Consumes: `formatCurrency`, `formatCompact`, `projections`, `profile` già presenti.
- Produces: nessun cambio di interfacce (solo opzioni ECharts).

- [ ] **Step 1: Allineare font e tooltip alle palette Tailwind**

In entrambi gli oggetti options aggiungere in radice:
```js
textStyle: {
	fontFamily: 'Inter, ui-sans-serif, system-ui, -apple-system, sans-serif'
},
```
Nel `tooltip` di entrambi aggiungere:
```js
backgroundColor: 'rgba(17, 24, 39, 0.92)',
borderWidth: 0,
textStyle: { color: '#f9fafb', fontSize: 12 }
```
Nient'altro: colori serie, legende e markLine restano invariati.

- [ ] **Step 2: Verificare che il gate passi**

Run: `npm run check`
Expected: PASS (0 errori)

- [ ] **Step 3: Commit**

```bash
git add src/routes/+page.svelte
git commit -m "style: tema ECharts dashboard coerente (font, tooltip, dark-mode)"
```

---

### Task 4: Collegamento desktop locale (`.bat`)

**Files:**
- Create: `%Desktop%\FIRE Planner (locale).bat` (Desktop = output di `[Environment]::GetFolderPath('Desktop')`)

**Interfaces:**
- Consumes: `fire20_repo/build/index.html` (deve esistere; se manca il .bat lo segnala).
- Produces: server `vite preview` su `http://127.0.0.1:4173` + apertura browser.

- [ ] **Step 1: Creare il file `.bat` sul Desktop**

Contenuto esatto (path repo hardcoded, PC locale dell'utente):

```bat
@echo off
REM FIRE Planner (locale) - serve la build statica aggiornata e apre il browser.
REM Chiudere questa finestra per fermare il server.
cd /d "C:\Users\Stefano\OneDrive\Desktop\Arco  Monza\fire20_repo"
if not exist "build\index.html" (
    echo [ERRORE] Cartella build/ non trovata o non aggiornata.
    echo Esegui prima: npm run build
    pause
    exit /b 1
)
start "" "http://127.0.0.1:4173"
npm run preview -- --port 4173 --host 127.0.0.1
```

- [ ] **Step 2: Verificare che il file esista e il path repo sia corretto**

Run: `Test-Path "$([Environment]::GetFolderPath('Desktop'))\FIRE Planner (locale).bat"`
Expected: `True`

- [ ] **Step 3: Nessun commit (file fuori repo, sul Desktop)**

---

### Task 5: Rebuild finale, verifica visiva, push

**Files:**
- Generate: `fire20_repo/build/**` (output `npm run build`, gitignored, non committato)
- Push: branch `main` → `origin` (repo `Gurzo82/FIRE`)

**Interfaces:**
- Consumes: sorgenti aggiornati dai Task 1-3.
- Produces: `build/` aggiornata servita dal .bat del Task 4.

- [ ] **Step 1: Rigenerare la build statica**

Run: `npm run build`
Expected: termina con `Wrote site to "build"` e `✔ done`

- [ ] **Step 2: Verifica visiva**

Run: `npm run preview -- --port 4173 --host 127.0.0.1`
Expected: aprire `http://127.0.0.1:4173`, controllare hero gradiente, hover card, tooltip grafici, dark-mode; poi chiudere con Ctrl+C

- [ ] **Step 3: Push dei commit**

```bash
git log --oneline -4
git push origin main
```
Expected: i 3 commit style + fix precedenti risultano su `origin/main`
