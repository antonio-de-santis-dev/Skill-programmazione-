# Pattern — Outbox e Saga in Java/Spring

Il corso presenta questi pattern con esempi .NET. Qui sono tradotti in Java/Spring: la struttura è
quella del corso, il codice è l'equivalente sulla piattaforma che stai usando.

**Prima di implementare**, verifica che il problema esista davvero (`references/dati.md`). Una
Saga su un'operazione che sta dentro un solo servizio è complessità pura: lì basta
`@Transactional`.

Indice:
1. Il dual write problem
2. Transactional Outbox
3. Il relay
4. CDC come alternativa al polling
5. Saga per coreografia
6. Transazioni compensative
7. Saga per orchestrazione
8. Cosa verificare

---

## 1. Il dual write problem

```java
@Transactional
public void checkout(String userId) {
    orderRepository.save(order);              // 1) database
    kafkaTemplate.send("order.created", ev);  // 2) broker — NON è nella transazione
}
```

Sono due sistemi diversi. Se la 1 riesce e la 2 no, l'ordine esiste e nessuno lo sa. Se la 2 riesce
e la 1 fallisce con rollback, è stato annunciato un fatto mai avvenuto.

Non si risolve invertendo l'ordine né spostando la `send` fuori dalla transazione: si risolve
rendendo **atomica** la scrittura del dato e la registrazione dell'intenzione di pubblicare.

---

## 2. Transactional Outbox

L'evento viene scritto **nella stessa transazione locale** del dato di business, su una tabella
`outbox`. L'atomicità del database garantisce che o ci sono entrambi, o nessuno dei due.

```java
@Entity
@Table(name = "outbox_messages")
public class OutboxMessage {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String type;              // nome completo della classe dell'evento

    @Column(columnDefinition = "TEXT", nullable = false)
    private String payload;           // l'evento serializzato in JSON

    @Column(nullable = false)
    private Instant occurredOn;

    private Instant processedOn;      // null = non ancora pubblicato

    // costruttori e getter
}
```

```java
@Service
public class CheckoutService {

    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;

    @Transactional                                   // UNA transazione per entrambe le scritture
    public void checkout(CheckoutRequest request) throws JsonProcessingException {

        // 1) logica di business
        Order order = orderRepository.save(Order.from(request));

        // 2) l'evento va nella OUTBOX, non sul broker
        OrderCreatedEvent event = new OrderCreatedEvent(
                order.getId().toString(), order.getCustomerId(),
                order.getTotal(), Instant.now(), 1);

        outboxRepository.save(new OutboxMessage(
                OrderCreatedEvent.class.getName(),
                objectMapper.writeValueAsString(event),
                Instant.now()));
        // commit atomico: o entrambe, o nessuna
    }
}
```

Il punto da spiegare quando produci questo codice: **non stiamo pubblicando l'evento, stiamo
registrando che va pubblicato.** La pubblicazione è un passo separato e recuperabile.

---

## 3. Il relay

Un processo in background legge i record non elaborati, li pubblica e li marca.

```java
@Component
public class OutboxProcessor {

    private static final Logger log = LoggerFactory.getLogger(OutboxProcessor.class);

    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final ObjectMapper objectMapper;

    @Scheduled(fixedDelay = 5000)
    @Transactional
    public void publishPending() {
        List<OutboxMessage> pending =
            outboxRepository.findTop100ByProcessedOnIsNullOrderByOccurredOnAsc();

        for (OutboxMessage msg : pending) {
            try {
                Class<?> type = Class.forName(msg.getType());
                Object event = objectMapper.readValue(msg.getPayload(), type);

                kafkaTemplate.send(topicFor(type), event).get();   // attende la conferma

                msg.setProcessedOn(Instant.now());                 // marca DOPO la conferma
            } catch (Exception e) {
                log.error("pubblicazione outbox fallita per id {}", msg.getId(), e);
                break;      // non saltare avanti: si perderebbe l'ordine
            }
        }
    }
}
```

Tre dettagli che fanno la differenza fra un relay corretto e uno che perde messaggi:

- **Marcare dopo la conferma**, mai prima. Se si marca prima e la pubblicazione fallisce, il
  messaggio è perso per sempre.
- **Fermarsi al primo errore** invece di proseguire, altrimenti l'ordine degli eventi si rompe.
- **Limitare il batch** (`findTop100...`) per non caricare in memoria una tabella cresciuta.

