# Spring Boot — standard di sviluppo e review

Indice:
1. Architettura a livelli e dependency injection
2. API REST
3. Persistenza con Spring Data JPA
4. Gestione errori e validazione
5. DTO e mapping
6. Configurazione e profili
7. Transazioni
8. Actuator e logging
9. WebFlux

---

## 1. Architettura a livelli e dependency injection — *Base*

**Struttura attesa**

```
controller/   endpoint REST, nessuna logica di business
service/      logica di business e validazioni, interfaccia + implementazione
repository/   accesso ai dati
entity/       mapping delle tabelle
dto/          contratto verso l'esterno
exception/    eccezioni di dominio + handler globale
mapper/       conversione entity ↔ DTO
```

**Da scrivere così**
- Stereotipi corretti: `@RestController`, `@Service`, `@Repository`, `@Component`. Ognuno è un
  `@Component` specializzato: comunica il ruolo sia al framework sia a chi legge.
- Iniezione **via costruttore** e verso l'**interfaccia**, non verso l'implementazione. Il
  controller dipende da `ICarService`, non da `CarServiceImpl`.
- Configurazione letta con `@Value("${chiave:default}")`, con un default quando ha senso.

**Da segnalare in review**
- Logica di business nel controller o nel repository.
- `new` su un collaboratore che dovrebbe essere un bean.
- Field injection (`@Autowired` sul campo): rende la classe non istanziabile nei test senza Spring
  e nasconde le dipendenze obbligatorie.
- Classi fuori dal package della classe `@SpringBootApplication`, quindi mai scansionate.

**Diagnosi**
- *`NoSuchBeanDefinitionException`* → classe non annotata, o fuori dal perimetro del component scan.
- *Dipendenza circolare fra bean* → non risolverla con `@Lazy`: è il segnale di un confine di
  responsabilità sbagliato, estrarre la parte condivisa.
- *Campo `@Value` nullo* → property assente e nessun default dichiarato.

---

## 2. API REST — *Intermedio*

**Le regole di design**

1. **Sostantivi per le risorse, verbi HTTP per le azioni.** `/cars`, non `/getCars`.
2. **Verbo giusto sulla risorsa giusta.** `POST` sulla collezione (`/cars`), `PUT` e `DELETE` sulla
   singola (`/cars/{regNo}`). `GET` non modifica mai lo stato. `PUT` è idempotente.
3. **JSON per le rappresentazioni.**
4. **Niente annidamento profondo.** Invece di `/customers/1/orders/2/items/3`, due chiamate o un
   identificatore di relazione.

**Path variable o request parameter**

| | Dove sta | Esempio | Serve per |
|---|---|---|---|
| `@PathVariable` | dentro il path | `/cars/241G1` | identificare la risorsa |
| `@RequestParam` | dopo il `?` | `/cars/car?brandName=Tesla` | filtri, opzioni, ricerche |

**Da scrivere così**
- `ResponseEntity` per controllare status e header. Su una creazione: `201 Created` con l'header
  `Location` costruito con `UriComponentsBuilder`.
- Status coerenti: `200` lettura, `201` creazione, `204` cancellazione senza corpo, `400` input non
  valido, `404` non trovato, `409` conflitto.
- `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping` — la forma breve
  è quella normale. `@RequestMapping` resta utile a livello di classe per il prefisso, e per i verbi
  senza scorciatoia: `@RequestMapping(method = RequestMethod.OPTIONS)`.
- Per `OPTIONS`, dichiarare esplicitamente l'header `Allow` con i verbi realmente supportati: il
  default di Spring non è sempre corretto dal punto di vista semantico.
- Versionare il contratto prima che serva rompere qualcosa.
- HATEOAS quando ha senso: il DTO estende `RepresentationModel`, i link si costruiscono con
  `linkTo(methodOn(...))` di `WebMvcLinkBuilder`.

**Da segnalare in review**
- Verbi nell'URL.
- Tutto che risponde `200`, anche creazioni ed errori.
- Entity JPA restituite direttamente.
- Collezioni senza paginazione.
- Breaking change (campo rimosso o rinominato) senza versione.

**Diagnosi**
- *Parametro sempre nullo* → mismatch fra il nome dichiarato nell'annotazione e quello inviato dal
  client. `@RequestParam("brandName")` richiede esattamente `brandName`, non `brandname`.
- *`405 Method Not Allowed`* → mapping assente per quel verbo.
- *Client rotti dopo un deploy* → contratto modificato senza versionare.

---

## 3. Persistenza con Spring Data JPA — *Intermedio*

**Da scrivere così**
- Entity con `@Entity`, `@Id`, `@GeneratedValue(strategy = IDENTITY)`, e `@Column(name = "...")`
  **solo** quando il nome della proprietà differisce da quello della colonna.
