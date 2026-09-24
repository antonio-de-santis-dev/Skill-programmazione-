# Redux Toolkit

Indice: [I tre principi](#i-tre-principi) · [Setup](#setup-dello-store) ·
[createSlice e Immer](#createslice-e-immer) · [useSelector e useDispatch](#useselector-e-usedispatch) ·
[Selettori](#selettori-e-createselector) · [Asincrono](#operazioni-asincrone) ·
[extraReducers](#extrareducers) · [addMatcher](#addmatcher) ·
[Struttura](#struttura-delle-cartelle) · [DevTools](#devtools) · [TypeScript](#typescript)

```bash
npm install @reduxjs/toolkit react-redux
```

Redux Toolkit è il modo ufficiale di usare Redux dal 2019. Il Redux "classico" con
`createStore`, le costanti delle action type, gli action creator scritti a mano e gli spread
annidati nei reducer è **legacy**: serve solo per leggere codice esistente. Se un tutorial mostra
`combineReducers` e `switch` scritti a mano, è precedente a Toolkit.

---

## I tre principi

1. **Single source of truth** — un unico oggetto contiene lo stato condiviso dell'applicazione.
2. **Lo stato è in sola lettura** — cambia solo dispatchando azioni.
3. **I cambiamenti avvengono con funzioni pure** — i reducer.

Redux non è legato a React: `react-redux` è la libreria che fa da ponte fra i due.

---

## Setup dello store

```js
// src/redux/store.js
import { configureStore } from '@reduxjs/toolkit';
import cartReducer from './slices/cartSlice';
import filtersReducer from './slices/filtersSlice';

export const store = configureStore({
  reducer: {
    cart: cartReducer,
    filters: filtersReducer,
  },
});
```

```jsx
// main.jsx
import { Provider } from 'react-redux';
import { store } from './redux/store';

<Provider store={store}>
  <App />
</Provider>
```

`configureStore` fa gratis tre cose che nel Redux classico andavano configurate a mano: combina i
reducer, collega i DevTools, e installa i middleware standard — fra cui i controlli che avvisano in
sviluppo se modifichi lo stato per sbaglio o ci metti dentro valori non serializzabili.

È lo stesso pattern di `ThemeProvider` e di qualsiasi Provider di Context: un componente che
avvolge l'albero e rende un valore disponibile ovunque.

---

## createSlice e Immer

Una **slice** è una fetta dello stato con i suoi reducer e le sue azioni, tutto in un file.

```js
import { createSlice } from '@reduxjs/toolkit';

const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [], total: 0, amount: 0 },
  reducers: {
    itemAggiunto: (state, action) => {
      const esistente = state.items.find((i) => i.id === action.payload.id);
      if (esistente) {
        esistente.quantity += 1;              // sembra una mutazione
      } else {
        state.items.push({ ...action.payload, quantity: 1 });
      }
    },
    carrelloSvuotato: (state) => {
      state.items = [];
    },
  },
});

export const { itemAggiunto, carrelloSvuotato } = cartSlice.actions;
export default cartSlice.reducer;
```

**Perché la mutazione è lecita qui.** Redux Toolkit include **Immer**: ti passa una *bozza* dello
stato, tu la modifichi come un oggetto normale, e Immer produce un nuovo oggetto immutabile
applicando solo le differenze. Ottieni la leggibilità della mutazione con la sicurezza
dell'immutabilità. Confronta con il reducer scritto a mano, dove servivano `map` con ternario e
spread annidati per ottenere lo stesso risultato.

L'unica regola da ricordare: **o modifichi la bozza, o ritorni un nuovo stato. Mai entrambe nella
stessa funzione.**

```js
increment: (state) => { state.value += 1; }      // ok
reset:     () => ({ value: 0 })                  // ok
strano:    (state) => { state.value += 1; return { value: 0 }; }   // rotto
```

Attenzione: Immer funziona **solo dentro** i reducer di Redux Toolkit. Lo stesso codice in un
`useState` o in un `useReducer` di React muta davvero e non ridisegna nulla.

**Le azioni sono generate automaticamente** dai nomi dei reducer: `itemAggiunto()` produce
`{ type: 'cart/itemAggiunto' }`. Il formato `nomeSlice/nomeReducer` non è cosmetico — è ciò su cui
si basa `addMatcher` più avanti.

**Sui nomi delle azioni:** descrivono un *evento accaduto* (`itemAggiunto`, `checkoutConfermato`,
`filtriAzzerati`), non un'assegnazione (`setItems`, `setData`). Una slice piena di `setQualcosa` è
il segnale che lo store è usato come un contenitore di variabili invece che come un registro di
ciò che succede nell'applicazione — e i DevTools diventano illeggibili.

---

## useSelector e useDispatch

```jsx
import { useSelector, useDispatch } from 'react-redux';
import { itemAggiunto } from '../redux/slices/cartSlice';

const Prodotto = ({ prodotto }) => {
  const total = useSelector((store) => store.cart.total);
  const dispatch = useDispatch();

  return <button onClick={() => dispatch(itemAggiunto(prodotto))}>Aggiungi ({total} €)</button>;
};
```

`useSelector` riceve l'intero store e tu ritorni la parte che ti interessa. Per esplorare la
struttura in fase di sviluppo, `useSelector((s) => s)` stampa tutto.

**La regola d'oro delle performance: seleziona il valore più specifico possibile.**

```jsx
const cart = useSelector((s) => s.cart);          // si ridisegna a OGNI cambiamento del carrello
const total = useSelector((s) => s.cart.total);   // solo quando cambia il totale
```

È il vantaggio concreto rispetto al Context, dove ogni cambiamento del Provider ridisegna tutti i
consumatori.

**Non costruire oggetti nel selettore:**

```jsx
// render infiniti: l'oggetto è nuovo a ogni chiamata, quindi sempre "diverso"
const { total, items } = useSelector((s) => ({ total: s.cart.total, items: s.cart.items }));

// corretto: due selettori separati
const total = useSelector((s) => s.cart.total);
const items = useSelector((s) => s.cart.items);
```

`dispatch` è **stabile** fra i render: metterlo nelle dipendenze di un effetto è corretto e non
causa rigiri, e avvolgerlo in `useCallback` non serve.

---

## Selettori e createSelector

Un selettore definito accanto alla slice evita che la struttura dello store si diffonda in cento
componenti:

```js
export const selectCartItems = (state) => state.cart.items;
export const selectCartTotal = (state) => state.cart.total;
```

```jsx
const items = useSelector(selectCartItems);
```

Se domani `items` si sposta, cambi una riga invece di cercarla ovunque.

Per i valori **derivati** costosi, `createSelector` memorizza il risultato:

```js
import { createSelector } from '@reduxjs/toolkit';

export const selectItemsFiltrati = createSelector(
  [selectCartItems, (state) => state.filters.categoria],
  (items, categoria) =>
    categoria === 'tutti' ? items : items.filter((i) => i.categoria === categoria)
);
```

Senza memoizzazione, un selettore che ritorna `items.filter(...)` produce un array nuovo a ogni
chiamata e fa ridisegnare il componente sempre. È il motivo principale per cui esiste
`createSelector`, più ancora del costo del calcolo.

Il valore derivato **non va salvato nello store**: si calcola. Salvarlo significa doverlo
aggiornare a mano ogni volta che cambia la base, cioè creare la seconda fonte di verità che tutti i
bug di stato hanno in comune.

---

## Operazioni asincrone

I reducer sono funzioni pure: **niente chiamate API al loro interno**. L'asincrono sta fuori e
dispatcha i risultati.

### La forma esplicita

Utile per capire cosa succede, e perfettamente legittima:

```js
export const fetchPhotos = (page) => async (dispatch) => {
  dispatch(startLoading());
  try {
    const { data } = await client.get('/photos', { params: { page, per_page: 12 } });
    dispatch(saveData(data));
  } catch (error) {
    dispatch(catchError(messaggioErrore(error)));
  } finally {
    dispatch(stopLoading());
  }
};

// nel componente
dispatch(fetchPhotos(page));
```

### createAsyncThunk

Automatizza lo stesso schema generando tre azioni: `pending`, `fulfilled`, `rejected`.

```js
export const fetchPhotos = createAsyncThunk(
  'api/fetchPhotos',
  async (page, thunkAPI) => {
    try {
      const { data } = await client.get('/photos', { params: { page, per_page: 12 } });
      return data;
    } catch (error) {
      return thunkAPI.rejectWithValue(
        error.response?.status === 403 ? "Limite orario raggiunto" : "Errore nel caricamento"
      );
    }
  }
);
```

```js
extraReducers: (builder) => {
  builder
    .addCase(fetchPhotos.pending,   (state) => { state.loading = true; })
    .addCase(fetchPhotos.fulfilled, (state, action) => {
      state.loading = false;
      state.photos = action.payload;
    })
    .addCase(fetchPhotos.rejected,  (state, action) => {
      state.loading = false;
      state.error = { status: true, message: action.payload };
    });
}
```

`rejectWithValue` serve a passare un messaggio controllato invece dell'oggetto errore grezzo, che
fra l'altro non è serializzabile e fa protestare i middleware di Redux.

**Nota importante:** se le chiamate API sono molte e servono cache, deduplica e invalidazione,
`createAsyncThunk` non è la risposta giusta. Lo sono **RTK Query** (incluso in Redux Toolkit) o
TanStack Query. Vedi `scelta-dello-stato.md`.

---

## extraReducers

Permette a una slice di rispondere ad **azioni definite altrove**: in un'altra slice, o create a
mano con `createAction`.

```js
import { createSlice, createAction } from '@reduxjs/toolkit';

export const utenteDisconnesso = createAction('auth/utenteDisconnesso');

const cartSlice = createSlice({
  name: 'cart',
  initialState,
  reducers: { /* ... */ },
  extraReducers: (builder) => {
    builder.addCase(utenteDisconnesso, () => initialState);   // il logout svuota il carrello
  },
});
```

Le azioni generate da `reducers` appartengono a quella slice sola; `extraReducers` è il modo per
ascoltare quelle esterne. I due casi tipici: intercettare le azioni di `createAsyncThunk`, e far
reagire due slice diverse allo stesso evento senza dispatchare due azioni.

Il `builder` è un oggetto su cui si concatenano i casi — come uno `switch`, ma componibile e
tipizzabile.

---

## addMatcher

Intercetta azioni in base a una **condizione**, non a un nome esatto.

Il problema: il totale del carrello va ricalcolato sia quando si aggiunge, sia quando si rimuove,
sia quando si cambia quantità. Dispatchare un'azione `calcolaTotali` dopo ognuna è ridondante e
prima o poi qualcuno se ne dimentica.

```js
const isCartAction = (action) => action.type.startsWith('cart/');

extraReducers: (builder) => {
  builder
    .addCase(itemRimosso, (state, action) => {
      state.items = state.items.filter((i) => i.id !== action.payload);
    })
    .addMatcher(isCartAction, (state) => {
      const { total, amount } = state.items.reduce(
        (acc, item) => {
          acc.total += item.price * item.quantity;
          acc.amount += item.quantity;
          return acc;
        },
        { total: 0, amount: 0 }
      );
      state.total = parseFloat(total.toFixed(2));
      state.amount = amount;
    });
}
```

Ecco a cosa serviva il formato `nomeSlice/nomeReducer`: `startsWith('cart/')` cattura tutte le
azioni del carrello in una riga. Un solo dispatch dell'utente aggiorna sia la lista sia i totali.

**L'ordine conta:** prima vengono valutati i `addCase`, poi i `addMatcher` nell'ordine di
dichiarazione. Il matcher gira dopo che il case ha già modificato lo stato, quindi calcola sul dato
aggiornato.

Più matcher insieme, per gestire loading ed errore di dieci thunk diversi in un punto solo:

```js
const isPending   = (a) => a.type.endsWith('/pending');
const isRejected  = (a) => a.type.endsWith('/rejected');
const isFulfilled = (a) => a.type.endsWith('/fulfilled');

builder
  .addMatcher(isPending,   (state) => { state.loading = true; state.error = null; })
  .addMatcher(isFulfilled, (state) => { state.loading = false; })
  .addMatcher(isRejected,  (state, action) => { state.loading = false; state.error = action.payload; })
  .addDefaultCase((state) => state);
```

Con dieci thunk questo sostituisce trenta `addCase`. Usalo quando il comportamento è davvero
uniforme: se un thunk ha bisogno di un trattamento diverso, un `addCase` esplicito è più chiaro di
un matcher con un'eccezione dentro.

---

## Struttura delle cartelle

```
src/redux/
├── store.js
└── slices/
    ├── cartSlice.js        stato + reducer + azioni + selettori del carrello
    ├── filtersSlice.js
    └── authSlice.js
```

Le slice si organizzano **per dominio** (carrello, filtri, autenticazione), non per tipo tecnico
(una cartella `actions/`, una `reducers/`, una `selectors/`). La divisione per tipo è
l'organizzazione del Redux classico, e obbliga a toccare tre cartelle per ogni modifica.

Ogni file di slice esporta: il reducer come default, le azioni come export nominali, i selettori
come export nominali. Tutto ciò che riguarda un dominio sta in un punto solo.

---

## DevTools

Installa l'estensione Redux DevTools nel browser: `configureStore` la collega da sola.

Cosa ci guardi, in ordine di utilità:

- **la lista delle azioni** mentre usi l'applicazione: se un'azione non compare, non è stata
  dispatchata — il problema è nel componente, non nel reducer;
- **diff**: cosa è cambiato nello stato dopo quell'azione. Se l'azione c'è ma il diff è vuoto, il
  reducer non sta facendo quello che credi;
- **time travel**: torni indietro alle azioni precedenti e vedi l'interfaccia ricostruirsi.

È lo strumento che rende il debug di Redux diverso da quello di un Context: prima di formulare
ipotesi, guarda cosa è successo davvero.

---

## TypeScript

Se il progetto usa TypeScript, i tipi dello store vanno derivati, non scritti a mano:

```ts
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

Da lì in avanti si usano `useAppSelector` e `useAppDispatch` al posto degli originali: lo store è
tipizzato ovunque senza annotazioni ripetute.
