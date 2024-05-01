---
title: Introduction to Apache Kafka
description: This is a short intoductory survey about Apache Kafka.
author: rezataroosheh
date: 2024-12-29
categories: [Kafka]
tags: [Kafka,Message Bus]
pin: true
math: false
mermaid: false
copyright: 
media_subpath: '/posts/2024'
image:
---

Kafka functions as a distributed, partitioned, and replicated commit log service. It serves as a publish-subscribe messaging system aimed at addressing specific challenges. Often labeled as a "distributed commit log" or more recently as a "distributed streaming platform" Kafka was initially recognized as a robust message bus, facilitating the transmission of event streams without inherent processing or transformation features. Its dependable stream delivery functionality positions it ideally as a data source for stream-processing systems.

## 1. Topics and Partitions
In Kafka, messages are grouped into topics and subdivided into multiple partitions. Messages are added to partitions and read sequentially from beginning to end. Topics commonly consist of various partitions to ensure redundancy and scalability. Generally, we use the term "Event" to describe messages. Messages are durable and immutable in a partition. Although message sequencing is ensured within individual partitions, it is not guaranteed across the entire topic. Each message within a partition is assigned a distinct offset.

![anatomy of a topic](/assets/svg/kafka/anatomy-of-a-topic.svg)

Kafka records possess four primary characteristics:

- **Key**: This field, which is not mandatory, serves for record identification and possible partitioning. It may be null.
- **Value**: Representing the core message content, it contains the data intended for publication or consumption.
- **Timestamp**: Reflecting the record's production or creation time, this timestamp serves various functions such as message sequencing or deferred handling.
- **Headers**: Optional key-value pairs that furnish supplementary metadata about the record, facilitating tasks such as tracing, routing, or message prioritization.

I have pointed out the essential topic options:

1. **auto.create.topics.enable**: In Kafka's default setup, topics are automatically generated when a producer starts writing messages to the topic, a consumer begins reading messages from the topic, and a request for retrieving metadata of the topic is sent to Kafka. This automatic creation feature streamlines the management process, ensuring that topics are readily available when needed without requiring manual intervention.
2. **num.partitions**: The parameter num.partitions dictates the initial partition count for a newly created topic, especially in cases where automatic topic creation is enabled.
3. **log.retention.ms**: The log retention time is determined by parameters like log.retention.hours and log.retention.minutes, but using log.retention.ms provides more precise control.
4. **log.retention.bytes**: Messages can be automatically expired based on the total number of bytes of retained messages.



## 2. Brokers and Clusters
A single Kafka server is called a **Broker**. These Brokers are essential components of a **Kafka Cluster**, ensuring load balance within the system. Brokers in Kafka operate statelessly and rely on ZooKeeper to manage the cluster's state. They receive messages from producers, allocate offsets to them, and store them on disk. ZooKeeper facilitated the election of Kafka broker leaders. One Broker serves as the **Controller** in a cluster of brokers. This Controller oversees administrative tasks such as partition assignment to brokers and monitoring for broker failures. Each partition within the cluster is owned by a single Broker, known as the **Leader** of the partition. Replication of partitions across multiple brokers ensures redundancy in the system.

![kafka cluster](/assets/svg/kafka/kafka-cluster.svg)

### 2.1 Zookeeper-Based Controller
The controller, a Kafka broker, holds the responsibility of electing partition leaders. When the cluster initiates, the first broker to start becomes the controller by establishing an ephemeral node named "controller" in ZooKeeper. Subsequent brokers attempting to create this node encounter a "node already exists" exception, indicating the existence of the controller node and the presence of a controller in the cluster. To monitor changes, brokers set up a ZooKeeper watch on the ephemeral node, guaranteeing that the cluster always retains a solitary controller. If the controller broker stops or loses connection to ZooKeeper, the ephemeral node disappears. Other brokers are notified through the ZooKeeper watch and attempt to elect the new controller node. The first successful node becomes the new controller, while others retry. Each new controller gets a higher epoch number. Brokers ignore messages from older controllers, preventing issues like "zombie" controllers.


![kafka cluster](/assets/svg/kafka/zookeeper-based.svg)

