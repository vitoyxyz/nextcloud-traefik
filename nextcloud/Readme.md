## Dependent on MariaDB as separate container and network.

First you need to start MariaDB container and create MariaDB network. insert here link!

## Running the stack

### You first need to create a volume for nextcloud

```
docker volume create nextcloud
```

or you can use Portainer GUI.

## Running the stack

```
$ docker-compose up -d
```

```yml
# docker-compose.yml

version: "3.7"

networks:
  nextcloud:
  mariadb:
    external:
      name: mariadb
  proxy:
    external:
      name: proxy

services:
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
      - mariadb
      - proxy
    depends_on:
      - redis
    volumes:
      - nextcloud:/var/www/html
    environment:
      - REDIS_HOST=redis
      - TRUSTED_PROXIES=172.33.0.0/16
      - NEXTCLOUD_TRUSTED_DOMAINS=drive.vitoy.xyz
    labels:
      - traefik.enable=true
      - traefik.protocol=http
      - traefik.docker.network=proxy
      - traefik.http.routers.nextcloud.middlewares=nextcloud-mid
      - traefik.http.routers.nextcloud.tls=true
      - traefik.http.routers.nextcloud.entrypoints=websecure
      - traefik.http.routers.nextcloud.tls.certresolver=letsencrypt
      - traefik.http.routers.nextcloud.rule=Host(`drive.vitoy.xyz`)
      - traefik.http.middlewares.nextcloud.headers.contentSecurityPolicy=frame-ancestors 'self' vitoy.xyz *.vitoy.xyz
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

## Cron job outside of the container something is wrong with the image it doesn't work

```
*/5  *  *  *  * docker exec -t -u www-data nextcloud php -f /var/www/html/cron.php > /dev/null 2>&1
```
