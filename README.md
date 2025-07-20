# Book Notes: Apache Kafka in Action
Book: https://www.manning.com/books/apache-kafka-in-action

## Configure local WSL environment
- Setup guide: https://learn.microsoft.com/en-us/windows/wsl/setup/environment
- VS Code: https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-vscode

Update OS after installation
```shell
sudo apt update && sudo apt upgrade
sudo apt install wget ca-certificates
```

Install Java
```shell
sudo apt install openjdk-21-jre-headless
java --version
```

To open a WSL folder in windows explorer
```shell
explorer.exe .
```

To open the current folder in VS Code
```shell
code .
```

## Installing Kafka
https://kafka.apache.org/quickstart

Download kafka and add it to the system path.
```shell
wget https://dlcdn.apache.org/kafka/4.0.0/kafka_2.13-4.0.0.tgz
tar xfz kafka_2.13-4.0.0.tgz
rm kafka_2.13-4.0.0.tgz
mv kafka_2.13-4.0.0/ ~/kafka
cd kafka/
export PATH=~/kafka/bin/:"$PATH"
```

### Configure brokers
The following initializes three broker instances.

Add configuration files for each broker.

> The book instructions placed the logs in ~/kafka/data folder, using /tmp/ instead to avoid permissions issues.

Broker 1: ~/kafka/config/kafka1.properties
```shell
broker.id=1
log.dirs=/tmp/kafka1-logs
listeners=PLAINTEXT://:9092,CONTROLLER://:9192
process.roles=broker,controller
controller.quorum.voters=1@localhost:9192,2@localhost:9193,\
3@localhost:9194
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:PLAINTEXT,\
PLAINTEXT:PLAINTEXT
```

Broker 2: ~/kafka/config/kafka2.properties
```shell
broker.id=2
log.dirs=/tmp/kafka2-logs
listeners=PLAINTEXT://:9093,CONTROLLER://:9193
process.roles=broker,controller
controller.quorum.voters=1@localhost:9192,2@localhost:9193,3@localhost:9194
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
```

Broker 3: ~/kafka/config/kafka3.properties
```shell
broker.id=3
log.dirs=/tmp/kafka3-logs
listeners=PLAINTEXT://:9094,CONTROLLER://:9194
process.roles=broker,controller
controller.quorum.voters=1@localhost:9192,2@localhost:9193,3@localhost:9194
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
```

Create data directories.
```shell
mkdir -p /tmp/kafka1-logs /tmp/kafka2-logs /tmp/kafka3-logs
```

Set cluster UUID.
```shell
export KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
```

Format data folders
```shell
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c ~/kafka/config/kafka1.properties
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c ~/kafka/config/kafka2.properties
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c ~/kafka/config/kafka3.properties
```

Starting the first broker. It should start but the output will contain messages about the other nodes being unavailable.
 ```shell
 ~/kafka/bin/kafka-server-start.sh ~/kafka/config/kafka1.properties
 ```

Starting the other nodes in their own terminal windows will resolve those messages.

Broker 2
```shell
~/kafka/bin/kafka-server-start.sh ~/kafka/config/kafka2.properties
```

Broker 3
```shell
~/kafka/bin/kafka-server-start.sh ~/kafka/config/kafka3.properties
```

Use `Ctrl-C` to terminate each process. We can restart them as daemon background processes so they don't tie up the terminal.
```shell
~/kafka/bin/kafka-server-start.sh -daemon ~/kafka/config/kafka1.properties
```
```shell
~/kafka/bin/kafka-server-start.sh -daemon ~/kafka/config/kafka2.properties
```
```shell
~/kafka/bin/kafka-server-start.sh -daemon ~/kafka/config/kafka3.properties
```

Check the status.
```shell
~/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092,localhost:9093,localhost:9094
```

To stop all running brokers.
```shell
~/kafka/bin/kafka-server-stop.sh
```

To stop just a single broker, we can create our script ~/kafka/bin/kafka-broker-stop.sh.
```shell
#!/bin/bash
BROKER_ID="$1"

if [ -z "$BROKER_ID" ]; then
  echo "usage ./kafka-broker-stop.sh [BROKER-ID]"
  exit 1
fi

PIDS=$(ps ax | grep -i 'kafka\.Kafka' | grep java \
    | grep "kafka${BROKER_ID}.properties" | grep -v grep | awk '{print $1}')

if [ -z "$PIDS" ]; then
  echo "No kafka server to stop"
  exit 1
else
  kill -s TERM $PIDS
fi
```

Grant execute permissions to our new script.
```shell
chmod +x ~/kafka/bin/kafka-broker-stop.sh
```

Stopping an single broker.
```shell
~/kafka/bin/kafka-broker-stop.sh 1
```

