# Ricette Redux Toolkit

Codice funzionante da **adattare**: nomi, forma dei dati e struttura delle cartelle vanno allineati
al progetto reale.

Indice: [Setup minimo](#setup-minimo) · [Slice carrello](#slice-carrello-con-totali-automatici) ·
[Slice API](#slice-api-con-loading-ed-errore) · [Ricerca e paginazione](#ricerca-e-paginazione) ·
[Reset al logout](#reset-globale-al-logout) · [Persistenza](#persistenza-su-localstorage) ·
[RTK Query](#rtk-query-quando-le-chiamate-sono-molte)

---

## Setup minimo

```js
// src/redux/store.js
import { configureStore } from '@reduxjs/toolkit';
import cartReducer from './slices/cartSlice';

export const store = configureStore({
  reducer: { cart: cartReducer },
});
```

```jsx
// src/main.jsx
import { Provider } from 'react-redux';
import { store } from './redux/store';

createRoot(document.getElementById('root')).render(
  <Provider store={store}>
    <App />
  </Provider>
);
```

Se ci sono anche un ThemeProvider e un Router, l'ordine è: `Provider` (store) più esterno, poi il
tema, poi il router. Lo store non dipende da nulla, gli altri possono dipendere da lui.

---

## Slice carrello con totali automatici

Il caso completo: aggiunta, rimozione, quantità, e totali che si ricalcolano da soli.

```js
// src/redux/slices/cartSlice.js
import { createSlice, createAction } from '@reduxjs/toolkit';

const initialState = { items: [], total: 0, amount: 0 };

export const utenteDisconnesso = createAction('auth/utenteDisconnesso');

const cartSlice = createSlice({
  name: 'cart',
  initialState,
  reducers: {
    itemAggiunto: (state, action) => {
      const esistente = state.items.find((i) => i.id === action.payload.id);
      if (esistente) {
        esistente.quantity += 1;
      } else {
        state.items.push({ ...action.payload, quantity: 1 });
      }
    },
    itemRimosso: (state, action) => {
      state.items = state.items.filter((i) => i.id !== action.payload);
    },
    quantitaAumentata: (state, action) => {
      const item = state.items.find((i) => i.id === action.payload);
      if (item) item.quantity += 1;
    },
    quantitaDiminuita: (state, action) => {
      const item = state.items.find((i) => i.id === action.payload);
      if (!item) return;
      item.quantity -= 1;
      if (item.quantity === 0) {
        state.items = state.items.filter((i) => i.id !== action.payload);
      }
    },
    carrelloSvuotato: () => initialState,
  },
  extraReducers: (builder) => {
    builder
      .addCase(utenteDisconnesso, () => initialState)
      .addMatcher(
        (action) => action.type.startsWith('cart/'),
        (state) => {
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
        }
      );
  },
});

export const {
  itemAggiunto, itemRimosso, quantitaAumentata, quantitaDiminuita, carrelloSvuotato,
} = cartSlice.actions;

export const selectCartItems  = (state) => state.cart.items;
export const selectCartTotal  = (state) => state.cart.total;
export const selectCartAmount = (state) => state.cart.amount;

export default cartSlice.reducer;
```

```jsx
const CartItem = ({ id, title, price, quantity }) => {
  const dispatch = useDispatch();
  return (
    <article>
      <h4>{title}</h4>
      <p>{price} €</p>
      <button onClick={() => dispatch(quantitaAumentata(id))} aria-label="Aumenta">+</button>
      <span>{quantity}</span>
      <button onClick={() => dispatch(quantitaDiminuita(id))} aria-label="Diminuisci">−</button>
      <button onClick={() => dispatch(itemRimosso(id))}>Rimuovi</button>
    </article>
  );
};
```

Tre cose che meritano attenzione:

- Il **matcher** ricalcola i totali dopo ogni azione del carrello. Nessuno deve ricordarsi di
  dispatchare un `calcolaTotali`, ed è impossibile dimenticarlo aggiungendo una nuova azione.
- `parseFloat(total.toFixed(2))` perché `toFixed` ritorna una stringa: sui prezzi è un inciampo
  classico. Per un e-commerce vero, tieni i prezzi in **centesimi interi** ed evita del tutto
  l'aritmetica in virgola mobile.
- `carrelloSvuotato: () => initialState` ritorna un nuovo stato invece di modificare la bozza:
  entrambe le forme sono lecite, ma mai insieme nella stessa funzione.

---

## Slice API con loading ed errore

```js
// src/redux/slices/photosSlice.js
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import client from '../../api/client';

export const fetchPhotos = createAsyncThunk(
  'photos/fetch',
  async ({ page, query }, thunkAPI) => {
    try {
      const endpoint = query ? '/search/photos' : '/photos';
      const { data } = await client.get(endpoint, {
        params: { page, per_page: 12, ...(query && { query }) },
      });
      // l'endpoint di ricerca ritorna un oggetto, quello dei feed un array
      return query
        ? { photos: data.results, totalPages: data.total_pages }
        : { photos: data, totalPages: null };
    } catch (error) {
      const status = error.response?.status;
      return thunkAPI.rejectWithValue(
        status === 403 ? "Limite orario di richieste raggiunto. Riprova più tardi."
        : status === 401 ? "Chiave API non valida."
        : "Errore nel caricamento delle foto."
      );
    }
  }
);

const photosSlice = createSlice({
  name: 'photos',
  initialState: { items: [], totalPages: 0, loading: false, error: null },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchPhotos.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchPhotos.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload.photos;
        if (action.payload.totalPages !== null) state.totalPages = action.payload.totalPages;
      })
      .addCase(fetchPhotos.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload;
        state.items = [];
      });
  },
});

export default photosSlice.reducer;
```

```jsx
const Galleria = () => {
  const { items, loading, error } = useSelector((s) => s.photos);
  const { page, query } = useSelector((s) => s.filters);
  const dispatch = useDispatch();

  useEffect(() => {
    dispatch(fetchPhotos({ page, query }));
  }, [dispatch, page, query]);

  if (loading) return <SkeletonGrid count={12} />;
  if (error) return <ErrorMessage message={error} onRetry={() => dispatch(fetchPhotos({ page, query }))} />;
  if (items.length === 0) return <EmptyState query={query} />;

  return <PhotoGrid photos={items} />;
};
```

Quattro stati renderizzati: **caricamento, errore, vuoto, dati**. Lo stato vuoto è quello che viene
dimenticato e che produce una pagina bianca che sembra un bug.

Il messaggio di errore dice cosa è successo **e** cosa si può fare: un 403 da rate limit e un 401
da chiave sbagliata richiedono reazioni diverse dall'utente.

Se le slice con chiamate API diventano più di due o tre, guarda RTK Query più in basso.

---

## Ricerca e paginazione

```js
// src/redux/slices/filtersSlice.js
const filtersSlice = createSlice({
  name: 'filters',
  initialState: { query: '', page: 1, categoria: 'tutti' },
  reducers: {
    ricercaCambiata: (state, action) => {
      state.query = action.payload;
      state.page = 1;                 // una nuova ricerca riparte da pagina 1
    },
    paginaCambiata: (state, action) => { state.page = action.payload; },
    paginaSuccessiva: (state) => { state.page += 1; },
    paginaPrecedente: (state) => { state.page = Math.max(1, state.page - 1); },
    filtriAzzerati: () => ({ query: '', page: 1, categoria: 'tutti' }),
  },
});
```

```jsx
const Pagination = () => {
  const { page } = useSelector((s) => s.filters);
  const totalPages = useSelector((s) => s.photos.totalPages);
  const dispatch = useDispatch();

  return (
    <nav aria-label="Paginazione">
      <button onClick={() => dispatch(paginaPrecedente())} disabled={page === 1}>«</button>
      <span>Pagina {page} di {totalPages}</span>
      <button onClick={() => dispatch(paginaSuccessiva())} disabled={page >= totalPages}>»</button>
    </nav>
  );
};
```

**Il pattern architetturale:** i bottoni non chiamano l'API. Cambiano solo lo stato; un `useEffect`
osserva `page` e `query` e reagisce. Questo separa *cosa vuole l'utente* da *come si ottengono i
dati*, ed è ciò che rende manutenibile un'applicazione di medie dimensioni.

Se le pagine sono migliaia, elencarle tutte è impraticabile: mostra una finestra di cinque o sette
pagine attorno a quella corrente.

Alternativa da considerare prima di scrivere questa slice: tenere `query` e `page` nell'**URL**
(`useSearchParams`). Ottieni gratis la condivisibilità del link, il tasto indietro e la
sopravvivenza al refresh.

---

## Reset globale al logout

```js
// authSlice.js
export const utenteDisconnesso = createAction('auth/utenteDisconnesso');
```

Ogni slice che deve azzerarsi lo intercetta:

```js
extraReducers: (builder) => {
  builder.addCase(utenteDisconnesso, () => initialState);
}
```

Un solo dispatch pulisce carrello, filtri, preferenze e dati utente. L'alternativa — dispatchare
sei azioni di reset dal componente di logout — si rompe il giorno in cui qualcuno aggiunge la
settima slice e non aggiorna il logout.

---

## Persistenza su localStorage

Per pochi dati non serve una libreria:

```js
const caricaStato = () => {
  try {
    const salvato = localStorage.getItem('cart');
    return salvato ? JSON.parse(salvato) : undefined;
  } catch {
    return undefined;
  }
};

export const store = configureStore({
  reducer: { cart: cartReducer },
  preloadedState: caricaStato() ? { cart: caricaStato() } : undefined,
});

store.subscribe(() => {
  try {
    localStorage.setItem('cart', JSON.stringify(store.getState().cart));
  } catch { /* quota o modalità privata */ }
});
```

Tre avvertenze:

- persisti **solo ciò che serve davvero** (il carrello, le preferenze), non l'intero store: i dati
  delle API salvati diventano vecchi e producono interfacce incoerenti;
- i `try/catch` non sono paranoia: `localStorage` lancia in modalità privata e a quota esaurita, e
  un throw all'avvio fa sparire l'applicazione;
- mai token di autenticazione o dati sensibili: è leggibile da qualsiasi script sulla pagina.

Per casi più articolati esiste `redux-persist`, con whitelist, versioning e migrazioni.

---

## RTK Query, quando le chiamate sono molte

Incluso in Redux Toolkit: genera hook che gestiscono cache, deduplica, loading, errore e
invalidazione.

```js
// src/redux/api/photosApi.js
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const photosApi = createApi({
  reducerPath: 'photosApi',
  baseQuery: fetchBaseQuery({ baseUrl: 'https://api.unsplash.com' }),
  tagTypes: ['Photo'],
  endpoints: (builder) => ({
    getPhotos: builder.query({
      query: ({ page = 1, query = '' }) =>
        query ? `/search/photos?query=${query}&page=${page}` : `/photos?page=${page}`,
      providesTags: ['Photo'],
    }),
  }),
});

export const { useGetPhotosQuery } = photosApi;
```

```jsx
const { data, isLoading, error } = useGetPhotosQuery({ page, query });
```

Sparisce tutta la slice scritta a mano dell'esempio precedente: niente thunk, niente
`extraReducers`, niente stati di loading da mantenere. Vale la pena introdurlo quando le chiamate
API sono più di due o tre, o quando ti accorgi di stare riscrivendo una cache.

Il confine resta quello di `scelta-dello-stato.md`: RTK Query (o TanStack Query) per i dati del
server, le slice normali per lo stato dell'interfaccia.
