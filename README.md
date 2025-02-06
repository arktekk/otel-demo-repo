# otel-demo-repo

Examples for the arktekk blog series on otel

## Kom i gang med Open Telemetry

I denne delen av serien vil vi ta for oss hvordan man raskt kommer igang med Open Telemetry for en applikasjon på JVM’en
Vi vil ikke gjøre ting på den perfekte måten. Målet er å sørge for at applikasjonen er instrumentert og at vi har
bekreftet at den sender data.
Vi vil heller ikke forklare alle begrepene.

Noe av det som er vanskelig med Open Telemetry er å finne ut hvor man skal begynne. Skal man begynne med å sette opp
grafana? Eller bør man lese seg opp på hva Open Telemetry er? Skal man velge instrumentering eller manuell
innfallsvinkel? Det hjelper heller ikke at dokumentasjonen oppleves som en labyrint, der man rett som det er befinner
seg på et sted man har vært før.

Vi starter med å sette opp en Otel-collector. Dette er kort fortalt en proxy med muligheter til å prossessere input og
eksportere det videre. For å gjøre feedback-loopen så kort som overhodet mulig setter vi denne opp til å skrive ut alt
den får inn til stdout. På denne måten er det lett å se når vi har klart å få applikasjonen til å sende otel-data.

### Otel Collector

#### otel-collector-debug-config.yaml

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  debug:
    verbosity: detailed
  nop:

service:
  pipelines:
    traces:
      receivers: [ otlp ]
      exporters: [ nop ]
    metrics:
      receivers: [ otlp ]
      exporters: [ nop ]
    logs:
      receivers: [ otlp ]
      exporters: [ debug ]
```

Essensen av denne konfigurasjonen er at otel-collectoren tar imot `traces`, `metrics` og
`logs`. `traces` og `metrics` blir sendt til `nop` (altså kastet) og `logs` blir sendt til
`debug` (altså skrevet til stdout).

På denne måten kan vi sette opp en applikasjon til å sende traces, metrics og logs til otel-collectoren,
og bestemme at vi f.eks. kun ønsker å skrive traces til stdout. Vi vil enkelt se at vi har klart å sette applikasjonen
til å sende de ulike tingene, og vi vil kunne se hvordan de ulike tingene ser ut.

#### docker-compose.yml

Vi starter otel-collectoren med følgende `docker-compose.yml`

```yaml
services:
  otel-collector:
    image: otel/opentelemetry-collector:latest
    container_name: otel-collector
    command: [ "--config=/etc/otel-collector-config.yaml" ]
    volumes:
      - ./otel-collector-debug-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
```

`docker compose up` gir oss følgende output:

```terminaloutput
[+] Running 1/1
 ✔ Container otel-collector  Created                                                                                                                                                                                                            0.0s
Attaching to otel-collector
otel-collector  | 2025-02-09T11:56:41.697Z	info	service@v0.119.0/service.go:186	Setting up own telemetry...
otel-collector  | 2025-02-09T11:56:41.697Z	info	builders/builders.go:26	Development component. May change in the future.	{"kind": "exporter", "data_type": "logs", "name": "debug"}
otel-collector  | 2025-02-09T11:56:41.697Z	info	service@v0.119.0/service.go:252	Starting otelcol...	{"Version": "0.119.0", "NumCPU": 12}
otel-collector  | 2025-02-09T11:56:41.697Z	info	extensions/extensions.go:39	Starting extensions...
otel-collector  | 2025-02-09T11:56:41.698Z	info	otlpreceiver@v0.119.0/otlp.go:112	Starting GRPC server	{"kind": "receiver", "name": "otlp", "data_type": "metrics", "endpoint": "0.0.0.0:4317"}
otel-collector  | 2025-02-09T11:56:41.698Z	info	otlpreceiver@v0.119.0/otlp.go:169	Starting HTTP server	{"kind": "receiver", "name": "otlp", "data_type": "metrics", "endpoint": "0.0.0.0:4318"}
otel-collector  | 2025-02-09T11:56:41.698Z	info	service@v0.119.0/service.go:275	Everything is ready. Begin running and processing data.
```

Vi har nå en kjørende Otel-collector og går videre til å sette opp en liten scala-applikasjon til å sende data til denne
collectoren.

### Applikasjon

Vi ønsker å instrumentere applikasjonen vår slik at den sender data til otel-collectoren. For dette finnes det en
javaagent som kan lastes ned på:
https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v2.12.0/opentelemetry-javaagent.jar

Ved å starte applikasjonen med denne javaagenten vil bytekoden bli instrumentert før applikasjonen starter opp.

```shell 
java -javaagent:opentelemetry-javaagent.jar -jar server.jar
```

Følgende linje vil bli skrevet ut som viser at javaagenten er i bruk:

```terminaloutput
[otel.javaagent 2025-02-09 13:05:17:117 +0100] [main] INFO io.opentelemetry.javaagent.tooling.VersionLogger - opentelemetry-javaagent - version: 2.12.0
```

Vi kan også skru på debug-logging slik at vi ser hva som blir instrumentert:

```shell
java \
    -Dotel.javaagent.debug=true \
    -javaagent:opentelemetry-javaagent.jar \
    -jar server.jar \
    &> server-otel-startup.log
