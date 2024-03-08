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

- `sudo vi /etc/sysconfig/network-scripts/ifcfg-eth0`
  - make it look like this: 
```
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
HOSTNAME=<containername>
NM_CONTROLLED=no
IPADDR=
GATEWAY=
PREFEX=
```

# Step 4 - Restart the network

- `sudo systemctl restart network`

# Step 5 - Repeat steps 3 through 4 for each container

# Step 6 - Edit the /etc/hosts file on "Laptop" to add the container hostnames + usernames
- `sudo vi /etc/hosts/`
- add the following: 
```
 Host repo
    HostName repo
    User elastic
  Host sensor
    HostName sensor
    User elastic
  Host elastic0
    HostName elastic0
    User elastic
  Host elastic1
    HostName elastic1
    User elastic
  Host elastic2
    HostName elastic2
    User elastic
  Host pipeline0
    HostName pipeline0
    User elastic
  Host pipeline1
    HostName pipeline1
    User elastic
  Host pipeline2
    HostName pipeline2
    User elastic
  Host kibana
    HostName kibana
    User elastic
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


# Step 18 - Get onto the Sensor to configure Stenographer
- `ssh sensor`

# Step 19 - Verify sensor is pulling from local-rocknsm repo
- `sudo yum list stenographer`

# Step 20 - Install Stenographer
- `sudo yum install stenographer -y`

# Step 21 - Begin configuring Stenographer by editing the config file 
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


# Step 23 - Run stenographer script file to generate keys for steno user and group
- These keys are needed for...
- `sudo stenokeys.sh stenographer stenographer`
  - specify the user (stenographer) and the group (stenographer)
- verify keys were installed
  - `ll /etc/stenographer/certs/`

# Step 24 - Create the directories that the stenographer config file is referencing
- `sudo mkdir -p /data/stenographer/{packets,index}`
  - {} just makes 2 directories at the same time
- notice root owns these two directories

# Step 25 - Give Stenographer user and group ownership to packets and index directories
- `sudo chown -R stenographer:stenographer /data/stenographer`

# Step 26 - Start and Enable Stenographer service
- `sudo systemctl start stenographer`
- `^start^enable` - linux kungfu to replace start with enable

# Step 27 - Verify Stenographer is capturing packets
- On sensor `ping 8.8.8.8 -c 5`
- Carve it `sudo stenoread 'host 8.8.8.8' -nn`
- Traffic can take a while to carve. 
- Rerun stenoread command to eventually see the traffic. 
- 5+ minutes and a config might be messed up. 


# Step 28 - Verify another way
- check our directories to see if they're being written
- `ll /data/stenographer/packets`
- `ll /data/stenographer/index`

# SURICATA SURICATA SURICATA SURICATA SURICATA SURICATA 

# Step 29 - Get on the Sensor
- `ssh sensor` 

# Step 30 - Install Suricata
- `sudo yum install suricata -y`

# Step 31 - Configure the Suricata.yaml file
- `sudo vi /etc/suricata/suricata.yaml`
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

# Step 32 - Configure /etc/sysconfig/suricata
- This file plugs in the options into the service file so we can use af-packet and run it as user/group suricata
  - the service file is what configures services when managed by systemctl
- `sudo vi /etc/sysconfig/suricata`
- make the following change: 
```
OPTIONS="--af-packet=eth1 --user suricata --group suricata" 
```

# Step 33 - Import rules into Suricata
- we moved the emerging rules file 
to the fileshare
- we want to download this to import into Suricata
  - `sudo suricata-update add-source emergingthreats https://repo/fileshare/emerging.rules.tar.gz`
- Once downloaded we need to do the import 
  - `sudo suricata-update` 
- Rules will be loaded and then suricata will be tested with -T

# Step 34 - Create the data directory for Suricata
- `sudo mkdir /data/suricata`

# Step 35 - Change permissions of the /data/suricata directory
- `sudo chown -R suricata: /data/suricata`
  - suricata: will change user and group to suricata since they are identical 


