# Overview


# Step 1 - SSH into the sensor to configure FSF
- `ssh sensor`

# Step 2 - Install FSF
- `sudo yum list fsf`
- `sudo yum install fsf -y`

# Step 3 - Edit server config.py file
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

# Step 4 - Make /data/fsf and /data/fsf/archive
- `sudo mkdir -p /data/fsf/archive` (-p makes dir if it doesnt exist)

# Step 5 - Change permissions
- `sudo chown -R fsf: /data/fsf`

# Step 6 - Edit client config.py to match server config.py
- `sudo vi /opt/fsf/fsf-client/conf/config.py`
- make these edits: 
```
9 SERVER_CONFIG = {'IP_ADDRESS' : ['localhost'], 'PORT' : 5800}
```


# Step 7 - Start FSF
- fsf is running as service so systemctl is good
- `sudo systemctl enable --now fsf`
- `sudo systemctl status fsf`


# Step 8 - Test FSF 
- `cd` 
  - go to home directory
- `/opt/fsf/fsf-client/fsf_client.py --full interface.sh | jq` 
  - Manually use fsf to scan interface.sh
  - rockout.log gets created when you manually scan too
- `ll /data/fsf`

# Step 9 - Checkout the yara rule files
- `ll /var/lib/yara-rules`

# Step 10 - Make FSF work in conjunction with Zeek 
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
# Step 11 - Redeploy Zeek for changes to take effect
- `sudo -u zeek zeekctl deploy`
- `sudo -u zeek zeekctl status`

# Step 12 - Lets test zeek carves and FSF scans
- `curl google.com`
- `ll /data/zeek`
  - should see extract_files directory
- `tail /data/fsf/rockout.log | jq`
  - can see "extract" in the file name for zeek scanned files
  - can see "interface.sh" in the file name for that manually scanned file


# Summary







# Extras
- `curl -LO https://repo/fileshare/vim-cheatsheet.pdf` 
- `/opt/fsf/fsf-client/fsf_client.py --full vim-cheatshet.pdf`
- FSF will scan the pdf and examines all the objects embedded into the pdf (ex: jpg pictures)
- can see it matches a signature on PDFs

# Step 14 - 

# Step 15 - 

# Step 16 - 

# Step 17 - 

# Step 18 - 

# Step 19 - 

# Step 20 - 

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

# Step - 

