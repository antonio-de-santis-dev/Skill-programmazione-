# Pattern — messaggistica con Kafka (e RabbitMQ)

Implementazione della comunicazione asincrona fra servizi Spring Boot.

**Prima di arrivare qui**, verifica di aver bisogno davvero di un broker: `references/comunicazione.md`,
sezione *sincrono o asincrono*. Un secondo servizio che deve solo leggere un dato non giustifica
Kafka.

Indice:
1. Il broker in locale
2. Configurazione Spring
3. Producer
4. Consumer
5. Eventi tipizzati e serializzazione JSON
6. Idempotenza
7. Errori e dead letter
8. RabbitMQ come alternativa
9. Diagnosi

---

## 1. Il broker in locale

```yaml
# docker-compose.yml
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on: [zookeeper]
    ports: ["9092:9092"]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

**Due container, non uno**: nelle versioni tradizionali Kafka richiede Zookeeper, che tiene la
configurazione del cluster, i broker e i topic. Compose avvia prima Zookeeper, poi Kafka.

`KAFKA_ADVERTISED_LISTENERS` è l'indirizzo che Kafka comunica ai client per farsi contattare.
**È il pezzo che rompe più spesso le configurazioni.** Se i client girano sull'host va bene
`localhost:9092`; se girano dentro la stessa rete Docker serve il nome del servizio
(`kafka:9092`), e per entrambi i casi servono due listener distinti.

> Dalla versione 3.3+ Kafka supporta **KRaft** e Zookeeper non è più necessario. Le immagini
> recenti permettono una configurazione a singolo container. Il corso usa la forma con Zookeeper.

---

## 2. Configurazione Spring

Dipendenza: `spring-kafka`.

```properties
spring.kafka.bootstrap-servers=${KAFKA_SERVERS:localhost:9092}

# Producer
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer

# Consumer
spring.kafka.consumer.group-id=order-service
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=com.example.events
```

I messaggi viaggiano come **byte**: i serializer dichiarano come trasformare chiave e valore.
`StringSerializer` per i messaggi testuali semplici (la forma del corso), `JsonSerializer` per gli
oggetti.

`trusted.packages` non è burocrazia: senza, la deserializzazione di tipi arbitrari è un rischio di
sicurezza.

**Dichiara i topic esplicitamente** invece di affidarti alla creazione automatica, che in produzione
crea topic con partizioni e replica di default:
```java
@Bean
public NewTopic orderCreatedTopic() {
    return TopicBuilder.name("order.created").partitions(3).replicas(1).build();
}
```

---

## 3. Producer

```java
package com.example.order.messaging;

