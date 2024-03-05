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





# Step 4 - 




# Step 5 - 

















LOCAL CERTIFICATE AUTHORITY
localCA.key: A “.key” file is a digital private key file that is used in asymmetric encryption algorithms, such as RSA or ECC, to encrypt and decrypt data, or to sign and verify digital signatures.
localCA.crt: root certificate, we need private key to generate this
After these are generated, we will use the CA to sign new certificates that are used to secure internal web application communications.
REPO
repo.key: private key
repo.csr: certificate signing request, we need the private key to generate this. We will send the csr to the CA for a new certificate
repo.ext: certificate extension config. This is where the Subject Alternative Name can be set to the hostname of the repo. This step needs to be done in order for the browser to browse to the fileshare and package repository using the repo’s hostname. We create this
repo.crt: Use the CSR and the CA private key to sign a new x.509 certificate with the CA. A “.crt” file is a digital certificate file that contains a public key, and is used to verify the authenticity of a website, server, or client. It is one of the most commonly used file formats for X.509 digital certificates, which are used in SSL/TLS encryption to secure web traffic and other types of communication.
The repo.crt and the repo.key files can now be used to set up HTTPS for the fileshare and the package repository