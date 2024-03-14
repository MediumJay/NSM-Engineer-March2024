# Overview
- Friday Objective: Do a complete rebuild so far up to Zeek and Kafka
- Set static IPs
- Configure Capture interface
- Configure Stenographer
- Configure Suricata
- Configure Zeek 
- Configure FSF
- Configure Zookeeper and Kafka


# Step 1 - List your containers and check IPs
- `sudo lxc list`
- all IPs are randomized with exception of the repo

# Step 2 - Set Static IP for a container

- need to remove known_hosts first on the laptop since containers were reset
- `sudo vi /home/ubuntu/.ssh/known_hosts`
  - `:%d` wipes the file clean in vim
- `sudo vi /root/.ssh/known_hosts`
- `ssh elastic@<containerip>`

# Step 3 - Edit Eth0 interface

- `sudo vi /etc/sysconfig/network-scripts/ifcfg-eth0;sudo systemctl restart network`
  - make it look like this: 
```
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=<containername>
NM_CONTROLLED=no
IPADDR=
GATEWAY=
PREFIX=
```

# Step 4 - Restart the network

- `sudo systemctl restart network`

# Step 5 - Repeat steps 3 through 4 for each container


# Step 6 - Edit the /etc/hosts file on "Laptop" to add the container hostnames + ips
- `sudo vi /etc/hosts/`
```
ip hostname
ip hostname
ip hostname
```
- if these entries already exist then /etc/hosts is good

# Step 7 - Copy the /etc/hosts from the Laptop VM to the containers via the script
This is so we dont have to type IP addresses when SSH-ing

- run this script minus the repo hostname: 
```
for host in elastic{0..2} pipeline{0..2} kibana sensor; do sudo scp /etc/hosts elastic@$host:~/hosts && ssh -t elastic@$host 'sudo mv ~/hosts /etc/hosts && sudo systemctl restart network'; done
```

# Step 8 - The ssh-keygen was done already and so was the ssh config so move on
keys already generated (see 02-Configure Environment)

# Step 9 - Copy the ssh keys to all the containers
So we dont have to type password when ssh-ing
- run this script minus repo:
```
for host in sensor elastic{0..2} pipeline{0..2} kibana; do ssh-copy-id $host; done
```

# Step 10 - Push the LocalCA cert from repo container to the other containers

- certs already created and repo configured so skip that process
- ssh into the repo 
  - `ssh repo`
- use this script: 

```
for host in elastic{0..2} pipeline{0..2} kibana sensor; do sudo scp ~/certs/localCA.crt elastic@$host:~/localCA.crt && ssh -t elastic@$host 'sudo mv ~/localCA.crt /etc/pki/ca-trust/source/anchors/ && sudo update-ca-trust'; done
```

# Step 11 - Point the containers to the local repo


- pastable for time saving: 
```
mkdir ~/archive;sudo mv /etc/yum.repos.d/* ~/archive/;sudo vi /etc/yum.repos.d/local.repo;sudo yum makecache fast;sudo yum list suricata
```

- `ssh <container name>`
- `mkdir ~/archive`
- `sudo mv /etc/yum.repos.d/* ~/archive/`
- `sudo vi /etc/yum.repos.d/local.repo`
  - paste this config in to set up the local repos: 

```
[local-base]
name=local-base
baseurl=https://repo/packages/local-base/
enabled=1
gpgcheck=0

[local-rocknsm-2.5]
name=local-rocknsm-2.5
baseurl=https://repo/packages/local-rocknsm-2.5/
enabled=1
gpgcheck=0

[local-elasticsearch-7.x]
name=local-elasticsearch-7.x
baseurl=https://repo/packages/local-elastic-7.x/
enabled=1
gpgcheck=0

[local-epel]
name=local-epel
baseurl=https://repo/packages/local-epel/
enabled=1
gpgcheck=0

[local-extras]
name=local-extras
baseurl=https://repo/packages/local-extras/
enabled=1
gpgcheck=0

[local-updates]
name=local-updates
baseurl=https://repo/packages/local-updates/
enabled=1
gpgcheck=0
```
- `sudo yum makecache fast` 
- `sudo yum list suricata` - to check yum is puling from local repo



# Step 12 - Move on to the Sensor capture interface configuration

