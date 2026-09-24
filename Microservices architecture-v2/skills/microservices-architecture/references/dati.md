# Gestione dei dati distribuiti

Indice:
1. Database-per-Service e polyglot persistence
2. Relazionale o NoSQL, teorema CAP
3. Partizionamento e sharding
4. CQRS e viste materializzate
5. Event sourcing
6. Saga
7. Transactional Outbox e CDC
8. Caching distribuito

---

## 1. Database-per-Service e polyglot persistence — *Avanzato*

Ogni servizio possiede il proprio database, accessibile **solo** attraverso la sua API. È il pattern
fondante: senza questo, gli altri non hanno senso.

Ne discende la **polyglot persistence**: se ogni servizio possiede il suo storage, ognuno può usare
la tecnologia adatta al proprio modello di dati.

| Servizio | Natura dei dati | Scelta tipica |
|---|---|---|
| Catalog | schemi flessibili, strutture annidate, alto volume di lettura | documentale (MongoDB) o relazionale |
| Basket | temporanei, accesso velocissimo per chiave | key-value (Redis) |
| Ordering | transazionali, integrità, ACID | relazionale (PostgreSQL, SQL Server) |
| Search | full-text, ranking | motore di ricerca (Elasticsearch, OpenSearch) |

**Il prezzo, da dichiarare sempre**: spariscono le join. Non si può fare una join fra MongoDB e
PostgreSQL su macchine diverse. Le tre soluzioni legittime sono API composition, vista
materializzata e CQRS — e ognuna costa qualcosa.

**Shared database anti-pattern**: due servizi sullo stesso schema. Mantiene le transazioni ACID
locali e sembra pragmatico, ma lega i cicli di rilascio e trasforma ogni migrazione di schema in un
evento coordinato fra team. Il sintomo diagnostico: *un deploy bloccato da una migrazione che
impatta altri servizi.*

Livello intermedio accettabile: un solo server di database con **schemi separati** per servizio.
Non è il database-per-service, ma rende esplicita la proprietà e prepara la separazione.

---

## 2. Relazionale o NoSQL, teorema CAP — *Avanzato*

**ACID** (relazionale): Atomicity, Consistency, Isolation, Durability. Coerenza forte, integrità
referenziale, transazioni.

**BASE** (molti NoSQL): Basically Available, Soft state, Eventual consistency. Si rinuncia alla
coerenza immediata per disponibilità e scalabilità.

**I quattro tipi di NoSQL**

| Tipo | Struttura | Adatto a | Esempi |
|---|---|---|---|
| Documentale | documenti JSON | schemi flessibili, strutture annidate | MongoDB, Couchbase |
| Key-value | coppie chiave-valore | cache, sessioni, dati temporanei | Redis, Memcached, DynamoDB |
| Wide-column | righe con colonne dinamiche | volumi enormi, scritture massive | Cassandra, HBase, Bigtable |
| A grafo | nodi e relazioni | relazioni complesse da attraversare | Neo4j |

**Teorema CAP** (Brewer): in presenza di una partizione di rete si può garantire coerenza **o**
disponibilità, non entrambe. Siccome la partizione di rete non è opzionale in un sistema
distribuito, la scelta reale è fra **CP** e **AP**.

- **CP** — coerenza forte, si rifiutano le richieste durante la partizione. Cluster RDBMS, MongoDB
  in certe configurazioni.
- **AP** — disponibilità, si accettano scritture che convergeranno dopo. Cassandra, DynamoDB.

**Le quattro domande per scegliere**
1. Che livello di coerenza serve? Transazioni finanziarie → ACID. Feed, cataloghi, analytics →
   eventual accettabile.
2. Che forma hanno i dati? Strutturati e relazionali → relazionale. Denormalizzati o a schema
   variabile → NoSQL.
3. Che volume e che profilo di carico? Alto volume, write-heavy → NoSQL wide-column.
4. I confini transazionali sono allineati ai confini del servizio? Se no, la risposta sarà Saga.

