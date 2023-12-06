# Overview of my Docker System
I have a master docker folder that provisions nginx and servers. 

The dockers that contains the websites will expose a port, normaly between 8000 - 8999.

Nginx will point to the docker container and uses proxy pass to route.

`Nginx-Home` is the nginx docker system, called this as its a home made system.

*This readme was created to remind me how to add websites to my system, its not a guid to nginx or docker, but it may be of use to some. Take with a pinch of sault*
# Adding A Website
### Docker Compose

Add a new service

```yml
build: ./[website-folder]
expose: 8000 # Or any port not in use
```

### Docker file

Create a docker file in the chosen website folder, this will depend on the language, project.

This is an example
```dockerfile
FROM python:3.10.12

WORKDIR ./

RUN git clone https://github.com/TidalCub/StormsBurrow

RUN pip3 install Flask
RUN pip3 install gunicorn

EXPOSE 8000

CMD ["gunicorn", "-b", "0.0.0.0:8000", "StormsBurrow:app"]
```

### Configure nginx

Add a server block

```conf
server {

}
```

next add in the ports to listen to, for this example it is 80 (if using ssl have to server blocks, one listening to 80 and the other to 443 SSL) and the domain

```conf
  listen 80;
  listen [::]:80;
  server_name [your_domain] www.[your_domain];
```

next add the proxy details to point to the docker container
```conf
  location / {
    proxy_pass http://web:8000;  # Use the name of your app service in Docker Compose
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    }
```

the entire thing should look like this 
``` conf
server {
  listen 80;
  listen [::]:80;
  server_name [your_domain] www.[your_domain];

  location / {
    proxy_pass http://web:8000;  # Use the name of your app service in Docker Compose
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```
restart docker by `docker compose restart`

---
# SSL
### Getting a Cert
using certbot get a SSL certificate:

`sudo certbot certonly --standalone`

Follow the prompts, chose the option that doesn't require a http server (CertBot will set one up)

Enter in the domain you want the SSL Cert for.

### Copying the Cert
Once this is complete, copy over the folder 
`/etc/letsencrypt/live/[your_domain]` into the `certs` folder keeping the same file name

### Configuring Docker Image

*No config is necessary as docker mounts the certs folder as a volume*

### Configuring nginx
Nginx will need to be told where to find the SSL certs

add `ssl_certificate` and `ssl_certificate_key` to your nginx and point them to the `fullchain.pem` and `privkey.pem`

like:
```
ssl_certificate /etc/letsencrypt/live/[your_domain]/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/[your_domain]/privkey.pem;
```
### For www domains
this needs to be repeated to work with www.[your_domain] and just [your_domain]

Repeat the steps above. But changing [your_domain] to www.[your_domain]
like:
```
ssl_certificate /etc/letsencrypt/live/www.[your_domain]/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/www.[your_domain]/privkey.pem;
```

### Redirecting http to https
Its a good idea to redirect http requests to https.

change the server that listens to port 80 to:
```conf
server {
  listen 80;
  listen [::]:80;
  server_name [your_domain] www.[your_domain];

  # Redirect HTTP to HTTPS
  return 301 https://$host$request_uri;
}
```

all together it should look like:
```conf
  server {
    listen 80;
    listen [::]:80;
    server_name [your_domain] www.[your_domain];

    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name [your_domain] www.[your_domain];

  ssl_certificate /etc/letsencrypt/live/[your_domain]/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/[your_domain]/privkey.pem;

  # Include the SSL settings you have

  location / {
      proxy_pass http://web:8000;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
  }

  # Other configurations go here
}

```

# The End
---