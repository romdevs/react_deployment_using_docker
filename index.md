## Deploy React using Docker and Nginx
### Setup `docker-compose.yml` for testing the SSL certificate

```
    version: '3'
    
    services:
      whappyv-app:
        build:
          context: .
          dockerfile: Dockerfile
        image: whappyv-app
        container_name: nodejs
        restart: unless-stopped
        networks:
          - app-network
    
      webserver:
        image: nginx:mainline-alpine
        container_name: webserver
        restart: unless-stopped
        ports:
          - "80:80"
        volumes:
          - web-root:/var/www/html
          - ./nginx-conf:/etc/nginx/conf.d
          - certbot-etc:/etc/letsencrypt
          - certbot-var:/var/lib/letsencrypt
        depends_on:
          - whappyv-app
        networks:
          - app-network
    
      certbot:
        image: certbot/certbot
        container_name: certbot
        volumes:
          - certbot-etc:/etc/letsencrypt
          - certbot-var:/var/lib/letsencrypt
          - web-root:/var/www/html
        depends_on:
          - webserver
        command: certonly --webroot --webroot-path=/var/www/html --email sammy@example.com --agree-tos --no-eff-email --staging -d example.com  -d www.example.com
    
    volumes:
      certbot-etc:
      certbot-var:
      web-root:
        driver: local
        driver_opts:
          type: none
          device: /home/romdon/production/react/winona_happyv_app/build
          o: bind
    
    networks:
      app-network:
        driver: bridge
```
### There are 4 steps here:

- Setup our web app service to listen to port `80` 
- Setup the nginx services by expose port `80` and map the `build` folder to `/var/www/html` within the container using `- web-root:/var/www/html`
- Setup the certbot service with `--staging` command for testing the SSL certificate set the `volume` to `- certbot-etc:/etc/letsencrypt` and `- certbot-var:/var/lib/letsencrypt` to be able to access it from the outside
- Create and initilize the volume

### Setup `nginx.conf` for nginx configuration
```bash
server {
        listen 80;
        listen [::]:80;

        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;

        server_name winonahappyv.net www.winonahappyv.net;

        location / {
                proxy_pass http://whappyv-app:80;
        }

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }
}
```
- The `server_name` field to be the domain name
- `proxy_pass` need to be the same as our `web app service` which is in this case `whappyv-app`
- `proxy_pass` port need to be the same as `web app` exposing port

After all is set up run the following command:

`docker-compose up -d`

