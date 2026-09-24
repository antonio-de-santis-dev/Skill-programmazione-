# Design system — token, componenti, responsive

Un design system è l'insieme organizzato di **stili, componenti e regole** che rende un prodotto
coerente e modificabile in un punto solo. Costruirlo all'inizio sembra tempo perso: è l'unica cosa
che rende sostenibile l'iterazione.

**Regola di costruzione: dal più piccolo al più grande.** Token → atomi → molecole → organismi →
schermate. Mai il contrario.

Indice:
1. Token di colore
2. Scala tipografica
3. Spaziatura, raggi, ombre
4. Griglia e breakpoint
5. Naming
6. Atomic design e componenti
7. Stati e varianti
8. Responsive
9. Microinterazioni e motion
10. Documentazione e handover

---

## 1. Token di colore

Non usare colori "sciolti" nel codice. Definisci una palette con **categorie semantiche** e scale di
luminosità, sul modello delle utility CSS diffuse (50 → 900).

Categorie minime da avere sempre:

| Categoria | Uso |
| --- | --- |
| `neutral` | Testi, bordi, sfondi, superfici. È la categoria più usata di tutte. |
| `primary` | Azione principale, identità del brand |
| `secondary` | Azioni di supporto, accenti |
| `success` | Conferme, stati positivi (verde) |
| `warning` | Attenzione, stato intermedio (arancione/giallo) |
| `destructive` | Errori e azioni irreversibili (rosso) |
| `white` / `black` | Come token a sé, per non cercarli ogni volta |

Scala per categoria: almeno `50, 200, 400, 600, 900`; per un sistema grande aggiungi
`100, 300, 500, 700, 800`.

```css
:root {
  --color-neutral-50:  #f8fafc;
  --color-neutral-200: #e2e8f0;
  --color-neutral-400: #94a3b8;
  --color-neutral-600: #475569;
  --color-neutral-900: #0f172a;

  --color-primary-400: #818cf8;
  --color-primary-600: #4f46e5;   /* azione principale */
  --color-primary-900: #312e81;

  --color-destructive-600: #dc2626;
  --color-warning-600:     #d97706;
  --color-success-600:     #16a34a;

  /* alias semantici: il resto del codice usa SOLO questi */
  --bg-surface:    var(--color-neutral-50);
  --text-primary:  var(--color-neutral-900);
  --text-muted:    var(--color-neutral-600);
  --border-subtle: var(--color-neutral-200);
}
```

**Il doppio livello (scala + alias semantico) è ciò che rende possibile il dark mode** e il cambio di
brand: ridefinisci gli alias, non tocchi i componenti.

Accanto a ogni token scrivi **dove si usa** ("sfondo delle superfici", "testo secondario"). Serve a
chi legge il codice dopo di te.

### Psicologia del colore — quando serve scegliere

Usala come punto di partenza ragionato, non come regola assoluta: le associazioni cambiano da
cultura a cultura.

**Caldi** (si notano più dei freddi):
- **Rosso**: passione, energia / pericolo, aggressività. Riservalo alle azioni serie e distruttive
  ("Elimina"). Rosso scuro + grigio + bianco = eleganza, usato dai marchi di lusso.
- **Arancione**: energia, cordialità, appetito. Stato intermedio: allerta lieve. Tipico del food.
- **Giallo**: allegria, attenzione / cautela, pericolo. Molto flessibile: elegante nei toni oro,
  scadente se usato male.

**Freddi** (riservati, rassicuranti, professionali):
- **Verde**: natura, crescita, salute, denaro, conferma. Verde oliva = naturale, verde scuro =
  stabilità.
- **Blu**: autorità, affidabilità, calma. Il blu scuro è il colore di aziende, banche e università.
  Azzurri chiari = calma; blu brillanti = energia.
- **Viola**: creatività, mistero, lusso. Chiari per beauty e self-care, scuri per il premium.

