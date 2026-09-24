# Styling e design system in React

Indice: [Quale approccio](#quale-approccio-scegliere) · [CSS Modules](#css-modules) ·
[Tailwind](#tailwind) · [styled-components](#styled-components) ·
[styled-system](#styled-system-e-i-design-system) · [Componenti di base](#i-componenti-di-base) ·
[Classi condizionali](#classi-condizionali)

Qui si tratta il **come si scrivono** gli stili in React. Le decisioni visive — palette, gerarchia,
spaziature, accessibilità, layout responsive — sono compito della skill **ui-ux-design**.

---

## Quale approccio scegliere

| Approccio | Costo a runtime | Scope | Quando |
|---|---|---|---|
| CSS importato | zero | globale | progetti piccoli, CSS già esistente |
| CSS Modules | zero | automatico per file | buon default, nessuna dipendenza |
| Tailwind | zero | utility nel markup | team che lo conosce, prototipazione rapida |
| styled-components | calcolato a runtime | automatico | stili che dipendono molto dalle props |
| styled-system | come sopra | automatico | design system con tema centralizzato |

Non cambiare l'approccio di un progetto esistente senza un motivo forte: la coerenza vale più della
preferenza. Se il progetto usa già styled-components, scrivi styled-components.

---

## CSS Modules

```jsx
import styles from './Card.module.css';

<article className={styles.card}>
  <h3 className={styles.titolo}>Titolo</h3>
</article>
```

Le classi vengono rinominate in modo univoco alla build: niente collisioni fra file, nessun costo a
runtime, e si scrive CSS normale. Funziona già in Vite e in create-react-app senza configurazione.

Le variabili CSS restano lo strumento più semplice per i temi:

```css
:root       { --bg: #fff;    --text: #333; }
.dark-theme { --bg: #1a1a1a; --text: #f0f0f0; }
```

Cambiando una classe sull'elemento `<html>` cambia l'intero schema di colori senza toccare un solo
componente.

---

## Tailwind

```jsx
<article className="rounded-lg bg-white p-4 shadow-sm dark:bg-neutral-900">
  <h3 className="text-lg font-semibold">Titolo</h3>
</article>
```

Il vantaggio: nessun file CSS da mantenere e nessun nome di classe da inventare. Lo svantaggio: il
markup diventa lungo, e la ripetizione va gestita estraendo **componenti**, non classi.

```jsx
// giusto: un componente
const Card = ({ children }) => <article className="rounded-lg bg-white p-4 shadow-sm">{children}</article>;
```

Per le classi condizionali, `clsx` o `tailwind-merge` evitano concatenazioni illeggibili.

---

## styled-components

```bash
npm install styled-components
```

```jsx
import styled from 'styled-components';

const Aside = styled.aside`
  position: fixed;
  width: 300px;
  height: 100%;
  transform: translateX(${(props) => (props.$isOpen ? '0' : '-100%')});
  transition: transform 0.3s ease-in-out;

  @media screen and (min-width: 768px) { width: 400px; }
`;

<Aside $isOpen={isSidebarOpen}>...</Aside>
```

Il `$` davanti al nome della prop è la convenzione di styled-components v6 per le **transient
props**: dice alla libreria di non passarle al DOM, evitando il warning su attributi HTML non
validi.

Vantaggi: stile e componente nello stesso file (cancelli l'uno, cancelli l'altro), stili che
dipendono dalle props con JavaScript vero, scope automatico.

Svantaggi: il CSS è calcolato a runtime, e la sintassi con i template literal è meno familiare.
Molti progetti nuovi preferiscono Tailwind o CSS Modules per questo motivo.

---

## styled-system e i design system

`styled-system` si appoggia a styled-components e permette di passare gli stili come props,
prendendo i valori da un **tema centralizzato**.

```bash
npm install styled-components styled-system
```

```js
// theme.js
const theme = {
  colors: { primary: '#3d5a80', secondary: '#ee6c4d', dark: '#293241', light: '#e0fbfc' },
  space: [0, 4, 8, 16, 32, 64, 128],
  fontSizes: [12, 14, 16, 20, 24, 32, 48],
  breakpoints: ['576px', '768px', '992px', '1200px'],
};
export default theme;
```

```jsx
import { ThemeProvider } from 'styled-components';

<ThemeProvider theme={theme}>
  <App />
</ThemeProvider>
```

```jsx
<Box width="100%" p={3} bg="primary" mb={4} />
```

Il meccanismo da capire: **`p={3}` non significa 3 pixel**. Significa "il quarto valore della scala
`space`", cioè 16px. `bg="primary"` prende `colors.primary`. Tutti i valori dell'interfaccia
passano da un unico file, e cambiare il tema cambia l'intera applicazione.

Questo è ciò che si intende con design system: non stili scritti ovunque, ma un **vocabolario
condiviso e limitato**. Chi scrive un componente non inventa una dimensione: sceglie fra quelle che
esistono.

La sintassi responsive ad array è la parte più elegante:

```jsx
<Box width={[1, 1/2, 1/3]} p={[2, 3, 4]} />
```

Si legge: larghezza 100% su mobile, 50% da tablet, 33% da desktop. I valori corrispondono in ordine
ai `breakpoints` del tema. Tre valori di responsive in una riga, senza scrivere una media query.

**Nota sull'ecosistema:** styled-system non è più attivamente sviluppato. I suoi concetti — scale,
token, props di stile, varianti — sono confluiti in librerie più recenti e in Tailwind, che
implementa la stessa idea di vocabolario limitato con un approccio diverso. Vale la pena
conoscerlo perché lo si trova in progetti esistenti e perché **il concetto di scala è quello che
conta**, non la libreria.

---

## I componenti di base

Qualsiasi design system, con qualunque libreria, si regge su pochi mattoni. Vale la pena
costruirli una volta:

```jsx
// Box: il contenitore generico che accetta le props di stile
const Box = styled('div')(space, layout, color, flexbox, border, typography);

// Text: la tipografia, con varianti predefinite
const Text = styled('p')(
  space, color, typography,
  variant({
    variants: {
      title:    { fontSize: 5, fontWeight: 'bold', color: 'dark' },
      subtitle: { fontSize: 3, color: 'grey' },
      body:     { fontSize: 2, lineHeight: 1.6 },
      caption:  { fontSize: 0, color: 'grey' },
    },
  })
);

<Text variant="title">Wiki Photos</Text>
```

Le **varianti** impediscono la proliferazione di stili quasi-uguali: chi scrive il componente non
decide arbitrariamente la dimensione del testo, sceglie fra quelle esistenti. È così che i design
system reali garantiscono coerenza.

**Stack** — il componente che distribuisce i figli con una spaziatura uniforme:

```jsx
const Stack = styled(Box)`
  display: flex;
  flex-direction: ${(p) => p.direction ?? 'column'};
  gap: ${(p) => p.theme.space[p.space ?? 3]}px;
`;
```

L'idea: invece di mettere `margin-bottom` su ogni elemento (e poi doverlo togliere dall'ultimo), è
il contenitore a gestire la spaziatura. Con `gap` è banale; prima di `gap` si usava il selettore
`& > * + *`, "ogni figlio che ha un fratello prima di sé".

**Skeleton** — il segnaposto animato durante il caricamento. Meglio di uno spinner perché mostra la
forma del contenuto in arrivo e riduce il salto visivo. Codice in
`../patterns/component-recipes.md`.

---

## Classi condizionali

```jsx
// due stati: il ternario basta
<div className={attivo ? "card attiva" : "card"}>

// più condizioni: template literal, o clsx
<div className={`card ${attivo ? 'attiva' : ''} ${disabilitato ? 'disabilitata' : ''}`}>

import clsx from 'clsx';
<div className={clsx('card', { attiva: attivo, disabilitata: disabilitato })}>
```

Lo stile inline resta la scelta giusta per i valori **calcolati a runtime**, non per lo stile
statico:

```jsx
<div style={{ width: `${percentuale}%` }} />
```
