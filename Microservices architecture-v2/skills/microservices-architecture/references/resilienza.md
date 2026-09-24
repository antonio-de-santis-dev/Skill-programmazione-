# Resilienza, observability e rilasci

Indice:
1. Pattern di resilienza
2. Logging centralizzato
3. Tracing distribuito
4. Health check e monitoring
5. Sidecar e service mesh
6. Strategie di deployment

---

## 1. Pattern di resilienza — *Avanzato*

In un sistema distribuito il guasto non è un'eccezione: è una condizione operativa. Il caso difficile
non è il servizio **giù** — quello fallisce subito ed è gestibile. È il servizio **lento**, che tiene
occupati i thread del chiamante finché non li esaurisce.

**I cinque pattern, e quando ciascuno si applica**

| Pattern | Risolve | Attenzione |
|---|---|---|
| **Retry** | guasti transitori (un pacchetto perso, un riavvio) | inutile e dannoso sui guasti persistenti |
| **Circuit breaker** | guasti persistenti: smette di chiamare chi è chiaramente giù | le soglie vanno tarate su dati reali |
| **Timeout** | attese indefinite | va su **ogni** chiamata di rete, senza eccezioni |
| **Bulkhead** | isolamento: un endpoint lento non deve consumare tutto il pool | dimensionare gli scomparti |
| **Fallback** | risposta degradata ma utile | non deve nascondere il problema al monitoraggio |

**La distinzione fondamentale**: *retry* presume che riprovare possa funzionare, *circuit breaker*
presume di no. Si usano insieme — retry per i primi tentativi, circuit breaker per smettere quando è
evidente che il problema non è transitorio — ma rispondono a due domande diverse.

**Stati del circuit breaker**
- **closed** — traffico normale, si contano i fallimenti.
- **open** — superata la soglia, le chiamate falliscono immediatamente senza raggiungere il servizio.
  Questo protegge anche il servizio a valle, dandogli tregua per riprendersi.
- **half-open** — dopo un intervallo, si lasciano passare alcune chiamate di prova. Se passano,
  torna closed; se no, torna open.

**Da segnalare in review**
- Retry senza backoff esponenziale e jitter: sincronizza i client e amplifica il picco.
- Retry su errori non transitori. Un `400` non migliora ripetendolo.
- Retry annidati su più livelli: 3 tentativi per 3 livelli sono 27 chiamate.
- Assenza di timeout.
- Fallback silenzioso che non incrementa alcuna metrica.

**Diagnosi**
- *Effetto valanga durante un incidente* → i retry stanno aggravando il sovraccarico. Ridurre i
  tentativi e aprire prima il circuito.
- *Circuit breaker sempre aperto* → soglia troppo bassa o finestra troppo corta.
- *Thread pool esaurito* → manca il bulkhead; un singolo endpoint lento sta consumando tutto.

Nota: **Hystrix è deprecato**, su codice nuovo si usa Resilience4j.

---

## 2. Logging centralizzato — *Intermedio*

In un sistema distribuito collegarsi al singolo container per leggere i log non è una strategia. I
log vanno raccolti in un punto unico e interrogabile.

**Stack ELK**
- **Elasticsearch** — indicizza e rende ricercabili i documenti.
- **Logstash** — riceve, filtra e trasforma gli eventi prima di indicizzarli.
- **Kibana** — interfaccia di query e dashboard.

Configurazione Spring: un appender Logback verso Logstash via socket TCP, con encoder JSON, oltre a
quello di console.

```xml
<springProperty scope="context" name="appName" source="spring.application.name"/>
<appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
  <destination>localhost:4560</destination>
  <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
```

**Da segnalare in review**
- Log non strutturati: impediscono il filtraggio per campo, che è l'intero punto dell'operazione.
- Nome dell'applicazione non incluso nell'evento.
- Identificatore di correlazione assente: senza, i log centralizzati sono un mucchio, non un
  sistema.
- Dati personali o segreti spediti all'indice.
- Nessuna politica di retention.

**Diagnosi**
- *Nessun documento in Kibana* → verificare porta dell'appender, hostname del servizio Elasticsearch
  **nella rete Docker** (il nome del servizio, non `localhost`), e index pattern creato.
- *Indici che riempiono il disco* → mancano rollover e retention.

---

## 3. Tracing distribuito — *Intermedio*

Il logging dice cosa è successo in un servizio. Il tracing dice **dove è passata** una richiesta e
**dove ha perso tempo**.

- **OpenTelemetry** è lo standard di strumentazione, non un backend. Definisce come strumentare il
  codice e propagare il contesto; i dati vengono poi esportati verso Zipkin, Jaeger, o un servizio
  gestito. Il valore è che si può cambiare backend senza ristrumentare.
