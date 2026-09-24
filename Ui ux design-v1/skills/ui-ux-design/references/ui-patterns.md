# Pattern UI — decisioni concrete, con il motivo

Questi sono i casi che tornano in ogni progetto. Ogni voce dice **cosa fare** e soprattutto
**perché**: il "perché" è ciò che ti permette di decidere nei casi non elencati qui.

Premessa onesta: in diversi casi la risposta giusta è **"dipende dallo scopo e dal pubblico"**. È una
risposta legittima quando è motivata, non quando serve a non prendere posizione.

---

## Principi che generano tutte le decisioni

1. **Familiarità.** L'utente arriva con aspettative costruite altrove. Il carrello sta in alto a
   destra, il logo in alto a sinistra porta alla home, la X chiude. Rompere una convenzione costa
   attenzione: fallo solo se in cambio dai qualcosa.
2. **Ordine naturale di lettura.** Prima si guarda l'oggetto, poi il dettaglio. Mettere un dettaglio
   sopra l'oggetto gli toglie gerarchia.
3. **Prossimità = appartenenza.** Le informazioni che devono essere lette insieme stanno vicine.
4. **Meno fatica mentale possibile.** Se puoi fare un calcolo tu, fallo tu.
5. **Fiducia prima della conversione.** Un utente che si sente in trappola compra meno di uno che si
   sente libero.

---

## Pagine prodotto ed e-commerce

**Prezzo vicino alle varianti.** Se colori o taglie diverse hanno prezzi diversi e il prezzo è
lontano dal selettore, l'utente cambia variante e non si accorge che il prezzo è cambiato. È un
problema di fiducia, non di layout.

**Ordine delle informazioni:** prodotto → prezzo → varianti → azione. Il prezzo non va sopra il
prodotto: perde gerarchia e l'utente non ha ancora nulla da valutare.

**Le varianti di colore non sono la prima cosa da guardare.** Vanno sotto nome e prezzo, o dopo la
taglia: prima l'utente deve considerare il prodotto che ha aperto. E la variante selezionata deve
avere un indicatore **evidente** — un bordo sottilissimo non basta.

**Dropdown o opzioni visibili?** Dipende dalla strategia:
- **Dropdown** nasconde le alternative. Utile se vuoi spingere proprio quell'opzione, e sfrutta la
  pigrizia dell'utente. Più compatto.
- **Opzioni visibili** aiutano l'indeciso a trovare quello che vuole e a sentirsi sicuro della
  scelta. Più veloci: niente menu da aprire.
- Con poche opzioni (2-5), visibili quasi sempre. Con molte, dropdown con ricerca.

**Le immagini di prodotto devono essere coerenti tra loro:** stessa prospettiva, stesso trattamento,
e dimensioni che rispettino le proporzioni reali. Una sedia mostrata più piccola di una tastiera
costringe il cervello a lavorare per niente.

**Card prodotto:** immagine (in un contenitore a proporzioni fisse) → titolo → dato distintivo
(capacità, taglia, autore) → prezzo → azione. Stato hover con elevazione leggera.

**Slideshow orizzontale:** calcola quanti elementi ci stanno davvero
(`larghezza − 2 × margine ÷ (card + gap)`), non metterne tre perché "tre sta bene". Su viste
strette, meno elementi o scroll.

---

## Checkout e processi a più passaggi

**Mostra sempre a che punto è l'utente** con uno stepper o indicatore di avanzamento, e **permetti di
tornare indietro**.

Perché: sa quanti passaggi mancano (il processo sembra più breve), sa di poter correggere un errore,
e si fida. La versione senza ritorno che spinge a concludere sembra migliore "per l'azienda", ma
mette sotto pressione: **la fiducia porta più vendite del senso di trappola.**

**Non introdurre dubbi nel momento della decisione.** Proporre al checkout un'assicurazione contro il
fallimento dell'azienda inserisce una paura che l'utente non aveva, nel punto di massima tensione.
Chiedi solo ciò che serve per concludere.