**Neutri**:
- **Bianco**: pulizia, riposo dell'occhio. È il miglior sfondo per leggere. Rischio: freddo,
  impersonale.
- **Nero**: potere, eleganza, formalità. Raramente come sfondo — anche in dark mode si usa un nero
  attenuato (`#111`–`#1a1a1a`), mai `#000` puro, che affatica.
- **Grigio**: professionale. Ottimo per testi secondari e superfici, pessimo come sfondo unico.
- **Beige/marrone**: comfort, stabilità. Il beige chiarissimo è un'ottima alternativa al bianco
  puro.

---

## 2. Scala tipografica

**Massimo tre font, meglio uno solo.** Un sans serif con molti pesi copre l'intera interfaccia; un
secondo font, se serve, va al logo o ai titoli e non deve essere troppo diverso. Più font confondono
l'identità.

Classificazione e uso:
- **Serif** — libri, contesti editoriali e accademici, titoli. Tradizionalmente considerati
  leggibili, ma per alcune condizioni (autismo, dislessia) risultano più difficili.
- **Sans serif** — la scelta migliore per il corpo del testo di siti e app.
- **Display** — solo titoli molto caratterizzati.
- **Handwriting / Monospace** — usi specifici (firme, codice).

Scala consigliata: 6 livelli di heading + 2-3 livelli di paragrafo, ciascuno con i pesi che servono
(regular, semibold, bold).

```css
:root {
  --font-sans: "Inter", system-ui, -apple-system, "Segoe UI", sans-serif;

  --text-h1:   3.815rem;  /* ~61px */
  --text-h2:   3.052rem;
  --text-h3:   2.441rem;
  --text-h4:   1.953rem;
  --text-h5:   1.563rem;
  --text-h6:   1.25rem;
  --text-body: 1rem;      /* 16px base */
  --text-sm:   0.875rem;

  --leading-tight: 1.2;   /* titoli */
  --leading-body:  1.5;   /* minimo raccomandato per il testo corrente */
}
```

**Usa `rem`, non `px`.** Il `rem` è relativo alla dimensione di base del browser: se un utente
ingrandisce il testo nelle impostazioni, i `rem` si adattano e i `px` no. È una questione di
accessibilità, non di gusto.

Altre regole:
- **Interlinea almeno 1.5** per il testo corrente (raccomandazione degli enti per la dislessia).
- **Testo allineato a sinistra** per la lettura. Il centrato va bene per titoli brevi e slide, quasi
  mai per paragrafi.
- **Lunghezza riga 45-75 caratteri** (`max-width: 65ch`): più lunga e l'occhio perde la riga.
- **Spezza in paragrafi** invece di produrre muri di testo (*chunking*). Usa `<p>`, mai `<br>` per
  separare i paragrafi.
- Il maiuscolo si ottiene con `text-transform`, non riscrivendo il testo: gli screen reader leggono
  il contenuto reale, e il testo resta cercabile e traducibile.

---

## 3. Spaziatura, raggi, ombre

**Scala di spaziatura basata su 4px.** Tutti i gap, padding e margini vengono da qui. Niente valori
inventati: `23px` non esiste.

```css
:root {
  --space-1: 0.25rem;  /*  4px */
  --space-2: 0.5rem;   /*  8px */
  --space-3: 0.75rem;  /* 12px */
  --space-4: 1rem;     /* 16px */
  --space-6: 1.5rem;   /* 24px */
  --space-8: 2rem;     /* 32px */
  --space-12: 3rem;    /* 48px */
  --space-16: 4rem;    /* 64px */

  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 16px;
  --radius-full: 9999px;

  --shadow-xs: 0 1px 2px rgb(0 0 0 / .05);
  --shadow-sm: 0 1px 3px rgb(0 0 0 / .1);
  --shadow-md: 0 4px 12px rgb(0 0 0 / .1);
  --shadow-lg: 0 12px 32px rgb(0 0 0 / .14);
  --shadow-xl: 0 24px 60px rgb(0 0 0 / .18);
}
```

