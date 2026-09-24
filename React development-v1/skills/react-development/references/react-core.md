# React — fondamenta

Indice: [Creare il progetto](#creare-il-progetto) · [Struttura](#struttura-del-progetto) ·
[Componenti](#componenti) · [JSX](#le-regole-di-jsx) · [Props](#props) ·
[Children](#children) · [Liste e key](#liste-e-key) · [Eventi](#eventi) ·
[Render condizionale](#render-condizionale) · [Stili](#stili) · [Deploy](#deploy-e-variabili-dambiente)

---

## Creare il progetto

```bash
npm create vite@latest nome-app -- --template react
cd nome-app
npm install
npm run dev
```

`create-react-app` (`npx create-react-app nome-app`) è deprecato ma lo trovi in ogni corso
registrato prima del 2024 e in molti progetti esistenti. Le differenze che contano davvero:

| | create-react-app | Vite |
|---|---|---|
| Avvio | `npm start`, porta 3000 | `npm run dev`, porta 5173 |
| Variabili d'ambiente | prefisso `REACT_APP_`, lette con `process.env.REACT_APP_X` | prefisso `VITE_`, lette con `import.meta.env.VITE_X` |
| Entry point | `src/index.js` | `src/main.jsx` |
| Estensione file con JSX | `.js` va bene | deve essere `.jsx` |

Scambiare queste due cose produce errori che sembrano misteriosi: una variabile d'ambiente
`undefined`, o un file JSX che non compila. Verifica sempre quale dei due usa il progetto prima di
scrivere.

## Struttura del progetto

```
src/
├── components/     pezzi riutilizzabili (Navbar, Card, Button, Modal)
├── screens/        o pages/ — componenti associati a una rotta
├── hooks/          custom hook (useFetch, useTitle)
├── context/        provider e context
├── utils/          funzioni pure di supporto (helpers, formattatori)
├── assets/         immagini, icone
├── App.jsx
└── main.jsx        punto di ingresso: crea la root e monta <App />
```

La separazione `components/` vs `screens/` diventa necessaria appena entra il router: le screen
corrispondono a un URL, i components no. Su un progetto piccolo una sola cartella va benissimo —
non creare la gerarchia prima che serva.

Il punto di ingresso, che spiega cos'è una SPA:

```jsx
// main.jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';
import './index.css';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

Esiste **un solo** `index.html`, con dentro `<div id="root"></div>`. Tutta l'applicazione viene
disegnata lì dentro da JavaScript; navigando fra le sezioni il browser non ricarica nulla. Da qui
sia la sensazione di istantaneità sia il fatto che serva una libreria di routing per gestire gli
URL (vedi `routing.md`).

`StrictMode` in sviluppo monta, smonta e rimonta ogni componente: gli effetti girano **due volte**.
Non è un bug ed è la causa numero uno della domanda "perché la mia fetch parte due volte". In
produzione non succede. Serve a rendere visibili gli effetti senza cleanup.

## Componenti

Un componente è una funzione JavaScript che ritorna JSX.

```jsx
const Card = () => {
  return <article className="card">Contenuto</article>;
};

export default Card;
```

Due regole non negoziabili:

1. **Il nome inizia con la maiuscola.** React distingue `<Card />` (il tuo componente) da `<div>`
   (tag HTML nativo) esattamente così. Scritto minuscolo, React cerca un tag HTML chiamato "card",
   non lo trova, e non renderizza nulla — senza errori espliciti.
2. **Ritorna qualcosa**: JSX, oppure `null` se non deve mostrare niente. `null` è un ritorno
   legittimo, non un errore.

`export default` (uno per file, chi importa sceglie il nome) o export nominale (quanti ne vuoi, il
nome deve corrispondere):

```jsx
export default Card;                  // import Card from './Card';
export const Button = () => <button />;  // import { Button } from './Elementi';
```

## Le regole di JSX

JSX non è HTML e non è una stringa: viene compilato in chiamate a funzione.

```jsx
const el = <h1 className="titolo">Ciao</h1>;
// diventa
const el = React.createElement("h1", { className: "titolo" }, "Ciao");
```

Tutte le regole che seguono sono conseguenze di questo, non capricci.

| Regola | Perché |
|---|---|
| Un solo elemento radice | Una funzione ritorna un solo valore. Usa `<>...</>` (Fragment) per non aggiungere un `div` inutile al DOM |
| Tutti i tag vanno chiusi: `<img />`, `<br />`, `<input />` | È XML, non HTML permissivo |
| `className` al posto di `class` | `class` è parola riservata in JavaScript |
| camelCase sugli attributi: `onClick`, `tabIndex`, `htmlFor` | sono chiavi di un oggetto JavaScript |
| Lo stile inline è un oggetto: `style={{ backgroundColor: "red" }}` | le doppie graffe sono "entro in JS" + "oggetto" |
| Commenti: `{/* testo */}` | è un'espressione JavaScript dentro JSX |

Dentro le graffe vanno **espressioni**, non istruzioni: un'espressione produce un valore, `if` e
`for` eseguono un'azione.

```jsx
{ if (loggato) { ... } }                 // errore di sintassi
{ loggato ? <Profilo /> : <Login /> }    // corretto
```

È il motivo strutturale per cui in React si usano ovunque ternario e `&&` invece di `if`: non è
una scelta stilistica.

## Props

Le props sono l'oggetto che il genitore passa al figlio. Un componente è una funzione, e le props
sono il suo primo parametro.

```jsx
const Prodotto = ({ nome, prezzo, disponibile = true }) => (
  <article>
    <h3>{nome}</h3>
    <p>{prezzo} €</p>
    {!disponibile && <span>Esaurito</span>}
  </article>
);

<Prodotto nome="Cuffie" prezzo={49.9} disponibile={false} />
```

Quattro cose da sapere:

- **Le graffe servono per tutto ciò che non è stringa.** `numero="2"` passa la stringa `"2"`,
  `numero={2}` passa il numero. Senza graffe finisci con `"2" + 1 === "21"`.
- **Le props sono in sola lettura.** Un componente non modifica mai le props che riceve. Se un dato
  deve cambiare, è stato (`hooks.md`).
- **Ogni istanza ha le sue props.** Cinque `<Prodotto />` sono cinque oggetti separati che non
  condividono nulla.
- **Destruttura nella firma** e metti lì i valori di default: è JavaScript puro, funziona senza
  librerie, e sostituisce `defaultProps` (in via di deprecazione sulle function component).

Lo spread passa un oggetto intero come props:

```jsx
{prodotti.map((p) => <Prodotto key={p.id} {...p} />)}
```

Comodo e rischioso insieme: passa anche le proprietà che non ti aspetti, e rende più difficile
capire cosa riceve davvero il componente. Usalo quando la forma dei dati coincide con la firma del
componente, non per risparmiare tre righe.

## Children

`children` è la prop speciale che contiene tutto ciò che sta fra il tag di apertura e quello di
chiusura.

```jsx
const Card = ({ titolo, children }) => (
  <article className="card">
    <h3>{titolo}</h3>
    <div className="card-body">{children}</div>
  </article>
);

<Card titolo="Riepilogo">
  <p>Qualsiasi cosa qui dentro.</p>
  <Button>Conferma</Button>
</Card>
```

È il meccanismo con cui si costruiscono contenitori riutilizzabili — layout, modal, provider — che
non sanno nulla del contenuto che ospitano. Ogni Provider di Context, di tema o di store usa
esattamente questo.

## Liste e key

```jsx
{prodotti.map(({ id, nome }) => (
  <li key={id}>{nome}</li>
))}
```

`map` trasforma un array di dati in un array di elementi; `filter` lo restringe. React sa
renderizzare un array di elementi.

**La key** serve a React per riconoscere lo stesso elemento fra un render e l'altro. Senza, può
confrontare solo per posizione: inserendo un elemento in cima, crede che siano cambiati tutti.

Le regole: unica **fra fratelli** (non globalmente), **stabile** nel tempo, idealmente l'`id` che
arriva dai dati.

`key={index}` è accettabile **solo** se la lista è statica, mai riordinata né filtrata, e gli
elementi non hanno stato interno. Altrimenti produce bug visibili e difficili da capire: cancelli
il primo elemento, tutti gli indici scalano, e il testo scritto in un input "salta" su un'altra
riga. Non generare mai key con `Math.random()` o `useId()`: cambierebbero a ogni render,
annullando il meccanismo.

## Eventi

```jsx
const Bottone = () => {
  const gestisciClick = () => console.log("Cliccato");
  return <button onClick={gestisciClick}>Clicca</button>;
};
```

L'errore su cui inciampano tutti:

```jsx
<button onClick={gestisciClick}>    // corretto: passo la funzione
<button onClick={gestisciClick()}>  // sbagliato: la ESEGUO durante il render
```

Nel secondo caso, se la funzione modifica lo stato, ottieni un loop infinito di render.

Per passare un argomento serve una arrow function che avvolge:

```jsx
<button onClick={() => elimina(prodotto.id)}>Elimina</button>
```

L'oggetto evento è il primo parametro: `e.target.value` per il valore di un input,
`e.preventDefault()` per fermare il comportamento nativo (indispensabile sul submit di un form:
senza, il browser ricarica la pagina e in una SPA perdi tutto lo stato).

## Render condizionale

Tre forme, tutte legittime, scelte in base al caso:

```jsx
// 1. Return anticipato — per stati che escludono tutto il resto
if (loading) return <Spinner />;
if (error) return <ErrorMessage message={error} />;

// 2. Short circuit — per mostrare o non mostrare un pezzo
{messaggi.length > 0 && <Badge count={messaggi.length} />}

// 3. Ternario — per scegliere fra due alternative
{loggato ? <Profilo /> : <Login />}
```

La trappola dei numeri, subdola perché silenziosa: `{lista.length && <Lista />}` con `length === 0`
renderizza uno **0** solitario nella pagina, perché React non renderizza i booleani ma i numeri sì.
Scrivi sempre una condizione booleana esplicita: `{lista.length > 0 && <Lista />}`.

I return condizionali vanno **dopo tutte le chiamate agli hook**, altrimenti in alcuni render certi
hook non verrebbero eseguiti (vedi le regole in `hooks.md`).

Differenza che conta: `{visibile && <Modal />}` rimuove il componente dal DOM e ne resetta lo stato
interno; `<div className={visibile ? "modal show" : "modal"}>` lo tiene nel DOM e lo nasconde. La
seconda serve quando vuoi conservare lo stato o animare l'uscita con una transizione CSS.

## Stili

Quattro approcci, tutti in uso:

- **CSS normale importato** (`import './App.css'`): semplice, ma le classi sono globali.
- **CSS Modules** (`styles.module.css` → `className={styles.card}`): scope automatico, costo zero a
  runtime. Buon default.
- **Tailwind**: utility nel markup, nessun file CSS da mantenere. Oggi molto diffuso.
- **styled-components**: CSS dentro JavaScript, stili che dipendono dalle props. Comodo, ma il CSS
  è calcolato a runtime.

```jsx
// styled-components, per capire cosa leggerai nei progetti
const Aside = styled.aside`
  transform: translateX(${(props) => (props.$isOpen ? '0' : '-100%')});
  transition: transform 0.3s ease-in-out;
`;
```

Il `$` davanti al nome della prop (`$isOpen`) evita che finisca come attributo nel DOM: è la
convenzione di styled-components v6.

Lo stile inline serve per valori calcolati a runtime, non per lo stile statico:

```jsx
<div style={{ width: `${percentuale}%`, color: "var(--clr-primary)" }} />
```

Le variabili CSS funzionano anche qui, ed è il ponte più pulito fra il CSS del progetto e i valori
dinamici.

## Deploy e variabili d'ambiente

```bash
npm run build      # produce dist/ (Vite) o build/ (CRA)
```

Su Netlify: trascini la cartella prodotta su app.netlify.com/drop, oppure colleghi il repository
indicando comando di build e cartella di pubblicazione. Con un router client-side serve una regola
di rewrite, altrimenti il refresh su `/about` dà 404 (vedi `routing.md`).

Le chiavi API:

```bash
# .env — e .env DEVE stare in .gitignore
VITE_API_KEY=la_tua_chiave
```

```js
const key = import.meta.env.VITE_API_KEY;   // Vite
const key = process.env.REACT_APP_API_KEY;  // create-react-app
```

Da capire bene, perché è un equivoco frequente: **una variabile d'ambiente nel frontend non è un
segreto**. Finisce nel bundle JavaScript scaricato dal browser, e chiunque può leggerla. Serve a
non committare la chiave nel repository e a cambiarla fra ambienti. Una chiave che deve restare
segreta va tenuta su un backend che fa da proxy. Se l'utente sta per mettere una chiave con costi
associati in un'app React, diglielo.
