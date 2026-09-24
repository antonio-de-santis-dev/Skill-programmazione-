# Pattern — test

Come testare ogni livello di un servizio Spring Boot, con il tipo di test giusto per ciascuno.
Baseline JUnit 5 + Spring Boot 3.x (`spring-boot-starter-test` include JUnit 5, Mockito,
AssertJ e MockMvc).

Indice:
1. Quale test per quale livello
2. Controller — MockMvc
3. Service — Mockito puro
4. Repository — @DataJpaTest
5. Integrazione completa
6. Postman nella pipeline
7. Errori ricorrenti

---

## 1. Quale test per quale livello

| Cosa testi | Annotazione | Avvia |
|---|---|---|
| Controller isolato | `@WebMvcTest(XController.class)` | solo il livello web |
| Service isolato | nessuna (JUnit + Mockito) | niente Spring |
| Repository e query | `@DataJpaTest` | JPA + database in memoria |
| Il servizio intero | `@SpringBootTest` + `@AutoConfigureMockMvc` | tutto il contesto |

La regola pratica: **usa la slice più piccola che risponde alla domanda.** Una suite fatta solo di
`@SpringBootTest` diventa lentissima e nasconde dove sta davvero il problema quando fallisce.

Il corso mostra la forma `@SpringBootTest` + `@AutoConfigureMockMvc`, che è quella giusta per
imparare e per i test end-to-end. Le slice sono l'evoluzione naturale quando la suite cresce.

---

## 2. Controller — MockMvc

```java
package com.example.demo.controller;

import com.example.demo.dto.ProductDTO;
import com.example.demo.exception.ProductNotFoundException;
import com.example.demo.service.ProductService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import java.util.List;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.doThrow;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired private MockMvc mockMvc;
    @Autowired private ObjectMapper objectMapper;

    @MockBean private ProductService service;    // il service è sostituito da un mock

    @Test
    void getByCode_quandoEsiste_ritorna200EIlCorpo() throws Exception {
        given(service.findByCode("P-001"))
            .willReturn(new ProductDTO("P-001", "Tastiera", 49.90));

        mockMvc.perform(get("/products/{code}", "P-001"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.code").value("P-001"))
               .andExpect(jsonPath("$.unitPrice").value(49.90));
    }

    @Test
    void getByCode_quandoNonEsiste_ritorna404() throws Exception {
        given(service.findByCode("NOPE"))
            .willThrow(new ProductNotFoundException("NOPE"));

        mockMvc.perform(get("/products/{code}", "NOPE"))
               .andExpect(status().isNotFound());
    }

    @Test
    void create_conCorpoValido_ritorna201EHeaderLocation() throws Exception {
        ProductDTO dto = new ProductDTO("P-002", "Mouse", 19.90);
        given(service.create(any(ProductDTO.class))).willReturn(dto);

        mockMvc.perform(post("/products")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(dto)))
               .andExpect(status().isCreated())
               .andExpect(header().exists("Location"));
    }

    @Test
    void create_conPrezzoNegativo_ritorna400ConIlCampoInErrore() throws Exception {
        String body = """
            {"code":"P-003","description":"Cavo","unitPrice":-1}
            """;                                  // text block: Java 15+

        mockMvc.perform(post("/products")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(body))
               .andExpect(status().isBadRequest())
               .andExpect(jsonPath("$.unitPrice").exists());
    }

    @Test
    void delete_quandoEsiste_ritorna204() throws Exception {
        mockMvc.perform(delete("/products/{code}", "P-001"))
               .andExpect(status().isNoContent());
    }
}
```

**Nota sulle asserzioni**: verificare solo `status().isOk()` è quasi inutile. Un endpoint può
rispondere 200 con il corpo sbagliato. Asserire almeno un campo del corpo è ciò che rende il test
capace di trovare una regressione.

Nomi dei test: `metodo_condizione_risultatoAtteso`. Quando il test fallisce in pipeline, il nome è
l'unica cosa che vedi subito.

> `@MockBean` è deprecato da Spring Boot 3.4 in favore di `@MockitoBean`. Su progetti recenti usa
> il secondo; su quelli meno recenti il primo funziona.

---

## 3. Service — Mockito puro

Il service non ha bisogno di Spring: si testa con JUnit e Mockito, ed è istantaneo.

