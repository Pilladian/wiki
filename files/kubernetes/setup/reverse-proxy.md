[Back to Setup a Kubernetes Cluster](../README.md)

# Host your own Reverse Proxy to hide your Home Servers Identity

## Install Nginx and Certbot
```bash
sudo apt install nginx nginx-extras
sudo apt install python3-certbot-nginx
```

## Setup the Reverse Proxy
We need to create a new file in `/etc/nginx/conf.d/` which we will call `reverse-proxy.conf`. Place the following content:

```json
# Domain 1
server {
	listen 80;
	server_name DOMAIN_1;	
	location / {
		proxy_pass https://TARGET_IP/;
		proxy_set_header Host DOMAIN_1;	
        proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
	}
}

# Domain 2
server {
	listen 80;
	server_name DOMAIN_2;	
	location / {
		proxy_pass https://TARGET_IP/;
		proxy_set_header Host DOMAIN_2;	
        proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
	}
}
```

## Secure Nginx
Now we need to apply some security configurations to the default nginx server. First open `/etc/nginx/nginx.conf` and add the following content:
```nginx
http {
    ...
    server_tokens off;          # This removes the nginx-version from default pages
    more_clear_headers Server;  # This removes the Server Header from server responses
	ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA HIGH !RC4 !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS"; # This only allows specific TLS Ciphers
    ...
}
```
Lastly replace `/etc/nginx/sites-available/default` with the following content:
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 200;
}
```
This modifies the default server, such that all requests that do not match your DOMAINS specified earlier get a 200 response without any content.