**Da segnalare in review**
- NoSQL scelto per moda su dati fortemente relazionali e transazionali.
- Assunzione implicita di coerenza forte su uno store che offre eventual consistency.
- Nessuna motivazione scritta della scelta: è il problema più frequente, perché rende impossibile
  rivederla.

---

## 3. Partizionamento e sharding — *Avanzato*

**Tre strategie**
- **Orizzontale (sharding)** — righe divise per chiave su nodi diversi. Ogni shard ha lo stesso
  schema, dati diversi.
- **Verticale** — colonne divise per frequenza di accesso o sensibilità.
- **Funzionale** — dati divisi per funzione di business. Coincide con il database-per-service.

La **partition key** è la decisione che determina tutto. Deve distribuire uniformemente e comparire
nelle query più frequenti.

Gli store NoSQL distribuiti (Cassandra, MongoDB, DynamoDB, Cosmos DB) hanno sharding e replica
**nativi**: distribuiscono automaticamente in base alla partition key. Farlo a mano su un
relazionale è possibile ma molto più complesso, e le query cross-shard diventano un problema serio.

Cassandra è l'esempio istruttivo: architettura peer-to-peer senza master, ogni nodo ha lo stesso
ruolo, distribuzione tramite consistent hashing. È un sistema **AP** da manuale — e quindi non è il
posto dove mettere i saldi dei conti correnti.

**Da segnalare in review**
- Partition key a bassa cardinalità o correlata al tempo: crea hot partition.
- Query che non includono la partition key: colpiscono tutti i nodi (scatter-gather).
- Logica di routing degli shard scritta a mano nell'applicazione.
- Transazioni che attraversano più shard: quasi sempre non supportate.

---

## 4. CQRS e viste materializzate — *Avanzato*

Letture e scritture hanno profili opposti, e trattarle allo stesso modo penalizza entrambe.

| | Command (scrittura) | Query (lettura) |
|---|---|---|
| Richiede | validazione, integrità, ACID | velocità, forma pronta all'uso |
| Volume tipico | basso | alto |
| Modello ideale | normalizzato | denormalizzato |
| Esempi | `CreateOrder`, `UpdateProductPrice` | `GetProductDetails`, `ListCustomerOrders` |

**Due livelli di intensità** — questa distinzione evita di adottare CQRS in modo sproporzionato:
- **Leggero**: stesso database, viste o tabelle dedicate alla lettura. Basso costo, buona parte del
  beneficio.
- **Completo**: store fisicamente separati, tipicamente relazionale per la scrittura e NoSQL
  ottimizzato per la lettura, sincronizzati **tramite eventi**.

Instagram è l'esempio concreto: PostgreSQL per i dati utente che richiedono coerenza forte,
Cassandra per i feed che devono scalare massicciamente e dove l'eventual consistency è accettabile.

**Vista materializzata** — una copia pre-calcolata dei dati nella forma in cui serve leggerli,
mantenuta aggiornata consumando eventi. Spesso la si costruisce senza chiamarla così: un servizio
Basket che consuma `ProductPriceChangedEvent` e aggiorna i prezzi nella sua cache sta mantenendo una
vista materializzata.

**Da segnalare in review**
- CQRS introdotto su un CRUD semplice.
- Sincronizzazione fra write e read model fatta con **chiamata sincrona**: reintroduce esattamente
  l'accoppiamento che CQRS voleva eliminare. Deve passare da eventi.
- Aggiornamento del read model non idempotente.
- Requisito di lettura immediatamente coerente su un modello eventually consistent: è una
  contraddizione, va risolta a livello di requisito.

**Diagnosi**
- *Dati stantii nel read model* → ritardo o blocco della pipeline di eventi. Monitorare il lag è
  obbligatorio, non opzionale.