## First steps with consumers and producers
The below creates a new Kafka topic to track product price updates. This creates a new `products.prices.changelog` with one partition. Since the replication-factor is 1, it is not replicated yet.
```shell
~/kafka/bin/kafka-topics.sh \
    --create \
    --topic products.prices.changelog \
    --partitions 1 \
    --replication-factor 1 \
    --bootstrap-server localhost:9092
```

Send a new emssage to the topic using the producer script.
```shell
echo "coffee pods 10" | ~/kafka/bin/kafka-console-producer.sh \
    --topic products.prices.changelog \
    --bootstrap-server localhost:9092
```

Now we can begin reading the messages from the topic by starting a new consumer. By default the consumer will begin reading at the end of the topic, but we can include a flag to tell it to start from the beginning.
```shell
~/kafka/bin/kafka-console-consumer.sh \
    --topic products.prices.changelog \
    --from-beginning \
    --bootstrap-server localhost:9092
```

We can open multiple consumer consoles in parallel to simlulate clients reading the prices changes as they occur.

With multiple clients open in parallel, start a new producer in another shell. Each line we enter will send a new msg to each consumer. Use `Ctr-D` to send an EOF signal to stop the producer. 
```shell
~/kafka/bin/kafka-console-producer.sh \
    --topic products.prices.changelog \
    --bootstrap-server localhost:9092
```

Listing topics.
```shell
~/kafka/bin/kafka-topics.sh \
    --list \
    --bootstrap-server localhost:9092
```

Describing a topic.
```shell
~/kafka/bin/kafka-topics.sh \
    --describe \
    --topic products.prices.changelog \
    --bootstrap-server localhost:9092
```

Deleting a topic.
```shell
~/kafka/bin/kafka-topics.sh \
    --delete \
    --topic products.prices.changelog \
    --bootstrap-server localhost:9092
```

Recreating the topic with new configurations.
```shell
~/kafka/bin/kafka-topics.sh \
    --create \
    --topic products.prices.changelog \
    --replication-factor 2 \
    --partitions 2 \
    --bootstrap-server localhost:9092
```

Altering an existing topic.
```shell
~/kafka/bin/kafka-topics.sh \
    --alter \
    --topic products.prices.changelog \
    --partitions 3 \
    --bootstrap-server localhost:9092
```

Producing messages containing keys.
```shell
~/kafka/bin/kafka-console-producer.sh \
    --topic products.prices.changelog \
    --property parse.key=true \
    --property key.separator=: \
    --bootstrap-server localhost:9092
```

Consuming those messages.
```shell
~/kafka/bin/kafka-console-consumer.sh \
    --from-beginning \
    --topic products.prices.changelog \
    --property print.key=true \
    --property key.separator=":" \
    --bootstrap-server localhost:9092
```

We can specify a `--group` parameter to allow designate consumer groups. T
```shell
~/kafka/bin/kafka-console-consumer.sh \
    --from-beginning \
    --topic products.prices.changelog \
    --property print.key=true \
    --property key.separator=":" \
    --group products \
    --bootstrap-server localhost:9092
```

## Reliability
### Acknowledgments
Kafka uses ACKs to ensure data messages sent by producers are received. A producer property determines the reliability level.
- `--producer-property acks=all` - (all or -1) Requires the message to have been replicated from the leader to all followers before the producer gets a reqsponse that the message has been received.
  - Default setting since Kafka 3.0. 
  - When creating topics, we can use the `min.insync.replicas` config arg to specify the min number of replicas that must be in sync before ACK is sent back to producers on `acks=all`.
  - When a reasonable min replicas value us used, this can be considered at-least-once delivery, duplicate messages can occur.
  - To achieve exactly-once delivery, Kafka allows for a producer property `--producer-property enable.idempotence=true` that will assign a sequence ID to a message. If the broker receives a message with the same sequence ID, it will simply reply with the ACK. If it receives a message out of order, it will reply with a negative ACK (NACK) to the producer.
  - Strong recommended to keep idempotence enabled.
- `--producer-property acks=1` - Only requires the leader to have received the message for the ACK is sent to the producer.
- `--producer-property acks=0` - No ACK is sent from the broker back to the producer. Lowest latency but also the lowest reliability.
  - Since the producer doesn't retry when an ACK isn't received, this can be considered at-most-once delivery.

### Transactions
Kafka allows for the use of transactions to ensure multiple actions are completed/rolled back together. The below Python code sends two messages to the broker within the same transaction. Note the use of `murmur2_random`, this allows it to be compatible with Java producers.

```python
from confluent_kafka import Producer

producer = Producer({
    'bootstrap.servers': 'localhost:9092',
    'acks': 'all',
    'enable.idempotence': True,
    'partitioner': 'murmur2_random',
    'transactional.id': 'transaction-1',
})

producer.init_transactions()
producer.begin_transaction()

producer.produce("customer.balance",
    key="bob", value="-10")
producer.produce("customer.balance",
    key="alice", value="+10")

producer.commit_transaction()
```

