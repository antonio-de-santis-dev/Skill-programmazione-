# React Hooks

Indice: [Le due regole](#le-due-regole-degli-hook) · [useState](#usestate) ·
[useEffect](#useeffect) · [useRef](#useref) · [useReducer](#usereducer) ·
[useContext](#usecontext) · [Custom hook](#custom-hook) · [useId](#useid) ·
[Quale hook scegliere](#quale-hook-scegliere)

Gli hook sono funzioni che agganciano un componente-funzione alle funzionalità interne di React:
memoria che sopravvive ai render, effetti collaterali, riferimenti al DOM, dati condivisi. Iniziano
tutti con `use`, e non è decorazione: è la convenzione su cui si basano il linter e le regole qui
sotto.

---

## Le due regole degli hook

**1. Gli hook si chiamano solo al livello più alto del componente.** Mai dentro `if`, cicli,
funzioni annidate o dopo un return condizionale.

```jsx
// sbagliato
if (loggato) { const [nome, setNome] = useState(""); }

// corretto
const [nome, setNome] = useState("");
if (loggato) { /* usa nome qui */ }
```

Perché: React non conosce i nomi delle tue variabili di stato, le identifica **per ordine di
chiamata**. Al primo render registra "primo useState → questo valore, secondo useState → quello".
Se un `if` salta una chiamata, l'ordine slitta e React assegna il valore sbagliato alla variabile
sbagliata. L'errore che vedrai è `Rendered fewer hooks than expected`.

**2. Gli hook si chiamano solo dentro componenti React o dentro altri custom hook.** Non in
funzioni JavaScript qualsiasi. È anche il motivo per cui i custom hook devono chiamarsi
`useQualcosa`: il nome è il segnale che permette al linter di applicare queste regole.

Corollario pratico: la condizione va *dentro* l'hook, non attorno.

```jsx
useEffect(() => {
  if (valore > 1) { /* ... */ }
}, [valore]);
```

---

## useState

```jsx
const [titolo, setTitolo] = useState("Hello World");
//      valore    setter        valore iniziale (usato solo al primo render)
```

Perché serve, e vale la pena mostrarlo quando si insegna: modificare una variabile normale non
aggiorna lo schermo. Due problemi distinti insieme — React non sa che qualcosa è cambiato, e
comunque a ogni render la funzione verrebbe rieseguita da capo reinizializzando la variabile.
`useState` risolve entrambi: conserva il valore fra i render e avvisa React quando cambia.

**Mai modificare lo stato direttamente.** `titolo = "Nuovo"` non fa nulla di visibile; solo il
setter provoca un nuovo render.

### Immutabilità

React decide se ridisegnare confrontando il **riferimento** del vecchio valore con quello del nuovo
(`Object.is`, confronto superficiale). Modificare un array o un oggetto in posto lascia il
riferimento identico: React conclude che nulla è cambiato.

```jsx
lista.push(nuovo);                          // non ridisegna
setLista([...lista, nuovo]);                // corretto

setPersona({ nome: "Marco" });              // cancella gli altri campi
setPersona({ ...persona, nome: "Marco" });  // corretto
```

Il setter di `useState` **sostituisce**, non fonde (a differenza del vecchio `setState` delle
classi). Lo spread ricopia il resto; l'ordine conta, la proprietà che vuoi cambiare va **dopo** lo
spread.

Operazioni che ritornano sempre una struttura nuova, e quindi sono sicure: `map`, `filter`,
`slice`, `concat`, spread. Operazioni che mutano, e quindi no: `push`, `pop`, `splice`, `sort`,
`reverse` (queste ultime due su una copia: `[...lista].sort()`).

### Aggiornamento funzionale

```jsx
setValore(valore + 1);
setValore(valore + 1);   // risultato: +1, non +2

setValore((prev) => prev + 1);
setValore((prev) => prev + 1);   // risultato: +2
```

Durante l'esecuzione di un handler, `valore` è una costante congelata al render corrente; gli
aggiornamenti sono raggruppati (batching). La forma funzionale riceve il valore aggiornato dalla
chiamata precedente.

Regola pratica: **usa la forma funzionale ogni volta che il nuovo stato dipende dal precedente** —
contatori, toggle (`setAperto(prev => !prev)`), aggiunta a una lista. Per un valore assoluto
(`setNome("Marco")`) la forma semplice va benissimo.

Caso in cui è l'unica che funziona: dentro `setInterval` o `setTimeout`, dove la forma semplice
legge per sempre il valore congelato al momento in cui il timer è stato creato.

### Inizializzazione pigra

```jsx
const [tema, setTema] = useState(() => localStorage.getItem('theme') ?? 'light');
```

Si passa la **funzione**, non il suo risultato: React la esegue solo al primo render. Con le
parentesi (`useState(leggiTema())`) la funzione verrebbe chiamata a ogni render e il risultato
buttato via — inutile, e costoso se legge dal disco o calcola.

### Uno stato o molti

Più `useState` separati quando i valori sono indipendenti; un oggetto quando cambiano insieme;
`useReducer` quando le azioni che li modificano sono molte. Non c'è una risposta unica: la domanda
giusta è "questi valori cambiano sempre nello stesso momento?".

---

## useEffect

`useEffect` gestisce i **side effect**: tutto ciò che tocca il mondo fuori da React — titolo del
documento, event listener, timer, localStorage, chiamate API, sottoscrizioni.

La regola sotto: un componente deve limitarsi a calcolare cosa mostrare. Tutto il resto va isolato
in un effetto, che gira **dopo** che React ha aggiornato il DOM.

### Il secondo parametro

```jsx
useEffect(() => { /* ... */ });            // dopo OGNI render
useEffect(() => { /* ... */ }, []);        // una sola volta, al montaggio
useEffect(() => { /* ... */ }, [valore]);  // al montaggio e quando valore cambia
```

Senza array, un effetto che fa una fetch e salva con `setState` crea un **loop infinito**: effetto
→ stato → render → effetto. È l'errore che quasi tutti fanno la prima volta.

Attenzione a oggetti e array nelle dipendenze: a ogni render sono un riferimento nuovo, quindi
risultano sempre "cambiati" e l'effetto gira sempre. Metti valori primitivi (`utente.id`, non
`utente`), oppure memorizza con `useMemo`/`useCallback` (vedi `performance.md`).

Non zittire il warning `react-hooks/exhaustive-deps` togliendo dipendenze: leggere un valore non
dichiarato significa leggere per sempre quello vecchio (stale closure). Se l'effetto gira troppo
spesso, la soluzione è stabilizzare la dipendenza, non nasconderla.

### Cleanup

```jsx
useEffect(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id);     // gira prima del prossimo effetto e allo smontaggio
}, []);
```

Senza cleanup, il timer continua a girare quando il componente sparisce; montando e smontando
dieci volte ti ritrovi dieci timer. Stesso discorso per `addEventListener` →
`removeEventListener`, per le sottoscrizioni e per le richieste in volo (`AbortController`, vedi
`data-fetching.md`).

La sequenza è sempre: cleanup del vecchio effetto → nuovo effetto.

### La callback non può essere async

```jsx
// sbagliato: async ritorna una Promise, React si aspetta undefined o la cleanup
useEffect(async () => { ... }, []);

// corretto
useEffect(() => {
  const carica = async () => { /* ... */ };
  carica();
}, []);
```

### Quando NON serve useEffect

Metà degli useEffect che si vedono nel codice reale non dovrebbero esistere. Non serve un effetto:

- per **calcolare un valore derivato** dallo stato: calcolalo durante il render
  (`const totale = items.reduce(...)`), non con uno stato aggiuntivo aggiornato da un effetto;
- per **reagire a un evento dell'utente**: la logica va nell'handler del click, non in un effetto
  che osserva lo stato cambiato dal click;
- per **resettare lo stato quando cambia una prop**: spesso basta una `key` diversa sul componente.

Un effetto serve per sincronizzarsi con qualcosa **fuori** da React. Se non c'è niente di esterno,
probabilmente non serve.

---

## useRef

```jsx
const riferimento = useRef(valoreIniziale);   // si legge e scrive con .current
```

Un contenitore mutabile che sopravvive ai render e **non** provoca un nuovo render quando cambia.

**Uso 1 — accedere a un nodo del DOM:**

```jsx
const inputRef = useRef(null);

useEffect(() => { inputRef.current.focus(); }, []);

<input ref={inputRef} />
```

Il valore iniziale è `null` perché durante il primo render l'elemento non esiste ancora: viene
popolato subito dopo. Per questo l'accesso va fatto dentro un effetto o in un handler, **mai
durante il render**. Da lì hai il nodo vero, con `.focus()`, `.value`,
`.getBoundingClientRect()`, `.scrollIntoView()`.

**Uso 2 — memorizzare un valore che non si vede a schermo:** id di timer, valore del render
precedente, flag "è il primo render".

```jsx
const primoRender = useRef(true);
useEffect(() => {
  if (primoRender.current) { primoRender.current = false; return; }
  // qui solo dai render successivi
}, [valore]);
```

| | useState | useRef |
|---|---|---|
| Sopravvive ai render | sì | sì |
| Causa un nuovo render | sì | no |
| Lettura | `valore` | `ref.current` |
| Modifica | tramite setter | diretta |

La regola: **se cambiandolo lo schermo deve aggiornarsi, è stato; altrimenti è un ref.**

---

## useReducer

Centralizza in una sola funzione pura tutta la logica di aggiornamento di uno stato complesso.

```jsx
const initialState = { persone: [], modalAperto: false, messaggio: '' };

const reducer = (state, action) => {
  switch (action.type) {
    case 'AGGIUNGI':
      return { ...state, persone: [...state.persone, action.payload],
               modalAperto: true, messaggio: 'Persona aggiunta' };
    case 'RIMUOVI':
      return { ...state, persone: state.persone.filter((p) => p.id !== action.payload) };
    case 'CHIUDI_MODAL':
      return { ...state, modalAperto: false };
    default:
      throw new Error(`Azione non riconosciuta: ${action.type}`);
  }
};

const [state, dispatch] = useReducer(reducer, initialState);
dispatch({ type: 'RIMUOVI', payload: id });
```

I tre concetti: **state** (l'oggetto completo), **action** (un oggetto che descrive cosa è
successo, con `type` e spesso `payload`), **reducer** (`(state, action) => nuovoState`).

Le regole del reducer:

- ritorna **sempre** uno stato, mai `undefined`. Il `throw` nel default intercetta i typo nei nomi
  delle azioni, che altrimenti fallirebbero in silenzio;
- è una **funzione pura**: niente chiamate API, niente `Math.random()`, niente modifiche a
  variabili esterne. Solo calcolo. Le chiamate API stanno fuori, e dispatchano i risultati;
- ritorna un oggetto **nuovo**, non modifica `state`. Da qui lo `...state` all'inizio di ogni
  ritorno;
- **sta fuori dal componente**, perché non dipende da esso — ed è quindi testabile da solo,
  chiamandolo con uno stato e un'azione e verificando il risultato.

Quando preferirlo a `useState`: più campi correlati, molte azioni diverse sullo stesso stato, il
nuovo stato dipende fortemente dal precedente, o vuoi testare la logica separatamente.

È anche il ponte concettuale verso Redux: chi capisce reducer, action e dispatch qui, in Redux
Toolkit si ritrova in casa (vedi la skill **react-state-redux**).

---

## useContext

Rende un valore disponibile a tutto un sottoalbero senza passarlo di livello in livello.

Il problema che risolve è il **prop drilling**: passare props attraverso componenti intermedi che
non le usano, solo per farle arrivare in fondo. Con uno o due livelli è la soluzione più semplice e
leggibile — non "correggerlo" per principio. Diventa un peso quando la catena si allunga o quando
il dato serve a rami diversi dell'albero.

```jsx
// context.jsx
const AppContext = createContext();

export const AppProvider = ({ children }) => {
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);
  const openSidebar = () => setIsSidebarOpen(true);
  const closeSidebar = () => setIsSidebarOpen(false);

  return (
    <AppContext.Provider value={{ isSidebarOpen, openSidebar, closeSidebar }}>
      {children}
    </AppContext.Provider>
  );
};

export const useGlobalContext = () => {
  const context = useContext(AppContext);
  if (context === undefined) {
    throw new Error("useGlobalContext va usato dentro AppProvider");
  }
  return context;
};
```

```jsx
// nel componente, a qualsiasi profondità
const { openSidebar } = useGlobalContext();
```

Il custom hook che avvolge `useContext` è il pattern da usare sempre: evita di importare il context
in ogni file e dà un errore chiaro invece di un `undefined` misterioso quando qualcuno usa il
context fuori dal Provider.

**Avvertenza sulle performance:** quando il valore del Provider cambia, *tutti* i consumatori si
ridisegnano. Per questo non si mette tutto in un unico context gigante: meglio context separati per
dominio (tema, utente, carrello). Se il valore è un oggetto letterale ricreato a ogni render,
avvolgilo in `useMemo`.

Il pattern completo per stato globale complesso è **useContext + useReducer**: il reducer gestisce
la logica, il context la distribuisce. È una versione artigianale di Redux, e va benissimo finché
l'applicazione è di dimensioni medie.

---

## Custom hook

Una normale funzione che inizia con `use` e usa altri hook al suo interno. Serve a riusare
**logica**, non interfaccia (per l'interfaccia ci sono i componenti).

```jsx
// hooks/useFetch.js
const useFetch = (url) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    const fetchData = async () => {
      setLoading(true);
      try {
        const res = await fetch(url, { signal: controller.signal });
        if (!res.ok) throw new Error(`Errore ${res.status}`);
        setData(await res.json());
        setError(null);
      } catch (err) {
        if (err.name !== 'AbortError') setError(err.message);
      } finally {
        setLoading(false);
      }
    };
    fetchData();
    return () => controller.abort();
  }, [url]);

  return { data, loading, error };
};
```

Le regole: il nome inizia con `use` (è ciò che abilita le regole degli hook su quella funzione);
può chiamare altri hook; ritorna quello che vuoi (oggetto, array, singolo valore); **ogni chiamata
è indipendente** — due componenti che usano `useFetch` hanno stati separati, non condividono nulla.
Condividere è compito del Context o dello store.

Il segnale che serve un custom hook: due componenti con le stesse tre righe di `useState` e lo
stesso `useEffect`. Altri classici in `patterns/hooks-recipes.md`.

---

## useId

```jsx
const id = useId();
<label htmlFor={id}>Email</label>
<input id={id} type="email" />
```

Genera un identificativo univoco e **stabile fra server e client**. Serve esclusivamente a
collegare attributi di accessibilità (`htmlFor`, `aria-describedby`). Non usarlo per le `key` delle
liste, che devono derivare dai dati. Non usare `Math.random()` al suo posto: cambierebbe a ogni
render e romperebbe l'hydration nel rendering lato server.

Per più campi nello stesso componente si usa un prefisso: `htmlFor={`${id}-nome`}`.

---

## Quale hook scegliere

| Situazione | Hook |
|---|---|
| Un valore che si vede a schermo e cambia | `useState` |
| Più valori correlati, molte azioni diverse | `useReducer` |
| Un valore che non si vede, o un nodo del DOM | `useRef` |
| Sincronizzarsi con qualcosa fuori da React | `useEffect` |
| Un dato che serve a molti componenti lontani | `useContext` (+ `useReducer` se complesso) |
| La stessa logica ripetuta in più componenti | un custom hook |
| Collegare label e input | `useId` |
| Evitare render o calcoli inutili, **dopo aver misurato** | vedi `performance.md` |
