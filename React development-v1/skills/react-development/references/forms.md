# Form e validazione

Indice: [Input controllato](#input-controllato) · [Più campi](#piu-campi-un-solo-handler) ·
[Submit](#il-flusso-di-submit) · [Altri tipi di campo](#altri-tipi-di-campo) ·
[Input non controllati](#input-non-controllati) · [Validazione a mano](#validazione-a-mano) ·
[Formik e Yup](#formik-e-yup) · [Accessibilità](#accessibilita-dei-form)

---

## Input controllato

Un input controllato ha il valore determinato dallo stato React, non dal DOM. React diventa la
**single source of truth**.

```jsx
const [nome, setNome] = useState("");

<input
  type="text"
  value={nome}                              // il valore VIENE dallo stato
  onChange={(e) => setNome(e.target.value)} // ogni tasto AGGIORNA lo stato
/>
```

Il ciclo: l'utente digita → scatta `onChange` → il setter aggiorna lo stato → nuovo render →
l'input riceve il nuovo `value`. Ogni singolo carattere passa attraverso React. Sembra
inefficiente, ed è invece ciò che permette validazione in tempo reale, formattazione automatica,
campi che dipendono l'uno dall'altro, e il reset del campo dopo il submit.

Due errori frequenti:

- **`value` senza `onChange`**: l'input diventa di sola lettura e React avvisa in console. Se
  volevi davvero un campo bloccato, usa `readOnly`.
- **`value={undefined}` al primo render**: React lo considera non controllato, e quando arriva un
  valore lancia "A component is changing an uncontrolled input to be controlled". Inizializza
  sempre con stringa vuota, mai con `undefined` o `null`.

---

## Più campi, un solo handler

Con cinque campi, cinque `useState` diventano ripetitivi. Un oggetto e un handler solo:

```jsx
const [dati, setDati] = useState({ nome: "", email: "", eta: "" });

const gestisciCambio = ({ target: { name, value } }) => {
  setDati((prev) => ({ ...prev, [name]: value }));
};

<input name="nome"  value={dati.nome}  onChange={gestisciCambio} />
<input name="email" value={dati.email} onChange={gestisciCambio} type="email" />
```

Due meccanismi da capire:

1. **L'attributo `name`** deve corrispondere esattamente alla chiave nello stato: è ciò che
   permette a un solo handler di servire tutti i campi.
2. **Le chiavi dinamiche** `[name]: value`: le parentesi quadre significano "usa il *valore* della
   variabile come chiave". Senza, creeresti letteralmente una chiave chiamata "name", sempre la
   stessa.

Nota le parentesi tonde attorno all'oggetto in `(prev) => ({ ... })`: senza, JavaScript
interpreterebbe la graffa come apertura di blocco e la funzione ritornerebbe `undefined`.

---

## Il flusso di submit

```jsx
const gestisciSubmit = (e) => {
  e.preventDefault();
  if (!nome.trim()) {
    setErrore("Il nome è obbligatorio");
    return;
  }
  setPersone((prev) => [...prev, { id: crypto.randomUUID(), nome }]);
  setNome("");        // reset, possibile solo perché l'input è controllato
  setErrore(null);
};

<form onSubmit={gestisciSubmit}>
  <input value={nome} onChange={(e) => setNome(e.target.value)} />
  <button type="submit">Aggiungi</button>
</form>
```

- `e.preventDefault()` è obbligatorio: senza, il browser ricarica la pagina e in una SPA perdi
  tutto lo stato dell'applicazione.
- L'`onSubmit` va sul `<form>`, non un `onClick` sul bottone: così funziona anche il tasto Invio,
  che è il comportamento atteso dall'utente.
- `crypto.randomUUID()` per gli id locali; in produzione l'id arriva dal server. `Date.now()` va
  bene per un esercizio ma produce duplicati se due elementi nascono nello stesso millisecondo.
- Se il submit chiama un'API, disabilita il bottone durante l'invio con uno stato `isSubmitting`,
  altrimenti l'utente fa doppio click e invia due volte.

---

## Altri tipi di campo

```jsx
// checkbox: si usa checked, non value
<input type="checkbox" checked={attivo} onChange={(e) => setAttivo(e.target.checked)} />

// select
<select value={categoria} onChange={(e) => setCategoria(e.target.value)}>
  <option value="">Tutte</option>
  {categorie.map((c) => <option key={c} value={c}>{c}</option>)}
</select>

// radio: stesso name, value diverso
<input type="radio" name="piano" value="base" checked={piano === 'base'}
       onChange={(e) => setPiano(e.target.value)} />

// file: NON può essere controllato, si legge dall'evento
<input type="file" onChange={(e) => setFile(e.target.files[0])} />

// number: e.target.value è SEMPRE una stringa
onChange={(e) => setEta(e.target.value === '' ? '' : Number(e.target.value))}
```

L'ultimo punto è una fonte di bug silenziosi: `"5" + 1` fa `"51"`. Converti al momento della
lettura, non quando ti accorgi del problema.

---

## Input non controllati

Il valore resta nel DOM e si legge solo al submit, tramite `useRef`:

```jsx
const inputRef = useRef(null);

const gestisciSubmit = (e) => {
  e.preventDefault();
  console.log(inputRef.current.value);
};

<input type="text" ref={inputRef} defaultValue="valore iniziale" />
```

Nota `defaultValue` al posto di `value`. Utile quando il valore non serve a ogni battuta (un form
di contatto semplice), o per campi file, o per non ridisegnare a ogni carattere su form molto
grandi. React consiglia il controllato come impostazione predefinita, e la maggior parte delle
librerie moderne (React Hook Form) usa invece i non controllati proprio per le performance.

---

## Validazione a mano

Per uno o due campi, uno stato di errori e una funzione bastano:

```jsx
const [errori, setErrori] = useState({});

const valida = () => {
  const e = {};
  if (!dati.nome.trim()) e.nome = "Campo obbligatorio";
  if (!/^\S+@\S+\.\S+$/.test(dati.email)) e.email = "Email non valida";
  setErrori(e);
  return Object.keys(e).length === 0;
};
```

Il dettaglio di UX che fa la differenza: mostrare l'errore solo dopo che l'utente ha **toccato e
lasciato** il campo (`onBlur`), non mentre sta ancora scrivendo e non al primo caricamento. È
esattamente il ruolo di `touched` in Formik.

Oltre i tre o quattro campi, con validazione e stati di invio, il codice scritto a mano esplode: è
il momento di una libreria.

---

## Formik e Yup

```bash
npm install formik yup
```

```jsx
import { Formik } from 'formik';
import * as Yup from 'yup';

const validationSchema = Yup.object({
  nome: Yup.string().min(2, 'Troppo corto').required('Campo obbligatorio'),
  email: Yup.string().email('Email non valida').required('Campo obbligatorio'),
  cap: Yup.string().matches(/^[0-9]{5}$/, 'Il CAP deve avere 5 cifre').required('Obbligatorio'),
  civico: Yup.number().positive().integer().required('Obbligatorio'),
});

const Checkout = () => (
  <Formik
    initialValues={{ nome: '', email: '', cap: '', civico: '' }}
    validationSchema={validationSchema}
    onSubmit={(values, { setSubmitting, resetForm }) => {
      inviaOrdine(values);
      resetForm();
      setSubmitting(false);
    }}
  >
    {({ values, errors, touched, handleChange, handleBlur, handleSubmit, isSubmitting }) => (
      <form onSubmit={handleSubmit}>
        <input name="nome" value={values.nome} onChange={handleChange} onBlur={handleBlur} />
        {errors.nome && touched.nome && <span className="errore">{errors.nome}</span>}

        <button type="submit" disabled={isSubmitting}>Conferma</button>
      </form>
    )}
  </Formik>
);
```

La sintassi da decifrare sono le **render props**: `<Formik>` non riceve JSX come children ma una
*funzione* che riceve lo stato del form e ritorna il JSX. È un pattern precedente agli hook per
condividere logica; Formik lo usa ancora. (Esiste anche `useFormik`, la versione a hook, più
leggibile se il form è semplice.)

Cosa fornisce: `values`, `errors`, `touched`, `handleChange` (un solo handler per tutti i campi,
via `name`), `handleBlur`, `handleSubmit` (fa già `preventDefault` e valida), `isSubmitting`.

Yup costruisce uno **schema dichiarativo**: la validazione non è più sparsa in venti `if`, si legge
come un contratto. Validazioni condizionali comprese:

```js
partitaIva: Yup.string().when('tipoCliente', {
  is: 'azienda',
  then: (schema) => schema.required('Obbligatoria per le aziende'),
  otherwise: (schema) => schema.notRequired(),
})
```

**La validazione client è esperienza utente, non sicurezza.** Chiunque può disabilitare JavaScript
e inviare quello che vuole. La validazione lato server resta obbligatoria — se l'utente sta
costruendo anche il backend, ricordaglielo una volta.

**Nota sull'ecosistema:** Formik è ancora molto diffuso ma il suo sviluppo ha rallentato. Nei
progetti nuovi oggi si usa **React Hook Form** (più performante, basato su input non controllati)
con **Zod** (schema con tipizzazione TypeScript nativa). I concetti — schema, touched, errori —
sono identici, quindi chi ha imparato Formik si sposta in un pomeriggio. Se il progetto ha già
Formik, non proporre una migrazione senza motivo.

---

## Accessibilità dei form

Non è un dettaglio opzionale, ed è veloce da fare bene:

- Ogni input ha una `<label htmlFor>` collegata (`useId` genera l'id, vedi `hooks.md`).
  Il placeholder **non** è una label: sparisce appena si scrive.
- I messaggi di errore vanno collegati al campo con `aria-describedby` e marcati con
  `aria-invalid`.
- Il bottone di submit è un `<button type="submit">`, non un `div` con un onClick.
- Gli errori dopo il submit vanno annunciati: un contenitore con `role="alert"` che riceve il
  riepilogo, e il focus spostato sul primo campo invalido.

Per il resto delle regole di accessibilità e interfaccia, la skill **ui-ux-design**.
