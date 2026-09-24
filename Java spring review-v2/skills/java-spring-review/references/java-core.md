# Java Core (8 → 25)

Indice:
1. Lambda e interfacce funzionali
2. Stream API
3. Collections, generics, equals/hashCode
4. Record, sealed class, pattern matching
5. Text block
6. Sequenced Collections
7. Java Platform Module System
8. I/O e serializzazione
9. Localizzazione e annotazioni custom

> Verifica sempre la versione del progetto in `java-versions.md` prima di usare una funzionalità
> di questo file: `record`, `sealed`, text block e Sequenced Collections hanno versioni minime
> diverse.

---

## 1. Lambda e interfacce funzionali — *Base*

Riconoscere una interfaccia funzionale (regola SAM) e scrivere il comportamento come lambda o
method reference invece che come classe anonima.

**Da scrivere così**
- Usare le interfacce di `java.util.function` — `Predicate`, `Supplier`, `Consumer`, `Function`,
  le varianti `Bi-`, `UnaryOperator`, `BinaryOperator` — prima di inventare un'interfaccia custom.
- Passare comportamenti come parametri di metodo, non duplicare metodi che differiscono solo per
  una condizione.
- Preferire il method reference quando la lambda si limita a inoltrare gli argomenti:
  `String::concat`, `ArrayList::new`, `Collections::sort`.

**Da segnalare in review**
- Interfaccia custom a metodo singolo che replica qualcosa già presente in `java.util.function`.
- Lambda di più di 3-4 righe: va estratta in un metodo con un nome.
- Variabile catturata che viene riassegnata (deve essere `final` o effectively final).
- Method reference su un metodo sovraccarico dove il contesto non rende evidente quale versione
  viene risolta: qui la lambda esplicita è più leggibile, non meno.

**Diagnosi**
- *"variable used in lambda should be final or effectively final"* → lo stato mutabile va isolato in
  un oggetto contenitore o in un accumulatore; non aggirare il vincolo con un array di un elemento
  se la vera soluzione è `reduce` o `collect`.
- *Method reference che non compila* → controllare arità e tipi attesi dal contesto. Lo stesso
  metodo statico può risolversi come `Supplier`, `Function` o `BiFunction` a seconda di dove lo usi.

Keyword: `@FunctionalInterface` · `Predicate<T>` · `BiFunction<T,U,R>` · `UnaryOperator` ·
bound / unbound / static / constructor reference · effectively final

---

## 2. Stream API — *Intermedio*

Pipeline sorgente → operazioni intermedie → operazione terminale, con valutazione lazy ed
elaborazione verticale: ogni elemento attraversa l'intera pipeline prima che parta il successivo.

**Da scrivere così**
- `filter` prima di `map`, per non trasformare elementi che verranno scartati.
- Aggregare con `collect` e i `Collectors`: `toMap` (sempre con merge function se le chiavi possono
  ripetersi), `groupingBy`, `partitioningBy`, con fornitore di mappa esplicito quando l'ordine conta.
- Usare gli stream primitivi (`IntStream`, `LongStream`, `DoubleStream`) per i calcoli numerici, e
  `summaryStatistics()` quando servono più aggregati insieme.
- Ricordare quali terminali restituiscono `Optional` (`min`, `max`, `average`, `findFirst`,
  `reduce` senza identity) e quali no (`sum`, `count`, `reduce` con identity).

**Da segnalare in review**
- Pipeline senza operazione terminale: non esegue nulla.
- Accumulo in una collezione esterna dentro `forEach` invece di `collect`.
- `toMap` senza merge function su chiavi potenzialmente duplicate.
- `Stream<Integer>` dove serviva `IntStream`: boxing inutile su volumi grandi.
- Codice che si affida all'ordine di una `Map` restituita da `groupingBy`/`toMap`: non è garantito,
  va passato `TreeMap::new` o `LinkedHashMap::new`.
- `peek` usato per effetti collaterali reali e non per debug.

