# Custom hook pronti

Da adattare al progetto. Il nome deve sempre iniziare con `use`: è il segnale che abilita le regole
degli hook su quella funzione.

Indice: [useFetch](#usefetch) · [useTitle](#usetitle) · [useToggle](#usetoggle) ·
[useLocalStorage](#uselocalstorage) · [useDebounce](#usedebounce) ·
[useWindowSize](#usewindowsize) · [useClickOutside](#useclickoutside) ·
[usePrevious](#useprevious) · [Quando estrarne uno](#quando-estrarre-un-custom-hook)

---

## useFetch

```jsx
// hooks/useFetch.js
import { useState, useEffect } from 'react';

export const useFetch = (url) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!url) return;
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

```jsx
const { data, loading, error } = useFetch(`${baseUrl}/search?s=${termine}`);
if (loading) return <Skeleton />;
if (error) return <ErrorMessage message={error} />;
```

`[url]` come dipendenza fa scattare una nuova chiamata ogni volta che l'URL cambia — cioè a ogni
ricerca. L'`AbortController` annulla la richiesta precedente ed evita che una risposta lenta
sovrascriva una più recente.

Due avvertenze: ogni chiamata dell'hook ha stato **indipendente** (due componenti che usano
`useFetch` non condividono nulla), e se `url` viene costruito con un oggetto nelle dipendenze
l'effetto girerà a ogni render. Su progetti reali, TanStack Query fa questo più cache, retry e
deduplica.

---

## useTitle

```jsx
export const useTitle = (title) => {
  useEffect(() => {
    document.title = `${title} | Nome del sito`;
  }, [title]);
};
```

```jsx
const About = () => {
  useTitle("Chi siamo");
  return <section>...</section>;
};
```

Tre righe, e ogni pagina ha il titolo giusto nella scheda del browser. È l'esempio minimo che fa
capire il valore dei custom hook: un comportamento ripetitivo incapsulato in una riga.

Attenzione a chiamarlo **prima** di qualsiasi return condizionale: `if (loading) return ...` seguito
da `useTitle(...)` viola la prima regola degli hook.

---

## useToggle

```jsx
export const useToggle = (iniziale = false) => {
  const [valore, setValore] = useState(iniziale);
  const toggle = useCallback(() => setValore((prev) => !prev), []);
  const apri   = useCallback(() => setValore(true), []);
  const chiudi = useCallback(() => setValore(false), []);
  return { valore, toggle, apri, chiudi };
};
```

```jsx
const { valore: sidebarAperta, apri, chiudi } = useToggle();
```

Esporre `apri` e `chiudi` separati oltre a `toggle` rende esplicito cosa fa ogni bottone: la
Navbar apre, la Sidebar chiude. Più leggibile di due `toggle` che fanno cose opposte.

---

## useLocalStorage

```jsx
export const useLocalStorage = (chiave, valoreIniziale) => {
  const [valore, setValore] = useState(() => {
    try {
      const salvato = localStorage.getItem(chiave);
      return salvato !== null ? JSON.parse(salvato) : valoreIniziale;
    } catch {
      return valoreIniziale;
    }
  });

  useEffect(() => {
    try {
      localStorage.setItem(chiave, JSON.stringify(valore));
    } catch {
      // quota superata o modalità privata: non deve far crashare l'app
    }
  }, [chiave, valore]);

  return [valore, setValore];
};
```

Si usa esattamente come `useState`: `const [carrello, setCarrello] = useLocalStorage('cart', [])`.

I `try/catch` non sono paranoia: `localStorage` lancia eccezioni in modalità privata su alcuni
browser e quando la quota è esaurita, e un throw durante il primo render fa sparire l'applicazione.
`JSON.parse` su un valore corrotto fa lo stesso.

Non metterci token di autenticazione o dati sensibili: è leggibile da qualsiasi script eseguito
sulla pagina.

---

## useDebounce

```jsx
export const useDebounce = (valore, ritardo = 400) => {
  const [ritardato, setRitardato] = useState(valore);

  useEffect(() => {
    const timer = setTimeout(() => setRitardato(valore), ritardo);
    return () => clearTimeout(timer);    // il cuore del debounce
  }, [valore, ritardo]);

  return ritardato;
};
```

Il meccanismo sta nella cleanup: ogni nuovo carattere cancella il timer precedente, quindi il
valore si aggiorna solo quando l'utente smette di scrivere per `ritardo` millisecondi.

Serve per le ricerche che chiamano un'API (otto lettere = una richiesta invece di otto) e per
filtri su liste grandi.

---

## useWindowSize

```jsx
export const useWindowSize = () => {
  const [size, setSize] = useState({ width: window.innerWidth, height: window.innerHeight });

  useEffect(() => {
    const handler = () => setSize({ width: window.innerWidth, height: window.innerHeight });
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  }, []);

  return size;
};
```

Senza la rimozione del listener, React tenterebbe di aggiornare lo stato di un componente che non
esiste più.

Prima di usarlo, però: il layout responsive si fa con le media query CSS, non in JavaScript. Questo
hook serve quando la *logica* cambia con la dimensione (quante slide mostrare, se caricare una
mappa), non per nascondere elementi.

---

## useClickOutside

```jsx
export const useClickOutside = (ref, callback) => {
  useEffect(() => {
    const handler = (e) => {
      if (ref.current && !ref.current.contains(e.target)) callback();
    };
    document.addEventListener('mousedown', handler);
    return () => document.removeEventListener('mousedown', handler);
  }, [ref, callback]);
};
```

```jsx
const menuRef = useRef(null);
useClickOutside(menuRef, chiudi);
<div ref={menuRef}>...</div>
```

Per menu a tendina, modal e popover. Usa `mousedown` e non `click`, altrimenti un elemento rimosso
durante il mousedown fa scattare la chiusura in modo imprevisto. Aggiungi anche la chiusura con
Escape: chi naviga da tastiera non ha un "fuori" su cui cliccare.

Se `callback` è una funzione ricreata a ogni render, l'effetto si riattacca ogni volta: avvolgila
in `useCallback` nel chiamante.

---

## usePrevious

```jsx
export const usePrevious = (valore) => {
  const ref = useRef();
  useEffect(() => { ref.current = valore; }, [valore]);
  return ref.current;
};
```

Ritorna il valore del render precedente. Utile per animazioni che dipendono dalla direzione del
cambiamento, o per capire in debug cosa è cambiato davvero.

---

## Quando estrarre un custom hook

Il segnale concreto: **due componenti con le stesse righe di `useState` e lo stesso `useEffect`**.
Non prima.

Cosa può e cosa non può fare:

- riutilizza **logica**, non interfaccia (quella sono i componenti);
- ogni chiamata è indipendente: due componenti che usano lo stesso hook hanno stati separati.
  Condividere stato è compito di Context o dello store;
- può chiamare altri hook, incluso un altro custom hook;
- ritorna quello che vuoi: un oggetto quando i valori hanno nomi propri (`{ data, loading }`), un
  array quando il chiamante deve poterli rinominare (`[valore, setValore]`, come `useState`).

Non estrarre un hook per tre righe usate in un punto solo: l'indirezione costa più di quanto
risparmia.
