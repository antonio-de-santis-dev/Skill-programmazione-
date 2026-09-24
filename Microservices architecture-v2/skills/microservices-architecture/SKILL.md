---
name: microservices-architecture
description: Progetta, valuta e corregge sistemi distribuiti - monolite vs monolite modulare vs microservizi, decomposizione e confini dei servizi, database-per-service e ownership dei dati, comunicazione sincrona e asincrona (REST, Feign, gRPC, GraphQL, API Gateway, BFF, service discovery, Kafka, RabbitMQ), CQRS, Saga, Event Sourcing, Outbox, CDC, caching distribuito, resilienza (retry, timeout, circuit breaker), distributed tracing e anti-pattern architetturali. Usa questa skill quando si decide COME due servizi si parlano o CHI possiede un dato, quando si valuta se spezzare un monolite, e anche quando l'utente descrive solo un sintomo - "questa chiamata e' lenta", "i dati non sono allineati fra i due servizi", "se cade un servizio cade tutto", "come gestisco una transazione su piu' servizi" - senza nominare i pattern.
---

# Architettura a microservizi — decisione e progettazione

Questa skill decide **il contratto**: confini, chi possiede quale dato, come i servizi si parlano,
cosa succede quando qualcosa fallisce. L'implementazione dentro il singolo servizio appartiene a
**java-spring-review**.

---

## Il principio che governa tutto

Ogni pattern qui dentro esiste per ricomprare qualcosa che nel monolite era gratis: la transazione
ACID, la join, la chiamata in-process che non fallisce, lo stack trace unico.

Da qui la regola operativa più importante: **non introdurre un pattern prima di aver nominato il
problema concreto che risolve.** Un sistema con Saga, CQRS, event sourcing e service mesh su quattro
servizi e due sviluppatori non è avanzato, è ingestibile.

Quando valuti un'architettura, la domanda utile non è *"usa i pattern giusti?"* ma *"quale problema
ha portato a questa scelta, e quel problema esiste davvero?"*.

---

## Come ragionare su un pattern

Prima di proporne uno, sappi rispondere a tutte e sette queste domande. Se non sai rispondere a
"quando NON usarlo" e "cosa costa", non hai capito il pattern abbastanza da consigliarlo.

1. **Quale problema risolve** — concreto, non teorico.
2. **Quando usarlo** — le condizioni che lo rendono la scelta giusta.
3. **Quando NON usarlo** — altrettanto importante.
4. **Vantaggi**.
5. **Costi** — sviluppo, operatività, competenze richieste al team.
6. **Trade-off** — cosa si perde per ciò che si guadagna.
7. **Conseguenze sugli altri servizi** — nessuna decisione architetturale è locale.

Quando rispondi all'utente, non serve elencare tutti e sette i punti ogni volta. Serve che la
risposta li abbia attraversati, e che **costi e controindicazioni siano dette**, non solo i vantaggi.

---

## Dove trovare cosa

**Criteri di decisione e trade-off** → `references/`

| Argomento | File |
|---|---|
| Monolite, monolite modulare, microservizi, confini, Strangler Fig, scalabilità, serverless | `references/decomposizione.md` |
| Sincrono vs asincrono, REST, Feign, discovery, gateway, BFF, gRPC, GraphQL, Kafka, RabbitMQ | `references/comunicazione.md` |
| Database-per-service, CAP, sharding, CQRS, event sourcing, Saga, Outbox, CDC, caching | `references/dati.md` |
| Retry, circuit breaker, timeout, bulkhead, logging centralizzato, tracing, health check, rilasci | `references/resilienza.md` |

**Codice funzionante Java/Spring** → `patterns/`

| Serve | File |
|---|---|
| Config Server, Eureka, Feign, Gateway, LoadBalancer, Resilience4j: configurazione reale | `patterns/spring-cloud-setup.md` |
| Producer, consumer, topic, Compose del broker | `patterns/messaging-kafka.md` |
| Outbox e Saga implementati in Java/Spring | `patterns/outbox-saga-java.md` |

> Il corso *Design Microservices Architecture* usa esempi .NET. I `patterns/` traducono quei pattern
> in Java/Spring: la struttura è quella del corso, il codice è l'equivalente sulla piattaforma che
> stai usando. Dichiaralo quando citi quel materiale.

---

## Prima di rispondere: cosa devi sapere

Una raccomandazione architetturale data senza questi dati è un'ipotesi travestita da consiglio.
Chiedi ciò che manca, oppure dichiara l'assunzione.

- **Quanti servizi** esistono oggi, e quanti sviluppatori.
- **Esiste una pipeline** di build e deploy automatizzata?
- **Chi possiede quale dato** oggi — e se c'è un database condiviso.
- **Il requisito di coerenza**: l'utente può vedere un dato vecchio di qualche secondo, o no?
- **Volumi e profilo di carico**, almeno l'ordine di grandezza.
- **Dove gira** il sistema: macchine, container, Kubernetes, cloud gestito.

