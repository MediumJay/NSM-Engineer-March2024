# Overview

# Step 1 - Get onto pipeline containers to install Kafka 
- `ssh pipeline0`

# Step 2 - Install Kafka and Zookeeper
- `sudo yum install kafka zookeeper -y`

# Step 3 - Configure Zookeeper first so its ready to manage - create data directory
- `sudo mkdir -p /data/zookeeper`

# Step 4 - Change Permissions to directory
- `sudo chown -R zookeeper: /data/zookeeper/`

# Step 5 - Configure zoo.cfg
- `sudo vi /etc/zookeeper/zoo.cfg`
- Important sections: 
```

```

- Make these edits: 
  - `:set nu`
  - `:%d` - just nuke it and use the provided one
```
 # where zookeeper will store its data
 dataDir=/data/zookeeper

 # what port should clients like kafka connect on
 clientPort=2181

 # how many clients should be allowed to connect, 0 = unlimited
 maxClientCnxns=0

 # list of zookeeper nodes to make up the cluster
 # First port is how followers and leaders communicate
 # Second port is used during the election process to determine a leader
 server.1=pipeline0:2888:3888
 server.2=pipeline1:2888:3888
 server.3=pipeline2:2888:3888

 # more than one zookeeper node will have a unique server id.
 # Ex.) server.1, server.2, etc..

 # milliseconds in which zookeeper should consider a single tick
 tickTime=2000

 # amount of ticks a follow has to connect and sync with the leader
 initLimit=5

 # amount of ticks a follower has to sync with a leader before being dropped
 syncLimit=2
```

# Step 6 - Create an ID for each zookeeper instance
- `sudo touch /data/zookeeper/myid`

# Step 7 - Give zookeeper permissions to the id
- `sudo chown -R zookeeper: /data/zookeeper/myid`

# Step 8 - Enter the respective IDs into the myid file
instance | serverID
pipeline0 | 1
pipeline1 | 2
pipelien2 | 3

- `echo '#' | sudo tee /data/zookeeper/myid` 
  - one way
- `sudo vi /data/zookeeper/myid` 
  - another way

# Step 9 - Open the firewall
- `sudo firewall-cmd --add-port={2181,2888,3888}/tcp --permanent`
- `sudo firewall-cmd --reload`

# Step 10 - Repeat steps 2 through 9 on the other pipelines

# Step 11 - Enable and start zookeeper on all pipeline containers
-`sudo systemctl enable --now zookeeper`


# Step 12 - On the Ubuntu Laptop issue the stats command to each of the Zookeeper clients on client port
- this script verfies communications (leader and 2 followers)
```
for host in pipeline{0..2}; do (echo "stats" | nc $host 2181 -q 2); done
```
- looker for "mode: " it should have 2 followers and 1 leader
- order doesnt matter








# Step 13 - Configure Kafka by making /data/kafka 
- `sudo mkdir -p /data/kafka`
- `sudo chown -R kafka: /data/kafka`

# Step 14 - Create a copy of our server.properties before we edit it 
- `sudo cp /etc/kafka/server{.properties,.properties.bk}`


# Step 15 - Edit server.properties
- `sudo vi /etc/kafka/server.properties`
instance | brokerID
pipeline0 | 0
pipeline1 | 1
pipeline2 | 2
- change line 3 for the respective pipeline
- change line 15 listeners hostname for resp pipeline
- change line 21 advertised ilisteners hostname for resp pipeline
- make these changes: 
```
# The unique id of this broker should be different for each kafka node. Good practice is to match the kafka broker id to the zookeeper server id.
broker.id=0

# the port in wich kafka should use to communicate with other kafka clients
port=9092

# the hostname or IP address in which the server listens on
listeners=PLAINTEXT://pipeline0:9092

# hostname that will be advertised to producers and consumers
advertised.listeners=PLAINTEXT://pipeline0:9092

# number of threads used to send network responses
num.network.threads=3

# number of threads used to make I/O requests
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# where kafka should write its data to
log.dirs=/data/kafka

# how many partitions and replicas should be generated for topics that are created by other software
num.partitions=3
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
default.replication.factor = 3
min.insync.replicas = 2

# how many threads should be used for shutdown and start up
num.recovery.threads.per.data.dir=3

# how long should we retain logs in kafka
log.retention.hours=12
log.retention.bytes=90000000000

# max size of a single log file
log.segment.bytes=1073741824

# frequency in miliseconds to check if a log needs to be deleted
log.retention.check.interval.ms=300000
log.cleaner.enable=false

# will not allow a node to be elected leader if it is not in sync with other nodes. Prevents possible missing messages
unclean.leader.election.enable=false

# automatically create topics from external software
auto.create.topics.enable=false


# how to connect kafka to zookeeper
zookeeper.connect=pipeline0:2181,pipeline1:2181,pipeline2:2181
zookeeper.connection.timeout.ms=30000
```

# Step 16 - Open the firewall
- `sudo firewall-cmd --add-port=9092/tcp --permanent; sudo firewall-cmd --reload`

# Step 17 - Repeat steps 13 to 16 for the other pipelines

# Step 18 - Start all 3 kafkas but dont enable boot start
- `sudo systemctl start kafka`

# Step 19 - Create a test topic to verify Kafka cluster on pipeline0
## Create Topic
`sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --create --topic test --partitions 3 --replication-factor 3`
- no errors is good

## Validate Topic
`sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --list`
- should see "test" listed
## Validate Partitions
`sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --describe --topic test`
- shows partitions, leaders, and replicas. 
- shows the cluster is working

## Mark For Deletion
`sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --delete --topic test`



# Step 20 - Create zeek-raw topic and Move Zeek data over to kafka
- create zeek-raw topic so there is a place for the zeek data to go
  - `sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --create --topic zeek-raw --partitions 3 --replication-factor 3`
- checkout the topic
  - `sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --describe --topic zeek-raw`

# Step 21 - Go back to you sensor
- `ssh sensor`

# Step 22 - Add the kafka plugin into zeek via kafka.zeek script
- plugins are at /usr/share/zeek/site/scripts
- make these edits to kafka.zeek:
  - `:set nu` 
```
6 redef Kafka::kafka_conf = table(
7 ["metadata.broker.list"] = "pipeline0:9092,pipeline1:9092,pipeline2:9092"); 
```
# Step 23 - Add the kafka script to local.zeek 
- `sudo vi /usr/share/zeek/site/local.zeek`
- make these changes at the bottom: 
  - shift+g
```
109 @load scripts/kafka
```


# Step 24 - redeploy zeek
- `sudo -u zeek zeekctl deploy`
- `sudo -u zeek zeekctl status`

# Step 25 - Verify zeek traffic is making it to Kafka
- `ssh pipeline0`
- `sudo /usr/share/kafka/bin/kafka-console-consumer.sh --bootstrap-server pipeline0:9092 --topic zeek-raw`
- generate traffic with a curl and should see it output from the previous command