**Diagnosi**
- *`IllegalStateException: Duplicate key`* → aggiungere la funzione di merge a `toMap`.
- *Stream che non termina* → sorgente infinita (`Stream.generate`/`iterate`) senza `limit`.
- *Prestazioni basse* → riordinare le operazioni, evitare `sorted` quando basta `min`/`max`,
  eliminare il boxing.

Keyword: `stream()` · `flatMap` · `Collectors.groupingBy` · `partitioningBy` ·
`reduce(identity, accumulator)` · `Optional` · lazy evaluation

---

## 3. Collections, generics ed equals/hashCode — *Intermedio*

**Scelta della struttura dati**

| Serve | Usa |
|---|---|
| Accesso per indice, iterazione veloce | `ArrayList` |
| Inserimenti/rimozioni frequenti alle estremità | `LinkedList`, meglio `ArrayDeque` |
| Unicità senza ordine | `HashSet` |
| Unicità + ordine di inserimento | `LinkedHashSet` |
| Unicità + ordinamento automatico | `TreeSet` |
| Coda FIFO o pila LIFO | `ArrayDeque` (non `Stack`) |
| Ordine per priorità | `PriorityQueue` |
| Mappa ordinata per chiave | `TreeMap` |
| Mappa thread-safe | `ConcurrentHashMap` (non `Hashtable`) |

**Da scrivere così**
- `Comparable` per l'ordine naturale, `Comparator` per gli ordini alternativi e per l'ordinamento
  multi-campo (`Comparator.comparing(...).thenComparing(...)`).
- Wildcard bounded nelle firme pubbliche: `? extends T` per leggere, `? super T` per scrivere.
- `equals` e `hashCode` sovrascritti insieme, sempre, e basati solo su campi immutabili.

**Da segnalare in review**
- `equals` senza `hashCode` (o viceversa): rompe ogni collezione basata su hash.
- Campi mutabili usati nel calcolo dell'hash.
- `Stack` e `Hashtable` legacy dove esistono alternative migliori.
- Cast espliciti in uscita da una collezione: sintomo di generics mancanti.
- `TreeSet`/`TreeMap` popolati con tipi che non implementano `Comparable` e senza `Comparator`.

**Diagnosi**
- *Oggetto inserito in una `HashMap` e poi irreperibile* → un campo usato in `hashCode` è stato
  modificato dopo l'inserimento. L'oggetto è finito in un bucket e ora se ne cerca un altro.
- *`ClassCastException` su `TreeSet`* → passare un `Comparator` al costruttore, o implementare
  `Comparable`.
- *`ConcurrentModificationException` in un for-each* → usare `ListIterator` (che ha `add`),
  `removeIf`, `putAll` differito, o una collezione concorrente (`CopyOnWriteArrayList`,
  `CopyOnWriteArraySet`, `ConcurrentHashMap`).
- *`UnsupportedOperationException`* → la lista è immutabile (`List.of`, `Arrays.asList`):
  avvolgerla in `new ArrayList<>(...)` se serve modificarla.

Keyword: `equals`/`hashCode` · `Comparator.comparing` · `TreeSet` ·
`ConcurrentModificationException` · `? extends` / `? super` · type erasure

---

## 4. Record, sealed class e pattern matching — *Intermedio*

**Da scrivere così**
- `record` per DTO e value object, con **costruttore compatto** per la validazione:
  ```java
  public record CarRecord(String regNo, String owner) {
      public CarRecord {
          if (regNo == null || regNo.length() < 5)
              throw new IllegalArgumentException("regNo troppo corto");
      }
  }
  ```
- Gerarchie chiuse con `sealed` / `permits`, e ogni sottotipo dichiarato `final`, `sealed` o
  `non-sealed`. Il vantaggio non è il divieto in sé: è che il compilatore può verificare
  l'esaustività di uno `switch`.
- `switch` come espressione con frecce, `yield` nei blocchi, record pattern per destrutturare,
  guardie `when`, e `case null` esplicito.

**Da segnalare in review**
- `record` usato come entity JPA: JPA richiede costruttore senza argomenti e campi mutabili.
- Costruttore compatto assente dove i vincoli di dominio esistono già altrove (validazione
  duplicata nel service invece che nel tipo).
