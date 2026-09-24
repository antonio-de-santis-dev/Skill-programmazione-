# Chiamate API e gestione dei dati

Indice: [I tre stati](#i-tre-stati-canonici) · [fetch vs Axios](#fetch-vs-axios) ·
[Istanza Axios](#istanza-axios) · [Chiavi API](#chiavi-api-e-env) ·
[Race condition](#race-condition-e-abort) · [Errori](#gestire-gli-errori-per-tipo) ·
[Paginazione e ricerca](#paginazione-e-ricerca) · [Oltre useEffect](#oltre-useeffect-le-librerie-di-server-state)

---

## I tre stati canonici

Praticamente ogni chiamata API in React si gestisce con lo stesso terzetto: **dati, caricamento,
errore**.

```jsx
const [utenti, setUtenti] = useState([]);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);

useEffect(() => {
  const getUtenti = async () => {
    try {
      const res = await fetch(url);
      if (!res.ok) throw new Error(`Errore ${res.status}`);
      setUtenti(await res.json());
      setError(null);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };
  getUtenti();
}, []);

if (loading) return <Skeleton />;
if (error) return <ErrorMessage message={error} onRetry={getUtenti} />;
if (utenti.length === 0) return <EmptyState />;
return <Lista utenti={utenti} />;
```

Quattro stati da renderizzare, non tre: **caricamento, errore, vuoto, dati**. Lo stato vuoto viene
dimenticato quasi sempre, e produce una pagina bianca che sembra un bug.

Due dettagli tecnici che causano la maggior parte dei problemi:

- **Lo stato iniziale deve avere il tipo giusto.** `useState(null)` seguito da `utenti.map(...)`
  prima che i dati arrivino dà `Cannot read properties of null`. Inizializza con `[]` per le liste.
  È la causa numero uno delle schermate bianche.
- **La callback di `useEffect` non può essere `async`**: ritornerebbe una Promise dove React si
  aspetta `undefined` o la funzione di cleanup. Dichiara la funzione async dentro e chiamala.

Quando questo blocco si ripete in due componenti, estrailo in `useFetch`
(`patterns/hooks-recipes.md`).

---

## fetch vs Axios

| | fetch | Axios |
|---|---|---|
| Disponibilità | nativo, zero dipendenze | `npm install axios` |
| Parsing JSON | `await res.json()` a mano | automatico in `data` |
| Errori HTTP (404, 500) | **non** lancia eccezione | lancia eccezione |
| baseURL e header | ripetuti a ogni chiamata | configurati una volta |
| Interceptor | no | sì |
| Timeout | solo con AbortController | opzione nativa |

Il secondo punto è quello decisivo e sorprende chi arriva da altri linguaggi: con `fetch`, una
risposta 404 **non entra nel catch**. La Promise si risolve normalmente, e senza il controllo
`if (!res.ok) throw ...` finisci per salvare la pagina di errore come se fossero dati validi.

Nessuna delle due è "migliore": `fetch` basta per due o tre chiamate, Axios conviene quando le
chiamate sono molte e condividono configurazione.

---

## Istanza Axios

```js
// api/client.js
import axios from 'axios';

const client = axios.create({
  baseURL: 'https://api.unsplash.com',
  timeout: 10000,
  headers: { Authorization: `Client-ID ${import.meta.env.VITE_UNSPLASH_KEY}` },
});

client.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 403) {
      console.error("Limite di richieste superato");
    }
    return Promise.reject(error);
  }
);

export default client;
```

```js
const { data } = await client.get('/photos', { params: { page: 1, per_page: 12 } });
```

Il valore concreto: l'URL di base, la chiave e il timeout stanno in un punto solo. Cambiare
ambiente significa cambiare una riga, non trenta. Gli interceptor permettono di gestire in modo
centralizzato token scaduti, refresh dell'autenticazione, log.

Tieni le funzioni di chiamata in un modulo dedicato (`api/photos.js`) e non dentro i componenti: il
componente chiede "dammi le foto", non deve sapere com'è fatto l'endpoint. Questa separazione è ciò
che rende possibile cambiare API senza toccare l'interfaccia.

---

## Chiavi API e .env

```bash
# .env  → deve stare in .gitignore
VITE_UNSPLASH_KEY=la_tua_chiave
```

```js
import.meta.env.VITE_UNSPLASH_KEY     // Vite
process.env.REACT_APP_UNSPLASH_KEY    // create-react-app
```

Il prefisso è obbligatorio: senza, la variabile viene semplicemente ignorata e ottieni `undefined`
senza nessun errore.

**Una variabile d'ambiente nel frontend non è un segreto**: finisce nel bundle scaricato dal
browser. Serve a non committare la chiave e a cambiarla fra ambienti. Una chiave a consumo (OpenAI,
Stripe, un servizio a pagamento) non va mai in un'app React: serve un backend che faccia da proxy.
Se vedi una chiave del genere nel codice del frontend, è un rilievo che **blocca** la review.

Chi committa una chiave su un repository pubblico la vede usata da bot nel giro di minuti. Se è già
successo, non basta rimuoverla dal codice: va revocata e rigenerata, perché resta nella storia di
Git.

---

## Race condition e abort

Sintomo: in un campo di ricerca, digiti velocemente e per un istante compaiono i risultati della
ricerca precedente. Causa: due richieste partite in ordine, tornate in ordine inverso, e la più
lenta ha sovrascritto la più recente.

```jsx
useEffect(() => {
  const controller = new AbortController();

  const cerca = async () => {
    try {
      const res = await fetch(`${url}?q=${query}`, { signal: controller.signal });
      setRisultati(await res.json());
    } catch (err) {
      if (err.name !== 'AbortError') setError(err.message);   // l'abort non è un errore
    }
  };
  cerca();

  return () => controller.abort();
}, [query]);
```

La cleanup annulla la richiesta precedente quando `query` cambia o il componente si smonta. Il
controllo su `AbortError` è necessario: senza, ogni digitazione mostrerebbe un messaggio di errore.

Con Axios: `client.get(url, { signal: controller.signal })`, stessa logica.

Questo risolve anche il warning "Can't perform a React state update on an unmounted component", che
compare quando una risposta arriva dopo che l'utente ha già cambiato pagina.

---

## Gestire gli errori per tipo

Trattare tutti gli errori come "errore generico" è una scorciatoia che si paga in supporto.

```js
const messaggioErrore = (error) => {
  const status = error.response?.status;
  if (!error.response) return "Nessuna connessione. Controlla la rete.";
  if (status === 401) return "Sessione scaduta. Accedi di nuovo.";
  if (status === 403) return "Limite di richieste raggiunto. Riprova fra qualche minuto.";
  if (status === 404) return "Contenuto non trovato.";
  if (status >= 500) return "Il servizio non risponde. Riprova più tardi.";
  return "Si è verificato un errore.";
};
```

Il messaggio deve dire all'utente **cosa può fare**, non solo cosa è andato storto. E dove ha senso
(errore di rete, 500) offri un bottone "Riprova" invece di lasciare la pagina in un vicolo cieco.

Il **rate limit** merita attenzione in sviluppo: molte API gratuite concedono poche decine di
richieste all'ora, e con l'hot reload che ricarica di continuo si esauriscono in fretta. Se
l'utente riporta un 403 improvviso su codice che funzionava, è la prima ipotesi da verificare.

---

## Paginazione e ricerca

Il pattern architetturale che tiene insieme il tutto:

```jsx
const [page, setPage] = useState(1);
const [query, setQuery] = useState('');

useEffect(() => {
  carica({ page, query });
}, [page, query]);

<button onClick={() => setPage((p) => p + 1)}>Avanti</button>
```

**I bottoni non chiamano l'API.** Cambiano solo lo stato; un effetto osserva il cambiamento e
reagisce. Questo separa *cosa vuole l'utente* da *come si ottengono i dati*, ed è la struttura che
rende manutenibile un'applicazione di medie dimensioni. Vale anche al contrario: se vedi codice in
cui ogni bottone fa la sua fetch e aggiorna lo stato a mano, è il refactoring da proporre.

Due accortezze: una nuova ricerca riporta la pagina a 1 (altrimenti cerchi "pizza" e ti trovi a
pagina 7 di zero risultati), e la query va tenuta in uno stato condiviso o nell'URL, non in uno
`useState` locale che si perde navigando.

---

## Oltre useEffect: le librerie di server state

`useEffect` + `useState` va benissimo per imparare e per applicazioni piccole. Su un'applicazione
reale, però, ti ritrovi presto a riscrivere a mano cache, deduplica delle richieste, retry,
invalidazione dopo una modifica, aggiornamento in background.

**TanStack Query** (ex React Query) fa tutto questo:

```jsx
const { data, isLoading, error } = useQuery({
  queryKey: ['photos', page, query],
  queryFn: () => client.get('/photos', { params: { page, query } }).then((r) => r.data),
});
```

Il concetto da tenere: **i dati del server non sono stato dell'applicazione**. Sono una copia
locale di qualcosa che vive altrove e può cambiare. Trattarli come stato — copiandoli in Redux o in
un context — significa doverli sincronizzare a mano per sempre.

Questo è anche il motivo per cui, in un progetto nuovo, è sbagliato mettere i dati delle API dentro
Redux "perché così è centralizzato": Redux serve per lo stato dell'**interfaccia** (carrello,
filtri, sessione), le librerie di server state per i dati remoti. Vedi la skill
**react-state-redux** per la scelta completa.