- `ssh sensor`

- `sudo yum install ethtool -y` - to show features of interfaces

- Check if checksumming feature is turned on
  - `sudo ethtool -k eth1`
  - notice it is on so download script from repo to disable

- `sudo curl -LO https://repo/fileshare/interface.sh`

- `sudo chmod +x interface.sh`

- `sudo ./interface.sh eth1`

- `sudo ethtool -k eth1` - again to check

- quick pastable: 
```
sudo yum install ethtool -y;sudo ethtool -k eth1;sudo curl -LO https://repo/fileshare/interface.sh;sudo chmod +x interface.sh;sudo ./interface.sh eth1;sudo ethtool -k eth1
```

# Step 13 - Make the checksum disable persistent by creating a script

- `sudo vi /sbin/ifup-local`
  - insert this script: 
```
#!/bin/bash
if [[ "$1" == "eth1" ]]
then
for i in rx tx sg tso ufo gso gro lro rxvlan txvlan
do
/usr/sbin/ethtool -K $1 $i off
done
/usr/sbin/ethtool -N $1 rx-flow-hash udp4 sdfn
/usr/sbin/ethtool -N $1 rx-flow-hash udp6 sdfn
/usr/sbin/ethtool -n $1 rx-flow-hash udp6
/usr/sbin/ethtool -n $1 rx-flow-hash udp4
/usr/sbin/ethtool -C $1 rx-usecs 10
/usr/sbin/ethtool -C $1 adaptive-rx off
/usr/sbin/ethtool -G $1 rx 4096

/usr/sbin/ip link set dev $1 promisc on

fi
```
- make it executable
  - `sudo chmod +x /sbin/ifup-local`



# Step 14 - Modify the ifup script to call our ifup-local script

- `sudo vi /etc/sysconfig/network-scripts/ifup`
  - shift + g + o to go to bottom and then insert one line above the "exec" line
  - add this change: 
```
if [ -x /sbin/ifup-local ]; then
/sbin/ifup-local ${DEVICE}
fi
```



# Step 15 - Edit interface Eth1
- `sudo vi /etc/sysconfig/network-scripts/ifcfg-eth1`
- add these changes:
```
DEVICE=eth1
BOOTPROTO=none
ONBOOT=yes
NM_CONTROLLED=no
TYPE=Ethernet
```

# Step 16 - Restart the network to verify the checksumming remains off
- `sudo systemctl restart network`
- `sudo ethtool -k eth1`
- notice that the checksumming remains off. Yippee
- can also see via `ip a` that eth1 is in promisc mode

# Step 17 - test our capture interface
- we will use tcpdump to monitor our network traffic and test inteface
- `sudo tcpdump -nn -i eth1`
  - we are seeing SSH traffic so we will add a filter
  - `sudo tcpdump -nn -i eth1 '!port 22'`
- hop on another container and curl for google, should see it pop on tcpdump on the sensor












# STENOGRAPHER STENOGRAPHER STENOGRAPHER STENOGRAPHER STENOGRAPHER STENOGRAPHER


# Step 1 - Get onto the Sensor to configure Stenographer
- `ssh sensor`

# Step 2 - Speed command
- `sudo yum list stenographer;sudo yum install stenographer -y;sudo stenokeys.sh stenographer stenographer;ll /etc/stenographer/certs/;sudo mkdir -p /data/stenographer/{packets,index};sudo chown -R stenographer:stenographer /data/stenographer`

# Step 3 - Begin configuring Stenographer by editing the config file 
- Edit the config file
  - `sudo vi /etc/stenographer/config`
- make the following changes:
```
"PacketsDirectory": "/data/stenographer/packets"
"IndexDirectory": "/data/stenographer/index"
"DiskFreePercentage": 30 
"Interface":"eth1"
```
- Note: DiskFreePercentage is just the amount of disk that must always remain free

# Step 4 - Start and Enable Stenographer service
- `sudo systemctl enable stenographer --now;^enable^status`

# Step 5 - Verify Stenographer is capturing packets
- On sensor `ping 8.8.8.8 -c 5`
- Carve it `sudo stenoread 'host 8.8.8.8' -nn`
- Traffic can take a while to carve. 
- Rerun stenoread command to eventually see the traffic. 
- 5+ minutes and a config might be messed up. 