It's important to set the isolation.level to read_committed on the consumer, otherwise it would read messages that have not yet been committed by the producer's transaction. Consumers are agnostic of any transactions, brokers are solely responsible for handling those.

## Kafka GUIs
- (Free, open source) https://github.com/kafbat/kafka-ui
- (Enterprise) https://www.datastreamhouse.com/

## Producers
Sample that produces a message and prints a message regarding the result. The delivery_report method is called when it recieves the ACK response.
```python
from confluent_kafka import Producer
producer = Producer({
    'bootstrap.servers': 'localhost:9092',
    'acks': -1,
    'enable.idempotence': True,
    'partitioner': 'murmur2_random',
})

def delivery_report(err, msg):
    if err is not None:
        print(f"Delivery failed for record {msg.key()}: {err}")
        return
    print(f"Record {msg.key()} successfully produced to {msg.topic()}
        [{msg.partition()}] at offset {msg.offset()}")

producer.produce(
    "products.prices.changelog",
    key="cola",
    value="2",
    on_delivery=delivery_report)
```

## Consumers
Earlier samples used the `--from-beginning` flag to tell it to begin reading from the beginning instead of the end. Kafka also allows us to indicate a specific partition and offset.
```shell
~/kafka/bin/kafka-console-consumer.sh \
	--topic products.prices.changelog \
	--offset 0 \
	--partition 0 \
	--bootstrap-server localhost:9092
```

Consumers include args in their requests to the broker to indicate how quickly it should retrieve messages. The `fetch.min.bytes` setting defaults to 1 and the `fetch.max.wait.ms` setting defaults to 500ms.

## Kafka Connect
### Configuring a worker
Add a new configuration file `~/kafka/config/worker.properties`.
```shell
bootstrap.servers=localhost:9092
group.id=connect
config.storage.topic=connect-config
offset.storage.topic=connect-offset
status.storage.topic=connect-status
key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=org.apache.kafka.connect.storage.StringConverter   
plugin.path=/home/john/kafka/libs/
```

Start the new worker.
```shell
~/kafka/bin/connect-distributed.sh ~/kafka/config/worker.properties
```

### Creating a connector
Connectors are managed through the Rest API. In the below, we create a `FileStreamSource` connector to read data from a text file.

> Due to it's minimal capabilities and poor error handling, it's not recommended to use the `FileStreamSource` connector in production. The `FilePulse` connector is better suited to real world use cases.

We need to create a configuration file for the new connector, `~/kafka/config/connector.json`.
```json
{
  "name": "customers-source",
  "config": {
    "connector.class": "FileStreamSource",
    "tasks.max": "1",
    "file": "/tmp/customers.txt",
    "topic": "customers"
  }
}  
```

Now we can use `curl` to send the configuration to Kafka Connect. Note the @ symbol is needed when specifying a filename.
```shell
curl -X POST -H "Content-Type: application/json" \
    --data @/home/john/kafka/config/file-source-connector.json \
    http://localhost:8083/connectors
```

It should now show in the connector list when querying the cluster.
```shell
curl http://localhost:8083/connectors
```

And we can see it's status using the following:
> This uses `jq` for formatting the the json response and may need to be installed first `sudo apt install jq`.
```shell
curl http://localhost:8083/connectors/customers-source/status | jq
```

The worker is running but can't find the source text file, let's seed it with some initial records.
```shell
echo "Irving" >> /tmp/customers.txt
echo "Helly R." >> /tmp/customers.txt
```

Next, we'll start a consumer to read those messages. The file watcher keeps track of the last read line number, editing or deleting existing previously read rows will cause it to miss data.
```shell
~/kafka/bin/kafka-console-consumer.sh \
    --topic customers \
    --from-beginning \
    --bootstrap-server localhost:9092
```

To delete a connector:
```shell
curl -X DELETE http://localhost:8083/connectors/customers-source
```

To pause a connector:
```shell
curl -X PUT http://localhost:8083/connectors/customers-source/pause
```

To resume:
```shell
curl -X PUT http://localhost:8083/connectors/customers-source/resume
```

There are two primary types of connectors:
- Source connectors read data from external sources and produce it to Kafka.
- Sink connectors consume data and write to external systems.

