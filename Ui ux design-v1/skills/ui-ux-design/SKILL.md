---
name: ui-ux-design
description: Progetta e costruisce interfacce utente applicando regole reali di UI e UX - obiettivo di business, utente e personas, customer journey, architettura delle informazioni, priorità delle funzionalità, design system e design token, gerarchia visiva, layout responsive, colore e tipografia, stati e microinterazioni, accessibilità WCAG (A, AA, AAA), semantica HTML, ARIA, navigazione da tastiera e form. Usa questa skill ogni volta che si progetta, si scrive o si valuta qualunque interfaccia - "fammi una landing page", "crea un form di registrazione", "aggiungi una dashboard", "questa schermata non mi convince", "come organizzo il menu", "che colori uso", "rendilo responsive", "guarda questa UI" - sia per costruire da zero sia per rivedere interfacce esistenti, indipendentemente dallo stack (HTML/CSS, React, Vue, mobile, desktop) e anche quando l'utente chiede solo "una pagina" o "una schermata" senza nominare UI, UX, design o accessibilità.
---

# UI/UX — progettare e costruire interfacce

Questa skill serve a **costruire interfacce**, non solo a commentarle. La review è una delle
modalità, non l'unica.

Il principio che tiene insieme tutto: **UI è come appare, UX è come funziona per una persona reale
in una situazione reale.** La UI è un sottoinsieme della UX. Un'interfaccia bellissima che non fa
concludere l'azione è un lavoro fallito.

---

## Passo 0 — Capire prima di disegnare (sempre)

Non produrre mai un'interfaccia senza sapere per chi è e a cosa serve. Se mancano queste
informazioni, ricavale dal contesto o **dichiara l'assunzione invece di inventarla in silenzio**.

Le quattro domande minime:

1. **Qual è l'obiettivo?** Non "cosa mi hai chiesto di costruire", ma il risultato che deve
   produrre: vendere, farsi contattare, far completare un'iscrizione, far trovare un dato.
2. **Chi è l'utente e in che condizioni arriva?** Da mobile o desktop? Di fretta o rilassato?
   Esperto o digiuno di tecnologia? Stressato (portale della pubblica amministrazione) o euforico
   (prenotazione di una vacanza)? Il livello di stress cambia quanta pazienza avrà.
3. **Qual è l'azione principale della schermata?** Ce n'è una sola. Tutto il resto è secondario e
   deve pesare visivamente di meno.
4. **Come si misura se ha funzionato?** Click, invii, conversioni, tempo sul task. Se non è
   misurabile, non è un obiettivo: è un desiderio.

Se l'utente ti ha dato una lista della spesa di funzionalità, **non implementarle tutte allo stesso
livello**: ordinale per importanza (quanti utenti la usano, è in linea con l'obiettivo, se la
togliessi il prodotto funzionerebbe ancora?) e per fattibilità. Dillo esplicitamente.
Il dettaglio del metodo è in `references/ux-discovery.md`.

**Non chiedere un brief completo per un compito piccolo.** Per "aggiungi un pulsante di export" basta
capire chi lo userà e dove. Il Passo 0 è proporzionale alla dimensione della richiesta: a volte sono
dieci secondi di ragionamento, non tre domande all'utente.

---

## Il metodo: Think → Setup → Design → Iterate

**Think.** Elenca le sezioni o gli schermi prima di scrivere una riga. Decidi il flusso: da dove
arriva l'utente, cosa deve fare, dove finisce. Un flusso è un insieme di passaggi che permette di
completare qualcosa (registrarsi, pagare, cercare): progettalo intero, non schermata per schermata.

**Setup.** Definisci le fondamenta prima del primo componente: token di colore, scala tipografica,
scala di spaziatura, raggi, ombre, breakpoint, convenzioni di nomi. Cinque minuti qui risparmiano
ore dopo, e rendono il risultato modificabile invece che da rifare. Vedi
`references/design-system.md`.

**Design.** Costruisci dal piccolo al grande: token → elementi atomici (input, pulsante, icona) →
molecole (campo con label ed errore, card) → organismi (header, sezione, form completo) → pagina.
Non partire dalla pagina e ritagliare pezzi: produce componenti che non si riusano.

**Iterate.** Le modifiche sono la norma, non l'imprevisto. Scrivi codice che si cambia in un punto
solo: se per cambiare il colore primario devi toccare 40 file, hai sbagliato il Setup.

---

## Regole non negoziabili

Queste valgono su qualunque stack. Applicale senza che l'utente le chieda.

**1. Semantica prima dell'estetica.** Scegli l'elemento per ciò che significa, poi stilizzalo.
`<a>` porta a un'altra pagina, `<button>` esegue un'azione: mai `<a href="#" onclick=...>`. Un solo
`h1` per pagina, gerarchia dei titoli senza salti (mai da h2 a h4). Con CSS puoi far sembrare
qualunque cosa qualunque altra cosa; la struttura sottostante è ciò che leggono screen reader,
motori di ricerca e strumenti di test.

**2. Il colore non è mai l'unico indicatore.** Un errore in rosso e basta non esiste per chi ha
daltonismo (circa 1 persona su 20). Affianca sempre icona + testo esplicito. Stesso discorso per gli
stati attivi, i link (sottolineatura oltre al colore) e i grafici.

**3. Contrasto verificato, non "a occhio".** Testo normale 4.5:1, testo grande 3:1 (soglie AA);
7:1 e 4.5:1 per AAA. Vale anche per testo su immagine e per icone informative. Se non sei certo del
rapporto, calcolalo o scegli valori palesemente sicuri.