**La prossimità è il raggruppamento.** Spazio piccolo tra elementi della stessa categoria, spazio
2-3 volte più grande tra categorie diverse. È il modo più economico di rendere leggibile una pagina:
si capisce cosa sta insieme senza leggere.

**Lo spazio vuoto non è spazio sprecato.** Un pulsante con l'icona che riempie tutto il cerchio è
meno leggibile di uno con spazio attorno; elementi troppo vicini sembrano affollati e confusi.

Attenzione ai padding annidati: **il padding si somma al gap.** Contenitori con padding dentro
contenitori con padding producono distanze visive diverse da quelle previste. Controlla livello per
livello.

---

## 4. Griglia e breakpoint

Griglia di riferimento per dispositivo:

| Dispositivo | Larghezza tipica | Colonne | Margine |
| --- | --- | --- | --- |
| Desktop | 1440 | 12 | 64 |
| Tablet | 768-1024 | 8 | 40 |
| Mobile | 375-430 | 4-5 | 16 |

Il margine laterale della griglia è **lo stesso padding orizzontale** che usi su header, sezioni e
footer: è ciò che fa sembrare tutto allineato.

Breakpoint: scegline pochi (es. 640 / 768 / 1024 / 1280) e basali sul punto in cui il contenuto si
rompe, non sui modelli di telefono. Verifica sempre i due estremi: **320px di larghezza** e **200%
di zoom**.

---

## 5. Naming

I nomi servono a chi legge il codice dopo di te — altri designer, sviluppatori, il te stesso di tra
sei mesi.

**Nomina per ruolo, non per aspetto.**

| ✗ | ✓ |
| --- | --- |
| `Text 12` | `card-title` |
| `Rectangle 4` | `image-thumbnail` |
| `blue-button` | `button-primary` |
| `div-wrapper-2` | `product-card` |

Struttura consigliata: `elemento / variante-o-stato / caratteristica`

```
button/primary/hover
container/card/blur
design-system/heading/h1/bold
design-system/neutral/600
```

Usa convenzioni vicine a quelle del CSS: rendono il passaggio design → codice diretto, e i nomi
restano gli stessi in tutta la catena.

---

## 6. Atomic design e componenti

| Livello | Cosa | Esempi |
| --- | --- | --- |
| **Token** | Valori | colori, dimensioni testo, spaziature, ombre |
| **Atomi** | Elementi indivisibili | icona, checkbox, radio, toggle, input, avatar, badge |
| **Molecole** | Combinazioni di atomi | pulsante con icona, campo con label ed errore, barra di ricerca |
| **Organismi** | Blocchi completi | header, card prodotto, form, tabella, footer |
| **Template/Pagine** | Composizione | landing, dashboard, checkout |

Regole:
- **Un componente per un ruolo.** Se un componente serve a tre cose, sono tre componenti (o uno con
  varianti esplicite).
- **Proprietà invece di duplicati.** Un pulsante con proprietà `variant`, `size`, `icon-left`,
  `icon-right`, `loading` copre decine di casi. Se ti trovi con cinquanta varianti di un pulsante,
  hai usato le copie al posto dei parametri.
- **Contenuto dall'esterno.** Il testo, le icone e le immagini sono parametri, non sono scolpiti nel
  componente.
- **Un contenitore per le immagini.** L'immagine sta dentro un box con dimensione e `object-fit`
  definiti, mai libera: così quando il contenitore cambia dimensione il comportamento è prevedibile
  e non si deforma né rompe il layout.
- **È normale accorgersi a metà che serve un componente in più.** Si itera; non è un fallimento del
  sistema.

---

## 7. Stati e varianti

**Ogni elemento interattivo ha tutti i suoi stati.** Mancarne uno è un bug che arriva in produzione.

Per un controllo: `default`, `hover`, `focus` (obbligatorio e visibile), `active`, `disabled`,
`loading`, `error`, e se serve `selected` / `checked`.