- `switch` su tipi di riferimento senza `default` e senza gerarchia sigillata: non è esaustivo.
- Ordine dei `case` che rende un ramo irraggiungibile per dominanza (il caso più generale prima
  del più specifico).

**Diagnosi**
- *"instance field is not allowed in a record"* → il campo va promosso a componente del record o
  reso `static`.
- *`switch` che non compila per esaustività* → sigillare la gerarchia o aggiungere `default`.
- *`NullPointerException` su `switch`* → aggiungere `case null`.

Keyword: `record` · `sealed` / `permits` / `non-sealed` · switch expression · `yield` ·
record pattern · `case null` · guardia `when`

---

## 5. Text block — *Base* (Java 15+)

Stringa su più righe delimitata da `"""`, senza escape e senza concatenazione.

```java
String json = """
    {
      "nome": "Sean",
      "eta": 30
    }""";
```

Regole che generano errori di compilazione se ignorate:
- I delimitatori di apertura **devono** essere seguiti da un a capo: `"""abc"""` su una riga non
  compila.
- Le virgolette interne non vanno mai "escapate".
- La posizione del **delimitatore di chiusura** stabilisce il margine sinistro: gli spazi a sinistra
  di quella colonna sono "incidentali" e vengono rimossi. Spostare il delimitatore cambia
  l'indentazione del risultato.
- Se il delimitatore di chiusura sta su una riga propria, la stringa **termina con un a capo**. Se
  sta subito dopo l'ultimo carattere, no.

Un text block **è** una `String` a tutti gli effetti: `equals`, `substring` e ogni altro metodo
funzionano normalmente.

Dove conviene davvero: JSON e SQL nei test, query JPQL lunghe, messaggi multi-riga. Sono i punti in
cui l'escape rende il codice illeggibile.

**Da segnalare in review**: JSON o SQL costruiti per concatenazione in un progetto Java 15+;
indentazione del text block che non corrisponde a quella attesa (quasi sempre è il delimitatore di
chiusura fuori posto).

---

## 6. Sequenced Collections — *Base* (Java 21+)

Prima di Java 21 non esisteva un modo uniforme per chiedere il primo e l'ultimo elemento di una
collezione ordinata: `List` usava `get(0)` e `get(size()-1)`, `SortedSet` aveva `first()`/`last()`,
`LinkedHashSet` richiedeva un iteratore, e per l'inverso servivano `descendingSet()`,
`descendingIterator()` o una copia in un'altra collezione.

Le tre nuove interfacce risolvono l'incoerenza:

| Interfaccia | Estende | Implementazioni annotate come nuove |
|---|---|---|
| `SequencedCollection<E>` | `Collection` | `List` diventa sua sotto-interfaccia |
| `SequencedSet<E>` | `Set`, `SequencedCollection` | `LinkedHashSet` |
| `SequencedMap<K,V>` | `Map` | `LinkedHashMap` |

```java
SequencedCollection<String> sc = new ArrayList<>();
sc.addFirst("A");  sc.addLast("D");
sc.getFirst();  sc.getLast();
sc.removeFirst(); sc.removeLast();
sc.reversed();                       // vista in ordine inverso, non una copia

SequencedMap<Integer,String> sm = new LinkedHashMap<>();
sm.firstEntry();  sm.lastEntry();
sm.sequencedKeySet();  sm.sequencedValues();  sm.sequencedEntrySet();
```

`reversed()` restituisce una **vista**: riflette le modifiche alla collezione originale e non ne
duplica il contenuto.

**Da segnalare in review** (solo su progetti Java 21+): `list.get(list.size() - 1)` e le copie
fatte solo per ottenere l'ordine inverso.

---

## 7. Java Platform Module System — *Avanzato*

Incapsulamento forte a livello di modulo: `public` non basta più, serve anche `exports`.

**Da scrivere così**
- `module-info.java` con `requires`, `requires transitive` (solo per ciò che fa parte dell'API
  pubblica del modulo), `exports` mirati, `opens` solo dove serve la reflection.
