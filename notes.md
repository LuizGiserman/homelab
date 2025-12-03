## Setup traefik:

I first want to setup traefik running locally, to then move forward to running it with my domain. I don't want to expose any other service. Maybe just a hello world example.


```yml
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    ports:
      - 80:80
      - 443:443
    networks:
      - proxy
    command:
      - '--providers.docker=true'
      - '--providers.docker.exposedbydefault=false'
      - '--entrypoints.web.address=:80'
      - '--entrypoints.web.http.redirections.entryPoint.to=websecure'
      - '--entrypoints.web.http.redirections.entryPoint.scheme=https'
      - '--entrypoints.websecure.address=:443'
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"


networks:
  proxy:
    name: proxy
```

Getting this error : `ERROR: for 3fb3ca04cf07_traefik  'ContainerConfig'`

Trying this from traefik's docs:

```yml
services:
  traefik:
    image: "traefik:v3.4"
    container_name: "traefik"
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=proxy"
      - "--entryPoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"

  whoami:
    image: "traefik/whoami"
    restart: unless-stopped
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.docker.localhost`)"
      - "traefik.http.routers.whoami.entrypoints=web"

networks:
  proxy:
    name: proxy
```

This didn't work 
```shell
$ curl -H "Host: whoami.docker.localhost" http://localhost/`
```

It worked when I removed the 

```yml
security_opt:
    - no-new-privileges:true
```

I want to apply the middleware rules later.

Dude. I was actually exporting and port forwarding port 433 and not 443.

Let me try to run the one from jellyfin.
Didn't work. The DMC one worked !!

```shell
$ docker exec -it media-qbittorrent-1 /bin/sh
$ curl -4 ifconfig.me
$ 45.84.137.68
```