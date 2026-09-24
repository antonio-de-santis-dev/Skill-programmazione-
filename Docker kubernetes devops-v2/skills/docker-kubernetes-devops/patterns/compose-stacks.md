# Pattern — stack Docker Compose

Stack completi e funzionanti. Adatta immagini, versioni e porte al progetto reale dopo il preflight.

**La regola che vale per tutti**: dentro la rete Compose, il **nome del servizio è l'hostname**.
`localhost` è il container stesso. Ogni indirizzo che punta a un altro container deve usare il nome
del servizio.

Indice:
1. Applicazione + PostgreSQL
2. Microservizi: app + Eureka + Gateway + Zipkin
3. Stack ELK
4. Kafka
5. Comandi utili
6. Diagnosi

---

## 1. Applicazione + PostgreSQL

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: productdb
      POSTGRES_USER: ${DB_USER}            # dal file .env, MAI in chiaro qui
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data     # senza questo i dati spariscono al down
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d productdb"]
      interval: 5s
      retries: 10
    networks: [backend]

  product-service:
    image: registry.example.com/team/product-service:0.0.1
    depends_on:
      postgres:
        condition: service_healthy          # attende che sia PRONTO, non solo avviato
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/productdb
      SPRING_DATASOURCE_USERNAME: ${DB_USER}
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
    ports: ["8080:8080"]
    networks: [backend]

volumes:
  pgdata:
networks:
  backend:
```

Tre punti che fanno la differenza:

**`postgres:5432`, non `localhost:5432`.** Questo è l'errore numero uno.

**`depends_on` con `condition: service_healthy`.** Nella forma semplice, `depends_on` garantisce
solo l'**ordine di avvio**, non che il servizio sia pronto a rispondere. Un'applicazione Spring che
parte prima del database fallisce la connessione e muore.

**Le variabili Spring in maiuscolo con underscore.** Spring Boot mappa
`SPRING_DATASOURCE_URL` su `spring.datasource.url` automaticamente: è il modo di iniettare
configurazione senza toccare l'immagine.

File `.env` accanto al Compose, **escluso da Git**:
```
DB_USER=product
DB_PASSWORD=cambiami
```

---

## 2. Microservizi: app + Eureka + Gateway + Zipkin

```yaml
services:
  naming-server:
    image: registry.example.com/team/eureka-server:0.0.1
    ports: ["8761:8761"]
    networks: [msnet]

  zipkin:
    image: openzipkin/zipkin:latest
    ports: ["9411:9411"]
    networks: [msnet]

  product-service:
    image: registry.example.com/team/product-service:0.0.2
    depends_on: [naming-server, zipkin]
    environment:
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://naming-server:8761/eureka
      MANAGEMENT_ZIPKIN_TRACING_ENDPOINT: http://zipkin:9411/api/v2/spans
    ports: ["8081:8080"]
    networks: [msnet]

  api-gateway:
    image: registry.example.com/team/api-gateway:0.0.2
    depends_on: [naming-server]
    environment:
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://naming-server:8761/eureka
    ports: ["8765:8765"]
    networks: [msnet]

networks:
  msnet:
