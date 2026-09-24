# Pattern — Spring Cloud: configurazione reale

Configurazione funzionante dei componenti Spring Cloud insegnati nel corso. Le porte usate sono le
convenzioni de facto: **8888** Config Server, **8761** Eureka, **8765** Gateway, **9411** Zipkin.

**Attenzione alla versione**: Spring Cloud ha una propria linea allineata a Spring Boot. Non
sceglierla a caso — un abbinamento sbagliato produce errori di avvio oscuri. Verifica in
`java-spring-review/references/java-versions.md` anche le differenze di comportamento
(`bootstrap.yml`, componenti Netflix deprecati).

Indice:
1. Config Server e Config Client
2. Eureka: server e client
3. OpenFeign
4. Spring Cloud LoadBalancer
5. API Gateway
6. Resilience4j
7. Tracing distribuito
8. Ordine di avvio e diagnosi

---

## 1. Config Server e Config Client

**Il problema**: 200 microservizi significano 200 `application.properties` da mantenere e da
riavviare a ogni modifica. Il Config Server centralizza la configurazione in un repository e la
serve via HTTP.

```
[ Repository ]  ←  [ Config Server :8888 ]  ←  [ Client 1 … Client N ]
(file o Git)          legge i file               chiedono la config all'avvio
```

### Server

Dipendenza: `spring-cloud-config-server`.

```java
@SpringBootApplication
@EnableConfigServer          // questa sola annotazione lo trasforma in Config Server
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

Sorgente **Git** (la forma da usare in un progetto reale):
```properties
server.port=8888
spring.cloud.config.server.git.uri=https://github.com/org/config-repo
spring.cloud.config.server.git.cloneOnStart=true   # errori di repo emergono all'avvio, non dopo
```

Sorgente **file system** (comoda in locale, profilo `native`):
```yaml
server:
  port: 8888
spring:
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: file:///opt/config/
```

Nel repository, un file per applicazione e profilo: `product-service.properties`,
`product-service-dev.properties`, `product-service-prod.properties`, più `application.properties`
per ciò che è comune a tutti.

### Client

Dipendenza: `spring-cloud-starter-config`.

Forma **moderna** (Spring Cloud 2020.0 e successive), dentro il normale `application.properties`:
```properties
spring.application.name=product-service
spring.config.import=configserver:http://localhost:8888
```

Forma **legacy** con `bootstrap.yml`, che si incontra nel materiale del corso e nei progetti più
vecchi. Da Spring Cloud 2020.0 richiede la dipendenza `spring-cloud-starter-bootstrap`, altrimenti
il file viene semplicemente ignorato senza alcun errore:
```yaml
spring:
  application:
    name: product-service
  cloud:
    config:
      uri: http://localhost:8888
```

Il consumo è trasparente: la property remota si legge con `@Value` come se fosse locale.
```java
@Value("${welcome.message}")     // non esiste in nessun file del progetto client
private String welcomeMessage;
```

**Verifica prima di scrivere il client.** Il Config Server espone la configurazione come API REST:
```
GET http://localhost:8888/{applicazione}/{profilo}/{label}
GET http://localhost:8888/product-service/dev/main
```
Se questa chiamata non restituisce le property, il problema è nel server, non nel client. È il modo
più rapido per isolare il guasto.

**Da segnalare in review**: segreti in chiaro nel repository di configurazione (servono cifratura o
un gestore esterno); nessun meccanismo di refresh, con la conseguenza che cambiare una property
richiede comunque un riavvio.

---

## 2. Eureka: server e client

### Server

Dipendenza: `spring-cloud-starter-netflix-eureka-server`.

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication { ... }
```

```properties
server.port=8761
eureka.client.register-with-eureka=false   # il server non si registra a se stesso
eureka.client.fetch-registry=false         # non scarica il registro da un altro Eureka
```
Le due property a `false` valgono per un server **singolo**. In un cluster di Eureka vanno a `true`
perché le istanze si registrano e si sincronizzano fra loro.

### Client

Dipendenza: `spring-cloud-starter-netflix-eureka-client`.

```properties
spring.application.name=product-service    # questo È il nome con cui sarà trovato: non banale
eureka.client.serviceUrl.defaultZone=${EUREKA_URI:http://localhost:8761/eureka}
```

Nelle versioni recenti **non serve `@EnableDiscoveryClient`**: basta la dipendenza sul classpath.

