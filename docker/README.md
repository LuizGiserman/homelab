# How to set this up via docker

## Creating env files

*ALL RELATIVE PATHS WILL HAVE THIS DIR (`./docker`) AS THE ORIGIN*

```
mkdir ~/traefik_files
```

```
touch base-services/.env
```

And edit `base-services/.env` so that you add the info below with your actual secrets.
```
##
# General Variables
##

TRAEFIK_FILES=~/traefik_files

##
# Container: traefik -- Traefik variables
##

## Only used if Cloudflare is selected in labels
# Don't remember if we need the CF_API_EMAIL and CF_API_KEY if we have the tokens below.
# Will remove it from this file if research and find out that we don't need it.

# Cloudflare account email
CF_API_EMAIL=YOUR_EMAIL_HERE
# Cloudflare global api key
CF_API_KEY=YOUR_CLOUDFLARE_API_KEY

CLOUDFLARE_ZONE_API_TOKEN=YOUR_CLOUDFLARE_TOKEN_WITH_ZONE_ACCESS
CLOUDFLARE_DNS_API_TOKEN=YOUR_CLOUDFLARE_TOKEN_WITH_DNS_ACCESS
```

Now we need to create some files in the `~/traefik_files` directory:

```
touch ~/traefik_files/acme.json
touch ~/traefik_files/traefik.toml
```

Now edit `~/traefik_files/traefik.toml` so that you add the info below with your e-mail for the SSL certificate.

```
[global]
  checkNewVersion = false
  sendAnonymousUsage = false

[entryPoints]
  [entryPoints.web]
    address = ":80"

  [entryPoints.websecure]
    address = ":443"

[providers.docker]

[certificatesResolvers.cloudflare.acme]
  email = "YOUR_EMAIL_HERE"
  [certificatesResolvers.cloudflare.acme.dnsChallenge]
    provider = "cloudflare"
    delayBeforeCheck = 0

[certificatesResolvers.letsencrypt.acme]
  email = "YOUR_EMAIL_HERE"
  storage = "acme.json"
  [certificatesResolvers.letsencrypt.acme.httpChallenge]
    entryPoint = "web"
```

Ok, now let's prepare the `arr` dir.

We need a place where we're going to store our media.
This means the place where we will download files and move them to their associated folder
In my case, I created this under `~/data/
We need folders for:
- movies
- tv
- downloads
- downloads-incomplete

so 
```
mkdir -p ~/data/movies ~/tv ~/data/downloads ~/data/downloads-incomplete
```

We also need to create paths for the config files of each of our services. I reference this later as `SERVICES_DIR` in the `.env` file and in the `docker-compose.yml`

```
mkdir -p ~/config/cleanuparr ~/config/deluge ~/config/flaresolverr ~/config/jellyfin ~/config/jellyseerr ~/config/prowlarr ~/config/radarr ~/config/sabnzbd ~/config/sonarr
```

And we need a cache for jellyfin. I decided to keep it in the `arr` dir.

```
mkdir arr/cache
```

Now for env variables

```
touch ~/arr/.env
```

And add this content to it, with your own values:

```
# Data location for storing media, downloads and configs
DATA_DIR=_SHOULD_
SERVICES_DIR=/home/luizgis/config

# Timezone
TIMEZONE=Europe/Paris

# Linux user/group ID for file permissions
## User ID
ENV_PUID=1000
## Group ID
ENV_PGID=1000
JELLYFIN_PUBLICHTTPS=true
JELLYFIN_PUBLICPORT=443

# Domain
DOMAIN=YOUR_DOMAIN_WITHOUT_THE_WWW.

# Subdomains for accessing services
SUB_DOMAIN_JELLYFIN=jellyfin
SUB_DOMAIN_QBITTORRENT=qbittorrent
SUB_DOMAIN_SONARR=sonarr
SUB_DOMAIN_RADARR=radarr
SUB_DOMAIN_PROWLARR=prowlarr
SUB_DOMAIN_JELLYSEERR=jellyseerr

# You can follow a tutorial on how to get a wireguard private key for your vpn provider.
WIREGUARD_PRIVATE_KEY=YOUR_WIREGUARD_PRIVATE_KEY
```

If you don't use `nordvpn`, make sure to change this info on the `arr/docker-compose.yml` file.

Now, everything should be ready to start, so:

```
cd base-services
docker-compose up -d
```

You should see portainer and traefik starting to run

Now for the arr stack:

```
cd arr
docker-compose up -d
```

You should see all of remaining services starting.

You can access them all via localhost on their ports, except for jellyfin and jellyseerr that should be accessed by your domain like `jellyseerr.yourdomain.com`