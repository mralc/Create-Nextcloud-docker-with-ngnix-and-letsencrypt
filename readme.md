# Setup Nextcloud using docker with ngnix and letsencrypt

Tgit i

First create a fresh install of Ubuntu 18.04 server, which has been fully updated, I am using a digital dropet with a public IP address.

## Install docker and docker-compose
First let get docker and docker compose installed on the server.
As usual, it’s a good idea to update the local database of software to make sure you’ve got access to the latest revisions.
```
sudo apt-get update
```
Install docker and docker compose
```
sudo apt install docker.io
sudo apt install docker-compose
```

## Setup docker compose

Docker Compose is a simple way to manage contationers, all the config is held in one docker-compose.yaml file.

I normally setup docker so that it is siting on a second hard drive under opt. So i first create three folders a root dir,dir for docker compose file and data and config will be keep.
```
sudo mkdir /opt/docker
sudo mkdir /opt/docker/data
sudo mkdir /opt/docker/dockercompose
```
I then cd to "/opt/docker/dockercompose" and wget the following file:
```
https://raw.githubusercontent.com/mralc/Create-Nextcloud-docker-with-ngnix-and-letsencrypt/master/docker-compose.yaml
```
Edit the file and change the following settings so that the docker can access the file systems via docker volumes. edit the first part before the colon (:) to map to the local filesystem.
Under Nextcloud:
```
    volumes:                                                                                                                         
      - /opt/docker/data/nc/config/:/config                                                                                       
      - /opt/docker/data/nc/data/:/data            
```
Under db 
```
    volumes:                                                                                                                         
      - /opt/docker/data/db/:/var/lib/postgresql/data/pgdata     
```
Under letsencrypt:
```
    volumes:
      - /opt/docker/data/ngnixconfig/:/config

```
Edit and define the postgres username,password and database name for the nextcloud database.
Under db:
```
      POSTGRES_USER: nextclouduser                                                                                                   
      POSTGRES_PASSWORD: changeme                                                                                              
      POSTGRES_DB: nextcloud           
```
Edit the following URL to match for what DNS letsencrypt cert you would like to be, remeber to make sure that there is a correct DNS A entry set pointing to the servers public IP addresss. 

edit under letsencrypt:
```
    environment:
      - URL=nextcloud.server.com
```
Start Dockercompse in the "/opt/docker/dockercompose" this will start docker as a demion and will reading the config from the yaml file.
```
docker-compose up -d
```
get the ip address of the docker demaoin
```
ip a
```
look for:"docker0" and make a note of ipaddress assicated with it mine was 172.1.0.1


visit https:\\ipaddressoftheserver:444

create a admin user account

click on storage and database and select PostigrceSQL

edit the docker IP and username,database and password and click "???"

you will get a ngnix error. browser to https:\\ipaddressoftheserver:444 and enter username and password and you should be in.

## ngnix and letsencrypt

Before you tackle this, make sure you actually have setup a domain setup with it's DNS pointing to your server.

In your letsencrypt container "appdata" go to /config/nginx/site-confs/ and create a file called nextcloud containing this code below. Make sure you change nc.domain.com to
your actual domain name, 

```
server {
	listen 443 ssl;
	server_name nextcloud.server.com;

	root /config/www;
	index index.html index.htm index.php;
	
	###SSL Certificates
	ssl_certificate /config/keys/letsencrypt/fullchain.pem;
	ssl_certificate_key /config/keys/letsencrypt/privkey.pem;
	
	###Diffie–Hellman key exchange ###
	ssl_dhparam /config/nginx/dhparams.pem;
	
	###SSL Ciphers
	ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
	
	###Extra Settings###
	ssl_prefer_server_ciphers on;

        ### Add HTTP Strict Transport Security ###
	add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
	add_header Front-End-Https on;

	client_max_body_size 0;

	location / {
		proxy_pass https://192.168.0.1:444/;
        proxy_max_temp_file_size 2048m;
        include /config/nginx/proxy.conf;
	}
}
```
Then you need to enter your nextcloud "appdata" and edit the file /config/www/nextcloud/config/config.php

so it looks like this
```
<?php
$CONFIG = array (
 'memcache.local' => '\\OC\\Memcache\\APCu',
 'datadirectory' => '/data',
 'instanceid' => 'xxxxxxxxxxxx',
 'passwordsalt' => 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
 'secret' => 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
 'trusted_domains' => 
 array (
 0 => '192.168.0.1:444',
 1 => 'nextcloud.server.com',
 ),
 'overwrite.cli.url' => 'https://nextcloud.server.com',
 'overwritehost' => 'nextcloud.server.com',
 'overwriteprotocol' => 'https',
 'dbtype' => 'mysql',
 'version' => '11.0.1.2',
 'dbname' => 'nextcloud',
 'dbhost' => '192.168.0.1:3305',
 'dbport' => '',
 'dbtableprefix' => 'oc_',
 'dbuser' => 'oc_CHBMB1',
 'dbpassword' => 'xxxxxxxxxxxxxxxxxxxx',
 'logtimezone' => 'UTC',
 'installed' => true,
 );
```


Next restart docker-compose to make the change live
```
docker-compose restart
```