La sintassi `${EUREKA_URI:default}` legge la variabile d'ambiente e usa il default se assente. È
esattamente ciò che serve per far funzionare lo stesso artefatto in locale e dentro Docker Compose,
dove l'indirizzo diventa `http://naming-server:8761/eureka`.

Il ciclo di vita: il servizio si **registra** all'avvio, invia **heartbeat** periodici, si
**cancella** alla chiusura. Se gli heartbeat smettono di arrivare, Eureka rimuove l'istanza.

---

## 3. OpenFeign

Client HTTP dichiarativo: si descrive il servizio remoto con un'interfaccia, Feign genera
l'implementazione a runtime.

```java
@SpringBootApplication
@EnableFeignClients          // fa scansionare le interfacce @FeignClient
public class Application { ... }
```

**Senza registry** — indirizzo cablato, va bene solo in locale:
```java
@FeignClient(name = "catalog-service", url = "localhost:8080")
public interface CatalogClient {
    @GetMapping("/api/products/{id}")
    ResponseEntity<ProductDTO> getProduct(@PathVariable Long id);
}
```

**Con registry** — l'`url` sparisce, resta solo il nome logico:
```java
@FeignClient("catalog-service")     // deve coincidere con lo spring.application.name remoto
public interface CatalogClient {
    @GetMapping("/api/products/{id}")
    ResponseEntity<ProductDTO> getProduct(@PathVariable Long id);
}
```

Questa è la differenza che rende Feign utile: nel codice resta un **nome**, l'indirizzo lo risolve
il registry a runtime.

Uso: si inietta l'interfaccia come un normale bean.
```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final CatalogClient catalogClient;

    public OrderController(CatalogClient catalogClient) {
        this.catalogClient = catalogClient;
    }
    // ...
}
```

Timeout — **mai lasciare i default**, sono troppo permissivi:
```properties
spring.cloud.openfeign.client.config.default.connectTimeout=2000
spring.cloud.openfeign.client.config.default.readTimeout=5000
spring.cloud.openfeign.client.config.catalog-service.readTimeout=10000
```

Un Feign senza timeout, retry e circuit breaker è una chiamata remota senza rete di protezione: una
lentezza a valle diventa un esaurimento di thread a monte.

---

## 4. Spring Cloud LoadBalancer

Bilanciamento **client-side**: è il chiamante a scegliere l'istanza, dopo aver chiesto al registry
la lista di quelle vive. Eureka fornisce solo la lista — **non vede passare il traffico
applicativo**. È una precisazione che conta, perché viene spesso fraintesa.

Con Eureka, il bilanciamento è automatico: Feign risolve il nome, ottiene N istanze e distribuisce.
Non serve configurare nulla.

Con `WebClient`, serve marcare il builder:
```java
@Configuration
public class WebClientConfig {
    @Bean
    @LoadBalanced                      // intercetta gli URL con nome logico e li risolve
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}
```
```java
webClient.get().uri("http://product-service/products").retrieve()...
```

Senza registry, la lista di istanze va fornita a mano implementando
`ServiceInstanceListSupplier`. Serve a capire il meccanismo; in un sistema reale quel ruolo lo
svolge il registry.

> **Ribbon è deprecato.** Su codice nuovo si usa Spring Cloud LoadBalancer. Su codice esistente
> segnalalo come debito tecnico, non come errore da correggere subito.

---

## 5. API Gateway

Dipendenze: `spring-cloud-starter-gateway` + `spring-cloud-starter-netflix-eureka-client`.

**Route automatiche da registry** — zero configurazione di routing:
```properties
server.port=8765
spring.cloud.gateway.discovery.locator.enabled=true
spring.cloud.gateway.discovery.locator.lower-case-service-id=true
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka
```
Il Gateway legge il registro e crea una route per ogni servizio registrato. Un microservizio nuovo
viene esposto senza toccare la configurazione.

> Eureka **normalizza i nomi in maiuscolo**. Senza `lower-case-service-id`, la route è
> `/PRODUCT-SERVICE/**`. È una fonte frequente di 404 apparentemente inspiegabili.

