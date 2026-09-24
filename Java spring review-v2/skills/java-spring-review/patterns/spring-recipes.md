# Pattern — ricette Spring Boot

Frammenti di implementazione per i problemi ricorrenti, oltre al CRUD di base
(`crud-vertical-slice.md`). Baseline Java 17 + Spring Boot 3.x.

Indice:
1. Transazioni: propagazione e isolamento
2. Query personalizzate e ricerca
3. Paginazione e ordinamento
4. Profili e configurazione per ambiente
5. Actuator e informazioni dinamiche
6. Logging
7. HATEOAS
8. Concorrenza dentro un servizio
9. WebFlux

---

## 1. Transazioni: propagazione e isolamento

```java
@Service
public class OrderService {

    private final PaymentService paymentService;
    private final AuditService auditService;

    // REQUIRED (default): si aggancia alla transazione del chiamante, o ne crea una.
    @Transactional
    public void placeOrder(OrderDTO dto) {
        saveOrder(dto);
        paymentService.charge(dto);       // stessa transazione: rollback insieme
        auditService.log(dto);            // vedi sotto: transazione separata
    }
}

@Service
public class AuditService {

    // REQUIRES_NEW: sospende quella del chiamante e ne apre una indipendente.
    // L'audit sopravvive anche se l'ordine fallisce e viene annullato.
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(OrderDTO dto) { ... }
}
```

| Propagazione | Comportamento |
|---|---|
| `REQUIRED` (default) | partecipa a quella esistente, o ne crea una |
| `REQUIRES_NEW` | sospende quella esistente, ne apre una indipendente |
| `SUPPORTS` | partecipa se c'è, altrimenti gira senza transazione |
| `MANDATORY` | errore se non ce n'è già una |
| `NEVER` | errore se ne trova una |

Isolamento: alzarlo riduce le anomalie (dirty read, non-repeatable read, phantom read) e aumenta la
contesa. `SERIALIZABLE` **non è un default prudente**, è una scelta costosa da giustificare.

```java
@Transactional(propagation = Propagation.REQUIRES_NEW,
               isolation = Isolation.SERIALIZABLE)
```

**Rollback**: per default avviene solo su eccezioni *unchecked*. Per le checked serve dichiararlo:
```java
@Transactional(rollbackFor = IOException.class)
```

**La trappola del proxy** (vale la pena ripeterla): l'annotazione non ha effetto su metodi privati
né su chiamate interne alla stessa classe. Se serve, estrai il metodo in un altro bean.

---

## 2. Query personalizzate e ricerca

```java
public interface BookRepository extends JpaRepository<Book, Long> {

    // Ricerca parziale: la derived query non basta, serve LIKE.
    @Query("select b from Book b where lower(b.authorName) like lower(concat('%', :name, '%'))")
    List<Book> searchByAuthor(@Param("name") String name);

    // SQL nativo quando servono funzioni specifiche del database.
    @Query(value = "select * from books where published_year > :year", nativeQuery = true)
    List<Book> publishedAfter(@Param("year") int year);

    // Update con parametri posizionali (la forma usata nel corso).
    @Modifying
    @Transactional
    @Query("update Book b set b.title = ?1, b.authorName = ?2 where b.isbn = ?3")
    int updateBook(String title, String author, String isbn);
}
```

Parametri **nominati** (`:name` + `@Param`) o **posizionali** (`?1`): i primi sono più leggibili e
resistono al riordino degli argomenti. Preferiscili.

Filtro opzionale su un endpoint — due mapping distinti sullo stesso path, discriminati dal
parametro:
```java
@GetMapping(path = "/search", params = "author")
public ResponseEntity<List<BookDTO>> byAuthor(@RequestParam String author) { ... }

@GetMapping(path = "/search", params = "year")
public ResponseEntity<List<BookDTO>> byYear(@RequestParam int year) { ... }
```

---

## 3. Paginazione e ordinamento

Una collezione senza paginazione è un incidente che aspetta il primo cliente con molti dati.

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    Page<Product> findByDescriptionContainingIgnoreCase(String q, Pageable pageable);
}

@GetMapping
public ResponseEntity<Page<ProductDTO>> list(
        @RequestParam(defaultValue = "") String q,
        @PageableDefault(size = 20, sort = "code") Pageable pageable) {
    return ResponseEntity.ok(
        repository.findByDescriptionContainingIgnoreCase(q, pageable).map(ProductMapper::toDto));
}
```

Chiamata: `GET /products?q=tast&page=0&size=20&sort=unitPrice,desc`

---

## 4. Profili e configurazione per ambiente

```
src/main/resources/
├── application.yml            # comune a tutti
├── application-dev.yml        # H2, ddl-auto update, log DEBUG
├── application-test.yml
└── application-prod.yml       # database reale, ddl-auto validate, log INFO
```

```yaml
# application-prod.yml
spring:
  datasource:
    url: ${DB_URL}                    # dall'ambiente, MAI committato
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
```

```bash
java -jar app.jar --spring.profiles.active=prod
# oppure
SPRING_PROFILES_ACTIVE=prod java -jar app.jar
```

```java
@Value("${messaggio:benvenuto}")     // valore di default dopo i due punti
private String messaggio;

