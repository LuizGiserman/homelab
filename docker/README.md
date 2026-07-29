# How to set this up via docker

## Setting up the base-services

*ALL RELATIVE PATHS WILL HAVE THIS DIR (`./docker`) AS THE ORIGIN*

```
mkdir ~/traefik_files
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

## Setting up the arr services

Ok, now let's prepare the `arr` dir.

We need a place to store our media.
This means the place where we'll download files to and then store them.
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
DATA_DIR=_SHOULD_BE_THE_DATA_DIR_PATH_YOU_CREATED
SERVICES_DIR=_SHOULD_BE_THE_CONFIG_DIR_PATH_YOU_CREATED

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

## Setting up the VPN stack (Headscale + Tailscale subnet router)

This gives you remote access into the LAN (and any internal-only services that
are *not* routed through Traefik) with zero extra inbound ports beyond the
Traefik one you already have forwarded on 80/443. `headscale` is a self-hosted
control server (same client software as Tailscale, just pointed at your own
server), and `tailscale-subnet-router` is a container on your Docker host that
joins your LAN into the tailnet.

First, the config directory for headscale:

```
mkdir ~/headscale_files
touch ~/headscale_files/config.yaml
```

Edit `~/headscale_files/config.yaml`. This is a minimal config — cross-check
it against the [official example](https://github.com/juanfont/headscale/blob/v0.23.0/config-example.yaml)
for the exact image tag you use, since the schema does change between
releases:

```yaml
server_url: https://headscale.YOUR_DOMAIN_WITHOUT_THE_WWW
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 127.0.0.1:9090

private_key_path: /var/lib/headscale/private.key
noise:
  private_key_path: /var/lib/headscale/noise_private.key

database:
  type: sqlite
  sqlite:
    path: /var/lib/headscale/db.sqlite

derp:
  server:
    enabled: false
  urls:
    - https://controlplane.tailscale.com/derpmap/default
  auto_update_enabled: true
  update_frequency: 24h

disable_check_updates: true
ephemeral_node_inactivity_timeout: 30m

dns:
  magic_dns: true
  base_domain: ts.YOUR_DOMAIN_WITHOUT_THE_WWW
  nameservers:
    global:
      - 1.1.1.1
      - 1.0.0.1

log:
  level: info
```

Now the `.env` for the stack:

```
touch vpn/.env
```

```
# Domain (same as your other stacks)
DOMAIN=YOUR_DOMAIN_WITHOUT_THE_WWW.
SUB_DOMAIN_HEADSCALE=headscale

# Path to the config directory created above
HEADSCALE_FILES=~/headscale_files

# CIDR of your real LAN, e.g. 192.168.1.0/24 -- this is what gets
# advertised into the tailnet so clients can reach internal-only services.
TAILSCALE_ADVERTISE_ROUTES=YOUR_LAN_CIDR

# Filled in after headscale is up and a preauth key has been created (see below).
# Leave blank for the very first `docker-compose up`.
TAILSCALE_AUTHKEY=
```

Add a DNS record for `headscale.yourdomain.com` pointing at your public IP,
same as your other subdomains.

One-time host change so the subnet router can actually forward LAN traffic:

```
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Now bring up just `headscale` first (the subnet router needs an auth key
from it before it can register):

```
cd vpn
docker-compose up -d headscale
```

Create a user and a preauth key:

```
docker exec headscale headscale users create me
docker exec headscale headscale preauthkeys create --user me --expiration 24h --reusable
```

Copy the printed key into `TAILSCALE_AUTHKEY` in `vpn/.env`, then start the
subnet router:

```
docker-compose up -d tailscale-subnet-router
```

Approve the advertised LAN route (it won't route traffic until you do this):

```
docker exec headscale headscale routes list
docker exec headscale headscale routes enable -r <ROUTE_ID_FROM_ABOVE>
```

### Connecting a laptop or phone

Install the official Tailscale client, then point it at your own control
server instead of Tailscale's:

- **Desktop/CLI**: `tailscale up --login-server=https://headscale.yourdomain.com --accept-routes`
  - If you didn't use an authkey, it prints a `headscale nodes register --user me --key nodekey:...`
    command — run that on the server to approve the device.
- **Mobile (iOS/Android)**: in the Tailscale app, before logging in, look for
  "use an alternate/custom coordination server" and enter
  `https://headscale.yourdomain.com`, then log in the same way (authkey or
  the register-on-server flow above). After connecting, make sure "use
  subnet routes" is enabled for this device in the app so LAN traffic
  actually gets routed through the tunnel.

Once connected, your device can reach anything on `TAILSCALE_ADVERTISE_ROUTES`
directly by its LAN IP — no Traefik route or public DNS entry needed for
internal-only services.

**Note on key expiry:** a preauth key's expiration only limits how long that
key is valid for *new* registrations — it does not force already-registered
devices to periodically re-authenticate. For actual recurring re-auth of
connected devices, headscale needs to be configured with OIDC login (e.g.
via Google, GitHub, or a self-hosted provider like Authentik) and an
`oidc.expiry` value, instead of (or alongside) preauth keys.