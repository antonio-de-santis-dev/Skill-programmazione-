# Diagnosi dei problemi di stato

Indice: [Come si indaga](#come-si-indaga) · [Non si aggiorna](#lo-stato-non-si-aggiorna) ·
[Render eccessivi](#render-eccessivi-e-loop) · [Redux](#problemi-specifici-di-redux) ·
[Context](#problemi-specifici-del-context) · [Dati incoerenti](#dati-incoerenti) ·
[Checklist](#checklist-di-review-dello-stato)

---

## Come si indaga

Con Redux, **prima i DevTools**: l'azione è stata dispatchata? con quale payload? il diff dello
stato è quello atteso? Tre domande che chiudono la maggior parte dei casi senza toccare il codice.

- L'azione **non compare** → il problema è nel componente: l'handler non parte, o il dispatch non
  viene chiamato.
- L'azione compare ma il **diff è vuoto** → il reducer non fa quello che credi, o hai dispatchato
  l'azione di un'altra slice.
- Il diff è corretto ma **l'interfaccia non cambia** → il problema è nella selezione, non nello
  stato.

Con Context o stato locale: React DevTools, scheda Components, per vedere lo stato **reale** del
componente nel momento del problema. E l'opzione "Highlight updates when components render" per
vedere a colpo d'occhio chi si ridisegna quando non dovrebbe.

---

## Lo stato non si aggiorna

| Sintomo | Causa | Correzione |
|---|---|---|
| Il reducer sembra corretto ma nulla cambia | Mutazione diretta **fuori** da Redux Toolkit: `useReducer` di React non ha Immer | Ritorna un nuovo oggetto: `{ ...state, campo: valore }` |
| `state.items.push(...)` non funziona in un `useReducer` di React | Stesso motivo: Immer esiste solo dentro `createSlice` | `items: [...state.items, nuovo]` |
| Un reducer con `state.x += 1` **e** un `return` | Immer non sa quale dei due usare | O modifichi la bozza, o ritorni un nuovo stato |
| Lo stato torna al valore iniziale a ogni interazione | Il Provider è dentro un componente che si rimonta, o la `key` del componente cambia | Sposta il Provider più in alto, verifica le key |
| L'azione non compare nei DevTools | Manca il dispatch, o è stato chiamato il reducer invece dell'action creator | `dispatch(azione(payload))`, non `azione(payload)` da solo |
| Dispatch dentro il corpo del componente, errore "Cannot update a component while rendering" | Un'azione dispatchata durante il render | Spostala in un handler o in un `useEffect` |

---

## Render eccessivi e loop

| Sintomo | Causa | Correzione |
|---|---|---|
| Render infiniti con `useSelector` | Il selettore costruisce un oggetto o un array nuovo a ogni chiamata | Selettori separati per ogni valore, o `createSelector` |
| Tutta l'applicazione si ridisegna a ogni digitazione | Il testo di un input tenuto in uno store globale | Tienilo locale, e propaga solo il valore confermato (o con debounce) |
| Tutti i consumatori di un Context si ridisegnano insieme | È il comportamento del Context: cambia il valore, cambiano tutti | Context separati per dominio, `useMemo` sul valore, oppure passa a uno store con selezione granulare |
| Il Context ridisegna anche quando nulla è cambiato | Il `value` è un oggetto letterale ricreato a ogni render del Provider | `useMemo` sul value, `useCallback` sulle funzioni esposte |
| Un componente si ridisegna a ogni cambiamento di una slice che non usa | Seleziona un ramo intero: `useSelector(s => s.cart)` | Seleziona il campo specifico |
| Un effetto che dipende da un valore dello store gira sempre | Il valore selezionato è un oggetto o un array nuovo ogni volta | `createSelector`, o dipendi da un primitivo |

---

## Problemi specifici di Redux

| Sintomo | Causa | Correzione |
|---|---|---|
| "A non-serializable value was detected in the state" | Nello store ci sono Date, Map, Set, funzioni, istanze di Error, nodi DOM | Metti valori serializzabili: una stringa ISO invece di una Date, un messaggio invece dell'oggetto Error |
| "Cannot read properties of undefined" dentro un selettore | Il nome della chiave in `configureStore` non corrisponde: `store.cart` vs `store.carrello` | Allinea i nomi; `useSelector(s => s)` mostra la struttura reale |
| Il thunk parte ma lo stato non cambia | Le azioni del thunk sono gestite in `reducers` invece che in `extraReducers` | Le azioni generate fuori dalla slice si intercettano in `extraReducers` |
| `action.payload` è `undefined` | L'action creator è stato chiamato senza argomento, o il thunk non ritorna nulla | Un thunk deve `return` il dato; il reducer riceve ciò che viene ritornato |
| Il matcher sembra non scattare | `startsWith` su un prefisso sbagliato: il tipo è `nomeSlice/nomeReducer`, dove `nomeSlice` è la proprietà `name` della slice, non il nome del file | Verifica il tipo esatto nei DevTools |
| Lo store si svuota al refresh | Redux vive in memoria, non persiste | `redux-persist`, oppure salva a mano su localStorage il poco che serve davvero |
| I totali sono giusti solo a volte | Ricalcolati con un'azione dedicata che qualcuno dimentica di dispatchare | `addMatcher` che ricalcola dopo ogni azione della slice, o un selettore derivato |

---

## Problemi specifici del Context

| Sintomo | Causa | Correzione |
|---|---|---|
| `Cannot destructure property ... of undefined` | Il componente è fuori dal Provider | Sposta il Provider più in alto; il custom hook con il `throw` esplicito rende l'errore comprensibile |
| Funziona in un componente e non in un altro | Ci sono due Provider annidati dello stesso Context: vince il più vicino | Un Provider solo, o Context distinti se è voluto |
| Il valore è quello iniziale, non quello del Provider | `createContext(valoreIniziale)` restituisce il default quando non c'è Provider sopra | Verifica l'albero in React DevTools |
| Il Context diventa ingestibile man mano che cresce | Sta facendo il lavoro di uno store | Spezzalo per dominio, o passa a Redux/Zustand |

---

## Dati incoerenti

Il sintomo generale: due punti dell'interfaccia mostrano valori diversi per la stessa cosa. La
causa è sempre la stessa — **lo stesso dato esiste in due posti**.

| Forma | Correzione |
|---|---|
| Il totale è salvato nello stato e ricalcolato a mano dopo ogni operazione | Calcolalo con un selettore derivato o un matcher: non è stato, è un calcolo |
| I dati dell'API sono copiati in uno `useState` locale oltre che nello store | Una sola copia; se servono entrambe le cose, il componente legge dallo store |
| Un `useEffect` che copia una prop in uno stato locale per "tenerli allineati" | Usa la prop direttamente; se serve uno stato derivato, calcolalo durante il render |
| Lo stesso filtro in uno stato React e nella query string | L'URL è la fonte di verità, lo stato React legge da lì |
| Il carrello nello store e in localStorage, aggiornati separatamente | Lo store è la fonte, localStorage è solo una copia scritta da un unico punto |

La domanda da farsi davanti a ogni campo dello stato: **posso calcolarlo da qualcos'altro che ho
già?** Se sì, non è stato.

---

## Checklist di review dello stato

**Correttezza (blocca)**
- [ ] Nessuna mutazione dello stato fuori dai reducer di Redux Toolkit
- [ ] Reducer puri: nessuna fetch, `Date.now()`, `Math.random()` al loro interno
- [ ] Nessun dispatch durante il render
- [ ] Solo valori serializzabili nello store

**Fonte di verità**
- [ ] Nessun valore derivabile salvato come stato (totali, conteggi, "è valido")
- [ ] I dati del server non sono duplicati fra store e stato locale
- [ ] Nessun `useEffect` il cui unico scopo è tenere allineati due stati

**Confini**
- [ ] Nello store c'è solo ciò che è davvero condiviso
- [ ] Il testo degli input mentre si digita è locale
- [ ] Filtri, ricerca e paginazione sono nell'URL se l'utente si aspetta di poterli condividere
- [ ] Le slice sono divise per dominio, non per tipo tecnico

**Selezione**
- [ ] I selettori ritornano il valore più specifico possibile
- [ ] Nessun oggetto o array costruito dentro un selettore senza `createSelector`
- [ ] I selettori riusati sono definiti accanto alla slice, non ripetuti nei componenti

**Leggibilità**
- [ ] I nomi delle azioni descrivono eventi accaduti, non assegnazioni
- [ ] Un nuovo arrivato capisce dove vive un dato guardando la cartella `redux/`
- [ ] La gestione di loading ed errore è uniforme fra le slice