- Disaccoppiare con i servizi: un modulo API che esporta l'interfaccia, moduli di implementazione
  che dichiarano `provides ... with ...` senza esportare la classe concreta, consumatori che
  dichiarano `uses` e risolvono con `ServiceLoader`.
- La forma lazy di `ServiceLoader` evita di istanziare tutte le implementazioni:
  ```java
  ServiceLoader.load(SoftDrinkService.class).stream()
      .filter(p -> p.type() == SmallSoftDrinkService.class)  // nessuna istanza creata
      .map(ServiceLoader.Provider::get)                       // istanza creata qui
  ```

**Da segnalare in review**
- `exports` su package di implementazione.
- `opens` generalizzato all'intero modulo per far funzionare un framework, invece che mirato.
- Dipendenza dichiarata `requires` semplice quando i suoi tipi compaiono nelle firme pubbliche
  (deve essere `requires transitive`, altrimenti il consumatore deve dichiararla anche lui).
- Automatic module (jar senza `module-info`) usati come soluzione stabile e non di transizione.

**Diagnosi**
- *Classe `public` non accessibile* → manca `exports` nel modulo produttore.
- *Framework che fallisce in reflection* → serve `opens` verso quel modulo.
- *Migrazione da classpath* → partire da `jdeps -s --generate-module-info <dir> <jar>` per ricavare
  i `requires` reali invece di dichiararli a mano.

Keyword: `module-info.java` · `requires transitive` · `exports` / `opens` · `provides ... with` ·
`uses` · `ServiceLoader` · `jdeps` · `java --describe-module`

---

## 8. I/O, NIO.2 e serializzazione — *Base*

**Da scrivere così**
- `Path` / `Files` per creare, copiare, spostare, leggere. `Path.resolve` invece di concatenare
  stringhe.
- Ogni risorsa in `try-with-resources`.
- Charset dichiarato esplicitamente nelle letture e scritture.
- `transient` sui campi che non devono essere serializzati.

**Da segnalare in review**
- Stream chiusi a mano in `finally`.
- File letti interamente in memoria quando potrebbero essere grandi: usare `Files.lines` con
  try-with-resources, o leggere a blocchi.
- Classi `Serializable` che espongono campi sensibili senza `transient`.

**Diagnosi**
- *File lock o descriptor esauriti* → risorsa non chiusa su un percorso di eccezione.
- *`NotSerializableException`* → un campo raggiungibile non è serializzabile.
- *Caratteri corrotti* → charset non dichiarato, si sta usando quello di default della piattaforma.

Keyword: `Path` / `Files` · `try-with-resources` · `BufferedReader` · `Serializable` · `transient`

---

## 9. Localizzazione e annotazioni custom — *Base*

**Da scrivere così**
- `Locale` esplicito preso dalla richiesta, non quello di default della JVM.
- `ResourceBundle` per i messaggi, `NumberFormat`/`DecimalFormat` per numeri e valute,
  `DateTimeFormatter` con `FormatStyle` per le date.
- Annotazioni custom con `@interface`, `@Target` e `@Retention` coerenti con l'uso previsto;
  `@Repeatable` e `@Inherited` dove servono.

**Da segnalare in review**
- Stringhe utente, formati numerici o pattern di data hardcoded.
- Annotazione custom con retention `SOURCE` letta poi in reflection a runtime.
- `@Override` assente dove l'intenzione è sovrascrivere: senza, un errore di firma passa silenzioso.

**Diagnosi**
- *`MissingResourceException`* → chiave o bundle assente per quel `Locale`; verificare la catena di
  fallback fino al bundle di default.
- *Formattazione diversa fra ambienti* → dipendenza dal `Locale` di sistema.
- *Annotazione ignorata dal framework* → `@Retention` o `@Target` sbagliati.

Keyword: `Locale` · `ResourceBundle` · `NumberFormat` · `DateTimeFormatter` · `@interface` ·
`@Retention(RUNTIME)` · `@Target` · `@Repeatable`
