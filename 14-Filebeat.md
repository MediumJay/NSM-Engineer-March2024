Next we will setup Filebeat to get Suricata and FSF to kafka (Zeek did so through plugin)

# Step 1 - SSH into the sensor

# Step 2 - Install FIlebeat
- `sudo yum install filebeat -y`

# Step 3 - Move some yaml files to back them up 
- `sudo mv /etc/filebeat/filebeat{.yml,.yml.bk}`

# Step 4 - cd into filebeat directory, curl the filebeat yml and then edit it to reflect the pipelines
- `cd /etc/filebeat/`
- `sudo curl -LO https://repo/fileshare/filebeat/filebeat.yml`
- `sudo vi filebeat.yml`
  - make these changes: 
  - `:set nu`
```
34 hosts: ["pipeline0:9092","pipeline1:9092","pipeline2:9092"]
```


# Step 5 - ssh into pipeline0 and create fsf-raw and suricata-raw topics with kafka-topics.sh
- `ssh pipeline0`
- create the new topics
  - `sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --create --topic fsf-raw --partitions 3 --replication-factor 3`
  - `sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --create --topic suricata-raw --partitions 3 --replication-factor 3`
- `sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --list` to check topics 

# Step 6 - verify your topic information
- `sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --describe --topic suricata-raw`
- `sudo /usr/share/kafka/bin/kafka-topics.sh --bootstrap-server pipeline0:9092 --describe --topic fsf-raw`

# Step 7 - Start filebeat on the Sensor
- `sudo systemctl start filebeat`
- `^start^status`

# Step 8 - Valid logs on pipeline0 by generating traffic from another container (sensor is fine)
- `sudo /usr/share/kafka/bin/kafka-console-consumer.sh --bootstrap-server pipeline0:9092 --topic suricata-raw`
- `sudo /usr/share/kafka/bin/kafka-console-consumer.sh --bootstrap-server pipeline0:9092 --topic fsf-raw`
- Note: you may need to restart suricata first. It can take some time

# Step 9 - 

# Step 10 - 


