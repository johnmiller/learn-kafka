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