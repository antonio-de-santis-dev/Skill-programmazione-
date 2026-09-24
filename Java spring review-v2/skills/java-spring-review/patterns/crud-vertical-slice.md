# Pattern — CRUD completo, dal database all'endpoint

Implementazione di riferimento di una risorsa REST completa. È la struttura insegnata nel corso
(applicazione `Car`, esercizio `Book`), qui generalizzata e resa adattabile.

**Baseline**: Java 17 + Spring Boot 3.x (import `jakarta.*`). Le varianti per Java 8 / Spring Boot 2
sono indicate dove cambiano. Verifica la versione con `references/java-versions.md` prima di
copiare.

Adatta nomi, package e campi al progetto reale. Questo è uno scheletro, non un template da
incollare.

---

## Struttura dei file

```
src/main/java/com/example/demo/
├── entity/Product.java
├── repository/ProductRepository.java
├── dto/ProductDTO.java
├── mapper/ProductMapper.java
├── service/ProductService.java            (interfaccia)
├── service/ProductServiceImpl.java
├── controller/ProductController.java
├── exception/ProductNotFoundException.java
├── exception/ProductCodeAlreadyExistsException.java
├── exception/ErrorResponse.java
└── exception/GlobalExceptionHandler.java
src/main/resources/application.yml
src/test/java/com/example/demo/controller/ProductControllerTest.java
```

Il criterio dei package: **per responsabilità tecnica** è quello del corso e va bene su un servizio
piccolo. Su un servizio che cresce, i package **per dominio** (`product/`, `order/`, ognuno con i
propri layer dentro) reggono meglio. Se il progetto ne usa già uno, mantieni quello.

---

## 1. Entity

```java
package com.example.demo.entity;

import jakarta.persistence.*;   // Spring Boot 2 → javax.persistence.*

@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String code;

    private String description;

    @Column(name = "unit_price")          // serve SOLO perché unitPrice ≠ unit_price
    private Double unitPrice;

    protected Product() { }               // richiesto da JPA

    public Product(String code, String description, Double unitPrice) {
        this.code = code;
        this.description = description;
        this.unitPrice = unitPrice;
    }

    public Long getId() { return id; }
    public String getCode() { return code; }
    public void setCode(String code) { this.code = code; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public Double getUnitPrice() { return unitPrice; }
    public void setUnitPrice(Double unitPrice) { this.unitPrice = unitPrice; }
}
```

**Decisioni da spiegare quando produci questo codice:**
- `@Column(name = ...)` serve solo quando il nome della proprietà differisce da quello della
  colonna. Metterlo ovunque è rumore.
- Il costruttore `protected` senza argomenti è un requisito di JPA, non una scelta di stile.
- **Se il progetto usa Lombok**: `@Getter @Setter @NoArgsConstructor @AllArgsConstructor`. Non
  `@Data` su un'entity: genera `equals`/`hashCode` su tutti i campi, incluse le relazioni lazy.
- **Un `record` non può essere un'entity JPA**: serve un costruttore senza argomenti e campi
  mutabili. Il `record` va bene per il DTO, non per l'entity.
- Per denaro, `BigDecimal` è più corretto di `Double`. Il corso usa tipi semplici; segnalalo come
  miglioramento quando il dominio è finanziario.

---

## 2. Repository

```java
package com.example.demo.repository;

import com.example.demo.entity.Product;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

public interface ProductRepository extends JpaRepository<Product, Long> {

    // Derived query: il nome del metodo genera l'SQL. Deve rispecchiare
    // le PROPRIETÀ dell'entity, non i nomi delle colonne.
    Optional<Product> findByCode(String code);

    List<Product> findAllByDescriptionContainingIgnoreCase(String fragment);

    boolean existsByCode(String code);

    @Transactional
    void deleteByCode(String code);

    // Query custom: quando la derived query diventerebbe illeggibile.
    @Query("select p from Product p where p.unitPrice between :min and :max")
    List<Product> findInPriceRange(@Param("min") Double min, @Param("max") Double max);

    // Scrittura custom: servono SIA @Modifying SIA @Transactional.
    @Modifying
    @Transactional
    @Query("update Product p set p.unitPrice = :price where p.code = :code")
    int updatePrice(@Param("code") String code, @Param("price") Double price);
}
```