Sulla piattaforma di esecuzione fai attenzione a una sovrapposizione specifica: **su Kubernetes,
`Service` + DNS interno svolgono già il ruolo del service registry.** Proporre Eureka su un sistema
già orchestrato duplica un meccanismo esistente. Vale anche per Config Server (→ `ConfigMap` e
`Secret`) e per il gateway (→ `Ingress`).

---

## Ordine di una review architetturale

1. **Confini** — ogni servizio possiede i propri dati? Può essere rilasciato da solo? Se una delle
   due risposte è no, il resto è secondario: il problema è lì.
2. **Accoppiamento** — quante chiamate sincrone per completare un'operazione utente? Tre servizi al
   99,9% in catena danno 99,7%.
3. **Dati** — chi è la fonte di verità per ogni entità? Cosa succede se una scrittura riesce a metà?
4. **Guasti** — ogni chiamata remota ha un timeout? Cosa succede quando il servizio a valle è
   **lento**, non quando è giù (quello è il caso facile).
5. **Diagnosticabilità** — si può seguire una singola richiesta attraverso tutti i servizi?

---

## Anti-pattern da riconoscere per primi

**Distributed monolith.** Servizi separati che si rilasciano sempre insieme, condividono il database
o si chiamano in catene sincrone. Ha tutti i costi dei microservizi e nessuno dei benefici. Test
diagnostico: *si può rilasciare un servizio senza toccare gli altri?*

**Shared database.** Due servizi sullo stesso schema. Lega i cicli di rilascio e rende ogni
migrazione un evento coordinato fra team.

**Chatty services.** Decine di messaggi per una singola operazione. Il confine passa nel punto
sbagliato: valutare la fusione, o spostare il dato con un evento e una vista materializzata.

**God gateway.** Logica di business nell'API Gateway perché era il posto comodo. Il gateway
instrada, autentica, limita — non decide.

---

## Anti-overengineering — come rispondere a una richiesta sproporzionata

Quando arriva una richiesta del tipo *"mettiamo CQRS, Saga, Kafka e Kubernetes"* su un sistema
piccolo, **non eseguire e non rifiutare**. Fai emergere il problema:

1. Chiedi quale problema concreto ciascuna tecnologia dovrebbe risolvere.
2. Per ognuna che non risolve un problema reale, **dillo e proponi l'alternativa più semplice.**
3. Se l'obiettivo è imparare, dillo pure: costruire un esempio didattico è legittimo, ma va chiamato
   così, non spacciato per una scelta architetturale.

Le sostituzioni più frequenti:

| Proposto | Spesso sufficiente | Condizione per salire |
|---|---|---|
| Microservizi | monolite modulare con schemi separati | parti che scalano diversamente, team indipendenti |
| Kafka | REST sincrono, o eventi solo dove serve disaccoppiare | fan-out reale, replay, disaccoppiamento temporale |
| Saga | transazione locale singola | l'operazione attraversa davvero più servizi |
| CQRS | indici e query dedicate sullo stesso database | letture e scritture con profili di carico opposti |
| Event Sourcing | tabella di audit | serve ricostruire stati passati o la storia è il dominio |
| Kubernetes | Docker Compose, o container gestiti | scaling automatico, molti servizi, team di piattaforma |
| Service mesh | Resilience4j nella libreria | molti servizi in linguaggi diversi |

Una risposta che dice *"qui bastano due servizi che si chiamano in REST, Kafka lo aggiungiamo quando
avremo un terzo consumatore dello stesso evento"* vale più di un'architettura completa.

---

## Contratto con le altre skill

| Situazione | Chi comanda |
|---|---|
| Decidere **se** due servizi comunicano in modo sincrono o asincrono, quale contratto, chi possiede il dato | **questa skill** |
| Scrivere il `@FeignClient`, il `@KafkaListener`, l'entity, il repository, il test | **java-spring-review** |
| Dockerfile, Compose, manifest Kubernetes, pipeline | **docker-kubernetes-devops** |

**Il passaggio di consegne, concretamente**: quando hai deciso il contratto, dichiaralo in modo che
sia implementabile — protocollo, forma del messaggio, chi è il proprietario del dato, cosa succede in
caso di errore, se l'operazione è idempotente. Poi passa all'implementazione.

Esempio di handoff ben fatto:

> **Decisione**: `Order` non chiama `Payment` in modo sincrono. `Order` pubblica
> `OrderCreated` su un topic; `Payment` lo consuma e pubblica `PaymentProcessed` o
> `PaymentFailed`. `Order` possiede lo stato dell'ordine, `Payment` possiede la transazione.
> Coreografia, tre passi, compensazione su fallimento. I consumer devono essere idempotenti perché
> la consegna è at-least-once.
>
> **Implementazione** → java-spring-review, con `patterns/messaging-kafka.md` e
> `patterns/outbox-saga-java.md`.

Nel workflow completo di una nuova applicazione, questa skill viene **per prima**:
architettura → implementazione → test → container → deploy → verifica. Ma "per prima" non significa
"lunga": su un sistema semplice la parte architetturale può essere tre righe.
