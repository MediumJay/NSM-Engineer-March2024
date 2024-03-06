# Overview 
- Documentation: https://docs.suricata.io/en/latest/


# Step 1 - Get on the Sensor
- `ssh sensor` 

# Step 2 - Install Suricata
- `sudo yum install suricata -y`

# Step 3 - Configure the Suricata.yaml file
- `sudo vi /etc/suricata/suricata.yaml`
- This config is HUGE (version 5.0.1 for class)
- Important locations by line number (`set nu` in vi): 
  - line 12: vars section for variables
  - line 56: default logging directory
  - line 75: fast.log for summary/overview of events
  - line 579: af-packet interface config
  - line 582: threads usage ("#threads: auto" by default)
  - line 980: run-as section determines who suri runs as
  - line 1434: CPU pinning enable section
  - line 1452: CPU pinning core section
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
1452 cpu: [0-2] (pinning cores 0 through 2)
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

# Step 5 - Import rules into Suricata
- we moved the emerging rules file 
to the fileshare
- we want to download this to import into Suricata
  - `sudo suricata-update add-source emergingthreats https://repo/fileshare/emerging.rules.tar.gz`
- Once downloaded we need to do the import 
  - `sudo suricata-update` 
- Rules will be loaded and then tested with -T

# Step 6 - Checkout Suricata Rules
- `sudo cat /var/lib/suricata/suricata.rules` 

# Step 7 - Create the data directory for Suricata
- `sudo mkdir /data/suricata`

# Step 8 - Change permissions of the /data/suricata directory
- `sudo chown -R suricata: /data/suricata`
  - suricata: will change user and group to suricata since they are identical 


# Step 9 - Start and enable Suricata
- `sudo systemctl enable --now suricata` to start+enable

# Step 10 - Checkout logs with Journalctl (for errors)
- `journalctl -xeu suricata`

# Step 11 - Generate some traffic and Look at eve.json
- `ping 8.8.8.8; curl google.com`
- `sudo cat /data/suricata/eve.json`

# Summary
- We used yum to install suricata on the sensor
- We made edits to the suricata.yaml file to disable unnecessary logs, enable af-packet, and pin specific CPUs
- We made edits to the /etc/sysconfig/suricata service file to configure af-packet and the collection interface for suricata when started by systemctl  
- We downloaded the emerging threats rule package from our repo and imported it into Suricata
- We created the data/suricata directory and altered its permissions to be owned by suricata user/group
- We started/enabled suricata and created test traffic with pings/curls
- We examined eve.json to verify rules were being triggered. 










# Other student's Helpful Notes
## Table for changes to suricata.yaml
| line | field | change |
| --- | --- | --- |
| line 56  | default-log-dir: | /data/suricata/ |
| line 60 |   global stats config:  | enabled: no |
| line 76 |  fast.log: | enabled: no |
| line 404 | stats.log | enabled: no |
| line 557 | console | enabled: no |
| line 580 | af-packet | interface: eth1 |
| line 582 | af-packet | (uncomment ) threads: 3 |
| line 981 | run as | (uncomment) user: suricata, group: suricata |
| line 1434 | threading | set-cpu-affinity: yes |
| line 1452 | worker-cpu-set | cpu [ "0-2"] |
| line 1459 | prio | medium 1, high 2, default "high" |
| line 1500 | Rule performance log | enabled: no |
| line 1516 | keywords | enabled: no |
| line 1521 | prefilter | enabled: no |
| line 1527 | rulegroups | enabled: no |
| line 1536 | profiling | enabled: no |