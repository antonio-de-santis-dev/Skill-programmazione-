---
name: react-state-redux
description: Progetta, implementa, rifattorizza e revisiona la gestione dello stato in applicazioni React - stato locale, lifting, Context API, useReducer, Redux Toolkit (store, createSlice, Immer, useSelector, useDispatch, createAsyncThunk, extraReducers, addMatcher, selettori, DevTools), stato nell'URL, e alternative come Zustand e TanStack Query. Usa questa skill ogni volta che si decide DOVE vive un dato o CHI lo possiede - "mi serve Redux?", "crea lo store", "aggiungi una slice", "il carrello deve essere condiviso fra le pagine", "passo questa prop attraverso cinque componenti", "come gestisco loading ed error nello store", "il componente si ridisegna troppo" - sia per impostare l'architettura da zero sia per rimettere in ordine uno stato che e' cresciuto male, anche quando l'utente non nomina Redux ma descrive solo il sintomo.
---

# Stato in React — architettura, Redux Toolkit, refactoring

Questa skill decide **dove vive un dato e chi lo possiede**. La scrittura dei componenti che lo
consumano resta compito di **react-development**.

La domanda a cui rispondere non è "quale libreria usiamo", ma: *chi ha bisogno di questo dato,
quando cambia, e chi deve accorgersene.* Le librerie vengono dopo.

---

## Passo 0 — Contesto (sempre, prima di proporre un'architettura)

1. `package.json` → `@reduxjs/toolkit`, `react-redux`, `zustand`, `@tanstack/react-query`, `jotai`,
   `recoil`. Cosa c'è già decide più di cosa è "migliore".
2. `src/main.jsx` / `index.js` → quali Provider avvolgono l'app, e in che ordine.
3. `src/redux/` o `src/store/` → come sono organizzate le slice esistenti, che convenzioni di
   naming e di cartelle seguono.
4. Due o tre componenti che consumano lo stato → come selezionano, come dispatchano, se ci sono già
   selettori riusabili.

**Non introdurre Redux in un progetto che non ce l'ha senza dirlo esplicitamente e senza motivarlo.**
Aggiungere uno store è una decisione architetturale che tocca tutta l'applicazione: va proposta,
non eseguita di nascosto perché sembrava ordinato.

Se il progetto ha già Redux, **scrivi nello stile che c'è**: se usa thunk scritti a mano invece di
`createAsyncThunk`, segnalalo una volta come possibile miglioramento e poi adeguati.

---

## Le cinque modalità

Se la richiesta è ambigua, la modalità di default è **DECIDE**: prima di scrivere uno store, va
stabilito che serva davvero.

### DECIDE — "mi serve Redux?", "dove metto questo dato?"
Usa la scala in `references/scelta-dello-stato.md`. Rispondi con una raccomandazione motivata, non
con un elenco di opzioni: chi chiede vuole una decisione, e il "dipende" senza conclusione non
aiuta nessuno.

### IMPLEMENT — "crea lo store", "aggiungi la slice del carrello"
Codice completo: store, slice, Provider, e almeno un componente che consuma, altrimenti non si vede
se funziona. Parti da `patterns/redux-recipes.md`.

### REFACTOR — "questo Context è ingestibile", "troppo prop drilling"
Una trasformazione per volta, con il comportamento preservato. I tre refactoring più frequenti:
prop drilling → Context, Context sovraccarico → Context separati o store, stato duplicato →
un'unica fonte con valori derivati.

### DEBUG — "render infiniti", "lo store non si aggiorna", "il componente non reagisce"
`references/diagnosi-stato.md` è organizzato per sintomo → causa → correzione. I Redux DevTools
mostrano ogni azione con lo stato prima e dopo: quasi sempre la risposta è lì, e vale più di
qualsiasi ipotesi.

### REVIEW — "guarda come ho organizzato lo stato"
Ordine per gravità più in basso.

---

## Dove trovare cosa

| Argomento | File |
|---|---|
| La scala di decisione: locale, lifting, URL, Context, Context+reducer, Redux, server state. Zustand e TanStack Query | `references/scelta-dello-stato.md` |
| Redux Toolkit completo: store, createSlice, Immer, useSelector/useDispatch, selettori, async, extraReducers, addMatcher, DevTools, struttura cartelle | `references/redux-toolkit.md` |
| Sintomo → causa → correzione, e checklist di review dello stato | `references/diagnosi-stato.md` |
| Codice pronto: slice carrello con totali automatici, slice API con loading ed errore, ricerca e paginazione, thunk | `patterns/redux-recipes.md` |