You will see the output like following:
```
Creating nodejs ... done
Creating webserver ... done
Creating certbot   ... done
```
Using [`docker-compose ps`](https://docs.docker.com/compose/reference/ps/), check the status of your services:

```
Name                 Command               State          Ports
------------------------------------------------------------------------
certbot     certbot certonly --webroot ...   Exit 0
nodejs      node app.js                      Up       8080/tcp
webserver   nginx -g daemon off;             Up       0.0.0.0:80->80/tcp
```
`whappyv-app` and `webserver` services is `Up` and the `certbot` container have exited with a `0` status message this means work just fine

Check the service logs with the [`docker-compose logs`](https://docs.docker.com/compose/reference/logs/) command:

`docker-compose logs service_name`

Check that your credentials have been mounted to the `webserver` container with

`docker-compose exec webserver ls -la /etc/letsencrypt/live`

you will see output like this:
```
total 16
drwx------ 3 root root 4096 Dec 23 16:48 .
drwxr-xr-x 9 root root 4096 Dec 23 16:48 ..
-rw-r--r-- 1 root root  740 Dec 23 16:48 README
drwxr-xr-x 2 root root 4096 Dec 23 16:48 example.com
```
### Renew the testing certificate for Production
```
...
	certbot:
	    image: certbot/certbot
	    container_name: certbot
	    volumes:
	      - certbot-etc:/etc/letsencrypt
	      - certbot-var:/var/lib/letsencrypt
	      - web-root:/var/www/html
	    depends_on:
	      - webserver
	    command: certonly --webroot --webroot-path=/var/www/html --email sammy@example.com --agree-tos --no-eff-email --force-renewal -d example.com  -d www.example.com
...
```
Change the `--staging` to `--force-renewal` then recreate the `certbot` using `--force-recreate` and `--no-deps`

`docker-compose up --force-recreate --no-deps certbot`

You will see output:
```
certbot      | IMPORTANT NOTES:
certbot      |  - Congratulations! Your certificate and chain have been saved at:
certbot      |    /etc/letsencrypt/live/example.com/fullchain.pem
certbot      |    Your key file has been saved at:
certbot      |    /etc/letsencrypt/live/example.com/privkey.pem
certbot      |    Your cert will expire on 2019-03-26. To obtain a new or tweaked
certbot      |    version of this certificate in the future, simply run certbot
certbot      |    again. To non-interactively renew *all* of your certificates, run
certbot      |    "certbot renew"
certbot      |  - Your account credentials have been saved in your Certbot
certbot      |    configuration directory at /etc/letsencrypt. You should make a
certbot      |    secure backup of this folder now. This configuration directory will
certbot      |    also contain certificates and private keys obtained by Certbot so
certbot      |    making regular backups of this folder is ideal.
certbot      |  - If you like Certbot, please consider supporting our work by:
certbot      |
certbot      |    Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
certbot      |    Donating to EFF:                    https://eff.org/donate-le
certbot      |
certbot exited with code 0
```

### Modifying the Web Server Configuration and Service Definition

Stop the `webserver` container

`docker-compose stop webserver`

Next, create a directory in your current project directory for your Diffie-Hellman key:

`sudo openssl dhparam -out /home/romdon/production/react/winona_happyv_app/dhparam/dhparam-2048.pem 2048`

```
server {
	listen 80;
	listen [::]:80;
	server_name winonahappyv.net www.winonahappyv.net;
	location ~ /.well-known/acme-challenge {
	allow all;
	root /var/www/html;
}

	location / {
		rewrite ^ https://$host$request_uri? permanent;
	}

}

server {
	listen 443 ssl http2;
	listen [::]:443 ssl http2;
	server_name winonahappyv.net www.winonahappyv.net;
	server_tokens off;
	ssl_certificate /etc/letsencrypt/live/winonahappyv.net/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/winonahappyv.net/privkey.pem;
	ssl_buffer_size 8k;
	ssl_dhparam /etc/ssl/certs/dhparam-2048.pem;
	ssl_protocols TLSv1.2 TLSv1.1 TLSv1;
	ssl_prefer_server_ciphers on;
	ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DH+3DES:!ADH:!AECDH:!MD5;
	ssl_ecdh_curve secp384r1;
	ssl_session_tickets off;
	ssl_stapling on;
	ssl_stapling_verify on;
	resolver 8.8.8.8;
	location / {
		try_files $uri @whappyv-app;
	}

	location @whappyv-app {
		proxy_pass http://whappyv-app:80;
		add_header X-Frame-Options "SAMEORIGIN" always;
		add_header X-XSS-Protection "1; mode=block" always;
		add_header X-Content-Type-Options "nosniff" always;
		add_header Referrer-Policy "no-referrer-when-downgrade" always;
		add_header Content-Security-Policy "default-src * data: 'unsafe-eval' 'unsafe-inline'" always;
		# add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
		# enable strict transport security only if you understand the implications
	}

	root /var/www/html;
	index index.html index.htm index.nginx-debian.html;
}	
```

Then update the `docker-compose.yml` as follow:
```
version: '3'
    
    services:
      whappyv-app:
        build:
          context: .
          dockerfile: Dockerfile
        image: whappyv-app
        container_name: nodejs
        restart: unless-stopped
        networks:
          - app-network
    
      webserver:
        image: nginx:mainline-alpine
        container_name: webserver
        restart: unless-stopped
        ports:
          - "80:80"
          - "443:443"
        volumes:
          - web-root:/var/www/html
          - ./nginx-conf:/etc/nginx/conf.d
          - certbot-etc:/etc/letsencrypt
          - certbot-var:/var/lib/letsencrypt
          - dhparam:/etc/ssl/certs
        depends_on:
          - whappyv-app
        networks:
          - app-network
    
      certbot:
        image: certbot/certbot
        container_name: certbot
        volumes:
          - certbot-etc:/etc/letsencrypt
          - certbot-var:/var/lib/letsencrypt
          - web-root:/var/www/html
        depends_on:
          - webserver
        command: certonly --webroot --webroot-path=/var/www/html --email sammy@example.com --agree-tos --no-eff-email --staging -d example.com  -d www.example.com
    
    volumes:
      certbot-etc:
      certbot-var:
      web-root:
        driver: local
        driver_opts:
          type: none
          device: /home/romdon/production/react/winona_happyv_app/build
          o: bind
	  dhparam:
		driver: local
		driver_opts:
			type: none
			device: /home/romdon/production/react/winona_happyv_app/dhparam
			o: bind
    
 networks:
   app-network:
     driver: bridge
```

Recreate the `webserver` service:

`docker-compose up -d --force-recreate --no-deps webserver`

Check your services with `docker-compose ps`:

`docker-compose ps`
```
Name                  Command               State                                    Ports                                 
------------------------------------------------------------------------------------------------------------------------------
certbot       certbot certonly --webroot ...   Exit 0                                                                         
webserver     /docker-entrypoint.sh ngin ...   Up       0.0.0.0:443->443/tcp,:::443->443/tcp, 0.0.0.0:80->80/tcp,:::80->80/tcp
whappyv-app   nginx -g daemon off;             Up       80/tcp 
```
Finally, you can visit your domain to ensure that everything is working as expected.
For more information visit:
https://www.digitalocean.com/community/tutorials/how-to-secure-a-containerized-node-js-application-with-nginx-let-s-encrypt-and-docker-compose 
