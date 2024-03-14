# Overview

# Step 1 - Get onto Pipleine
- `ssh pipeline0`

# Step 2 - Install logstash
- `sudo yum install logstash -y`


# Step 3 - Curl the logstash XXX
- `curl -LO https://repo/fileshare/logstash/logstash.tar.gz`

# Step 4 - unzip the tar
- `sudo tar -zxvf logstash.tar.gz -C /etc/logstash`

# Step 5 - change perms and cd into conf.d
- `sudo chown logstash: /etc/logstash/conf.d/*`
- `sudo chmod -R 744 /etc/logstash/conf.d/ruby/`
- `cd /etc/logstash/conf.d/`
  - in here are where the inputs, filters, and outputs are for logstash 

# Step 6 - Edit the input files
- `:%s/127.0.0.1:9092/pipeline0:9092,pipeline1:9092,pipeline2:9092/g`
  - use this shortcut
- `sudo vi logstash-100-input-zeek.conf`
- `sudo vi logstash-100-input-kafka-fsf.conf`
- `sudo vi logstash-100-input-kafka-suricata.conf`



# Step 7 - Edit the output file
- `:%s/"127.0.0.1"/"elastic0","elastic1","elastic2"/g`
  - use this shortcut (6 subs on 6 lines)
- `sudo vi logstash-9999-output-elasticsearch.conf`


# Step 8 - Have logstash verify configs
- `sudo -u logstash /usr/share/logstash/bin/logstash -t --path.settings /etc/logstash/`
  - MAKE SURE TO HAVE -u logstash FOR YOUR USER


# Step 9 - Repeat steps 2-8 on pipeline1&2

# Step 10 - Start and enable logstash on all pipelines
- `sudo systemctl enable logstash --now`

# Step 11 - On pipeline0 check plain log for start up errors
- `sudo tail -f /var/log/logstash/logstash-plain.log`
- logstash can take up to 5 min to start

# Step 12 - Navigate on kibana to Stack management to create index patterns
- Management > stack management > index patterns > create index pattern
- make `ecs-zeek-*`, `ecs-suricata-*`, and `fsf-*`
- choose @timestamp 

# Step 13 - Go to discover in kibana and see events!

# Step - 

# Step - 

# Step - 

# Step - 

# Step - 

# Step - 

# Step - 

# Step - 

# Step - 

# Step - 

# Step - 

# Step - 