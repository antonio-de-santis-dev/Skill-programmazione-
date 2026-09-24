# Performance e ottimizzazione

Indice: [Prima di ottimizzare](#prima-di-ottimizzare) · [memo](#reactmemo) ·
[useMemo](#usememo) · [useCallback](#usecallback) · [useTransition](#usetransition) ·
[useDeferredValue](#usedeferredvalue) · [lazy e Suspense](#lazy-e-suspense) ·
[Cose che valgono di più](#cose-che-valgono-piu-della-memoizzazione)

---

## Prima di ottimizzare

`memo`, `useMemo` e `useCallback` hanno un costo: React deve conservare i valori vecchi e fare
confronti a ogni render. Applicarli ovunque per abitudine rende l'applicazione **più lenta** e il
codice più difficile da leggere.

L'ordine corretto è sempre: **misura → trova il collo di bottiglia → intervieni solo lì**.

Come si misura:

1. React DevTools → scheda **Profiler** → registra un'interazione → guarda quali componenti si
   ridisegnano e quanto ci mettono.
2. L'opzione "Highlight updates when components render" nelle impostazioni del DevTools: evidenzia
   a schermo cosa si ridisegna mentre usi l'app. È il modo più rapido per vedere il problema.
3. Un `console.log` dentro il componente sospetto, quando serve una risposta in dieci secondi.

Se non riesci a percepire il problema usando l'applicazione, e il Profiler non mostra render da
decine di millisecondi, non c'è niente da ottimizzare.

**Nota su React 19:** il React Compiler applica automaticamente la memoizzazione dove serve. Nei
progetti che lo usano, `useMemo` e `useCallback` scritti a mano diventano in gran parte superflui.
Verifica nel `package.json` prima di riempire il codice di memoizzazioni.

---

## React.memo

Avvolge un componente e ne blocca il render quando le props non sono cambiate.

```jsx
const Item = ({ nome, immagine }) => (
  <div><img src={immagine} alt={nome} /><h4>{nome}</h4></div>
);

export default memo(Item);
```

Il problema che risolve: quando un componente si ridisegna, **tutti i suoi figli si ridisegnano**,
anche se le loro props sono identiche. Una lista di 40 elementi con un contatore sopra ridisegna 40
elementi a ogni click.

Il confronto è **superficiale**: le props primitive si confrontano per valore, oggetti, array e
funzioni per riferimento. Da qui il problema che i due hook successivi risolvono — una funzione
dichiarata nel corpo del genitore è nuova a ogni render, quindi `memo` non blocca niente e hai
pagato il confronto per nulla.

`memo` è utile su componenti che (a) si ridisegnano spesso senza motivo, (b) sono costosi da
renderizzare, (c) ricevono props stabili. Se manca una di queste tre condizioni, lascia perdere.

---

## useMemo

Memorizza il **risultato di un calcolo**, ricalcolandolo solo quando cambiano le dipendenze.

```jsx
const piuCostoso = useMemo(
  () => prodotti.reduce((max, p) => (p.prezzo > max.prezzo ? p : max), prodotti[0]),
  [prodotti]
);
```

Due usi legittimi:

1. Un calcolo davvero pesante (ordinamenti e filtri su migliaia di elementi, elaborazioni
   ricorsive). Filtrare venti elementi non è un calcolo pesante.
2. **Stabilizzare un riferimento**: un oggetto o un array passato come prop a un componente
   `memo`, o usato come dipendenza di un `useEffect`. Questo secondo uso è spesso il più
   importante, e non riguarda il costo del calcolo ma l'identità del risultato.

```jsx
// senza useMemo, l'effetto gira a ogni render: l'oggetto è sempre nuovo
const params = useMemo(() => ({ page, query }), [page, query]);
useEffect(() => { fetchData(params); }, [params]);
```

---

## useCallback

Memorizza una **funzione**, così mantiene lo stesso riferimento fra un render e l'altro.

```jsx
const rimuovi = useCallback((id) => {
  setProdotti((prev) => prev.filter((p) => p.id !== id));
}, []);
```

È il complemento indispensabile di `memo`: senza, un figlio memoizzato che riceve un handler si
ridisegna comunque, perché la funzione è nuova ogni volta.

Nota l'uso della forma funzionale del setter: permette di lasciare l'array di dipendenze vuoto,
perché non serve leggere `prodotti` dall'esterno. Se invece leggi una variabile, va dichiarata, e
allora la funzione cambia quando cambia quella — il che spesso annulla il vantaggio. Ripensare la
dipendenza vale più che aggiungere memoizzazione.

`dispatch` (sia di `useReducer` sia di React-Redux) e i setter di `useState` sono **già stabili**:
metterli nelle dipendenze non fa danno, avvolgerli in `useCallback` non serve.

---

## Il trio in una tabella

| | Memorizza | Serve contro |
|---|---|---|
| `memo` | un componente | render inutili di un figlio |
| `useMemo` | un valore | calcoli costosi, riferimenti instabili |
| `useCallback` | una funzione | riferimenti nuovi che rompono `memo` e fanno rigirare gli effetti |

---

## useTransition

Marca un aggiornamento di stato come **non urgente**, così non blocca l'interfaccia.

```jsx
const [isPending, startTransition] = useTransition();

const gestisciCambio = (e) => {
  setTesto(e.target.value);              // urgente: l'input risponde subito
  startTransition(() => {
    setRisultati(filtraMigliaiaDiElementi(e.target.value));   // può attendere
  });
};

{isPending ? <Spinner /> : <Lista items={risultati} />}
```

Il caso tipico: un campo di ricerca che filtra una lista enorme. Senza, ogni carattere digitato
blocca il thread e l'input diventa scattoso. React dà priorità all'input e interrompe il lavoro
sulla lista se l'utente continua a scrivere.

`isPending` serve a mostrare un indicatore nel frattempo, altrimenti l'utente non capisce che sta
succedendo qualcosa.

---

## useDeferredValue

Stessa idea, applicata al **valore** invece che all'aggiornamento.

```jsx
const [testo, setTesto] = useState("");
const testoRitardato = useDeferredValue(testo);

<input value={testo} onChange={(e) => setTesto(e.target.value)} />
<ListaPesante testo={testoRitardato} />   // ListaPesante è avvolta in memo
```

Quale dei due usare: **useTransition** quando controlli tu il codice che aggiorna lo stato,
**useDeferredValue** quando ricevi un valore dall'esterno (una prop, un hook di libreria) e non
puoi intervenire sull'aggiornamento. Va quasi sempre abbinato a `memo` sul componente pesante,
altrimenti il vantaggio si perde.

---

## lazy e Suspense

Il code splitting spezza il bundle: ogni pagina viene scaricata solo quando serve.

```jsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./screens/Dashboard'));

<Suspense fallback={<SkeletonPagina />}>
  <Routes>
    <Route path="/dashboard" element={<Dashboard />} />
  </Routes>
</Suspense>
```

Senza `lazy`, la build produce un unico file con tutta l'applicazione: chi apre la home scarica
anche il codice della dashboard e di ogni altra pagina. Su applicazioni grandi la differenza sul
primo caricamento è sostanziale.

Il punto naturale dove spezzare è la rotta. Altri buoni candidati: modal complessi, editor,
librerie di grafici, tutto ciò che è pesante e non serve al primo caricamento.

Più `Suspense` a granularità diversa sono spesso meglio di uno solo, perché ogni sezione appare
appena è pronta invece di aspettare la più lenta. Il `fallback` ideale è uno **skeleton** con la
forma del contenuto in arrivo, non uno spinner: riduce il salto visivo quando i dati caricano.

---

## Cose che valgono più della memoizzazione

Nell'ordine in cui conviene guardarle:

1. **Immagini.** Sono quasi sempre il peso maggiore. Dimensioni giuste, formati moderni,
   `loading="lazy"`, `width` e `height` dichiarati per evitare il salto del layout.
2. **Liste lunghe.** Oltre qualche centinaio di righe visibili, nessuna memoizzazione basta: serve
   la virtualizzazione (renderizzare solo le righe visibili) o la paginazione.
3. **Dove vive lo stato.** Uno stato tenuto troppo in alto ridisegna mezza applicazione. Spostarlo
   nel componente che lo usa davvero, o isolare la parte che cambia in un componente separato,
   risolve più render inutili di qualsiasi `memo`.
4. **Chiamate API ripetute.** Una cache (TanStack Query o simili) elimina richieste, non
   millisecondi di render.
5. **Dipendenze inutili nel bundle.** Una libreria di date da 70 kB importata per formattare una
   data. `npx vite-bundle-visualizer` mostra cosa c'è dentro.

La memoizzazione arriva dopo tutte queste, e solo sul punto che il Profiler ha indicato.
