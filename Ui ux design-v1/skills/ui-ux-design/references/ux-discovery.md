# UX discovery — dall'obiettivo alla struttura

Indice:
1. I sette fattori della UX
2. Il brief: obiettivo, utente, competizione, funzionalità, metriche
3. Target e personas
4. Le cinque variabili dell'utente
5. Ricerca: sondaggi e interviste
6. Messy middle e customer journey
7. Architettura delle informazioni

---

## 1. I sette fattori della UX

Un prodotto ha probabilità di successo quando li soddisfa tutti, non solo i primi due. Usali come
griglia di valutazione rapida di qualunque prodotto o schermata.

| Fattore | Domanda |
| --- | --- |
| **Utile** | Risolve un bisogno reale? Il beneficio può anche essere divertimento o estetica. |
| **Usabile** | Permette di raggiungere l'obiettivo in modo efficace ed efficiente? |
| **Trovabile** | Si trova il prodotto, e si trovano i contenuti dentro il prodotto? |
| **Credibile** | Fa ciò che promette? Le informazioni sono accurate? Oggi non c'è seconda occasione: le alternative sono troppe. |
| **Desiderabile** | Suscita desiderio? Identità, estetica, design emozionale. |
| **Accessibile** | Funziona per utenti con tutto lo spettro di abilità? Circa 1 persona su 5 ha una disabilità: su un milione di utenti sono 200.000 persone escluse. |
| **Di valore** | Dà valore sia a chi lo usa sia a chi lo produce? Un prodotto da 100 che risolve un problema da 10.000 ha ottime probabilità; il contrario no. |

**Regola di lettura:** se una schermata fallisce su uno di questi, il problema non si risolve con la
grafica.

---

## 2. Il brief

Cinque blocchi, **in quest'ordine**. L'ordine non è decorativo: senza obiettivo, utente e
competizione non hai criteri per decidere le funzionalità.

### 2.1 Obiettivo

L'obiettivo di business è il **risultato misurabile** che il progetto deve produrre, non la cosa che
è stata chiesta.

Distingui tre livelli:
- **Problema**: "mi fa male il dente"
- **Obiettivo**: "voglio che il dolore sparisca"
- **Soluzione**: la decide chi è esperto

Chi chiede arriva quasi sempre con il problema **e** la sua ipotesi di soluzione ("voglio un'app").
Il lavoro è verificare se quella soluzione serve davvero all'obiettivo. Non dare per scontato che
sappia spiegare il proprio obiettivo: succede di sentirsi chiedere un'app mobile per arrivare primi
su Google.

Due condizioni: l'obiettivo va **detto esplicitamente**, e deve esserci **accordo** che sia quello.

### 2.2 Utente

Vedi sezioni 3 e 4.

### 2.3 Competizione

"Non abbiamo competitor" significa quasi sempre "non ci ho ancora pensato". Se davvero non ne hai,
probabilmente il mercato non esiste.

**Definizione operativa: il competitor non è chi fa un prodotto simile, è chiunque risolva lo stesso
bisogno.** Uno scooter compete con l'autobus, il car sharing, il monopattino, i piedi e con la
decisione di non andare all'appuntamento. Un gestionale di task compete con Excel, con carta e penna
e con "tengo tutto a mente".

Tre domande su ogni competitor:
1. **Come comunica?** Quali caratteristiche sottolinea, e perché proprio quelle? Da lì deduci chi si
   immagina davanti. Un software che ripete "siamo i più usati al mondo" sta rassicurando un manager
   che non vuole fare brutta figura scegliendo; uno che parla informale e disegnato a mano sta
   parlando a un team piccolo e remoto.
2. **Chi è il suo target?** È lo stesso tuo? Un'azienda strutturata ha processi d'acquisto lunghi e
   multi-ruolo; un team piccolo compra d'impulso ma cambia software più facilmente.
3. **Qual è il modello di business?** Vende, noleggia, abbona? A quanto? Il tuo prodotto viene
   percepito più economico o più caro?

A cosa serve: capire il mercato, stabilire **quante funzionalità servono già nella prima versione**
(mercato saturo = devi uscire più completo), e i tempi (pochi competitor = puoi testare con calma).

### 2.4 Funzionalità

Vengono **quarte**, non prime. Due assi:

**Importanza** — tre domande:
- Quanti utenti la useranno? 100%? 50%? Nessuno, e la facciamo solo perché è stata chiesta?
- È in linea con l'obiettivo di business? (Se l'obiettivo è ricevere richieste, il form di contatto
  non è opzionale.)
- La prova del nove: se sparisse per magia, il prodotto funzionerebbe ancora? L'utente se ne
  accorgerebbe e si lamenterebbe?

**Fattibilità** — tre domande:
- Ci sono i soldi? Un'app mobile seria significa iOS + Android + backend.
- Le persone sono disponibili, o sono su altro?
- È chiaro il costo di **manutenzione**? Un blog non è una funzionalità, è qualcuno che scrive
  contenuti per sempre.

**Il risultato è una matrice a quattro quadranti:** importanti e fattibili si fanno; poco importanti
e poco fattibili si rimandano; gli altri due quadranti si negoziano. Il vantaggio pratico: quando
arriva l'imprevisto — e arriva — hai la prova di aver lavorato sulle cose più importanti.

**Attenzione alle funzionalità che ne trascinano altre.** "Voglio vendere online" sembra una
funzionalità: in realtà implica ricevuta o fattura, mail di conferma ordine, mail di attesa
pagamento, notifica all'amministratore, lista ordini, registrazione utente e quindi recupero
password. Se ne manca una, per l'utente finale **il sito non funziona**.

**L'importanza è relativa al tempo, non assoluta.** La versione in lingua straniera è inutile oggi e
fondamentale tra un mese se c'è un piano di espansione.

### 2.5 Metriche

Chi fa UX porta l'utente da uno stato A a uno stato B, e in mezzo c'è un'azione. Quindi il lavoro si
valuta sui numeri, non sull'entusiasmo della presentazione. Le domande hanno risposta binaria: più
visite sì o no, più acquisti sì o no, più richieste di contatto sì o no.

| Non misurabile | Misurabile |
| --- | --- |
| "fare un sito per diventare ricco" | +N clienti rispetto all'anno scorso |
| "ricevere mail da tantissimi utenti" | numero di mail ricevute al giorno |
| "far invidia al competitor" | tempo medio di navigazione, condivisioni social |
| "darsi un tono" | ticket di assistenza aperti, prodotti acquistati vs resi |

Regola: **prima di chiudere la fase di brief devi avere un obiettivo misurabile scritto.**

---

## 3. Target e personas

Sono cose diverse e vengono confuse di continuo.

**Target** = il gruppo che vogliamo raggiungere, definito da caratteristiche demografiche o
comportamentali. Sono **dati quantitativi**. Non si inventa: si ricava dalle statistiche.

Cosa guardare e perché serve:

| Dato | A cosa serve |
| --- | --- |
| Genere ed età | Verificare se chi arriva è chi immaginavi. Sull'età le sorprese sono frequenti: proiettiamo la nostra età sugli utenti. |
| Dimensione schermo, % mobile vs desktop | È l'errore più comune: progettare sulla propria risoluzione e scoprire che il 60-70% è da mobile. Progetta sulla risoluzione più diffusa tra i **tuoi** utenti. |
| Nuovi vs di ritorno | Indicatore di fiducia e fidelizzazione. |
| Pagine di entrata e uscita | Il flusso di navigazione reale, non quello ipotizzato. |
| Sorgenti di traffico | Sorgenti diverse = atteggiamenti diversi. Chi arriva da un post social è superficiale (per lui sei un'interruzione); chi arriva da newsletter o URL diretta resta molto più a lungo. |
| Orario | Cambia il contesto: alle 4 di notte si guardano immagini e pulsanti, non i testi. È uno dei motivi reali della dark mode. |

**Personas** = il profilo **narrativo e qualitativo** di un utente tipo. Chi è, che lavoro fa, quali
obiettivi ha, quali problemi, come parla, che vita fa. L'idea è di Alan Cooper: stampare il profilo
dell'utente ideale e tenerlo davanti allo schermo, per non dimenticare che si progetta per persone e
non per numeri.

Servono a due cose: conoscere gli obiettivi di chi userà il prodotto, e da lì capire **quali
funzionalità hanno priorità massima, quali aspettano e quali non si fanno**.

Esempio: la stessa sedia da ufficio ha quattro personas — il ragazzo che gioca 20 ore al giorno, il
bambino delle elementari, la pensionata, l'impiegato di banca. Esigenze completamente diverse.

**Le personas divergono sui dettagli e convergono sul bisogno.** Un software di fatturazione può
avere l'artigiana che non vuole aprire Excel, l'amministratore di srl che ha problemi con le
scadenze e il libero professionista: tre profili, un unico bisogno comune (fatturare senza
sbattimenti).

**Nessun cliente ti darà queste informazioni in prima riunione.** Il lavoro è tuo: domande in
riunione + statistiche + eventuali interviste.

---

## 4. Le cinque variabili dell'utente

Su queste si costruisce una persona utile e si prendono decisioni di layout.

**Fisiche** — come usa fisicamente il prodotto. Mobile o desktop? Una mano o due? Verticale o
orizzontale? Dove guarda quando apre l'app? È il motivo per cui su mobile un'azione in basso viene
cliccata più di una in alto: è vicina al pollice. Ed è il motivo dei video verticali sui social.

**Ambientali** — cosa sta facendo mentre ti usa. Rumore? Venti schede aperte? Per strada o sul
divano? Ha tempo? Esempio limite: le app di navigazione. Un'interfaccia complicata mentre si guida
può causare un incidente, quindi devono mostrare solo l'essenziale e ridurre al minimo i tap.

**Preferenziali** — come preferisce consumare i contenuti: testo, video, audio? Quali strumenti usa?
Esempio reale: un sito con target anziano dove gli utenti **stampavano** le pagine invece di
leggerle a schermo. Soluzione: un pulsante gigante "Stampa" a fine pagina. Obiezione ovvia: potevano
usare il menu del browser. Risposta: quel tipo di utenza non sa che esiste — qualcuno ha risposto
"ma che cos'è un browser?".

**Cognitive** — capacità di imparare cose nuove, linguaggio, età, distanza dal mondo tecnico.
Attenzione a ciò che diamo per scontato: che sappia cos'è un'app, come si scarica, come si legge un
ebook. Per chi lavora ogni giorno con la tecnologia è facilissimo credere che tutti sappiano almeno
accendere un computer.

**Emozionali** — quanto è stressato, quanta motivazione ha, cosa succede se non ottiene quel che
vuole. Ogni azione che chiedi, anche solo una mail, è uno stress. Confronta i due estremi: chi
prenota una vacanza (estasi, tolleranza alta) e chi deve chiedere un rimborso a un portale pubblico
(obbligato, già esasperato). Davanti alla stessa pagina lenta, non reagiscono allo stesso modo.

---

## 5. Ricerca: sondaggi e interviste

**Quantitativa = numeri. Qualitativa = parole.**

| | Quantitativa | Qualitativa |
| --- | --- | --- |
| Metodo | Sondaggi | Interviste, test |
| Partecipanti | Da 5-30 a migliaia | Da 3 a 10 (ogni intervista costa tempo) |
| Domande | Chiuse o a scala | Aperte |
| Risultato | Grafici, tendenze | Trascrizioni, temi ricorrenti |
| Esempio | "Da 1 a 10, quanto dolore senti?" | "Com'è il tuo dolore?" |

### Sondaggi — come si fanno

1. Definisci scopo e obiettivi (di solito si parte da un problema: pochi utenti, difficoltà
   segnalate, volontà di espandersi a un nuovo target).
2. Scomponi in sotto-obiettivi.
3. Formula domande **oggettive**.
4. Per ogni domanda chiediti: **questa risposta mi dà qualcosa che posso usare?** Se no, togli la
   domanda.
5. Usa le scale: trasformano emozioni difficili da descrivere in numeri confrontabili.
6. Aggiungi uno o due campi aperti facoltativi: spesso la risposta non è sì o no ("prendo appunti
   solo durante le riunioni" è un'informazione preziosa).
7. Testa il sondaggio e **cronometra**. Dichiarare il tempo è rispettoso; dire "5 minuti" quando ne
   servono 30 fa abbandonare tutti.

**Analisi:** pulisci i dati → individua tendenze generali e vittorie facili → cerca correlazioni
(età, genere, provenienza) → visualizza → **riassumi le intuizioni e agisci**. L'ultimo passo è il
più importante: se non porta a una modifica, lo studio non è servito a niente.

### Interviste — come si fanno

- Domande **aperte**: quasi nessuna deve avere risposta sì/no. Le chiuse valgono solo come ponte
  verso un approfondimento.
- Chiedi **perché** di continuo, anche quando pensi di sapere la risposta.
- Domande sempre oggettive: mai "lo fai perché sei pigro?".
- Durata: 10 minuti sono pochi, oltre un'ora è troppo. **30 minuti** è il punto giusto.

**Analisi:** mai discutere "a memoria" con i colleghi, la memoria non è affidabile. Crea le
trascrizioni, leggile tutte, evidenzia per temi (un colore per tema), estrai citazioni rilevanti,
confronta i risultati con le tue ipotesi e aggiorna le personas.

**Attenzione al divario tra ciò che le persone dicono di fare e ciò che fanno.** A volte si
presentano meglio di come sono ("ho la macchina ma non la uso per il carburante", quando la macchina
non ce l'ha). Anche quella bugia è un dato: ti dice che tiene al suo status, e che una soluzione che
lo faccia sentire povero verrà rifiutata anche se gli sarebbe utile.

### Empatia

L'empatia è capire l'esperienza altrui mettendo da parte la propria cultura e le proprie opinioni.
L'**empathy gap** è la distanza tra ciò che pensiamo dell'utente e ciò che l'utente vive davvero.

Le qualità che la rendono possibile: abbandonare l'ego, umiltà, ascolto vero (non formulare la
risposta mentre l'altro parla), osservazione (linguaggio del corpo, ambiente: ciò che dicono è solo
una parte della storia), cura reale, curiosità, sincerità.

Strumento pratico, **Cosa – Come – Perché**, per passare dal concreto all'astratto osservando:
- **Cosa**: i fatti. Cosa fa, cosa ha in mano, cosa succede intorno.
- **Come**: il modo, con aggettivi. Fatica? Sorride? Usa scorciatoie?
- **Perché**: l'interpretazione delle motivazioni emotive. È un'ipotesi, va verificata.

---

## 6. Messy middle e customer journey

Nessuno decide su due piedi un mutuo o una vacanza. Più la scelta è importante, più tempo e
informazioni servono.

**Messy middle** (modello di Google): lo spazio decisionale disordinato tra lo stimolo iniziale e
l'acquisto. Due movimenti:
- **Esplorazione**: l'utente raccoglie opzioni, non ha ancora chiari i criteri. Movimento espansivo.
- **Valutazione**: scarta e confronta. Movimento riduttivo.

Le due fasi **non sono consecutive**: si rimbalza tra loro per ore, giorni o settimane,
reintroducendo opzioni già scartate.

Due conseguenze operative:
1. **L'esperienza si forma nella testa dell'utente molto prima che arrivi da te.** Amici, forum,
   comparatori, esperienze precedenti col tuo brand o con quello del competitor: tutto questo
   determina le aspettative con cui atterra sulla tua pagina. Un'esperienza precedente ottima fa
   saltare l'intero processo di confronto.
2. **In fase di valutazione devi rendere le caratteristiche confrontabili**, e quindi oggettive.
   "Scegli noi perché siamo i migliori" non è confrontabile. "Rispondiamo entro 24 ore a qualunque
   richiesta di supporto" lo è.

### La customer journey

Tabella: **colonne = fasi**, **righe = quattro domande**.

Fasi: esplorazione → valutazione → decisione → post-decisione → post-esperienza.

Righe: cosa sta facendo? cosa pensa? cosa prova? **quali opportunità abbiamo qui?**

Esempio completo (acquisto di un biglietto aereo):

- **Esplorazione.** Non è ancora da te: chiede in un gruppo, un amico gli racconta, prova un
  comparatore. Prova entusiasmo. *Opportunità*: esistere nella sua testa — passaparola, pubblicità,
  e soprattutto un'esperienza precedente memorabile.
- **Valutazione.** Arriva sul tuo sito. **Non va più convinto a partire**: quella battaglia è già
  vinta, ha già rotto il salvadanaio. Pensa: "voglio il volo più economico, comodo e breve".
  *Opportunità*: rendere l'acquisto il più facile possibile — meno step, meno testo, possibilità di
  parlare con qualcuno se va storto — e dargli tutte le informazioni per procedere. Ma attenzione al
  rovescio: **un eccesso di informazioni trasmette la paura di sbagliare.** Proporre al checkout
  un'assicurazione contro il fallimento della compagnia introduce un dubbio che non esisteva, nel
  momento peggiore.
- **Decisione.** Massima tensione: ha la carta di credito in mano. Teme di perdere l'ultimo posto, di
  sbagliare la prenotazione, che qualcosa vada storto coi dati di pagamento. *Opportunità*:
  tranquillizzare — salvare i dati per la prossima volta, dare una finestra per modificare l'ordine
  dopo l'acquisto.
- **Post-decisione.** Esaltazione, voglia di raccontarlo (ecco perché la condivisione social compare
  subito dopo l'acquisto). *Opportunità*: rassicurare. Mail di riepilogo, evento da aggiungere al
  calendario, un contatto reale in caso di problemi.
- **Post-esperienza.** Ha usato il servizio. *Opportunità*: dargli modo di dirti com'è andata e di
  condividerlo, e restare in contatto in vista del prossimo bisogno.

**Il lato oscuro, da progettare sempre:** richieste di supporto, rimborsi, cancellazioni,
disiscrizioni. Il contro-esempio classico è il rimborso che richiede di scaricare un PDF, compilare
sei pagine e sperare che qualcuno risponda. Anche chi sta per andarsene deve poterlo fare
facilmente: è l'ultima cosa che racconterà di te.

---

## 7. Architettura delle informazioni

**Definizione:** l'organizzazione dei contenuti e del flusso di navigazione che ci aspettiamo
l'utente compia. In pratica: progettare la struttura, prima della grafica.

**La metafora della libreria.** Hai scaffali vuoti (la struttura) e centinaia di libri (i contenuti)
da disporre in modo che chi entra trovi ciò che cerca. Guarda come è organizzata una libreria fisica
e traduci:

| Zona della libreria | Corrispettivo digitale |
| --- | --- |
| Vicino all'entrata, i titoli del momento | I più popolari, i più cliccati: quello che dai in pasto a chi non ha ancora chiaro l'obiettivo |
| Area "in evidenza" (stessa casa editrice) | Prodotti sponsorizzati: qui vince l'obiettivo di business |
| I generi letterari | Le categorie. Qui la **terminologia è decisiva** |
| Dentro il genere, ordine alfabetico | L'ordinamento: arriva per ultimo, non per primo |

Tieni sempre insieme **due obiettivi**: quello di business (cosa fa guadagnare) e quello dell'utente
(perché è arrivato qui). Se ne servi solo uno, il progetto fallisce dall'altro lato.

### Le tre domande

**1. Come organizziamo il contenuto? Che priorità diamo alle informazioni?**
Quale informazione serve all'utente per poter compiere un'azione qualunque? Esempio: un sito di
ristoranti chiede subito la città, perché senza quella la ricerca non ha senso — ed è un dato che
l'utente conosce sempre. Contro-esempio reale: negozi di sedie da ufficio con schede tecniche
lunghissime **senza il prezzo**. Risultato: l'utente ha comprato su Amazon per 800 euro pur avendo
un negozio a 10 minuti da casa, perché non se la sentiva di entrare senza sapere se vendevano sedie
da 50, 1.000 o 10.000 euro.

**2. Che nome diamo ai contenuti?**
Voci di menu, titoli, categorie, tag. Quali termini si aspetta l'utente? "Narrativa di mistero",
"noir" o "gialli" non sono equivalenti per chi legge. La regola: usa la parola che usa l'utente, non
quella interna all'azienda. Le decisioni migliori (per esempio: mettere il numero di bagni tra i
primi dati di un annuncio immobiliare) non si prendono a naso, si prendono dopo aver parlato con gli
utenti.

**3. Qual è il flusso di navigazione atteso?**
Da dove entra, dove deve arrivare, quanti passaggi servono. Progetta il flusso intero prima delle
singole schermate.

### Regole pratiche

- **Raggruppa per significato e comunica il gruppo con la spaziatura**, non con i bordi.
- **Gerarchia esplicita:** ciò che conta di più è più grande, più in alto, più contrastato. Tre
  livelli bastano quasi sempre.
- **Breadcrumb** quando la struttura ha più livelli: dicono dove sei e come risalire.
- **Ordine semantico prima dell'ordine visivo.** Se l'azione più importante è "Prenota online",
  mettila per prima nel markup anche se visivamente la sposti altrove: chi naviga da tastiera o con
  screen reader la incontra per prima.
- **Niente vicoli ciechi.** Ogni pagina deve dire cosa si può fare dopo, comprese le pagine di
  errore e le liste vuote.