**Non dare per scontata la conoscenza.** Due tipi di biglietto "standard" e "plus" senza spiegare la
differenza costringono l'utente a spendere tempo ed energie nel momento peggiore.

**Dopo l'acquisto, rassicura:** riepilogo, cosa succede adesso, come contattare qualcuno se qualcosa
va storto, e se possibile una finestra per modificare l'ordine. Costa poco e cementifica la
relazione.

---

## Form e input

**Nome unico o nome + cognome?** Dipende: separarli aiuta l'archiviazione dei dati, un campo unico è
più comodo per chi ha più nomi, usa il secondo nome o il patronimico. Se non ti serve davvero
separarli, non separarli.

**Password e conferma affiancate** permettono di confrontare a colpo d'occhio e accorgersi di un
errore. Meglio ancora: un solo campo con toggle "mostra password".

**Niente pulsante "Reset" nei form brevi di registrazione.** Vuoi che l'utente invii, non che
cancelli tutto per sbaglio. Ha senso solo nei form lunghi compilati ripetutamente per persone
diverse.

**Campi obbligatori e opzionali vanno distinti**, e l'errore deve dire **quale** campo è sbagliato
(vedi `accessibility.md`, sezione Form).

**Argomenti delicati:** offri le opzioni reali. "Altro" e "Preferisco non dirlo" non sono
equivalenti, né per la persona né per i dati che raccogli.

---

## Navigazione e menu

**Icone + testo battono il solo testo.** L'utente riconosce le voci senza leggere e si sente padrone
dell'interfaccia già alla prima visita. Anche due secondi risparmiati cambiano la percezione.
Condizione: **le icone devono essere dello stesso set**; una sola icona di stile diverso si nota
subito e fa sembrare tutto sciatto.

**Lo stato attivo deve essere ovvio.** Icona piena quando sei nella sezione, vuota quando sei
altrove: capisci dove sei senza leggere il titolo.

**Raggruppa per funzione.** Se hai funzioni sociali e funzioni personali, tienile in zone diverse
della barra. Il raggruppamento si comunica con la spaziatura: spazio piccolo dentro la categoria,
spazio 2-3 volte più grande tra categorie.

**Metti vicine le due cose più importanti.** Logo e menu, o logo e ricerca se la ricerca è l'azione
più frequente (in un e-commerce o in un social lo è).

**Tooltip solo quando l'icona non è autoesplicativa** — e un tooltip deve dire **cosa fa**, non cosa
è. "Contenuti divertenti" è utile; "LOL" no.

**Mantieni visibile ciò che serve sempre.** In un e-commerce la barra di ricerca dovrebbe restare
accessibile durante lo scroll.

**Un solo pulsante grande e colorato per l'azione principale del prodotto.** È così che si capisce a
cosa serve l'applicazione appena la si apre.

---

## Liste, dashboard e dati

**Prima il dato che permette di decidere.** In un annuncio immobiliare è il prezzo: se non puoi
permettertela, il resto è inutile da leggere. In una lista di voli è prezzo e durata.

**Non far fare calcoli all'utente.** "5 anni" invece di "costruita nel 2016". "Tra 20 minuti" invece
di un orario assoluto quando conta l'attesa.

**Icone + numero in grassetto** per i dati ripetuti e scansionabili (camere, bagni, posti): si legge
senza leggere.

**Una faccia crea fiducia.** La foto dell'agente, del venditore, dell'autore conta soprattutto nelle
decisioni importanti.

**Piani e prezzi:** se tre opzioni sono graficamente identiche, l'utente sceglie la più economica.
Se vuoi guidare la scelta, il piano consigliato è **più grande, più ricco di contenuto, evidenziato**
— e dice esplicitamente perché è consigliato.

**Contenuto infinito o "Carica altro"?** Lo scroll infinito funziona per contenuti di consumo
(feed). "Carica altro" o paginazione servono quando l'utente deve poter arrivare al footer, o
confrontare e ritrovare elementi. Se la pagina ha informazioni importanti in fondo, lo scroll
infinito le rende irraggiungibili — a meno di spostarle in una colonna laterale.