Per i componenti, gli hook, il routing e le chiamate API, la skill **react-development**.

---

## Il principio che regge tutto: una sola fonte di verità

La maggior parte dei bug di stato nasce dallo stesso errore: **lo stesso dato esiste in due posti**
e i due posti divergono.

Tre forme in cui si presenta, tutte da correggere allo stesso modo:

- Un valore **derivabile** tenuto nello stato. Il totale del carrello, il numero di elementi
  filtrati, il "form è valido" sono calcoli, non stato. Calcolali durante il render o con un
  selettore.
- Dati del server **copiati** nello stato locale. Ogni copia va risincronizzata a mano per sempre.
- Lo stesso dato in un Context **e** in uno `useState` di un componente, tenuti allineati da un
  `useEffect`. Quell'effetto è il sintomo, non la soluzione.

La domanda da farsi davanti a ogni pezzo di stato: *posso calcolarlo da qualcos'altro che ho già?*
Se sì, non è stato.

---

## Formato di output quando implementi

1. **Cosa stiamo costruendo** e perché lì (una o due righe di motivazione architetturale).
2. **File coinvolti**, con il percorso completo.
3. **Il codice**: store, slice, Provider, e il componente che consuma.
4. **Le decisioni non ovvie** — perché questa slice e non un'altra, perché il totale è calcolato
   con un matcher invece che con un'azione dedicata.
5. **Come verificare**: cosa fare nell'interfaccia e cosa deve comparire nei Redux DevTools.
6. **Cosa non è finito nello store, e perché.** È la parte che manca sempre e che evita che lo
   store diventi una discarica.

Procedi in modo incrementale: una slice che funziona, verificata nei DevTools, poi la successiva.

---

## Ordine di una review dello stato

1. **Correttezza** — stato mutato fuori da Immer, reducer non puri (fetch, `Date.now()`,
   `Math.random()` dentro un reducer), azioni dispatchate durante il render. Bloccano.
2. **Fonte di verità** — dati duplicati, valori derivabili salvati come stato, dati del server
   trattati come stato dell'applicazione.
3. **Confini** — cosa sta nello store e non dovrebbe (stato di un singolo componente, testo di un
   input mentre si digita, apertura di un modal usato in un punto solo).
4. **Selezione** — selettori che ritornano oggetti nuovi, componenti che selezionano interi rami
   dello store quando gli serve un campo.
5. **Leggibilità** — nomi delle azioni che descrivono l'evento (`checkoutConfermato`) e non
   l'assegnazione (`setDati`), slice per dominio e non per tipo di dato.

Un rilievo utile dice **cosa** non va, **perché** è un problema e **cosa fare**.

---

## Anti-overengineering

Lo stato globale è la fonte più comune di complessità accidentale in un'applicazione React. Non
introdurre:

- Redux per un'applicazione con tre pagine e un modal.
- Una slice per un booleano che riguarda un solo componente.
- `createAsyncThunk` per una chiamata che avviene in un punto solo e i cui dati non servono
  altrove.
- Un Context per due componenti padre e figlio.
- Lo stato dell'intera applicazione in un unico Context: ogni cambiamento ridisegna tutti i
  consumatori.
- Una normalizzazione a entità con `createEntityAdapter` su venti elementi.

Se basta uno `useState`, **dillo esplicitamente** invece di costruire lo store in silenzio. "Questo
dato serve solo qui: uno `useState` nel componente, e se domani servirà altrove lo spostiamo" è
una risposta migliore di una slice.

Vale anche al contrario: se l'utente sta passando la stessa prop attraverso cinque livelli e chiede
come farlo meglio, la risposta è un Context, non "va bene così".

---

## Quando passare la palla

| Segnale | Cosa fare |
|---|---|
| Componenti, hook, JSX, routing, form, chiamate API | Passa a **react-development** |
| Layout, colori, gerarchia visiva, accessibilità | Passa a **ui-ux-design** |
| L'API che alimenta lo store va scritta o cambiata, contratto dei dati lato server | Passa alla skill del backend (**java-spring-review** o equivalente) |
| "Quale servizio possiede questo dato" fra più sistemi, non dentro il frontend | Passa a **microservices-architecture** |
