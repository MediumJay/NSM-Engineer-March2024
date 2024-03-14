# overview

# Step 1 - SSH into elastic0
- `ssh elastic0`

# Step 2 - Get copies of localCA.crt and .key
- `scp "repo:~/certs/localCA.{crt,key}" ~/.`

# Step 3 - Create a ymal to streamline generating the certs
- `sudo vi stack.yml`
- insert:
(adds subject alternative names for each node) 
```
instances:
  - name: "elastic0" 
    ip: 
      - "10.81.139.30"
    dns: 
      - "elastic0"

  - name: "elastic1" 
    ip: 
      - "10.81.139.31"
    dns: 
      - "elastic1"

  - name: "elastic2" 
    ip: 
      - "10.81.139.32"
    dns: 
      - "elastic2"

  - name: "kibana" 
    ip: 
      - "10.81.139.50"
    dns: 
      - "kibana"
```

# Step 4 - Generate certs with elastic cert utility using localCA cert and key 
- `sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca-cert ~/localCA.crt --ca-key ~/localCA.key --days 1095 --in ~/stack.yml --out ~/stack-certs.zip`

# Step 5 -Install unzip
- `sudo yum install unzip -y` 

# Step 6 - Change ownership of the zip
- `sudo chown elastic: stack-certs.zip`

# Step 7 - Unzip
- `unzip stack-certs.zip`


# Step 8 - Make a copy of required key store ?
- `cp ~/elastic0/elastic0.p12 ~/.`

# Step 9 - Copy the certs to hosts
- `for host in elastic{1..2} kibana; do scp ~/$host/$host.p12 $host:~/.; done`

# Step 10 - Make copy of localCA.crt over to kibana
- `scp ~/localCA.crt kibana:~/.`

# Step 11 - Make a directory
- `sudo mkdir /etc/elasticsearch/certs`

# Step 12 - copy the elastic# p12 over
- `sudo cp elastic#.p12 /etc/elasticsearch/certs`

# Step 13 - Chown the p12
- `sudo chown root:elasticsearch /etc/elasticsearch/certs/elastic#.p12`

# Step 14 - 
- `sudo chmod 640 /etc/elasticsearch/certs/elastic0.p12`