**L'ironia da conoscere**: l'Outbox risolve il dual write dell'applicazione ma lo reintroduce nel
relay — pubblica, poi marca; se cade in mezzo, ripubblica. Per questo **i consumer devono essere
idempotenti comunque**. L'Outbox garantisce *at-least-once*, non *exactly-once*.

Serve anche una **purge** dei record già elaborati, altrimenti la tabella cresce indefinitamente:
```java
@Scheduled(cron = "0 0 3 * * *")
@Transactional
public void purge() {
    outboxRepository.deleteByProcessedOnBefore(Instant.now().minus(7, ChronoUnit.DAYS));
}
```

In un servizio **replicato**, `@Scheduled` parte su ogni istanza: serve un lock distribuito
(ShedLock, o un `SELECT ... FOR UPDATE SKIP LOCKED` sulla tabella outbox), altrimenti più repliche
pubblicano gli stessi messaggi.

---

## 4. CDC come alternativa al polling

Il polling ha due difetti: latenza pari all'intervallo, e carico costante sul database anche quando
non c'è nulla da pubblicare.

**Change Data Capture** legge il log delle transazioni del database e pubblica sul broker.
L'applicazione non conosce né Kafka né il meccanismo di pubblicazione: scrive solo nella sua tabella.

- **Debezium** è lo strumento open source di riferimento, con connettori per PostgreSQL, MySQL,
  SQL Server e MongoDB.
- Alcuni database hanno il CDC integrato: CockroachDB Change Feeds, Cosmos DB Change Feed,
  DynamoDB Streams.

Il costo è operativo, non applicativo: serve gestire Debezium e Kafka Connect. Su un sistema
piccolo, il relay con polling è la scelta proporzionata; **dichiaralo** invece di proporre Debezium
per default.

---

## 5. Saga per coreografia

Ogni servizio reagisce agli eventi degli altri. Nessun coordinatore.

```
Ordering  → transazione locale + outbox → pubblica OrderCreated
Payment   → consuma OrderCreated → paga → pubblica PaymentProcessed
                                        ↘ fallisce → pubblica PaymentFailed
Inventory → consuma PaymentProcessed → riserva → pubblica StockReserved
                                              ↘ fallisce → StockReservationFailed
Payment   → consuma StockReservationFailed → RIMBORSA (compensazione)
Ordering  → marca l'ordine come fallito
```

```java
@Service
public class PaymentSagaHandler {

    @KafkaListener(topics = "order.created", groupId = "payment-service")
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        if (processedEvents.existsByEventId(event.orderId())) return;   // idempotenza

        try {
            Payment payment = paymentService.charge(event);
            outbox.save(toOutbox(new PaymentProcessedEvent(event.orderId(), payment.getId())));
        } catch (PaymentDeclinedException e) {
            outbox.save(toOutbox(new PaymentFailedEvent(event.orderId(), e.getReason())));
        }
        processedEvents.save(new ProcessedEvent(event.orderId()));
    }

    // Compensazione: annulla l'effetto di un passo già completato.
    @KafkaListener(topics = "stock.reservation.failed", groupId = "payment-service")
    @Transactional
    public void onStockReservationFailed(StockReservationFailedEvent event) {
        paymentService.refund(event.orderId());     // deve essere idempotente
        outbox.save(toOutbox(new PaymentRefundedEvent(event.orderId())));
    }
}
```

Nota che ogni passo usa **outbox + idempotenza**: sono i due prerequisiti di una Saga, non
opzionali.

**Il difetto della coreografia, da dire esplicitamente**: non esiste un punto del codice dove il
flusso è scritto per intero. Per sapere cosa succede dopo `PaymentProcessed` bisogna cercare chi lo
sottoscrive in tutti i servizi. Su 3-4 passi è gestibile; oltre, diventa incomprensibile.

---

## 6. Transazioni compensative

Una compensazione **non è un rollback**. Il rollback ripristina esattamente lo stato precedente; la
compensazione applica una nuova operazione che ne annulla l'effetto di business.

| Passo | Compensazione | Stato risultante |
|---|---|---|
| Addebita 100€ | rimborsa 100€ | due movimenti visibili, non zero movimenti |
| Riserva 3 pezzi | rilascia 3 pezzi | la riserva è esistita |
| Invia email di conferma | invia email di annullamento | l'utente ha ricevuto entrambe |

