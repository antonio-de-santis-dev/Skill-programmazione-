# Concorrenza e secure coding in Java

Indice:
1. Concorrenza classica: thread, executor, thread safety
2. Concorrenza moderna: virtual thread, scoped value, gatherer
3. Secure coding e progettazione difensiva

---

## 1. Concorrenza classica — *Avanzato*

**Da scrivere così**
- Non creare thread a mano. Usare `ExecutorService`:
  `newSingleThreadExecutor`, `newFixedThreadPool(n)`, `newCachedThreadPool`.
- `ScheduledExecutorService` per i task ricorrenti, distinguendo `scheduleAtFixedRate` (cadenza
  fissa dall'inizio) da `scheduleWithFixedDelay` (pausa fissa dopo la fine).
- `Callable` + `Future.get(timeout, unit)` quando serve un risultato, gestendo i tre esiti distinti:
  `TimeoutException` (troppo lento), `ExecutionException` (il task ha lanciato un'eccezione),
  `InterruptedException` (l'attesa è stata interrotta).
- Contatori e flag condivisi: `AtomicInteger`, `AtomicLong`, `AtomicBoolean`.
- Sezioni critiche: `synchronized`, oppure `Lock`/`ReentrantLock` quando serve `tryLock` non
  bloccante o un lock con timeout.

**Da segnalare in review**
- Executor mai chiuso con `shutdown()`: l'applicazione non termina.
- `lock()` senza `try { ... } finally { lock.unlock(); }`.
- `synchronized` su un oggetto che non è effettivamente condiviso. Se ogni thread ha la propria
  istanza, il lock è su oggetti diversi e non c'è alcuna mutua esclusione. Per il lock di classe
  serve `synchronized(NomeClasse.class)` o un metodo `static synchronized`.
- Operazioni composte read-modify-write su variabili non atomiche (`count++` è tre operazioni).
- `HashMap` condivisa fra thread senza sincronizzazione.

**Diagnosi**
- *Contatore che perde incrementi sotto carico* → data race. `AtomicInteger` risolve il singolo
  incremento; se la logica coinvolge più variabili correlate, serve un lock sull'intero blocco.
- *Applicazione che non si chiude* → scheduler o executor attivo senza `shutdown`.
- *`TimeoutException` frequenti* → distinguere se il task è lento o se il pool è saturo e il task
  non è ancora partito.

Keyword: `ExecutorService` · `Executors.newFixedThreadPool` · `Callable`/`Future` ·
`AtomicInteger` · `synchronized` · `ReentrantLock` · `tryLock` · `InterruptedException`

---

## 2. Concorrenza moderna (Java 21 / 25) — *Avanzato*

**Virtual thread**

Un virtual thread consuma un thread del sistema operativo solo mentre calcola. Quando si blocca su
I/O lo rilascia. Il guadagno è quindi sui carichi **I/O-bound**, non su quelli CPU-bound.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> executor.submit(task));
}
```
Con un pool fisso di platform thread, 10.000 task bloccanti portano a `OutOfMemoryError`; con i
virtual thread no.

**Scoped value**

Sostituisce `ThreadLocal`, che ha tre difetti: il binding è mutabile, la vita del valore non è
limitata a uno scope, e con i pool di thread il valore può sopravvivere al task.

```java
static final ScopedValue<String> userId = ScopedValue.newInstance();
ScopedValue.where(userId, "SK001").run(() -> { /* qui userId è leggibile e immutabile */ });
```

**Stream gatherer**

Operazione intermedia custom quando `filter`/`map` non bastano: deduplica per chiave, finestre
scorrevoli, accumulo con stato. Composta da initializer (lo stato), integrator (l'elaborazione per
elemento), combiner (l'unione degli stati in parallelo), finisher.

**Da segnalare in review**
- Virtual thread usati per carichi CPU-bound: nessun beneficio, solo complessità.
- Blocchi `synchronized` lunghi dentro un virtual thread: immobilizzano il carrier thread e
  annullano il vantaggio. Preferire `ReentrantLock`.
- `ThreadLocal` senza `remove()` in codice che gira su un pool: il valore resta per il task
  successivo.
- Gatherer con stato condiviso e combiner assente o errato, usato in una pipeline parallela.

**Diagnosi**
- *`OutOfMemoryError` con molti task bloccanti su pool fisso* → passare ai virtual thread.
- *Contesto che scompare fra chiamate* → lo `ScopedValue` è valido solo dentro il blocco
  `where(...).run(...)`.
- *Risultati non deterministici in parallelo* → il gatherer ha stato condiviso senza combiner.

Keyword: `newVirtualThreadPerTaskExecutor` · carrier thread · `ScopedValue.where` ·
`Stream.gather` · `Gatherer`

---

## 3. Secure coding e progettazione difensiva — *Avanzato*

### Estensibilità e immutabilità

Rendere `final` (o sigillata) una classe che non è progettata per essere estesa. L'ereditarietà non
prevista è un vettore: un sottotipo può sovrascrivere un metodo e cambiare gli invarianti.

Un oggetto immutabile non basta dichiararlo. Il tranello sono i **campi mutabili**: se il costruttore
memorizza il riferimento a una `List` ricevuta dall'esterno, chi l'ha passata continua a puntare allo
stesso oggetto e può modificarlo dopo. Lo stesso vale in uscita, con un getter che restituisce il
riferimento interno.

Serve la **copia difensiva in entrambe le direzioni**:

```java
private Department(List<String> employees) {
    this.employees = new ArrayList<>(employees);   // copia in INGRESSO
}
public List<String> getEmployees() {
    return new ArrayList<>(this.employees);        // copia in USCITA
}
```

I campi di tipo immutabile (`String`, `Integer`, `LocalDate`) non hanno bisogno di copia. I tipi
mutabili (`ArrayList`, `StringBuilder`, array, `Date`) sì, sempre.

### Injection

Non costruire mai una query concatenando stringhe con input esterno.

```java
// Vulnerabile
String sql = "SELECT * FROM users WHERE name = '" + name + "'";

// Sicuro
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE name = ?");
ps.setString(1, name);
```

Lo stesso principio vale ovunque un input esterno finisca in un interprete: comandi di sistema,
percorsi di file, espressioni, template.

### Denial of service

Un attacco DoS non richiede necessariamente un attaccante: basta un input non limitato. Limitare
sempre dimensione, profondità e tempo di elaborazione di ciò che arriva dall'esterno — body delle
richieste, file caricati, file letti da disco, espressioni regolari su input arbitrario.

### Informazioni riservate

Azzerare i riferimenti a password, token e chiavi appena non servono più. Non scriverli nei log,
nei messaggi di errore o negli attributi di traccia.

**Checklist di review**
- Getter che restituiscono direttamente una collezione o un array interno.
- Costruttori che memorizzano il riferimento ricevuto senza copiarlo.
- Query, comandi o path costruiti per concatenazione.
- Lettura di file o body senza tetto di dimensione.
- Credenziali in campi `String` di lunga vita, o nei log.
- Classi estendibili senza motivo.

**Diagnosi**
- *Stato di un oggetto "immutabile" che cambia* → manca una delle due copie difensive.
- *Comportamento anomalo dopo input utente* → cercare concatenazioni verso un interprete.
- *Memoria che esplode su un endpoint* → input non limitato; applicare un tetto e lo streaming a
  blocchi.

Keyword: defensive copy · `final class` · `PreparedStatement` · bind variable · injection ·
denial of service · immutabilità