@Bean
@Profile("dev")                       // bean attivo solo in un profilo
public DataInitializer devData() { ... }
```

Precedenza (dalla più bassa alla più alta): file `application.yml` → file per profilo → variabile
d'ambiente → argomento di riga di comando. Quando una property "non viene letta", è quasi sempre
qualcosa più in alto in questa scala che la sovrascrive.

Ambienti tipici: **sviluppo** (servizi esterni simulati, es. un finto gateway di pagamento che
risponde sempre ok), **test**, **collaudo**, **produzione**. Ciò che cambia sono endpoint, pool e
livelli di log, non la struttura.

---

## 5. Actuator e informazioni dinamiche

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when-authorized
  info:
    env:
      enabled: true
```

Informazioni dinamiche in `/actuator/info` (per esempio un conteggio da database):

```java
@Component
public class ProductInfoContributor implements InfoContributor {

    private final ProductRepository repository;

    public ProductInfoContributor(ProductRepository repository) {
        this.repository = repository;
    }

    @Override
    public void contribute(Info.Builder builder) {
        Map<String, Object> data = new HashMap<>();
        data.put("totalProducts", repository.count());
        builder.withDetail("catalog", data);
    }
}
```

Health indicator custom, per dichiarare che una dipendenza esterna è giù:

```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        return isReachable()
            ? Health.up().build()
            : Health.down().withDetail("reason", "payment gateway unreachable").build();
    }
}
```

**Non esporre `*`** e non lasciare gli endpoint senza autenticazione: `/actuator/env` rivela la
configurazione, `/actuator/beans` l'intera struttura dell'applicazione.

---

## 6. Logging

```java
private static final Logger log = LoggerFactory.getLogger(ProductService.class);

log.info("Prodotto {} aggiornato da {}", code, user);       // placeholder, non concatenazione
log.error("Salvataggio fallito per {}", code, ex);          // l'eccezione va per ultima, senza {}
```

La concatenazione (`"Prodotto " + code`) costruisce la stringa anche quando il livello è
disabilitato. Con i placeholder no.

```yaml
logging:
  level:
    root: INFO
    com.example.demo: DEBUG
    org.hibernate.SQL: DEBUG          # solo in sviluppo
```

Mai nei log: password, token, numeri di carta, dati personali.

---

## 7. HATEOAS

Porta l'API a un livello REST più completo: la risposta include i link per gli stati successivi,
così il client non costruisce URL a mano. Da qui il nome — *Representational State Transfer*.

Dipendenza: `spring-boot-starter-hateoas`.

```java
public class ProductDTO extends RepresentationModel<ProductDTO> {
    // ... campi ...
}
```

```java
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

@GetMapping
public ResponseEntity<List<ProductDTO>> getAll() {
    List<ProductDTO> products = service.findAll();
    for (ProductDTO p : products) {
        Link self = linkTo(methodOn(ProductController.class).getByCode(p.getCode()))
                        .withSelfRel();
        p.add(self);
    }
    return ResponseEntity.ok(products);
}
```

Nota che un `record` non può estendere `RepresentationModel`: con HATEOAS il DTO torna a essere una
classe. È un buon esempio del fatto che la scelta del tipo dipende dai requisiti, non dalla moda.

**Quando vale la pena**: API pubbliche o consumate da client che non controlli. Su un'API interna
consumata da un frontend che rilasci insieme, aggiunge peso senza beneficio proporzionato.

---

## 8. Concorrenza dentro un servizio

```java
@Service
public class ReportService {

    // Spring gestisce il pool: non creare thread a mano.
    @Async
    public CompletableFuture<Report> generate(String id) {
        return CompletableFuture.completedFuture(heavyWork(id));
    }
}

@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(4);
        ex.setMaxPoolSize(8);
        ex.setQueueCapacity(100);
        ex.setThreadNamePrefix("report-");
        ex.initialize();
        return ex;
    }
}
```

`@Async` ha la **stessa trappola del proxy** di `@Transactional`: non funziona su chiamate interne
alla stessa classe.

Su **Java 21 + Spring Boot 3.2+** i virtual thread si abilitano con una property, senza toccare il
codice:
```yaml
spring:
  threads:
    virtual:
      enabled: true
```
Ha senso su carichi **I/O-bound**. Su carichi CPU-bound non porta nulla.

Task schedulati:
```java
@Scheduled(cron = "0 0 2 * * *")     // ogni notte alle 2
public void cleanup() { ... }
```
In un servizio replicato, `@Scheduled` parte su **ogni** replica. Se l'operazione non è idempotente,
serve un lock distribuito o un solo nodo designato.

---

## 9. WebFlux

Da usare quando il collo di bottiglia è l'attesa di I/O, non la CPU, e quando **tutto** lo stack è
non bloccante. Un repository JDBC dentro un servizio WebFlux annulla il beneficio.

```java
@RestController
public class EmployeeController {

    private final EmployeeRepository repository;

    @GetMapping("/employees/{id}")
    public Mono<Employee> getById(@PathVariable String id) {     // 0 o 1 elemento
        return repository.findById(id);
    }

    @GetMapping("/employees")
    public Flux<Employee> getAll() {                              // 0 o N elementi
        return repository.findAll();
    }
}
```

Lato client, `WebClient`:
```java
WebClient client = WebClient.create("http://localhost:8080");

Mono<Employee> mono = client.get()
        .uri("/employees/{id}", id)
        .retrieve()
        .bodyToMono(Employee.class);

mono.subscribe(
    employee -> log.info("ricevuto {}", employee),   // successo
    error    -> log.error("errore", error),          // errore: non ometterlo mai
    ()       -> log.info("completato"));             // completamento
```

**Niente accade finché non c'è una `subscribe`** (o finché non è Spring a sottoscrivere restituendo
il `Mono` dal controller). Un `Mono` creato e mai sottoscritto è codice morto che non dà errore.
