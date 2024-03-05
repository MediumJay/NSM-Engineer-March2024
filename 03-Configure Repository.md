# Step 1 - Access the repo box

- Get into the REPO box
  - ssh repo


# Step 2 - Install the repo tool NGINX

- Use yum
  - `sudo yum install nginx`

- where is the repo at? 

# Step 3 - Unzip the class files in home directory to stage our environment

- unzip the class files to the nginz directory
  - `sudo unzip ~/all-class-files.zip -d /usr/share/nginx`

- rename all-class-files to fileshare ll
  - `sudo mv /usr/share/nginx/all-class-files/ /usr/share/nginx/fileshare`

- verify files were unzipped properly
  - `ll /usr/share/nginx/fileshare/`

- ? what is emerging rules for? suricata rules?
  - `sudo mv ~/emerging.rules.tar.giz /usr/share/nginx/fileshare/`

- verify files were moved properly again
  - `ll /usr/share/nginx/fileshare/`


# Step 4 - 