# Step 6 - Verify another way
- check our directories to see if they're being written
- `ll /data/stenographer/packets;ll /data/stenographer/index`
















# SURICATA SURICATA SURICATA SURICATA SURICATA SURICATA 

# Step 1 - Get on the Sensor
- `ssh sensor` 

# Step 2 - Speedrun command
- `sudo yum install suricata -y;sudo vi /etc/suricata/suricata.yaml;sudo suricata-update add-source emergingthreats https://repo/fileshare/emerging.rules.tar.gz;sudo suricata-update;sudo mkdir /data/suricata;sudo chown -R suricata: /data/suricata;sudo systemctl enable --now suricata`

# Step 3 - Configure the Suricata.yaml file
- This config is HUGE (version 5.0.1 for class)
- make the following changes on the respective lines:
  - `:###` will help you jump around 
```
56 default-log-dir: /data/suricata/
60 enabled: no (turns off global statistics)
76 enabled: no (turns off fast.log)
404 enabled: no (turns off statistics log)
557 enabled: no (turns console output off)
580 - interface: eth1 (sets capture int for af-packet)
582 threads: 3 (uncomment and set to 3)
981 run-as (uncomment)
982 user: suricata (uncomment and change to suricata)
983 group: suricata (uncomment and change to suricata)

1434 set-cpu-affinity: yes (turns on CPU pinning)
1452 cpu: ["0-2"] (pinning cores 0 through 2)
1458 low [0] (sets affinity???)
1459 medium: [1] (sets affinity???)
1460 high [2] (sets affinity???)
1461 default: "high" (sets affinity???)

1500 enabled: no (turns off rule_perf log)
1516 enabled: no (turns off keyword_perf log)
1521 enabled: no (turns off prefilter_perf log) 
1527 enabled: no (turns off rule_group_perf log)
1536 enabled: no (turns off packet_stats log)
```

# Step 4 - Configure /etc/sysconfig/suricata
- This file plugs in the options into the service file so we can use af-packet and run it as user/group suricata
  - the service file is what configures services when managed by systemctl
- `sudo vi /etc/sysconfig/suricata`
- make the following change: 
```
OPTIONS="--af-packet=eth1 --user suricata --group suricata" 
```

# Step 38 - Generate some traffic and Look at eve.json
- `ping 8.8.8.8; curl google.com;sudo cat /data/suricata/eve.json`


























# ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK 

# Step 1 - SSH into the sensor to begin configuring Zeek
- `ssh sensor`

# Step 2 - quick command

- `sudo yum list zeek;sudo yum install zeek zeek-plugin-kafka zeek-plugin-af_packet -y;sudo vi /etc/zeek/zeekctl.cfg;sudo vi /etc/zeek/node.cfg;sudo mkdir /usr/share/zeek/site/scripts;cd /usr/share/zeek/site/scripts;`

# Step 3 - edit zeekctl.cfg
- Insert the following changes:
  - `:set nu`
```
67 LogDir = /data/zeek/ (location of archived logs)
68 lb_custom.InterfacePrefix=af_packet:: (tells zeek to use afpacket)
69 /n
```

# Step 3 - Edit node.cfg
- Insert the following changes:
  - `:set nu`
```
Comment 8-11 to disable standalone
8 #[zeek]
9 #type=standalone
10 #host=localhost
11 #interface=eth0

Uncomment 16-36 to enable clustered and make changes: 
16 [logger]
17 type=logger
18 host=localhost
19
20 [manager]
21 type=manager
22 host=localhost
23 pin_cpus=1(pin a cpu core)
24 
25 [proxy-1]
26 type=proxy
27 host=localhost
28
29 [worker-1]
30 type=worker
31 host=localhost
32 inteface=eth1
33 lb_method=custom (load balance method we said was afpacket)
34 lb_procs=2 (how many processes allocated for workers)
35 pin_cpus=2,3 (cores to assign)
36 env_vars=fanout_id=77 (77 is arbitrary but what we are using; groups all processes to single network socket with afpacket)

delete lines 34-37 for worker 2
```

