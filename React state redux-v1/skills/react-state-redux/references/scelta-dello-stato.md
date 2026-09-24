# Dove mettere lo stato

Indice: [La scala](#la-scala-in-sette-gradini) · [Locale](#1-stato-locale) ·
[Lifting](#2-lifting-state-up) · [URL](#3-stato-nellurl) · [Context](#4-context) ·
[Context + reducer](#5-context--usereducer) · [Redux Toolkit](#6-redux-toolkit) ·
[Server state](#7-server-state-tanstack-query) · [Zustand](#zustand-la-via-di-mezzo) ·
[Tabella di decisione](#tabella-di-decisione)

---

## La scala in sette gradini

Si sale **solo quando il gradino precedente non basta più**. Saltare al sesto gradino perché
"un'applicazione seria usa Redux" è il modo più comune di rendere complicato un progetto semplice.

1. Stato locale (`useState`, `useReducer`)
2. Lifting state up
3. Stato nell'URL
4. Context
5. Context + useReducer
6. Redux Toolkit (o Zustand)
7. Libreria di server state, in parallelo a tutto il resto

---

## 1. Stato locale

Il default. Un input, un accordion aperto, un modal usato in un punto solo, il flag "immagine
caricata" di una singola foto.

```jsx
const [aperto, setAperto] = useState(false);
```

Vale la pena dirlo, perché viene sottovalutato: la maggior parte dello stato di un'applicazione è
locale e deve restare tale. Portarlo più in alto lo rende visibile a chi non deve vederlo e fa
ridisegnare componenti che non c'entrano.

## 2. Lifting state up

Due componenti fratelli hanno bisogno dello stesso dato: lo stato sale al primo antenato comune e
scende come props.

```jsx
const [filtro, setFiltro] = useState('tutti');
<Filtri valore={filtro} onCambia={setFiltro} />
<Lista filtro={filtro} />
```

È la soluzione giusta fino a due livelli di distanza. Le props sono esplicite: leggendo il
componente si sa da dove arrivano i dati, cosa che nessuna soluzione globale ti dà.

## 3. Stato nell'URL

Spesso dimenticato, e spesso la risposta migliore. Filtri, ricerca, pagina corrente, tab attiva,
id dell'elemento aperto: tutto ciò che l'utente si aspetta di poter **condividere con un link** o
ritrovare dopo un refresh.

```jsx
const [searchParams, setSearchParams] = useSearchParams();
const page = Number(searchParams.get('page') ?? 1);
```

Vantaggi che nessuno store può dare: il tasto indietro funziona, l'URL è condivisibile, lo stato
sopravvive al refresh senza scrivere una riga di persistenza.

## 4. Context

Un dato serve a molti componenti sparsi nell'albero, e passarlo come prop significherebbe
attraversare componenti che non lo usano (**prop drilling**).

Casi tipici: tema chiaro/scuro, utente autenticato, lingua, apertura della sidebar.

```jsx
const AppContext = createContext();

export const AppProvider = ({ children }) => {
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);
  const openSidebar = () => setIsSidebarOpen(true);
  const closeSidebar = () => setIsSidebarOpen(false);

  const value = useMemo(
    () => ({ isSidebarOpen, openSidebar, closeSidebar }),
    [isSidebarOpen]
  );

  return <AppContext.Provider value={value}>{children}</AppContext.Provider>;
};

export const useGlobalContext = () => {
  const context = useContext(AppContext);
  if (context === undefined) throw new Error("useGlobalContext va usato dentro AppProvider");
  return context;
};
```

Tre accortezze che fanno la differenza fra un Context utile e uno problematico:

- **Il custom hook che avvolge `useContext`**: evita di importare il context ovunque e dà un errore
  chiaro invece di un `undefined` misterioso.
- **`useMemo` sul valore**: senza, l'oggetto è nuovo a ogni render del Provider e tutti i
  consumatori si ridisegnano anche quando nulla è cambiato.
- **Context separati per dominio** (tema, utente, carrello) invece di uno solo con tutto dentro:
  quando cambia il valore di un Provider, *tutti* i suoi consumatori si ridisegnano, e un context
  unico significa ridisegnare l'applicazione a ogni cambiamento.

Il prop drilling **non è sempre un errore**: per uno o due livelli resta la soluzione più semplice
e leggibile. Il Context si introduce quando la catena si allunga o quando il dato serve a rami
diversi dell'albero.

## 5. Context + useReducer

Quando lo stato condiviso ha più campi correlati e molte azioni diverse. Il reducer gestisce la
logica, il Context la distribuisce.

```jsx
export const AppProvider = ({ children }) => {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <AppContext.Provider value={{
      ...state,
      increase: (id) => dispatch({ type: 'INCREASE', payload: id }),
      removeItem: (id) => dispatch({ type: 'REMOVE_ITEM', payload: id }),
    }}>
      {children}
    </AppContext.Provider>
  );
};
```

I componenti chiamano `increase(id)` e non sanno nulla di reducer e action type: la logica resta
incapsulata. Esempio completo in `../../react-development/patterns/component-recipes.md`.

È **una versione artigianale di Redux**, e chi l'ha costruita capisce Redux Toolkit in un
pomeriggio: stesso stato centrale, stesse azioni, stesso dispatch, stessi reducer puri.

I limiti che prima o poi si incontrano: nessun DevTools, la gestione dell'asincrono va scritta a
mano, e la granularità dei render è grossolana (tutti i consumatori si ridisegnano insieme).

## 6. Redux Toolkit

Si sale qui quando almeno **due** di queste condizioni sono vere:

- lo stato condiviso è ampio e tocca molte parti dell'applicazione;
- cambia spesso e da punti diversi;
- serve poter **ispezionare** cosa è successo (cronologia delle azioni, time travel);
- ci sono molte operazioni asincrone con loading ed errore da gestire in modo uniforme;
- il team è numeroso e serve una struttura riconoscibile a chiunque entri nel progetto.

Cosa aggiunge rispetto a Context + reducer:

| | Context + reducer | Redux Toolkit |
|---|---|---|
| Debug | nessuno strumento | DevTools con cronologia e time travel |
| Asincrono | a mano | `createAsyncThunk`, middleware |
| Render | tutti i consumatori insieme | `useSelector` ridisegna solo chi usa quel valore |
| Struttura su progetti grandi | si sfilaccia | slice separate per dominio |
| Costo | zero dipendenze | due pacchetti, un po' di struttura |

Tutto il dettaglio in `redux-toolkit.md`.

## 7. Server state (TanStack Query)

Questo gradino è **ortogonale** agli altri: non sostituisce Redux, risolve un problema diverso.

I dati che arrivano da un'API non sono stato dell'applicazione: sono una **copia locale di
qualcosa che vive altrove** e che può cambiare senza che tu lo sappia. Trattarli come stato
significa doversi occupare a mano di cache, invalidazione, deduplica, retry, aggiornamento in
background — cioè riscrivere una libreria.

```jsx
const { data, isLoading, error } = useQuery({
  queryKey: ['photos', page, query],
  queryFn: () => client.get('/photos', { params: { page, query } }).then((r) => r.data),
});
```

La divisione di lavoro che funziona nei progetti reali:

- **server state** (elenchi, dettagli, tutto ciò che arriva da un'API) → TanStack Query;
- **client state** (carrello, filtri attivi, sessione, preferenze, apertura di pannelli) → Redux,
  Zustand o Context.

Se un'applicazione usa Redux *solo* per memorizzare risposte di API, molto probabilmente non le
serve Redux.

---

## Zustand, la via di mezzo

```js
import { create } from 'zustand';

export const useCartStore = create((set) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
  clear: () => set({ items: [] }),
}));
```

```jsx
const items = useCartStore((s) => s.items);      // stessa selezione granulare di useSelector
```

Nessun Provider, nessun boilerplate, selezione granulare come Redux. Va considerata quando serve
stato globale ma non l'apparato completo di Redux. Redux resta preferibile quando servono DevTools
avanzati, middleware, o quando il team lo conosce già.

Non proporre una migrazione da Redux a Zustand (o viceversa) su un progetto che funziona: il
guadagno è marginale e il costo reale.

---

## Tabella di decisione

| Il dato… | Dove va |
|---|---|
| serve a un solo componente | `useState` locale |
| serve a due componenti vicini | lifting, props |
| l'utente deve poterlo condividere con un link o ritrovarlo dopo un refresh | URL (`useSearchParams`) |
| è globale e cambia di rado (tema, utente, lingua) | Context |
| è globale, complesso, con molte azioni | Context + `useReducer` |
| è globale, ampio, cambia spesso, e serve ispezionarlo | Redux Toolkit |
| arriva da un'API | TanStack Query (o `useFetch` su progetti piccoli) |
| è calcolabile da un altro dato | **da nessuna parte**: calcolalo |
