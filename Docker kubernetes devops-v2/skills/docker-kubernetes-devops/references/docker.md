# Docker

Indice:
1. Modello immagine / container
2. Comandi e ciclo di vita
3. Registry, repository, tag
4. Creare immagini per Spring Boot
5. Docker Compose

---

## 1. Modello immagine / container — *Base*

L'**immagine** è l'artefatto immutabile, il **container** è l'istanza in esecuzione. In termini
Java: l'immagine è la classe, il container è l'oggetto. Da una sola immagine si istanziano quanti
container si vuole.

**Container contro macchina virtuale**: la VM porta con sé un intero sistema operativo, il container
si appoggia a quello dell'host e contiene solo ciò che serve all'applicazione — librerie, file di
configurazione, filesystem. Ne consegue: più leggero, più veloce ad avviarsi, e configurabile con un
singolo file condivisibile invece che con una procedura di installazione.

---

## 2. Comandi e ciclo di vita — *Base*

```bash
docker run <img>                    # crea e avvia (scarica l'immagine se manca)
docker run -p 8081:8080 <img>       # portaEsterna:portaInterna — senza, è irraggiungibile
docker run -d -p 8083:8080 <img>    # detach: gira in background

docker images                       # immagini presenti in locale
docker image rm <ID>                # rimuove un'immagine
docker pull mysql                   # senza tag scarica :latest

docker container ls                 # container avviati (serve "ls", non basta "docker container")
docker container stop <ID>          # arresto gentile
docker container kill <ID>          # terminazione forzata
docker container pause/unpause <ID>

docker logs <ID>                    # "logs", non "log"
docker logs -f <ID>                 # in tempo reale
```

Il meccanismo di `docker run`: Docker controlla se l'immagine esiste in locale; se non c'è la scarica
dal registry; poi avvia il container.

Per far girare più istanze della stessa immagine basta cambiare la porta esterna: `8081:8080`,
`8083:8080`, e così via.

**Da segnalare in review**
- Container avviati senza mappatura delle porte.
- Tag `latest` fuori dal locale.
- Dati importanti scritti nel filesystem effimero del container.
- Container in detach senza alcuna raccolta dei log.

**Diagnosi**
- *Nessun comando funziona* → il motore Docker non è avviato. Senza il motore acceso non funziona
  niente, nemmeno `docker images`.
- *Porta già occupata* → cambiare la porta **esterna**; quella interna è quella su cui ascolta
  l'applicazione e non va toccata.
- *Container che esce subito* → leggere i log: quasi sempre è un errore di avvio
  dell'applicazione, non di Docker.

---

## 3. Registry, repository, tag — *Base*

```
REGISTRY  (hub.docker.com, o il registry aziendale)
└── REPOSITORY  (mysql, utente/mio-servizio)
    └── TAG  (latest, 0.0.1-SNAPSHOT, 1.2.3)
```

Un registry contiene repository; un repository contiene immagini correlate distinte per tag. Senza
tag esplicito si scarica `:latest`.

In azienda si usano i **registry privati**, non l'hub pubblico. È una precisazione che vale la pena
fare esplicitamente in review quando si vede un `docker push` verso un profilo personale.

**Da segnalare in review**
- Immagini aziendali pubblicate su un registry pubblico.
- Tag riscritti: rendono impossibile sapere cosa gira davvero.
- Credenziali del registry nel repository di codice.
- Nessuna scansione di vulnerabilità sull'immagine base.

---

## 4. Creare immagini per Spring Boot — *Intermedio*

**Senza Dockerfile** — il plugin Spring Boot costruisce l'immagine da solo, con nome derivato
dall'`artifactId` e tag dalla versione del progetto:

```bash
mvn spring-boot:build-image -DskipTests
docker push <utente>/<nome>:<tag>
```

La configurazione del nome e del push automatico si mette nel `pom.xml` sotto la configurazione del
plugin. È la via più rapida e va benissimo per iniziare.

**Con Dockerfile** — quando serve controllo su base image, utente, layer e contenuto. Regole:

- **Multi-stage build**: uno stage compila, uno stage finale copia solo il jar. Il risultato non
  contiene sorgenti, Maven, né la cache delle dipendenze.
- **Base image minimale**: JRE, non JDK completo, se non serve compilare a runtime.
- **Utente non root**.
- **Ordine dei layer**: prima le dipendenze (cambiano raramente), poi il codice (cambia a ogni
  commit). Invertirli significa riscaricare tutto a ogni build.

**Da segnalare in review**
- Base image full JDK dove basta il runtime.
- Sorgenti, test o credenziali nell'immagine finale.
- Esecuzione come root.
- Nessun limite di memoria, o heap della JVM non allineato al limite del container.

**Diagnosi**
- *Immagine enorme* → manca il multi-stage, o si sta copiando l'intero contesto di build (serve
  `.dockerignore`).
- *Build lentissima* → ordine dei layer che invalida la cache delle dipendenze a ogni commit.
- *Container terminato dal sistema* → limite di memoria superato. La JVM va configurata per
  rispettare il limite del container, non quello della macchina.
- *Versione non allineata* → il tag segue la versione del `pom.xml`: va aggiornata prima di
  costruire.

---

## 5. Docker Compose — *Intermedio*

Descrive l'intero stack in un file YAML e lo avvia con `docker-compose up`.

```yaml
version: '3.7'
services:
  mia-app:
    image: utente/mia-app:0.0.2-SNAPSHOT
    ports: ["8081:8080"]
    environment:
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://naming-server:8761/eureka
    networks: [rete-app]
    depends_on: [naming-server]
  naming-server:
    image: utente/eureka-server:0.0.1-SNAPSHOT
    ports: ["8761:8761"]
    networks: [rete-app]
networks:
  rete-app:
```

**La regola che rompe più configurazioni**: dentro la rete Docker, il **nome del servizio** scritto
nel compose diventa l'hostname. `http://localhost:8761/eureka` non funziona da dentro un container,
perché `localhost` è il container stesso. Va scritto `http://naming-server:8761/eureka`.

Stessa cosa per ELK: Logstash deve puntare a `elasticsearch`, non a `localhost`. E per Kafka:
`KAFKA_ADVERTISED_LISTENERS` è l'indirizzo che Kafka comunica ai client per farsi contattare — se
resta `localhost`, i client dentro la rete non si collegano.

Stack tipici da comporre: applicazione + service registry + gateway + server di tracing;
Elasticsearch + Logstash + Kibana; broker + coordinatore.

**Da segnalare in review**
- `localhost` negli indirizzi fra container.
- Password e segreti in chiaro nel compose committato.
- Nessun volume per i dati che devono sopravvivere al container.
- Versioni delle immagini non fissate.

**Diagnosi**
- *Un servizio non raggiunge l'altro* → hostname errato, rete non condivisa, o servizio non ancora
  pronto (`depends_on` garantisce l'ordine di avvio, non che il servizio sia pronto a rispondere).
- *"Immagine non trovata"* → la versione indicata nel compose non è mai stata costruita o
  pubblicata. Ricostruire e ripubblicare, poi rilanciare.
- *Dati persi al riavvio* → manca il volume.