**4. Una sola azione primaria per schermata.** Un pulsante pieno e colorato, gli altri secondari o
testuali. Se tutto urla, l'utente non sa dove guardare. E non colorare di rosso "Annulla": crea
tensione e fa sembrare enorme una decisione normale.

**5. Priorità all'informazione che serve per decidere.** Se l'utente non può agire senza il prezzo,
il prezzo va in alto e grande. Non fargli fare calcoli ("costruita nel 2016" → "5 anni"), non
nascondere il dato decisivo in fondo a una scheda tecnica.

**6. Tutto è raggiungibile da tastiera.** Ordine di tab logico, focus **sempre visibile** (non
rimuovere mai l'outline senza sostituirla), Esc chiude i modali, il focus si sposta dentro il modale
e torna dove stava alla chiusura. Non usare `disabled` sui pulsanti di invio: esce dal tab order e
l'utente non lo trova più — lascialo attivo e spiega cosa manca.

**7. Ogni campo ha una label vera.** Il placeholder non è una label: sparisce appena scrivi, ha
contrasto basso e spesso viene scambiato per un valore già inserito. Usalo solo per il formato
(`GG/MM/AAAA`), oppure non usarlo affatto. Gli errori vanno sul campo specifico, con testo che dice
cosa fare, collegati con `aria-describedby`.

**8. Ogni elemento interattivo ha i suoi stati.** default, hover, focus, active, disabled, loading,
errore. E ogni lista ha i suoi: vuota, in caricamento, in errore, popolata, troppo lunga. Uno stato
non progettato è un bug che arriverà in produzione.

**9. Spaziatura su scala, non a caso.** Usa una scala (4/8/12/16/24/32/48/64) e raggruppa per
prossimità: poco spazio tra elementi della stessa categoria, molto più spazio tra categorie diverse.
La spaziatura comunica cosa sta insieme meglio di qualunque bordo.

**10. Responsive per davvero.** Progetta per la risoluzione più usata dai tuoi utenti, non per il tuo
monitor. Contenitori flessibili invece di larghezze fisse, target touch di almeno 44×44 px ben
distanziati (chi ha tremori o Parkinson non colpisce bersagli piccoli), niente contenuto tagliato a
320px di larghezza né a 200% di zoom.

**11. Coerenza sopra la creatività.** Un solo font (due se serve, mai più di tre), icone dello stesso
set, stessi raggi e stesse ombre ovunque. L'incoerenza fa sembrare il prodotto un errore, anche
quando è intenzionale.

**12. Niente autoplay, niente lampeggi.** L'audio e il video partono su richiesta. Animazioni brevi e
sobrie, e rispetta `prefers-reduced-motion`.

---

## Quando consultare i reference

Leggi il file, non andare a memoria: contengono le soglie numeriche e gli esempi concreti.

| File | Quando |
| --- | --- |
| `references/ux-discovery.md` | Progetto nuovo, riorganizzazione di un sito, menu e categorie da nominare, decidere quali funzionalità fare, capire il target, definire metriche |
| `references/design-system.md` | Prima di scrivere il primo componente: token, scale, naming, atomic design, responsive, stati, microinterazioni |
| `references/accessibility.md` | Sempre che ci sia markup: semantica, ARIA, form, colore, immagini, media, tastiera, livelli A/AA/AAA, strumenti di test |
| `references/ui-patterns.md` | Decisioni concrete ricorrenti: dove va il prezzo, dropdown o opzioni visibili, modale o bottom sheet, quanti step nel checkout, icone sì o no |
| `references/review-checklist.md` | Prima di consegnare, o quando ti chiedono di valutare una UI esistente |

---

## Formato di output

**Quando costruisci:** codice completo e funzionante, non pseudocodice. Token definiti in cima (CSS
custom properties, oggetto theme, o quello che lo stack usa). Markup semantico con gli attributi di
accessibilità già dentro — non aggiunti dopo come toppa. Poi, sotto il codice, **3-6 righe** che
spiegano le decisioni non ovvie: perché quell'ordine di informazioni, perché quel componente, cosa
resta da validare con utenti veri. Niente sermoni.

**Quando rivedi:** parti da ciò che funziona (esiste sempre), poi i problemi ordinati per impatto
sull'utente, non per facilità di correzione. Per ognuno: cosa non va, **perché** è un problema per
una persona specifica, come si corregge. Se una scelta dipende dall'obiettivo, dillo: "dipende" è una
risposta legittima quando è motivata, non quando è un modo per non prendere posizione.

**Segnala i compromessi.** Un'icona unica per ogni categoria è più memorabile ma costa giorni di
lavoro: è giusto dire che una soluzione è troppo costosa per il contesto.

---

## Errori tipici da non fare

- Costruire "bello da vedere" ignorando cosa deve fare l'utente. Sono due mestieri diversi e questo
  è quello che conta.
- Dare per scontato che l'utente sia come te. Tu sai cos'è un browser, cos'è un'app, come si scarica.
  Lui no. Tu usi il desktop tutto il giorno, lui è su un telefono in metropolitana.
- Aggiungere l'accessibilità alla fine. Va decisa nella struttura: dopo diventa una riscrittura.
- Mettere tutto sullo stesso piano perché "il cliente lo ha chiesto". Se tutto è prioritario, niente
  lo è.
- Ottimizzare solo il percorso felice. Vanno progettati anche gli errori, i rimborsi, la
  cancellazione dell'account e la disiscrizione: anche chi se ne va deve avere un'esperienza
  decente, perché lo racconterà agli altri.
- Copiare un pattern perché lo fa un'azienda grande. Anche le grandi sbagliano; copia il
  ragionamento, non il risultato.
