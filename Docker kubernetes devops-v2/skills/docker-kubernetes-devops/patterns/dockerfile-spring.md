# Pattern — Dockerfile per Spring Boot

**Prima di scrivere**: esegui il preflight in `SKILL.md`. Versione Java, nome del jar e porta reale
sono i tre dati senza i quali questo file non serve a niente.

Indice:
1. Multi-stage Maven
2. Multi-stage Gradle
3. Layered jar
4. Senza Dockerfile: build-image
5. La JVM dentro il container
6. .dockerignore
7. Checklist e diagnosi

---

## 1. Multi-stage Maven

```dockerfile
# ---------- build ----------
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /build

# Prima le dipendenze: cambiano di rado, questo layer resta in cache.
COPY pom.xml .
RUN mvn -B dependency:go-offline

# Poi il codice: cambia a ogni commit, invalida solo da qui in giù.
COPY src ./src
RUN mvn -B clean package -DskipTests

# ---------- runtime ----------
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# Utente non privilegiato: un processo compromesso non è root nel container.
RUN addgroup -S app && adduser -S app -G app
USER app

# Il nome del jar deve corrispondere a artifactId-version del pom.
COPY --from=build /build/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

**Perché due stage.** Lo stage di build contiene Maven, il JDK completo, i sorgenti e la cache delle
dipendenze: centinaia di megabyte che in produzione non servono e sono superficie d'attacco. Lo
stage finale contiene solo JRE e jar.

**Perché `COPY pom.xml` prima di `COPY src`.** Docker invalida la cache dal primo layer che cambia
in poi. Copiando prima il `pom.xml` e scaricando le dipendenze, una modifica al codice non
riscarica l'intero repository Maven. Invertire i due `COPY` è l'errore che rende le build lente.

**Sostituisci `17` con la versione reale del progetto.** Se il `pom.xml` dichiara Java 21, l'immagine
base deve essere 21: un jar compilato per 21 non gira su una JRE 17 (`UnsupportedClassVersionError`).

**Su `-DskipTests`**: qui è corretto, perché i test devono girare nella pipeline di CI, non nella
build dell'immagine. Se non esiste una pipeline, toglilo.

---

## 2. Multi-stage Gradle

```dockerfile
FROM gradle:8-jdk17 AS build
WORKDIR /build
COPY build.gradle settings.gradle ./
RUN gradle dependencies --no-daemon || true
COPY src ./src
RUN gradle bootJar --no-daemon -x test

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
USER app
COPY --from=build /build/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Nota il percorso dell'output, diverso da Maven: `build/libs/` invece di `target/`.

---

## 3. Layered jar

Ottimizzazione utile quando l'immagine viene ricostruita spesso: Spring Boot può separare il jar in
strati, così le dipendenze (grandi e stabili) e il codice dell'applicazione (piccolo e volatile)
finiscono in layer Docker diversi.

```dockerfile
FROM eclipse-temurin:17-jre-alpine AS extract
WORKDIR /tmp
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
USER app
COPY --from=extract /tmp/dependencies/ ./
COPY --from=extract /tmp/spring-boot-loader/ ./
COPY --from=extract /tmp/snapshot-dependencies/ ./
COPY --from=extract /tmp/application/ ./
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

Il risultato: un deploy che cambia solo il codice trasferisce pochi megabyte invece dell'intero jar.
Su Spring Boot 2 la classe del launcher è `org.springframework.boot.loader.JarLauncher` (senza
`.launch`).

Vale la pena solo se si costruisce e si distribuisce spesso. Su un progetto che rilascia una volta
al mese, aggiunge complessità senza beneficio.

---

## 4. Senza Dockerfile: build-image

Il plugin Spring Boot costruisce l'immagine con i buildpack, senza scrivere alcun Dockerfile:

```bash
mvn spring-boot:build-image -DskipTests
```

Nome e tag derivano da `artifactId` e versione del progetto. Si configurano nel `pom.xml`:

```xml
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <configuration>
    <image>
      <name>registry.example.com/team/${project.artifactId}:${project.version}</name>
      <publish>true</publish>
    </image>
  </configuration>
</plugin>
```

| | Buildpack | Dockerfile |
|---|---|---|
| Velocità di partenza | immediata, zero configurazione | va scritto |
| Controllo | limitato | totale su base image, utente, layer |
| Immagine prodotta | ottimizzata, con utente non root | quello che scrivi tu |

Usa i buildpack per partire in fretta o quando il team non ha competenze Docker. Passa al Dockerfile
quando servono base image specifiche, tool aggiuntivi o vincoli di sicurezza aziendali.

---

## 5. La JVM dentro il container

Le JVM moderne (Java 11+) riconoscono i limiti del container, ma **il default è prudente**: usano
circa il 25% della memoria disponibile. Su un container da 1 GB significa 256 MB di heap.

```dockerfile
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0"
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

Usa `MaxRAMPercentage`, **non `-Xmx` fisso**: la percentuale si adatta quando cambia il limite del
container, il valore fisso no e produce un OOM alla prima riduzione.

Lascia margine fra heap e limite del container: la JVM usa memoria anche fuori dall'heap
(metaspace, stack dei thread, buffer diretti). Un `MaxRAMPercentage` a 100 garantisce che il
container venga terminato dal sistema.

Altre due voci utili:
```
-XX:+ExitOnOutOfMemoryError    il container termina e l'orchestratore lo riavvia,
                               invece di restare vivo e inutilizzabile
-Duser.timezone=Europe/Rome    i container partono in UTC
```

---

## 6. .dockerignore

Senza questo file l'intero contesto viene inviato al daemon, `target/` e `.git` compresi.

```
target/
build/
.git
.gitignore
.idea
*.iml
.mvn/wrapper/maven-wrapper.jar
**/*.md
.env
```

Il `.env` nella lista non è un dettaglio: è il modo in cui i segreti finiscono nelle immagini.

---

## 7. Checklist e diagnosi

Prima di considerare finito un Dockerfile:

- [ ] Base image con la **versione Java del progetto**, non una a caso.
- [ ] Multi-stage: il JDK e i sorgenti non sono nell'immagine finale.
- [ ] Utente non root.
- [ ] `EXPOSE` uguale a `server.port` reale.
- [ ] Nessun segreto in `ENV`, `ARG` o nei layer.
- [ ] `.dockerignore` presente.
- [ ] Layer delle dipendenze prima di quello del codice.
- [ ] Memoria JVM configurata in percentuale.
- [ ] Tag dell'immagine legato alla versione o al commit, mai `latest`.

| Sintomo | Causa |
|---|---|
| Il container parte e subito esce | errore di avvio dell'applicazione: `docker logs <id>` |
| Il container gira ma non risponde | porta sbagliata in `EXPOSE` o nella mappatura `-p` |
| `UnsupportedClassVersionError` | jar compilato con una versione Java superiore alla JRE dell'immagine |
| Container terminato senza messaggio | limite di memoria superato: heap JVM non allineato |
| `no main manifest attribute` | copiato il jar sbagliato (in `target/` ce n'è anche uno `.original`) |
| Build lentissima | ordine dei layer, o `.dockerignore` mancante |
| Immagine da oltre 500 MB | manca il multi-stage, o base image `jdk` invece di `jre` |
| Non raggiunge il database | `localhost` invece del nome del servizio nella rete |