```

Denne output'en blir litt for stor til å ta med her, så vi gjør et lite uttrekk for å se om den inneholder det vi
ønsker:

```shell
grep Transformed server-otel-startup.log|grep -v null|cut -d'-' -f4
```

```terminaloutput
 Transformed okhttp3.internal.concurrent.TaskRunner$runnable$1
 Transformed io.opentelemetry.sdk.metrics.export.PeriodicMetricReader$Scheduled
 Transformed io.opentelemetry.sdk.trace.export.BatchSpanProcessor$Worker
 Transformed io.opentelemetry.sdk.logs.export.BatchLogRecordProcessor$Worker
 Transformed coursier.bootstrap.launcher.ac$$Lambda
 Transformed coursier.bootstrap.launcher.SharedClassLoader
 Transformed io.undertow.server.HttpHandler
 Transformed io.undertow.server.handlers.BlockingHandler
 Transformed scala.concurrent.impl.ExecutionContextImpl
 Transformed ch.qos.logback.classic.Logger
 Transformed ch.qos.logback.classic.spi.LoggingEvent
 Transformed cask.main.Main$DefaultHandler
 Transformed org.xnio.XnioIoThread
 Transformed org.xnio.nio.WorkerThread
 Transformed org.xnio.XnioWorker$1
 Transformed org.jboss.threads.EnhancedQueueExecutor
 Transformed org.jboss.threads.EnhancedQueueExecutor$AbstractScheduledFuture
 Transformed org.jboss.threads.EnhancedQueueExecutor$RepeatingScheduledFuture
 Transformed org.jboss.threads.EnhancedQueueExecutor$FixedDelayRunnableScheduledFuture
 Transformed org.jboss.threads.EnhancedQueueExecutor$RunnableScheduledFuture
 Transformed org.jboss.threads.EnhancedQueueExecutor$CallableScheduledFuture
 Transformed org.jboss.threads.EnhancedQueueExecutor$FixedRateRunnableScheduledFuture
 Transformed org.jboss.threads.JBossExecutors$4
 Transformed org.jboss.threads.JBossExecutors$3
 Transformed org.jboss.threads.NullRunnable
 Transformed org.jboss.threads.EnhancedQueueExecutor$SchedulerTask
 Transformed org.xnio.XnioWorker$WorkerThreadFactory$1$1
 Transformed org.xnio.nio.WorkerThread$SynchTask
 Transformed org.xnio.nio.NioTcpServerHandle$1
 Transformed org.xnio.nio.QueuedNioTcpServer2$$Lambda
 Transformed org.xnio.nio.NioTcpServerHandle$2