# Step 4 - Curl down zeek scripts from local repo
```
mass curl: 
sudo curl -LO https://repo/fileshare/zeek/afpacket.zeek;sudo curl -LO https://repo/fileshare/zeek/extension.zeek;sudo curl -LO https://repo/fileshare/zeek/extract-files.zeek;sudo curl -LO https://repo/fileshare/zeek/fsf.zeek;sudo curl -LO https://repo/fileshare/zeek/json.zeek;sudo curl -LO https://repo/fileshare/zeek/kafka.zeek
```



# Step 5 - Set zeek to utilize our custom zeek scripts
- edit the local.zeek file
- `sudo vi /usr/share/zeek/site/local.zeek`
- make these edits at the bottom: 
  - `:set nu`
  - shift+g+o
```
104 @load scripts/json
105 @load scripts/afpacket
106 @load scripts/extension
107 /n
108 redef ignore_checksums = T; (ignore checksums)
```


# Step 6 - Create /data/zeek adn Change ownership of directories & binaries to Zeek
- mass chown
```
sudo mkdir /data/zeek;sudo chown -R zeek: /etc/zeek;sudo chown -R zeek: /var/spool/zeek;sudo chown -R zeek: /data/zeek;sudo chown -R zeek: /usr/share/zeek;sudo chown -R zeek: /usr/bin/zeek;sudo chown -R zeek: /usr/bin/capstats
```

# Step 7 - Set capabilities for zeek user to look at raw network traffic and administer interface
- quick pastable: 
```
sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/zeek;sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/capstats;sudo getcap /usr/bin/zeek;sudo getcap /usr/bin/capstats
```


# Step 8 - Deploy zeek as zeek user
- `sudo -u zeek zeekctl deploy`
  - `sudo -u zeek zeekctl diag` for error checking


# Step 9 - Test zeek by checking out a log using JQ
- `curl google.com;tail /data/zeek/current/http.log | jq` 




























# FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF 

# Step 1 - SSH into the sensor to configure FSF
- `ssh sensor`



# Step 2 - Quick Command
- `sudo yum list fsf;sudo yum install fsf -y;sudo vi /opt/fsf/fsf-server/conf/config.py;sudo mkdir -p /data/fsf/archive;sudo chown -R fsf: /data/fsf;sudo vi /opt/fsf/fsf-client/conf/config.py;sudo systemctl enable --now fsf;sudo systemctl status fsf;cd;/opt/fsf/fsf-client/fsf_client.py --full interface.sh | jq;ll /data/fsf;cd /usr/share/zeek/site/scripts;sudo vi /usr/share/zeek/site/local.zeek`

# Step 3 - Edit server config.py file
- make these edits: 
```
9 'LOG_PATH' : '/data/fsf',
10 'YARA_PATH' : '/var/lib/yara-rules/rules.yara', 
11 'PID_PATH' : '/run/fsf/fsf.pid', 
12 'EXPORT_PATH' : '/data/fsf/archive',
15 'ACTIVE_LOGGING_MODULES' : ['rockout'],
18 SERVER_CONFIG = {'IP_ADDRESS' : "localhost", 'PORT' : 5800 }
```


# Step 4 - Edit client config.py to match server config.py
- make these edits: 
```
9 SERVER_CONFIG = {'IP_ADDRESS' : ['localhost'], 'PORT' : 5800}
```


# Step 5 - Make FSF work in conjunction with Zeek 
- make these edits: 
  - shift + g
```
107 @load scripts/extract-files
108 @load scripts/fsf
109
110 redef ignore_checksums = T; 
```


# Step 6 - Redeploy Zeek for changes to take effect
- `sudo -u zeek zeekctl deploy;sudo -u zeek zeekctl status`

# Step 7 - Lets test zeek carves and FSF scans
- `curl google.com;ll /data/zeek;tail /data/fsf/rockout.log | jq`
  - can see "extract" in the file name for zeek scanned files
  - can see "interface.sh" in the file name for that manually scanned file


































  # ZOOKEEPER ZOOKEEPER ZOOKEEPER ZOOKEEPER ZOOKEEPER ZOOKEEPER ZOOKEEPER ZOOKEEPER ZOOKEEPER

# Step 1 - Get onto pipeline containers to install Kafka and Zookeeper
- `ssh pipeline0` (look at step 72 for pastable)