```java
@ExtendWith(MockitoExtension.class)
class ProductServiceImplTest {

    @Mock private ProductRepository repository;
    @InjectMocks private ProductServiceImpl service;

    @Test
    void create_seIlCodiceEsiste_lanciaConflitto() {
        given(repository.existsByCode("P-001")).willReturn(true);

        assertThatThrownBy(() -> service.create(new ProductDTO("P-001", "x", 1.0)))
            .isInstanceOf(ProductCodeAlreadyExistsException.class);

        then(repository).should(never()).save(any());   // verifica che NON abbia salvato
    }

    @Test
    void findByCode_seNonEsiste_lanciaNotFound() {
        given(repository.findByCode("NOPE")).willReturn(Optional.empty());

        assertThatThrownBy(() -> service.findByCode("NOPE"))
            .isInstanceOf(ProductNotFoundException.class)
            .hasMessageContaining("NOPE");
    }
}
```

`then(repository).should(never()).save(any())` verifica un'assenza. È spesso l'asserzione più
importante: dimostra che il controllo ha davvero impedito l'effetto collaterale.

---

## 4. Repository — @DataJpaTest

Serve per verificare che le query facciano ciò che dichiarano, soprattutto le derived query e le
`@Query` scritte a mano.

```java
@DataJpaTest
class ProductRepositoryTest {

    @Autowired private ProductRepository repository;
    @Autowired private TestEntityManager em;

    @Test
    void findByCode_trovaIlProdottoInserito() {
        em.persistAndFlush(new Product("P-001", "Tastiera", 49.90));

        Optional<Product> found = repository.findByCode("P-001");

        assertThat(found).isPresent();
        assertThat(found.get().getDescription()).isEqualTo("Tastiera");
    }

    @Test
    void findInPriceRange_escludeIFuoriIntervallo() {
        em.persist(new Product("A", "economico", 5.0));
        em.persist(new Product("B", "medio", 50.0));
        em.persist(new Product("C", "caro", 500.0));
        em.flush();

        assertThat(repository.findInPriceRange(10.0, 100.0))
            .extracting(Product::getCode)
            .containsExactly("B");
    }
}
```

`@DataJpaTest` usa un database in memoria e fa **rollback dopo ogni test**: i test non si sporcano
a vicenda. Se il progetto usa funzionalità specifiche di PostgreSQL, H2 può ingannare: lì serve
Testcontainers, che avvia un PostgreSQL reale in un container.

---

## 5. Integrazione completa

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@ActiveProfiles("test")
class ProductIntegrationTest {

    @Autowired private MockMvc mockMvc;

    @Test
    void cicloCompleto_creaLeggiCancella() throws Exception {
        String body = """
            {"code":"IT-1","description":"Integrazione","unitPrice":9.99}
            """;

        mockMvc.perform(post("/products").contentType(MediaType.APPLICATION_JSON).content(body))
               .andExpect(status().isCreated());

        mockMvc.perform(get("/products/IT-1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.description").value("Integrazione"));

        mockMvc.perform(delete("/products/IT-1"))
               .andExpect(status().isNoContent());

        mockMvc.perform(get("/products/IT-1"))
               .andExpect(status().isNotFound());
    }
}
```

Questo è il test che verifica che i livelli siano davvero collegati: validazione, mapping,
transazioni e handler degli errori insieme. Ne bastano pochi.

---

## 6. Postman nella pipeline

Le collection Postman sono utili in sviluppo e possono girare in CI. I test si scrivono in
**JavaScript**, non in Java; l'oggetto `pm` è fornito da Postman.

```javascript
pm.test("status 201", function () {
    pm.response.to.have.status(201);
});

pm.test("il corpo contiene il code inviato", function () {
    const json = pm.response.json();
    pm.expect(json.code).to.eql(pm.environment.get("productCode"));
});

pm.test("risponde sotto i 500ms", function () {
    pm.expect(pm.response.responseTime).to.be.below(500);
});
```

URL e token vanno in **variabili d'ambiente**, mai cablati nella richiesta. La collection va
versionata insieme al codice del servizio, altrimenti diverge in una settimana.

---

## 7. Errori ricorrenti

| Sintomo | Causa |
|---|---|
| Passa in locale, fallisce in CI | dipendenza da ordine, fuso orario, dati preesistenti o risorse locali |
| Il contesto non si carica | property mancanti nel profilo di test, o un `@MockBean` che manca per una dipendenza |
| Suite lentissima | troppi `@SpringBootTest` dove basterebbe una slice |
| Test verde su un endpoint rotto | si asserisce solo lo status, mai il corpo |
| `UnnecessaryStubbingException` | uno stub Mockito dichiarato e mai usato: rimuovilo |
| Test che falliscono a rotazione | stato condiviso fra test, o `@DirtiesContext` mancante dove serve |

E un principio che non è una regola tecnica: **un test disabilitato non è un test.** O si sistema o
si cancella. Lasciarlo `@Disabled` nel repository dà l'illusione della copertura.