import com.example.events.OrderCreatedEvent;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class OrderEventPublisher {

    private static final String TOPIC = "order.created";
    private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;

    public OrderEventPublisher(KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void publish(OrderCreatedEvent event) {
        // La CHIAVE determina la partizione: stessa chiave → stessa partizione → ordine garantito.
        kafkaTemplate.send(TOPIC, event.orderId(), event);
    }
}
```

**L'ordine dei messaggi è garantito dentro una partizione, non nel topic.** Se l'ordine degli eventi
di un ordine conta, la chiave deve essere l'`orderId`. È una decisione di progettazione, non un
dettaglio.

Per sapere se la pubblicazione è andata a buon fine:
```java
kafkaTemplate.send(TOPIC, event.orderId(), event)
    .whenComplete((result, ex) -> {
        if (ex != null) log.error("pubblicazione fallita per {}", event.orderId(), ex);
    });
```

**Attenzione**: pubblicare dentro un metodo `@Transactional` che scrive anche sul database è il
**dual write problem** — le due operazioni sono su sistemi diversi e possono divergere. La soluzione
è il pattern Outbox: vedi `outbox-saga-java.md`.

---

## 4. Consumer

```java
package com.example.payment.messaging;

import com.example.events.OrderCreatedEvent;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

@Service
public class OrderCreatedListener {

    private final PaymentService paymentService;

    public OrderCreatedListener(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    @KafkaListener(topics = "order.created", groupId = "payment-service")
    public void onOrderCreated(OrderCreatedEvent event) {
        paymentService.processPayment(event);      // deve essere IDEMPOTENTE
    }
}
```

**Il `groupId` determina il comportamento del fan-out**, ed è la cosa da capire per prima:
- Istanze con lo **stesso** groupId si **dividono** le partizioni: ogni messaggio va a una sola
  istanza. È così che si scala un consumer.
- Servizi con groupId **diversi** ricevono **tutti** lo stesso messaggio. È così che funziona il
  publish/subscribe fra servizi diversi.

Un groupId sbagliato non dà errore: produce silenziosamente il comportamento opposto a quello
voluto.

---

## 5. Eventi tipizzati

```java
package com.example.events;

import java.math.BigDecimal;
import java.time.Instant;

// Nome al PASSATO: constata un fatto avvenuto, non ordina un'azione.
public record OrderCreatedEvent(
    String orderId,
    String customerId,
    BigDecimal totalAmount,
    Instant occurredOn,
    int version                 // versiona lo schema fin dall'inizio
) { }
```

Il nome dell'evento è una decisione di accoppiamento, non di stile: `OrderCreated` non presume chi
lo consumerà, `CreateOrder` sì.

Lo schema di un evento pubblicato su un event bus è **un contratto pubblico**. Va versionato come
un'API: aggiungere campi opzionali è compatibile, rimuoverli o rinominarli no.

Distingui sempre:
- **domain event** — interno al servizio, in-process, granulare;
- **integration event** — sull'event bus, parte del contratto verso gli altri servizi.

Confonderli è l'errore strutturale più comune: espone il modello interno al resto del sistema.

---

## 6. Idempotenza

Kafka garantisce **at-least-once**: un messaggio può arrivare due volte. Non è un caso eccezionale
da gestire "se capita" — è il comportamento normale, e il consumer deve reggerlo per costruzione.

```java
@Service
public class PaymentService {

    @Transactional
    public void processPayment(OrderCreatedEvent event) {
        // La chiave di idempotenza va persistita con un vincolo UNIQUE:
        // è il database a garantire che il secondo tentativo non passi.
        if (processedEventRepository.existsByEventId(event.orderId())) {
            log.debug("evento {} già processato, ignorato", event.orderId());
            return;
        }
        charge(event);
        processedEventRepository.save(new ProcessedEvent(event.orderId(), Instant.now()));
    }
}
```

Il controllo e la scrittura devono stare **nella stessa transazione**, altrimenti due consegne
concorrenti passano entrambe il controllo.

---

## 7. Errori e dead letter

Senza gestione degli errori, un singolo messaggio malformato blocca la partizione all'infinito.

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template);   // → topic ".DLT"
    var backoff = new ExponentialBackOff(1000L, 2.0);
    return new DefaultErrorHandler(recoverer, backoff);
}
```

Dopo i tentativi configurati il messaggio finisce in un topic di dead letter, la partizione riparte,
e il messaggio problematico resta disponibile per l'analisi.

Un topic DLT che nessuno guarda è inutile: serve un allarme sul suo riempimento.

---

## 8. RabbitMQ come alternativa

| | Kafka | RabbitMQ |
|---|---|---|
| Modello | log persistente di eventi | broker di messaggi con routing |
| Rilettura della storia | sì, dall'offset | no, il messaggio consumato sparisce |
| Routing | per topic e partizione | exchange con regole ricche |
| Adatto a | event streaming, event sourcing, Saga | code di lavoro, routing complesso |

```java
@RabbitListener(queues = "order.created.queue")
public void onOrderCreated(OrderCreatedEvent event) { ... }
```

La scelta segue il caso d'uso: se serve **rileggere** gli eventi, Kafka; se serve **instradare**
messaggi con regole, RabbitMQ.

---

## 9. Diagnosi

| Sintomo | Causa |
|---|---|
| Il client non si collega dentro Docker | `KAFKA_ADVERTISED_LISTENERS` su `localhost` invece del nome di servizio |
| Il consumer non riceve nulla | groupId con offset già avanzato; prova `auto-offset-reset=earliest` su un gruppo nuovo |
| Ogni istanza riceve lo stesso messaggio | groupId diversi dove ne serviva uno solo |
| Una sola istanza lavora, le altre no | più istanze che partizioni: aumenta le partizioni |
| Messaggi elaborati due volte | comportamento normale at-least-once: manca l'idempotenza |
| Ordine non rispettato | l'ordine vale per partizione — serve una chiave coerente |
| Il consumer si blocca su un messaggio | manca l'error handler con dead letter |
| Errore di deserializzazione | `trusted.packages` non configurato, o schema cambiato senza versione |
