Were making the repo a CA because... security? to secure our repo and fileshare?


# Step 1 - Make directory to hold certificates

- `mkdir ~/certs`
- `cd ~/certs`

# Step 2 - Generate a private key using OpenSSl for certificate authority

- `openssl genrsa -des3 -out localCA.key 2048`
  - des3 is encryption
  - out is output where we wanna save it
  - localCA.key is the name
  - 2048 is length in bits
- enter a pass phrase
  - "training"
- check that the cert was generated
  - `ll ~/certs` 




# Step 3 - Use private key to generate a root certificate

- `openssl req -x509 -new -nodes -key localCA.key -sha256 - days 1095 -out localCA.crt`
  - x509 is for self-signed 509 cert
  - new for
  - nodes for 
  - localCA.key is the location of the key to use for gen cert
  - sha256 is hashing algorithm
  - days 1095 length of validity for cert (3 years)
- enter the private key passphrase to sign the cert
  - "training"
- fill in cert details
  - Country Name: US
  - State: CA
  - Locality name: Mountain View
  - Org Name: Elastic
  - Org Unit Name: Security
  - Common Name: Instructor
  - email: 
- check certs directory for localCA.crt
  - `ll ~/certs`





# Step 4 - Create repo box certs
- `openssl genrsa -out repo.key 2048`
  - this is the private key for the repo

-




# Step 5 - Use the private key to create a certificate signing request

- `openssl req -new -key repo.key -out repo.csr`
  - this is for??

- Give information about the rep
  - Country Name: US
  - State: CA
  - Locality name: Mountain View
  - Org Name: Elastic
  - Org Unit Name: Security
  - Common Name: Repo
  - email: 
- Enter extra attributes
  - challenge password: 

# Step 6 - Create a certificate browsing extension?
for us to browse to the fileshare using hostname

- `sudo vi ~/certs/repo.ext`
  - make it look like this: 
```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = repo
IP.1 = 10.81.139.10

```

# Step 7 - Use certificate signing request and extension to sign a new certificate with the CA

- `openssl x509 -req -in repo.csr -CA localCA.crt -CAkey localCA.key -CAcreateserial -out repo.crt -days 365 -sha256 -extfile repo.ext`
  - take the csr and signing it with the local CA and producing a signed cert which is repo.crt?
  - enter the passphare for localCA.key: "Training" 
  - 


When you're done you should have in your certs directory: 
- localCA.crt
- localCA.key
- localCA.srl
- repo.crt
- repo.csr
- repo.ext
- repo.key 


# Steps 8 - now we can move these into the right directory to do something for our repo

- `sudo mv ~/certs/repo.crt /etc/nginx/`
  - Moves the ... 
- `sudo mv ~/certs/repo.key /etc/nginx/` 
  - Moves the private repo key
- check the keys are in the directory
  - `ll /etc/nginx/`


# Step 9 - We can set up the config file for our proxy (redirection and reverse proxy)
- `sudo vi /etc/nginx/conf.d/proxy.conf`
  - put this inside: 
```
server {
  listen 80;
  server_name _;
  ssl_certificate /etc/nginx/repo.crt;
  ssl_certificate_key /etc/nginx/repo.key;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  server_name repo;

  ssl_certificate /etc/nginx/repo.crt;
  ssl_certificate_key /etc/nginx/repo.key;

  ssl_session_cache shared:SSL:10m;
  ssl_session_timeout 1d;
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
  ssl_prefer_server_ciphers off;

  proxy_max_temp_file_size 0;

  location /fileshare/ {
    proxy_pass http://127.0.0.1:8000/;
  }

  location /packages/ {
    proxy_pass http://127.0.0.1:8008/;
  }
}
```

# Step 10 - Modify the nginx configuration to comment out some lines
- `set nu` in vi to see line numbers
- comment out lines 39-41 to prevent conflicts



# Step 11 - Create a Conffiguration file for our packages and fileshare
- `sudo vi /etc/nginx/conf.d/packages.conf`
  - insert this: 
