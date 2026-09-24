---
name: react-development
description: Progetta, implementa, rifattorizza, testa, debugga e revisiona codice React (16.8-19). Copre componenti e JSX, props e children, liste e key, eventi, render condizionale, tutti gli hook (useState, useEffect, useRef, useReducer, useContext, useMemo, useCallback, useTransition, useDeferredValue, useId), custom hook, form controllati, Formik e Yup, React Router v6, data fetching con fetch e Axios, ottimizzazione delle performance, code splitting, struttura delle cartelle e deploy. Usa questa skill ogni volta che si scrive o si tocca codice React - "crea un componente", "fammi una lista con filtro", "aggiungi una pagina", "perche' l'useEffect gira due volte", "ho un loop infinito", "l'input non si aggiorna", "guarda questo componente", "come lo miglioro" - sia per costruire da zero sia per valutare codice esistente, anche quando l'utente non nomina React esplicitamente ma il contesto e' un progetto con package.json, JSX, Vite o create-react-app.
---

# React — implementazione, refactoring, review

Questa skill serve a **scrivere interfacce che funzionano**, non solo a giudicarle. La review è una
delle sei modalità, non l'unica.

---

## Passo 0 — Contesto e versione (sempre, prima di scrivere codice)

React cambia molto fra una versione e l'altra e le decisioni tecniche dipendono da fatti che vanno
letti, non assunti. Se hai accesso al progetto, leggi in quest'ordine e fermati appena hai quello
che ti serve:

1. `package.json` → **versione di React**, bundler (`vite` o `react-scripts`), `react-router-dom` e
   la sua versione, `@reduxjs/toolkit`, la libreria di stile (styled-components, tailwind, CSS
   Modules), TypeScript sì o no, `formik`/`react-hook-form`.
2. `vite.config.js` / `tsconfig.json` / `.env` → alias di percorso, prefisso delle variabili
   d'ambiente (`VITE_` con Vite, `REACT_APP_` con create-react-app: sbagliarlo significa
   `undefined` a runtime).
3. `src/main.jsx` o `src/index.js` → cosa avvolge l'app: `StrictMode`, `Provider` di Redux,
   `ThemeProvider`, il Router. Dice l'architettura in dieci righe.
4. Due o tre componenti simili a quello che stai per scrivere → **convenzioni reali**: cartelle
   (`components/` vs `screens/`), naming, export default o nominale, dove vivono gli stili, come
   si fanno le chiamate API.

**Se non riesci a determinare la versione, dichiara l'assunzione invece di inventarla.** Per
esempio: "assumo React 18+ con Vite; se sei su create-react-app le variabili d'ambiente hanno il
prefisso `REACT_APP_` e il comando di avvio cambia". Non è formalità: con React 17 non esiste
`createRoot`, con React 18 `useEffect` gira due volte in sviluppo sotto `StrictMode`, e questo è il
motivo di metà delle domande su "perché l'effetto parte due volte".

**Adatta il codice al progetto, non il progetto a un template.** Se il progetto usa ancora classi o
`defaultProps`, segnalalo una volta come miglioramento possibile ma scrivi nello stile esistente, a
meno che l'utente non stia chiedendo proprio di rifattorizzare.

---

## Le sei modalità

Riconosci quale ti viene chiesta e comportati di conseguenza. Se la richiesta è ambigua, la
modalità di default è **IMPLEMENT**: in caso di dubbio l'utente vuole un componente che funziona,
non la descrizione di cosa dovrebbe fare.

### IMPLEMENT — "crea", "aggiungi", "fammi un componente"
Produci codice completo e funzionante, non pseudocodice. Parti da `patterns/` per non riscrivere da
zero strutture già canoniche. Un componente che non compila perché manca un import non è una bozza:
è un errore.