1. **Cluster Metadata Management**: ZooKeeper maintains metadata about Kafka brokers, topics, partitions, and their configurations. It helps in discovering the current state of the Kafka cluster.
2. **Leader Election**: ZooKeeper facilitates leader election for Kafka broker nodes. It ensures that only one broker acts as the leader for a partition at any given time.
3. **Broker Registration**: Kafka brokers register themselves in ZooKeeper upon startup and update their status regularly. This registration process helps in dynamic cluster membership and management.
4. **Configuration Management**: ZooKeeper stores Kafka's configuration settings, allowing for dynamic reconfiguration of the cluster without requiring a full restart.
5. **Failure Detection**: ZooKeeper detects failures within the Kafka cluster, such as broker failures or network partitions, and triggers appropriate actions, such as leader re-elections.

### 2.2 KRaft, Raft-Based Controller
Various identified issues prompted the transition from the ZooKeeper-based controller to a Raft-based controller quorum:
1. Metadata updates are written synchronously to ZooKeeper but sent to brokers asynchronously, leading to occasional inconsistencies that are difficult to detect.
2. Controller restarts involve reading all metadata from ZooKeeper, creating a bottleneck exacerbated by the growing number of partitions and brokers.
3. The metadata ownership architecture lacks consistency, with operations distributed across the controller, brokers, and ZooKeeper.
4. Operating ZooKeeper alongside Kafka adds complexity, requiring developers to master two distributed systems instead of just one.

![kafka cluster](/assets/svg/kafka/raft-based.svg)

The Apache Kafka community has decided to replace the ZooKeeper-based controller due to concerns. As previously mentioned, ZooKeeper manages controller election and cluster metadata storage. The new controller design utilizes Kafka's log-based architecture, treating metadata as a stream of events to simplify management. In this setup, controller nodes create a Raft quorum that manages a log of metadata events, replacing ZooKeeper's role. The Raft algorithm allows controller nodes to elect leaders, eliminating the need for external systems. This design enables quick failover without requiring lengthy state transfers.

The design of the updated architecture is outlined in [KIP-500](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum).

### 2.2 Replication
It is the way Kafka guarantees availability and durability when individual nodes inevitably fail. Each topic is partitioned, and each partition can have multiple replicas. Each partition has one server which acts as the "**Leader**" and zero or more servers that act as "**Follower**" or in-sync replicas (ISR). All produce and consume requests go through the leader, to guarantee consistency. If the leader fails, one of the followers will automatically become the new leader. The replication mechanisms within the Kafka clusters are designed only to work within a single cluster, not between multiple clusters.

![Replication of partitions in a cluster](/assets/svg/kafka/replication.svg)

A replication factor of N allows you to lose N-1 brokers while still being able to read and write data to the topic reliably.

### 2.3 Partition Leader Election
When the leader for a partition is no longer available, one of the in-sync replicas will be chosen as the new leader. This leader election is "clean" in the sense that it guarantees no loss of committed data - by definition, committed data exists on all in-sync replicas.

1. Wait for a replica in the ISR to come back to life and choose this replica as the leader (hopefully it still has all its data). In this way, we lose availability.
2. Choose the first replica (not necessarily in the ISR) that comes back to life as the leader. In this way, we lose consistency.

## 3. Producers
Producers publish messages into Kafka topics. By default, the producer does not care what partition a specific message is written to and will balance messages over all partitions of a topic evenly.
We start producing messages to Kafka by creating a ProducerRecord, which must include the topic we want to send the record to and a value. Optionally, we can also specify a key and/or a partition. Once we send the ProducerRecord, the first thing the producer will do is serialize the key and value objects to ByteArrays so they can be sent over the network.

![A high-level overview of Kafka producer components](/assets/img/kafka/overview-Kafka-producer-components.png)

Next, the data is sent to a partitioner. If we specified a partition in the ProducerRecord, the partitioner doesn't do anything and simply returns the partition we specified. If we didn't, the partitioner will choose a partition for us, usually based on the ProducerRecord key. Once a partition is selected, the producer knows which topic and partition the record will go to. It then adds the record to a batch of records that will also be sent to the same topic and partition. A separate thread is responsible for sending those batches of records to the appropriate Kafka brokers.

### 3.1 Partitioner
Kafka messages are key-value pairs and while it is possible to create a ProducerRecord with just a topic and a value, with the key set to null by default partitioner chooses randomly a partition. 
But most applications produce records with keys. Keys serve two goals: they are additional information that gets stored with the message, and they are also used to decide which one of the topic partitions the message will be written to. So partition algorithm uses a formula which calculates modulo of the number of partitions, like:

```shell
Abs(Murmur2(keyBytes)) % numPartitions
```

![Messages with the same key go to the same Partition](/assets/img/kafka/Messages-with-same-key.png)

