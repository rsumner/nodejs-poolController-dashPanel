# nodejs-poolController-dashPanel
## What is nodejs-poolController-dashPanel?
dashPanel is a controller designed to operate using a [nodejs-poolController](https://github.com/tagyoureit/nodejs-poolController) server backend.  You will need to set up your nodejs-poolController server and have it communicating with your pool equipment prior to setting up this server.  Once you have done that you can set up the dashPanel to communicate with that server.

While this project was originally developed using an IntelliCenter control panel it should operate equally well with an IntelliTouch or EasyTouch control panel.
![image](https://user-images.githubusercontent.com/47839015/83304160-38a86780-a1b3-11ea-8214-442db6c6bdc4.png)

## Configuring the dashPanel
To configure the dashPanel you need to place the url for your [nodejs-poolController](https://github.com/tagyoureit/nodejs-poolController) server in the configuration.  Click the bars menu on the top left of the screen and fill in the ip address and port.  Then press the Apply button.  If this button is grayed out you will need to edit the config.json file manually and enter the settings under the services menu.

## What is Message Manager?
Message manager allows you to inspect your RS485 communication coming from and going to the [nodejs-poolController](https://github.com/tagyoureit/nodejs-poolController) server.  This tool decodes the messages and displays them in a manner where important chatter on the RS485 connection can be decoded while eliminating the chatter that don't matter.  Special filters can be applied to reduce the information to only the items you are interested in.
![image](https://user-images.githubusercontent.com/47839015/83314254-7a92d700-a1ce-11ea-8891-545db084624e.png)

## Quick Start (docker-compose)
Below is a minimal example running both the backend `nodejs-poolController` (service name `njspc`) and this dashPanel UI (service name `njspc-dash`). Adjust volumes and device mappings as needed. The dashPanel writes its configuration to `/app/config.json`, so we bind mount a host file to persist it. Additional runtime state (queues/uploads/logs) uses named volumes.

```yaml
services:
   njspc:
      image: ghcr.io/sam2kb/njspc
      container_name: njspc
      restart: unless-stopped
      environment:
         - TZ=${TZ:-UTC}
         - NODE_ENV=production
         # Serial vs network connection options
         # - POOL_NET_CONNECT=true
         # - POOL_NET_HOST=raspberrypi
         # - POOL_NET_PORT=9801
         # Provide coordinates so sunrise/sunset (heliotrope) works immediately - change as needed
         - POOL_LATITUDE=28.5383
         - POOL_LONGITUDE=-81.3792
      ports:
         - "4200:4200"
      devices:
         - /dev/ttyACM0:/dev/ttyUSB0
      # Persistence (create host directories/files first)
      volumes:
         - ./server-config.json:/app/config.json   # Persisted config file on host
         - njspc-data:/app/data                    # State & equipment snapshots
         - njspc-backups:/app/backups              # Backup archives
         - njspc-logs:/app/logs                    # Logs
         - njspc-bindings:/app/web/bindings/custom # Custom bindings
      # OPTIONAL: If you get permission errors accessing /dev/tty*, prefer adding the container user to the host dialout/uucp group;
      # only as a last resort temporarily uncomment the two lines below to run privileged/root (less secure).
      # privileged: true
      # user: "0:0"

   njspc-dash:
     image: ghcr.io/sam2kb/njspc-dash
     container_name: njspc-dash
     restart: unless-stopped
     depends_on:
       - njspc
     environment:
       - TZ=${TZ:-UTC}
       - NODE_ENV=production
       - POOL_WEB_SERVICES_IP=njspc      # Link to backend service name
     ports:
       - "5150:5150"
     volumes:
       - ./dash-config.json:/app/config.json
       - njspc-dash-data:/app/data
       - njspc-dash-logs:/app/logs
       - njspc-dash-uploads:/app/uploads

volumes:
  njspc-data:
  njspc-backups:
  njspc-logs:
  njspc-bindings:
  njspc-dash-data:
  njspc-dash-logs:
  njspc-dash-uploads:
```

After starting, browse to: `http://localhost:5150` and configure any remaining settings via the UI. The dashPanel will connect automatically to `njspc:4200` unless overridden.

## Persistence & Configuration
The application loads configuration from `/app/config.json` at startup and rewrites it atomically after changes (writes to a temporary file then renames). To persist across container recreations:

1. Create a host directory and seed the file (optional – if omitted, an empty file will be populated after first change):
  ```bash
  mkdir -p config
  docker run --rm ghcr.io/sam2kb/njspc-dash cat /app/config.json > config/config.json
  ```
2. Use the bind mount shown in the compose example: `./config/config.json:/app/config.json`.
3. If the mounted file is empty, defaults + environment overrides are applied and the file will be written once you change settings via the UI/API.

If a write is interrupted, the app can recover from a `.tmp` file; if corruption is detected the previous file is backed up as `config.json.corrupt` (when non-empty) and defaults are re-applied.

Environment variable overrides (new hierarchical form) include:
* `POOL_WEB_SERVICES_IP`
* `POOL_WEB_SERVICES_PORT`
* `POOL_WEB_SERVICES_PROTOCOL`
* `POOL_WEB_SERVERS_HTTP_PORT`, `POOL_WEB_SERVERS_HTTPS_PORT`
* `POOL_WEB_SERVERS_HTTP_ENABLED`, `POOL_WEB_SERVERS_HTTPS_ENABLED`

Legacy variables `POOL_HTTP_IP` and `POOL_HTTP_PORT` are still honored.

For production hardenings consider: enabling HTTPS, adding reverse proxy headers, mounting persistent volumes, and restricting exposed ports. Ensure ownership of the mounted `config.json` permits writes by the container user (UID 1000 in the official image); otherwise configuration changes will be disabled.

## Remote access
As configured in Quick Start above, the dashboard is only suitable to be used on your local network.  To secure the website for accessing remotely on the internet you will need to use a [reverse proxy](https://en.wikipedia.org/wiki/Reverse_proxy) that is conigured to use encryption and authentication.  There are several reverse proxies available, including [Nginx](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/) and [Caddy](https://caddyserver.com/docs/quick-starts/reverse-proxy), but for this example [YARP](https://dotnet.github.io/yarp/) will be used.

Let's Encrypt provides free SSL certificates that requires a domain name which can be obtained from [Duck DNS](https://www.duckdns.org/) after signup.  [WebAuthn](https://en.wikipedia.org/wiki/WebAuthn) enables strong authentication and is designed to enable passwordless login through hardware keys, biometrics (fingerprint/face), or mobile authenticators.  With these in place you can remotely access your dashboard in a secure manner over the internet.

You will need to modify the docker compose file that was previously setup and confirmed running under http://localhost:5150 and add the following additional services (retaining njspc & njspc-dash) and new volume (to existing volumes). You will need to replace values for `DUCKDNS_DOMAIN` & `DUCKDNS_TOKEN` with the appropriate details from your Duck DNS account. 

```yaml
services:
  ddns:
    image: docker.io/maksimstojkovic/duckdns:latest
    container_name: ddns
    environment:
      - DUCKDNS_DOMAIN=example.duckdns.org
      - DUCKDNS_TOKEN=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
      - DUCKDNS_DELAY=60
    restart: unless-stopped
  certs:
    image: docker.io/maksimstojkovic/letsencrypt:latest
    container_name: certs
    environment:
      - DUCKDNS_DOMAIN=example.duckdns.org
      - DUCKDNS_TOKEN=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
      - LETSENCRYPT_WILDCARD=true
    volumes:
      - proxy-config:/etc/letsencrypt
    restart: unless-stopped
  proxy:
    image: docker.io/mguinness/yarpwebauthn:latest
    container_name: proxy
    ports:
      - 8443:8443
    volumes:
      - proxy-config:/app/config
    restart: unless-stopped

volumes:
  proxy-config:
```

On the initial run you will need to edit the `customsettings.json` file located in the `proxy-config` volume with the following and replace `example.duckdns.org` with your domain.

```json
{
  "Kestrel": {
    "Certificates": {
      "Default": {
        "Path": "config/live/example.duckdns.org/fullchain.pem",
        "KeyPath": "config/live/example.duckdns.org/privkey.pem"
      }
    }
  },
  "ReverseProxy": {
    "Routes": {
      "route1": {
        "ClusterId": "cluster1",
        "AuthorizationPolicy": "default",
        "Match": {
          "Hosts": [ "njspc.example.duckdns.org" ],
          "Path": "{**catch-all}"
        }
      }
    },
    "Clusters": {
      "cluster1": {
        "Destinations": {
          "destination1": {
            "Address": "http://njspc-dash:5150/"
          }
        }
      }
    }
  },
  "Hosts": {
    "njspc.example.duckdns.org": {
    }
  }
}
```

Once running, the proxy will be available on TCP port 8443. Typically you would configure your home router to setup a [port forward](https://www.noip.com/support/knowledgebase/general-port-forwarding-guide) rule accepting TCP port 443 and forwarding to TCP port 8433 on the machine running the proxy. Then your website should be publicly (and securely) accessible at https://njspc.example.duckdns.org/ (substituting example with the custom domain name that you selected).

You will then need to register your security key as described in https://github.com/mguinness/YarpWebAuthn#usage.  If you have any problems or questions, please create an issue at https://github.com/mguinness/YarpWebAuthn for further assistance.