`JpaRepository` fornisce già `save`, `findById`, `findAll`, `deleteById`, `deleteAll`, `count`. Non
riscriverli.

---

## 3. DTO e mapper

```java
package com.example.demo.dto;

import jakarta.validation.constraints.*;

// Java 16+: record. Su Java 8/11 → classe con costruttore, getter e setter.
public record ProductDTO(

    @NotBlank(message = "code is mandatory")
    @Size(max = 20, message = "code must be at most 20 characters")
    String code,

    @NotBlank(message = "description is mandatory")
    String description,

    @NotNull(message = "unitPrice is mandatory")
    @Positive(message = "unitPrice must be positive")
    Double unitPrice
) { }
```

Il DTO **non contiene `id`**: la chiave primaria è un dettaglio interno, e conoscerla facilita
l'attacco al database. La risorsa si identifica dall'esterno con una chiave di business (`code`).

```java
package com.example.demo.mapper;

import com.example.demo.dto.ProductDTO;
import com.example.demo.entity.Product;

public final class ProductMapper {

    private ProductMapper() { }

    public static ProductDTO toDto(Product p) {
        return new ProductDTO(p.getCode(), p.getDescription(), p.getUnitPrice());
    }

    public static Product toEntity(ProductDTO d) {
        return new Product(d.code(), d.description(), d.unitPrice());
    }
}
```

**Le tre strategie di mapping**, da scegliere consapevolmente:

| | Quando | Costo |
|---|---|---|
| Mapper manuale (sopra) | pochi campi, nessuna dipendenza in più | si dimenticano i campi nuovi |
| **ModelMapper** | molte conversioni simili, setup rapido | riflessione a runtime, più lento |
| **MapStruct** | percorsi ad alto volume | interfaccia `@Mapper` + annotation processor |

ModelMapper: un `@Bean ModelMapper` e poi `modelMapper.map(entity, ProductDTO.class)`.
MapStruct: un'interfaccia `@Mapper` con le sole firme, l'implementazione è generata a compile time.
In sintesi: **ModelMapper = comodità, MapStruct = prestazioni.**

---

## 4. Service

```java
package com.example.demo.service;

import com.example.demo.dto.ProductDTO;
import java.util.List;

public interface ProductService {
    List<ProductDTO> findAll();
    ProductDTO findByCode(String code);
    ProductDTO create(ProductDTO dto);
    ProductDTO update(String code, ProductDTO dto);
    void deleteByCode(String code);
}
```

```java
package com.example.demo.service;

import com.example.demo.dto.ProductDTO;
import com.example.demo.entity.Product;
import com.example.demo.exception.*;
import com.example.demo.mapper.ProductMapper;
import com.example.demo.repository.ProductRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class ProductServiceImpl implements ProductService {

    private final ProductRepository repository;

    // Iniezione via COSTRUTTORE. Con un solo costruttore @Autowired è superflua.
    // Con Lombok: @RequiredArgsConstructor sulla classe e niente costruttore a mano.
    public ProductServiceImpl(ProductRepository repository) {
        this.repository = repository;
    }

    @Override
    @Transactional(readOnly = true)
    public List<ProductDTO> findAll() {
        return repository.findAll().stream().map(ProductMapper::toDto).toList();
        // Java 8/11: .collect(Collectors.toList())
    }

    @Override
    @Transactional(readOnly = true)
    public ProductDTO findByCode(String code) {
        return repository.findByCode(code)
                .map(ProductMapper::toDto)
                .orElseThrow(() -> new ProductNotFoundException(code));
    }

    @Override
    @Transactional
    public ProductDTO create(ProductDTO dto) {
        if (repository.existsByCode(dto.code())) {
            throw new ProductCodeAlreadyExistsException(dto.code());
        }
        Product saved = repository.save(ProductMapper.toEntity(dto));
        return ProductMapper.toDto(saved);
    }

    @Override
    @Transactional
    public ProductDTO update(String code, ProductDTO dto) {
        if (!code.equals(dto.code())) {
            throw new ProductCodeMismatchException(code, dto.code());
        }
        Product existing = repository.findByCode(code)
                .orElseThrow(() -> new ProductNotFoundException(code));
        existing.setDescription(dto.description());
        existing.setUnitPrice(dto.unitPrice());
        return ProductMapper.toDto(repository.save(existing));
    }

    @Override
    @Transactional
    public void deleteByCode(String code) {
        if (!repository.existsByCode(code)) {
            throw new ProductNotFoundException(code);
        }
        repository.deleteByCode(code);
    }
}
```