Per una schermata o una lista, i quattro stati sempre da progettare:
- **vuoto**: cosa vede chi non ha ancora dati? Deve dire cosa fare, non solo "nessun risultato".
- **caricamento**: skeleton o indicatore, mai un salto improvviso di layout.
- **errore**: cosa è andato storto e cosa può fare adesso.
- **pieno / troppo pieno**: cosa succede con 500 elementi, con un nome di 80 caratteri, con
  un'immagine mancante.

**Stato attivo visivamente ovvio.** Nella navigazione, l'icona della sezione corrente si distingue
(piena vs vuota, colore, indicatore): si capisce dove si è senza leggere il titolo.

---

## 8. Responsive

Il modello mentale: **la pagina è un insieme di contenitori, disposti in orizzontale e in
verticale**. Ogni contenitore decide la propria dimensione in uno di tre modi:

| Modalità | Significato | CSS |
| --- | --- | --- |
| **Fixed** | dimensione fissa, non cambia | `width: 320px` |
| **Hug** | grande quanto il suo contenuto | `width: fit-content` / default in flex |
| **Fill** | occupa lo spazio disponibile | `flex: 1` / `width: 100%` |

**Quasi tutto il responsive si fa con Fill.** Larghezze fisse solo dove sono davvero necessarie
(icone, avatar, colonne di dimensione nota).

Regole pratiche:
- Contenitori flessibili (`flex`, `grid`) e `gap`, non margini a mano.
- `max-width: 100%` sulle immagini, sempre.
- Contenuto largo (tabelle, codice, diagrammi) dentro un contenitore con `overflow-x: auto`, così la
  pagina non scorre lateralmente.
- **Non trasformare la pagina intera in un unico layout automatico sperando che si adatti da
  desktop a mobile**: le viste molto diverse vanno pensate separatamente. Il componente si adatta;
  la composizione della pagina spesso cambia.
- Su mobile le azioni importanti stanno **in basso**, vicino al pollice: una voce in basso viene
  toccata più di una in alto. È anche il motivo per cui una conferma distruttiva in un bottom sheet
  è più raggiungibile — e lascia visibile il contenuto che si sta per perdere.
- Target touch minimo **44×44 px**, con spazio tra un target e l'altro.

---

## 9. Microinterazioni e motion

Una microinterazione è una piccola animazione di feedback in risposta a un'azione: hover, click,
toggle, invio riuscito. Serve a confermare che il sistema ha ricevuto l'input.

Regole:
- **Tieni le animazioni semplici.** 150-300 ms per feedback immediati, fino a ~500 ms per
  transizioni di layout. Più lungo è il valore, più il movimento è morbido e lento: oltre una certa
  soglia diventa un'attesa.
- Anima proprietà economiche (`transform`, `opacity`), non `width`/`top`.
- **Ogni animazione deve dire qualcosa.** Se non comunica stato, direzione o causa-effetto, toglila.
- Rispetta sempre `prefers-reduced-motion`:

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: .01ms !important;
    transition-duration: .01ms !important;
  }
}
```

- **Mai contenuti lampeggianti.** Causano problemi reali a chi ha epilessia e disturbano tutti.
- Un overlay scuro dietro un modale (scrim) non è decorazione: dice all'utente dove deve guardare.

---

## 10. Documentazione e handover

Un componente senza documentazione viene usato male.

Per ogni componente riusabile scrivi, accanto al codice:
- **cosa fa e quando usarlo** — e soprattutto **quando non usarlo** ("solo per la CTA principale,
  non per azioni comuni");
- le proprietà disponibili e i valori ammessi;
- i requisiti di accessibilità specifici (es. "richiede una `aria-label` se usato solo con icona").

Mantieni i nomi coerenti lungo tutta la catena: token → componente → classe CSS → nome nel codice.
Quando cambia una decisione di design, deve cambiare **in un punto solo**.
