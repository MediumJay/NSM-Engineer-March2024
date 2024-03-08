# Overview
- Version for this class: 3.0.1
- https://try.zeek.org
- 


# Step 1 - SSH into the sensor to begin configuring Zeek
- `ssh sensor`

# Step 2 - Install 3 Necessary Packages 
- zeek and 2 plugins 
  - one for afpacket and one for zeek logs over network straight to kafka
- `sudo yum list zeek` - verify it is local repo
- `sudo yum install zeek zeek-plugin-kafka zeek-plugin-af_packet -y`
  - this repo installed configs to /etc/zeek
  - default install location for zeek is to /opt/zeek

# Step 3 - Examine /etc/zeek/networks.cfg
- `cat /etc/zeek/networks.cfg`
- this is where we specify our network ranges

# Step 4 - Edit zeekctl.cfg file
- `sudo vi /etc/zeek/zeekctl.cfg`
- important sections: 
```
4 Mail options
26 Logging options
48 other options 
30 LogRotationInterval = 3600 (seconds till logs get rolled over into archive)
63 SitePolicyScripts = local.zeek (place where our custom scripts go [local.zeek])
```
- Insert the following changes:
  - `:set nu`
```
67 LogDir = /data/zeek/ (location of archived logs)
68 lb_custom.InterfacePrefix=af_packet:: (tells zeek to use afpacket)
```

# Step 5 - Edit node.cfg
- `sudo vi /etc/zeek/node.cfg`
- zeek can be set as standalone or clustered; we want clustered
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
36 env_vars=fanout_id=77 (for scaling processing across threads; 77 is arbitrary but what we are using; groups all processes to single network socket with afpacket)

delete lines 34-37 for worker 2
```
- LOGGER is responsible for collecting and storing log data
- MANAGER coordinates the activity of the worker nodes
- PROXY is the intermediary between workers and managers (for comms)
- WORKER is responsible for analyzing the traffic and generating the logs/events


# Step 6 - Make and cd into Zeek Scripts Directory
- `sudo mkdir /usr/share/zeek/site/scripts`
  - where we are gonna place all our zeek scripts at
- `cd /usr/share/zeek/site/scripts`
- 

# Step 7 - Curl down zeek scripts from local repo
- 7 scripts in repo, we will curl the first 6 (not local.zeek cuz already on the sensor)
- `sudo curl -LO https://repo/fileshare/zeek/afpacket.zeek`
  - this one tells zeek to use afpacket with fanoutid we set
- `sudo curl -LO https://repo/fileshare/zeek/extension.zeek`
  - this one adds a variable that tags logs with what sensor it came from as well as tags the zeek worker that logged the event
- `sudo curl -LO https://repo/fileshare/zeek/extract-files.zeek`
  - this one tells zeek were to extract for fsf
- `sudo curl -LO https://repo/fileshare/zeek/fsf.zeek`
  - this one use fsf framework
- `sudo curl -LO https://repo/fileshare/zeek/json.zeek`
  - this one converts our zeek logs into JSON format
- `sudo curl -LO https://repo/fileshare/zeek/kafka.zeek`
  - this one ...
```
mass curl: 
sudo curl -LO https://repo/fileshare/zeek/afpacket.zeek;sudo curl -LO https://repo/fileshare/zeek/extension.zeek;sudo curl -LO https://repo/fileshare/zeek/extract-files.zeek;sudo curl -LO https://repo/fileshare/zeek/fsf.zeek;sudo curl -LO https://repo/fileshare/zeek/json.zeek;sudo curl -LO https://repo/fileshare/zeek/kafka.zeek
```



# Step 8 - Set zeek to utilize our custom zeek scripts
- edit the local.zeek file
- `sudo vi /usr/share/zeek/site/local.zeek`
- make these edits at the bottom: 
  - `:set nu`
  - shift+g+o
  - .zeek extension not needed in edits but can put if desired
```
104 @load scripts/json
105 @load scripts/afpacket
106 @load scripts/extension
107 /n
108 redef ignore_checksums = T; (ignore checksums)
```


# Step 9 - Create /data/zeek adn Change ownership of directories & binaries to Zeek
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

# Step 10 - Set capabilities for zeek user to look at raw network traffic and administer interface
- `sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/zeek`
- `sudo /sbin/setcap cap_net_raw,cap_net_admin=eip /usr/bin/capstats`


# Step 11 - Check for persistence to capabilities changes
- `sudo getcap /usr/bin/zeek`
- `sudo getcap /usr/bin/capstats`

# Step 12 - Deploy zeek as zeek user
- `sudo -u zeek zeekctl deploy`
  - `sudo -u zeek zeekctl diag` for error checking
- zeekctl has more options than systemd for us

# Step 13 - checkout a log using JQ
- `tail /data/zeek/current/http.log | jq` 
- JQ is a JSON processor, we used a json processing plugin so we cannot use zeekcut because the format is in json (for elastic/splunk) 

# Summary

# Step - 