# Step 2 - Quick command
- Enter the respective IDs into the myid file
instance | serverID
pipeline0 | 1
pipeline1 | 2
pipelien2 | 3
- easy pastable just make sure to replace the # with the respective serverID:
```
sudo yum install kafka zookeeper -y;sudo mkdir -p /data/zookeeper;sudo chown -R zookeeper: /data/zookeeper/;sudo vi /etc/zookeeper/zoo.cfg;echo '#' | sudo tee /data/zookeeper/myid;sudo firewall-cmd --add-port={2181,2888,3888}/tcp --permanent;sudo firewall-cmd --reload;sudo firewall-cmd --list-all
```

# Step 3 - Configure zoo.cfg
- `sudo vi /etc/zookeeper/zoo.cfg`
- Make these edits: 
  - `:set nu`
  - `:%d` - just nuke it and use the provided one:

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

# Step 4 - Enable and start zookeeper on all pipeline containers
- `sudo systemctl enable --now zookeeper`


# Step 5 - On the Ubuntu Laptop issue the stats command to each of the Zookeeper clients on client port
- this script verfies communications (leader and 2 followers)
```
for host in pipeline{0..2}; do (echo "stats" | nc $host 2181 -q 2); done
```
- looking for "mode: " it should have 2 followers and 1 leader
- order doesnt matter












































 # KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA

# Step 1 - Get onto the Pipelines and Configure Kafka by making /data/kafka 
- `ssh pipeline0` 
- easy pastable: 

```
sudo mkdir -p /data/kafka;sudo chown -R kafka: /data/kafka;sudo cp /etc/kafka/server{.properties,.properties.bk};sudo vi /etc/kafka/server.properties;sudo firewall-cmd --add-port=9092/tcp --permanent;sudo firewall-cmd --reload;sudo firewall-cmd --list-all
```

# Step 2 - Edit server.properties
- `sudo vi /etc/kafka/server.properties`
instance | brokerID
pipeline0 | 0
pipeline1 | 1
pipeline2 | 2
- change line 3 for the respective pipeline
- change line 15 listeners hostname for resp pipeline
- change line 21 advertised ilisteners hostname for resp pipeline
- make them in the good config: 
  -`:%d` and then add above lines
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



# Step 3 - Start all 3 kafkas but dont enable boot start
- `sudo systemctl start kafka`

# Step 4 - Create a test topic to verify Kafka cluster on pipeline0
```
sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --create --topic test --partitions 3 --replication-factor 3;sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --list;sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --describe --topic test;sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --delete --topic test
```



# Step 5 - Create zeek-raw topic and Move Zeek data over to kafka
- create zeek-raw topic so there is a place for the zeek data to go
  - `sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --create --topic zeek-raw --partitions 3 --replication-factor 3`

# Step 6 - Go back to your sensor
- `ssh sensor`

# Step 7 - Add the kafka plugin into zeek via kafka.zeek script
- `sudo vi /usr/share/zeek/site/scripts/kafka.zeek`
- make these edits to kafka.zeek:
  - `:set nu` 
```
6 redef Kafka::kafka_conf = table(
7 ["metadata.broker.list"] = "pipeline0:9092,pipeline1:9092,pipeline2:9092"); 
```

# Step 8 - Add the kafka script to local.zeek 
- `sudo vi /usr/share/zeek/site/local.zeek`
- make these changes at the bottom: 
  - shift+g
```
109 @load scripts/kafka
```


# Step 9 - redeploy zeek
- `sudo -u zeek zeekctl deploy;sudo -u zeek zeekctl status`

# Step 10 - Verify zeek traffic is making it to Kafka
- `ssh pipeline0`
- `sudo /usr/share/kafka/bin/kafka-console-consumer.sh --bootstrap-server pipeline0:9092 --topic zeek-raw`
- generate traffic with a curl and should see it output from the previous command


 




















































# FILEBEAT FILEBEAT FILEBEAT


Next we will setup Filebeat to get Suricata and FSF to kafka (Zeek did so through plugin)

# Step 1 - SSH into the sensor

- `ssh sensor`
- Speed run command:

```
sudo yum install filebeat -y;sudo mv /etc/filebeat/filebeat{.yml,.yml.bk};cd /etc/filebeat/;sudo curl -LO https://repo/fileshare/filebeat/filebeat.yml;sudo vi filebeat.yml
```

