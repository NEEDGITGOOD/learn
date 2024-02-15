# webui_monitor

Create Reverse Proxy with:
- monitor (monitor.yourdomain.com)
- blog - wordpress (blog.yourdomain.com)
- db-admin (db-admin.yourdomain.com)

### Requirements

On your DNS Provider create A Records pointing to your Public IP (where you have this setup)
A Records:
- monitor.yourdomain.com
- blog.yourdomain.com
- db-admin.yourdomain.com

### Configure
#### Install Package
````bash
sudo apt-get install apache2-utils
````

#### Generate Password for Admin

````bash
htpasswd -nb admin secure_password
````

Output

````bash
admin:$apr1$ruca84Hq$mbjdMZBAG.KWn7vfN/SNK/
````

#### Define Ports to listen

````bash
nano traefik.toml
````

File:
````yaml
[entryPoints]
  [entryPoints.web]
    address = ":80"
    [entryPoints.web.http.redirections.entryPoint]
      to = "websecure"
      scheme = "https"

  [entryPoints.websecure]
    address = ":443"

[api]
  dashboard = true

[certificatesResolvers.lets-encrypt.acme]
  email = "your_email@your_domain"
  storage = "acme.json"
  [certificatesResolvers.lets-encrypt.acme.tlsChallenge]

[providers.docker]
  watch = true
  network = "web"

[providers.file]
  filename = "traefik_dynamic.toml"
````

- Change `your_email@your_domain` to your emailaddress.

#### Define User and Monitor Domain


````bash
nano traefik_dynamic.toml
````

File:
````yaml
[http.middlewares.simpleAuth.basicAuth]
  users = [
    "admin:$apr1$ruca84Hq$mbjdMZBAG.KWn7vfN/SNK/"
  ]

[http.routers.api]
  rule = "Host(`monitor.your_domain`)"
  entrypoints = ["websecure"]
  middlewares = ["simpleAuth"]
  service = "api@internal"
  [http.routers.api.tls]
    certResolver = "lets-encrypt"
````

- Change the "admin:hash" to prior Input
- Replace `monitor.your_domain` with your domain (i.e -> "monitor.randomdomain.com")


#### Create Docker Network

````bash
docker network create web
````

#### Create Cert File

````bash
touch acme.json
````

````bash
chmod 600 acme.json
````

### Run Docker

````bash
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $PWD/traefik.toml:/traefik.toml \
  -v $PWD/traefik_dynamic.toml:/traefik_dynamic.toml \
  -v $PWD/acme.json:/acme.json \
  -p 80:80 \
  -p 443:443 \
  --network web \
  --name traefik \
  traefik:v2.2
````


### Access traefik WebUI

Now go to https://monitor.your_domain/dashboard/

- Replace `https://monitor.your_domain/dashboard/` with your domain (i.e -> www.monitor.randomdomain.com/dashboard/)

### Create Docker Compose File

Create Docker Compose Files with Wordpress, MySQL, adminer Containers

````bash
nano docker-compose.yml
````

File:
````yaml
version: "3"

networks:
  web:
    external: true
  internal:
    external: false

services:
  blog:
    image: wordpress:4.9.8-apache
    environment:
      WORDPRESS_DB_PASSWORD:
    labels:
      - traefik.http.routers.blog.rule=Host(`blog.your_domain`)
      - traefik.http.routers.blog.tls=true
      - traefik.http.routers.blog.tls.certresolver=lets-encrypt
      - traefik.port=80
    networks:
      - internal
      - web
    depends_on:
      - mysql

  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD:
    networks:
      - internal
    labels:
      - traefik.enable=false

  adminer:
    image: adminer:4.6.3-standalone
    labels:
    labels:
      - traefik.http.routers.adminer.rule=Host(`db-admin.your_domain`)
      - traefik.http.routers.adminer.tls=true
      - traefik.http.routers.adminer.tls.certresolver=lets-encrypt
      - traefik.port=8080
    networks:
      - internal
      - web
    depends_on:
      - mysql
````

- Replace `db-admin.your_domain` with your domain (i.e -> "db-admin.randomdomain.com")

#### Set Environmental Variables for Docker Compose

````bash
export WORDPRESS_DB_PASSWORD=secure_database_password
export MYSQL_ROOT_PASSWORD=secure_database_password
````

#### Run Docker Compose

````bash
docker-compose up -d
````

### Access Wordpress & Adminer

Now go to https://blog.your_domain/

- Replace `https://blog.your_domain/dashboard/` with your domain (i.e -> www.blog.randomdomain.com/)

Now go to https://db-admin.your_domain/

Login: 
- Username:root
- Server: mysql
- Password: <the value you have set using `export MYSQL_ROOT_PASSWORD=secure_database_password`>
- Database: <empty>

That was it.

Source: https://www.digitalocean.com/community/tutorials/how-to-use-traefik-v2-as-a-reverse-proxy-for-docker-containers-on-ubuntu-20-04