- `interface XRepository extends JpaRepository<X, Long>`: i CRUD arrivano gratis.
- **Derived query** per i casi semplici: `findByRegNo`, `findAllByCarType`, `deleteByIsbn`. Il nome
  del metodo deve rispecchiare le proprietà dell'entità, non i nomi delle colonne.
- `@Query` (JPQL o SQL nativo) quando la derived query diventerebbe illeggibile o non basta.
  Su insert/update/delete servono anche `@Modifying` e `@Transactional`.
- `Optional<T>` come ritorno delle ricerche singole: o contiene il valore, o è vuoto, mai `null`.

**Da segnalare in review**
- `@Modifying` dimenticata su `@Query` di scrittura.
- `ddl-auto: update` o `create` in profili non locali.
- `@Data` di Lombok su entity JPA: genera `equals`/`hashCode` su tutti i campi, incluse le relazioni
  lazy, con effetti imprevedibili. Sulle entity usare `@Getter`, `@Setter` e
  `@ToString(exclude = ...)` separati.
- Relazioni `EAGER` di default.
- Nomi di metodi derivati lunghissimi: lì serve `@Query`.

**Diagnosi**
- *Query derivata che non parte all'avvio* → il nome del metodo non corrisponde alle proprietà
  dell'entità. L'errore arriva al boot, non a runtime: è un vantaggio, non un difetto.
- *Colonna non trovata su nomi riservati* (`year`, `order`, `group`) → usare `@Column` con il
  quoting, oppure rinominare la proprietà.
- *Rallentamenti progressivi su una lista* → problema N+1. Attivare `show-sql: true` e contare le
  query effettive; risolvere con fetch join o proiezione.

---

## 4. Gestione errori e validazione — *Intermedio*

**Da scrivere così**
- Eccezioni di dominio specifiche: `CarNotFoundException`, `CarRegNoAlreadyExistsException`,
  `CarNumbersDoNotMatchException`. Il nome deve dire cosa è successo nel dominio.
- Un solo punto di traduzione: classe `@RestControllerAdvice` che estende
  `ResponseEntityExceptionHandler`, con un `@ExceptionHandler` per tipo, che costruisce sempre lo
  stesso `ErrorResponse`.
- Validazione dichiarativa sui DTO (`@NotBlank`, `@Email`, `@Size`, `@Min`) **più** `@Valid` sul
  parametro del controller. Senza `@Valid` le annotazioni non scattano: è l'errore più frequente.
- Intercettare `MethodArgumentNotValidException` per restituire la mappa campo → messaggio invece
  del `400` grezzo di default.

```java
@PostMapping("/users")
ResponseEntity<String> addUser(@Valid @RequestBody User user) { ... }

@ResponseStatus(HttpStatus.BAD_REQUEST)
@ExceptionHandler(MethodArgumentNotValidException.class)
public Map<String, String> handleValidation(MethodArgumentNotValidException ex) { ... }
```

**Da segnalare in review**
- `try`/`catch` ripetuti in ogni metodo del controller.
- `RuntimeException` o `Exception` generiche lanciate al posto di eccezioni di dominio.
- Stack trace o messaggi interni restituiti al client.
- Vincoli di validazione presenti ma `@Valid` assente.
- Status scelti a caso: `500` per un input non valido è un errore di attribuzione della colpa.

**Diagnosi**
- *Validazione che non scatta* → manca `@Valid`, o manca la dipendenza di validation.
- *Handler ignorato* → la classe advice è fuori dal component scan, oppure un handler più generico
  ha la precedenza.
- *`500` invece di `404`/`409`* → manca il mapping dell'eccezione di dominio nell'advice.

---

## 5. DTO e mapping — *Intermedio*

Perché il DTO: esporre l'entity rivela la struttura del database e la chiave primaria (con una
chiave primaria nota è più facile attaccare il DB), e lega il contratto pubblico allo schema.

**Da scrivere così**
- DTO senza chiave primaria e senza campi interni.
- Tre strategie di mapping, da scegliere consapevolmente:

| Strategia | Pro | Contro |
|---|---|---|
| Mapper manuale | controllo totale, zero dipendenze | verboso, si dimenticano i campi nuovi |
| ModelMapper | configurazione minima (`@Bean` + `map(obj, Dto.class)`) | riflessione a runtime, più lento |
| MapStruct | codice generato a compile time, veloce | setup iniziale più laborioso (`@Mapper`) |

In sintesi: ModelMapper per comodità, MapStruct per prestazioni. Su Java 17+ un `record` è spesso
il DTO migliore.

**Da segnalare in review**
- Entity restituita dal controller.
- DTO che replica l'entità campo per campo senza alcuna differenza: segnale che manca un vero
  confine, o che il DTO è cargo cult.
- ModelMapper su percorsi ad alto volume.
- Relazione bidirezionale esposta nel DTO: la serializzazione va in loop.