**Un pulsante "torna su"** su pagine lunghe.

---

## Modali, conferme e azioni distruttive

**Le azioni distruttive vanno confermate**, con il pulsante di conferma in rosso e un testo che dice
**esattamente cosa si perde** — non "Sei sicuro?", ma "Verranno eliminati definitivamente i tuoi 340
appunti".

**Posizione della conferma:** su mobile, un bottom sheet è preferibile a un modale centrato. Due
motivi:
- **psicologico**: il contenuto che si sta per perdere resta visibile sopra, e questo fa ripensare
  l'utente. Al centro dello schermo si vede solo il pulsante rosso e si agisce d'impulso;
- **ergonomico**: in basso è più facile da raggiungere col pollice.

Anche quando un utente se ne va, la sua esperienza deve essere buona: lo racconterà.

**Non colorare "Annulla" di rosso.** Crea tensione e fa sembrare enorme una decisione normale.
L'azione secondaria va senza colore o in tono neutro: l'occhio deve andare dritto all'azione
principale, e annullare deve sembrare possibile senza drammi.

**Non usare più di due colori forti nello stesso pop-up.** Blu + verde + rosso insieme sovraccaricano
e nessuno dei tre guida più nulla.

**Overlay scuro dietro il modale:** non è decorazione, dice dove guardare.

**Rendi facile andarsene.** Disiscrizione, cancellazione dell'account e richiesta di rimborso vanno
progettate bene quanto l'iscrizione. Un rimborso che richiede di scaricare un PDF, compilare sei
pagine e sperare in una risposta è il modo migliore per perdere un cliente per sempre e farglielo
raccontare a tutti.

---

## Immagini e testo sovrapposti

**Sfumatura, non riquadro colorato.** Se le immagini sono caricate dagli utenti (quindi
imprevedibili), un riquadro di colore fisso sotto la foto stonerà con qualcuna; e se il colore si
adatta alla foto, a volte il testo deve diventare scuro e si perde coerenza. Una sfumatura verso il
nero funziona con qualsiasi foto, tiene il testo sempre bianco e non taglia l'immagine.

Il riquadro pieno funziona solo se **controlli** le immagini (foto aziendali tutte con lo stesso
sfondo).

**Barre di controllo piene o semitrasparenti?** Dipende dal focus: piene se l'attenzione è sugli
strumenti (editing), semitrasparenti se conta vedere il contenuto (inquadratura, video). Attenzione:
testo bianco su nero semitrasparente sopra uno sfondo chiaro diventa illeggibile — serve un
fallback.

---

## Densità: quando "affollato" non è un difetto

Un'interfaccia con molte funzioni non è automaticamente mal progettata. Se **tutto ciò che si può
fare è visibile sullo schermo**, l'utente non deve cercare e non deve imparare: per certi prodotti è
esattamente la scelta giusta, perché gli utenti sono pigri e vanno messi davanti a qualcosa che non
richiede sforzo.

Il problema non è la quantità, è **l'assenza di gerarchia**: quando dieci blocchi hanno tutti lo
stesso peso visivo, l'utente non sa dove guardare. La correzione non è togliere roba, è decidere
cosa conta di più adesso (stagione, contesto, obiettivo) e dargli più spazio, raggruppando il resto
in righe tematiche coerenti.

Al contrario, un'interfaccia minimale funziona quando le funzioni sono poche e una sola è davvero
principale.

---

## Costi realistici

Non tutto ciò che è meglio va fatto. Icone illustrate uniche per ogni categoria sono più memorabili
di icone generiche, ma costano giorni di lavoro a designer e sviluppatori. **È professionale
ammettere che una soluzione è troppo impegnativa per il contesto** e proporre l'alternativa
sostenibile — purché tu lo dica, invece di far finta che non esistesse l'opzione migliore.
