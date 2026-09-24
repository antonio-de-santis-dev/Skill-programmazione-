# Kubernetes

Indice:
1. Gli oggetti fondamentali
2. Esposizione e discovery interna
3. Aggiornamenti, rollout e rollback
4. Manifest dichiarativi, ConfigMap e Secret
5. Architettura del cluster
6. Helm

---

## 1. Gli oggetti fondamentali — *Intermedio*

La catena da tenere ferma: **Deployment → ReplicaSet → Pod**.

- **Pod** — l'unità minima schedulabile. Contiene uno o più container che condividono rete e
  volumi. Oltre al container principale può ospitare **init container** (girano prima, preparano
  l'ambiente) e **sidecar** (girano in parallelo: monitoraggio, proxy, configurazione).
- **ReplicaSet** — garantisce che il numero desiderato di pod sia sempre attivo. **Non si crea e
  non si tocca a mano**: lo crea e lo gestisce il Deployment.
- **Deployment** — l'oggetto di livello superiore. Si dichiara "voglio 3 repliche di questa immagine"
  e lui si occupa di ReplicaSet e pod, più aggiornamenti e rollback.

```bash
kubectl create deployment mio-servizio --image=utente/mio-servizio:0.0.1
kubectl scale deployment mio-servizio --replicas=3

kubectl get pod -o wide          # pod, nodo, IP, stato
kubectl describe pod <id>        # dettaglio: nome del container, e soprattutto gli Events
kubectl get rs                   # ReplicaSet: desiderati / correnti / pronti
kubectl get all                  # tutto insieme
```

Le **label** sono il filo che lega il Deployment ai suoi pod, attraverso il selector. Se non
corrispondono, il Deployment non gestisce nulla.

**Da segnalare in review**
- Pod o ReplicaSet creati a mano.
- Selector non allineato alle label del template.
- Una sola replica per servizi critici.
- `requests` e `limits` assenti.
- Nessuna probe configurata.

**Diagnosi**
- *Pod cancellato che si ricrea da solo* → è il ReplicaSet che fa il suo lavoro. Per rimuoverlo
  davvero si agisce sul Deployment.
- *Desiderati ≠ pronti* → leggere la sezione **Events** di `kubectl describe pod`. È lì che sta
  quasi sempre la risposta.
- *Pod in Pending* → risorse insufficienti sul nodo, o selector/taint non soddisfatti.

---

## 2. Esposizione e discovery interna — *Intermedio*

Gli IP dei pod sono **effimeri**: cambiano a ogni ricreazione. Il **Service** fornisce un punto di
accesso stabile a un insieme logico di pod.

```bash
kubectl expose deployment mio-servizio --type=LoadBalancer --port=8081
kubectl get services
```

| Tipo | Accesso | Ha EXTERNAL-IP |
|---|---|---|
| **ClusterIP** | solo dall'interno del cluster | no |
| **NodePort** | porta su ogni nodo | no |
| **LoadBalancer** | dall'esterno, tramite il bilanciatore del cloud | sì |
| **Ingress** | routing HTTP verso più servizi da un solo punto | — |

**Il Service sostituisce il service registry.** Su Kubernetes, `Service` + DNS interno svolgono il
ruolo che in Spring Cloud avevano Eureka e Ribbon. Trascinare Eureka dentro Kubernetes duplica un
meccanismo già presente.

| Spring Cloud | Su Kubernetes |
|---|---|
| Eureka | `Service` + DNS interno |
| Ribbon / load balancer | `Service` ClusterIP / LoadBalancer |
| Spring Cloud Gateway | `Service` LoadBalancer, o `Ingress` |
| Config Server | `ConfigMap` e `Secret` |

Per raggiungere un servizio da un altro pod si usa il suo nome DNS. Kubernetes inietta anche
variabili d'ambiente per ogni Service: nei nomi, **i trattini diventano underscore** e tutto è in
maiuscolo (`mio-servizio` → `MIO_SERVIZIO_SERVICE_HOST`).

**Da segnalare in review**
- Deployment applicato senza il relativo Service: i pod girano ma sono irraggiungibili.
- IP di pod usati come indirizzo di destinazione.
- `LoadBalancer` creato per ogni servizio interno: costa e non serve, basta `ClusterIP`.

**Diagnosi**
- *Servizio non raggiungibile da un altro pod* → nome DNS errato, selector che non seleziona alcun
  pod (`kubectl describe service` mostra gli endpoint: se sono vuoti, è quello), o porta sbagliata.
- *Variabile d'ambiente del Service assente* → il Service è stato creato **dopo** il pod. Le
  variabili si iniettano all'avvio: va ricreato il pod.

---

## 3. Aggiornamenti, rollout e rollback — *Intermedio*

```bash
kubectl set image deployment/<nome> <nome-container>=<immagine>:<tag>
kubectl rollout status deployment/<nome>
kubectl rollout undo deployment/<nome>       # torna alla revisione precedente
```

Il nome del container non coincide necessariamente con quello del deployment: si legge sotto
`Containers` in `kubectl describe pod`.

**Da segnalare in review**
- Aggiornamenti applicati con comandi imperativi e mai riportati nei manifest versionati.
- Tag `latest`: il rollout non è deterministico e il rollback non sa a cosa tornare.
- Readiness probe assente durante il rolling update.

**Diagnosi**
- *Pod nuovo in `InvalidImageName` o `ImagePullBackOff`* → nome o tag inesistente. Il vecchio
  ReplicaSet resta attivo e il servizio continua a funzionare: c'è tempo per correggere o annullare.
  `kubectl describe pod` sotto Events mostra il motivo esatto.
- *Rollout fermo a metà* → la nuova versione non passa mai la readiness.

---

## 4. Manifest dichiarativi, ConfigMap e Secret — *Avanzato*

Il modo corretto di lavorare è dichiarativo: lo stato desiderato sta in file YAML versionati.

```bash
kubectl get deployment <nome> -o yaml > deployment.yaml   # estrai e ripulisci
kubectl apply -f deployment.yaml
kubectl apply -f k8s/                                      # un'intera cartella
```

**Servono due file**: uno per il Deployment, uno per il Service. Applicare solo il primo crea i pod
ma non li rende raggiungibili.

Campi da leggere sempre in review:
- `replicas` — quanti pod.
- `selector.matchLabels` e `template.metadata.labels` — devono corrispondere.
- `strategy.type: RollingUpdate`.
- `imagePullPolicy: IfNotPresent` — scarica solo se non presente in locale.
- `resources.requests` / `resources.limits`.
- `livenessProbe` / `readinessProbe`.

**ConfigMap** per i dati di configurazione non sensibili, **Secret** per password e chiavi. Sono
l'equivalente Kubernetes del Config Server.

**Da segnalare in review**
- Modifiche applicate solo dal comando e mai riportate in Git.
- Secret usati come ConfigMap in chiaro (o Secret con valori solo codificati in base64 committati,
  che non è cifratura).
- Manifest senza namespace, senza limiti di risorse, senza label coerenti.

**Diagnosi**
- *Modifica non applicata* → `apply` su un campo immutabile, o selector cambiato.
- *Configurazione vecchia ancora attiva* → le variabili d'ambiente da ConfigMap si leggono
  all'avvio: serve un rollout per rileggerle.
- *Cluster divergente dai manifest* → qualcuno ha modificato a mano. È esattamente il problema che
  GitOps risolve.

---

## 5. Architettura del cluster — *Avanzato*

**Control plane**
- **API server** — unico punto di ingresso. Ogni `kubectl` finisce qui.
- **Scheduler** — decide su quale nodo va ogni pod.
- **Controller manager** — riconcilia continuamente stato reale e stato desiderato.
- **etcd** — il database dello stato del cluster.

**Nodi worker**
- **kubelet** — avvia e sorveglia i container sul nodo.
- **kube-proxy** — gestisce le regole di rete per i Service.

Sapere questo serve per la diagnosi: un pod che non viene schedulato è un problema dello scheduler
(risorse, taint, affinità); comandi che falliscono per tutti sono un problema dell'API server; un
nodo `NotReady` è kubelet o rete del nodo.

**Da segnalare in review**
- Carichi applicativi schedulati sui nodi del control plane.
- Nessuna quota per namespace.
- Nessuna segregazione fra ambienti.
- Accessi al cluster senza ruoli e permessi definiti.

---

## 6. Helm — *Intermedio*

Un microservizio in produzione porta con sé un YAML di Deployment, uno di Service, una ConfigMap, un
Secret, forse un Ingress. Moltiplicato per venti servizi e tre ambienti, la gestione a mano non
regge.

Helm impacchetta tutto in un **chart**: template più un file di valori. Si installa e si aggiorna con
un comando, e si differenziano gli ambienti cambiando **solo i valori**, non duplicando i manifest.

**Da segnalare in review**
- Chart duplicati per ambiente invece di un chart con file di valori diversi.
- Segreti dentro i file di valori committati.
- Template senza default sensati.
- Versione del chart non allineata a quella dell'applicazione.

**Diagnosi**
- *Installazione che fallisce sul rendering* → valore obbligatorio non fornito o template malformato.
  Ispezionare il manifest generato prima di applicarlo.
- *Upgrade che non cambia nulla* → i valori non sono stati effettivamente passati.
