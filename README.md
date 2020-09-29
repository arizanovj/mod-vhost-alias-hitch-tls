# Dynamically Configured Mass Virtual Hosting with mod_vhost_alias, Let's encrypt and Hitch TLS proxy

This is documentation on how to configure Dynamically Configured Mass Virtual Hosting on Apache. While using mod_vhost_alias you'll be able to to serve large number of web sites with similar server configs. 

With mod_vhost_alias you can use one server config for all the website and just create folders on the server that correspond to the domain. 
In front of apache you'll have Hitch TLS proxy so you have secure communication with the clients. 
# Setup Apache
First you need to setup two apache vhosts. The first vhost will serve the traffic comming from the Hitch proxy. Hitch forwards the traffic comming from the frontend to 127.0.0.1:80. 
```sh
<VirtualHost 127.0.0.1:80>
    UseCanonicalName Off
    VirtualDocumentRoot /var/www/%0/html
        <Directory /var/www/>
           Options Indexes FollowSymLinks
           AllowOverride All
           Require all granted
        </Directory>
        <IfModule mod_dir.c>
            DirectoryIndex index.html index.xhtml index.htm
        </IfModule>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
You can notice that here you don't have redirect from http -> https. Without this config the client will get into loop and at the end you'll get the error "Too many reidrects". 
This will happen because Hitch will forward the traffic to Apache and Apace will forward it back to https which is  proxied by Hitch. The vhost above serves the traffic comming locally from Hitch.
 
The following vhost serves https traffic comming to Apache - http traffic is redirected to https.
```sh
<VirtualHost *:80>
    UseCanonicalName Off
    VirtualDocumentRoot /var/www/%0/html
    
    RewriteEngine on
    RewriteCond %{SERVER_PORT} !^443$
    RewriteRule ^/(.*) https://%{HTTP_HOST}/$1 [NC,R=301,L]
    
        <Directory /var/www/%0>
           Options Indexes FollowSymLinks
           AllowOverride All
           Require all granted
        </Directory>
        <IfModule mod_dir.c>
            DirectoryIndex index.html index.xhtml index.htm
        </IfModule>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
When you want new domain served this way you add the domain folder in /var/www.
To serve example.com you need to following strucutre /var/www/example.com/html. The website goes into the html folder.
# Setup Hitch
This section has been taken from [here](https://docs.varnish-software.com/tutorials/hitch-letsencrypt/) with small modifications. 
First you need to install Hitch. On Debian based system you can install it with the following command:
```sh
sudo apt-get install htich
```
Create a new file /usr/local/bin/hitch-deploy-hook with your editor and paste this into it:
```sh
#!/bin/bash
# Full path to pre-generated Diffie Hellman Parameters file
dhparams=/etc/hitch/dhparams.pem

if [[ "${RENEWED_LINEAGE}" == "" ]]; then
    echo "Error: missing RENEWED_LINEAGE env variable." >&2
    exit 1
fi

umask 077
cat ${RENEWED_LINEAGE}/privkey.pem \
${RENEWED_LINEAGE}/fullchain.pem \
${dhparams} > ${RENEWED_LINEAGE}/hitch-bundle.pem
```
Make the script executable:
```sh
sudo chmod a+x /usr/local/bin/hitch-deploy-hook
```
In order to enable Perfect Forward Secrecy, we need to create a Diffie Hellman Parameter file that Hitch will use, this is done using openssl:
```sh
openssl dhparam 2048 | sudo tee /etc/hitch/dhparams.pem
```
Verify that Hitch is set up with the correct backend in /etc/hitch/hitch.conf
```sh
backend = "[127.0.0.1]:80"
```
Lastly enable the systemd service:
```sh
sudo systemctl enable hitch
```
# Setup certbot
First install certbot:
```sh
sudo apt-get install certbot
```
Generate the certificate for one of the domains you want to host in the following way:
```sh
sudo certbot --apache certonly --preferred-challenges http \
--http-01-port 80 -d example.com \
--deploy-hook="/usr/local/bin/hitch-deploy-hook"
```
 The first time you run this certbot will ask you for some info. 
 You need to start the certbot-renew timer, which handles automatic certificate renewals once per day:
 ```sh
sudo systemctl enable certbot-renew.timer
sudo systemctl start certbot-renew.timer
```
# Start Hitch

The final Hich config file ( /etc/hitch/hitch.conf ) needs to look like this:

 ```sh
backend = "[127.0.0.1]:80"
frontend="[*]:443"
pem-file = "/etc/letsencrypt/live/example.com/hitch-bundle.pem"

```


Start hitch with the following command
 ```sh
sudo systemctl start hitch
```

When you add a new domain the process will again be:
 - Setup domain at /var/www/example1.com/html
 - Generate the SSL for example1.com as we did for example.com
 - Add the pem file at the end of the Htich config like this:
  ```sh
backend = "[127.0.0.1]:80"
frontend="[*]:443"
pem-file = "/etc/letsencrypt/live/example.com/hitch-bundle.pem"
pem-file = "/etc/letsencrypt/live/example1.com/hitch-bundle.pem"

```
 - Reload hitch 
  ```sh
sudo systemctl reload hitch
```
The benefits of this setup are:
 - You don't need to restart Apache or do graceful reload every time you add new domain
 - The load of secure traffic goes to the Htich proxy

If you have a large number of websites you can automate the process easily. 