## Chapter Summaries
### Chapter 1 - Introduction to Apache Kafka
- Kafka is a powerful distributed streaming platform operating on a publish-subscribe model, allowing seamless data flow between producers and consumers.
- Widely adopted across industries, Kafka excels in real-time analytics, event sourcing, log aggregation, and stream processing, supporting organizations in making informed decisions based on up-to-the-minute data.
- Kafka’s architecture prioritizes fault tolerance, scalability, and durability, ensuring reliable data transmission and storage even in the face of system failures.
- From finance to retail and telecommunications, Kafka finds applications in real-time fraud detection, transaction processing, inventory management, order processing, network monitoring, and large-scale data stream processing.
- Beyond its core messaging system, Kafka offers an ecosystem with tools such as Kafka Connect and Kafka Streams, providing connectors to external systems and facilitating the development of stream processing applications, enhancing its overall utility.
- Kafka can serve as a central hub for diverse system integration.
- Producers send messages to Kafka for distribution.
- Consumers receive and process messages from Kafka.
- Topics organize messages into channels or categories.
- Partitions divide topics to parallelize and scale processes.
- Brokers are Kafka servers managing storage, distribution, and retrieval.
- KRaft/ZooKeeper coordinates and manages tasks in a Kafka cluster.
- Kafka ensures data resilience through replication.
- Kafka scales horizontally by adding more brokers to the cluster.
- Kafka can run on general-purpose hardware.
- Kafka is implemented in Java and Scala, but there are clients for other programming languages as well, for example, Python.

### Chapter 2 - First steps with Kafka
- Kafka includes many useful scripts for managing topics and producing or consuming messages.
- Kafka topics can be created with the kafka-topics.sh command.
- Messages can be produced with the kafka-console-producer.sh command.
- Topics can be consumed with the kafka-console-consumer.sh command.
- We can consume topics again from the beginning.
- Multiple consumers can consume topics independently and at the same time.
- Multiple producers can produce into topics in parallel.
- Kafka GUIs enable users to view real-time messages within a topic, displaying details such as message key, value, and timestamp.
- These GUIs aid in effective monitoring and troubleshooting of Kafka data streams.

### Chapter 3 - Exploring Kafka topics and messages
- Kafka organizes data in topics.
- Topics can be distributed among partitions for increased performance.
- Topics can be replicated among different brokers to improve reliability.
- Topics can be created, viewed, altered, and deleted with the kafka-topics.sh script.
- The number of partitions of a topic can never be decreased.
- Partitions can be reassigned among brokers to redistribute load.
- Complex partition reassignments can be performed with the kafka-reassign-partitions.sh script.
- Kafka is optimized to exchange many (trillions of) small messages of 1 MB or less.
- Kafka messages can be classified into states, deltas, events, and commands.
- States contain the complete information about an object.
- Deltas consist only of the changes and are therefore very data-efficient, but a single delta is often not very useful. They require either a context or a complete state.
- Events add context to a message and describe a business event that happened.
- Commands are used to instruct other systems to perform actions.
- Data formats and schemas play a crucial role in Kafka to ensure consistency.
- Messages in Kafka consist of technical metadata, including a timestamp, optional custom headers (metadata), an optional key, and a value, which is the main payload.
- Messages with the same key are produced to the same partition, so the order of those messages is guaranteed for a single producer.
- Keys can also be used for log compaction to clean up deprecated data.

### Chapter 4 - Kafka as a distributed log
- A log is a sequential list where we add elements at the end and read them from a specific position.
- Kafka is a distributed log in which the data of a topic is distributed to several partitions on several brokers.
- Offsets are used to define the position of a message inside a partition.
- Kafka is used to exchange data between systems; it doesn’t replace databases, key-value stores, or search engines.
- Partitions are used to scale topics horizontally and enable parallel processing.
- Producers use partitioners to decide which partition to produce to.
- Messages with the same keys end up in the same partition.
- Consumer groups are used to scale consumers and allow them to share the workload, and one partition is always consumed by one consumer inside a group.
- Replication is used to ensure reliability by duplicating partitions across multiple brokers within a Kafka cluster.
- There is always one leader replica per partition that’s responsible for the coordination of the partition.
- Kafka consists of a coordination cluster, brokers, and clients.
- The coordination cluster is responsible for orchestrating the Kafka cluster—in other words, for managing brokers.
- Brokers form the actual Kafka cluster and are responsible for receiving, storing, and making messages available for retrieval.
- Clients are responsible for producing or consuming messages, and they connect to brokers.
- There are various frameworks and tools to easily integrate Kafka into an existing corporate infrastructure.

### Chapter 5 - Reliability
- ACKs are pillars of Kafka’s reliability.
- ACKs are optional and configurable for each producer.
- Producers can opt for no ACK (acks=0) for speed, but this sacrifices resilience.
- Producers can choose to receive ACKs after the leader receives the message (acks=1) or after replication to all ISR (acks=all), with the latter now the default and most reliable choice.
- Kafka offers three message delivery guarantees: at most once, at least once, and exactly once. At most once is achieved with acks=0; at least once is achieved with acks=all and sufficient minimum ISRs; and exactly once requires enabling idempotence and setting acks=all.
- Transactions in Kafka enable atomic writes across multiple partitions, ensuring all messages in a transaction are written together or not at all, maintaining data consistency.
- Kafka uses idempotent producers for reliable message production and a variation of the Two-Phase-Commit protocol to manage transaction commit markers, ensuring messages are processed exactly once.
- Consumers must set the isolation.level to read_committed to avoid reading uncommitted messages, ensuring only fully completed transactions are processed to maintain data integrity.
- Kafka’s transaction coordinator ensures reliability, even in the event of broker or producer crashes, although it introduces some performance overhead.
- For each partition, there’s a broker acting as the leader, handling all requests, while followers replicate data by fetching from the leader.
- If a partition leader becomes unavailable, an in-sync replica (ISR) takes over the leader role.
- When the previous leader is back in sync, it can become the leader again, as Kafka aims to reinstate the original leader, known as the preferred leader.
- In critical failures, Kafka can perform an unclean leader election by allowing non-ISRs to become leaders, which can lead to data loss.
- Kafka’s leader-follower principle ensures high availability and fault tolerance in the event of broker failures.