### REFACTOR — "migliora", "pulisci", "questo componente è diventato enorme"
Cambia una cosa per volta e spiega perché. I refactoring più frequenti in React sono tre: estrarre
un custom hook per riusare logica, spezzare un componente che fa troppe cose, sostituire prop
drilling con Context. Preserva il comportamento osservabile.

### DEBUG — "non funziona", "loop infinito", "lo stato non si aggiorna"
Parti dal sintomo, non dalla soluzione. `references/diagnosi.md` è organizzato esattamente così:
sintomo → causa → correzione. Formula un'ipotesi, indica come verificarla (un `console.log`, il
Profiler di React DevTools), e solo dopo proponi la correzione.

### REVIEW — "guarda questo codice", "va bene?", "fai una code review"
Usa l'ordine per gravità più in basso. Marca ogni rilievo come **blocca** / **da sistemare** /
**spunto**: senza questa distinzione chi legge non sa cosa deve correggere davvero.

### TEST — "scrivi i test", "come testo questo componente"
Testing Library, non Enzyme: si testa cosa vede l'utente, non lo stato interno. Copri sempre almeno
uno stato di errore o di caricamento, non solo il percorso felice.

### EXPLAIN — "come funziona", "perché si fa così", "spiegami useEffect"
Spiega il meccanismo e il motivo, con un esempio minimo che si possa incollare e provare. Non
produrre file: la risposta sta nella conversazione.

---

## Dove trovare cosa

**Concetti, regole, criteri** → `references/`

| Argomento | File |
|---|---|
| Setup del progetto, JSX, componenti, props, children, liste e key, eventi, render condizionale, stili, deploy | `references/react-core.md` |
| Tutti gli hook: regole, useState, useEffect, useRef, useReducer, useContext, custom hook, useId | `references/hooks.md` |
| memo, useMemo, useCallback, useTransition, useDeferredValue, lazy + Suspense, quando NON ottimizzare | `references/performance.md` |
| Form controllati, validazione, Formik, Yup, alternative moderne | `references/forms.md` |
| React Router v6: rotte, Link, useParams, useNavigate, Outlet, rotte annidate, 404 | `references/routing.md` |
| Chiamate API: i tre stati, fetch vs Axios, istanza Axios, chiavi in `.env`, race condition, errori HTTP | `references/data-fetching.md` |
| CSS Modules, Tailwind, styled-components, styled-system, temi, varianti, classi condizionali | `references/styling.md` |
| Sintomo → causa → correzione, e checklist di review | `references/diagnosi.md` |

**Codice pronto da adattare** → `patterns/`

| Serve | File |
|---|---|
| Toggle e modal, lista con rimozione, filtro per categoria, slider, dark mode con localStorage, navbar animata, skeleton, copia negli appunti | `patterns/component-recipes.md` |
| useFetch, useTitle, useToggle, useLocalStorage, useDebounce, useWindowSize, useClickOutside | `patterns/hooks-recipes.md` |

I `patterns/` contengono codice funzionante da **adattare**, non da copiare alla lettera: nomi,
percorsi e forma dei dati vanno allineati al progetto reale.

---

## Formato di output quando implementi

Salta i punti che non si applicano — per un singolo handler aggiunto non serve il comando di
installazione.

1. **Cosa stiamo costruendo** — una o due righe.
2. **File coinvolti**, con il percorso completo di ognuno.
3. **Il codice**, completo per i file nuovi, come modifica precisa e localizzata per quelli
   esistenti.
4. **Le decisioni non ovvie** e il perché. Solo quelle: non commentare ogni riga.
5. **Dipendenze da installare** e variabili d'ambiente necessarie.
6. **Come provarlo** — cosa fare nel browser per vedere che funziona.
7. **Cosa controllare se non funziona** — di norma: console del browser, React DevTools, Network.

