---
name: docker-kubernetes-devops
description: Genera, valuta e corregge artefatti di containerizzazione e deployment - Dockerfile, immagini, docker-compose, networking fra container, registry, containerizzazione di applicazioni Spring Boot, Kubernetes (Pod, Deployment, Service, Ingress, ConfigMap, Secret, probe, resources, rollout), Helm, pipeline CI/CD, Infrastructure as Code, GitOps e troubleshooting operativo. Usa questa skill ogni volta che si scrive o si corregge un Dockerfile, un docker-compose.yml, un manifest Kubernetes, un chart Helm, un workflow di pipeline o un file Terraform, e anche quando l'utente descrive solo un sintomo operativo - "il container esce subito", "il pod non parte", "i container non si vedono fra loro", "come lo metto in produzione", "va in CrashLoopBackOff" - senza nominare esplicitamente Docker, Kubernetes o CI/CD.
---

# Docker, Kubernetes e catena di rilascio

Questa skill **produce artefatti**: Dockerfile, Compose, manifest, pipeline. Non li genera a
memoria — li deriva da ciò che l'applicazione è davvero.

---

## Passo 0 — Preflight, obbligatorio prima di generare qualsiasi artefatto

Un Dockerfile scritto senza guardare il progetto è un'ipotesi. Espone la porta sbagliata, usa la
versione di Java sbagliata, copia un jar che si chiama diversamente.

Raccogli questi dati **prima** di scrivere. Se non hai accesso al progetto, chiedi quelli che ti
mancano o dichiara l'assunzione in modo visibile.

| Cosa serve | Dove si legge |
|---|---|
| **Versione Java** | `pom.xml` (`java.version`), `build.gradle` (toolchain) |
| **Maven o Gradle** | presenza di `pom.xml` / `build.gradle` |
| **Nome del jar prodotto** | `artifactId` + `version` nel `pom.xml`, o `archiveFileName` in Gradle |
| **Porta dell'applicazione** | `server.port` in `application.yml` (default 8080) |
| **Context path** | `server.servlet.context-path`, se presente |
| **Profili Spring** | file `application-*.yml` |
| **Datasource** | `spring.datasource.*` — determina i servizi collegati |
| **Variabili d'ambiente attese** | i `${...}` nella configurazione |
| **Health endpoint** | Actuator presente? `management.endpoints.web.exposure.include` |
| **Dipendenze esterne** | database, broker, cache, altri servizi |
| **Configurazioni già esistenti** | `Dockerfile`, `docker-compose.yml`, cartella `k8s/` |

Il dettaglio più sbagliato in assoluto: **la porta**. `EXPOSE 8080` su un'applicazione che ascolta
sulla 8081 produce un container che parte e non risponde, senza alcun messaggio d'errore.

Se esistono già artefatti nel progetto, **partine** invece di generarne di nuovi: mantieni le
convenzioni, la base image e lo stile già in uso.

Quando dichiari un'assunzione, rendila verificabile:
> Assumo porta 8080, Java 17, Maven, jar `product-service-0.0.1-SNAPSHOT.jar`. Se `server.port` è
> diverso, cambia `EXPOSE` e la mappatura nel Compose.

---

## Dove trovare cosa

**Concetti e criteri** → `references/`

| Argomento | File |
|---|---|
| Immagini, container, comandi, registry, Compose, networking | `references/docker.md` |
| Pod, Deployment, Service, Ingress, ConfigMap, Secret, rollout, Helm, architettura del cluster | `references/kubernetes.md` |
| Pipeline, IaC, GitOps, cultura DevOps, piattaforme gestite | `references/cicd-gitops.md` |

**Artefatti pronti da adattare** → `patterns/`

| Serve | File |
|---|---|
| Dockerfile per Spring Boot, multi-stage, JVM nel container | `patterns/dockerfile-spring.md` |
| Compose: app + database, + Eureka + Gateway + Zipkin, ELK, Kafka | `patterns/compose-stacks.md` |
| Deployment, Service, ConfigMap, Secret, probe, Ingress, HPA | `patterns/k8s-manifests.md` |

---

## I due principi che governano tutto