# Step 36 - Start and enable Suricata
- `sudo systemctl enable --now suricata` to start+enable

# Step 37 - Checkout logs with Journalctl (for errors)
- `journalctl -xeu suricata`

# Step 38 - Generate some traffic and Look at eve.json
- `ping 8.8.8.8; curl google.com`
- `sudo cat /data/suricata/eve.json`


# ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK ZEEK 

# Step 39 - SSH into the sensor to begin configuring Zeek
- `ssh sensor`

# Step 40 - Install 3 Necessary Packages 

- `sudo yum list zeek` 
  - verify it is local repo
- `sudo yum install zeek zeek-plugin-kafka zeek-plugin-af_packet -y`

# Step 41 - Edit zeekctl.cfg file
- `sudo vi /etc/zeek/zeekctl.cfg`
- Insert the following changes:
  - `:set nu`
```
67 LogDir = /data/zeek/ (location of archived logs)
68 lb_custom.InterfacePrefix=af_packet:: (tells zeek to use afpacket)
69 /n
```

# Step 42 - Edit node.cfg
- `sudo vi /etc/zeek/node.cfg`
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


# Step 43 - Make and cd into Zeek Scripts Directory
- `sudo mkdir /usr/share/zeek/site/scripts`
  - where we are gonna place all our zeek scripts at
- `cd /usr/share/zeek/site/scripts`
- 

# Step 44 - Curl down zeek scripts from local repo
- 7 scripts in repo, we will curl the first 6 (not local.zeek cuz already on the sensor)
- `sudo curl -LO https://repo/fileshare/zeek/afpacket.zeek`
- `sudo curl -LO https://repo/fileshare/zeek/extension.zeek`
- `sudo curl -LO https://repo/fileshare/zeek/extract-files.zeek`
- `sudo curl -LO https://repo/fileshare/zeek/fsf.zeek`
- `sudo curl -LO https://repo/fileshare/zeek/json.zeek`
- `sudo curl -LO https://repo/fileshare/zeek/kafka.zeek`
```
mass curl: 
sudo curl -LO https://repo/fileshare/zeek/afpacket.zeek;sudo curl -LO https://repo/fileshare/zeek/extension.zeek;sudo curl -LO https://repo/fileshare/zeek/extract-files.zeek;sudo curl -LO https://repo/fileshare/zeek/fsf.zeek;sudo curl -LO https://repo/fileshare/zeek/json.zeek;sudo curl -LO https://repo/fileshare/zeek/kafka.zeek
```



# Step 45 - Set zeek to utilize our custom zeek scripts
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


# Step 46 - Create /data/zeek adn Change ownership of directories & binaries to Zeek
- `sudo mkdir /data/zeek`
- `sudo chown -R zeek: /etc/zeek`
- `sudo chown -R zeek: /var/spool/zeek`
- `sudo chown -R zeek: /data/zeek`
- `sudo chown -R zeek: /usr/share/zeek`
- `sudo chown -R zeek: /usr/bin/zeek`
- `sudo chown -R zeek: /usr/bin/capstats`
```
mass chown: 

sudo chown -R zeek: /etc/zeek;sudo chown -R zeek: /var/spool/zeek;sudo chown -R zeek: /data/zeek;sudo chown -R zeek: /usr/share/zeek;sudo chown -R zeek: /usr/bin/zeek;sudo chown -R zeek: /usr/bin/capstats
```

# Step 47 - Set capabilities for zeek user to look at raw network traffic and administer interface
- `sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/zeek`
- `sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/capstats`
- quick pastable: 
```
sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/zeek;sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/capstats
```

# Step 48 - Check for persistence to capabilities changes
- `sudo getcap /usr/bin/zeek`
- `sudo getcap /usr/bin/capstats`

# Step 49 - Deploy zeek as zeek user
- `sudo -u zeek zeekctl deploy`
  - `sudo -u zeek zeekctl diag` for error checking