This means that if a process is reading only a subset of the partitions in a topic, all the records for a single key will be read by the same process. If a key exists and the default partitioner is used, Kafka will hash the key (using its hash algorithm), and use the result to map the message to a specific partition. However, the moment you add new partitions to the topic, this is no longer guaranteed while new records will get written to a different partition.

### 3.2 Sending Messages
There are three primary methods of sending messages:

1. **Fire-and-forget**: We send a message to the server and don't care if it arrives successfully or not. Most of the time, it will arrive successfully, since Kafka is highly available and the producer will retry sending messages automatically. However, some messages will get lost using this method.
2. **Synchronous send**: We send a message, the send() method returns a Future object, and we use get() to wait on the future and see if the send() was successful or not.
3. **Asynchronous send**: We call the send() method with a callback function, which gets triggered when it receives a response from the Kafka broker.

### 3.3 Acknowledgments
The number of acknowledgments the producer requires the leader to have received before considering a request complete. This controls the durability of records that are sent. The following settings are common:

1. **acks=0**
The producer will not wait for a reply from the broker before assuming the message was sent successfully. No guarantee can be made that the server has received the record in this case, and the retries configuration will not take effect. The offset given back for each record will always be set to -1. this setting can be used to achieve very high throughput.
2. **acks=1**
The producer will receive a successful response from the broker the moment the leader replica received the message. In this case, the leader should fail immediately after acknowledging the record but before the followers have replicated it then the record will be lost. the producer will receive an error response and can retry sending the message, avoiding potential loss of data. If the client uses callbacks, latency will be hidden, but throughput will be limited by the number of in-flight messages.
3. **acks=all**
This means the leader will wait for the full set of in-sync replicas to acknowledge the record. This guarantees that the record will not be lost as long as at least one in-sync replica remains alive. This is the strongest available guarantee.

![full acknowledge](/assets/img/kafka/full-acknowledge.png)

4. **Other acks settings**
Other settings such as acks=2 are also possible and will require the given number of acknowledgments but this is generally less useful.

### 3.4 Messages and Batches
The unit of data within Kafka is called a message. For efficiency, messages are written into Kafka in batches. A batch is just a collection of messages, all of which are being produced to the same topic and partition. There are two options to manage batches:

1. **batch.size**
When multiple records are sent to the same partition, the producer will batch them together. Too small value will add some overhead and too large will not cause delays in sending messages.

2. **linger.ms**
controls the amount of time to wait for additional messages before sending the current batch.


### 3.5 Segment
A segment is simply a collection of messages of a partition. Instead of storing all the messages of a partition in a single file (think of the log file analogy again), Kafka splits them into chunks called segments. Kafka will close a log segment either when the size limit is reached or when the time limit is reached, whichever comes first. There are some important options for the segment:

### 3.6 Idempotent Producer
Due to some issue acknowledgment does not come back to the producer while a message is sent by the producer to the broker. Producer retries and sends the same message again. The Broker sends an acknowledgment to the producer and writes the message again to the topic partition.

![failed ack](/assets/img/kafka/failed-ack.png)

Idempotent delivery ensures that messages are delivered exactly once to a particular topic partition during the lifetime of a single producer. Transactional delivery allows producers to send data to multiple partitions such that either all messages are successfully delivered, or none of them are. Every new producer will be assigned a unique PID during initialization. For a given PID, sequence numbers will start from zero and be monotonically increasing, with one sequence number per topic partition produced. The sequence number will be incremented by the producer on every message sent to the broker. The broker maintains in memory the sequence numbers it receives for each topic partition from every PID. The broker will reject a produce request if its sequence number is not exactly one greater than the last committed message from that PID/TopicPartition pair. Messages with a lower sequence number result in a duplicate error, which can be ignored by the producer.

![retry msg](/assets/img/kafka/retry-msg.png)

## 4. Consumers
The consumer subscribes to one or more topics and reads the messages in the order in which they were produced. Before discussing the specifics of Apache Kafka Consumers, we need to understand the concept of publish-subscribe messaging, message queueing and theirs differences.

### 4.1 Traditional Message Queuing
In a queue, a pool of consumers may read from a server and each message goes to one of them.

![traditional message queueing](/assets/img/kafka/traditional-message-queueing.png)

### 4.2 Publish-Subscribe
Publish-Subscribe messaging is a pattern that is characterized by the sender (publisher) of a piece of data (message) not specifically directing it to a receiver. Instead, the publisher classifies the message somehow, and that receiver (subscriber) subscribes to receive certain classes of messages. Pub/Sub systems often have a broker, a central point where messages are published, to facilitate this.