```

Vi ser oss fornøyd med at denne listen inneholder både Logback og Cask (webserver).

Når vi starter applikasjonen vil vi se at logg-linjene blir sendt til otel-collectoren.
Her er et eksempel på en slik linje i otel-collectoren:

```terminaloutput
otel-collector  | LogRecord #0
otel-collector  | ObservedTimestamp: 2025-02-09 12:25:50.134533 +0000 UTC
otel-collector  | Timestamp: 2025-02-09 12:25:50.130629 +0000 UTC
otel-collector  | SeverityText: INFO
otel-collector  | SeverityNumber: Info(9)
otel-collector  | Body: Str(Server starting on port 8080)
otel-collector  | Trace ID:
otel-collector  | Span ID:
otel-collector  | Flags: 0
```

Hvis vi endrer traces i otel-configen til å være debug og logs til å være nop, og kjører:

```shell
curl http://localhost:8080/fib/1
```

Så får vi blant annet dette i otel-loggen:

```terminaloutput
otel-collector  | Span #0
otel-collector  |     Trace ID       : b5413f08a72d883cca2ec1b9192465e1
otel-collector  |     Parent ID      :
otel-collector  |     ID             : 46f999d2f432bb74
otel-collector  |     Name           : GET
otel-collector  |     Kind           : Server
otel-collector  |     Start time     : 2025-02-09 12:32:27.869428 +0000 UTC
otel-collector  |     End time       : 2025-02-09 12:32:27.870366875 +0000 UTC
otel-collector  |     Status code    : Unset
otel-collector  |     Status message :
otel-collector  | Attributes:
otel-collector  |      -> thread.id: Int(46)
otel-collector  |      -> http.request.method: Str(GET)
otel-collector  |      -> http.response.status_code: Int(200)
otel-collector  |      -> url.path: Str(/fib/1)
otel-collector  |      -> server.address: Str(localhost)
otel-collector  |      -> client.address: Str(127.0.0.1)
otel-collector  |      -> server.port: Int(8080)
otel-collector  |      -> network.peer.address: Str(127.0.0.1)
otel-collector  |      -> url.scheme: Str(http)
otel-collector  |      -> thread.name: Str(XNIO-1 I/O-8)
otel-collector  |      -> network.protocol.version: Str(1.1)
otel-collector  |      -> user_agent.original: Str(curl/8.7.1)
otel-collector  |      -> network.peer.port: Int(54881)
otel-collector  | 	{"kind": "exporter", "data_type": "traces", "name": "debug"}
```

Da gjenstår det bare å teste metrics

Metrics blir ikke sendt like ofte default, så her kan det være greit å konfigurere litt for å slippe å vente så lenge:

```
java \
    -javaagent:opentelemetry-javaagent.jar \
    -Dotel.metric.export.interval=2000 \
    -jar server.jar
```

Det vil nå bli sendt metrics hvert 2000ms istedenfor hvert 60s som er default.

Noe som ligner dette vil dukke opp i otel-collectoren:

```
otel-collector  | ScopeMetrics #0
otel-collector  | ScopeMetrics SchemaURL:
otel-collector  | InstrumentationScope io.opentelemetry.sdk.trace
otel-collector  | Metric #0
otel-collector  | Descriptor:
otel-collector  |      -> Name: queueSize
otel-collector  |      -> Description: The number of items queued
otel-collector  |      -> Unit: 1
otel-collector  |      -> DataType: Gauge
otel-collector  | NumberDataPoints #0
otel-collector  | Data point attributes:
otel-collector  |      -> processorType: Str(BatchSpanProcessor)
otel-collector  | StartTimestamp: 2025-02-09 12:40:35.411811 +0000 UTC
otel-collector  | Timestamp: 2025-02-09 12:40:37.421577 +0000 UTC
otel-collector  | Value: 0
```

Vi har nå en applikasjon som sender logs, traces og metrics til otel-collectoren!

Lenke til github-repo
Lenke til dokumentasjon av otel-metrics
Lenke til biblioteker som er støttet av javaagenten
Lenke alt det er mulig å konfigurere vha env vars


Til senere:
- Javaagent
  - Hvordan få den inn i bygget?
- Kamelåse
  - https://www.youtube.com/watch?v=ykj3Kpm3O0g

spm:
- forenkle eksempelet
  - trenger vi database i første omgang?

