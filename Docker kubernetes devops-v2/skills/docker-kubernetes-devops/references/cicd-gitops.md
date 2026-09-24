# CI/CD, Infrastructure as Code e GitOps

Indice:
1. Cultura DevOps
2. Pipeline CI/CD
3. Infrastructure as Code
4. GitOps
5. Piattaforme gestite

---

## 1. Cultura DevOps — *Intermedio*

DevOps non è uno strumento né un ruolo: è una filosofia di collaborazione fra chi sviluppa e chi
gestisce. Se è una cultura, CI/CD è il motore automatizzato che la rende praticabile.

La conseguenza pratica da tenere presente in ogni valutazione architetturale: **senza maturità
DevOps i microservizi non funzionano.** Non è una questione di preferenze — senza automazione del
deploy e osservabilità, un sistema distribuito produce solo costi. Quando mancano, la
raccomandazione corretta è il monolite modulare, non "microservizi fatti meglio".

**Da segnalare**
- Team separati per sviluppo e operations con handover formale.
- Rilasci concentrati in finestre rare e rischiose.
- Nessuna metrica sul processo di consegna (frequenza di deploy, tempo di ripristino, tasso di
  fallimento dei cambiamenti).
- Incidenti chiusi senza azioni di miglioramento tracciate. I post-mortem devono essere non
  punitivi, altrimenti smettono di essere veritieri e diventano inutili.

---

## 2. Pipeline CI/CD — *Intermedio*

**Gli stage**

```
1. BUILD    → compila e produce l'artefatto
2. TEST     → test automatici; se falliscono, la pipeline si ferma
3. ANALYZE  → analisi statica, scansione delle dipendenze e dell'immagine
4. PACKAGE  → costruisce l'immagine e la pubblica sul registry
5. DEPLOY   → aggiorna il deployment con il nuovo tag
```

**Continuous delivery** — l'artefatto è sempre pronto al rilascio, che avviene con
un'approvazione. **Continuous deployment** — supera tutti gli stage e arriva in produzione senza
intervento umano. La differenza è il gate, ed è una scelta di rischio, non di maturità: molti team
maturi tengono il gate consapevolmente.

**Da segnalare in review**
- Pipeline che non fallisce quando i test falliscono: allora i test sono decorazione.
- Build non riproducibile, che dipende dallo stato dell'agente (versioni degli strumenti non
  fissate).
- Credenziali in chiaro nella definizione della pipeline.
- Tag dell'immagine non legato al commit: impedisce di sapere cosa sta girando.
- Nessun gate prima della produzione, in contesti dove il rischio lo richiederebbe.

**Diagnosi**
- *Funziona in locale ma non in pipeline* → dipendenza dall'ambiente dello sviluppatore.
- *Il deploy non corrisponde al codice* → il tag pubblicato e quello applicato divergono. Derivarli
  entrambi dallo stesso commit elimina la classe di problema.

---

## 3. Infrastructure as Code — *Avanzato*

L'infrastruttura descritta in file versionati e applicata in modo ripetibile, invece che
configurata a mano dalla console.

```
1. DEFINE  → file dichiarativi (Terraform in HCL, o CDK in un linguaggio di programmazione)
2. PLAN    → mostra cosa verrà creato, modificato o distrutto
3. APPLY   → applica le modifiche
```

Due famiglie di strumenti: quelli **dichiarativi** (Terraform, CloudFormation, ARM, Bicep) e quelli
che permettono di definire l'infrastruttura in un **linguaggio reale** (AWS CDK in TypeScript,
Python, C#, Java) generando poi i template. La seconda famiglia dà astrazione e riuso al prezzo di
un livello di indirezione in più.

**La gestione dello stato è la parte critica.** Lo stato registra la corrispondenza fra il codice e
le risorse reali. Va tenuto in un backend condiviso e con lock, mai in locale e mai committato.

**Da segnalare in review**
- Risorse create a mano e mai descritte nel codice: è il **drift**, e si scopre quando il piano
  propone di distruggere qualcosa che serve.
- File di stato locale o nel repository.
- Segreti nelle variabili in chiaro.
- Applicazione senza la fase di piano.
- Nessuna separazione fra ambienti.

**Diagnosi**
- *Il piano propone di distruggere risorse esistenti* → drift fra stato reale e codice. Importare le
  risorse o allineare il codice, mai applicare alla cieca.
- *Applicazioni concorrenti che si sovrascrivono* → manca il lock sullo stato condiviso.

---

## 4. GitOps — *Avanzato*

La differenza rispetto al CI/CD tradizionale sta nella **direzione**:

```
CI/CD tradizionale:   [pipeline] ──push──> [cluster]
GitOps:               [Git] <──pull──── [operatore dentro il cluster]
```

**I quattro principi**
1. Lo stato desiderato del sistema è **dichiarativo**.
2. È **versionato in Git**, unica fonte di verità.
3. I cambiamenti approvati vengono **applicati automaticamente** dall'operatore.
4. L'operatore **riconcilia continuamente** lo stato reale con quello desiderato.

Il quarto è il più potente: se qualcuno modifica manualmente il cluster, GitOps annulla la modifica.
Per cambiare qualcosa bisogna passare da Git. Questo elimina strutturalmente il drift.

**Il workflow**
```
1. CHANGE  → pull request che modifica un manifest (es. la versione dell'immagine)
2. REVIEW  → approvazione
3. SYNC    → al merge, l'operatore GitOps rileva la modifica
4. APPLY   → la applica al cluster
```

Strumenti: Argo CD, Flux.

**Un beneficio di sicurezza spesso trascurato**: nel modello pull, la pipeline di CI non ha bisogno
delle credenziali di accesso al cluster. È l'operatore, già dentro, a tirare. Questo riduce
significativamente la superficie di attacco.

**Da segnalare in review**
- Modifiche applicate direttamente al cluster con comandi imperativi.
- Credenziali del cluster distribuite alla CI in un contesto GitOps: non servono.
- Repository di configurazione mescolato con quello applicativo senza struttura per ambiente.
- Segreti in chiaro nel repository: servono segreti cifrati o un gestore esterno.

**Diagnosi**
- *Modifica manuale che sparisce dopo pochi minuti* → è la riconciliazione. Va fatta in Git.
- *Cluster non allineato* → l'operatore non vede il repository, o la sincronizzazione automatica è
  disattivata.
- *Come si fa rollback* → si ripristina il commit precedente. Il cluster non si tocca.

---

## 5. Piattaforme gestite — *Intermedio*

Tre livelli di controllo, con costi operativi molto diversi:

| Livello | Cosa gestisci tu | Quando ha senso |
|---|---|---|
| Container gestiti | l'immagine e la configurazione | pochi servizi, team piccolo |
| Kubernetes gestito | manifest, chart, politiche | molti servizi, team con competenze di piattaforma |
| Serverless | il codice della funzione | carichi a eventi, traffico irregolare |

I **backing service** — database, cache, broker — come risorse gestite sono quasi sempre la scelta
giusta: gestire Kafka o PostgreSQL in produzione è un mestiere a sé.

**Da segnalare in review**
- Configurazione specifica del provider sparsa nel codice applicativo.
- Risorse create manualmente dalla console e non riproducibili.
- Nessuna separazione di rete fra ambienti.
- Nessun budget né allarme di costo. È la voce che manca più spesso e produce le sorprese più
  sgradevoli.

**Diagnosi**
- *Costi inattesi* → risorse lasciate attive dopo i test, o dimensionamento sovrastimato.
- *Servizio che non parte in cloud ma va in locale* → variabili d'ambiente e stringhe di connessione
  non iniettate come previsto.
