# Overview

# Step 1 - SSH onto Kibana
- `ssh kibana`

# Step 2 - Install Kibana
- `sudo yum install kibana -y`

# Step 3 - Backup the kibana.yml 
- `sudo mv /etc/kibana/kibana{.yml,.yml.bk}`

# Step 4 - Edit Kibana.yml
- `sudo vi /etc/kibana/kibana.yml`
- make these edits: 
```
server.port: 5601
server.port: localhost
server.name: kibana
elasticsearch.hosts: ["http://elastic0:9200","http://elastic1:9200","http://elastic2:9200"]
```

# Step 5 - install nginx for reverse proxy
- `sudo yum install nginx -y`

# Step 6 - edit kibana.conf inside nginx directory
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

# Step 7 - Edit the nginx.conf in the nginx directory
- `sudo vi /etc/nginx/nginx.conf`
- comment these lines out to prevent conflicts
  - `:set nu`
```
39 #
40 #
41 #
```


# Step 8 - Firewall
- `sudo firewall-cmd --add-port=80/tcp --permanent`
- `sudo firewall-cmd --reload`
- `sudo firewall-cmd --list-all`
```
sudo firewall-cmd --add-port=80/tcp --permanent;sudo firewall-cmd --reload;sudo firewall-cmd --list-all
```

# Step 9 - Enable and start nginx
`sudo systemctl enable nginx --now`


# Step 10 - Enable and start kibana
`sudo systemctl enable kibana --now`

# Step 11 - Go to browser and navigate to Kibana
- http://kibana

# Step 12 - Explore kibana
- Checkout Management > Stack Managemetn 
- Checkout management > dev tools

# Step 13 - Pull down ECS mappings from fileshare, unzip, and install JQ
- `curl -LO https://repo/fileshare/kibana/ecskibana.tar.gz`
- `tar -zxvf ecskibana.tar.gz`
- `sudo yum install jq -y`

# Step 14 - Add the mapping templates to kibana

- `cd ecskibana` 
- `./import-index-templates.sh http://elastic0:9200`

# Step 15 - Verify the mapping templates were added to kibana using the API
- navigate to devtools (management> dev tools)
- `GET _cat/templates?v`
- look for `ecs_zeek` and `ecs_suricata`
- cannot create an index pattern for FSF until logstash is configured


# Step - troubleshoot
- `journalctl -xeu kibana`
- Make sure /etc/hosts only has our entries and not the defaults below; i made this error
- Also if its bugged, stop and restart nginx+kibana

