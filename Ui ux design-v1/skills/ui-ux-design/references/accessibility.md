# Accessibilità — WCAG, semantica, ARIA

Progettare per l'accessibilità non serve solo a chi ha una disabilità: rende il prodotto migliore per
tutti. I sottotitoli servono ai sordi, ma anche a chi non parla bene la lingua e a chi guarda senza
audio. Il linguaggio semplice aiuta chi ha una disabilità intellettiva, ma anche chi ha un QI
altissimo, perché elaborare il linguaggio ha comunque un costo cognitivo.

In molti paesi, Unione Europea compresa, è anche un **obbligo di legge**.

Indice:
1. WCAG: i principi POUR e i livelli
2. Le disabilità e cosa implicano
3. HTML semantico
4. Testo e titoli
5. Link e pulsanti
6. Colore e contrasto
7. Immagini
8. Form
9. ARIA
10. Tastiera e focus
11. Audio e video
12. Strumenti di test

---

## 1. WCAG: i principi POUR e i livelli

Le **Web Content Accessibility Guidelines** del W3C definiscono cosa si considera accessibile.
Quattro principi:

| Principio | Domanda da farsi |
| --- | --- |
| **Perceivable** (percepibile) | Le persone riescono a percepire e capire cosa c'è? |
| **Operable** (operabile) | Si può usare in circostanze diverse, per esempio solo da tastiera? |
| **Understandable** (comprensibile) | È tutto chiaro e prevedibile? |
| **Robust** (robusto) | Regge input e situazioni fuori dal percorso felice senza rompersi? |

**I tre livelli di conformità:**

- **A** — minimo. Non è un traguardo: è il pavimento. Si può fare meglio praticamente sempre.
- **AA** — **lo standard di fatto, ed è il default operativo di questa skill.** Buon equilibrio tra
  accessibilità e libertà creativa, sostenibile anche per organizzazioni piccole.
- **AAA** — il livello più alto. Ha senso per enti pubblici, servizi essenziali, sanità,
  organizzazioni che lavorano con persone con disabilità. Ha un costo reale: molti siti governativi
  hanno un design austero proprio perché al livello AAA non ci si può permettere quasi niente di
  decorativo.

Come cambia concretamente la scala, con l'esempio di un video sul sito:
- **A**: didascalie (captions) sul video preregistrato.
- **AA**: in più, **audio description** — la descrizione di cosa succede, non solo di cosa viene
  detto.
- **AAA**: in più, una versione in **lingua dei segni**.

**Regola operativa:** costruisci sempre almeno AA. Quando il contesto è pubblico, sanitario o
rivolto a persone con disabilità, alza a AAA e dichiaralo. Quando qualcosa costerebbe troppo,
segnala il compromesso invece di ignorarlo in silenzio.

---

## 2. Le disabilità e cosa implicano

### Vista
Circa il 3% della popolazione ha una perdita della vista significativa per la vita quotidiana.

- **Ciechi totali** — screen reader (JAWS, NVDA, VoiceOver, TalkBack) o display braille. Su mobile
  gli screen reader sono integrati nel sistema, quindi molto diffusi.
- **Ipovedenti** — zoom del browser, ingranditori di sistema, oppure semplicemente la faccia
  vicinissima allo schermo. Conseguenza di design: chi zooma molto **vede solo una piccola porzione
  di schermo alla volta** e non coglie mai l'insieme. Il layout deve reggere lo zoom.
- **Daltonismo** — circa **1 persona su 20**, più frequente negli uomini. La combinazione più comune
  è rosso/verde. È una scala: c'è chi non distingue affatto e chi distingue se i colori sono ben
  separati. Conseguenza: **mai il colore come unico indicatore**.

### Udito
Sordità totale, udito ridotto, acufene. Riguarda essenzialmente i contenuti audio e video: si
risolve con didascalie e trascrizioni.

### Motorie
Difficoltà nel controllo fine dei movimenti. Lo spettro è ampio: chi naviga solo da tastiera, chi
usa una mano sola, chi usa un dispositivo dedicato, chi controlla con la bocca o con la voce. Caso
frequentissimo e meno estremo: il Parkinson, che rende difficile portare il puntatore dove si vuole
e colpire bersagli piccoli.

