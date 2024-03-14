# Overview 



# Step 1 - SSH onto elastic0 
- `ssh elastic0` 

# Step 2 - Install Elastic Search
- `sudo yum install elasticsearch -y`

# Step 3 - Create data directory and change permissions
- `sudo mkdir -p /data/elasticsearch`
- `sudo chown elasticsearch: /data/elasticsearch`
- `sudo chmod 755 /data/elasticsearch`
- `sudo mv /etc/elasticsearch/elasticsearch{.yml,.yml.bk}`
- Pastable: 
```
sudo mkdir -p /data/elasticsearch;sudo chown elasticsearch: /data/elasticsearch;sudo chmod 755 /data/elasticsearch;sudo mv /etc/elasticsearch/elasticsearch{.yml,.yml.bk}
```
# Step 4 - Edit the elasticsearch.yml
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


# Step 5 - Create directory for elasticsearch.service.d and change permissions
- `sudo mkdir /usr/lib/systemd/system/elasticsearch.service.d`
- `sudo chmod 755 /usr/lib/systemd/system/elasticsearch.service.d`
- pastable: 
```
sudo mkdir /usr/lib/systemd/system/elasticsearch.service.d;sudo chmod 755 /usr/lib/systemd/system/elasticsearch.service.d
```

# Step 6 - Edit override.conf and change perms
- `sudo vi /usr/lib/systemd/system/elasticsearch.service.d/override.conf`
- make these changes: 
```
[Service]
LimitMEMLOCK=infinity
```
- `sudo chmod 644 /usr/lib/systemd/system/elasticsearch.service.d/override.conf`
```
sudo vi /usr/lib/systemd/system/elasticsearch.service.d/override.conf;sudo chmod 644 /usr/lib/systemd/system/elasticsearch.service.d/override.conf
```

# Step 7 - Reload daemons
- `sudo systemctl daemon-reload`

# Step 8 - Edit jvm_override.conf
- `sudo vi /etc/elasticsearch/jvm.options.d/jvm_override.conf`
- make these changes: 
```
-Xms2g
-Xmx2g
```

# Step 9 - Edit firewall
- `sudo firewall-cmd --add-port={9200,9300}/tcp --permanent`
- `sudo firewall-cmd --reload`
- `sudo firewall-cmd --list-all`
```
sudo firewall-cmd --add-port={9200,9300}/tcp --permanent;sudo firewall-cmd --reload;sudo firewall-cmd --list-all
```

# Step 10 - Repeats Steps 1 through 9 on all Elastic Containers

# Step 11 - Start Elastic Search on all containers
`sudo systemctl enable elasticsearch --now`

# Step 12 - Verify the cluster with Elastic API command
- `curl elastic0:9200/_cat/nodes?v`
- `curl elastic0:9200`



