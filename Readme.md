## Introduction

Set-up a Nextcloud, Redis and MariaDB with Traefik reverse proxy, using Docker Compose.

**Prerequisites**

- Ubuntu 20.04 server(or any distro you want)
- [Docker](https://docs.docker.com/engine/install/) & [Docker-Compose](https://docs.docker.com/compose/install/) installed
- [Traefik](https://doc.traefik.io/traefik/getting-started/install-traefik/) installed
- Domain name

## Running the stack

### Clone the repository

```shell
git clone https://github.com/vitoyxyz/nextcloud-traefik && cd nextcloud-traefik
```

## You need edit

- MYSQL environment PASSWORDS
- TRUSTED_PROXIES(yor traefik container IP)
- Network name of traefik(here is proxy)
- you_domain.com with your own domain name

### Running the stack

```shell
docker-compose up -d
```

```yml
# docker-compose.yml

version: "3.7"

networks:
  proxy:
    external:
      name: proxy

services:
  db:
    image: mariadb
    container_name: nextcloud_db
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    restart: always

    volumes:
      - nextcloud_mariadb:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_PASSWORD=random_password_123
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
  redis:
    image: redis:latest
    container_name: nextcloud_redis
    restart: always
    volumes:
      - nextcloud_data:/data

  app:
    image: nextcloud
    container_name: nextcloud_app
    restart: always
    ports:
      - 8080:80
    networks:
      - proxy
      - default
    depends_on:
      - redis
      - db
    volumes:
      - nextcloud_data:/var/www/html
    environment:
      - MYSQL_PASSWORD=random_password_123
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
      - REDIS_HOST=redis
      - TRUSTED_PROXIES=172.33.0.0/16
      - NEXTCLOUD_TRUSTED_DOMAINS=drive.your_domain.com
    labels:
      - traefik.enable=true
      - traefik.protocol=http
      - traefik.docker.network=proxy
      - traefik.http.routers.nextcloud.middlewares=nextcloud-mid
      - traefik.http.routers.nextcloud.tls=true
      - traefik.http.routers.nextcloud.entrypoints=websecure
      - traefik.http.routers.nextcloud.tls.certresolver=letsencrypt
      - traefik.http.routers.nextcloud.rule=Host(`drive.your_domain.com`)
      - traefik.http.middlewares.nextcloud.headers.contentSecurityPolicy=frame-ancestors 'self' your_domain.com *.your_domain.com
      - traefik.http.middlewares.nextcloud.headers.stsSeconds=155520011
      - traefik.http.middlewares.nextcloud.headers.stsIncludeSubdomains=true
      - traefik.http.middlewares.nextcloud.headers.stsPreload=true
      - traefik.http.middlewares.nextcloud-mid.replacepathregex.regex=^/.well-known/ca(l|rd)dav
      - traefik.http.middlewares.nextcloud-mid.replacepathregex.replacement=/remote.php/dav/
  cron:
    image: nextcloud
    container_name: nextcloud_cron
    restart: always
    volumes:
      - nextcloud_data:/var/www/html
    networks:
      - proxy
      - default
    entrypoint: /cron.sh
    depends_on:
      - db
      - redis

volumes:
  nextcloud_mariadb:
  nextcloud_data:
```