- *Divergenza permanente fra i due modelli* → manca un meccanismo di rebuild completo della vista
  dagli eventi.

---

## 5. Event sourcing — *Avanzato*

Invece di memorizzare lo stato corrente, si memorizza la **sequenza immutabile dei fatti** che lo
hanno prodotto. Lo stato si ricostruisce riproducendo gli eventi.

Proprietà: gli eventi sono al passato, immutabili, append-only. Si ottiene gratis una storia
completa e verificabile, e la possibilità di ricostruire qualunque stato passato.

Si accompagna naturalmente a CQRS: l'event store è il modello di scrittura, i read model si
costruiscono consumando gli eventi.

**Distinguere sempre due tipi di evento**, perché confonderli è l'errore strutturale più comune:
- **Domain event** — interno al servizio, in-process, granulare.
- **Integration event** — pubblicato sull'event bus, parte del contratto pubblico verso gli altri
  servizi, va versionato come un'API.

**Da segnalare in review**
- Eventi modificati o cancellati dopo la scrittura: viola la premessa.
- Eventi che trasportano l'intero stato invece del fatto accaduto.
- Nessuna strategia di versioning dello schema degli eventi.
- Nessuno snapshot su aggregati con storia molto lunga.

**Diagnosi**
- *Ricostruzione dello stato lentissima* → introdurre snapshot periodici.
- *Consumer che si rompe su eventi vecchi* → manca l'upcasting o il versionamento.

---

## 6. Saga — *Avanzato*

Il problema: un'operazione di business attraversa più servizi, ognuno con il proprio database, e le
transazioni ACID distribuite non sono praticabili.

**Perché non il 2PC.** Il commit a due fasi estende l'ACID su più database tramite un coordinatore,
ma blocca le risorse per tutta la durata, introduce un single point of failure e non è supportato
dalla maggior parte dei NoSQL e dei broker. Non è la risposta nei microservizi.

**La Saga** è una sequenza di transazioni locali. Ogni passo è ACID nel proprio servizio, e ogni
passo ha una **transazione compensativa** che ne annulla l'effetto se un passo successivo fallisce.

**La cosa da capire prima di tutto**: una Saga **non garantisce isolamento** — la "I" di ACID.
Durante la saga uno stato intermedio è visibile agli altri: l'ordine esiste ma non è ancora pagato.
Il monolite lo nascondeva, la Saga no. Questo va **progettato** nel dominio (stati espliciti come
`PENDING`, `CONFIRMED`, `FAILED`), non scoperto in produzione.

**I due stili**

| | Coreografia | Orchestrazione |
|---|---|---|
| Come | ogni servizio reagisce agli eventi degli altri | un orchestratore guida la sequenza |
| Accoppiamento | basso | l'orchestratore conosce tutti |
| Leggibilità del flusso | nessun punto dove è scritto per intero | centralizzata ed esplicita |
| Adatta a | saghe brevi, 2-4 passi | saghe lunghe o con logica condizionale |

Il difetto della coreografia è preciso e va detto: per capire cosa succede dopo `PaymentProcessed`
bisogna cercare nel codice di tutti i servizi chi lo sottoscrive. Su saghe lunghe questo rende il
sistema incomprensibile.

**Esempio di flusso (coreografia)**
```
Ordering: transazione locale → pubblica OrderCreated
Payment:  consuma OrderCreated → paga → pubblica PaymentProcessed
Inventory: consuma PaymentProcessed → riserva → pubblica StockReserved
           (fallimento) → pubblica StockReservationFailed
Payment:  consuma StockReservationFailed → RefundPayment (compensazione)
Ordering: marca l'ordine come fallito
```

**Da segnalare in review**
- 2PC fra microservizi.
- Passi privi di compensazione.
- Coreografia su saghe di molti passi.
- Stato intermedio esposto senza essere stato progettato.
- Compensazioni non idempotenti.
- Nessun timeout per passo: una saga bloccata a metà non si sblocca da sola.