```
server {
  listen 127.0.0.1:8008;
  location / {
    root /repo;
    autoindex on;
    index index.html index.htm;
  }
}
```
  - this does...

- `sudo vi /etc/nginx/conf.d/fileshare.conf`
  - insert this: 
```
server {
  listen 127.0.0.1:8000;
  location / {
    root /usr/share/nginx/fileshare;
    autoindex on;
    index index.html index.htm;
  }
}
```
  - this does ...

# Step 12 - Enable 80 and 443 to be allowed through firewall
- `sudo firewall-cmd --add-port={80,443}/tcp --permanent`
- `sudo firewall-cmd --reload`
- `sudo firewall-cmd --list-all`


# Step 13 - start NGINX

- `sudo systemctl start nginx`
- `sudo systemctl enable nginx`
- `sudo systemctl status nginx`
- verify your reverse proxy
  - `ss -lnt`
  - can see the sockets open, we see loopback on 8000 and 8008
- browse to repo, check packages and fileshare in browser
  - https://repo/fileshare
  - https://repo/packages
  - we receive error since cert is invalid so we will fix this and do it the correct way
- scp the cert to the student laptop 
  - `sudo scp elastic@repo:/home/elastic/certs/localCA.crt ~/localCA.crt`
- go into browser settings
  - privacy and security > security > manage certificates > authorities tab > import
  - select the localCA.crt and check identify websites, click ok
- any files added to the /usr/share/nginx/fileshare directory will be hosted on the fileshare.


# Step 14 - point containers to our local repo

- install reposync and yum-utils
  - `sudo yum install yum-utils`
- local repositories are at /repo but one is missing
  - add it with repo-sync
  - `sudo reposync -l --repoid=extras --download_path=/repo/local-extras`
  - the -l enables yum support
- use createrepo to turn that local-extras into a repo yum can use
  - `sudo yum install createrepo -y`
  -`sudo createrepo local-extras`
    - I was in repo directory 
- use script to go through the containers and copy the localCA.crt to the other boxes and update the CA trust
```
for host in elastic{0..2} pipeline{0..2} kibana sensor; do sudo scp ~/certs/localCA.crt elastic@$host:~/localCA.crt && ssh -t elastic@$host 'sudo mv ~/localCA.crt /etc/pki/ca-trust/source/anchors/ && sudo update-ca-trust'; done
```

- configure sensor to see local repository
 - doing a yum list for suricata will show internet repositories
- to point it to local we must make an archive directory
  - `mkdir ~/archive`
- /etc/yum.repos.d has all current repos that we will move into the archive directory
  - `sudo mv /etc/yum.repos.d/* ~/archive/`









# Step 15 - Create a new local repo config file

- `sudo vi /etc/yum.repos.d/local.repo`
  - paste this config in: 

```
[local-base]
name=local-base
baseurl=https://repo/packages/local-base/
enabled=1
gpgcheck=0

[local-rocknsm-2.5]
name=local-rocknsm-2.5
baseurl=https://repo/packages/local-rocknsm-2.5/
enabled=1
gpgcheck=0

[local-elasticsearch-7.x]
name=local-elasticsearch-7.x
baseurl=https://repo/packages/local-elastic-7.x/
enabled=1
gpgcheck=0

[local-epel]
name=local-epel
baseurl=https://repo/packages/local-epel/
enabled=1
gpgcheck=0

[local-extras]
name=local-extras
baseurl=https://repo/packages/local-extras/
enabled=1
gpgcheck=0

[local-updates]
name=local-updates
baseurl=https://repo/packages/local-updates/
enabled=1
gpgcheck=0
```
  - baseurl is where the repo is
  - enabled = 1 means its on
  - gpgcheck = 0 means no gpgkey checks; 1 in production
- `sudo yum makecache fast` 
  - will show local repos being added
- `sudo yum list suricata`
  - will show you that the suricata is coming from the local repo now





# What do these files mean?

## LOCAL CERTIFICATE AUTHORITY
- ### **localCA.key**: A “.key” file is a digital private key file that is used in asymmetric encryption algorithms, such as RSA or ECC, to encrypt and decrypt data, or to sign and verify digital signatures.
- ### **localCA.crt**: root certificate, we need private key to generate this
After these are generated, we will use the CA to sign new certificates that are used to secure internal web application communications.