# Step 50 - Test zeek by checking out a log using JQ
- `curl google.com`
- `tail /data/zeek/current/http.log | jq` 


# FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF FSF 

# Step 51 - SSH into the sensor to configure FSF
- `ssh sensor`

# Step 52 - Install FSF
- `sudo yum list fsf`
- `sudo yum install fsf -y`

# Step 53 - Edit server config.py file
- `sudo vi /opt/fsf/fsf-server/conf/config.py`
- make these edits: 
```
9 'LOG_PATH' : '/data/fsf',
10 'YARA_PATH' : '/var/lib/yara-rules/rules.yara', 
11 'PID_PATH' : '/run/fsf/fsf.pid', 
12 'EXPORT_PATH' : '/data/fsf/archive',
15 'ACTIVE_LOGGING_MODULES' : ['rockout'],
18 SERVER_CONFIG = {'IP_ADDRESS' : "localhost", 'PORT' : 5800 }
```

# Step 54 - Make /data/fsf and /data/fsf/archive
- `sudo mkdir -p /data/fsf/archive` (-p makes dir if it doesnt exist)

# Step 55 - Change permissions
- `sudo chown -R fsf: /data/fsf`

# Step 56 - Edit client config.py to match server config.py
- `sudo vi /opt/fsf/fsf-client/conf/config.py`
- make these edits: 
```
9 SERVER_CONFIG = {'IP_ADDRESS' : ['localhost'], 'PORT' : 5800}
```


# Step 57 - Start FSF
- fsf is running as service so systemctl is good
- `sudo systemctl enable --now fsf`
- `sudo systemctl status fsf`


# Step 58 - Test FSF 
- `cd` 
  - go to home directory
- `/opt/fsf/fsf-client/fsf_client.py --full interface.sh | jq` 
  - Manually use fsf to scan interface.sh
  - rockout.log gets created when you manually scan too
- `ll /data/fsf`

# Step 59 - Checkout the yara rule files
- `ll /var/lib/yara-rules`

# Step 60 - Make FSF work in conjunction with Zeek 
- `cd /usr/share/zeek/site/scripts`
- we already loaded 3 of the scripts
- we will load fsf, extract-files, and kafka
- `sudo vi /usr/share/zeek/site/local.zeek`
- make these edits: 
  - shift + g
```
107 @load scripts/extract-files
108 @load scripts/fsf
109
110 redef ignore_checksums = T; 
```
# Step 61 - Redeploy Zeek for changes to take effect
- `sudo -u zeek zeekctl deploy`
- `sudo -u zeek zeekctl status`

# Step 62 - Lets test zeek carves and FSF scans
- `curl google.com`
- `ll /data/zeek`
  - should see extract_files directory
- `tail /data/fsf/rockout.log | jq`
  - can see "extract" in the file name for zeek scanned files
  - can see "interface.sh" in the file name for that manually scanned file



  # ZOOKEEPER ZOOKEEPER ZOOKEEPER ZOOKEEPER ZOOKEEPER ZOOKEEPER ZOOKEEPER ZOOKEEPER ZOOKEEPER

# Step 63 - Get onto pipeline containers to install Kafka and Zookeeper
- `ssh pipeline0`

# Step 64 - Install Kafka and Zookeeper
- `sudo yum install kafka zookeeper -y`

# Step 65 - Configure Zookeeper first so its ready to manage - create data directory
- `sudo mkdir -p /data/zookeeper`

# Step 66 - Change Permissions to directory
- `sudo chown -R zookeeper: /data/zookeeper/`

# Step 67 - Configure zoo.cfg
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

# Step 68 - Create an ID for each zookeeper instance
- `sudo touch /data/zookeeper/myid`

# Step 69 - Give zookeeper permissions to the id
- `sudo chown -R zookeeper: /data/zookeeper/myid`