**La stessa immagine in tutti gli ambienti, con la configurazione iniettata dall'esterno.** Da qui
discende quasi tutto: niente configurazione compilata nell'immagine, niente build separate per
produzione, niente segreti nei layer. Quando qualcosa "funziona in sviluppo ma non in produzione",
la prima domanda è se l'artefatto sia davvero lo stesso.

**Lo stato desiderato sta in un file versionato, non nella memoria di chi ha eseguito i comandi.**
Un cluster configurato a mano non è riproducibile, e un cluster non riproducibile non è
ripristinabile.

---

## Ordine di una review

1. **Segreti** — password, chiavi, token in Dockerfile, Compose, manifest o pipeline. Blocca tutto
   il resto.
2. **Riproducibilità** — versioni fissate (mai `latest` fuori dal locale), build indipendente dallo
   stato della macchina, manifest che descrivono ciò che gira davvero.
3. **Sicurezza del runtime** — utente non root, base image minimale, limiti di risorse.
4. **Correttezza operativa** — probe configurate **e distinte**, volumi per i dati persistenti,
   rete e nomi corretti.
5. **Efficienza** — dimensione dell'immagine, ordine dei layer, tempi di build.

---

## Errori che ricorrono più spesso

**`localhost` dentro una rete di container.** Dentro Docker o Kubernetes, `localhost` è il container
stesso. Per raggiungere un altro servizio si usa il nome del servizio. È l'errore singolo più
frequente, e si manifesta come "non si vedono fra loro".

**Deployment senza Service.** Applicare solo il Deployment crea i pod ma non li rende
raggiungibili. Servono due oggetti.

**Liveness e readiness sullo stesso endpoint, che verifica il database.** Quando il database
rallenta, la liveness fallisce e Kubernetes **riavvia tutti i pod**, trasformando un rallentamento
in un'interruzione totale.

**Tag `latest`.** Rollout non deterministico e rollback impossibile: non si sa a cosa si torna.

**Heap della JVM non allineato al limite del container.** Il container viene terminato dal sistema
e il log non dice perché.

**Modifiche imperative mai riportate nei manifest.** Cluster e repository divergono, e la
divergenza si scopre al primo ripristino.

---

## Proporzionare la complessità

La catena completa — Kubernetes, Helm, service mesh, GitOps, Terraform — ha un costo operativo
reale, e va presidiata da qualcuno. Non proporla per default.

| Situazione | Soluzione proporzionata |
|---|---|
| Un servizio, sviluppo locale | Dockerfile + `docker run` |
| Pochi servizi, un ambiente | Docker Compose |
| Un'applicazione da mettere online, team piccolo | container gestiti o piattaforma applicativa |
| Molti servizi, scaling, team di piattaforma | Kubernetes |
| Molti team, molti ambienti | Kubernetes + Helm + GitOps |

Se una soluzione più semplice basta, **dillo**. "Per due servizi, Compose su una VM vi porta in
produzione questa settimana; Kubernetes ha senso quando avrete bisogno di scaling automatico" è una
risposta migliore di un cluster completo.

---

## Contratto con le altre skill

| Situazione | Chi comanda |
|---|---|
| Codice dell'applicazione, `application.yml`, endpoint di health, test | **java-spring-review** |
| Quali servizi esistono, chi possiede quale database, come si parlano | **microservices-architecture** |
| Immagine, runtime, rete, orchestrazione, pipeline | **questa skill** |

Questa skill arriva **per ultima** nel workflow end-to-end: architettura → implementazione → test →
**container → deploy** → verifica. Se ti viene chiesto di containerizzare un servizio che non esiste
ancora, l'ordine è sbagliato: prima il servizio.

**Cosa restituire a monte.** A volte il preflight rivela che l'applicazione non è pronta per essere
containerizzata: stato scritto sul filesystem locale, nessun endpoint di health, configurazione
cablata nel codice, `@Scheduled` che partirebbe su ogni replica. Queste non sono cose da aggirare
nel Dockerfile — vanno segnalate e corrette nel codice. Passale a **java-spring-review** invece di
compensarle con un volume o una replica singola.
