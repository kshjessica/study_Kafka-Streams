# study_Kafka-Streams

📚 [Kafka Streams Docs](https://kafka.apache.org/documentation/streams/)  
The below contains an abstract note of the **Kafka Streams API Introduction** videos.

## 📌 [Intro To Streams](https://www.youtube.com/watch?v=Z3JKCLG3VP4)

### What is a Stream?

An unbounded, continuous real-time flow of records.

- No need to explicitly request new records, just receive.
- Records are key-value pairs.

### What is the Kafka Streams API?

Transforms and enriches data.

- Supports per-record stream processing with millisecond latency.
- Supports stateless processing, stateful processing, windowing operations.

Write standard Java applications and microservices to process data in real time.

- No separate processing cluster required.
- Develop on Mac, Linux, Windows.
- Elastic, highly scalable, fault-tolerant.
- Deploy to containers, VMs, bare metal, cloud, on prem.
- Equally viable for small, medium, and large use cases.
- Fully integrated with Kafka security.
- Supports exactly-once semantics as of 0.11.0 release.

The Kafka Streams API is part of the open-source Apache Kafka project.

### Using the Kafka Streams API

Call the Kafka Streams from Java or Scala application.

- The Kafka Streams API interacts with a Kafka cluster.
- The application does not run directly on Kafka brokers.

## 📌 [Creating A Streams Application](https://www.youtube.com/watch?v=LxxeXI1mPKo)

### Configuring the Kafka Streams Application

Configuring the Kafka Streams application with a StreamsConfig instance.

- Is is created from a java.util.Properties instance.

Specify configuration parameters.

- APPLICATION_ID_CONFIG: app name must be unique in the Kafka cluster.
- BOOTSTRAP_SERVERS_CONFIG: where to find Kafka broker(s).

```
import org.apache.kafka.streams.StreamsConfig;

...

final Properties streamsConfig = new Properties();

streamsConfiguration.put(StreamsConfig.APPLICATION_ID_CONFIG, "kafka-music-charts");
streamsConfiguration.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "broker1:9092, broker2:9092");
streamsConfiguration.put(..., ...);
```

- Continue to specify configuration options.

### Defining Serializers and Deserializers

Use Serializers and Deserializers (Serdes) to convert bytes of the record to a specific type.

- SERializer
- DESerializer

Keys Serdes can be Independent from value Serdes.

There are many built-in Serdes.

- e.g. Serdes.String, etc.

Can define custom Serdes.

- e.g. playEventSerdes

  ```
  ...

  final Map<String, String> serdeConfig = Collections.singletonMap(AbstractKafkaAvroSerde.SCHEMA_REGISTRY_URL_CONFIG, SCHEMA_REGISTRY_URL);
  final SpecificAvroSerde<PlayEvent> playEventSerdes = new SpecificAvroSerde<>();

  playEventSerdes.configure(serdeConfig, false);
  ```

### Creating the Streams

Create a KStream object from one or more Kafka topics.

- e.g. play-events

  ```
  final KStreamBuilder  = new KStreamBuilder();
  final KStream<String, PlayEvent> playEvents = builder.stream(Serdes.String(0, playEventSerde, "play-events");
  ```

KSream is used to get stream of records.

KTable is used to get changelog with only the latest value for a given key.

## 📌 Transforming Data

### [Pt. 1](https://www.youtube.com/watch?v=7JYEEx7SBuE)

#### Stateless Transformation Example

Transformation example below has two effects:

- Filters records based on some specific criteria.
- Repartitions based on the new key and value.

```
final KStream<Long, PlayEvent> playsBySongId =
    playEvents.filter((region, event) -> event.getDuration() >= MIN_CHARTABLE_DURATION).map((key, value) -> KeyValue.pair(value.getSongId(), value));
```

#### Join Example

Merging data from different sources can enrich records.

The example below does a primary key table lookup join:

- Effectively, where playsBySongId.key == songTable.key, use the table value song.

```
final KStream<Long, Song> songPlays =
    playsBySongId.leftJoin(songTable, (playEvent, song) -> song, Serdes.Long(), playEventSerdes);
```

#### Stateful processing

Building Stream processing topologies is able with mixed stateless and stateful processing.

### [Pt. 2](https://www.youtube.com/watch?v=3kJgYIkAeHs)

#### Count

Based on an existing key or repartition based on a new key, grouping data may be done.

Example below shows a count method:

- groupBy: re-partitions the data based on a new key.
- count: counts the number of occurrences of a song, required the application to keep state

Stateful operations can use state stores to store and query data.

```
final KGroupedTable<Long, Long> groupedBySongId =
    songPlays.groupBy((songId, song) -> songId, Serdes.Long(), Serdes.Long());

groupedBySongId.count("song-plays-count");
```

- Count results are backed by a state store called "song-play-count".

#### Reduce or Aggregate

Combining current record values with previous record values is possible.

Using lambda expressions may be done.

- In the example below, TopFiveSongs keeps track of the latest top 5 songs and it is automatically updated as new data comes in.
  ```
  final KTable<Song, Long> songPlayCounts =
    songPlaysKGroupedTable.aggregate(TopFiveSongs::new,
      (aggKey, value, aggregate) -> { aggregate.add(value); return aggregate; },
      (aggKey, value, aggregate) -> { aggregate.remove(value); return aggregate; },
      topFiveSerde,
      "top-five-songs-by-genre"
    );
  ```
  - KGroupedTable requires a subtractor, unlike KGroupedStream.
