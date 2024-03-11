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

# Step 5 - install nginx
- `sudo yum install nginx -y`

# Step - edit kibana.conf inside nginx directory
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

# Step - Edit the nginx.conf in the nginx directory
- `sudo vi /etc/nginx/nginx.conf`
- comment these lines out to prevent conflicts
  - `:set nu`
```
39 #
40 #
41 #
```


# Step - Firewall
- `sudo firewall-cmd --add-port=80/tcp --permanent`
- `sudo firewall-cmd --reload`
- `sudo firewall-cmd --list-all`
```
sudo firewall-cmd --add-port=80/tcp --permanent;sudo firewall-cmd --reload;sudo firewall-cmd --list-all
```

# Step - Enable and start nginx
`sudo systemctl enable nginx --now`


# Step - Enable and start kibana
`sudo systemctl enable kibana --now`

# Step - troubleshoot
`journalctl -xeu kibana`
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

# Step - 

# Step - 

# Step - 

# Step - 

# Step - 

# Step - 