# Step 15 - Edit elasticsearch.yml
- `sudo vi /etc/elasticsearch/elasticsearch.yml` 
- shift + g + o at the bottom (dont forget to edit the node #)
```
xpack.security.enabled: true

xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: full
xpack.security.transport.ssl.keystore.path: certs/elastic0.p12

xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/elastic0.p12
```

# Step 16 - Run these two commands
- `sudo /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password`

- `sudo /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password`
# Step 17 - Validate Key Stores

- `sudo /usr/share/elasticsearch/bin/elasticsearch-keystore list`








# Step 18 - repeat steps 11 through 17 on the other elastic nodes


# Step 19 - Hop into kibana and go to dev tools
- we will prep the cluster for a full restart
```
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "primaries"
  }
}
```

# Step 20 - Stop kibana, and logstash, restart elastic search
- `sudo systemctl stop kibana`
-`sudo systemctl stop logstash` on all pipelines
- `sudo systemctl restart elasticsearch` on all elastics
sudo 
# Step 21 - curl the API
- `curl https://elastic1:9200/_cat/nodes`
- should get an error here
- we turned on rolebased AC and we have no credentails rn


# Step 22 - Create some accounts and set their passwords
- `sudo /usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive`

# Step 23 - do the curl again with elastic user
- `curl https://elastic1:9200/_cat/nodes -u elastic`
- `curl https://elastic1:9200/_cluster/health?pretty -u elastic`
  - cluster health is yellow because replication not enabled?

# Step 24 - Run this api call to turn replication back on and check it
```
curl -X PUT "https://elastic0:9200/_cluster/settings?pretty" -u elastic -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "cluster.routing.allocation.enable": null
  }
}
'
```
- `curl https://elastic1:9200/_cluster/health?pretty -u elastic`











# Step 25 - Move on to Kibana
- `ssh kibana`
- kibana.p12 and localCA.crt should be in ~


# Step 26 - make certs directory
- `sudo mkdir /etc/kibana/certs`

# Step 27 - move LocalCA.crt
- `mv ~/localCA.crt /etc/kibana/certs/`

# Step 28 - Change perms
- `sudo chown root:kibana /etc/kibana/certs/localCA.crt`
- `sudo chmod 640 /etc/kibana/certs/localCA.crt`

# Step 29 - Update kibana.yml
```
server.port: 5601
server.host: localhost
server.name: kibana
elasticsearch.hosts: ["https://elastic0:9200", "https://elastic1:9200", "https://elastic2:9200"]

elasticsearch.username: kibana_system

elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/localCA.crt"]
```

# Step 30 - Create a keystore
- `sudo -u kibana /usr/share/kibana/bin/kibana-keystore create`

# Step 31 - Create a random pass
- `shuf -er -n32  {A..Z} {a..z} {0..9} | tr -d '\n' | tr -dc 'a-zA-Z0-9'`

# Step 32 - 
- `sudo -u kibana /usr/share/kibana/bin/kibana-keystore add xpack.security.encryptionKey`
- enter the random vlaue from 31

# Step 33 - add password for kibana user
- `sudo -u kibana /usr/share/kibana/bin/kibana-keystore add elasticsearch.password`

# Step 34 - List the two passwords
- `sudo -u kibana /usr/share/kibana/bin/kibana-keystore list`

# Step 35 - Move certs over for NGINX to use
- `openssl pkcs12 -in kibana.p12 -clcerts -nokeys -out kibana.crt`

- `openssl pkcs12 -in kibana.p12 -nocerts -nodes -out kibana.key`

- `sudo mv ~/kibana.* /etc/nginx/.`

# Step - change perms again
- `sudo chown root: /etc/nginx/kibana.*`
- `sudo chmod 640 /etc/nginx/kibana.*`

# Step - Add a password file for NGINX so it can read the .. file

- `sudo vi /etc/nginx/ssl_kibana.txt`
- wq, you dont need to put anything in there
- `sudo chown root: /etc/nginx/ssl_kibana.txt`
- `sudo chmod 640 /etc/nginx/ssl_kibana.txt`

# Step - Update kibana.conf
-  `sudo vi /etc/nginx/conf.d/kibana.conf`
```
server {
  listen 80;
  server_name _;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  server_name kibana;

  ssl_certificate /etc/nginx/kibana.crt;
  ssl_certificate_key /etc/nginx/kibana.key;
  ssl_password_file /etc/nginx/ssl_kibana.txt;

  ssl_session_cache shared:SSL:10m;
  ssl_session_timeout 1d;
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
  ssl_prefer_server_ciphers off;

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


# Step 39 - reload nginx
- `sudo nginx -s reload`
- `sudo systemctl status nginx` 

# Step 40 - Add 443 to firewall
- `sudo firewall-cmd --add-port=443/tcp --permanent`
- `sudo firewall-cmd --reload`

# Step 41 - Start kibana
- `sudo systemctl start kibana`







# Step 42 - Move on to securing logstash
- go to dev tools in kibana
- create a role "logstash_writer"
```
POST _security/role/logstash_writer
{
  "cluster": ["manage_index_templates", "monitor", "manage_ilm"], 
  "indices": [
    {
      "names": [ "ecs-*", "fsf-*", "parse-failures-*", "indexme-*" ], 
      "privileges": ["write","create","delete","create_index","manage","manage_ilm"]  
    }
  ]
}
```

# Step 43 - Create a user to use the role
```
POST _security/user/logstash_internal
{
  "password" : "training",
  "roles" : [ "logstash_writer"],
  "full_name" : "Internal Logstash User"
}
```

# Step 44 - get on pipeline0
- `ssh pipeline0`

# Step - get into /etc/logstash/conf.d/
- `cd /etc/logstash/conf.d/`

# Step - edit the config file
- `sudo vi logstash-9999-output-elasticsearch.conf`
- new yaml: 
```
output {
  stdout { codec => json }
  # Requires event module and category
  if [event][module] and [event][category] {

    # Requires event dataset
    if [event][dataset] {
      elasticsearch {
          hosts => [ "elastic0", "elastic1", "elastic2" ]
          index => "ecs-%{[event][module]}-%{[event][category]}-%{+YYYY.MM.dd}"
          manage_template => false
          user => logstash_internal
          password => "${ES_PWD}"
          ssl => true
      }
    }

    else {
      # Suricata or Zeek JSON error possibly, ie: Suricata without a event.dataset seen with filebeat error, but doesn't have a tag
      if [event][module] == "suricata" or [event][module] == "zeek" {
        elasticsearch {
            hosts => [ "elastic0", "elastic1", "elastic2" ]
            index => "parse-failures-%{+YYYY.MM.dd}"
            manage_template => false
            user => logstash_internal
            password => "${ES_PWD}"
            ssl => true

        }
      }
      else {
        elasticsearch {
            hosts => [ "elastic0", "elastic1", "elastic2" ]
            index => "ecs-%{[event][module]}-%{[event][category]}-%{+YYYY.MM.dd}"
            manage_template => false
            user => logstash_internal
            password => "${ES_PWD}"
            ssl => true
             
       }
      }
    }
  }

  else if [@metadata][stage] == "fsfraw_kafka" {
    elasticsearch {
        hosts => [ "elastic0", "elastic1", "elastic2" ]
        index => "fsf-%{+YYYY.MM.dd}"
        manage_template => false
        user => logstash_internal
        password => "${ES_PWD}"
        ssl => true

    }

  }

  else if [@metadata][stage] == "_parsefailure" {
    elasticsearch {
        hosts => [ "elastic0", "elastic1", "elastic2" ]
        index => "parse-failures-%{+YYYY.MM.dd}"
        manage_template => false
        user => logstash_internal
        password => "${ES_PWD}"
        ssl => true

    }
  }

  # Catch all index that is not RockNSM or ECS or parse failures
  else {
    elasticsearch {
        hosts => [ "elastic0", "elastic1", "elastic2" ]
        index => "indexme-%{+YYYY.MM.dd}"
        manage_template => false
        user => logstash_internal
        password => "${ES_PWD}"
        ssl => true

    }
  }

}
```

# Step - Run this command
- `sudo /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash create`

# Step - change perms
- `sudo chown root:logstash /etc/logstash/logstash.keystore;sudo chmod 640 /etc/logstash/logstash.keystore`

# Step - Runt his command
- `sudo /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash add ES_PWD`
- enter password

# Step - List it
- `sudo /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash list`

# Step - Repeat on other logstash pipelines

# Step - Restart logstash
- `sudo systemctl start logstash`

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

