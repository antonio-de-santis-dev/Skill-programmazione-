# Versioni Java e Spring — cosa è disponibile dove

Questo file esiste per una ragione operativa precisa: **"Java 8-25" non significa che tutte le
funzionalità siano usabili contemporaneamente.** Generare un `record` su un progetto Java 8 produce
codice che non compila, e l'errore non si vede finché non si prova a buildare.

Consulta questa tabella **prima** di usare una funzionalità moderna in un progetto reale.

---

## 1. Come determinare la versione effettiva

Nell'ordine, fermandoti al primo dato attendibile:

**Maven** — `pom.xml`:
```xml
<properties>
    <java.version>17</java.version>          <!-- forma Spring Boot -->
    <maven.compiler.release>17</...>          <!-- forma standard -->
</properties>
<parent>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>                  <!-- versione Spring Boot -->
</parent>
```

**Gradle** — `build.gradle` / `build.gradle.kts`:
```groovy
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
// oppure la forma vecchia:
sourceCompatibility = '11'
```

**Altre fonti**, utili come conferma o quando le prime mancano:
- `Dockerfile` — l'immagine base (`eclipse-temurin:17-jre`) dice su cosa gira davvero.
- Configurazione CI — la versione del JDK impostata per la build.
- Maven Toolchains (`~/.m2/toolchains.xml`).
- La versione di Spring Boot, che implica un minimo (vedi sotto).

**Se nessuna fonte è disponibile**: dichiara l'assunzione in modo visibile e spiega cosa cambierebbe.
Non scegliere in silenzio.

> Assumo Java 17 + Spring Boot 3.x. Se il progetto è su Java 8, `record` e `var` non sono
> disponibili: il DTO diventa una classe con costruttore e getter, e serve `jakarta` → `javax` in
> tutti gli import.

---

## 2. Funzionalità del linguaggio → versione minima

Le versioni **LTS** sono 8, 11, 17, 21, 25: sono quelle che si incontrano nei progetti reali.

| Funzionalità | Da | Note |
|---|---|---|
| Lambda, method reference, Stream, `Optional`, default method | **8** | la base |
| `private` method nelle interfacce | **9** | |
| Module system (JPMS), `module-info.java` | **9** | |
| `var` su variabile locale (LVTI) | **10** | |
| `var` nei parametri di lambda | **11** | |
| `switch` come espressione, frecce, `yield` | **14** | |
| Text block (`"""`) | **15** | |
| Pattern matching per `instanceof` | **16** | |
| `record` | **16** | |
| `sealed` / `permits` / `non-sealed` | **17** | |
| Pattern matching per `switch` | **21** | inclusi `case null` e guardie `when` |
| Record pattern (destrutturazione) | **21** | |
| Virtual thread | **21** | |
| Sequenced Collections (`getFirst`, `reversed`…) | **21** | |
| Unnamed variables e pattern (`_`) | **22** | |
| Javadoc in Markdown | **23** | |
| Stream Gatherers (`Stream.gather`) | **24** | |
| Scoped Values | **25** | |
| Module Import Declarations | **25** | |
| Compact source file + instance `main` | **25** | |
| Flexible constructor bodies | **25** | |

**Regola pratica per i progetti diffusi**: su **Java 8** niente `var`, `record`, `switch`
espressione, text block. Su **Java 11** si aggiunge solo `var`. Su **Java 17** si sblocca la maggior
parte di ciò che oggi si considera "Java moderno" — `record`, `sealed`, `switch` espressione, text
block. Su **Java 21** arrivano virtual thread e pattern matching completo.

Le funzionalità in *preview* (`--enable-preview`) non vanno usate in codice di produzione, e non
vanno proposte come soluzione se non è l'utente a chiederlo esplicitamente per sperimentare.

---

## 3. Spring Boot → Java minimo

| Spring Boot | Java minimo | Namespace |
|---|---|---|
| 2.x | 8 | `javax.*` |
| 3.0 – 3.x | **17** | `jakarta.*` |

È la rottura più impattante degli ultimi anni e va verificata sempre: su Spring Boot 3 gli import
sono `jakarta.persistence.Entity`, `jakarta.validation.constraints.NotBlank`,
`jakarta.servlet.*`. Su Spring Boot 2 sono `javax.*`. Sbagliare namespace significa codice che
non compila, e il messaggio d'errore non spiega il perché.

Note aggiuntive:
- Il supporto ai **virtual thread** in Spring Boot arriva dalla 3.2, e richiede comunque Java 21.
- **Spring Cloud** ha una propria linea di versioni allineata a Spring Boot: non si sceglie a caso,
  si prende quella compatibile dalla tabella ufficiale. Un Spring Cloud sbagliato produce errori di
  avvio oscuri.

---

## 4. Cambiamenti di comportamento che dipendono dalla versione

Oltre alle funzionalità, alcune cose cambiano di significato fra versioni. Verificarle evita
diagnosi sbagliate.

**Spring Cloud — `bootstrap.yml`.** Fino a Spring Cloud Hoxton, la configurazione remota si
dichiarava in `bootstrap.yml`. Da Spring Cloud 2020.0 il bootstrap context è **disabilitato di
default**: o si aggiunge `spring-cloud-starter-bootstrap`, oppure — forma moderna e preferibile —
si usa nel normale `application.yml`:
```properties
spring.config.import=configserver:http://localhost:8888
```
Un `bootstrap.yml` che "viene ignorato" su un progetto recente è questo, non un errore di sintassi.

**Spring Cloud — annotazioni diventate superflue.** `@EnableDiscoveryClient` non è più necessaria
nelle versioni recenti: basta la dipendenza sul classpath. Lo stesso vale per `@EnableFeignClients`
in alcune configurazioni, ma lì conviene tenerla per esplicitezza.

**Componenti Netflix deprecati.** Ribbon è sostituito da **Spring Cloud LoadBalancer**, Hystrix da
**Resilience4j**, Zuul da **Spring Cloud Gateway**. Su codice nuovo non vanno proposti; su codice
esistente vanno segnalati come debito, non come errore da correggere subito.

**Annotazioni REST abbreviate.** `@GetMapping` e simili esistono da Spring 4.3 e sono oggi la forma
normale. `@RequestMapping(method = ...)` non è sbagliato e resta corretto a livello di classe per
il prefisso, ma su codice nuovo si usa la forma breve.

**Git.** `git checkout` è stato diviso in `git switch` (spostarsi) e `git restore` (ripristinare
file) dalla 2.23. Entrambe le forme funzionano; le nuove sono più chiare e sono quelle che `git
status` stesso suggerisce.

---

## 5. Quando proporre un aggiornamento di versione

Solo se risolve un problema che l'utente ha davvero, e dicendo cosa costa. Un salto Spring Boot
2 → 3 comporta la migrazione `javax` → `jakarta` su tutto il progetto: non è una riga nel `pom.xml`.

Formula utile: *"questo si risolverebbe in modo pulito con X, disponibile da Java 21. Sul tuo Java
17 la soluzione è Y, che funziona bene. Vale la pena aggiornare solo se avete altri motivi per
farlo."*