# step 2 edit filebeat.yml

  - make these changes: 
  - `:set nu`

```
34 hosts: ["pipeline0:9092","pipeline1:9092","pipeline2:9092"]
```


# Step 3 - ssh into pipeline0 and create fsf-raw and suricata-raw topics with kafka-topics.sh then verify them
- `ssh pipeline0`
- create the new topics and list them
  
```
sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --create --topic fsf-raw --partitions 3 --replication-factor 3;sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --create --topic suricata-raw --partitions 3 --replication-factor 3;sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --list` to check topics;sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --describe --topic suricata-raw;sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --describe --topic fsf-raw
```

# Step 4 - Start filebeat on the Sensor
- `sudo systemctl start filebeat`
- `^start^status`

# Step 5 - Validate logs on pipeline0 by generating traffic from another container (sensor is fine)
- `sudo /usr/share/kafka/bin/kafka-console-consumer.sh --bootstrap-server pipeline0:9092 --topic suricata-raw`
- `sudo /usr/share/kafka/bin/kafka-console-consumer.sh --bootstrap-server pipeline0:9092 --topic fsf-raw`
- Note: you may need to restart suricata first. It can take some time






































# ElasticSearch ElasticSearch ElasticSearch ElasticSearch ElasticSearch

# Step 1 - SSH onto elastic0 and use speed run command
- `ssh elastic0` (step 10 for total pastable)

- easy pastable: 
```
sudo yum install elasticsearch -y;sudo mkdir -p /data/elasticsearch;sudo chown elasticsearch: /data/elasticsearch;sudo chmod 755 /data/elasticsearch;sudo mv /etc/elasticsearch/elasticsearch{.yml,.yml.bk};sudo vi /etc/elasticsearch/elasticsearch.yml;sudo mkdir /usr/lib/systemd/system/elasticsearch.service.d;sudo chmod 755 /usr/lib/systemd/system/elasticsearch.service.d;sudo vi /usr/lib/systemd/system/elasticsearch.service.d/override.conf;sudo chmod 644 /usr/lib/systemd/system/elasticsearch.service.d/override.conf;sudo systemctl daemon-reload;sudo vi /etc/elasticsearch/jvm.options.d/jvm_override.conf;sudo firewall-cmd --add-port={9200,9300}/tcp --permanent;sudo firewall-cmd --reload;sudo firewall-cmd --list-all
```

# Step 2 - Edit the elasticsearch.yml
- `sudo vi /etc/elasticsearch/elasticsearch.yml`
- make these changes (paste it in and change line 2): 
  - es-node-# (0=0, 1=1, 2=2)
  - `:set nu`
  - `:%d`
```
cluster.name: nsm-cluster
node.name: es-node-0
path.data: /data/elasticsearch
path.logs: /var/log/elasticsearch
bootstrap.memory_lock: true
network.host: _site:ipv4_
http.port: 9200
discovery.seed_hosts: ["elastic0", "elastic1", "elastic2"]
cluster.initial_master_nodes: ["es-node-0", "es-node-1", "es-node-2"]
```

# Step 3 - Edit override.conf and change perms
- `sudo vi /usr/lib/systemd/system/elasticsearch.service.d/override.conf`
- make these changes: 
```
[Service]
LimitMEMLOCK=infinity
```


# Step 4 - Edit jvm_override.conf
- `sudo vi /etc/elasticsearch/jvm.options.d/jvm_override.conf`
- make these changes: 
```
-Xms2g
-Xmx2g
```



# Step 5 - Start Elastic Search on all containers and Verify the cluster with Elastic API command
```
sudo systemctl enable elasticsearch --now;curl elastic0:9200/_cat/nodes?v;curl elastic0:9200
```


































# Kibana Kibana Kibana Kibana

# Overview

# Step 1 - SSH onto Kibana
- `ssh kibana`
```
easy pastable - enable seperate!