---

## 6. Configurazione e profili — *Base*

**Da scrivere così**
- `application.yml` per il comune, un file per profilo per le differenze.
- Attivazione da riga di comando o variabile d'ambiente:
  `java -jar app.jar --spring.profiles.active=test`
- Credenziali fuori dal repository, sempre.
- Ambienti tipici: sviluppo (servizi esterni simulati), test, collaudo, produzione. Ciò che cambia
  sono endpoint, pool, livelli di log — non la struttura.

**Da segnalare in review**
- URL, password e chiavi committati.
- Valori di produzione come default nel file principale.
- Property presenti in un profilo e assenti in un altro.

**Diagnosi**
- *Comportamento diverso fra ambienti* → stampare i profili attivi all'avvio e verificare da quale
  sorgente arriva davvero la property (file, variabile d'ambiente, argomento: la precedenza cresce
  in quest'ordine).

---

## 7. Transazioni — *Avanzato*

**Propagazione** — cosa succede quando un metodo transazionale ne chiama un altro:

| Livello | Comportamento |
|---|---|
| `REQUIRED` (default) | si aggancia alla transazione esistente, o ne crea una |
| `REQUIRES_NEW` | sospende quella esistente e ne apre una indipendente |
| `SUPPORTS` | partecipa se c'è, altrimenti gira senza |
| `MANDATORY` | richiede che ce ne sia già una, altrimenti errore |
| `NEVER` | errore se ne trova una |

**Isolamento** — alzarlo riduce le anomalie (dirty read, non-repeatable read, phantom read) e
aumenta la contesa. `SERIALIZABLE` non è un default prudente: è una scelta costosa da giustificare.

**Da segnalare in review**
- `@Transactional` su metodo privato, o invocato dall'interno della stessa classe: il proxy AOP non
  interviene e l'annotazione non ha alcun effetto. È il bug più subdolo di quest'area.
- Transazioni che racchiudono chiamate di rete o operazioni lunghe: tengono aperto un lock sul
  database per tutta la durata.
- `SERIALIZABLE` applicato ovunque per sicurezza.
- Rollback atteso su eccezioni checked senza aver dichiarato `rollbackFor`.

**Diagnosi**
- *Rollback che non avviene* → chiamata interna che bypassa il proxy, oppure eccezione checked non
  configurata.
- *Deadlock o timeout sotto carico* → isolamento troppo alto o transazione troppo ampia.
- *Dati scritti a metà* → operazioni correlate non racchiuse in un'unica transazione.

---

## 8. Actuator e logging — *Base*

**Da scrivere così**
- Esporre selettivamente gli endpoint Actuator: `health`, `info`, `metrics`. Arricchire `/info` con
  un `InfoContributor` custom quando servono dati dinamici.
- SLF4J con placeholder, mai concatenazione:
  `log.info("Articolo {} aggiornato da {}", id, user)`.
- Livelli coerenti: `ERROR` per ciò che richiede intervento, `WARN` per ciò che è anomalo ma
  gestito, `INFO` per gli eventi di business, `DEBUG` per il resto.
- Identificatore di correlazione sempre presente nel messaggio.

**Da segnalare in review**
- Tutti gli endpoint Actuator esposti senza autenticazione.
- `System.out.println`.
- Concatenazione nei log: il costo si paga anche quando il livello è disabilitato.
- `INFO` dentro cicli caldi.
- Dati personali, token o password nei log.

**Diagnosi**
- *Endpoint Actuator `404`* → non incluso nella lista di esposizione.
- *Health sempre `UP` con una dipendenza giù* → manca l'health indicator per quella risorsa.

---

## 9. WebFlux — *Avanzato*

Ha senso quando il collo di bottiglia è l'attesa di I/O, non la CPU. In un server tradizionale ogni
richiesta occupa un thread finché non finisce; in WebFlux il thread viene liberato durante l'attesa.

- `Mono<T>` — zero o un elemento. `Flux<T>` — zero o più.
- Lato client, `WebClient` sostituisce `RestTemplate`; la `subscribe` accetta il consumer di
  successo, quello di errore e quello di completamento.

**Da segnalare in review**
- `block()` dentro una catena reattiva: annulla il beneficio e può bloccare l'event loop.
- Driver o repository bloccanti (JDBC) dentro un servizio WebFlux.
- `Mono`/`Flux` creati e mai sottoscritti: non succede nulla.
- `subscribe` senza gestione dell'errore: i fallimenti spariscono in silenzio.

**Diagnosi**
- *Nessun effetto osservabile* → la pipeline non è mai stata sottoscritta.
- *Throughput che crolla* → una chiamata bloccante è finita sullo scheduler sbagliato.

Keyword: `Mono` / `Flux` · `WebClient` · `subscribe()` · Project Reactor · back-pressure
