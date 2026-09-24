# Pattern — manifest Kubernetes

Manifest completi per un servizio Spring Boot. Adatta nomi, immagine, porte e risorse dopo il
preflight.

**Servono sempre almeno due oggetti**: un `Deployment` e un `Service`. Applicare solo il primo crea
i pod ma non li rende raggiungibili — è l'errore che il corso segnala esplicitamente.

Indice:
1. Deployment
2. Service
3. ConfigMap e Secret
4. Probe
5. Ingress
6. HorizontalPodAutoscaler
7. Struttura dei file e comandi
8. Diagnosi

---

## 1. Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  labels:
    app: product-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service          # DEVE corrispondere alle label del template
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0             # nessuna riduzione di capacità durante il rollout
  template:
    metadata:
      labels:
        app: product-service        # il filo che lega il Deployment ai suoi pod
    spec:
      containers:
        - name: product-service
          image: registry.example.com/team/product-service:0.0.3   # mai :latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080   # la porta REALE dell'applicazione
          envFrom:
            - configMapRef:
                name: product-service-config
            - secretRef:
                name: product-service-secret
          env:
            - name: JAVA_OPTS
              value: "-XX:MaxRAMPercentage=75.0"
          resources:
            requests:               # quanto serve per essere schedulato
              memory: "512Mi"
              cpu: "250m"
            limits:                 # il tetto: superarlo significa OOMKilled
              memory: "1Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 3
```

**`selector.matchLabels` e `template.metadata.labels` devono corrispondere.** Se divergono, il
Deployment non gestisce nessun pod e non c'è un messaggio d'errore chiaro.

**`maxUnavailable: 0`** garantisce che durante il rollout non si scenda sotto la capacità
dichiarata. Richiede una readiness probe funzionante, altrimenti il rollout si blocca.

**`requests` e `limits` non sono opzionali.** Senza `requests` lo scheduler non sa dove mettere il
pod; senza `limits` un pod può affamare gli altri sul nodo. `MaxRAMPercentage` va allineato al
limite di memoria: senza, la JVM ignora il tetto e il container viene terminato.

**Non si creano Pod o ReplicaSet a mano.** Il Deployment li gestisce: `replicas`, rollout e rollback
passano da lui.

---

## 2. Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  type: ClusterIP                   # default: raggiungibile solo dall'interno
  selector:
    app: product-service            # seleziona i pod per label
  ports:
    - port: 80                      # porta del Service
      targetPort: 8080              # porta del container
```

| Tipo | Accesso | Quando |
|---|---|---|
| `ClusterIP` | solo dall'interno del cluster | comunicazione fra servizi — **il default corretto** |
| `NodePort` | porta su ogni nodo | sviluppo, cluster locali |
| `LoadBalancer` | bilanciatore del cloud, IP esterno | un servizio realmente pubblico |

Gli IP dei pod sono **effimeri**: cambiano a ogni ricreazione. Il Service fornisce un punto di
accesso stabile, ed è questo che sostituisce il service registry.

Dall'interno del cluster, un altro pod chiama:
```
http://product-service              # stesso namespace
http://product-service.default.svc.cluster.local   # forma completa
```

**Non creare un `LoadBalancer` per ogni servizio interno**: costa e non serve. Un solo punto di
ingresso pubblico, tramite `Ingress`.

---

## 3. ConfigMap e Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: product-service-config
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres:5432/productdb"
  MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: "health,info,metrics"
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: product-service-secret
type: Opaque
stringData:                          # stringData accetta testo in chiaro e codifica lui
  SPRING_DATASOURCE_USERNAME: "product"
  SPRING_DATASOURCE_PASSWORD: "cambiami"