**Route esplicite** — quando serve controllo su path, filtri e riscritture:
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: product-route
          uri: lb://product-service          # lb:// = risolvi e bilancia tramite registry
          predicates:
            - Path=/catalog/**
          filters:
            - RewritePath=/catalog/(?<segment>.*), /products/${segment}
            - AddRequestHeader=X-Source, gateway
```

**Equivalente in Java**, più flessibile quando la logica di routing è dinamica:
```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route(r -> r.path("/catalog/**")
                         .filters(f -> f.rewritePath("/catalog/(?<s>.*)", "/products/${s}")
                                        .addRequestHeader("X-Source", "gateway"))
                         .uri("lb://product-service"))
            .build();
    }
}
```

I filtri sono il punto in cui si concentra il **gateway offloading**: autenticazione, rate limiting,
header di correlazione. Non è il posto per la logica di business.

---

## 6. Resilience4j

Dipendenza: `spring-cloud-starter-circuitbreaker-resilience4j`.

```java
@Service
public class CatalogFacade {

    private final CatalogClient client;

    @CircuitBreaker(name = "catalog", fallbackMethod = "fallbackProduct")
    @Retry(name = "catalog")
    @TimeLimiter(name = "catalog")
    public ProductDTO getProduct(Long id) {
        return client.getProduct(id).getBody();
    }

    // Stessa firma + il Throwable in coda. Restituisce un valore degradato ma utile.
    private ProductDTO fallbackProduct(Long id, Throwable t) {
        log.warn("catalog non disponibile per {}: {}", id, t.toString());
        return ProductDTO.unavailable(id);
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      catalog:
        slidingWindowSize: 10
        failureRateThreshold: 50            # % di fallimenti che apre il circuito
        waitDurationInOpenState: 10s        # quanto resta aperto prima di half-open
        permittedNumberOfCallsInHalfOpenState: 3
  retry:
    instances:
      catalog:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true      # senza backoff i retry amplificano il picco
  timelimiter:
    instances:
      catalog:
        timeoutDuration: 3s
```

**Retry e circuit breaker rispondono a due domande diverse**: il retry presume che riprovare possa
funzionare (guasto transitorio), il circuit breaker presume di no (guasto persistente). Si usano
insieme, non in alternativa.

Il fallback **non deve nascondere il problema**: deve incrementare una metrica o produrre un log,
altrimenti il sistema degrada in silenzio e nessuno se ne accorge.

> **Hystrix è deprecato**: su codice nuovo si usa Resilience4j.

---

## 7. Tracing distribuito

Zipkin si avvia con una riga, senza configurare nulla:
```bash
docker run -d -p 9411:9411 openzipkin/zipkin
```

Lato applicazione (Spring Boot 3 usa Micrometer Tracing; Spring Boot 2 usava Sleuth):
```xml
<dependency>
  <groupId>io.micrometer</groupId><artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
  <groupId>io.zipkin.reporter2</groupId><artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```
```properties
management.tracing.sampling.probability=1.0        # 1.0 solo in sviluppo
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans
```

Il trace id viene propagato automaticamente sulle chiamate HTTP fatte con `RestTemplate`,
`WebClient` e Feign. **Non** viene propagato da solo su thread creati a mano né attraverso un broker
di messaggi: lì va passato esplicitamente. Sono i due punti in cui le tracce si spezzano.

Perché conta: in produzione, davanti a un errore, la traccia dice **quale passo** della catena ha
fallito. Senza, si cercano i log servizio per servizio.

---

## 8. Ordine di avvio e diagnosi

L'ordine conta, e non rispettarlo produce errori che sembrano di configurazione:

```
1. Config Server   :8888
2. Eureka Server   :8761
3. I microservizi
4. API Gateway     :8765
```

| Sintomo | Causa più probabile |
|---|---|
| Il client non trova le property remote | Config Server non avviato, o `bootstrap.yml` ignorato su Spring Cloud recente |
| Servizio non visibile nella dashboard Eureka | `defaultZone` errato, oppure avviato prima del registry |
| Feign: errore di risoluzione del nome | il nome non corrisponde allo `spring.application.name` remoto |
| Feign: `404` sul metodo | path o verbo divergenti dal controller remoto |
| Gateway: `404` su tutte le route | nome del servizio in maiuscolo (manca `lower-case-service-id`) |
| Chiamata risolta su un'istanza morta | lease non ancora scaduto — la protezione vera è il circuit breaker |
| Dentro Docker non si vedono | `localhost` invece del nome del servizio nella rete Compose |
