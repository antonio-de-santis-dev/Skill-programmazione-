# Diagnosi e checklist di review

Indice: [Come si indaga](#come-si-indaga) · [Non si aggiorna](#lo-schermo-non-si-aggiorna) ·
[Loop e render doppi](#loop-infiniti-e-render-doppi) · [Crash](#crash-e-schermate-bianche) ·
[Errori in console](#warning-e-errori-in-console) · [Liste e form](#liste-e-form) ·
[Dopo il deploy](#problemi-che-compaiono-solo-dopo-il-deploy) ·
[Checklist di review](#checklist-di-review)

---

## Come si indaga

Prima di proporre una correzione, isola il sintomo. Tre strumenti, in quest'ordine:

1. **La console del browser.** React scrive messaggi molto precisi, e quasi sempre nominano il
   componente. Chiedi il testo esatto dell'errore se manca: "non funziona" non è diagnosticabile.
2. **React DevTools**, scheda Components: guarda props e stato **reali** del componente nel momento
   del problema. Metà dei bug sono dati diversi da quelli che si immagina di avere.
3. **La scheda Network**, quando c'è di mezzo un'API: la richiesta è partita? con quale URL? cosa
   ha risposto?

Formula un'ipotesi, indica come verificarla, poi correggi. Proporre tre correzioni insieme senza
sapere quale sia la causa insegna solo a provare a caso.

---

## Lo schermo non si aggiorna

| Sintomo | Causa | Correzione |
|---|---|---|
| Cambio una variabile, la console stampa il valore nuovo, lo schermo resta fermo | È una variabile normale, non stato: React non sa che è cambiata e comunque il valore si perde al render | `useState` |
| Chiamo il setter ma non cambia nulla | Mutazione diretta di array/oggetto: il riferimento è lo stesso, React lo considera invariato | Crea una struttura nuova: `[...lista, x]`, `{...obj, k: v}`, `map`, `filter` |
| Tre `setValore(valore + 1)` di fila aumentano di 1 | `valore` è congelato al render corrente e gli aggiornamenti sono raggruppati | Forma funzionale: `setValore(prev => prev + 1)` |
| Il valore dentro `setInterval`/`setTimeout` resta sempre quello iniziale | Stale closure: il timer ha catturato il valore del render in cui è stato creato | Forma funzionale nel setter, oppure un `useRef` per il valore corrente |
| Leggo lo stato subito dopo il setter e vedo il vecchio valore | Gli aggiornamenti non sono sincroni | Usa il valore che stai per impostare, o leggilo in un `useEffect` su quella dipendenza |
| Un componente riceve la prop nuova ma mostra quella vecchia | Stato inizializzato da una prop: `useState(props.valore)` usa il valore solo al primo render | Deriva il valore durante il render, oppure `key` diversa per rimontare il componente |

---

## Loop infiniti e render doppi

| Sintomo | Causa | Correzione |
|---|---|---|
| "Too many re-renders" appena carico la pagina | Una funzione viene **eseguita** invece che passata: `onClick={elimina()}` | `onClick={() => elimina(id)}` |
| La fetch parte all'infinito | `useEffect` senza array di dipendenze che aggiorna lo stato | Aggiungi `[]` o le dipendenze giuste |
| L'effetto gira a ogni render anche con le dipendenze | Una dipendenza è un oggetto, un array o una funzione: riferimento nuovo ogni volta | Metti valori primitivi, o stabilizza con `useMemo`/`useCallback` |
| L'effetto gira **due volte** al montaggio, solo in sviluppo | `StrictMode` monta, smonta e rimonta di proposito | Non è un bug. Se dà fastidio è perché manca la cleanup: aggiungila (è esattamente ciò che StrictMode vuole far emergere) |
| Un componente memoizzato si ridisegna comunque | Riceve una funzione o un oggetto creato nel genitore | `useCallback` / `useMemo` sul valore passato |
| Render infiniti con Redux | Un selettore che costruisce un oggetto nuovo a ogni chiamata | Selettori separati per ogni valore, o `createSelector` |

---

## Crash e schermate bianche

| Errore | Causa | Correzione |
|---|---|---|
| `Cannot read properties of undefined (reading 'map')` | I dati non sono ancora arrivati, o lo stato iniziale è `null` | Inizializza con `[]`, o usa l'optional chaining `dati?.map(...)` con un fallback |
| `Cannot read properties of undefined (reading 'nome')` dentro un dettaglio | Un `find` che non ha trovato nulla (spesso: id stringa dall'URL confrontato con `===` a un id numerico) | `Number(id)` nel confronto, e gestisci il caso non trovato con un return anticipato |
| `Objects are not valid as a React child` | Stai renderizzando un oggetto invece di una stringa: `{utente}` invece di `{utente.nome}` | Renderizza una proprietà, o `JSON.stringify` per il debug |
| `Rendered fewer hooks than expected` | Un hook dentro un `if`, o dopo un return condizionale | Tutti gli hook in cima, le condizioni dentro l'hook |
| `Invalid hook call` | Hook chiamato fuori da un componente o da un custom hook — oppure due copie di React nel progetto | Controlla dove lo chiami; se è corretto, verifica le dipendenze duplicate |
| Pagina bianca senza errori in console | Un componente ritorna `undefined` (manca il `return`, o c'è un `return` a capo prima del JSX) | Verifica i return; con le parentesi dopo `return (` il problema sparisce |

Un **Error Boundary** attorno alle sezioni principali evita che un errore in un componente faccia
sparire l'intera applicazione:

```jsx
<ErrorBoundary fallback={<p>Qualcosa è andato storto.</p>}>
  <Dashboard />
</ErrorBoundary>
```

Non intercetta gli errori dentro gli handler e nel codice asincrono: quelli restano compito di
try/catch.

---

## Warning e errori in console

| Messaggio | Significato | Correzione |
|---|---|---|
| `Each child in a list should have a unique "key" prop` | Manca la key in un `map` | `key={item.id}`, mai `Math.random()` |
| `A component is changing an uncontrolled input to be controlled` | `value` inizialmente `undefined` | Inizializza lo stato con `""` |
| `You provided a value prop without an onChange handler` | Input bloccato | Aggiungi `onChange`, o usa `readOnly` se volevi davvero bloccarlo |
| `Can't perform a React state update on an unmounted component` | Una risposta asincrona arriva dopo lo smontaggio | `AbortController` nella cleanup |
| `Warning: validateDOMNesting` | HTML non valido: un `<div>` dentro un `<p>`, un `<li>` fuori da una lista | Correggi la struttura, non ignorare: rompe anche il CSS |
| `react-hooks/exhaustive-deps` | Una dipendenza usata ma non dichiarata | Dichiarala; se l'effetto gira troppo, stabilizza la dipendenza invece di togliere il warning |

---

## Liste e form

| Sintomo | Causa | Correzione |
|---|---|---|
| Elimino un elemento e il testo scritto in un input "salta" su un'altra riga | `key={index}`: gli indici scalano e React associa lo stato all'elemento sbagliato | `key` dall'id stabile dei dati |
| Compare uno **0** solitario nella pagina | `{lista.length && <Lista />}`: React non renderizza i booleani ma i numeri sì | `{lista.length > 0 && <Lista />}` |
| Il form ricarica la pagina al submit | Manca `e.preventDefault()` | Aggiungilo, e metti l'`onSubmit` sul `<form>` |
| Il campo numerico si comporta in modo strano nei calcoli | `e.target.value` è sempre una stringa | `Number(e.target.value)` alla lettura |
| L'errore di validazione compare prima che l'utente scriva | Manca il concetto di "campo toccato" | Mostra l'errore solo dopo `onBlur` (`touched`) |
| Doppio invio dell'ordine | Il bottone resta attivo durante la chiamata | Stato `isSubmitting` e `disabled` |

---

## Problemi che compaiono solo dopo il deploy

| Sintomo | Causa | Correzione |
|---|---|---|
| 404 se ricarico la pagina su una rotta diversa dalla home | Il server cerca un file che non esiste | Regola di rewrite verso `index.html` (vedi `routing.md`) |
| Le variabili d'ambiente sono `undefined` | Prefisso sbagliato (`VITE_` vs `REACT_APP_`), oppure `.env` non presente sul servizio di build | Controlla il prefisso e le variabili configurate nel pannello di Netlify/Vercel |
| Le immagini non si vedono | Percorsi assoluti dal filesystem locale, o file in `src/` referenziati come stringa | Importa le immagini, o mettile in `public/` e usa percorsi relativi alla root |
| Funziona in locale, in produzione compaiono errori CORS | In sviluppo c'era il proxy del dev server | Il server deve dichiarare le origini consentite: è un problema lato backend |
| `navigator.clipboard` non funziona | Richiede HTTPS o localhost | Non è un bug: è una restrizione di sicurezza del browser |

---

## Checklist di review

Nell'ordine in cui vanno guardate. Marca ogni rilievo come **blocca** / **da sistemare** /
**spunto**.

**1. Stato ed effetti (blocca)**
- [ ] Nessuna mutazione diretta di stato o props
- [ ] Dipendenze degli effetti corrette, nessun warning `exhaustive-deps` zittito
- [ ] Ogni timer, listener e sottoscrizione ha la sua cleanup
- [ ] Nessun hook dentro condizioni o dopo un return
- [ ] `key` stabili e derivate dai dati
- [ ] Forma funzionale del setter dove il nuovo stato dipende dal precedente

**2. Sicurezza (blocca)**
- [ ] Nessuna chiave API a consumo nel frontend; `.env` in `.gitignore`
- [ ] `dangerouslySetInnerHTML` assente, o su contenuto sanificato
- [ ] Niente token o dati sensibili in localStorage se evitabile
- [ ] Nessun controllo di autorizzazione affidato solo al client

**3. Struttura**
- [ ] Componenti con una responsabilità riconoscibile; se la funzione supera le ~150 righe, chiediti cosa estrarre
- [ ] Logica duplicata in due componenti → custom hook
- [ ] Prop drilling oltre due livelli → Context (ma non per due componenti vicini)
- [ ] Stato nel componente più basso che lo usa davvero
- [ ] Chiamate API in un modulo dedicato, non sparse nei componenti

**4. Interfaccia e accessibilità**
- [ ] Stati di caricamento, errore e lista vuota tutti gestiti
- [ ] Errori con un messaggio utile e, dove ha senso, un "Riprova"
- [ ] `alt` sulle immagini, label collegate agli input
- [ ] Elementi cliccabili che sono `button` o `a`, non `div`
- [ ] Niente `console.log` dimenticati

**5. Performance (solo se misurata)**
- [ ] Nessuna memoizzazione applicata "per sicurezza" senza un problema osservato
- [ ] Liste molto lunghe paginate o virtualizzate
- [ ] Rotte pesanti in `lazy` + `Suspense`
- [ ] Immagini dimensionate e in formati moderni

Un commento utile dice **cosa** non va, **perché** è un problema e **cosa fare**. E se il codice è
di qualcuno che sta imparando, dì anche cosa è fatto bene: una review che elenca solo difetti non
viene applicata, viene subita.
