# Ricette di componenti

Codice funzionante da **adattare**, non da copiare alla lettera: nomi, forma dei dati e classi CSS
vanno allineati al progetto reale.

Indice: [Lista con rimozione](#lista-con-rimozione) · [Toggle e modal](#toggle-e-modal) ·
[Dark mode](#dark-mode-con-persistenza) · [Filtro per categoria](#filtro-per-categoria) ·
[Ricerca con debounce](#ricerca-con-debounce) · [Slider](#slider-con-navigazione-circolare) ·
[Navbar animata](#navbar-con-altezza-animata) · [Skeleton](#skeleton-e-caricamento-immagini) ·
[Copia negli appunti](#copia-negli-appunti) · [Context + reducer](#context--reducer-carrello)

---

## Lista con rimozione

Il mattone di base: stato con array, `map`, `key`, immutabilità.

```jsx
const Lista = () => {
  const [persone, setPersone] = useState(data);

  const rimuovi = (id) => setPersone((prev) => prev.filter((p) => p.id !== id));

  return (
    <section>
      {persone.map(({ id, nome, data: quando, img }) => (
        <article key={id} className="persona">
          <img src={img} alt={nome} />
          <div>
            <h4>{nome}</h4>
            <p>{quando}</p>
          </div>
          <button onClick={() => rimuovi(id)} aria-label={`Rimuovi ${nome}`}>×</button>
        </article>
      ))}

      {persone.length === 0 && <p>Nessun elemento.</p>}
      {persone.length > 0 && <button onClick={() => setPersone([])}>Cancella tutto</button>}
    </section>
  );
};
```

Nota `aria-label` sul bottone: un `×` da solo non dice niente a chi usa uno screen reader.

---

## Toggle e modal

```jsx
const [aperto, setAperto] = useState(false);

<button onClick={() => setAperto((prev) => !prev)}>
  {aperto ? "Nascondi" : "Mostra"}
</button>

{aperto && <Messaggio />}
```

`{aperto && <Modal />}` **rimuove** il componente dal DOM: quando riappare, lo stato interno è
resettato. Se ti serve conservarlo o animare l'uscita con una transizione CSS, tienilo montato e
cambia classe:

```jsx
<div className={aperto ? "modal mostra" : "modal"}>
```

Per un modal vero servono anche: chiusura con Escape, click sullo sfondo, focus spostato dentro e
restituito al ritorno. Considera `<dialog>` nativo, che fa gran parte di questo da solo.

---

## Dark mode con persistenza

```jsx
const leggiTema = () => localStorage.getItem('theme') === 'dark';

const App = () => {
  const [dark, setDark] = useState(leggiTema);   // funzione, SENZA parentesi: lazy init

  useEffect(() => {
    document.documentElement.classList.toggle('dark-theme', dark);
    localStorage.setItem('theme', dark ? 'dark' : 'light');
  }, [dark]);

  return <button onClick={() => setDark((prev) => !prev)}>
    {dark ? "Tema chiaro" : "Tema scuro"}
  </button>;
};
```

```css
:root       { --bg: #fff;    --text: #333; }
.dark-theme { --bg: #1a1a1a; --text: #f0f0f0; }
```

Tre punti che valgono più del codice:

- `useState(leggiTema)` senza parentesi: React esegue la funzione solo al primo render. Con le
  parentesi verrebbe chiamata a ogni render, e il risultato buttato via.
- La classe va sull'elemento `<html>`, non su un componente: il tema riguarda tutta la pagina.
  Cambiando una classe cambia l'intero schema di colori senza toccare un solo componente.
- Come default, la preferenza di sistema è meglio di `false`:
  `window.matchMedia('(prefers-color-scheme: dark)').matches`.

---

## Filtro per categoria

Il pattern è **due stati sulla stessa collezione**: i dati completi non si perdono mai.

```jsx
const tutteLeCategorie = ['tutti', ...new Set(data.map((item) => item.categoria))];

const Menu = () => {
  const [items, setItems] = useState(data);          // ciò che si vede
  const [attiva, setAttiva] = useState('tutti');

  const filtra = (categoria) => {
    setAttiva(categoria);
    setItems(categoria === 'tutti' ? data : data.filter((i) => i.categoria === categoria));
  };

  return (
    <>
      {tutteLeCategorie.map((c) => (
        <button key={c} onClick={() => filtra(c)} className={c === attiva ? 'attivo' : ''}>
          {c}
        </button>
      ))}
      <ItemList items={items} />
    </>
  );
};
```

`new Set(...)` elimina i duplicati: le categorie nascono dai dati, quindi aggiungendo un prodotto
con una categoria nuova il bottone compare da solo. Scriverle a mano significa doverle aggiornare
per sempre.

Variante più semplice quando i dati sono già in memoria: tieni solo `attiva` nello stato e calcola
la lista durante il render (`const visibili = attiva === 'tutti' ? data : data.filter(...)`). Un
valore derivabile non ha bisogno di uno stato proprio.

---

## Ricerca con debounce

```jsx
const [testo, setTesto] = useState('');
const testoRitardato = useDebounce(testo, 400);   // hook in hooks-recipes.md

useEffect(() => {
  if (!testoRitardato) return;
  cerca(testoRitardato);
}, [testoRitardato]);

<input value={testo} onChange={(e) => setTesto(e.target.value)} placeholder="Cerca..." />
```

L'input resta reattivo (si aggiorna a ogni tasto), la chiamata parte solo quando l'utente smette di
scrivere per 400 ms. Senza, una ricerca di otto lettere fa otto richieste e consuma il rate limit
in pochi minuti.

---

## Slider con navigazione circolare

```jsx
const Slider = ({ persone }) => {
  const [indice, setIndice] = useState(0);

  useEffect(() => {
    const id = setInterval(() => setIndice((prev) => (prev + 1) % persone.length), 5000);
    return () => clearInterval(id);
  }, [persone.length]);

  const prev = () => setIndice((i) => (i - 1 + persone.length) % persone.length);
  const next = () => setIndice((i) => (i + 1) % persone.length);

  return (
    <div className="slider">
      {persone.map((persona, i) => {
        let posizione = 'nextSlide';
        if (i === indice) posizione = 'activeSlide';
        if (i === indice - 1 || (indice === 0 && i === persone.length - 1)) posizione = 'lastSlide';

        return (
          <article key={persona.id} className={posizione}>
            <img src={persona.img} alt={persona.nome} />
            <h4>{persona.nome}</h4>
          </article>
        );
      })}
      <button onClick={prev} aria-label="Precedente">‹</button>
      <button onClick={next} aria-label="Successivo">›</button>
    </div>
  );
};
```

```css
.slider article { position: absolute; transition: transform 0.4s ease-in-out; opacity: 0; }
.activeSlide { transform: translateX(0);    opacity: 1; }
.lastSlide   { transform: translateX(-100%); }
.nextSlide   { transform: translateX(100%); }
```

Il modulo `%` gestisce il giro: `(indice + 1) % lunghezza` torna a 0 dopo l'ultimo. Per l'indietro
serve `+ lunghezza` prima del modulo, perché in JavaScript `-1 % 5` fa `-1`, non `4`.

La forma funzionale nel `setInterval` non è facoltativa: senza, il timer leggerebbe per sempre
l'indice congelato al momento in cui è stato creato. E il `clearInterval` nella cleanup evita di
accumulare timer.

---

## Navbar con altezza animata

Il problema: `height: auto` non è animabile, e l'altezza dipende dal numero di voci di menu, quindi
va misurata a runtime.

```jsx
const Navbar = () => {
  const [show, setShow] = useState(false);
  const containerRef = useRef(null);   // il contenitore con altezza controllata
  const linksRef = useRef(null);       // la lista, per misurarne l'altezza reale

  useEffect(() => {
    const altezza = linksRef.current.getBoundingClientRect().height;
    containerRef.current.style.height = show ? `${altezza}px` : '0px';
  }, [show]);

  return (
    <nav>
      <button onClick={() => setShow((prev) => !prev)} aria-expanded={show}>☰</button>
      <div className="links-container" ref={containerRef}>
        <ul className="links" ref={linksRef}>
          {links.map(({ id, url, text }) => <li key={id}><a href={url}>{text}</a></li>)}
        </ul>
      </div>
    </nav>
  );
};
```

```css
.links-container { overflow: hidden; transition: height 0.3s ease; }
```

**Perché due ref:** uno misura l'elemento con l'altezza naturale, l'altro applica l'altezza
calcolata. Con un solo ref, impostare l'altezza a 0 cancellerebbe la misura.

**Perché ref e non state:** non stiamo conservando un dato da mostrare, stiamo leggendo e scrivendo
direttamente sul DOM. Una misura in pixel non deve causare un render.

---

## Skeleton e caricamento immagini

```jsx
const Photo = ({ url, alt }) => {
  const [loaded, setLoaded] = useState(false);

  return (
    <div className="photo-wrapper">
      {!loaded && <div className="skeleton" />}
      <img
        src={url}
        alt={alt}
        loading="lazy"
        onLoad={() => setLoaded(true)}
        style={{ opacity: loaded ? 1 : 0, transition: 'opacity 0.3s' }}
      />
    </div>
  );
};
```

```css
.skeleton {
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}
@keyframes shimmer { from { background-position: 200% 0; } to { background-position: -200% 0; } }
```

Il punto concettuale: `loading: false` dalla tua API significa "la risposta è arrivata", non "le
immagini sono visibili". Sono due momenti diversi, e senza questa distinzione l'utente vede
riquadri bianchi al posto delle foto. Lo stato è locale a ogni immagine: metterlo in uno store
globale sarebbe sovraingegneria.

Uno skeleton con la forma del contenuto in arrivo è meglio di uno spinner: riduce il salto visivo
quando i dati caricano.

---

## Copia negli appunti

```jsx
const [copiato, setCopiato] = useState(false);

useEffect(() => {
  if (!copiato) return;
  const timer = setTimeout(() => setCopiato(false), 2000);
  return () => clearTimeout(timer);     // evita che click ripetuti accavallino i timer
}, [copiato]);

const copia = async () => {
  await navigator.clipboard.writeText(valore);
  setCopiato(true);
};

<button onClick={copia}>{copiato ? "Copiato!" : "Copia"}</button>
```

`navigator.clipboard` funziona solo su HTTPS o localhost: è una restrizione di sicurezza, non un
bug. Il `clearTimeout` nella cleanup è il pattern generale per tutti i messaggi temporanei.

---

## Context + reducer (carrello)

Lo stato complesso condiviso, senza librerie esterne. È l'architettura che Redux Toolkit
formalizza — se cresce oltre questo, vedi la skill **react-state-redux**.

```jsx
// context.jsx
const initialState = { products: [], total: 0, itemCounter: 0, loading: true };

const reducer = (state, action) => {
  switch (action.type) {
    case 'FETCH_SUCCESS':
      return { ...state, loading: false, products: action.payload };
    case 'REMOVE_ITEM':
      return { ...state, products: state.products.filter((p) => p.id !== action.payload) };
    case 'INCREASE':
      return { ...state, products: state.products.map((p) =>
        p.id === action.payload ? { ...p, amount: p.amount + 1 } : p) };
    case 'DECREASE':
      return { ...state, products: state.products
        .map((p) => (p.id === action.payload ? { ...p, amount: p.amount - 1 } : p))
        .filter((p) => p.amount > 0) };
    case 'GET_TOTALS': {
      const { total, itemCounter } = state.products.reduce(
        (acc, p) => {
          acc.total += p.price * p.amount;
          acc.itemCounter += p.amount;
          return acc;
        },
        { total: 0, itemCounter: 0 }
      );
      return { ...state, total: parseFloat(total.toFixed(2)), itemCounter };
    }
    default:
      throw new Error(`Azione non riconosciuta: ${action.type}`);
  }
};

export const AppProvider = ({ children }) => {
  const [state, dispatch] = useReducer(reducer, initialState);

  useEffect(() => { dispatch({ type: 'GET_TOTALS' }); }, [state.products]);

  return (
    <AppContext.Provider value={{
      ...state,
      removeItem: (id) => dispatch({ type: 'REMOVE_ITEM', payload: id }),
      increase:   (id) => dispatch({ type: 'INCREASE', payload: id }),
      decrease:   (id) => dispatch({ type: 'DECREASE', payload: id }),
    }}>
      {children}
    </AppContext.Provider>
  );
};
```

Tre tecniche da assimilare:

1. **`map` con ternario** per modificare un solo elemento: si ritorna l'oggetto originale per tutti
   tranne quello che interessa, di cui si crea una copia modificata. Immutabilità a ogni livello.
2. **`map` + `filter` concatenati** in DECREASE: prima decrementa, poi elimina chi è arrivato a
   zero. Una sola espressione leggibile.
3. **`reduce` per i totali**, calcolati da un `useEffect` che osserva `products`. Non devi
   ricordarti di ricalcolarli in ogni funzione: reagisci al cambiamento invece di eseguire una
   sequenza di istruzioni. `parseFloat(total.toFixed(2))` perché `toFixed` ritorna una stringa —
   classico inciampo con i prezzi.

I componenti chiamano `increase(id)` e non sanno nulla di reducer e action type: la logica resta
incapsulata nel provider.