Conseguenze di design:
- link e pulsanti troppo ravvicinati diventano impossibili da colpire;
- elenchi enormi di link sono frustranti da tastiera (si tabula all'infinito) → servono gli **skip
  link**;
- target sufficientemente grandi e distanziati.

### Cognitive
Disabilità intellettive, problemi di memoria, dislessia, discalculia, neurodiversità (autismo,
ADHD). In molte culture la neurodiversità non è considerata una disabilità ma un modo diverso di
essere: il punto non è etichettare, è che il prodotto funzioni per tutti i neurotipi.

Cosa fare concretamente:
- **Linguaggio semplice** — "tutto il più semplice possibile, ma non più semplice".
- **Interlinea almeno 1.5**.
- **Chunking**: spezzare le informazioni in blocchi gestibili.
- **Call to action evidente**: deve essere ovvio cosa fare.
- **Navigazione facile**, con breadcrumb dove la struttura ha livelli.
- **Look and feel coerente** in tutto il prodotto: la sorpresa è un costo cognitivo.

---

## 3. HTML semantico

"Semantico" significa che il tag dice **cosa** è quel pezzo di pagina, non come va disegnato.
Incapsulare tutto in `<div>` e `<span>` toglie ogni significato: il software che elabora la pagina
non sa cosa sia un div.

| Tag | Significato |
| --- | --- |
| `<header>` / `<footer>` | intestazione e piè di pagina |
| `<nav>` | navigazione |
| `<main>` | contenuto principale (uno solo per pagina) |
| `<article>` | contenuto autonomo e compiuto |
| `<section>` | sezione tematica, normalmente con un titolo |
| `<aside>` | contenuto complementare, sidebar |
| `<figure>` / `<figcaption>` | immagini, grafici e diagrammi con didascalia |

```html
<header>Il mio sito</header>
<nav>
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/chi-siamo">Chi siamo</a></li>
  </ul>
</nav>
<main>
  <article>
    <h1>I miei posti preferiti</h1>
    <p>Questi sono i posti che preferisco visitare in vacanza.</p>
  </article>
  <aside>
    <h2>Articoli correlati</h2>
  </aside>
</main>
<footer>Grazie della visita.</footer>
```

**Perché conta:** un utente vedente che vuole saltare il menu semplicemente scrolla. Chi usa uno
screen reader ottiene lo stesso comportamento **gratis** se i tag dicono com'è fatta la pagina.

Vale anche fuori dal web: nei framework nativi significa usare i componenti standard della
piattaforma e valorizzare i loro attributi di accessibilità, invece di ridisegnare tutto da zero con
contenitori generici.

---

## 4. Testo e titoli

Con i CSS puoi far sembrare un `h1` minuscolo e un `h6` gigante: **il tag si sceglie per il
significato, non per l'aspetto.**

Regole:
- **un solo `<h1>` per pagina**, ed è il titolo di ciò di cui parla tutto il resto;
- sotto, `<h2>` per gli argomenti principali, `<h3>`/`<h4>` per le sottosezioni;
- **gerarchia senza salti**: mai da h2 a h4.

Perché i salti sono un problema concreto: chi naviga per titoli sente "intestazione livello 4" subito
dopo un livello 2, pensa di essersi perso un h3 e torna indietro a cercarlo. Ma non è mai esistito.

Nelle pagine articolo: l'`h1` è il titolo dell'articolo; sidebar e footer, se hanno un titolo, sono
`h2`; le sottosezioni scendono a h3/h4.

Layout del testo: contrasto forte, corpo adeguato, `<p>` per i paragrafi (mai `<br>` a coppie),
interlinea ampia, sans serif per il corpo del testo.

---

## 5. Link e pulsanti

| Tag | Quando usarlo | Quando NON usarlo |
| --- | --- | --- |
| `<a>` | portare a un'altra pagina o ancora | con URL finti e handler onClick |
| `<button>` | eseguire un'azione (inviare, aprire, cambiare qualcosa) | come semplice link |

L'anti-pattern da non scrivere mai:

```html
<!-- SBAGLIATO: un link con URL finto dirottato su un'azione -->
<a href="#" onclick="doSomething()">Clicca qui</a>

<!-- CORRETTO -->
<button type="button" onclick="doSomething()">Salva le modifiche</button>
```

Con i CSS puoi far sembrare un link un pulsante e viceversa: **prima la semantica, poi l'estetica.**

**Indica i link con più del colore.** La sottolineatura è il secondo indicatore, quello che funziona
per chi non distingue i colori. Stesso discorso per i link già visitati: lo screen reader annuncia
"visited link", quindi distinguerli anche visivamente dà a chi vede la stessa informazione.

**Testo descrittivo, mai "clicca qui".** Lo screen reader annuncia "link" seguito dal testo: dieci
link che dicono tutti "clicca qui" o "leggi di più" non dicono nulla su dove portano. Identico
problema per i motori di ricerca.

```html
<!-- ✗ --> <a href="/report-2024.pdf">Leggi di più</a>
<!-- ✓ --> <a href="/report-2024.pdf">Leggi il report annuale 2024 (PDF, 2 MB)</a>
```

---

## 6. Colore e contrasto

**Rapporto di contrasto** = rapporto tra la luminanza del testo e quella dello sfondo.

| | AA | AAA |
| --- | --- | --- |
| Testo normale | **4.5:1** | 7:1 |
| Testo grande (≥ 24px, o ≥ 18.66px bold) | **3:1** | 4.5:1 |
| Componenti UI e grafica informativa (bordi input, icone) | **3:1** | — |

Il testo grande può permettersi meno contrasto perché è già più facile da leggere.

Il caso difficile non è il nero su bianco: è il **testo sopra un'immagine**, dove il contrasto cambia
da zona a zona. Soluzioni: overlay o sfumatura sotto il testo, oppure un contenitore pieno. Una
sfumatura scura con testo sempre bianco funziona con qualsiasi foto; un riquadro colorato sotto una
foto arbitraria stona e costringe a cambiare colore del testo caso per caso.

**Il colore non deve mai essere l'unico indicatore.**

L'esempio più chiaro è il messaggio di errore. Versione sbagliata: testo solo rosso. Per chi ha
daltonismo rosso/verde quel testo non comunica "errore". Versione corretta: icona di attenzione +
testo alternativo dell'icona + testo esplicito del messaggio. Così l'informazione arriva da **quattro
canali** (colore, icona, alt dell'icona, testo): se uno salta, restano gli altri.

Nota culturale: non è scontato che il rosso significhi errore. È una convenzione che varia.

Lo stesso vale per grafici (usa pattern o etichette dirette oltre al colore), stati attivi, campi
obbligatori, disponibilità di un prodotto.

---

## 7. Immagini

L'attributo `alt` è la descrizione testuale che lo screen reader legge al posto dell'immagine.

**Immagine informativa** — l'alt descrive il contenuto e la sua funzione:
```html
<img src="cavallo.jpg" alt="Donna con un vestito bianco accanto a un cavallo bianco, in un campo.">
```

**Immagine decorativa** — l'alt va **vuoto, ma presente**:
```html
<img src="icona-busta.svg" alt="">
```

**Questa è la distinzione più importante della lezione.** Se ometti l'attributo, lo screen reader
annuncia "immagine non etichettata" e l'utente si chiede cosa si stia perdendo. Con `alt=""` sa che è
decorativa e la ignora in silenzio.

Caso tipico: un pulsante "Iscriviti alla newsletter" con accanto una piccola icona di busta. L'icona
non aggiunge niente al testo, quindi descriverla è solo rumore → `alt=""`.

**Evita il testo dentro le immagini.** Gli screen reader non lo leggono, e gli strumenti che
modificano lo stile del testo (contrasto, dimensione) non possono agire sui pixel. Se è
indispensabile, deve comunque rispettare i rapporti di contrasto.

---

## 8. Form

È la parte dove si fanno più danni.

### Label
Ogni `<input>` ha una `<label>` collegata via `for` → `id`:

```html
<label for="user-id">Nome utente</label>
<input type="text" id="user-id" name="userId">
```

### I placeholder non sostituiscono le label
Il placeholder sparisce appena scrivi, ha contrasto basso e **viene spesso scambiato per un valore
già inserito o per un autocompletamento**: gli utenti saltano il campo credendolo compilato.

Usalo solo per aggiungere informazione che la label non dà — il **formato**:
```html
<label for="dob">Data di nascita</label>
<input type="text" id="dob" placeholder="GG/MM/AAAA">
```
Mettere "Data di nascita" nel placeholder è duplicazione inutile. Nel dubbio, non usarlo affatto.

### Non disabilitare il pulsante di invio
`disabled` toglie l'elemento dal tab order: chi naviga da tastiera tabula tutto il form, arriva in
fondo e **il pulsante non c'è**. Lascialo attivo, lascia che venga premuto, e poi spiega cosa manca.

Se lo vuoi far *sembrare* non disponibile: stile CSS + `aria-disabled="true"`, che comunica allo
screen reader la stessa informazione che riceve chi vede, senza uscire dal tab order.

### Errori
- porta l'utente alla fonte dell'errore (focus sul primo campo sbagliato);
- di' **quale campo specifico** è sbagliato. L'anti-esempio classico è "Compila tutti i campi
  obbligatori": alcuni sono opzionali, e non sai quale hai sbagliato;
- di' **cosa fare**, non solo cosa è andato storto;
- collega il messaggio al campo con `aria-describedby`.

```html
<label for="email">Indirizzo email</label>
<input type="email" id="email" aria-describedby="email-err" aria-invalid="true">
<p id="email-err" role="alert">
  <svg aria-hidden="true">…</svg>
  Questo indirizzo è già registrato: accedi oppure usane un altro.
</p>
```

### Altre regole
- `type` corretto (`email`, `tel`, `number`, `date`): cambia la tastiera su mobile e attiva la
  validazione nativa.
- `autocomplete` sui campi anagrafici: meno digitazione per tutti, molta meno per chi ha difficoltà
  motorie.
- Raggruppa i gruppi di radio/checkbox in `<fieldset>` con `<legend>`.
- Per argomenti delicati (per esempio il genere) offri le opzioni reali: "Altro" e "Preferisco non
  dirlo" non sono la stessa cosa, né per la persona né per le statistiche.
- Form lunghi: dividili in passaggi con un indicatore di avanzamento, e **permetti di tornare
  indietro**.

---

## 9. ARIA

**WAI-ARIA** è l'insieme di attributi del W3C nato quando abbiamo iniziato a costruire nel browser
applicazioni complesse in stile desktop, per le quali l'HTML non aveva modo di trasmettere il
significato alle tecnologie assistive.

**La regola d'oro: ARIA non sostituisce l'HTML semantico.** L'HTML semantico è sempre la prima
scelta. ARIA si usa solo quando il significato non è esprimibile con l'HTML puro. Un ARIA sbagliato
è peggio di nessun ARIA.

### `role`
Dice che tipo di elemento è, quando stai simulando un elemento nativo con altri tag:
- `role="navigation"` su un menu custom senza `<nav>`;
- `role="checkbox"` su una checkbox stilizzata con `<span>` annidati;
- `role="tablist"` / `role="tab"` / `role="tabpanel"` per le tab, che in HTML non esistono come tag.

**Evita la ridondanza:** se usi `<nav>` non aggiungere `role="navigation"`, è già implicito.

### Gli attributi più usati

| Attributo | A cosa serve |
| --- | --- |
| `aria-label` | Dà un nome a un elemento che non ha testo. Caso classico: l'icona hamburger → `aria-label="Menu"` |
| `aria-labelledby` | Collega l'elemento a un altro che lo **nomina** (letto per primo: dice cos'è) |
| `aria-describedby` | Collega l'elemento a una **descrizione** aggiuntiva (letta dopo: dettagli, errori, formato) |
| `aria-hidden="true"` | Toglie dall'albero di accessibilità ciò che è puramente decorativo (icone accanto a un testo che dice già tutto) |
| `aria-expanded` | Stato aperto/chiuso di menu, accordion, dropdown |
| `aria-current="page"` | La voce di navigazione corrispondente alla pagina corrente |
| `aria-checked` / `aria-selected` | Stato di controlli simulati |
| `aria-disabled` | "Non disponibile" senza uscire dal tab order |
| `aria-live` | Annuncia i cambiamenti dinamici della pagina |

### Live region
Nelle applicazioni moderne i contenuti cambiano senza ricaricare: se sei in fondo alla pagina e
qualcosa cambia in cima, senza `aria-live` non lo sapresti mai.

- `aria-live="polite"` — annuncia quando lo screen reader ha finito quello che sta leggendo. È il
  default sensato (risultati di ricerca aggiornati, conferma di salvataggio).
- `aria-live="assertive"` — interrompe subito. Solo per cose urgenti.
- `role="alert"` — scorciatoia per il messaggio letto immediatamente (errori, avvisi).

---

## 10. Tastiera e focus

**Tab index:** l'ordine in cui gli elementi interattivi ricevono il focus.

| Valore | Effetto |
| --- | --- |
| numero positivo | Forza una posizione specifica nell'ordine. **Da evitare quasi sempre**: rompe il flusso naturale. |
| `0` | Inserisce l'elemento nel flusso normale della pagina. È quello che vuoi quasi sempre. |
| `-1` | Non raggiungibile con Tab, ma focalizzabile via JavaScript (utile per spostare il focus su un messaggio di errore o dentro un modale). |

Regole:
- **Il focus deve essere sempre visibile.** Non rimuovere `outline` senza mettere qualcosa di almeno
  altrettanto evidente. `:focus-visible` è il modo giusto di distinguere mouse e tastiera.
- **Ordine di tab = ordine logico del contenuto.** Se l'ordine visivo e quello del DOM divergono,
  l'esperienza si rompe.
- **Skip link**: link nascosto all'inizio del documento che diventa visibile quando riceve il focus e
  porta direttamente al contenuto principale. Chi vede non lo incontra mai; chi naviga da tastiera lo
  trova al primo Tab e salta header e menu.

```html
<a class="skip-link" href="#main">Salta al contenuto principale</a>
…
<main id="main" tabindex="-1">…</main>
```
```css
.skip-link { position:absolute; left:-9999px; }
.skip-link:focus { left:1rem; top:1rem; position:fixed; z-index:999; }
```

- **Modali**: il focus entra dentro, resta intrappolato finché è aperto, Esc chiude, alla chiusura
  torna esattamente dove stava.
- **L'ordine semantico può battere quello visivo, e questa è una leva.** Se la call to action
  principale è "Prenota online", mettila per prima nel markup anche se visivamente la sposti in
  fondo al menu: chi usa uno screen reader incontra per prima la cosa più importante.

---

## 11. Audio e video

**In produzione:**
- niente immagini lampeggianti: causano problemi reali a chi ha epilessia e disturbano tutti;
- registra le audio description **mentre giri il video**: rifarle dopo è molto difficile;
- il testo dentro il video deve essere grande e ad alto contrasto.

**Nel player:**
- usa i tag nativi `<video>` e `<audio>`, non player proprietari: una tecnologia assistiva può
  ispezionare l'HTML e dare all'utente il controllo, con un player chiuso non ci arriva;
- **niente autoplay**;
- se implementi controlli personalizzati, testali da tastiera e con uno screen reader.

**Tre cose diverse, da non confondere:**
- **Trascrizione** — la versione scritta di tutto il contenuto parlato, separata dal video.
- **Captions (didascalie)** — sottotitoli sincronizzati dentro il video, che descrivono sia ciò che
  viene detto sia ciò che accade.
- **Subtitles (sottotitoli)** — tecnicamente la traduzione in lingua straniera, senza le descrizioni
  dei suoni.

La trascrizione conviene a tutti: si legge sui mezzi, senza cuffie, più velocemente di quanto si
ascolti — e rende il contenuto **ricercabile**.

---

## 12. Strumenti di test

Gli strumenti automatici trovano una parte dei problemi, mai tutti. Il test vero è provare il flusso
da tastiera e con uno screen reader.

| Strumento | Cos'è | Note |
| --- | --- | --- |
| **Lighthouse** | Audit integrato in Chrome (DevTools), punteggio 0-100 su performance, accessibilità, best practice, SEO | Panoramica rapida; cliccando l'errore ti porta all'elemento colpevole. Meno dettagliato di WAVE |
| **WAVE** (wave.webaim.org) | Annota la pagina con icone su ogni problema | Più dettagliato: errori, avvisi ("testo sospetto" tipo "Read more"), struttura dei titoli come la vedrebbe uno screen reader, uso di ARIA, contrast checker integrato |
| **WebAIM Contrast Checker** | Calcolo del rapporto di contrasto tra due colori | Esistono anche estensioni che lo calcolano direttamente sulla pagina |
| **W3C Validator** (validator.w3.org) | Valida l'HTML | Non misura l'accessibilità, ma se il markup è rotto browser e screen reader lo interpretano in modo imprevedibile |
| **axe DevTools** | Estensione di analisi automatica | Buona copertura, pochi falsi positivi |

**Screen reader:**
- **VoiceOver** (macOS/iOS, integrato) — Cmd+F5 per attivarlo; modificatori Ctrl+Option + frecce;
  Ctrl+Option+Spazio per attivare l'elemento. Mostra un indicatore visivo di dove si trova, utile
  anche per chi testa.
- **NVDA** (Windows, gratuito) e **JAWS** (Windows, a pagamento, storicamente il più diffuso) — in
  JAWS frecce su/giù per riga, `Q` per saltare al main, il numero del livello per saltare ai titoli
  (2 → h2). Entra in "forms mode" quando incontra un modulo.
- **Narrator** (Windows, integrato) — Ctrl+Win+Invio.
- **TalkBack** (Android).

**Il test minimo da fare sempre, anche senza strumenti:**
1. Naviga l'intero flusso **solo con Tab, Invio, Spazio ed Esc**. Riesci a completarlo? Vedi sempre
   dove sei?
2. Zoom al 200%: il contenuto si rompe o si taglia?
3. Larghezza 320px: si può ancora usare?
4. Guarda la pagina in scala di grigi: le informazioni passano ancora?
5. Leggi solo i titoli: si capisce la struttura della pagina?
