# Comunicazione fra servizi

Indice:
1. Sincrono o asincrono
2. Design di API REST fra servizi
3. OpenFeign
4. Service registry e discovery
5. API Gateway
6. Backends for Frontends
7. gRPC e GraphQL
8. Messaggistica: Kafka e RabbitMQ

---

## 1. Sincrono o asincrono — *Avanzato*

| | Sincrono | Asincrono |
|---|---|---|
| Il chiamante | attende la risposta | prosegue subito |
| Protocolli | HTTP/REST, gRPC, GraphQL | AMQP, messaggistica |
| Accoppiamento | temporale: entrambi devono essere vivi | nessun accoppiamento temporale |
| Quando | serve la risposta per completare la richiesta utente | notifiche, propagazione, operazioni lunghe |

**Comando o evento** — la distinzione che determina l'accoppiamento:

- Un **comando** è diretto a un destinatario specifico e imperativo: `CreateOrder`. Chi lo invia sa
  chi deve riceverlo. Mantiene un accoppiamento.
- Un **evento** constata un fatto avvenuto, al passato: `OrderCreated`. Chi lo pubblica non sa chi
  se ne interesserà, e non deve saperlo. È qui che sta il disaccoppiamento.

Esempi di nomi corretti: `OrderPlacedEvent`, `ProductPriceChangedEvent`, `UserRegisteredEvent`,
`InventoryUpdatedEvent`.

**Da segnalare in review**
- Catene sincrone di tre o più servizi su un percorso utente: la latenza si somma e la disponibilità
  si moltiplica (tre servizi al 99,9% danno 99,7%).
- Chiamate sincrone usate solo per notificare: candidate a diventare eventi.
- Eventi con nomi all'imperativo dove servirebbe un fatto al passato.
- Nessun timeout sulla chiamata remota.

---

## 2. Design di API REST fra servizi — *Intermedio*

Le regole sono quelle del design REST generale (sostantivi per le risorse, verbi HTTP per le azioni,
JSON, niente annidamento profondo), con tre attenzioni specifiche al contesto distribuito:

- **Versionare il contratto** prima di averne bisogno. Un breaking change fra servizi non è un
  fastidio, è un incidente di produzione.
- **Paginare sempre** le collezioni. Il chiamante è remoto: un payload che in-process era gratuito
  ora costa banda e latenza.
- **Non annidare le risorse fra servizi diversi.** Se per ottenere gli articoli di un ordine serve
  chiamare il Catalog per ognuno, il problema non è l'URL: è il numero di round-trip.

**Da segnalare in review**
- Azioni nell'URL, `GET` che modifica stato, `PUT` non idempotente.
- Risposte con forma variabile a seconda dei casi.
- Campi rimossi o rinominati senza versione.

---

## 3. OpenFeign — *Intermedio*

Client HTTP dichiarativo: si descrive il servizio remoto con un'interfaccia Java annotata, Feign
genera l'implementazione a runtime.

```java
@SpringBootApplication
@EnableFeignClients
public class Application { }

@FeignClient("CatalogService")            // solo il nome logico: l'indirizzo lo risolve il registry
public interface CatalogClient {
    @GetMapping("/api/products/{id}")
    ResponseEntity<Product> getProduct(@PathVariable Long id);
}
```

Il passaggio chiave: **senza registry** si scrive `@FeignClient(name = "X", url = "localhost:8080")`;
**con registry** l'`url` sparisce e resta solo il nome, che deve corrispondere esattamente allo
`spring.application.name` del servizio remoto.

**Da segnalare in review**
- URL fisici cablati in presenza di service discovery.
- Nome del client che non corrisponde al nome registrato.
- Nessun timeout, retry o fallback configurato: un Feign nudo è una chiamata remota senza rete di
  protezione.
- DTO duplicati fra chiamante e chiamato senza un contratto condiviso o generato.

**Diagnosi**
- *Errore di risoluzione del nome* → servizio non registrato, o discovery non attivo sul chiamante.
- *`404` sul metodo Feign* → path o verbo divergenti dal controller remoto.
- *Latenza anomala* → timeout di default troppo permissivi.

---

## 4. Service registry e discovery — *Intermedio*

Ogni servizio si registra all'avvio con nome logico e indirizzo, invia heartbeat periodici e si
cancella alla chiusura. Chi deve chiamarlo chiede al registry la lista delle istanze vive.