## REPO
- ### **repo.key**: private key
- ### **repo.csr**: certificate signing request, we need the private key to generate this. We will send the csr to the CA for a new certificate
- ### **repo.ext**: certificate extension config. This is where the Subject Alternative Name can be set to the hostname of the repo. This step needs to be done in order for the browser to browse to the fileshare and package repository using the repo’s hostname. We create this
- ### **repo.crt**: Use the CSR and the CA private key to sign a new x.509 certificate with the CA. A “.crt” file is a digital certificate file that contains a public key, and is used to verify the authenticity of a website, server, or client. It is one of the most commonly used file formats for X.509 digital certificates, which are used in SSL/TLS encryption to secure web traffic and other types of communication.
- The repo.crt and the repo.key files can now be used to set up HTTPS for the fileshare and the package repository




# Summary: 


Configure Repo and Fileshare Summary

Fileshare
- Install nginx
- Move class files and emerging ruleset into fileshare directory
Repo container = Local Certificate Authority
- Generate localCA.key: CA private key
- Use the CA private key to generate localCA.crt: root certificate. Fill out info
Generate repo.key: Repo private key
- Use the Repo private key to generate repo.csr: certificate signing request to send to the CA to request a new certificate. Fill out info
- Create a certificate extension config so that the browser can browse to the fileshare using the repo’s host name
- Use the CSR and the extension config to sign a new certificate with the CA: repo.crt
repo.crt (public key/digital cert file) and repo.key (private) can now be used to set up HTTPS for the fileshare a package repository
- Set up reverse proxy with nginx (proxy.conf)
- Modify nginx.conf to disable port 80 listener
- Create packages.conf and fileshare.conf to listen on localhost
- Enable the appropriate ports
- Start nginx
- Trust CA on Student laptop and import it to the browser. All certificates signed by the CA will now be trusted and the browser will successfully validate the certs when accessing the repo and fileshare resources.
- Trust CA on Kit boxes: In order for the other boxes in the kit to be able to reach the package repository and install packages via yum, they need to validate the repo certificate that is used to encypt the yum communications over the network.
- Modify local.repo so it points to the new hosted location using https
- End goal: Browse repo packages and fileshare using tls/https and point to secured local repository packages from any container.





# ChatGPT Simplications to the Summary
Fileshare Setup (Using Nginx):

## Nginx Installation: 

Nginx is like a server that helps us share files with others.
Moving Files: We put our class files and ruleset into a special folder that Nginx can access. This folder is where we store files that others can see and use.

## Setting up Nginx as a CA: 

We made Nginx act like a special authority that can give out secure certificates. These certificates help ensure that our communications are safe and private.
Creating Keys and Certificates:
We made a secret key for our special authority (CA).
With this key, we made a certificate that says our authority is trustworthy (localCA.crt).
We also made a key for our file repository (repo.key).
We asked our special authority (CA) to give us a certificate for our file repository (repo.crt).


## Configuring Nginx:

We told Nginx to act as a middleman using a reverse proxy. It helps manage our connections.
We made sure Nginx only listens to secure connections (HTTPS) and not unsecured ones (HTTP).
We set up configurations for our packages and fileshare to work securely.

## Starting Nginx: 

We turned on Nginx so that others can access our files using a secure connection.
Certificate Trust:

## Trusting CA Certificate: 

We made sure that laptops and browsers trust our special authority (CA) so they know the certificates it gives out are safe.

## Trusting CA Certificate on Kit Boxes: 

We did the same thing for other computers (Kit boxes) in our setup to ensure they can access our files securely.
Repository Configuration:

## Changing Repository Settings: 

We updated our repository settings to use the new secure location, making sure all communications are secure.
Overall Goal: We set up a way to share files securely using HTTPS, making sure everything is safe and private within our network environment.

I hope this breakdown helps clarify the process a bit more! Let me know if there's anything specific you'd like to understand better.