# Step 70 - Enter the respective IDs into the myid file
instance | serverID
pipeline0 | 1
pipeline1 | 2
pipelien2 | 3

- `echo '#' | sudo tee /data/zookeeper/myid` 

# Step 71 - Open the firewall
- `sudo firewall-cmd --add-port={2181,2888,3888}/tcp --permanent`
- `sudo firewall-cmd --reload`

# Step 72 - Repeat steps 64 through 71 on the other pipelines
- easy pastable just make sure to replace the # with the respective serverID:

```
sudo yum install kafka zookeeper -y;sudo mkdir -p /data/zookeeper;sudo chown -R zookeeper: /data/zookeeper/;sudo vi /etc/zookeeper/zoo.cfg;echo '#' | sudo tee /data/zookeeper/myid;sudo firewall-cmd --add-port={2181,2888,3888}/tcp --permanent;sudo firewall-cmd --reload;sudo firewall-cmd --list-all
```


# Step 73 - Enable and start zookeeper on all pipeline containers
-`sudo systemctl enable --now zookeeper`


# Step 74 - On the Ubuntu Laptop issue the stats command to each of the Zookeeper clients on client port
- this script verfies communications (leader and 2 followers)
```
for host in pipeline{0..2}; do (echo "stats" | nc $host 2181 -q 2); done
```
- looking for "mode: " it should have 2 followers and 1 leader
- order doesnt matter






 # KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA KAFKA

# Step 75 - Get onto the Pipelines and Configure Kafka by making /data/kafka 
- `ssh pipeline0`
- `sudo mkdir -p /data/kafka`
- `sudo chown -R kafka: /data/kafka`

# Step 76 - Create a copy of our server.properties before we edit it 
- `sudo cp /etc/kafka/server{.properties,.properties.bk}`


# Step 77 - Edit server.properties
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

# Step 78 - Open the firewall
- `sudo firewall-cmd --add-port=9092/tcp --permanent; sudo firewall-cmd --reload`

# Step 79 - Repeat steps 75 to 78 for the other pipelines
- Easy pastable: 

```
sudo mkdir -p /data/kafka;sudo chown -R kafka: /data/kafka;sudo cp /etc/kafka/server{.properties,.properties.bk};sudo vi /etc/kafka/server.properties;sudo firewall-cmd --add-port=9092/tcp --permanent;sudo firewall-cmd --reload;sudo firewall-cmd --list-all
```

# Step 80 - Start all 3 kafkas but dont enable boot start
- `sudo systemctl start kafka`

# Step 81 - Create a test topic to verify Kafka cluster on pipeline0
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



# Step 82 - Create zeek-raw topic and Move Zeek data over to kafka
- create zeek-raw topic so there is a place for the zeek data to go
  - `sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --create --topic zeek-raw --partitions 3 --replication-factor 3`
- checkout the topic
  - `sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --describe --topic zeek-raw`

# Step 83 - Go back to your sensor
- `ssh sensor`

# Step 84 - Add the kafka plugin into zeek via kafka.zeek script
- `sudo vi /usr/share/zeek/site/scripts/kafka.zeek`
- make these edits to kafka.zeek:
  - `:set nu` 
```
6 redef Kafka::kafka_conf = table(
7 ["metadata.broker.list"] = "pipeline0:9092,pipeline1:9092,pipeline2:9092"); 
```
# Step 85 - Add the kafka script to local.zeek 
- `sudo vi /usr/share/zeek/site/local.zeek`
- make these changes at the bottom: 
  - shift+g
```
109 @load scripts/kafka
```


# Step 86 - redeploy zeek
- `sudo -u zeek zeekctl deploy`
- `sudo -u zeek zeekctl status`

# Step 87 - Verify zeek traffic is making it to Kafka
- `ssh pipeline0`
- `sudo /usr/share/kafka/bin/kafka-console-consumer.sh --bootstrap-server pipeline0:9092 --topic zeek-raw`
- generate traffic with a curl and should see it output from the previous command


 
