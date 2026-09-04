# Design: restyle leggero GUI + collegamento desktop locale (FIRE Planner)

Data: 2026-09-04 — Stato: approvato dall'utente (restyle approccio A + shortcut .bat locale).
Scope: solo FIRE Planner (`fire20_repo`). Niente Vercel/deploy. Niente SYRECO.

## 1. Restyle leggero (approccio A)

Obiettivo: modernizzare senza cambi strutturali, zero nuove dipendenze.

- `src/app.css`: stack font moderno di sistema, token radius/ombre, scrollbar sottile,
  focus-visible coerente, colore selezione testo. Solo CSS.
- `src/routes/+page.svelte` (dashboard): hero con gradiente che riusa la descrizione
  programma esistente in forma compattata; hover-lift su StatCard e Quick Actions.
- Grafici dashboard (proiezione portafoglio + allocazione): opzioni ECharts allineate
  al CSS (fontFamily esplicita, palette Tailwind blu/verde/ambra/rosso, tooltip leggibili,
  sfondi trasparenti per dark-mode coerente).
- Non toccare: layout/sidebar/navbar, logica di calcolo, traduzioni i18n, guida
  interattiva, pagine simulazione/calcolatore (descrizioni già aggiunte).

## 2. Collegamento desktop locale

- File: `%Desktop%\FIRE Planner (locale).bat` (Desktop risolto via
  `[Environment]::GetFolderPath('Desktop')`, non hardcoded).
- Comportamento: `cd` in `fire20_repo`; se `build/index.html` manca, messaggio che
  chiede di lanciare `npm run build` ed esce; altrimenti avvia
  `npm run preview -- --port 4173 --host 127.0.0.1` e apre
  `http://127.0.0.1:4173` nel browser predefinito. La console resta aperta
  (chiuderla ferma il server).
- Alternativa scartata: link diretto a `build/index.html` via `file://` —
  non funziona con SPA (routing/fetch). Motivo tecnico.
- Nota: dopo future modifiche al codice serve `npm run build` per aggiornare `build/`.

## 3. Verifica

- `npm run check` (type check, gate obbligatorio del repo) deve passare.
- `npm run build` deve riuscire e includere le modifiche.
- Verifica visiva via `npm run preview` + `.bat` dal Desktop.
- Nessun test automatizzato da aggiornare (nessuna logica toccata); guida invariata
  (nessuna nuova funzionalità utente).
