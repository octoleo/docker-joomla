# Joomla! Octoleo Docker Image

Welcome to the Joomla! Octoleo project, where we provide Docker images tailored for Joomla! enthusiasts and developers. These images are curated by @llewellynvdm and function as a playground for innovating and testing ideas that could eventually enhance the [official Joomla-Docker](https://github.com/joomla-docker/docker-joomla) images. While these images are experimental and not intended for production use, your feedback and contributions are highly valued.

[![Docker Pulls](https://img.shields.io/docker/pulls/llewellyn/joomla.svg)](https://hub.docker.com/r/llewellyn/joomla)

[![Joomla Logo](https://avatars.githubusercontent.com/u/69416061?s=200&v=4)](https://hub.docker.com/r/llewellyn/joomla)

## Quick Start

To get started, ensure you have [Docker Compose](https://docs.docker.com/compose/install/) installed.

### Setting Up Traefik as a Reverse Proxy

We recommend using [Traefik](https://doc.traefik.io/traefik/) for reverse proxying. Create a `docker-compose.yml` file for Traefik in a directory like `/home/username/Docker/traefik`.

Here's a basic Traefik configuration:

```yml
services:
  traefik:
    container_name: traefik
    image: "traefik:latest"
    command:
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --providers.docker
      - --log.level=ERROR
      - --certificatesresolvers.octoleoresolver.acme.httpchallenge=true
      - --certificatesresolvers.octoleoresolver.acme.email=joomla@yourdomain.com
      - --certificatesresolvers.octoleoresolver.acme.storage=/acme.json
      - --certificatesresolvers.octoleoresolver.acme.httpchallenge.entrypoint=web
      - --providers.file.directory=/conf
      - --providers.file.watch=true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "/home/username/Docker/traefik/conf:/conf"
      - "/home/username/Docker/traefik/acme.json:/acme.json"
    labels:
      - "traefik.http.routers.http-catchall.rule=hostregexp(`{host:.+}`)"
      - "traefik.http.routers.http-catchall.entrypoints=web"
      - "traefik.http.routers.http-catchall.middlewares=redirect-to-https"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
    networks:
      - traefik

networks:
  traefik:
    external: true
    name: traefik_webgateway
```

Deploy Traefik using:

```shell
cd /home/username/Docker/traefik
docker network create traefik_webgateway
docker-compose up -d
```

### Deploying Joomla

Choose the Joomla! Docker image version you need. Available tags include:

- `llewellyn/joomla:latest`
- `llewellyn/joomla:x.x`
- `llewellyn/joomla:x.x-phpx.x`
- `llewellyn/joomla:x.x-phpx.x-variant`

Find all tags here: [Docker Hub Tags](https://hub.docker.com/r/llewellyn/joomla/tags)

For Joomla setup, create a `docker-compose.yml` file in `/home/username/Docker/websitename`.

Example Joomla setup:

```yml
services:
  mariadb_websitename:
    image: mariadb:latest
    container_name: mariadb_websitename
    restart: unless-stopped
    environment:
      MARIADB_USER: octoleo
      MARIADB_DATABASE: octoleo
      MARIADB_PASSWORD: your_password_here
      MARIADB_ROOT_PASSWORD: your_root_password_here
    volumes:
      - '/home/username/Projects/websitename/db:/var/lib/mysql'
    networks:
      - traefik

  joomla_websitename:
    image: llewellyn/joomla:5.2
    container_name: joomla_websitename
    restart: unless-stopped
    environment:
      JOOMLA_SITE_NAME: Octoleo
      JOOMLA_ADMIN_USER: Octoleo
      JOOMLA_ADMIN_USERNAME: octoleo
      JOOMLA_ADMIN_PASSWORD: your_password_here
      JOOMLA_ADMIN_EMAIL: docker@octoleo.org
      JOOMLA_DB_HOST: mariadb_websitename:3306
      JOOMLA_DB_USER: octoleo
      JOOMLA_DB_NAME: octoleo
      JOOMLA_DB_PASSWORD: your_password_here
      JOOMLA_EXTENSIONS_URLS: https://git.vdm.dev/joomla/pkg-component-builder/archive/5.x.zip
    depends_on:
      - mariadb_websitename
    volumes:
      - '/home/username/Projects/websitename/website:/var/www/html'
    networks:
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.joomla_websitename.rule=Host(`octoleo.yourdomain.com`)"
      - "traefik.http.routers.joomla_websitename.entrypoints=websecure"
      - "traefik.http.services.joomla_websitename.loadbalancer.server.port=80"
      - "traefik.http.routers.joomla_websitename.service=joomla_websitename"
      - "traefik.http.routers.joomla_websitename.tls.certresolver=octoleoresolver"

  phpmyadmin_websitename:
    image: phpmyadmin/phpmyadmin
    container_name: phpmyadmin_websitename
    restart: unless-stopped
    environment:
      PMA_HOST: mariadb_websitename
      PMA_PORT: 3306
      UPLOAD_LIMIT: 300M
    depends_on:
      - mariadb_websitename
    networks:
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.phpmyadmin_websitename.rule=Host(`octoleo-db.yourdomain.com`)"
      - "traefik.http.routers.phpmyadmin_websitename.entrypoints=websecure"
      - "traefik.http.services.phpmyadmin_websitename.loadbalancer.server.port=80"
      - "traefik.http.routers.phpmyadmin_websitename.service=phpmyadmin_websitename"
      - "traefik.http.routers.phpmyadmin_websitename.tls.certresolver=octoleoresolver"

networks:
  traefik:
    external: true
    name: traefik_webgateway
```

Deploy Joomla using:

```shell
cd /home/username/Docker/websitename
docker-compose up -d
```

## Usage Scenarios

This setup is ideal for running multiple Joomla instances on the same server. It simplifies access and configuration, especially when paired with a domain service like [Cloudflare](https://cloudflare.com/).

- Use Cloudflare to point your domain to your server.
- Organize your server with two main directories: **Docker** for Docker Compose files and **Projects** for project-related files.

Configure Traefik once, and then add as many Joomla containers as needed. Access your Joomla site through your domain, secured with SSL.

Enjoy experimenting with Joomla on Docker!

### License

```
Copyright (C) 2021 Llewellyn van der Merwe. All rights reserved.
Licensed under the GNU General Public License version 2 or later; see LICENSE.txt
```
