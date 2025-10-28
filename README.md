# Event-Driven Architecture avec Kafka et Spring Cloud Streams

## 📋 Table des matières
- [Introduction](#introduction)
- [Concepts fondamentaux](#concepts-fondamentaux)
- [Architecture du projet](#architecture-du-projet)
- [Structure du code](#structure-du-code)
- [Configuration](#configuration)
- [Démarrage du projet](#démarrage-du-projet)
- [Tests et visualisation](#tests-et-visualisation)
- [Concepts avancés illustrés](#concepts-avancés-illustrés)
- [Points clés à retenir](#points-clés-à-retenir)

---

## Introduction

Ce projet démontre l'implémentation d'une architecture événementielle (Event-Driven Architecture) utilisant **Apache Kafka** et **Spring Cloud Streams**. Il illustre la production, la consommation et le traitement en temps réel d'événements de pages web avec visualisation analytique.

### 🎯 Objectifs du projet
- Produire des événements vers des topics Kafka
- Consommer et traiter des événements en temps réel
- Utiliser Kafka Streams pour l'agrégation et le windowing
- Visualiser les données analytiques en temps réel
  ![Analytics en temps réel](https://raw.githubusercontent.com/FatihaELHABTI/activite1_event_driven_architecture_kafka/main/images/kafka5.png)


---

## Concepts fondamentaux

### 🔹 Event-Driven Architecture (EDA)
Une architecture pilotée par les événements où les composants communiquent en émettant et en consommant des événements de manière asynchrone. Les avantages incluent:
- **Découplage**: Les producteurs et consommateurs sont indépendants
- **Scalabilité**: Chaque composant peut être mis à l'échelle indépendamment
- **Résilience**: Les événements peuvent être rejouées en cas d'échec

### 🔹 Apache Kafka
Plateforme distribuée de streaming d'événements qui permet:
- **Topics**: Canaux de communication nommés pour les événements
- **Producers**: Publient des messages vers les topics
- **Consumers**: S'abonnent aux topics pour recevoir les messages
- **Partitions**: Division des topics pour la parallélisation

### 🔹 Spring Cloud Streams
Framework qui simplifie le développement d'applications événementielles en fournissant:
- **Binder**: Abstraction pour se connecter à différents systèmes de messaging
- **Functional Programming**: Utilisation de `Supplier`, `Consumer`, `Function`
- **Kafka Streams**: Support natif pour le traitement de flux

### 🔹 Kafka Streams
Bibliothèque de traitement de flux qui permet:
- **Windowing**: Regroupement temporel des événements (tumbling, sliding, session)
- **Aggregations**: Count, sum, reduce sur des fenêtres de temps
- **State Stores**: Stockage matérialisé pour les résultats intermédiaires
- **Interactive Queries**: Interrogation des state stores en temps réel

---

## Architecture du projet

```
┌─────────────────────────────────────────────────────────────┐
│                    Kafka Cluster                            │
│  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐            │
│  │ T1   │    │ T2   │    │ T3   │    │ T4   │            │
│  └──┬───┘    └──────┘    └──┬───┘    └──┬───┘            │
└─────┼─────────────────────────┼──────────┼────────────────┘
      │                         │          │
      │                         │          │
┌─────▼─────────────────────────▼──────────▼────────────────┐
│         Application Spring Cloud Streams                   │
│                                                             │
│  ┌─────────────────┐  ┌──────────────────┐               │
│  │ RestController  │  │  PageEventHandler │               │
│  │  - publish()    │  │                   │               │
│  │  - analytics()  │  │  3 Beans:         │               │
│  └────────┬────────┘  │  • Consumer       │               │
│           │           │  • Supplier       │               │
│           │           │  • KStream Func   │               │
│           │           └──────────────────┘               │
│           │                                               │
│  ┌────────▼────────────────────────────┐                 │
│  │  InteractiveQueryService            │                 │
│  │  (count-store)                      │                 │
│  └─────────────────────────────────────┘                 │
└──────────────────────────┬──────────────────────────────┘
                           │
                    ┌──────▼────────┐
                    │  Frontend     │
                    │  (Smoothie.js)│
                    └───────────────┘
```

### Flux de données:
1. **T1**: Topic de consommation simple (pageEventConsumer)
2. **T3**: Topic alimenté par le Supplier automatique
3. **T3 → KStream → T4**: Traitement avec filtrage, windowing et agrégation
4. **count-store**: State store matérialisé pour les requêtes interactives
5. **REST /analytics**: Exposition SSE des données agrégées

---

## Structure du code

### 1️⃣ **PageEvent.java** - Modèle d'événement

```java
package ma.enset.kafkaspringcloudstreams.events;

import java.util.Date;

public record PageEvent(String name, String user, Date date, long duration) {
}
```

- **Record Java**: Structure immutable représentant un événement de visite de page
- **Attributs**:
  - `name`: Nom de la page (P1, P2)
  - `user`: Identifiant utilisateur (U1, U2)
  - `date`: Timestamp de l'événement
  - `duration`: Durée de visite en millisecondes

### 2️⃣ **PageEventHandler.java** - Gestion fonctionnelle des événements

```java
package ma.enset.kafkaspringcloudstreams.handlers;

import ma.enset.kafkaspringcloudstreams.events.PageEvent;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KeyValue;
import org.apache.kafka.streams.kstream.Grouped;
import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.Materialized;
import org.apache.kafka.streams.kstream.TimeWindows;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.util.Date;
import java.util.Random;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Supplier;

@Component
public class PageEventHandler {

    @Bean
    public Consumer<PageEvent> pageEventConsumer(){
        return (input)->{
            System.out.println("***************");
            System.out.println(input.toString());
            System.out.println("***************");
        };
    }

    @Bean
    public Supplier<PageEvent> pageEventSupplier(){
        return ()->{
            return new PageEvent(
                    Math.random()>0.5?"P1":"P2",
                    Math.random()>0.5?"U1":"U2",
                    new Date(),
                    10 +new Random().nextInt(10000)
            );
        };
    }

    @Bean
    public Function<KStream<String, PageEvent>, KStream<String, Long>> KStreamFunction(){
        return (input)->
                input.filter((k,v)->v.duration()>100)
                        .map((k,v)->new KeyValue<>(v.name(),v.duration()))
                        .groupByKey(Grouped.with(Serdes.String(),Serdes.Long()))
                        .windowedBy(TimeWindows.of(Duration.ofSeconds(5)))
                        .count(Materialized.as("count-store"))
                        .toStream()
                        .map((k,v)->new KeyValue<>(k.key(),v));
    }
}
```

#### a) Consumer - Consommation simple
- **Fonction**: Consomme les événements du topic T1
- **Traitement**: Affichage dans la console
- **Binding**: `pageEventConsumer-in-0` → Topic T1

#### b) Supplier - Production automatique
- **Fonction**: Génère des événements toutes les 200ms
- **Données**: PageEvents aléatoires (P1/P2, U1/U2)
- **Binding**: `pageEventSupplier-out-0` → Topic T3

#### c) KStream Function - Traitement avec Kafka Streams

**Pipeline de traitement**:
1. **Filter**: `duration > 100` - Ne garde que les visites significatives
2. **Map**: Transformation en `KeyValue<pageName, duration>`
3. **GroupByKey**: Regroupement par nom de page
4. **WindowedBy**: Fenêtres de 5 secondes (tumbling window)
5. **Count**: Comptage des événements dans chaque fenêtre
6. **Materialized**: Stockage dans "count-store"
7. **ToStream**: Conversion en KStream de sortie

**Concept de Windowing**:
```
Time: 0s─────5s─────10s─────15s
      [Window1][Window2][Window3]
      Events   Events   Events
      → Count  → Count  → Count
```

### 3️⃣ **PageEventController.java** - API REST

```java
package ma.enset.kafkaspringcloudstreams.controllers;

import ma.enset.kafkaspringcloudstreams.events.PageEvent;
import org.apache.kafka.streams.KeyValue;
import org.apache.kafka.streams.kstream.Windowed;
import org.apache.kafka.streams.state.KeyValueIterator;
import org.apache.kafka.streams.state.QueryableStoreTypes;
import org.apache.kafka.streams.state.ReadOnlyWindowStore;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.binder.kafka.streams.InteractiveQueryService;
import org.springframework.cloud.stream.function.StreamBridge;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

import java.time.Duration;
import java.time.Instant;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.Random;

@RestController
public class PageEventController {

    @Autowired
    private StreamBridge streamBridge;

    @Autowired
    private InteractiveQueryService interactiveQueryService;

    @GetMapping("/publish")
    public PageEvent publish(String name ,String topic ){
        PageEvent event = new PageEvent(
                name,
                Math.random()>0.5?"U1":"U2",
                new Date(),
                10+new Random().nextInt(100)
        );
        streamBridge.send(topic, event);
        return event;
    }

    @GetMapping(path = "/analytics",produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Map<String, Long>> analytics(){
        return Flux.interval(Duration.ofSeconds(1))
                .map(sequence->{
                    Map<String,Long> stringLongMap=new HashMap<>();
                    ReadOnlyWindowStore<String, Long> windowStore = interactiveQueryService.getQueryableStore("count-store", QueryableStoreTypes.windowStore());
                    Instant now=Instant.now();
                    Instant from=now.minusMillis(5000);
                    KeyValueIterator<Windowed<String>, Long> fetchAll = windowStore.fetchAll(from, now);
                    while (fetchAll.hasNext()){
                        KeyValue<Windowed<String>, Long> next = fetchAll.next();
                        stringLongMap.put(next.key.key(),next.value);
                    }
                    return stringLongMap;
                });
    }
}
```

#### a) Endpoint /publish
- **Fonction**: Publication manuelle d'événements
- **StreamBridge**: Permet d'envoyer dynamiquement vers n'importe quel topic
- **Usage**: `GET /publish?name=P1&topic=T1`

#### b) Endpoint /analytics (Server-Sent Events)

**Mécanisme**:
1. **Flux.interval**: Émet un signal chaque seconde
2. **InteractiveQueryService**: Récupère le state store "count-store"
3. **ReadOnlyWindowStore**: Lecture des fenêtres temporelles
4. **fetchAll(from, now)**: Récupère les 5 dernières secondes
5. **SSE**: Envoie les données au client en temps réel

**Structure des données retournées**:
```json
{
  "P1": 12,
  "P2": 8
}
```

### 4️⃣ **analytics.html** - Visualisation temps réel

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Analytics</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/smoothie/1.34.0/smoothie.min.js"></script>
</head>
<body>

<canvas id="chart2" width="600" height="400"></canvas>
<script>
    var index=-1;
    randomColor = function() {
        ++index;
        if (index >= colors.length) index = 0; return colors[index];
    }
    var pages=["P1","P2"];
    var colors=[
        { sroke : 'rgba(0, 255, 0, 1)', fill : 'rgba(0, 255, 0, 0.2)' },
        { sroke : 'rgba(255, 0, 0, 1)', fill : 'rgba(255, 0, 0, 0.2)'}
    ];
    var courbe = [];
    var smoothieChart = new SmoothieChart({tooltip: true});
    smoothieChart.streamTo(document.getElementById("chart2"), 500);
    pages.forEach(function(v){
        courbe[v]=new TimeSeries();
        col = randomColor();
        smoothieChart.addTimeSeries(courbe[v], {strokeStyle : col.sroke, fillStyle : col.fill, lineWidth : 2
        });
    });
    var stockEventSource= new EventSource("/analytics");
    stockEventSource.addEventListener("message", function (event) {
        pages.forEach(function(v){
            val=JSON.parse(event.data)[v];
            courbe[v].append(new Date().getTime(),val);
        });
    });

</script>
</body>
</html>
```

**Technologies**:
- **SmoothieChart**: Bibliothèque de graphiques en temps réel
- **EventSource**: API JavaScript pour SSE
- **TimeSeries**: Séries temporelles pour chaque page

**Fonctionnement**:
1. Création de courbes pour P1 et P2
2. Connexion SSE à `/analytics`
3. Mise à jour du graphique à chaque événement reçu
4. Animation fluide avec interpolation

---

## Configuration

### application.properties

```properties
spring.application.name=kafka-spring-cloud-streams
server.port=8080

# Bindings des fonctions
# Consumer simple
spring.cloud.stream.bindings.pageEventConsumer-in-0.destination=T1

# Supplier automatique
spring.cloud.stream.bindings.pageEventSupplier-out-0.destination=T3
spring.cloud.stream.bindings.pageEventSupplier-out-0.producer.poller.fixed-delay=200

# KStream Function
spring.cloud.stream.bindings.KStreamFunction-in-0.destination=T3
spring.cloud.stream.bindings.KStreamFunction-out-0.destination=T4

# Déclaration des fonctions actives
spring.cloud.function.definition=pageEventConsumer;pageEventSupplier;KStreamFunction

# Configuration Kafka Streams
spring.cloud.stream.kafka.streams.binder.configuration.commit.interval.ms=1000
```

**Explication des bindings**:
- Pattern: `<functionName>-<in|out>-<index>`
- `in-0`: Première entrée de la fonction
- `out-0`: Première sortie de la fonction

---

## Démarrage du projet

### Prérequis
- Java 17+
- Docker et Docker Compose
- Maven

### 1️⃣ Démarrage de l'infrastructure Kafka

Créer un fichier `docker-compose.yml`:

```yaml
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.3.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  broker:
    image: confluentinc/cp-kafka:7.3.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

Lancer les conteneurs:
```bash
docker-compose up -d
```
![Analytics en temps réel](https://raw.githubusercontent.com/FatihaELHABTI/activite1_event_driven_architecture_kafka/main/images/kafka1.png)


### 2️⃣ Vérification des conteneurs
```bash
docker ps
```
![Analytics en temps réel](https://raw.githubusercontent.com/FatihaELHABTI/activite1_event_driven_architecture_kafka/main/images/Kafka2.png)

Vous devriez voir:
- `broker` (Kafka) sur le port 9092
- `zookeeper` sur le port 2181

### 3️⃣ Démarrage de l'application Spring Boot
```bash
mvn spring-boot:run
```

---

## Tests et visualisation

### Test 1: Console Producer/Consumer

**Producer**:
```bash
docker exec -it broker kafka-console-producer \
  --bootstrap-server broker:9092 \
  --topic R2
```
Taper des messages: `Hello`, `A`, `B`, `C`, `Red`

![Analytics en temps réel](https://raw.githubusercontent.com/FatihaELHABTI/activite1_event_driven_architecture_kafka/main/images/kafka4.png)


**Consumer**:
```bash
docker exec -it broker kafka-console-consumer \
  --bootstrap-server broker:9092 \
  --topic R2 \
  --from-beginning
```
![Analytics en temps réel](https://raw.githubusercontent.com/FatihaELHABTI/activite1_event_driven_architecture_kafka/main/images/kafka3.png)


Les messages apparaissent dans l'ordre de réception.

### Test 2: Publication manuelle via REST
```bash
curl "http://localhost:8080/publish?name=P1&topic=T1"
curl "http://localhost:8080/publish?name=P2&topic=T3"
```

### Test 3: Visualisation analytique

1. Ouvrir le navigateur: `http://localhost:8080/index.html`
2. Observer les courbes en temps réel
   - **Courbe verte**: Comptage des événements P1
   - **Courbe rouge**: Comptage des événements P2
3. Les données sont mises à jour chaque seconde via SSE

![Analytics en temps réel](https://raw.githubusercontent.com/FatihaELHABTI/activite1_event_driven_architecture_kafka/main/images/kafka6.png)


**Interprétation du graphique**:
- L'axe Y représente le nombre d'événements dans la fenêtre de 5 secondes
- L'axe X représente le temps
- Les pics indiquent une activité intense sur une page

---

## Concepts avancés illustrés

### 1. Windowing Tumbling
```
Window 1: [0-5s]  → Count: 10 events
Window 2: [5-10s] → Count: 15 events
Window 3: [10-15s]→ Count: 8 events
```
Chaque fenêtre est indépendante, sans chevauchement.

### 2. State Store matérialisé
Le `count-store` permet:
- Stockage local des agrégations
- Requêtes interactives en temps réel
- Reconstruction automatique en cas de redémarrage

### 3. Stream Processing Topology
```
Source (T3) → Filter → Map → GroupBy → Window → Count → Sink (T4)
                                              ↓
                                         count-store
```

### 4. Server-Sent Events (SSE)
- Connexion unidirectionnelle serveur → client
- Léger et natif dans les navigateurs
- Idéal pour les push de données en temps réel

---

## 🎓 Points clés à retenir

1. **Spring Cloud Streams** simplifie Kafka avec une approche fonctionnelle
2. **Kafka Streams** permet le traitement complexe sans infrastructure externe
3. **Windowing** est essentiel pour l'analyse temporelle
4. **Interactive Queries** transforment Kafka en base de données temps réel
5. **SSE** est parfait pour les dashboards en temps réel

---

**Technologies**: Spring Boot, Kafka, Kafka Streams, Spring Cloud Stream, SSE