---

## 7. Transactional Outbox e CDC — *Avanzato*

**Il dual write problem**: aggiornare il database e pubblicare l'evento sono due operazioni su
sistemi diversi. Se la prima riesce e la seconda no, il dato è cambiato e nessuno lo sa. Se la
seconda riesce e la prima no, si è annunciato un fatto mai avvenuto.

**La soluzione — Outbox**: scrivere il dato di business **e** il record dell'evento nella **stessa
transazione locale**, su una tabella `outbox`. L'atomicità del database garantisce che o ci sono
entrambi, o nessuno dei due.

```java
var strategy = dbContext.Database.CreateExecutionStrategy();
// transazione
//   1) logica di business (es. rimuove il carrello)
//   2) inserisce OutboxMessage { Type, Content (JSON), OccurredOn, ProcessedOn = null }
// commit atomico
```

Poi un **relay** pubblica gli eventi: un processo in background che fa polling sui record con
`ProcessedOn == null`, li pubblica sul broker e li marca come elaborati.

L'ironia da conoscere: l'Outbox risolve il dual write dell'applicazione ma lo reintroduce nel relay
(pubblica, poi marca — se cade in mezzo, ripubblica). Per questo **i consumer devono essere
idempotenti comunque**.

**CDC (Change Data Capture)** è la versione migliore del relay: legge il log delle transazioni del
database e pubblica sul broker, senza polling e senza che l'applicazione conosca il broker.
Debezium è lo strumento di riferimento; alcuni database hanno il CDC integrato (CockroachDB Change
Feeds, Cosmos DB Change Feed, DynamoDB Streams).

**Da segnalare in review**
- Evento pubblicato prima del commit, o dopo il commit ma fuori transazione.
- Relay che marca il record come elaborato **prima** della conferma di pubblicazione.
- Nessun indice sulla colonna di stato dell'outbox.
- Tabella outbox che cresce senza purge.

---

## 8. Caching distribuito — *Intermedio*

In un servizio replicato una cache in memoria locale è un problema, non una soluzione: ogni istanza
vede dati diversi. Serve una cache **distribuita** e condivisa.

**Le quattro strategie**

| Strategia | Chi gestisce il miss | Coerenza | Velocità in scrittura |
|---|---|---|---|
| Cache-aside | l'applicazione | buona | normale |
| Read-through | la cache | buona | normale |
| Write-through | scrive in cache e **sincronamente** nel DB | forte | lenta |
| Write-behind | scrive in cache e **asincronamente** nel DB | eventual | velocissima, rischio di perdita |

**Cache-aside** è il default sensato: si cerca in cache; se miss, si legge dalla fonte e si popola la
cache. In lettura:

```java
return cache.getOrCreate(userName, token -> repository.findByUserName(userName));
```

In scrittura, scrivere **sempre prima nel database** (fonte di verità) e poi aggiornare la cache.

Il punto architetturale che vale la pena verificare: **la cache non è la fonte di verità.** Se Redis
viene svuotato e i dati spariscono, quello non era un uso di Redis come cache — era un uso di Redis
come database, e va reso esplicito o corretto.

Livelli: L1 in memoria locale (velocissima, può divergere fra repliche), L2 distribuita (condivisa).
Le librerie di hybrid cache gestiscono entrambi, con TTL separati.

**Da segnalare in review**
- Cache in memoria locale in un servizio replicato.
- TTL assente, o identico per tutte le chiavi.
- Nessuna invalidazione sulle scritture.
- Chiavi senza namespace: collisioni fra servizi.

**Diagnosi**
- *Dati vecchi mostrati all'utente* → invalidazione mancante sulla scrittura.
- *Picco sul database dopo un restart o una scadenza di massa* → cache stampede. Sfalsare i TTL e
  proteggere il ricalcolo.
- *Cache miss costanti* → chiave calcolata in modo non deterministico (ordine di campi, timestamp).
