
# Overview 
- Googles 80/20 project by Graeme Connell
- Written by Graeme Connell 
- Automatically manages Disk space
- Written in Go
- Github for Steno: https://github.com/google/stenographer/blob/master/README.md

# Step 1 - Get onto the Sensor
- `ssh sensor`

# Step 2 - Verify sensor is pulling from local-rocknsm repo
- `sudo yum list stenographer`

# Step 3 - Install Stenographer
- `sudo yum install stenographer -y`

# Step 4 - Begin configuring Stenographer
- Examine contents of /etc/stenographer
  - `ll` shows certs directory and config file



# Step 4 - Edit the config file 
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


# Step 5 - Run stenographer script file to generate keys for steno user and group
- These keys are needed for...
- `sudo stenokeys.sh stenographer stenographer`
  - specify the user (stenographer) and the group (stenographer)
- verify keys were installed
  - `ll /etc/stenographer/certs/`

# Step 6 - Create the directories that the stenographer config file is referencing
- `sudo mkdir -p /data/stenographer/{packets,index}`
  - {} just makes 2 directories at the same time
- notice root owns these two directories

# Step 7 - Give Stenographer user and group ownership to packets and index directories
- `sudo chown -R stenographer:stenographer /data/stenographer`

# Step 8 - Start and Enable Stenographer service
- `sudo systemctl start stenographer`
- `^start^enable` - linux kungfu to replace start with enable

# Step 9 - Verify Stenographer is capturing packets
- On sensor `ping 8.8.8.8 -c 5`
- Carve it `sudo stenoread 'host 8.8.8.8' -nn`
- Traffic can take a while to carve. 
- Rerun stenoread command to eventually see the traffic. 
- 5+ minutes and a config might be messed up. 


# Step 10 - Verify another way
- check our directories to see if they're being written
- `ll /data/stenographer/packets`
- `ll /data/stenographer/index`


# Summary
- We used yum to install Stenographer from the local rocknsm repo
- We edited the stenographer config file to point to the packet and index storage directories (not yet created), configured the collection interface, and up-ed the limit on the free disk percentage. 
- We generated keys for stenographer via the stenokeys.sh script
- We created the packets and index directories in /data/stenographer
- We gave the stenographer group and user permission over the index and packets directories
- Finally, we started, enabled, and tested stenographer. 