![publish subscribe](/assets/img/kafka/publish-subscribe.png)

Kafka offers a single consumer abstraction that generalizes both of these by the consumer group. Kafka consumers are typically part of a consumer group. When multiple consumers are subscribed to a topic and belong to the same consumer group, each consumer in the group will receive messages from a different subset of the partitions in the topic. Consumer instances can be in separate processes or on separate machines.

### 4.3 Consumers in the same consumer group
If all the consumer instances have the same consumer group, then this works just like a traditional queue balancing load over the consumers. By storing the offset of the last consumed message for each partition, either in Zookeeper or in Kafka itself, a consumer can stop and restart without losing its place. When a consumer subscribes to a topic can be assigned to partitions by brokers as:

1. **Assigned to more than one partitions**: Consumers in a consumer group are less than the partitions. In this case, the brokers assign to some consumers more than one partitions.

![partitions more than one consumers](/assets/img/kafka/partitions-more-than-one-consumers.png)

2. **Assigned to exact one partition**: Consumers in a consumer group are equal to the partitions. In this case, the brokers will assign partitions to consumers one by one.

![Partitions count equals to consumers count](/assets/img/kafka/partitions-count-equals-to-consumers-count.png)

3. **Not Assigned to partition**: Consumers are more than partitions in a consumer group. In this case, brokers don't assign partitions to some consumers and those consumers will be Idle.

![Not Assigned to partition](/assets/img/kafka/not-sssigned-to-partition.png)

### 4.4 Consumers in different consumer groups
If all the consumer instances have different consumer groups, then this works like publish-subscribe and all messages are broadcast to all consumers.

![different consumer](/assets/img/kafka/different-consumer.png)

Just increasing the number of consumers won't increase the parallelism. You need to scale your partitions accordingly. To read data from a topic in parallel with two consumers, you create two partitions so that each consumer can read from its partition. Also since partitions of a topic can be on different brokers, two consumers of a topic can read the data from two different brokers.

### 4.5 Message Delivery Semantics
1. At most once: Messages may be lost but are never redelivered.
2. At least once: Messages are never lost but may be redelivered.
3. Exactly once: this is what people want, each message is delivered once and only once.

## 4.6 Reliability Guarantees
1. Kafka guarantees messages order in a partition.
2. Produced messages are considered "committed" when they were written to the partition on all its in-sync replicas (but not necessarily flushed to disk).
3. Messages that are committed will not be lost as long as at least one replica remains alive.
4. Consumers can only read messages that are committed.


## References
- Kafka: The Definitive Guide: Real-Time Data and Stream Processing at Scale  -  by Neha Narkhede, Gwen Shapira, Todd Palino
- Apache Kafka - Nishant Garg
- Learning Apache Kafka Second Edition - Nishant Garg
- Cloudera: Reference Guide for Deploying and Configuring Apache Kafka
- [https://docs.confluent.io](https://docs.confluent.io)
- [https://kafka.apache.org/081/documentation.html](https://kafka.apache.org/081/documentation.html)
- [https://medium.com/@stephane.maarek/how-to-use-apache-kafka-to-transform-a-batch-pipeline-into-a-real-time-one-831b48a6ad85](https://medium.com/@stephane.maarek/how-to-use-apache-kafka-to-transform-a-batch-pipeline-into-a-real-time-one-831b48a6ad85)
- [https://medium.com/@durgaswaroop/a-practical-introduction-to-kafka-storage-internals-d5b544f6925f](https://medium.com/@durgaswaroop/a-practical-introduction-to-kafka-storage-internals-d5b544f6925f)
- [https://www.confluent.io/blog/5-things-every-kafka-developer-should-know/](https://www.confluent.io/blog/5-things-every-kafka-developer-should-know/)
- [https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.0.1/kafka-working-with-topics/content/creating_a_kafka_topic.html](https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.0.1/kafka-working-with-topics/content/creating_a_kafka_topic.html)
- [https://hevodata.com/blog/kafka-exactly-once-semantics/](https://hevodata.com/blog/kafka-exactly-once-semantics/)
- [https://www.udemy.com/certificate/UC-e00cba5e-6ef1-4034-b002-7f8c0ab34b0b/](https://www.udemy.com/certificate/UC-e00cba5e-6ef1-4034-b002-7f8c0ab34b0b/)
- [https://developer.confluent.io/courses/architecture/control-plane/](https://developer.confluent.io/courses/architecture/control-plane/)