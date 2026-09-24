# Decomposizione, confini e scalabilità

Indice:
1. Scegliere lo stile architetturale
2. Decomposizione per business capability e per sottodominio
3. Validare i confini proposti
4. Strangler Fig: migrazione incrementale
5. Scalabilità e servizi stateless
6. Load balancing e consistent hashing
7. Serverless

---

## 1. Scegliere lo stile architetturale — *Avanzato*

L'evoluzione naturale è **monolite → monolite modulare → microservizi**, e ogni passo va giustificato
da un problema misurato, non anticipato.

| Stile | Quando è la scelta giusta | Cosa costa |
|---|---|---|
| Monolite | dominio non ancora capito, team piccolo, time to market | diventa un big ball of mud se non si difendono i confini interni |
| Monolite modulare | confini di dominio chiari ma nessuna necessità di deployment indipendente | resta un unico deployable: si scala tutto insieme |
| Microservizi | parti che scalano diversamente, team indipendenti, cicli di rilascio divergenti | rete, coerenza eventuale, osservabilità, maturità DevOps |

**Principi da applicare prima di complicare**
- **KISS** — la soluzione più semplice che funziona.
- **YAGNI** — non costruire per requisiti che non esistono ancora.
- **DRY** — con un'avvertenza nei sistemi distribuiti: un po' di duplicazione fra servizi è preferibile
  a un accoppiamento che li lega. Estrarre una libreria condivisa per evitare dieci righe duplicate
  crea una dipendenza di rilascio fra servizi che dovevano essere indipendenti.

**Il monolite modulare in pratica**

Moduli autonomi dentro un unico deployable, ciascuno con la propria struttura interna
(`Data/`, `Services/`, `Endpoints/`) e il proprio **schema di database separato** dentro lo stesso
server. Gli schemi separati sono la best practice chiave: mantengono i benefici ACID e la semplicità
operativa, ma rendono esplicito chi possiede cosa, così l'estrazione futura è un'operazione
meccanica invece di un'archeologia.

