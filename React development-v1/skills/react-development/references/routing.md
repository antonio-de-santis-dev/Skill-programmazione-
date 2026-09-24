# React Router (v6 / v7)

Indice: [Il problema](#il-problema-che-risolve) · [Setup](#setup-base) · [Link](#link-e-navlink) ·
[Parametri](#parametri-dinamici-e-useparams) · [Navigazione da codice](#usenavigate) ·
[Rotte annidate](#outlet-e-rotte-annidate) · [Query string](#query-string) ·
[Rotte protette](#rotte-protette) · [useRoutes](#useroutes) · [Deploy](#il-404-al-refresh-dopo-il-deploy) ·
[v5 → v6](#differenze-v5--v6)

---

## Il problema che risolve

Esiste un solo `index.html`. Scrivendo `/about` nella barra degli indirizzi non c'è nessun
`about.html` da servire: serve qualcosa che intercetti l'URL e decida quale componente mostrare
dentro il `div` di root, senza ricaricare la pagina.

```bash
npm install react-router-dom
```

---

## Setup base

```jsx
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';

const App = () => (
  <Router>
    <Navbar />                        {/* fuori da Routes: resta su tutte le pagine */}
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/about" element={<About />} />
      <Route path="/cocktail/:id" element={<SingleCocktail />} />
      <Route path="*" element={<ErrorPage />} />   {/* catch-all: la 404 */}
    </Routes>
    <Footer />
  </Router>
);
```

- **BrowserRouter** abilita il sistema e va avvolto attorno a tutto ciò che usa il routing.
- **Routes** sceglie la sola rotta che corrisponde meglio all'URL corrente.
- **Route** associa un percorso a un elemento. Nota `element={<Home />}`, con le parentesi
  angolari: è un elemento JSX, non un riferimento al componente.
- `path="*"` in fondo intercetta qualsiasi URL non riconosciuto. Mettilo sempre: senza, un URL
  sbagliato mostra una pagina vuota e l'utente non capisce cosa sia successo.

Convenzione di cartelle: i componenti-pagina in `screens/` o `pages/`, distinti da `components/`.

---

## Link e NavLink

```jsx
import { Link, NavLink } from 'react-router-dom';

<Link to="/about">About</Link>

<NavLink to="/about" className={({ isActive }) => (isActive ? "attivo" : "")}>
  About
</NavLink>
```

Perché non `<a href="/about">`: il tag `<a>` provoca una richiesta HTTP e un ricaricamento
completo. Perdi tutto lo stato dell'applicazione e annulli il vantaggio della SPA. `Link`
intercetta il click, aggiorna l'URL tramite la History API e lascia che il router ridisegni solo
ciò che serve.

`NavLink` sa se è attivo: serve a evidenziare la voce di menu corrente senza confrontare a mano
l'URL.

Link esterni (un altro sito) restano `<a href>` normali: `Link` è solo per la navigazione interna.

---

## Parametri dinamici e useParams

```jsx
<Route path="/persona/:id" element={<SingolaPersona />} />
```

```jsx
const { id } = useParams();
const persona = data.find((p) => p.id === Number(id));

if (!persona) return <h2>Persona non trovata</h2>;
```

**Il parametro è sempre una stringa**, anche quando l'URL contiene un numero. Da qui il `Number()`
o il `parseInt()`: confrontare `"3" === 3` con l'uguaglianza stretta dà sempre `false`, ed è un
errore molto comune che produce una pagina "non trovato" su dati che esistono.

Il link dinamico si costruisce con un template literal:

```jsx
{persone.map((p) => <Link key={p.id} to={`/persona/${p.id}`}>{p.nome}</Link>)}
```

Gestisci sempre il caso "id inesistente": l'utente può digitare l'URL a mano, e un `find` che non
trova nulla ritorna `undefined`, che poi manda in crash il render con
`Cannot read property 'nome' of undefined`.

---

## useNavigate

Navigare da codice, senza che l'utente clicchi un link:

```jsx
const navigate = useNavigate();

await autentica();
navigate('/dashboard', { replace: true });   // sostituisce la voce nella cronologia
navigate(-1);                                 // equivale al tasto "indietro"
```

`replace: true` dopo un login impedisce che il tasto indietro riporti alla pagina di accesso.

Si può anche passare uno stato invisibile nell'URL, utile per tornare indietro con contesto:

```jsx
navigate('/dettaglio', { state: { from: '/lista', scrollY: window.scrollY } });
// nella pagina di destinazione:
const { state } = useLocation();
```

---

## Outlet e rotte annidate

Un layout comune resta fisso mentre cambia solo una parte della pagina.

```jsx
<Routes>
  <Route path="/" element={<SharedLayout />}>
    <Route index element={<Home />} />
    <Route path="about" element={<About />} />
    <Route path="prodotti" element={<Prodotti />} />
    <Route path="prodotti/:id" element={<SingoloProdotto />} />
    <Route path="*" element={<ErrorPage />} />
  </Route>
</Routes>
```

```jsx
const SharedLayout = () => (
  <>
    <Navbar />
    <Outlet />      {/* qui viene inserita la rotta figlia */}
    <Footer />
  </>
);
```

Due dettagli: i percorsi figli **non** hanno lo slash iniziale (sono relativi al genitore), e
`index` identifica la rotta predefinita, quella mostrata su `/`. Navbar e Footer vengono montati
una volta sola e non si ridisegnano cambiando pagina.

---

## Query string

Per filtri, ricerche e paginazione, che devono sopravvivere a un refresh e essere condivisibili:

```jsx
const [searchParams, setSearchParams] = useSearchParams();

const query = searchParams.get('q') ?? '';
const page = Number(searchParams.get('page') ?? 1);

setSearchParams({ q: nuovoTesto, page: 1 });
```

Il vantaggio rispetto a uno `useState`: l'utente può copiare l'URL e ritrovare la stessa ricerca, e
il tasto indietro funziona come si aspetta. Per una ricerca che filtra mentre si digita, abbina un
debounce (vedi `patterns/hooks-recipes.md`) per non riscrivere l'URL a ogni carattere.

---

## Rotte protette

```jsx
const ProtectedRoute = ({ children }) => {
  const { utente, caricamento } = useAuth();
  const location = useLocation();

  if (caricamento) return <Spinner />;            // senza questo, redirect al refresh
  if (!utente) return <Navigate to="/login" replace state={{ from: location }} />;

  return children;
};

<Route path="/dashboard" element={<ProtectedRoute><Dashboard /></ProtectedRoute>} />
```

Il controllo sul caricamento non è un dettaglio: se l'informazione sull'utente arriva in modo
asincrono, al primo render è `null` e l'utente viene buttato fuori a ogni refresh anche se ha fatto
il login.

E va detto chiaramente quando serve: **una rotta protetta lato client non è sicurezza**. Nasconde
l'interfaccia, non i dati. L'autorizzazione vera la fa il server su ogni richiesta.

---

## useRoutes

Le rotte come array di oggetti invece che come JSX:

```jsx
const elemento = useRoutes([
  { path: '/', element: <SharedLayout />, children: [
      { index: true, element: <Home /> },
      { path: 'about', element: <About /> },
      { path: '*', element: <ErrorPage /> },
  ]},
]);
return elemento;
```

Serve quando le rotte vanno generate dinamicamente — filtrate per permessi, caricate da una
configurazione — perché un array si manipola con `filter` e `map`, cosa impossibile con il JSX.
Nei casi normali la forma JSX è più leggibile.

---

## Il 404 al refresh dopo il deploy

Sintomo classico: in locale tutto funziona, online l'applicazione va, ma se ricarichi la pagina su
`/about` il server risponde 404. Il motivo: il server cerca davvero un file `/about`, che non
esiste. Serve una regola che rimandi tutte le richieste a `index.html`.

```
# Netlify: file public/_redirects
/*    /index.html   200
```

Su Vercel funziona già; su Apache serve una `.htaccess` con una RewriteRule, su nginx un
`try_files $uri /index.html`. È l'unico modo: non è un problema di React.

---

## Differenze v5 → v6

Utili per leggere codice esistente o materiale didattico più vecchio.

| v5 | v6 |
|---|---|
| `<Switch>` | `<Routes>` |
| `component={Home}` o `render={...}` | `element={<Home />}` |
| `exact path="/"` | non serve: il match è esatto per default |
| `useHistory()` → `history.push()` | `useNavigate()` → `navigate()` |
| `<Redirect to="/x" />` | `<Navigate to="/x" replace />` |
| rotte annidate dichiarate dentro il componente | `<Outlet />` |

**Nota sulla v7:** react-router 7 unifica il pacchetto con Remix e introduce i data router
(`createBrowserRouter`, `loader`, `action`) per caricare i dati prima di renderizzare la pagina. Le
API mostrate qui continuano a funzionare. Verifica la versione nel `package.json` prima di
proporre `createBrowserRouter` a chi sta usando la forma dichiarativa.
