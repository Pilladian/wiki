[Back to Setup a Kubernetes Cluster](../README.md)

# Host your own Reverse Proxy to hide your Home Servers Identity

## Install Nginx and Certbot
```bash
sudo apt install nginx nginx-extras
sudo apt install python3-certbot-nginx
```

## Setup the Reverse Proxy
We need to create a new file in `/etc/nginx/conf.d/` which we will call `DOMAIN.conf`. Place the following content:

```json
# Domain 1
server {
	listen 80;
	server_name DOMAIN;	
	location / {
		proxy_pass https://TARGET_IP/;
		proxy_set_header Host $host;	
        proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
	}
	if ($request_method !~ ^(GET|HEAD|POST)$ ) {
		return 405; 
	}
	add_header X-Frame-Options "SAMEORIGIN";
	add_header X-XSS-Protection "1; mode=block";
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
Now create a self signed certificate for our default ssl server and store it in `/etc/nginx/cert/`
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout localhost.key -out localhost.crt
```

Lastly replace `/etc/nginx/sites-available/default` with the following content:
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 200;
}

server {
	listen 443 ssl default_server;
	listen [::]:443 ssl default_server;
	server_name _;
	return 200;

	ssl_certificate /etc/nginx/cert/localhost.crt;
	ssl_certificate_key /etc/nginx/cert/localhost.key;
}
```
This modifies the default server, such that all requests that do not match your DOMAINS specified earlier get a 200 response without any content.
