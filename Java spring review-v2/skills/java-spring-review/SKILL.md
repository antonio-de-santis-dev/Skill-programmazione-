---
name: java-spring-review
description: Progetta, implementa, rifattorizza, testa, debugga e revisiona codice Java (8-25) e Spring Boot. Copre controller, service, repository, entity, DTO, mapping, validation, exception handling, Spring Data JPA, Hibernate, transazioni, configurazione e profili, concorrenza, sicurezza applicativa, JUnit, Mockito, MockMvc, refactoring e qualità del codice. Usa questa skill ogni volta che si scrive o si tocca codice Java o Spring Boot - "crea un CRUD", "aggiungi un endpoint", "scrivi l'entity", "questo test non passa", "perché ottengo LazyInitializationException", "guarda questo codice", "come lo miglioro" - sia per implementare da zero sia per valutare codice esistente, anche quando l'utente non nomina Java o Spring esplicitamente ma il contesto e' un progetto Maven/Gradle con Spring Boot.
---

# Java & Spring Boot — implementazione, refactoring, review

Questa skill serve a **scrivere software**, non solo a giudicarlo. La review è una delle sei
modalità, non l'unica.

---

## Passo 0 — Contesto e versione (sempre, prima di scrivere codice)

Non generare mai codice per un progetto esistente senza aver prima guardato cosa c'è. Le decisioni
tecniche dipendono da fatti che vanno letti, non assunti.

Se hai accesso al progetto, leggi in quest'ordine e fermati appena hai quello che ti serve:

1. `pom.xml` / `build.gradle` / `build.gradle.kts` → **versione Java**, versione Spring Boot,
   starter presenti, Lombok sì o no, MapStruct sì o no.
2. `application.yml` / `application.properties` e i file per profilo → datasource, porta,
   `ddl-auto`, profili esistenti.
3. Due o tre classi dello stesso tipo di quella che stai per scrivere → **convenzioni reali** del
   progetto: struttura dei package, naming, iniezione via costruttore o Lombok, uso di DTO, stile
   dei test.
4. Le classi correlate a quella che tocchi → non reinventare un'entity, un DTO o un endpoint che
   esiste già.

Poi consulta `references/java-versions.md` per verificare che le funzionalità che stai per usare
siano disponibili in quella versione.

**Se non riesci a determinare la versione, dichiara l'assunzione invece di inventarla.** Per esempio:
"assumo Java 17 e Spring Boot 3.x; se sei su Java 8 dimmelo, perché `record` e `var` non sono
disponibili e il codice cambia". Non è una formalità: generare un `record` su un progetto Java 8 è
codice che non compila.

**Adatta il codice al progetto, non il progetto a un template.** Se il progetto usa
`@Autowired` su campo ovunque, segnalalo una volta come miglioramento possibile ma scrivi il codice
nello stile esistente, a meno che l'utente non stia chiedendo proprio di rifattorizzare.

---

## Le sei modalità

Riconosci quale ti viene chiesta e comportati di conseguenza. Se la richiesta è ambigua, la
modalità di default è **IMPLEMENT**: in caso di dubbio l'utente vuole codice che funziona, non una
descrizione di cosa dovrebbe fare.

### IMPLEMENT — "crea", "aggiungi", "scrivi", "fammi un endpoint"
Produci codice completo e compilabile, non pseudocodice né elenchi di passi. Segui il formato di
output più in basso. Parti da `patterns/` per non riscrivere da zero strutture già canoniche.

### REFACTOR — "migliora", "pulisci", "questo non mi piace"
Cambia una cosa per volta e spiega perché. Non riscrivere ciò che funziona solo per uniformarlo al
tuo gusto. Preserva il comportamento osservabile: se non ci sono test che lo verificano, dillo e
proponi di scriverli prima.

### DEBUG — "non funziona", "ottengo questa eccezione", "il test fallisce"
Parti dal sintomo, non dalla soluzione. Chiedi lo stack trace completo se manca. Le sezioni
**Diagnosi** dei reference sono organizzate proprio per sintomo → causa. Formula un'ipotesi, indica
come verificarla, e solo dopo proponi la correzione.

### REVIEW — "guarda questo codice", "va bene?", "fai una code review"
Usa l'ordine per gravità più in basso. Marca esplicitamente ogni rilievo come **blocca** /
**da sistemare** / **spunto**: senza questa distinzione chi legge non sa cosa deve correggere.

### TEST — "scrivi i test", "come testo questo"
Vai a `patterns/testing-recipes.md`. Copri sempre almeno un percorso di errore, non solo quello
felice.

### EXPLAIN — "come funziona", "perché si fa così", "spiegami"
Spiega il meccanismo e il motivo, con un esempio minimo. Non produrre file: la risposta sta nella
conversazione.

---

## Dove trovare cosa

**Concetti, criteri, cosa verificare** → `references/`