**Due punti che vanno spiegati, perché sono la fonte di bug ricorrenti:**

`@Transactional` funziona tramite un proxy. Non ha effetto su metodi privati, né quando il metodo è
invocato **dall'interno della stessa classe** (`this.altroMetodo()`): la chiamata non passa dal
proxy. È il bug più subdolo di quest'area.

`@Transactional(readOnly = true)` sulle letture non è decorazione: permette ottimizzazioni e
previene scritture accidentali.

---

## 5. Eccezioni e handler globale

```java
package com.example.demo.exception;

public class ProductNotFoundException extends RuntimeException {
    public ProductNotFoundException(String code) {
        super("Product not found: " + code);
    }
}
```
(Analoghe: `ProductCodeAlreadyExistsException`, `ProductCodeMismatchException`.)

```java
package com.example.demo.exception;

import java.time.Instant;

public record ErrorResponse(int status, String error, String message, String path, Instant timestamp) { }
```

```java
package com.example.demo.exception;

import org.springframework.http.*;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.context.request.WebRequest;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;

import java.time.Instant;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ProductNotFoundException ex, WebRequest req) {
        return build(HttpStatus.NOT_FOUND, ex.getMessage(), req);
    }

    @ExceptionHandler(ProductCodeAlreadyExistsException.class)
    public ResponseEntity<ErrorResponse> handleConflict(ProductCodeAlreadyExistsException ex, WebRequest req) {
        return build(HttpStatus.CONFLICT, ex.getMessage(), req);
    }

    @ExceptionHandler(ProductCodeMismatchException.class)
    public ResponseEntity<ErrorResponse> handleMismatch(ProductCodeMismatchException ex, WebRequest req) {
        return build(HttpStatus.BAD_REQUEST, ex.getMessage(), req);
    }

    // Trasforma gli errori di @Valid in una mappa campo → messaggio,
    // al posto del 400 grezzo di default.
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders headers,
            HttpStatusCode status, WebRequest request) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(e -> errors.put(e.getField(), e.getDefaultMessage()));
        return new ResponseEntity<>(errors, HttpStatus.BAD_REQUEST);
    }

    private ResponseEntity<ErrorResponse> build(HttpStatus s, String msg, WebRequest req) {
        ErrorResponse body = new ErrorResponse(
            s.value(), s.getReasonPhrase(), msg,
            req.getDescription(false), Instant.now());
        return new ResponseEntity<>(body, s);
    }
}
```

Un solo punto di traduzione errore → risposta HTTP. Senza questo, ogni controller si riempie di
`try`/`catch` e le risposte d'errore divergono fra endpoint.

**Non mettere mai** stack trace o messaggi interni nel corpo della risposta.

---

## 6. Controller

