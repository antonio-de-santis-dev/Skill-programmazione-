# Checklist di review

Due usi: prima di consegnare qualcosa che hai costruito, e quando ti viene chiesto di valutare
un'interfaccia esistente.

**Come si consegna una review:** parti da ciò che funziona (esiste sempre, anche nei prodotti
peggiori), poi i problemi **ordinati per impatto sull'utente**, non per facilità di correzione. Per
ognuno: cosa non va → perché è un problema **per una persona specifica** → come si corregge.
Anche le aziende grandi sbagliano: non dare per buono un pattern solo perché è diffuso.

---

## 1. Scopo e priorità

- [ ] Si capisce in 5 secondi di cosa si tratta e cosa si può fare qui?
- [ ] C'è **una** azione primaria evidente, e le secondarie pesano meno?
- [ ] L'informazione necessaria per decidere è visibile senza scorrere né cercare (prezzo,
      disponibilità, durata, requisiti)?
- [ ] C'è qualcosa che chiede all'utente uno sforzo che potremmo fare noi (calcoli, conversioni,
      ricerca di un dato)?
- [ ] Ogni elemento presente serve a qualcosa? Cosa succederebbe togliendolo?

## 2. Struttura e gerarchia

- [ ] Gerarchia visiva a massimo tre livelli, coerente in tutte le schermate?
- [ ] Il raggruppamento è comunicato dalla spaziatura (poco spazio dentro la categoria, molto tra
      categorie)?
- [ ] I nomi di menu, categorie e pulsanti usano le parole dell'utente, non quelle interne?
- [ ] Si capisce sempre dove si è e come si torna indietro?
- [ ] Le pagine di errore, le liste vuote e i risultati zero dicono cosa fare adesso?

## 3. Coerenza

- [ ] Un solo set di icone, stesso stile e stesso peso?
- [ ] Massimo tre font (meglio uno), scala tipografica rispettata?
- [ ] Spaziature dalla scala (4/8/12/16/24/32/48/64), niente valori inventati?
- [ ] Raggi e ombre dai token, non decisi caso per caso?
- [ ] Gli stessi elementi si comportano allo stesso modo ovunque?
- [ ] I colori vengono dai token semantici, non da valori hex sparsi nel codice?

## 4. Stati

- [ ] default, hover, **focus visibile**, active, disabled, loading per ogni elemento interattivo?
- [ ] Stati vuoto / caricamento / errore / pieno per ogni lista e schermata?
- [ ] Cosa succede con un testo lunghissimo, un'immagine mancante, 500 elementi, connessione lenta?
- [ ] Ogni azione dà un feedback immediato (non lascia l'utente a chiedersi se ha funzionato)?

## 5. Accessibilità (dettagli in `accessibility.md`)

- [ ] Markup semantico: `main`, `nav`, `header`, `footer`, titoli gerarchici senza salti, un solo
      `h1`?
- [ ] `<a>` per navigare, `<button>` per agire — nessun `<a href="#" onclick>`?
- [ ] Contrasto verificato: 4.5:1 testo normale, 3:1 testo grande e componenti (AA)?
- [ ] Nessuna informazione veicolata dal solo colore?
- [ ] Ogni immagine ha `alt` — descrittivo se informativa, **vuoto ma presente** se decorativa?
- [ ] Ogni campo ha una label vera; i placeholder non sostituiscono le label?
- [ ] Gli errori dicono quale campo e cosa fare, collegati con `aria-describedby`?
- [ ] Nessun pulsante di invio `disabled` che esce dal tab order?
- [ ] Tutto il flusso si completa con Tab/Invio/Spazio/Esc, focus sempre visibile?
- [ ] Skip link presente; nei modali il focus entra, resta ed esce correttamente?
- [ ] Niente autoplay, niente lampeggi, `prefers-reduced-motion` rispettato?
- [ ] Video e audio hanno didascalie e, se il livello lo richiede, audio description?

## 6. Responsive

- [ ] Funziona a **320px** di larghezza e al **200%** di zoom senza tagli né scroll orizzontale?
- [ ] Target touch ≥ 44×44 px, ben distanziati?
- [ ] Le azioni frequenti su mobile sono raggiungibili col pollice?
- [ ] Le tabelle e i contenuti larghi scorrono dentro il loro contenitore, non nella pagina?
- [ ] Il layout è stato pensato per la vista mobile, non solo rimpicciolito?

## 7. Testi

- [ ] Linguaggio semplice, frasi corte, nessun gergo interno?
- [ ] Testo dei link descrittivo (mai "clicca qui" / "leggi di più")?
- [ ] Interlinea ≥ 1.5, righe da 45-75 caratteri, paragrafi spezzati?
- [ ] I messaggi di errore dicono cosa fare, non solo cosa è andato storto?
- [ ] Le etichette dei pulsanti dicono l'azione ("Crea account"), non "OK" o "Invia"?

## 8. Fiducia e percorsi scomodi

- [ ] Nei processi a più passi l'utente sa a che punto è e può tornare indietro?
- [ ] Le azioni distruttive sono confermate, con l'elenco esatto di ciò che si perde?
- [ ] Si può annullare o correggere dopo aver agito?
- [ ] Disiscrizione, cancellazione account, rimborso e supporto sono facili quanto l'iscrizione?
- [ ] Ci sono dubbi introdotti nel momento sbagliato (upsell o avvisi ansiogeni al checkout)?

## 9. Verifiche rapide da fare davvero

1. Percorri il flusso principale **solo da tastiera**.
2. Guarda la pagina **in scala di grigi**: l'informazione passa ancora?
3. Leggi **solo i titoli**: si capisce la struttura?
4. Porta lo zoom al **200%** e la larghezza a **320px**.
5. Fai provare il flusso a qualcuno che non ha mai visto il prodotto e **non aiutarlo**: guarda dove
   si ferma. Il punto in cui esita è il tuo prossimo lavoro.

---

## Cosa NON è una review

- Un elenco di preferenze personali travestite da regole.
- "Rendilo più moderno" senza dire cosa lo renderebbe tale e perché servirebbe all'utente.
- Riscrivere tutto per uniformarlo al proprio gusto quando il problema era uno solo.
- Trovare solo difetti: se non riesci a dire cosa funziona, non hai capito il prodotto.