- **Zipkin** è un backend di tracing con UI: raccoglie le tracce, le correla e le visualizza.
  Si avvia con un container e una porta.

Il **correlation ID / trace ID** è ciò che lega tutto: compare nella traccia **e** nei log, e permette
di passare dall'una agli altri.

**Da segnalare in review**
- Contesto di traccia non propagato su chiamate asincrone, attraverso il broker, o su thread creati
  a mano. Sono i tre punti dove le tracce si spezzano.
- Campionamento al 100% in produzione su servizi ad alto volume.
- Span senza attributi utili: una traccia che dice solo "qui ci sono voluti 3 secondi" non aiuta.
- Correlation ID generato ma non scritto nei log.

**Diagnosi**
- *Tracce spezzate in due* → gli header di propagazione non attraversano quel tratto.
- *Nessuna traccia visibile* → sampling a zero o esportatore non configurato.

---

## 4. Health check e monitoring — *Intermedio*

**Tre tipi, con conseguenze diverse**

| Tipo | Domanda | Cosa fa l'orchestratore se fallisce |
|---|---|---|
| **Liveness** | il processo è vivo? | **riavvia** il container |
| **Readiness** | è pronto a ricevere traffico? | **toglie** dal bilanciamento, non riavvia |
| **Performance** | risponde entro i tempi attesi? | alimenta allarmi e decisioni di scaling |

**L'errore da cercare sempre**: liveness e readiness mappate sullo stesso endpoint, e quell'endpoint
verifica il database. Quando il database rallenta, la liveness fallisce e Kubernetes **riavvia tutti
i pod** — trasformando un rallentamento in un'interruzione totale. La liveness deve verificare solo
che il processo sia sano; le dipendenze esterne vanno nella readiness.

**Prometheus e Grafana** — divisione del lavoro: Prometheus raccoglie le metriche e le interroga
(PromQL), Grafana le visualizza. Le dashboard senza allarmi non servono a nulla: nessuno le guarda
alle tre di notte.

---

## 5. Sidecar e service mesh — *Avanzato*

**Sidecar** — un container affiancato a quello applicativo dentro lo stesso pod, che ne supporta il
funzionamento: proxy, monitoraggio, configurazione. Vive e muore con l'applicazione ma può essere
scritto in una tecnologia diversa.

**Service mesh** — sidecar proxy su ogni istanza, coordinati da un control plane. Fornisce in modo
uniforme e **indipendente dal linguaggio**: mTLS, routing avanzato, retry, circuit breaking,
telemetria. Data plane = i proxy, control plane = chi li configura.

**Quando ha senso**: molti servizi, in linguaggi diversi, dove riscrivere le stesse politiche in
ogni stack è insostenibile. Su una manciata di servizi tutti in Java, la complessità operativa della
mesh supera il beneficio: Resilience4j fa lo stesso lavoro con molta meno infrastruttura.

**Da segnalare in review**
- Politiche di resilienza duplicate nella mesh **e** nel codice: i comportamenti si sommano in modo
  non ovvio (3 retry dell'applicazione × 3 della mesh = 9).
- Sidecar senza limiti di risorse.
- Traffico che aggira il proxy.

**Diagnosi**
- *Latenza aggiunta su ogni chiamata* → ogni hop passa da due proxy. Va misurata, non ignorata.
- *Il pod non parte* → ordine di avvio fra sidecar e container applicativo.

---

## 6. Strategie di deployment — *Avanzato*

| Strategia | Come | Pro | Contro |
|---|---|---|---|
| **Rolling** | sostituisce le istanze progressivamente | nessun costo aggiuntivo, zero downtime | le due versioni convivono |
| **Blue-green** | due ambienti identici, si commuta il traffico | rollback istantaneo | costo doppio dell'infrastruttura |
| **Canary** | piccola percentuale di utenti sulla nuova versione | rischio minimo, si osserva prima di procedere | richiede routing fine e metriche di confronto |

**Il vincolo che accomuna rolling e canary**: durante il rilascio le due versioni coesistono e
parlano allo stesso database. Il contratto — API e schema — deve essere **retrocompatibile**. Da qui
la regola **expand and contract**: prima si aggiunge (colonna nuova, campo nuovo, entrambe le
versioni funzionano), si rilascia, e solo dopo si rimuove il vecchio in un secondo rilascio.

Una migrazione distruttiva rende il rollback impossibile, e un rollback impossibile trasforma
qualunque problema in un incidente lungo.

**Da segnalare in review**
- Rolling update senza readiness probe: traffico verso pod non pronti.
- Nessuna verifica di retrocompatibilità fra le versioni che convivranno.
- Canary senza metriche di confronto: si sta solo esponendo alcuni utenti al rischio senza
  raccogliere informazione.
- Rollback mai provato. Un rollback che non è stato testato non esiste.