```java
package com.example.demo.controller;

import com.example.demo.dto.ProductDTO;
import com.example.demo.service.ProductService;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.support.ServletUriComponentsBuilder;

import java.net.URI;
import java.util.List;

@RestController
@RequestMapping("/products")
public class ProductController {

    private final ProductService service;

    public ProductController(ProductService service) {
        this.service = service;
    }

    @GetMapping
    public ResponseEntity<List<ProductDTO>> getAll() {
        return ResponseEntity.ok(service.findAll());
    }

    @GetMapping("/{code}")
    public ResponseEntity<ProductDTO> getByCode(@PathVariable String code) {
        return ResponseEntity.ok(service.findByCode(code));
    }

    // POST sulla COLLEZIONE, 201 + header Location verso la risorsa creata.
    @PostMapping
    public ResponseEntity<ProductDTO> create(@Valid @RequestBody ProductDTO dto) {
        ProductDTO created = service.create(dto);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{code}").buildAndExpand(created.code()).toUri();
        return ResponseEntity.created(location).body(created);
    }

    // PUT sulla RISORSA SINGOLA, idempotente.
    @PutMapping("/{code}")
    public ResponseEntity<ProductDTO> update(@PathVariable String code,
                                             @Valid @RequestBody ProductDTO dto) {
        return ResponseEntity.ok(service.update(code, dto));
    }

    @DeleteMapping("/{code}")
    public ResponseEntity<Void> delete(@PathVariable String code) {
        service.deleteByCode(code);
        return ResponseEntity.noContent().build();     // 204
    }
}
```

**Senza `@Valid` le annotazioni sul DTO non scattano.** È l'errore più frequente della validazione
Spring, e non produce nessun avviso: il codice compila e accetta input non valido.

Path variable o request parameter: `@PathVariable` identifica la risorsa (`/products/ABC`),
`@RequestParam` filtra o modifica (`/products?minPrice=10`).

---

## 7. Configurazione

```yaml
# src/main/resources/application.yml
server:
  port: 8080

spring:
  application:
    name: product-service
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password:
  h2:
    console:
      enabled: true
  jpa:
    hibernate:
      ddl-auto: update        # SOLO in sviluppo. In produzione: validate o none + migrazioni
    show-sql: true            # utile in sviluppo per contare le query (problema N+1)
    properties:
      hibernate.format_sql: true

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics     # non "*"
```

Dipendenze minime nel `pom.xml`: `spring-boot-starter-web`, `spring-boot-starter-data-jpa`,
`spring-boot-starter-validation`, il driver del database, `spring-boot-starter-test` in scope test.

> `spring-boot-starter-validation` non è incluso in `starter-web` dalle versioni recenti: se manca,
> `@Valid` non fa nulla e non c'è alcun errore a segnalarlo.

---

## 8. Verifica

```bash
mvn clean spring-boot:run
```

```bash
# create → 201 + header Location
curl -i -X POST http://localhost:8080/products \
  -H 'Content-Type: application/json' \
  -d '{"code":"P-001","description":"Tastiera","unitPrice":49.90}'

# read → 200
curl http://localhost:8080/products/P-001

# validazione → 400 con { "unitPrice": "unitPrice must be positive" }
curl -i -X POST http://localhost:8080/products \
  -H 'Content-Type: application/json' \
  -d '{"code":"P-002","description":"x","unitPrice":-5}'

# duplicato → 409
# inesistente → 404
curl -i http://localhost:8080/products/NON-ESISTE

# delete → 204
curl -i -X DELETE http://localhost:8080/products/P-001
```

Console H2 su `http://localhost:8080/h2-console` per verificare che le righe ci siano davvero.

---

## 9. Cosa controllare se non funziona

| Sintomo | Causa più probabile |
|---|---|
| L'applicazione non parte, errore sul repository | il nome della derived query non corrisponde a una proprietà dell'entity |
| `404` su tutti gli endpoint | controller fuori dal package scansionato da `@SpringBootApplication` |
| `@Valid` non blocca input invalidi | manca `spring-boot-starter-validation`, o manca `@Valid` sul parametro |
| `500` dove ti aspetti `404` o `409` | eccezione di dominio non mappata nell'advice |
| Colonna non trovata su nomi come `year`, `order` | sono parole riservate SQL: `@Column(name = "\"year\"")` o rinomina |
| Update che non persiste | metodo senza `@Transactional`, o entity staccata dal contesto |
| Le query crescono con il numero di righe | problema N+1: guarda `show-sql`, usa fetch join o proiezione |