| Argomento | File |
|---|---|
| Quale funzionalità Java è disponibile in quale versione | `references/java-versions.md` |
| Lambda, stream, collections, generics, record, sealed, pattern matching, moduli, I/O | `references/java-core.md` |
| Thread, executor, virtual thread, immutabilità, injection, input non fidato | `references/java-concurrency-security.md` |
| Livelli, REST, JPA, transazioni, validazione, errori, configurazione, Actuator, WebFlux | `references/spring-boot.md` |
| Test, Postman, code smell, Git, Lombok | `references/testing-quality.md` |

**Codice pronto da adattare** → `patterns/`

| Serve | File |
|---|---|
| Un CRUD completo: entity → repository → DTO → mapper → service → controller → advice → test | `patterns/crud-vertical-slice.md` |
| Validazione, gestione errori, transazioni, profili, Actuator, HATEOAS, query custom | `patterns/spring-recipes.md` |
| Test di controller, service e repository | `patterns/testing-recipes.md` |

I `patterns/` contengono codice funzionante da **adattare**, non da copiare alla lettera: nomi,
package e tipi vanno allineati al progetto reale.

---

## Formato di output quando implementi

Quando produci un'implementazione, segui questa struttura. Salta i punti che non si applicano —
per un singolo metodo aggiunto non serve il comando di build.

1. **Cosa stiamo implementando** — una o due righe.
2. **File coinvolti**, con il percorso completo di ognuno.
3. **Il codice**, completo per i file nuovi, come modifica precisa e localizzata per quelli esistenti.
4. **Le decisioni non ovvie** e il perché. Solo quelle: non commentare ogni riga.
5. **Configurazione** necessaria (property, dipendenze da aggiungere al `pom.xml`).
6. **Test**.
7. **Come compilare ed eseguire**.
8. **Come verificare** che funzioni — chiamata `curl`, richiesta Postman, query SQL.
9. **Risultato atteso**.
10. **Cosa controllare se non funziona**.

Se l'utente sta imparando la tecnologia, procedi **in modo incrementale**: un pezzo che funziona,
verificato, poi il successivo. Sommergerlo di dodici file insieme è il modo più rapido per
perderlo.

---

## Ordine di una review

Costruito perché ciò che blocca il merge emerga prima delle osservazioni di stile.

1. **Sicurezza e correttezza** — injection, segreti committati, risorse non chiuse, stato condiviso
   non protetto, `@Transactional` che non ha effetto. Questi bloccano.
2. **Confini architetturali** — logica nel posto giusto, entity non esposte, iniezione via
   costruttore verso interfacce.
3. **Contratto esterno** — semantica HTTP, status code, input validato, errori centralizzati.
4. **Manutenibilità** — duplicazione, responsabilità multiple, nomi, magic number.
5. **Verificabilità** — test sui percorsi di errore, non solo su quello felice.

Un commento utile dice **cosa** non va, **perché** è un problema e **cosa fare**. Senza il perché
chi legge impara solo a obbedire; senza il cosa fare il commento sposta il lavoro senza aiutare.

---

## Quando passare la palla

Questa skill possiede tutto ciò che sta **dentro un singolo servizio**. Appena il problema
attraversa un confine di processo, serve un'altra skill.

| Segnale nella richiesta | Cosa fare |
|---|---|
| "questo servizio deve chiamare quell'altro", scelta fra REST/Feign/gRPC/eventi, coerenza dei dati fra servizi, Saga, CQRS, Outbox, circuit breaker, service discovery, gateway | Consulta **microservices-architecture** per decidere il contratto, poi torna qui per implementarlo |
| "containerizza", "scrivi il Dockerfile", "mettilo su Kubernetes", "la pipeline" | Passa a **docker-kubernetes-devops** |
| "progettiamo da zero un'applicazione a microservizi" | Workflow completo: **architettura → implementazione (qui) → test (qui) → container → deploy → verifica** |

La divisione di lavoro con microservices-architecture è netta: **loro decidono il contratto di
comunicazione, questa skill lo implementa.** Se ti viene chiesto di scrivere un `@FeignClient`
senza che nessuno abbia deciso se la chiamata debba essere sincrona, la domanda architetturale
viene prima.

---

## Anti-overengineering

La complessità va proporzionata al problema. Non introdurre, se non serve davvero:

- Un'interfaccia per ogni service quando esiste una sola implementazione e nessun test la sostituisce.
- Un DTO identico all'entity campo per campo, quando l'entity non esce dal servizio.
- Un mapper generato quando i campi sono tre.
- Astrazioni sui repository di Spring Data, che è già un'astrazione.
- Un `@Transactional` su ogni metodo, incluse le letture semplici.
- Pattern architetturali dentro un singolo servizio che non ne ha bisogno.

Se una soluzione più semplice basta, **dillo esplicitamente** invece di implementare quella
complessa in silenzio. "Qui basta un metodo nel service, non serve un factory" è una risposta
migliore di un factory.
