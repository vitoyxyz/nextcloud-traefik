## Running the stack

### You first need to create a volume for nextcloud

```shell
docker volume create nextcloud
```

or you can use [Portainer](https://github.com/vitoyxyz/traefik-portainer) GUI.

### Clone the repository

```shell
git clone https://github.com/vitoyxyz/nextcloud-traefik

```

### You need edit

- TRUSTED_PROXIES(yor traefik container IP)
- 'you_domain.com' with your own domain name

```yml
environment:
     ...
      - TRUSTED_PROXIES=172.33.0.0/16
      - NEXTCLOUD_TRUSTED_DOMAINS=drive.your_domain.com
     ...
      labels:
     ...
      - traefik.http.routers.nextcloud.rule=Host(`drive.your_domain.com`)
      - traefik.http.middlewares.nextcloud.headers.contentSecurityPolicy=frame-ancestors 'self' your_domain.com *.your_domain.com
     ...

```

### Running the stack

```shell
docker-compose up -d
```

```yml
# docker-compose.yml

version: "3.7"

networks:
  nextcloud:
  proxy:
    external:
      name: proxy

services:
  db:
    image: mariadb
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    restart: always
    volumes:
      - nextcloud:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=

  redis:
    image: redis:latest
    container_name: nextcloud_redis
    restart: always
    networks:
      - nextcloud
    volumes:
      - nextcloud:/var/lib/redis
  app:
    image: nextcloud
    container_name: nextcloud
    restart: always
    ports:
      - 5555:443
    networks:
      - nextcloud
      - proxy
    depends_on:
      - redis
      - db
    volumes:
      - nextcloud:/var/www/html
    environment:
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

volumes:
  nextcloud:
    external:
      name: nextcloud
```

## Cron job outside of the container (something is wrong with the image)

```shell
*/5  *  *  *  * docker exec -t -u www-data nextcloud php -f /var/www/html/cron.php > /dev/null 2>&1
```