```

Nella configurazione dell'applicazione, l'indirizzo di Eureka va scritto con un default
sovrascrivibile, così **lo stesso artefatto** funziona in locale e in Compose:

```properties
eureka.client.serviceUrl.defaultZone=${EUREKA_URI:http://localhost:8761/eureka}
```

**Le porte pubblicate sono due cose diverse.** `8081:8080` significa: dall'host si arriva sulla
8081, dentro la rete il servizio resta sulla 8080. Gli altri container usano
`http://product-service:8080`, non la 8081.

Verifica: `http://localhost:8761` mostra i servizi registrati, `http://localhost:9411` le tracce.

Nota: qui Eureka e Gateway hanno senso perché siamo su Compose. **Su Kubernetes non servono**:
`Service` + DNS e `Ingress` fanno lo stesso lavoro. Vedi `microservices-architecture`.

---

## 3. Stack ELK

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.2
    environment:
      discovery.type: single-node
      ES_JAVA_OPTS: "-Xms512m -Xmx512m"
      ELASTIC_PASSWORD: ${ELASTIC_PASSWORD}
    ports: ["9200:9200", "9300:9300"]     # 9200 = API REST, 9300 = comunicazione fra nodi
    volumes: [esdata:/usr/share/elasticsearch/data]
    networks: [elk]

  logstash:
    image: docker.elastic.co/logstash/logstash:7.15.2
    depends_on: [elasticsearch]
    ports: ["4560:4560"]                  # la porta su cui ascolta l'appender Logback
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
    networks: [elk]

  kibana:
    image: docker.elastic.co/kibana/kibana:7.15.2
    depends_on: [elasticsearch]
    ports: ["5601:5601"]
    networks: [elk]

volumes:
  esdata:
networks:
  elk:
```

`logstash.conf`, accanto al Compose:
```
input {
  tcp {
    port  => 4560
    codec => json_lines        # deve corrispondere al LogstashEncoder lato Logback
  }
}
output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]   # il NOME DEL SERVIZIO, non localhost
    index => "%{[appName]}-%{+YYYY.MM.dd}"
    user  => "elastic"
    password => "${ELASTIC_PASSWORD}"
  }
}
```

Lato Spring, `logback-spring.xml`:
```xml
<configuration>
  <springProperty scope="context" name="appName" source="spring.application.name"/>

  <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>${LOGSTASH_HOST:-localhost}:4560</destination>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
  </appender>

  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder><pattern>%d{HH:mm:ss} %-5level %logger{20} - %msg%n</pattern></encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="LOGSTASH"/>
    <appender-ref ref="CONSOLE"/>
  </root>
</configuration>
```

Dipendenza: `net.logstash.logback:logstash-logback-encoder`.

Tieni **entrambi** gli appender: la console serve quando Logstash è giù, ed è la prima cosa che
guardi per capire perché è giù.

In Kibana va creato l'**index pattern** corrispondente al nome dell'indice, altrimenti i documenti
ci sono ma non si vedono. È la causa più frequente di "Kibana è vuoto".

---

## 4. Kafka

Configurazione in `microservices-architecture/patterns/messaging-kafka.md`, con il dettaglio su
`KAFKA_ADVERTISED_LISTENERS`, che è il punto che rompe la maggior parte delle configurazioni.

Aggiunta utile in sviluppo, una UI per ispezionare topic e messaggi:
```yaml
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on: [kafka]
    ports: ["8090:8080"]
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

---

## 5. Comandi utili

```bash
docker compose up -d                   # avvia in background
docker compose up --build              # ricostruisce le immagini con build locale
docker compose ps                      # stato dei servizi
docker compose logs -f product-service # log di un servizio, in tempo reale
docker compose exec product-service sh # shell dentro il container
docker compose down                    # ferma e rimuove i container
docker compose down -v                 # ...e anche i volumi: CANCELLA I DATI
```

`docker compose down -v` in un ambiente con dati che servono è un errore irreversibile. Vale la
pena saperlo prima.

---

## 6. Diagnosi

| Sintomo | Causa |
|---|---|
| "Connection refused" fra due container | `localhost` invece del nome del servizio |
| L'app parte prima del database e muore | `depends_on` senza `condition: service_healthy` |
| "Immagine non trovata" | la versione nel Compose non è mai stata costruita o pubblicata |
| Dati persi dopo `down` | manca il volume, oppure è stato usato `-v` |
| Porta già in uso sull'host | cambia la porta **esterna**: `8082:8080` |
| I container si vedono a intermittenza | reti diverse per servizi che devono parlarsi |
| Un servizio non compare in Eureka | variabile `EUREKA_CLIENT_SERVICEURL_DEFAULTZONE` errata |
| Kibana vuoto | index pattern non creato, o Logstash non riceve (verifica la porta 4560) |
| Modifica alla configurazione senza effetto | le variabili si leggono all'avvio: serve `up -d --force-recreate` |