### Chapter 6 - Performance
- High throughput doesn’t imply low latency, but both can be equally important.
- Partitioning allows distributing the load and thus increasing performance.
- Partitioning strategy involves identifying performance bottlenecks in consumers or Kafka and adjusting partitions accordingly.
- Consider balancing partition counts to manage client RAM usage and operational complexity.
- Start with a default of 12 partitions, scaling up as needed for high throughput, while considering operational and cost implications.
- The number of partitions can never be decreased.
- Increasing the number of partitions can lead to consuming messages in the wrong order.
- A consumer group distributes load across its members.
- Batching can increase the bandwidth but also the latency.
- Batching can be configured with batch.size and linger.ms.
- Producers can compress batches to reduce the required bandwidth, but this might increase latency.
- The usage of acks=all reduces the performance of producers by a bit; the same goes for idempotence.
- Brokers won’t decompress batches; this is the task of the consumer.
- In most cases, brokers don’t require any further fine-tuning.
- Brokers open file descriptors for every partition.
- Kafka heavily depends on the operating system, necessitating specific operating system optimizations to maximize its performance.
- Consumer performance depends mostly on the number of consumers in a consumer group but can be also fine-tuned by setting fetch.max.wait.ms and fetch.min.bytes.

### Chapter 7 - Cluster management
- Kafka 3.3.0 introduced KRaft, a new coordination method based on the Raft protocol, replacing ZooKeeper from Kafka 3.5.0 onward.
- The Raft protocol resolves the complexities of achieving consensus in distributed systems, previously handled by ZooKeeper in Kafka.
- KRaft integrates coordination directly within Kafka brokers, eliminating the need for a separate ZooKeeper system, thereby improving scalability and reducing operational complexity.
- Controllers in KRaft manage partition assignments, leader elections, and failover mechanisms, ensuring cluster stability and facilitating efficient scaling and management of Kafka clusters.
- KRaft introduces the __cluster_metadata topic, ensuring consistent metadata storage across all brokers, which was previously managed by ZooKeeper, thereby streamlining cluster operations and enhancing reliability.
- Despite ZooKeeper being phased out in Kafka 4.0, a significant number of Kafka clusters currently rely on ZooKeeper for essential functions.
- It’s advisable to migrate these clusters to KRaft promptly to use improved performance and simplified management offered by the Raft-based coordination.
- ZooKeeper serves critical roles in Kafka, such as electing the controller and storing metadata (e.g., partition assignments and leader information), ensuring consistent state across all nodes.
- While ZooKeeper’s reliability in maintaining a consistent state is crucial, its inherent performance limitations and operational complexities have prompted Kafka’s shift toward the more integrated KRaft solution.
- Migrating metadata from ZooKeeper to KRaft in Kafka involves careful planning and execution without downtime. We start by upgrading our Kafka cluster to the latest version that supports ZooKeeper.
- We provision three or five additional brokers as KRaft controllers with identical cluster IDs as the ZooKeeper cluster. - We enable migration mode and configure ZooKeeper connection details.
- We configure existing brokers for metadata migration by enabling migration mode and specifying new controller-node connections. We verify successful metadata migration.
- We transition normal brokers to KRaft mode by adjusting configurations, testing stability, and finalizing the migration. We disable migration mode on controllers and remove ZooKeeper connections for a full transition to KRaft mode.
- Connecting to a Kafka cluster involves querying metadata to understand broker roles and partition leadership distribution across the cluster.
- Clients initiate connectivity through a bootstrap server, typically one of several Kafka brokers listed in the --bootstrap-server parameter, which provides initial metadata.
- To ensure resilience, production setups should specify multiple brokers in --bootstrap-server parameters or use DNS for load balancing.
- If a broker failure occurs, clients can send a new metadata request to a bootstrap server to discover new partition leaders, enabling uninterrupted message production or consumption by establishing connections with the newly elected leaders.