Se l'utente sta imparando React, procedi **in modo incrementale**: un pezzo che funziona,
verificato nel browser, poi il successivo. Dodici file insieme sono il modo più rapido per
perderlo. E fai vedere prima l'errore, poi la soluzione, quando il concetto lo richiede: capire
*perché* una variabile normale non aggiorna lo schermo vale più che imparare `useState` a memoria.

---

## Ordine di una review

Costruito perché ciò che rompe l'applicazione emerga prima delle osservazioni di stile.

1. **Correttezza dello stato e degli effetti** — mutazione diretta di stato o props, dipendenze di
   `useEffect` sbagliate o mancanti, effetti senza cleanup, hook chiamati dentro condizioni,
   `key={index}` su liste che cambiano. Questi producono bug reali e bloccano.
2. **Sicurezza** — chiavi API nel codice o in variabili senza `.gitignore`,
   `dangerouslySetInnerHTML` su contenuto non fidato, dati sensibili in localStorage.
3. **Struttura dei componenti** — componenti che fanno troppe cose, logica duplicata che vuole un
   custom hook, prop drilling oltre i due livelli, stato tenuto più in alto del necessario.
4. **Interfaccia e accessibilità** — stati di caricamento ed errore gestiti, liste vuote, `alt`
   sulle immagini, label collegate agli input, elementi cliccabili che sono `div` invece di
   `button`.
5. **Performance** — solo dopo le precedenti, e solo se misurata: render inutili, calcoli pesanti
   nel corpo del componente, bundle non spezzato.

Un commento utile dice **cosa** non va, **perché** è un problema e **cosa fare**. Senza il perché
chi legge impara solo a obbedire; senza il cosa fare il commento sposta il lavoro senza aiutare.

---

## Quando passare la palla

| Segnale nella richiesta | Cosa fare |
|---|---|
| Stato globale che cresce, store, slice, "serve Redux?", carrello condiviso fra pagine, Context che diventa ingestibile | Passa a **react-state-redux** per decidere l'architettura dello stato, poi torna qui per i componenti |
| Gerarchia visiva, palette, spaziature, accessibilità, "questa schermata non mi convince", scelta dei colori, layout responsive | Passa a **ui-ux-design**: qui si decide *come funziona*, lì *come si presenta* |
| L'API che il frontend consuma va scritta o cambiata, endpoint, DTO, CORS lato server | Passa a **java-spring-review** (o alla skill del backend in uso) |
| "mettilo in produzione", Dockerfile, pipeline, variabili d'ambiente in CI | Passa a **docker-kubernetes-devops** |

---

## Anti-overengineering

La complessità va proporzionata al problema. Non introdurre, se non serve davvero:

- `useMemo`, `useCallback` e `memo` sparsi ovunque per abitudine. Hanno un costo e rendono il
  codice più difficile da leggere. Prima misura con il Profiler, poi ottimizza il punto reale.
- Context per dati che servono a due componenti vicini: le props sono più esplicite e più semplici.
- `useReducer` per due booleani indipendenti.
- Una libreria di gestione form per un campo di ricerca.
- Un file per ogni componente di tre righe, con relativa cartella e `index.js`.
- Astrazioni "per il futuro" su componenti che esistono in un solo punto.

Se una soluzione più semplice basta, **dillo esplicitamente** invece di implementare quella
complessa in silenzio. "Qui basta uno `useState` nel componente, non serve il Context" è una
risposta migliore di un Provider.

---

## Nota sulle versioni

Il materiale didattico più diffuso è precedente al 2024 e usa `create-react-app`, `defaultProps` e
PropTypes. Oggi: **Vite** per creare il progetto (`npm create vite@latest nome -- --template
react`), **default nella destrutturazione** al posto di `defaultProps`, **TypeScript** al posto di
PropTypes nei progetti nuovi. La logica di React non cambia — cambia lo strumento. Quando l'utente
arriva da un corso più vecchio, insegnagli il concetto e segnala la differenza una volta sola,
senza trasformare ogni risposta in un elenco di deprecazioni.
