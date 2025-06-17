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

Opening WSL folder in windows explorer
```shell
explorer.exe .
```

Open current folder in VS Code
```shell
code .
```

Install Java
```shell
sudo apt install openjdk-21-jre-headless
java --version
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


### Kafka GUIs
- (Free, open source) https://github.com/kafbat/kafka-ui
- (Enterprise) https://www.datastreamhouse.com/

## Chapter Summaries
### Chapter 1
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

### Chapter 2
- Kafka includes many useful scripts for managing topics and producing or consuming messages.
- Kafka topics can be created with the kafka-topics.sh command.
- Messages can be produced with the kafka-console-producer.sh command.
- Topics can be consumed with the kafka-console-consumer.sh command.
- We can consume topics again from the beginning.
- Multiple consumers can consume topics independently and at the same time.
- Multiple producers can produce into topics in parallel.
- Kafka GUIs enable users to view real-time messages within a topic, displaying details such as message key, value, and timestamp.
- These GUIs aid in effective monitoring and troubleshooting of Kafka data streams.

### Chapter 3
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

### Chapter 4
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

### Chapter 5
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

### Chapter 6
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

### Chapter 7
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