```

> **Un Secret non è cifrato: è codificato in base64.** Chiunque abbia accesso al cluster o al
> repository può leggerlo. Un Secret committato in Git è un segreto esposto. In produzione servono
> Sealed Secrets, External Secrets o un gestore esterno.

ConfigMap e Secret sono l'equivalente Kubernetes del Config Server: se il sistema gira su
Kubernetes, non servono entrambi.

**Attenzione**: le variabili d'ambiente da ConfigMap si leggono **all'avvio del pod**. Modificare la
ConfigMap non aggiorna i pod in esecuzione — serve un rollout:
```bash
kubectl rollout restart deployment/product-service
```

---

## 4. Probe

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      probes:
        enabled: true                # espone /health/liveness e /health/readiness
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

| Probe | Domanda | Se fallisce |
|---|---|---|
| `livenessProbe` | il processo è sano? | il container viene **riavviato** |
| `readinessProbe` | è pronto a ricevere traffico? | tolto dal Service, **non riavviato** |
| `startupProbe` | ha finito di avviarsi? | sospende le altre due durante l'avvio |

**L'errore da non commettere**: mappare entrambe sullo stesso endpoint che verifica il database.
Quando il database rallenta, la liveness fallisce e Kubernetes riavvia **tutti** i pod,
trasformando un rallentamento in un'interruzione totale.

La regola: la **liveness** verifica solo che il processo sia sano; le **dipendenze esterne** vanno
nella readiness. Spring Boot separa già i due endpoint con `probes.enabled: true`.

Per applicazioni lente ad avviarsi, meglio una `startupProbe` che un `initialDelaySeconds` alto
sulla liveness:
```yaml
startupProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
  failureThreshold: 30
  periodSeconds: 10          # concede fino a 5 minuti per l'avvio
```

---

## 5. Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /products(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 80
          - path: /orders(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
```

L'Ingress è l'equivalente Kubernetes dell'API Gateway per il routing HTTP: un solo punto di ingresso
pubblico, TLS terminato lì, i servizi interni restano `ClusterIP`.

Richiede un **ingress controller** installato nel cluster: senza, il manifest si applica e non
succede nulla.

---

## 6. HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

Richiede `resources.requests` sul Deployment — l'utilizzo si calcola in percentuale della request —
e il metrics-server installato nel cluster.

Lo scaling orizzontale funziona solo se il servizio è **stateless**. Se tiene stato in memoria
locale, replicarlo produce comportamenti incoerenti: il problema è nel codice, non nell'HPA. Vedi
`microservices-architecture/references/decomposizione.md`.

---

## 7. Struttura dei file e comandi

```
k8s/
├── deployment.yaml
├── service.yaml
├── configmap.yaml
├── secret.yaml          # in produzione: cifrato o gestito esternamente
├── ingress.yaml
└── hpa.yaml
```

```bash
kubectl apply -f k8s/                          # applica l'intera cartella
kubectl get all -l app=product-service
kubectl rollout status deployment/product-service
kubectl rollout undo deployment/product-service
kubectl logs -f deployment/product-service
kubectl describe pod <nome-pod>                # la sezione Events è dove sta la risposta
kubectl port-forward svc/product-service 8080:80   # test locale senza esporre nulla
```

Per estrarre un manifest da una risorsa creata con comandi imperativi e portarlo sotto controllo di
versione:
```bash
kubectl get deployment product-service -o yaml > deployment.yaml   # poi vanno ripuliti
                                                                   # status e campi generati
```

---

## 8. Diagnosi

| Stato / sintomo | Causa |
|---|---|
| `Pending` | risorse insufficienti sul nodo, o `requests` troppo alte |
| `ImagePullBackOff` | immagine o tag inesistente, o credenziali del registry mancanti |
| `InvalidImageName` | errore di sintassi nel nome dell'immagine |
| `CrashLoopBackOff` | l'applicazione muore all'avvio: `kubectl logs --previous` |
| `OOMKilled` | limite di memoria superato: heap JVM non allineato a `limits` |
| Pod `Running` ma non raggiungibile | Service assente, o selector che non seleziona nulla |
| `kubectl describe svc` mostra endpoint vuoti | le label del Service non corrispondono a quelle dei pod |
| Rollout fermo a metà | la nuova versione non passa mai la readiness |
| Riavvii a catena sotto carico | liveness che verifica una dipendenza esterna |
| ConfigMap modificata senza effetto | serve `kubectl rollout restart` |
| Variabile d'ambiente del Service assente | Service creato dopo il pod: le variabili si iniettano all'avvio |

Sui nomi delle variabili d'ambiente generate automaticamente per i Service: i **trattini diventano
underscore** e tutto è in maiuscolo — `product-service` diventa `PRODUCT_SERVICE_SERVICE_HOST`.

Il primo comando da eseguire davanti a un pod che non funziona è sempre
`kubectl describe pod <nome>`: la sezione **Events** contiene quasi sempre la risposta.
