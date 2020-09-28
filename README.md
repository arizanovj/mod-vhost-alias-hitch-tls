# Dynamically Configured Mass Virtual Hosting with mod_vhost_alias, Let's encrypt and Hitch TLS proxy

This is documentation on how to configure Dynamically Configured Mass Virtual Hosting on Apache. While using mod_vhost_alias you'll be able to to serve large number of web sites with similar server configs. 

This way you'll use one server config for all the website and just create folders on the server that correspond to the domain. 
In front of apache you'll have Hitch TLS proxy so you have secure communication with the clients. 