### Chapter 8 - Producing and persisting messages
- Producers in Kafka typically use either the official Kafka Java library or librdkafka. Avoid using other libraries due to potential lack of features and optimizations.
- The producer workflow involves serialization, partitioning, and buffer management.
- Handling acknowledgments (ACKs) in the producer includes timeout settings and retry mechanisms.
- Kafka brokers delegate much of their work to clients, focusing on message reception and efficient message persistence.
- Upon receiving a produce request, the broker writes data to the operating system’s page cache, potentially waiting for followers to replicate before sending
an ACK.
- Network threads manage message reception, queuing them for I/O threads to write to the filesystem.
- Kafka relies on the operating system to persist messages to disk, with options to influence disk flush timing.
- Broker components can be optimized and configured for specific use cases, with professional support advised for complex environments.
- Kafka’s data structures—including metadata, checkpoints, and topics—organize and manage data within brokers.
- Partitions divide topics into segments, each with log and index files for efficient data retrieval.
- Log data and indices optimize message storage and retrieval within log segments.
- Segments, based on size or time, manage partition growth and optimize data storage efficiency.
- Replication involves followers fetching data from leaders, ensuring all brokers stay up-to-date.
- In-sync replicas (ISRs) are followers that have received all messages from the leader within a specified time frame.
- The Log End Offset (LEO) marks the last received message position, determining ISR status.
- The High Watermark (HWM) indicates the offset replicated and committed to by all ISRs, affecting message consumption and availability.
- Delays in replication can slow down or halt message consumption and production, with ISR lag affecting HWM and consumer availability.
- Adjusting parameters such as replica.lag.time.max.ms and acks=all can affect replication efficiency and system performance, requiring careful configuration for balance.

### Chapter 9 - Consuming messages
- Kafka uses a fetch-based approach for consumers to retrieve messages.
- Kafka brokers handle most of the coordination work, minimizing overhead.
- Kafka brokers support rack IDs for fault tolerance and load distribution.
- Consumer groups help manage offsets and distribute workload among consumers.
- - Offsets are stored in Kafka as part of the consumer group’s metadata in a special internal topic called __consumer_offsets.
- Offsets are crucial for consumers to keep track of which messages they have already consumed.
- The Kafka Rebalance Protocol coordinates task distribution among consumer group members.
- Consumer groups facilitate parallel processing by distributing partitions among consumers.
- Rebalances are triggered by changes in group membership, topic partitions, or consumer failures.
- Range Assignor and Round Robin Assignor are strategies for partition assignment.
- Range Assignor ensures that the same consumer handles the same partitions across topics.
- Round Robin Assignor evenly distributes partitions among consumers when joins across topics aren’t required.
- Static memberships and Cooperative Sticky Assignor optimize rebalance behavior.
- Static memberships reduce rebalance frequency by extending session timeouts and using unique group instance IDs.
- Cooperative Sticky Assignor improves rebalance efficiency by iteratively approaching the desired state.

### Chapter 10 - Cleaning up messages
- Kafka offers two main approaches for message cleanup: log retention and log compaction.
- Log retention deletes messages based on age, while log compaction removes outdated data based on keys.
- Log compaction ensures that only the latest message for each key is retained, which is suitable for scenarios such as updating customer data.
- Log retention is simpler to implement, clearing messages based on age and supporting various use cases such as data privacy compliance and changelog data management.
- Both methods have tradeoffs: log retention is easier but less precise, while log compaction requires more effort but ensures data accuracy.
- Log retention can be configured based on partition size or time, offering flexibility in managing storage space.
- Log compaction parameters determine the threshold for segment cleaning and ensure that only outdated data is removed.
- Log compaction maintains message order and offsets, preserving data consistency.
- If a consumer tries to fetch messages that don’t exist, the brokers simply return the next messages.
- Kafka’s log cleaner periodically checks for outdated data and optimizes log storage by removing unnecessary segments.
- Regular segment rotation ensures efficient log management, preventing log bloat and optimizing storage usage.
- Kafka’s flexibility in message cleanup allows for tailored data management strategies based on specific use cases.
- Tombstone messages facilitate selective deletion in log compaction, ensuring efficient data cleanup.
- Tombstones are eventually removed to prevent log inflation, with configurable retention periods.