Da qui tre requisiti sulle compensazioni:
- Devono essere **idempotenti**: possono arrivare più volte.
- Devono essere **sempre possibili**: se un passo non è compensabile (un'email inviata, un pagamento
  irreversibile), va messo **alla fine** della Saga.
- Devono essere **osservabili**: uno stato intermedio visibile agli altri servizi va modellato nel
  dominio (`PENDING`, `CONFIRMED`, `FAILED`), non nascosto.

Quest'ultimo punto è la conseguenza più importante: **la Saga non garantisce isolamento** — la "I"
di ACID. Durante la Saga l'ordine esiste ma non è ancora pagato, e gli altri lo vedono. Il monolite
lo nascondeva, la Saga no. Va progettato, non scoperto in produzione.

---

## 7. Saga per orchestrazione

Un orchestratore conosce la sequenza, mantiene lo stato e decide il passo successivo.

```java
@Entity
public class OrderSagaState {
    @Id private String sagaId;
    private String orderId;
    @Enumerated(EnumType.STRING) private SagaStep currentStep;
    @Enumerated(EnumType.STRING) private SagaStatus status;
    private Instant lastUpdated;
}

public enum SagaStep { STARTED, PAYMENT_REQUESTED, STOCK_REQUESTED, COMPLETED, COMPENSATING }
```

```java
@Service
public class OrderSagaOrchestrator {

    @Transactional
    public void start(String orderId) {
        var state = new OrderSagaState(UUID.randomUUID().toString(), orderId,
                                       SagaStep.PAYMENT_REQUESTED, SagaStatus.RUNNING);
        sagaRepository.save(state);
        outbox.save(toOutbox(new ChargePaymentCommand(state.getSagaId(), orderId)));
    }

    @KafkaListener(topics = "payment.processed")
    @Transactional
    public void onPaymentProcessed(PaymentProcessedEvent e) {
        OrderSagaState state = sagaRepository.findById(e.sagaId()).orElseThrow();
        state.setCurrentStep(SagaStep.STOCK_REQUESTED);
        outbox.save(toOutbox(new ReserveStockCommand(state.getSagaId(), state.getOrderId())));
    }

    @KafkaListener(topics = "stock.reservation.failed")
    @Transactional
    public void onStockFailed(StockReservationFailedEvent e) {
        OrderSagaState state = sagaRepository.findById(e.sagaId()).orElseThrow();
        state.setCurrentStep(SagaStep.COMPENSATING);
        outbox.save(toOutbox(new RefundPaymentCommand(state.getSagaId(), state.getOrderId())));
    }

    // Una Saga bloccata non si sblocca da sola: serve un timeout.
    @Scheduled(fixedDelay = 60000)
    @Transactional
    public void recoverStuck() {
        sagaRepository.findStuckBefore(Instant.now().minus(5, ChronoUnit.MINUTES))
                      .forEach(this::compensate);
    }
}
```

Nell'orchestrazione i messaggi verso i servizi sono **comandi** (`ChargePayment`), non eventi:
l'orchestratore sa a chi si sta rivolgendo.

| | Coreografia | Orchestrazione |
|---|---|---|
| Accoppiamento | basso | l'orchestratore conosce tutti |
| Flusso leggibile | in nessun punto | centralizzato |
| Punto di fallimento | distribuito | l'orchestratore va reso resiliente |
| Adatta a | 2-4 passi | saghe lunghe o con logica condizionale |

---

## 8. Cosa verificare

Prima di considerare finita un'implementazione Outbox/Saga:

- [ ] Dato di business ed evento scritti nella **stessa** transazione.
- [ ] Il relay marca il record **dopo** la conferma di pubblicazione.
- [ ] Indice sulla colonna di stato dell'outbox (`processedOn`).
- [ ] Purge dei record elaborati.
- [ ] Lock distribuito se il servizio è replicato.
- [ ] Ogni consumer è idempotente, con chiave persistita e vincolo UNIQUE.
- [ ] Ogni passo della Saga ha una compensazione, o è collocato alla fine.
- [ ] Le compensazioni sono idempotenti.
- [ ] Gli stati intermedi sono espliciti nel modello di dominio.
- [ ] Esiste un timeout per Saga e un modo per recuperare quelle bloccate.
- [ ] Esiste un allarme sul topic di dead letter e sulle Saga in stato `COMPENSATING`.