sudo yum install kibana -y;sudo mv /etc/kibana/kibana{.yml,.yml.bk};sudo vi /etc/kibana/kibana.yml;sudo yum install nginx -y;sudo vi /etc/nginx/conf.d/kibana.conf;sudo vi /etc/nginx/nginx.conf;sudo firewall-cmd --add-port=80/tcp --permanent;sudo firewall-cmd --reload;sudo firewall-cmd --list-all
```


# Step 2 - Edit Kibana.yml
- `sudo vi /etc/kibana/kibana.yml`
- make these edits: 
```
server.port: 5601
server.host: localhost
server.name: kibana
elasticsearch.hosts: ["http://elastic0:9200","http://elastic1:9200","http://elastic2:9200"]
```


# Step 3 - edit kibana.conf inside nginx directory
- `sudo vi /etc/nginx/conf.d/kibana.conf`
- past this inside: 
```
server {
  listen 80;
  server_name kibana;
  proxy_max_temp_file_size 0;

  location / {
    proxy_pass http://127.0.0.1:5601/;

    proxy_redirect off;
    proxy_buffering off;

    proxy_http_version 1.1;
    proxy_set_header Connection "Keep-Alive";
    proxy_set_header Proxy-Connection "Keep-Alive";

  }
}
```

# Step 4 - Edit the nginx.conf in the nginx directory
- `sudo vi /etc/nginx/nginx.conf`
- comment these lines out to prevent conflicts
  - `:set nu`
```
39 #
40 #
41 #
```



# Step 5 - Enable and start nginx
`sudo systemctl enable nginx --now`

# Step 6 - Enable and start kibana
`sudo systemctl enable kibana --now`

# Step 7 - Go to browser and navigate to Kibana
- http://kibana

# Step 8 - Explore kibana
- Checkout Management > Stack Managemetn 
- Checkout management > dev tools

# Step 9 - Pull down ECS mappings from fileshare, unzip, and install JQ and Add the mapping templates to kibana
```
curl -LO https://repo/fileshare/kibana/ecskibana.tar.gz;tar -zxvf ecskibana.tar.gz;sudo yum install jq -y;cd ecskibana;./import-index-templates.sh http://elastic0:9200
```

# Step 10 - Verify the mapping templates were added to kibana using the API
- navigate to devtools (management> dev tools)
- `GET _cat/templates?v`
- look for `ecs_zeek` and `ecs_suricata`
- cannot create an index pattern for FSF until logstash is configured








































# Logstash Logstash Logstash Logstash

# Step 1 - Get onto Pipleine
- `ssh pipeline0`

# Step 1.5 - Initial steps
- quick pastable: 
```
sudo yum install logstash -y;curl -LO https://repo/fileshare/logstash/logstash.tar.gz;sudo tar -zxvf logstash.tar.gz -C /etc/logstash;sudo chown logstash: /etc/logstash/conf.d/*;sudo chmod -R 744 /etc/logstash/conf.d/ruby/;cd /etc/logstash/conf.d/;sudo vi logstash-100-input-kafka-zeek.conf;sudo vi logstash-100-input-kafka-fsf.conf;sudo vi logstash-100-input-kafka-suricata.conf;sudo vi logstash-9999-output-elasticsearch.conf;sudo -u logstash /usr/share/logstash/bin/logstash -t --path.settings /etc/logstash/
```


# Step 2 - Edit the input files
- `:%s/127.0.0.1:9092/pipeline0:9092,pipeline1:9092,pipeline2:9092/g`
  - use this shortcut
- `sudo vi logstash-100-input-kafka-zeek.conf`
- `sudo vi logstash-100-input-kafka-fsf.conf`
- `sudo vi logstash-100-input-kafka-suricata.conf`



# Step 3 - Edit the output file
- `:%s/"127.0.0.1"/"elastic0","elastic1","elastic2"/g`
  - use this shortcut (6 subs on 6 lines)
- `sudo vi logstash-9999-output-elasticsearch.conf`



# Step 4 - Start and enable logstash on all pipelines
- `sudo systemctl enable logstash --now`

# Step 5 - On pipeline0 check plain log for start up errors
- `sudo tail -f /var/log/logstash/logstash-plain.log`
- logstash can take up to 5 min to start

# Step 6 - Navigate on kibana to Stack management to create index patterns
- Management > stack management > index patterns > create index pattern
- make `ecs-zeek-*`, `ecs-suricata-*`, and `fsf-*`
- choose @timestamp 

# Step 7 - Go to discover in kibana and see events!