### Chapter 11 - Integrating external systems with Kafka Connect
Kafka Connect is a framework designed for scalable and reliable data integration between Kafka and other systems. It enables the movement of large amounts of data into and out of Kafka effortlessly using connectors.
- Connectors can be classified as source connectors (importing data into Kafka) or sink connectors (exporting data from Kafka).
- Each connector can be configured with various parameters to meet specific requirements.
- Kafka Connect uses a REST API for managing connectors and worker configurations, and it can also be configured using a properties file at startup.
- The REST API can be used to create, update, and delete connectors, as well as to check their status.
- Worker configurations play a vital role in determining how Kafka Connect operates, including aspects such as resource allocation, task distribution, and the overall resilience of the integration process.
- Error handling in Kafka Connect can be configured, including options for ignoring problematic entries via the errors.tolerance parameter.
- Logging errors is essential for troubleshooting, achieved through the errors.log.enable parameter.
- For sink connectors, data can be directed to a dead-letter queue for handling unprocessable records.
- Retry mechanisms can be configured for transient errors using parameters such as errors.retry.timeout and errors.retry.delay.max.ms.
- Single Message Transformations (SMTs) enable simple data modifications, including renaming fields, masking data, or moving values to keys during data transfer.
- SMTs work for both source and sink connectors but aren’t suited for complex ETL tasks.
- SMTs are set up via JSON in the Kafka Connect API, specifying transformation order, Java class types, and configurations. Common SMTs include ReplaceField, ValueToKey, ExtractField, and MaskField.
- JDBC connectors have specific configurations for managing database connection retries and backoff strategies.
- The operational mode of connectors can be set to either incrementing, timestamp-based, or both, depending on change tracking requirements.
- Change data capture (CDC) can be implemented using connectors such as Debezium to monitor database changes.
- PostgreSQL requires specific configurations (e.g., setting wal_level to logical) for effective change data capture.
- Kafka Connect can automatically create Kafka topics based on database tables, ensuring seamless integration.
- The topic.prefix parameter organizes topics generated by connectors and prevents naming conflicts.
- Connector configurations can specify database connection details and table filtering using parameters such as table.include.list.
- The framework’s flexibility allows for tailored solutions for various data integration scenarios, enhancing data flow management.

### Chapter 12 - Stream processing
- Kafka enables near real-time data exchange, eliminating delays from traditional batch processing.
- Kafka Streams, Apache Flink, Faust, and Goka are common stream-processing libraries, each with different strengths.
- Stream processing processes events individually in real time but requires managing eventual consistency.
- Stream processors such as those in Kafka Streams and Apache Flink transform, filter, and analyze real-time data streams using functional or declarative programming approaches.
- Processor types include filter() for selecting data, map() for transforming data, merge() for combining streams, and split() for branching data based on conditions.
- Stateful processing enables operations such as counting (count()) and aggregating data, which requires storing intermediate states.
- Processor topologies connect multiple processors to build complex workflows, such as merging streams, filtering significant changes, and aggregating data for insights.
- Streaming SQL allows real-time data processing similar to traditional SQL but continuously outputs results instead of a final response.
- Flink SQL enables SQL-based stream processing by defining tables linked to Kafka topics rather than storing data within the Flink cluster.
- Queries such as filtering (WHERE), aggregating (GROUP BY), and counting (COUNT(*)) work similarly to database SQL but operate on continuous data streams.
- Production usage typically relies on headless mode, where predefined SQL
queries run continuously, with results written back to Kafka or other databases.
- Streams capture continuous data, while tables store the latest state for easier
querying. Kafka Streams and Flink support both, using key-value stores and
changelog topics to prevent data loss.
- Aggregations are used to determine the current state, such as summing transactions for an account. Results are stored in tables for further analysis.
- Joins enrich data by combining streams with tables or other streams. Stream-table joins trigger on new stream data, while table-table joins trigger on changes from both sides. Stream-stream joins are possible with time windows.
- Time in Kafka involves event time (event occurrence), create time (producer creation), log append time (broker write time), and stream time (processing time).
- Time-based operations in Kafka rely on time windows to aggregate or correlate data.
Common window types include Tumbling Windows (nonoverlapping), Hopping Windows (overlapping), Sliding Windows (based on new data), and Session Windows (based on inactivity).
- Kafka Streams parallelization relies on splitting tasks based on partitions. Tasks are the smallest processing units and are distributed across Kafka Streams instances.
- State stores are created per task, and if an instance fails, the state store is rebuilt from the changelog topic. Configuring num.standby.replicas helps avoid rebuilding by using backup state stores.
- Repartitioning occurs when operations such as selectKey() are performed. Kafka Streams creates an internal repartition topic for joins and aggregations to ensure correct processing based on keys, often splitting processing into subtopologies.

### Chapter 13 - Governance
- Proper schema management in Kafka helps ensure data consistency and compatibility.
- Using schema registries allows for the centralized management of data schemas, enabling easier updates.
- Schemas facilitate versioning, allowing producers and consumers to evolve independently without breaking changes.
- Compatibility checks can be enforced to prevent incompatible schema updates that might lead to data problems.
- Schemas can be serialized in different formats, such as Avro or JSON, enhancing interoperability across systems.
- Transport Layer Security (TLS) using encryption, authentication, and authorization is mandatory for safeguarding data in Kafka.
- Authentication in Kafka can be implemented using various mechanisms, such as Simple Authentication and Security Layer (SASL), to verify the identity of clients and ensure secure connections.
- Authorization is managed through access control lists (ACLs) to regulate which users or applications can access specific Kafka resources, ensuring only authorized actions are permitted.
- Data at rest must be encrypted using underlying storage systems, as Kafka lacks native support for this.
- End-to-end encryption is recommended to secure sensitive data, implemented at the client level.
- The key management process for encryption requires careful handling, potentially using tools such as Vault.
- ZooKeeper must be secured as it stores critical Kafka metadata and transmits data unencrypted by default. Kafka 2.5 and later supports TLS connections for secure communication with ZooKeeper, and mutual TLS is advised for broker-to-ZooKeeper communication.
- By sequentially configuring TLS listeners and restarting brokers, we can migrate from an unsecured Kafka cluster to a secured one without effecting availability.
- Quotas are essential for protecting Kafka clusters from excessive load caused by misconfigured or malicious clients.
- Quotas can limit the number of produce or consume requests and restrict CPU time on brokers.
- Quotas can be set at the client ID level or user level. User-level quotas provide better reliability.
- Kafka brokers monitor throttle-time metrics to detect and manage quota violations effectively.
- Care must be taken when setting quotas, as overly aggressive limits can disrupt service and lead to cascading failures.

