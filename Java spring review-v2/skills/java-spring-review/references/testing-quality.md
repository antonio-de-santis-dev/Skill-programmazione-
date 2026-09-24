# Test, qualità del codice e controllo di versione

Indice:
1. Test automatici di un servizio Spring Boot
2. Verifica delle API con Postman
3. Code smell ricorrenti
4. Git e pull request
5. Lombok

---

## 1. Test automatici di un servizio Spring Boot — *Intermedio*

**Da scrivere così**
- `@SpringBootTest` + `@AutoConfigureMockMvc`, con `MockMvc` iniettato: permette di testare i
  controller senza avviare un server reale.
- Una classe di test per controller. Ogni metodo annotato `@Test`.
- Asserire status **e** contenuto della risposta, non solo lo status.
- Isolare i collaboratori con i mock quando il test riguarda un singolo livello.
- Database in memoria o container effimero per i test di persistenza.
- Testare sempre almeno un percorso di errore per endpoint: non trovato, input non valido,
  conflitto.

```java
@SpringBootTest
@AutoConfigureMockMvc
class CarControllerTests {
    @Autowired private MockMvc mockMvc;

    @Test
    void getCar_notFound_returns404() throws Exception { ... }
}
```

**Da segnalare in review**
- Test che dipendono dall'ordine di esecuzione o da dati preesistenti nel database.
- Contesto Spring completo caricato per testare una singola classe: usare le slice
  (`@WebMvcTest`, `@DataJpaTest`) quando basta.
- Asserzioni solo sullo status.
- Solo il percorso felice testato.
- Test disabilitati lasciati nel repository: o si sistemano o si cancellano.

**Diagnosi**
- *Passa in locale, fallisce in pipeline* → dipendenza da stato, fuso orario, ordine o risorse
  locali.
- *Contesto che non si carica* → property mancanti nel profilo di test.
- *Suite lentissima* → troppi test che caricano l'intero contesto.

---

## 2. Verifica delle API con Postman — *Base*

**Da scrivere così**
- Una collection per servizio, con variabili d'ambiente per URL, porta e token. Mai valori cablati.
- Test negli script post-richiesta (in JavaScript, non Java; l'oggetto `pm` è fornito da Postman):
  status code, contenuto del JSON, header, tempo di risposta.
- Copertura dei casi di errore, non solo del successo.
- Esecuzione della collection dentro la pipeline di CI.
- Collection versionata insieme al codice del servizio.

**Da segnalare in review**
- Token e URL cablati nelle richieste.
- Test che verificano solo lo status: un endpoint può rispondere `200` con il corpo sbagliato.

**Diagnosi**
- *Collection che funziona in locale e non in CI* → variabili d'ambiente non fornite all'esecutore.

---

## 3. Code smell ricorrenti — *Avanzato*

Da cercare sistematicamente in review, in ordine di frequenza reale:

- **Logica di business nel controller.** Il controller traduce HTTP ↔ dominio, niente di più.
- **Metodo lungo con più responsabilità.** Se per descriverlo servono due "e", va spezzato.
- **`catch` generico e vuoto.** Nasconde il problema e rende impossibile la diagnosi. Se davvero
  l'eccezione è ignorabile, il commento deve spiegare perché.
- **Magic number e stringhe duplicate.** Vanno in costanti o in configurazione.
- **Dipendenze circolari fra bean.** Confine sbagliato, non problema di framework.
- **Getter che espongono campi mutabili.** Vedi copia difensiva.
- **Commenti che spiegano cosa fa il codice** invece del perché. Il cosa lo dice il codice; il
  perché no.
- **Duplicazione fra servizi o fra layer.** Due copie divergono sempre.

Quando gli stessi rilievi tornano in ogni review, il rimedio non è ripeterli: è automatizzarli con
analisi statica e regole di build. Una review che segnala per la decima volta la formattazione sta
sprecando l'attenzione di due persone.

**Sulla dimensione delle pull request**: una PR che non si riesce a leggere non viene letta, viene
approvata. Se una PR supera qualche centinaio di righe di diff significativo, chiedere di
spezzarla è un intervento di qualità, non burocrazia.

---

## 4. Git e pull request — *Base / Intermedio*

**Il flusso**

```
working directory --git add--> staging area --git commit--> repo locale --git push--> repo remoto
```

Comandi essenziali: `git init`, `git status`, `git add .`, `git restore --staged <file>`
(la forma moderna di `git reset <file>`), `git commit -m`, `git log`, `git push`, `git pull`.

`git pull` è `git fetch` + `git merge`. Regola pratica: **pull prima di iniziare, pull prima di
pushare**.

**Branch**

```
git switch -c feature-x     # crea e si sposta (forma moderna di git checkout -b)
git switch main             # torna sul principale
git merge feature-x         # porta feature-x DENTRO il branch corrente
git branch -d feature-x     # cancella (la -D forza, usarla consapevolmente)
```

La direzione del merge è la cosa da fissare: ci si sposta prima sul branch di **destinazione**, poi
si esegue `git merge <origine>`.

**Conflitti**: aprire il file, tenere ciò che serve, **rimuovere i marcatori** `<<<<<<<`, `=======`,
`>>>>>>>`, poi `git add` e `git commit`. Committare i marcatori è l'errore più comune di chi inizia.

**Pull request**

1. `git pull` sul principale
2. `git switch -c issue-180`
3. lavorare, `git add`, `git commit`
4. `git push -u origin issue-180` (la `-u` la prima volta collega il branch locale al remoto)
5. aprire la PR, farla revisionare, mergiare, cancellare il branch

La PR non è un bottone: è il luogo della code review e il punto in cui partono i controlli
automatici. `Closes #180` nella descrizione chiude la issue al merge.

**Da segnalare in review**
- Artefatti di build, dipendenze e configurazione locale versionati: manca il `.gitignore`.
- Segreti nella storia (rimuoverli da un commit non basta, restano nella storia).
- Commit enormi con messaggi non descrittivi.
- Push diretti sul branch principale, senza protezione né controlli obbligatori.
- Branch di lunga vita mai allineati: diventano un conflitto permanente.

---

## 5. Lombok — *Base*

**Da scrivere così**
- `@AllArgsConstructor` sui bean Spring per ottenere l'iniezione via costruttore senza scrivere il
  costruttore.
- `@Data` sui DTO e sulle classi di risposta, dove genera getter, setter, `equals`, `hashCode` e
  `toString`.
- Sulle **entity JPA**, annotazioni mirate: `@Getter`, `@Setter`,
  `@ToString(exclude = "relazioneLazy")`. Mai `@Data`.
- Su Java 17+, valutare il `record` al posto di Lombok per i portatori di dati immutabili.

**Da segnalare in review**
- `@Data` su entity JPA: genera `equals`/`hashCode` su tutti i campi, comprese le relazioni lazy,
  con comportamenti imprevedibili quando l'entità è caricata parzialmente.
- Dipendenza dichiarata senza versione gestita.
- Builder usati dove un costruttore sarebbe più chiaro.

**Diagnosi**
- *`equals` che si comporta in modo strano su entità caricate parzialmente* → accessor generati su
  tutti i campi; implementare `equals` sulla sola chiave di business.
- *Compilazione che fallisce in un ambiente diverso* → annotation processor non configurato.