```
[Provider] --register: "sono CatalogService, su 10.0.0.5:8080"--> [REGISTRY]
[Consumer] --"dammi le istanze di CatalogService"-------------->
           <--lista di indirizzi----------------------------------
           --chiamata diretta--> [Provider]
```

**Eureka** — server con `@EnableEurekaServer`, `register-with-eureka: false` e `fetch-registry:
false` (il server non si registra a se stesso e non scarica da altri, a meno di un cluster). I
client dichiarano `spring.application.name` e `eureka.client.serviceUrl.defaultZone`. Nelle versioni
recenti `@EnableDiscoveryClient` non è più necessaria: basta la dipendenza.

**Su Kubernetes il registry non serve.** Il ruolo lo svolgono `Service` + DNS interno del cluster.
Questa è una sostituzione da conoscere: molti sistemi trascinano Eureka dentro Kubernetes
duplicando un meccanismo che la piattaforma già fornisce.

| Spring Cloud | Equivalente Kubernetes |
|---|---|
| Eureka (discovery) | `Service` + DNS interno |
| Ribbon / load balancer | `Service` ClusterIP / LoadBalancer |
| Spring Cloud Gateway | `Service` LoadBalancer, o `Ingress` |
| Config Server | `ConfigMap` e `Secret` |

**Diagnosi**
- *Servizio non visibile nel registry* → ordine di avvio (il registry va per primo), `defaultZone`
  errato, o heartbeat non arrivati.
- *Chiamata risolta su un'istanza morta* → il lease non è ancora scaduto. Ridurre gli intervalli
  aiuta, ma la vera protezione è il circuit breaker.

---

## 5. API Gateway — *Avanzato*

Unico punto di ingresso che nasconde la topologia interna. Tre pattern distinti, spesso confusi:

- **Gateway Routing** — instrada la richiesta al servizio giusto in base al path.
- **Gateway Aggregation** — combina più chiamate a valle in una sola risposta, riducendo i
  round-trip del client remoto.
- **Gateway Offloading** — assorbe le preoccupazioni trasversali: autenticazione, rate limiting,
  terminazione TLS, logging, traduzione di protocollo.

L'offloading abilita una soluzione elegante: il browser parla REST, i servizi interni parlano gRPC,
il gateway traduce.

**Spring Cloud Gateway** — route dichiarative (predicate, filtri, URI) o via `RouteLocator` in Java.
Due meccanismi da conoscere:
- `lb://NOME-SERVIZIO` delega la risoluzione e il bilanciamento al registry.
- `spring.cloud.gateway.discovery.locator.enabled=true` crea automaticamente una route per ogni
  servizio registrato: un nuovo microservizio viene esposto senza toccare la configurazione. Il nome
  va scritto in maiuscolo, perché Eureka normalizza i nomi.

**Da segnalare in review**
- Logica di business nel gateway.
- Gateway senza rate limiting né autenticazione.
- Route troppo generiche che catturano traffico non previsto.
- Gateway non replicato: è diventato il single point of failure che l'architettura doveva evitare.

**Diagnosi**
- *`404` dal gateway* → predicate di path non corrispondente, o prefisso non rimosso nella
  trasformazione.
- *Servizio non raggiunto via `lb://`* → nome errato o non normalizzato.
- *Latenza aggiunta* → il gateway sta aggregando in serie ciò che potrebbe fare in parallelo.

---

## 6. Backends for Frontends — *Avanzato*

Un backend dedicato per ogni tipo di client, perché web e mobile hanno esigenze opposte: il web
tollera payload grandi e molte chiamate, il mobile ha banda limitata e connessione intermittente.

Ogni BFF appartiene al **team del frontend** che serve, e può usare uno stack diverso: un Web BFF in
ASP.NET o Spring, un Mobile BFF in Node.js con GraphQL se il team mobile è fatto di sviluppatori
JavaScript. Questo è il vantaggio più sottile e forse il più prezioso: allinea i confini del
software ai confini dei team.

**I tre costi da mettere in conto**
1. Più componenti da sviluppare, testare e rilasciare.
2. Duplicazione fra BFF. Il BFF risolve la duplicazione fra client, ma può reintrodurla fra BFF; si
   mitiga con librerie condivise di accesso ai servizi core.
3. Un hop di rete in più. Il BFF riduce la chattiness del client ma deve essere lui stesso
   performante e replicato.

**Da segnalare in review**
- BFF che diventa un secondo monolite di aggregazione.
- Logica di dominio nel BFF invece che nel servizio proprietario.
- Un solo BFF che serve client con esigenze opposte: non è un BFF, è un gateway travestito.