### Chapter 14 - Kafka reference architecture
- Kafka management can benefit from automated and declarative approaches, although it’s not a strict requirement.
- Manual creation of topics in production is strongly discouraged, as is the use of GUIs for such tasks.
- The GitOps approach is recommended for managing Kafka resources by treating a Git repository as the source of truth.
- Kafka resources such as topics, users, and ACLs should be created and modified through pull/merge requests in the Git repository.
- Automated solutions, such as kcctl, simplify the management of Kafka Connect connectors compared to using the REST API directly.
- Cruise Control for Apache Kafka is essential for balancing broker loads by redistributing partitions based on resource consumption.
- Kafka administrators should monitor broker load and manually trigger Cruise Control’s analyzer for load balancing.
- Automatic failure detection is available in Cruise Control, enabling proactive load balancing when targets aren’t met.
- Kafka can be deployed on various infrastructures, including self-managed clusters, Kubernetes, and managed services.
- The performance and reliability of Kafka are contingent on a well-planned infrastructure and a knowledgeable team.
- It’s recommended to start with at least three brokers to ensure reliability and allow for scalability based on workload.
- Storage solutions should prioritize reliable and fast disks, preferably multiple smaller SSDs over a few large ones.
- Kafka supports tiered storage, allowing older data to be offloaded to more cost-effective storage solutions.
- Sufficient RAM is critical, with a recommended minimum of 1 GB for JVM heaps per broker and an appropriate page cache based on data production rates.
- Network performance is vital for Kafka, with a minimum recommendation of 1 GBit/s Ethernet and the use of independent storage networks to avoid performance drops.
- The physical separation of brokers across different VM hosts or racks is essential to prevent data loss during hardware failures.

### Chapter 15 - Kafka monitoring and alerting
- Monitoring Kafka is vital for optimal performance and reliability.
- Infrastructure monitoring is crucial as performance problems can stem from insufficient CPU, memory, storage, or network capacity.
- Disk and network utilization should stay below 60% for effective partition rebalancing and to prevent data movement problems during broker failures.
- Contextualizing metrics is important; prolonged high resource usage may indicate a need for scaling rather than just temporary spikes.
- Kafka broker metrics, categorized under domains such as kafka_server and kafka_log, clarify their origin and significance.
- Key metrics such as under-replicated partitions and under-minimum ISRs should ideally be 0 for broker health and cluster reliability.
- Metrics related to request types and rejected messages help identify bottlenecks, emphasizing balanced load distribution among brokers.
- Monitoring network processor metrics is essential; a low average idle percentage can indicate broker overload, requiring immediate resource scaling.
- Client metrics are critical because even a strong Kafka cluster relies on clients for data production and consumption.
- General client metrics focus on infrastructure health, as CPU and network limitations can lead to processing delays affecting data reliability.
- Producer metrics such as record error total and request latency avg are vital for identifying message production problems.
- Consumer metrics, including consumer lag and rebalances, assess consumption efficiency and detect potential slowdowns.
- Kafka Connect metrics provide insights into connector health, highlighting the need to monitor dependencies on external systems.
- Kafka Streams metrics, such as processed records and processing duration, offer insights into application performance for early problem detection.
- Alerts are automated notifications based on predefined thresholds to detect anomalies early and reduce service disruptions.
- Avoid alert fatigue by setting meaningful thresholds and smoothing metrics to ensure alerts reflect real problems.
- Enhancing alerts with contextual information aids in efficient problem-solving during high-pressure situations.
- Deploying Kafka on physical hardware provides control but requires monitoring of system components such as disk health and CPU utilization.
- Kafka on VMs introduces complexities from resource contention; monitoring should cover both Kafka and the virtualization layer.
- Cloud environments offer scalability but pose challenges such as network latencies and instance performance variability; monitoring should focus on network throughput and instance health.
- Monitoring Kafka in Kubernetes involves using tools such as Prometheus and Grafana while tracking resource usage, with alerting based on critical thresholds.
- In managed Kafka services, while infrastructure is handled by providers, monitoring application-level metrics such as consumer lag and message throughput is essential for meeting SLAs.