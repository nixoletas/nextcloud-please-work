# nextcloud-please-work

```
# Either select the MariaDB or Postgres configurations.

# MariaDB hosted on a Docker container.
version: '2'

volumes:
  nextcloud:
  db:
  redis:

services:
  # See
  # https://help.nextcloud.com/t/mariadb-upgrade-from-10-5-11-to-10-6-causes-internal-server-error/120585
  db:
    hostname: db
    image: mariadb:10.5.11
    restart: always
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    volumes:
      - ./db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=${DATABASE_ROOT_PASSWORD}
      - MYSQL_PASSWORD=${DATABASE_PASSWORD}
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud

  app:
    image: nextcloud:production
    restart: always
    ports:
      - 4005:80
    links:
      - db
      - redis
    volumes:
      - ${DATA_PATH}:/var/www/html/data
      - ${HTML_DATA_PATH}:/var/www/html
    environment:
      - REDIS_HOST=redis
      - REDIS_HOST_PORT=6379
      - MYSQL_PASSWORD=${DATABASE_PASSWORD}
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
      - APACHE_DISABLE_REWRITE_IP=1
      - TRUSTED_PROXIES=127.0.0.1
      - OVERWRITEHOST=${FQDN}
      - OVERWRITEPROTOCOL=https
      - OVERWRITECLIURL=https://${FQDN}

  redis:
    image: redis:6.2.5
    restart: always
    hostname: redis
    volumes:
      - redis:/var/lib/redis

# PostgreSQL in host.
version: '2'

volumes:
  nextcloud:
  redis:

services:
  app:
    image: nextcloud:production
    restart: always
    ports:
      - 4005:80
    links:
      - redis
    volumes:
      - ${DATA_PATH}:/var/www/html/data
      - ${HTML_DATA_PATH}:/var/www/html
    environment:
      - REDIS_HOST=redis
      - REDIS_HOST_PORT=6379
      - POSTGRES_DB=${DATABASE_NAME}
      - POSTGRES_USER=${DATABASE_USER}
      - POSTGRES_HOST=${DATABASE_HOST}
      - POSTGRES_PASSWORD=${DATABASE_PASSWORD}
      - APACHE_DISABLE_REWRITE_IP=1
      - TRUSTED_PROXIES=127.0.0.1
      - OVERWRITEHOST=${FQDN}
      - OVERWRITEPROTOCOL=https
      - OVERWRITECLIURL=https://${FQDN}

  redis:
    image: redis:6.2.5
    restart: always
    hostname: redis
    volumes:
      - redis:/var/lib/redis

# Scalable setup.
version: '2'

volumes:
  app:
  web:

services:
  web:
    image: nginx:1.23.1
    restart: always
    ports:
      - 127.0.0.1:4005:80
    links:
      - app
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    volumes_from:
      - app
    networks:
      - mynetwork

  app:
    image: nextcloud:24.0.5-fpm
    restart: always
    links:
      - clamav
    volumes:
      - ${DATA_PATH}:/var/www/html/data
      - ${HTML_DATA_PATH}:/var/www/html
      - ${PHP_CONFIG_DATA_PATH}:/usr/local/etc/php/conf.d
      - /var/run/redis.sock:/var/run/redis.sock
    environment:
      - TZ=Europe/Rome
      - REDIS_HOST=/var/run/redis.sock
      - REDIS_HOST_PORT=0
      - POSTGRES_DB=${DATABASE_NAME}
      - POSTGRES_USER=${DATABASE_USER}
      - POSTGRES_HOST=${DATABASE_HOST}
      - POSTGRES_PASSWORD=${DATABASE_PASSWORD}
      - APACHE_DISABLE_REWRITE_IP=1
      - TRUSTED_PROXIES=127.0.0.1
      - OVERWRITEHOST=${FQDN}
      - OVERWRITEPROTOCOL=https
      - OVERWRITECLIURL=https://${FQDN}
      - PHP_UPLOAD_LIMIT=8G
      - PHP_MEMORY_LIMIT=2G

  clamav:
    image: clamav/clamav:latest
    restart: always
    hostname: clamav

# PostgreSQL in host with elasticsearch.
version: '2'

volumes:
  nextcloud:
  redis:

services:
  app:
    image: nextcloud:production
    restart: always
    ports:
      - 4005:80
    links:
      - redis
      - elasticsearch

      # Enable this if you use Collabora
      # - collabora
    depends_on:
      - redis
      - elasticsearch

      # Enable this if you use Collabora
      # - collabora
    volumes:
      - ${DATA_PATH}:/var/www/html/data
      - ${HTML_DATA_PATH}:/var/www/html
    environment:
      - REDIS_HOST=redis
      - REDIS_HOST_PORT=6379
      - POSTGRES_DB=${DATABASE_NAME}
      - POSTGRES_USER=${DATABASE_USER}
      - POSTGRES_HOST=${DATABASE_HOST}
      - POSTGRES_PASSWORD=${DATABASE_PASSWORD}
      - APACHE_DISABLE_REWRITE_IP=1
      - TRUSTED_PROXIES=127.0.0.1
      - OVERWRITEHOST=${FQDN}
      - OVERWRITEPROTOCOL=https
      - OVERWRITECLIURL=https://${FQDN}

  elasticsearch:
    image: elasticsearch:7.17.7
    environment:
      - xpack.security.enabled=false
      - "discovery.type=single-node"
    hostname: elasticsearch
    volumes:
      # NOTE # enable the data volume when insructed! # NOTE #
      # - ${ES_DATA_PATH}:/usr/share/elasticsearch/data
      - ${ES_PLUGINS_PATH}:/usr/share/elasticsearch/plugins
    ports:
      - 9200:9200
    # Very important: elastic search will use all memory available
    # so you must set an appropriate value here.
    mem_limit: 16g

  redis:
    image: redis:6.2.5
    restart: always
    hostname: redis
    volumes:
      - redis:/var/lib/redis

  collabora:
    image: collabora/code:23.05.3.1.1
    restart: always
    ports:
      - 127.0.0.1:9980:9980
    environment:
      - domain=${NEXTCLOUD_FQDN}
      - server_name=${COLLABORA_FQDN}
      - dictionaries=de en es it
      - extra_params=--o:ssl.enable=true --o:ssl.termination=true
    cap_add:
      - MKNOD
      # WARNING: SYS_ADMIN (CAP_SYS_ADMIN): man 7 capabilities
      # This is very privileged!
      # See also
      # https://docs.docker.com/compose/compose-file/compose-file-v3/#cap>
      # https://github.com/CollaboraOnline/online/issues/779
      # - SYS_ADMIN
```