---

## 7. gRPC e GraphQL — *Avanzato*

**gRPC** — per il traffico **interno** sensibile alle prestazioni. Contratto in file `.proto`,
serializzazione binaria con Protocol Buffers, trasporto HTTP/2, stub e skeleton generati.

```protobuf
service DiscountService {
  rpc GetDiscount (GetDiscountRequest) returns (DiscountModel);
}
```

gRPC **non sostituisce REST**. Per i client esterni, e in particolare per i browser, REST e JSON
restano la scelta giusta: gRPC non è richiamabile direttamente da un browser senza gRPC-Web, che
aggiunge complessità. Il taglio corretto è: REST verso l'esterno, gRPC per il sistema nervoso
interno.

**GraphQL** — per i client con esigenze di dati variabili. Un solo endpoint, schema tipizzato,
query che il client compone chiedendo esattamente i campi che vuole, resolver che recuperano i dati.

Il punto da capire, e da dire quando qualcuno propone GraphQL per risolvere il problema N+1:
**GraphQL non elimina le N chiamate, le sposta.** Prima le faceva il client attraverso internet,
ora le fa il layer GraphQL dentro il datacenter. È un miglioramento reale, ma è questo, non magia.

**Da segnalare in review**
- gRPC esposto direttamente ai browser senza livello intermedio.
- Resolver GraphQL che innescano query a cascata.
- Schema GraphQL senza limiti di profondità o complessità: è un vettore di DoS.
- Modifiche ai numeri di campo in un `.proto`: rompono la compatibilità binaria.

---

## 8. Messaggistica: Kafka e RabbitMQ — *Avanzato*

**I due tipi di comunicazione asincrona**

| | One-to-one (point-to-point) | One-to-many (publish/subscribe) |
|---|---|---|
| Struttura | code | topic |
| Chi riceve | un solo consumer | tutti i sottoscrittori |
| Tecnologie | RabbitMQ con code dirette, SQS | Kafka, RabbitMQ con exchange, SNS |

**Kafka** — piattaforma di streaming distribuita. Concetti da tenere fermi:
- **Broker** = un server Kafka. **Cluster** = un insieme di broker.
- **Topic** = un flusso di dati, diviso in **partizioni**, ognuna con un **offset** progressivo.
- **Consumer group** = l'insieme dei consumer che si dividono le partizioni di un topic.
- Il log degli eventi è **persistente**: si può rileggere dall'inizio. È la differenza che rende
  Kafka adatto all'event sourcing e alle Saga.

Configurazione Spring minima:
```properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.producer.key-serializer=...StringSerializer
spring.kafka.producer.value-serializer=...StringSerializer
```
```java
@Service class Producer { @Autowired KafkaTemplate<String,String> t;
    void send(String m) { t.send("my-topic", m); } }

@Service class Consumer {
    @KafkaListener(topics = "my-topic", groupId = "my-consumer-group")
    void listen(String m) { ... } }
```

**RabbitMQ** — broker di messaggistica tradizionale, forte su routing ed exchange. Scelta preferibile
quando servono routing complesso e code di lavoro, meno adatto quando serve rileggere la storia.

**Da segnalare in review**
- Topic creati implicitamente dall'auto-create invece che dichiarati esplicitamente: in produzione
  si configurano su Kafka, con partizioni e replica decise.
- Consumer senza `groupId`, o gruppi che si sovrappongono involontariamente.
- Nessuna gestione dei messaggi non processabili: serve una dead letter queue, altrimenti un
  singolo messaggio malformato blocca la partizione.
- Consumer non idempotenti su un broker con consegna at-least-once. Questa è la regola: **assumere
  sempre che un messaggio possa arrivare due volte.**
- Schema dei messaggi senza versione.

**Diagnosi**
- *Messaggi consumati più volte* → è il comportamento normale at-least-once. Deduplicare con una
  chiave di idempotenza lato consumer.
- *Consumer fermo* → rebalance continuo, o partizione assegnata a un'istanza bloccata.
- *Client che non si collega dentro Docker* → `KAFKA_ADVERTISED_LISTENERS` configurato su
  `localhost` invece che sul nome del servizio nella rete Docker. È il punto che rompe più
  spesso le configurazioni Compose.
- *Ordine non rispettato* → l'ordine è garantito **dentro una partizione**, non nel topic. Se
  serve, va scelta una chiave di partizionamento coerente.