I moduli si registrano e si montano dal punto di ingresso dell'applicazione
(`AddCatalogModule()` / `UseCatalogModule()` o l'equivalente Spring), non definiscono endpoint
sparsi nel progetto host.

**Da segnalare in review**
- Microservizi adottati senza pipeline CI/CD né automazione.
- Moduli che si chiamano attraversando i livelli invece di passare dal proprio confine pubblico.
- Schema di database unico in un sistema dichiarato modulare.
- Complessità introdotta per requisiti non ancora esistenti.

---

## 2. Decomposizione per business capability e per sottodominio — *Avanzato*

I confini si derivano dal **dominio di business**, non dalla struttura tecnica del codice.

**Decompose by business capability** — un servizio per ogni cosa che il business *fa*:
catalogo prodotti, carrello, ordini, identità, pagamenti, spedizioni, notifiche.

**Decompose by subdomain (DDD)** — un servizio per ogni sottodominio, con il suo **bounded context**
e il suo linguaggio. Lo stesso termine può significare cose diverse in contesti diversi: "ordine" per
Ordering è un aggregato con stato e articoli, per Shipping è un indirizzo e un peso. Il bounded
context è ciò che rende legittima questa divergenza invece che una contraddizione.

Le due strategie convergono quasi sempre sullo stesso taglio. Quando divergono, la capability dice
*cosa* separare, il sottodominio dice *dove* passa esattamente la linea.

**Da segnalare in review**
- Servizi organizzati per livello tecnico: un servizio per i controller, uno per il data access.
  Questo non è decomporre, è distribuire un monolite a strati.
- Entità condivise fra servizi.
- Un servizio che non può essere deployato senza un altro.
- Troppe chiamate incrociate fra due servizi.

---

## 3. Validare i confini proposti — *Avanzato*

Prima di accettare una proposta di decomposizione, passare queste domande. Ognuna che riceve una
risposta vaga è un confine da ridiscutere.

1. **Autonomia dei dati** — il servizio possiede tutti i dati che gli servono per le sue decisioni?
2. **Dati condivisi** — quelli di cui più servizi hanno bisogno, come vengono gestiti? Le tre
   strategie legittime sono: replica via eventi, chiamata API al proprietario, vista materializzata.
   "Leggiamo dalla sua tabella" non è una di queste.
3. **Coesione** — le cose che cambiano insieme stanno insieme?
4. **Accoppiamento** — quante chiamate servono per un'operazione di business?
5. **Deployment autonomo** — è il test cruciale. Si può rilasciare questo servizio da solo, in
   qualunque momento?
6. **Proprietà** — esiste un team che lo possiede end-to-end?
7. **Transazioni** — quali operazioni attraversano il confine? Ognuna diventerà una Saga.

---

## 4. Strangler Fig — *Avanzato*

Estrarre dal monolite un pezzo alla volta, senza rewrite a big bang.

```
Passo 0   SPA → [monolite] → DB condiviso

Passo 1   SPA → [FACADE] ─┬→ [Identity Service] → Identity DB
                          └→ [monolite] → DB condiviso

Passo 2   SPA → [FACADE] ─┬→ [Identity Service] → Identity DB
                          ├→ [Basket Service]   → Redis
                          └→ [monolite] → DB condiviso
```

Per ogni modulo: si definisce il confine, si realizza il servizio con **database dedicato**, si
migrano i dati, si aggiorna la facade per dirottare quel traffico. Il monolite si svuota
progressivamente.

**Da segnalare in review**
- Servizio estratto che continua a leggere il database del monolite: non è stato estratto, è stato
  spostato.
- Facade che accumula logica di business.
- Nessuna strategia di rollback per singola funzionalità.
- Migrazione dei dati non pianificata.

**Diagnosi**
- *Doppia scrittura durante la transizione* — un dato modificato da entrambi i lati. Serve una
  fonte di verità dichiarata per ogni entità e una sincronizzazione esplicita, tipicamente basata
  su eventi.

---

## 5. Scalabilità e servizi stateless — *Intermedio*

**Scale cube** — tre assi indipendenti:
- **X: cloning** — più istanze identiche dietro un load balancer. Il più semplice, il primo da usare.
- **Y: decomposizione funzionale** — sono i microservizi stessi.
- **Z: partizionamento dei dati** — sharding per chiave.

| | Verticale (scale up) | Orizzontale (scale out) |
|---|---|---|
| Come | macchina più potente | più istanze |
| Semplicità | alta, spesso nessuna modifica al codice | richiede servizi stateless |
| Limite | fisico e di costo, cresce in modo non lineare | molto più alto |
| Disponibilità | resta un single point of failure | ridondanza nativa |

Il prerequisito dello scale out è che il servizio sia **stateless**, o che lo stato sia
esternalizzato su cache distribuita o database accessibile a tutte le istanze.

**Da segnalare in review**
- Stato di sessione in memoria locale dell'istanza.
- File caricati sul filesystem del singolo nodo.
- Job schedulati che partono su tutte le repliche.
- Scaling orizzontale applicato al layer applicativo mentre il collo di bottiglia reale è il
  database. È il caso più comune: si replica l'API e non cambia niente, perché tutte le repliche
  puntano allo stesso PostgreSQL.

**Diagnosi**
- *Comportamento incoerente dopo l'aumento delle repliche* → stato locale non condiviso.
- *Nessun miglioramento dopo lo scale out* → il limite si è spostato a valle. Misurare prima di
  replicare ancora.

---

## 6. Load balancing e consistent hashing — *Avanzato*

Il load balancer deve conoscere le istanze **vive**: dialoga con il service registry, non con una
lista statica.

Nel modello **client-side** (Spring Cloud LoadBalancer) è il chiamante a scegliere l'istanza,
dopo aver chiesto al registry la lista. Il registry fornisce solo la lista: non vede passare il
traffico applicativo. Nel modello **server-side** un componente intermedio riceve e smista.

**Consistent hashing** — per distribuire dati (non richieste) su un cluster elastico. Con
`hash % N`, aggiungere o togliere un nodo rimescola quasi tutte le chiavi; con il consistent
hashing si rimescola solo la frazione corrispondente al nodo cambiato. I nodi virtuali migliorano
l'uniformità della distribuzione.

**Da segnalare in review**
- Bilanciamento su una lista statica invece che sul registry.
- `hash % N` per assegnare dati ai nodi.
- Health check assenti: traffico verso istanze non pronte.

**Diagnosi**
- *Aggiungendo un nodo la cache si azzera quasi tutta* → hashing modulo N.
- *Traffico non distribuito* → istanze registrate con lo stesso identificatore, o il client sta
  risolvendo un indirizzo fisso invece del nome logico.

Nota storica utile in review: **Netflix Ribbon è deprecato**. Sul codice nuovo si usa Spring Cloud
LoadBalancer. Lo stesso vale per Hystrix, sostituito da Resilience4j.

---

## 7. Serverless — *Intermedio*

L'unità di deployment diventa la funzione. Il provider gestisce server, scaling e disponibilità.

Tre tipi di invocazione, che corrispondono a tre stili di comunicazione:
- **sincrona** — richiesta HTTP attraverso un API Gateway
- **asincrona** — evento su un event bus
- **event source mapping** — la funzione fa polling su una coda o su uno stream

**Cosa il serverless semplifica**: scaling, alta disponibilità, gestione dell'infrastruttura,
versionamento delle funzioni.

**Cosa non semplifica**: i confini del dominio, la coerenza dei dati, l'osservabilità. Restano
identici.

**Da segnalare in review**
- Funzioni che mantengono stato fra invocazioni.
- Timeout e memoria lasciati ai default.
- Una funzione monolitica che gestisce decine di route.
- Consumer di eventi non idempotenti: le consegne possono ripetersi.

**Diagnosi**
- *Cold start su percorsi sensibili alla latenza* → concorrenza riservata, o un servizio always-on.
- *Costi imprevisti* → invocazioni in loop fra funzione ed evento che essa stessa produce.
