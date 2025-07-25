# Reverse Proxy Documentation

> [!NOTE]
> Please note that AIO comes secured with TLS out-of-the-box. So you don't need to necessarily set up your own reverse proxy if you only want to run Nextcloud AIO which is much easier. See [the normal readme](https://github.com/nextcloud/all-in-one?tab=readme-ov-file#how-to-use-this) in that case. However if port 443 should already be used because you already run a web server or reverse proxy (like Apache, Nginx, Caddy, Cloudflare Tunnel and else), you need to follow this reverse proxy documentation to set up Nextcloud AIO.

> [!TIP]
> If you don't have a domain yet, [Tailscale is recommended](https://github.com/nextcloud/all-in-one/discussions/5439). If you don't have a reverse proxy yet, [Caddy is recommended](https://github.com/nextcloud/all-in-one/discussions/575).

## Introduction
In order to run Nextcloud behind a web server or reverse proxy (like Apache, Nginx, Caddy, Cloudflare Tunnel and else), you need to:
1. add a specific config to your web server or reverse proxy. [See the documentation below.](#1-configure-the-reverse-proxy)
2. specify the port that AIO's integrated Apache container shall use via the environmental variable `APACHE_PORT` (that runs inside its own container and published this port on the host) and adjust the `docker run` command of AIO. [See the documentation below.](#2-use-this-startup-command).
3. Open the AIO interface at port `8080` and type in and validate your domain. [See the documentation below.](#4-open-the-aio-interface)

Here one example with all reverse proxy settings for Linux:
```
sudo docker run \
--init \
--sig-proxy=false \
--name nextcloud-aio-mastercontainer \
--restart always \
--publish 8080:8080 \
--env APACHE_PORT=11000 \
--env APACHE_IP_BINDING=0.0.0.0 \
--env APACHE_ADDITIONAL_NETWORK="" \
--env SKIP_DOMAIN_VALIDATION=false \
--volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
--volume /var/run/docker.sock:/var/run/docker.sock:ro \
ghcr.io/nextcloud-releases/all-in-one:latest
```

<details>

<summary>Explanation of the command</summary>

- `sudo docker run` This command spins up a new docker container. Docker commands can optionally be used without `sudo` if the user is added to the docker group (this is not the same as docker rootless, see FAQ in the normal readme).
- `--init` This option makes sure that no zombie-processes are created, ever. See [the Docker documentation](https://docs.docker.com/reference/cli/docker/container/run/#init).
- `--sig-proxy=false` This option allows to exit the container shell that gets attached automatically when using `docker run` by using `[CTRL] + [C]` without shutting down the container.
- `--name nextcloud-aio-mastercontainer` This is the name of the container. This line is not allowed to be changed, since mastercontainer updates would fail.
- `--restart always` This is the "restart policy". `always` means that the container should always get started with the Docker daemon. See the Docker documentation for further detail about restart policies: https://docs.docker.com/config/containers/start-containers-automatically/
- `--publish 8080:8080` This means that port 8080 of the container should get published on the host using port 8080. This port is used for the AIO interface and uses a self-signed certificate by default. You can also use a different host port if port 8080 is already used on your host, for example `--publish 8081:8080` (only the first port can be changed for the host, the second port is for the container and must remain at 8080).
- `--env APACHE_PORT=11000` This is the port that is published on the host that runs Docker and Nextcloud AIO at which the reverse proxy should point at.
- `--env APACHE_IP_BINDING=0.0.0.0` This can be modified to allow access to the published port on the host only from certain ip-addresses. [See this documentation](#3-limit-the-access-to-the-apache-container)
- `--env APACHE_ADDITIONAL_NETWORK=""` This can be used to put the sibling apache container that is created by AIO into a specified network - useful if your reverse proxy runs as a container on the same host. [See this documentation](#adapting-the-sample-web-server-configurations-below)
- `--env SKIP_DOMAIN_VALIDATION=false` This can be set to `true` if the domain validation does not work and you are sure that you configured everything correctly after you followed [the debug documentation](#6-how-to-debug-things).
- `--volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config` This means that the files that are created by the mastercontainer will be stored in a docker volume that is called `nextcloud_aio_mastercontainer`. This line is not allowed to be changed, since built-in backups would fail later on.
- `--volume /var/run/docker.sock:/var/run/docker.sock:ro` The docker socket is mounted into the container which is used for spinning up all the other containers and for further features. It needs to be adjusted on Windows/macOS and on docker rootless. See the applicable documentation on this. If adjusting, don't forget to also set `WATCHTOWER_DOCKER_SOCKET_PATH`! If you dislike this, see https://github.com/nextcloud/all-in-one/tree/main/manual-install.
- `ghcr.io/nextcloud-releases/all-in-one:latest` This is the docker container image that is used.
- Further options can be set using environment variables, for example `--env NEXTCLOUD_DATADIR="/mnt/ncdata"` (This is an example for Linux. See [this](https://github.com/nextcloud/all-in-one#how-to-change-the-default-location-of-nextclouds-datadir) for other OS' and for an explanation of what this value does. This specific one needs to be specified upon the first startup if you want to change it to a specific path instead of the default Docker volume). To see explanations and examples for further variables (like changing the location of Nextcloud's datadir or mounting some locations as external storage into the Nextcloud container), read through this readme and look at the docker-compose file: https://github.com/nextcloud/all-in-one/blob/main/compose.yaml

</details>

> [!Note] 
> If you run into troubles, see [the debug section](#6-how-to-debug-things).

---

> [!IMPORTANT] 
> If you need HTTPS between Nextcloud and the reverse proxy because it is running on a different server in the same network, simply add another reverse proxy to the chain that runs on the same server like AIO and takes care of HTTPS proxying (most likely via self-signed certificates). Another option would be to create a VPN between the server that runs AIO and the server that runs the reverse proxy which takes care of encrypting the connection.

> [!NOTE]
> Since the Apache container gets created by the mastercontainer, there is **NO** way to provide custom docker labels or custom environmental variables for the Apache container. So please do not attempt to do this because it will fail!

## Content

The process to run Nextcloud behind a reverse proxy consists of at least steps 1, 2 and 4:
1. **Configure the reverse proxy! See [point 1](#1-configure-the-reverse-proxy)**
1. **Use this startup command! See [point 2](#2-use-this-startup-command)**
1. Optional: if the reverse proxy is installed on the same host and in the host network, you should limit the apache container to only listen on localhost. See [point 3](#3-limit-the-access-to-the-apache-container)
1. **Open the AIO interface. See [point 4](#4-open-the-aio-interface)**
1. Optional: get a valid certificate for the AIO interface! See [point 5](#5-optional-get-a-valid-certificate-for-the-aio-interface)
1. Optional: how to debug things? See [point 6](#6-how-to-debug-things)

## 1. Configure the reverse proxy

### Adapting the sample web server configurations below
1. Replace `<your-nc-domain>` with the domain on which you want to run Nextcloud.
1. Adjust the port `11000` to match your chosen `APACHE_PORT`.
1. Adjust `localhost` or `127.0.0.1` to point to the Nextcloud server IP or domain depending on where the reverse proxy is running. See the following options.

    <details>

    <summary>On the same server without a container</summary>

    For this setup, the default sample configurations with `localhost:$APACHE_PORT` should work.

    </details>

    <details>

    <summary>On the same server in a Docker container</summary>

    The reverse-proxy container needs to be connected to the nextcloud containers. This can be achieved one of these 3 ways:
    1. Utilize host networking instead of docker bridge networking: Specify `--network host` option (or `network_mode: host` for docker-compose) as setting for the reverse proxy container to connect it to the host network. If you are using a firewall on the server, you need to open ports 80 and 443 for the reverse proxy manually. With this setup, the default sample configurations with reverse-proxy pointing to `localhost:$APACHE_PORT` should work directly.
    1. Connect nextcloud's external-facing containers to the reverse-proxy's docker network by specifying env variable APACHE_ADDITIONAL_NETWORK. With this setup, the reverse proxy can utilize Docker bridge network's DNS name resolution to access nextcloud at `http://nextcloud-aio-apache:$APACHE_PORT`. ⚠️⚠️⚠️ Note, the specified network must already exist before Nextcloud AIO is started. Otherwise it will fail to start the container because the network is not existing.
    1. Connect the reverse-proxy container to the `nextcloud-aio` network by specifying it as a secondary (external) network for the reverse proxy container. With this setup also, the reverse proxy can utilize Docker bridge network's DNS name resolution to access nextcloud at `http://nextcloud-aio-apache:$APACHE_PORT` .

    </details>

    <details>

    <summary>On a different server (in container or not)</summary>

    Use the private ip-address of the host that shall be running AIO. So e.g. `private.ip.address.of.aio.server:$APACHE_PORT` instead of `localhost:$APACHE_PORT`.
    
    If you are not sure how to retrieve that, you can run: `ip a | grep "scope global" | head -1 | awk '{print $2}' | sed 's|/.*||'` on the server that shall be running AIO (the commands only work on Linux).

    </details>

### Apache

<details>

<summary>click here to expand</summary>

**Disclaimer:** It might be possible that the config below is not working 100% correctly, yet. Improvements to it are very welcome!

Add this as a new Apache site config:

(The config below assumes that you are using certbot to get your certificates. You need to create them first in order to make it work.)

```
<VirtualHost *:80>
    ServerName <your-nc-domain>

    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
    RewriteCond %{SERVER_NAME} =<your-nc-domain>
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>

<VirtualHost *:443>
    ServerName <your-nc-domain>

    # Reverse proxy based on https://httpd.apache.org/docs/current/mod/mod_proxy_wstunnel.html
    RewriteEngine On
    ProxyPreserveHost On
    RequestHeader set X-Real-IP %{REMOTE_ADDR}s
    AllowEncodedSlashes NoDecode
    
    # Adjust the two lines below to match APACHE_PORT and APACHE_IP_BINDING. See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#adapting-the-sample-web-server-configurations-below
    ProxyPass / http://localhost:11000/ nocanon
    ProxyPassReverse / http://localhost:11000/
    
    RewriteCond %{HTTP:Upgrade} websocket [NC]
    RewriteCond %{HTTP:Connection} upgrade [NC]
    RewriteCond %{THE_REQUEST} "^[a-zA-Z]+ /(.*) HTTP/\d+(\.\d+)?$"
    RewriteRule .? "ws://localhost:11000/%1" [P,L,UnsafeAllow3F] # Adjust to match APACHE_PORT and APACHE_IP_BINDING. See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#adapting-the-sample-web-server-configurations-below

    # Enable h2, h2c and http1.1
    Protocols h2 h2c http/1.1
    
    # Solves slow upload speeds caused by http2
    H2WindowSize 5242880

    # TLS
    SSLEngine               on
    SSLProtocol             -all +TLSv1.2 +TLSv1.3
    SSLCipherSuite          ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305
    SSLHonorCipherOrder     off
    SSLSessionTickets       off

    # If running apache on a subdomain (eg. nextcloud.example.com) of a domain that already has an wildcard ssl certificate from certbot on this machine, 
    # the <your-nc-domain> in the below lines should be replaced with just the domain (eg. example.com), not the subdomain. 
    # In this case the subdomain should already be secured without additional actions
    SSLCertificateFile /etc/letsencrypt/live/<your-nc-domain>/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/<your-nc-domain>/privkey.pem

    # Disable HTTP TRACE method.
    TraceEnable off
    <Files ".ht*">
        Require all denied
    </Files>

    # Support big file uploads
    LimitRequestBody 0
    Timeout 86400
    ProxyTimeout 86400
</VirtualHost>
```

⚠️ **Please note:** Look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.

To make the config work you can run the following command:
`sudo a2enmod rewrite proxy proxy_http proxy_wstunnel ssl headers http2`

</details>

### Caddy (recommended)

<details>

<summary>click here to expand</summary>

**Hint:** You may have a look at [this guide](https://github.com/nextcloud/all-in-one/discussions/575#discussion-4055615) for a more complete but possibly outdated example.

Add this to your Caddyfile:

```
https://<your-nc-domain>:443 {
    reverse_proxy localhost:11000 # Adjust to match APACHE_PORT and APACHE_IP_BINDING. See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#adapting-the-sample-web-server-configurations-below
}
```
The Caddyfile is a text file called `Caddyfile` (no extension) which – if you should be running Caddy inside a container – should usually be created in the same location as your `compose.yaml` file prior to starting the container.

⚠️ **Please note:** look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.

**Advice:** You may have a look at [this](https://github.com/nextcloud/all-in-one/discussions/575#discussion-4055615) for a more complete example.

</details>

### Caddy with ACME DNS-challenge

<details>

<summary>click here to expand</summary>

You can get AIO running using the ACME DNS-challenge. Here is how to do it.

1. Follow [this documentation](https://caddy.community/t/how-to-use-dns-provider-modules-in-caddy-2/8148) in order to get a Caddy build that is compatible with your domain provider's DNS challenge.
1. Add this to your Caddyfile:
    ```
    https://<your-nc-domain>:443 {
        reverse_proxy localhost:11000 # Adjust to match APACHE_PORT and APACHE_IP_BINDING. See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#adapting-the-sample-web-server-configurations-below
        tls {
            dns <provider> <key>
        }
    }
    ```
    ⚠️ **Please note:** Look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.

    You also need to adjust `<provider>` and `<key>` to match your case.

1. Now continue with [point 2](#2-use-this-startup-command) but additionally, add `--env SKIP_DOMAIN_VALIDATION=true` to the docker run command of the mastercontainer (but before the last line `ghcr.io/nextcloud-releases/all-in-one:latest`) which will disable the domain validation (because it is known that the domain validation will not work when using the DNS-challenge since no port is publicly opened).

**Advice:** In order to make it work in your home network, you may add the internal ipv4-address of your reverse proxy as A DNS-record to your domain and disable the dns-rebind-protection in your router. Another way it to set up a local dns-server like a pi-hole and set up a custom dns-record for that domain that points to the internal ip-adddress of your reverse proxy (see https://github.com/nextcloud/all-in-one#how-can-i-access-nextcloud-locally). If both is not possible, you may add the domain to the hosts file which is needed then for any devices that shall use the server.

</details>

### OpenLiteSpeed

<details>

<summary>click here to expand</summary>

You can find the OpenLiteSpeed reverse proxy guide by @MorrowShore here: https://github.com/nextcloud/all-in-one/discussions/6370

</details>

### Citrix ADC VPX / Citrix Netscaler

<details>

<summary>click here to expand</summary>

For a reverse proxy example guide for Citrix ADC VPX / Citrix Netscaler, see this guide by @esmith443: https://github.com/nextcloud/all-in-one/discussions/2452

</details>

### Cloudflare Tunnel

<details>

<summary>click here to expand</summary>


**Hint:** You may have a look at [this guide](https://github.com/nextcloud/all-in-one/discussions/2845#discussioncomment-6423237) for a more complete but possibly outdated example.

Although it does not seem like it is the case but from AIO perspective a Cloudflare Tunnel works like a reverse proxy. Please see the [caveats](https://github.com/nextcloud/all-in-one#notes-on-cloudflare-proxytunnel) before proceeding. Here is then how to make it work:

1. Install the Cloudflare Tunnel on the same machine where AIO will be running on and point the Tunnel with the domain that you want to use for AIO to `http://localhost:11000`.<br>
⚠️ **Please note:** look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.
1. Now continue with [point 2](#2-use-this-startup-command) but add `--env SKIP_DOMAIN_VALIDATION=true` to the docker run command - which will disable the domain validation (because it is known that the domain validation will not work behind a Cloudflare Tunnel).

**Advice:** Make sure to [disable Cloudflare's Rocket Loader feature](https://help.nextcloud.com/t/login-page-not-working-solved/149417/8) as otherwise Nextcloud's login prompt will not be shown.

</details>

### HaProxy

<details>

<summary>click here to expand</summary>

**Disclaimer:** It might be possible that the config below is not working 100% correctly, yet. Improvements to it are very welcome!

Here is an example HaProxy config:

```
global
    chroot                      /var/haproxy
    log                         /var/run/log audit debug
    lua-prepend-path            /tmp/haproxy/lua/?.lua
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

    ssl-default-server-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305
    ssl-default-server-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-server-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
    log     global
    option redispatch -1
    retries 3
    default-server init-addr last,libc

# Frontend: LetsEncrypt_443 ()
frontend LetsEncrypt_443
    bind 0.0.0.0:443 name 0.0.0.0:443 ssl crt-list /tmp/haproxy/ssl/605f6609f106d1.17683543.certlist 
    mode http
    option http-keep-alive
    default_backend acme_challenge_backend
    option forwardfor
    # tuning options
    timeout client 30s

    # logging options
    # ACL: find_acme_challenge
    acl acl_605f6d4b6453d2.03059920 path_beg -i /.well-known/acme-challenge/
    # ACL: Nextcloud
    acl acl_60604e669c3ca4.13013327 hdr(host) -i <your-nc-domain>

    # ACTION: redirect_acme_challenges
    use_backend acme_challenge_backend if acl_605f6d4b6453d2.03059920
    # ACTION: Nextcloud
    use_backend Nextcloud if acl_60604e669c3ca4.13013327


# Frontend: LetsEncrypt_80 ()
frontend LetsEncrypt_80
    bind 0.0.0.0:80 name 0.0.0.0:80 
    mode tcp
    default_backend acme_challenge_backend
    # tuning options
    timeout client 30s

    # logging options
    # ACL: find_acme_challenge
    acl acl_605f6d4b6453d2.03059920 path_beg -i /.well-known/acme-challenge/

    # ACTION: redirect_acme_challenges
    use_backend acme_challenge_backend if acl_605f6d4b6453d2.03059920

# Frontend (DISABLED): 1_HTTP_frontend ()

# Frontend (DISABLED): 1_HTTPS_frontend ()

# Frontend (DISABLED): 0_SNI_frontend ()

# Backend: acme_challenge_backend (Added by Let's Encrypt plugin)
backend acme_challenge_backend
    # health checking is DISABLED
    mode http
    balance source
    # stickiness
    stick-table type ip size 50k expire 30m  
    stick on src
    # tuning options
    timeout connect 30s
    timeout server 30s
    http-reuse safe
    server acme_challenge_host 127.0.0.1:43580 

# Backend: Nextcloud ()
backend Nextcloud
    mode http
    balance source
    server Nextcloud localhost:11000 # Adjust to match APACHE_PORT and APACHE_IP_BINDING. See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#adapting-the-sample-web-server-configurations-below
```

⚠️ **Please note:** Look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.

</details>

### Nginx, Freenginx, Openresty, Angie

<details>

<summary>click here to expand</summary>

**Hint:** You may have a look at [this guide](https://github.com/nextcloud/all-in-one/discussions/588#discussioncomment-2811152) for a more complete but possibly outdated example.

**Disclaimer:** This config was tested and should normally work on all modern Nginx versions. Improvements to the config are very welcome!

Add the below template to your Nginx config.

**Note:** please check your Nginx version by running: `nginx -v` and adjust the lines marked with version notes to fit your version.

```
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen 80;
    listen [::]:80;            # comment to disable IPv6

    if ($scheme = "http") {
        return 301 https://$host$request_uri;
    }
    if ($http_x_forwarded_proto = "http") {
        return 301 https://$host$request_uri;
    }

    listen 443 ssl http2;      # for nginx versions below v1.25.1
    listen [::]:443 ssl http2; # for nginx versions below v1.25.1 - comment to disable IPv6

    # listen 443 ssl;      # for nginx v1.25.1+
    # listen [::]:443 ssl; # for nginx v1.25.1+ - keep comment to disable IPv6
    # http2 on;            # uncomment to enable HTTP/2 - supported on nginx v1.25.1+

    # listen 443 quic reuseport;       # uncomment to enable HTTP/3 / QUIC - supported on nginx v1.25.0+ - please remove "reuseport" if there is already another quic listener on port 443 with enabled reuseport
    # listen [::]:443 quic reuseport;  # uncomment to enable HTTP/3 / QUIC - supported on nginx v1.25.0+ - please remove "reuseport" if there is already another quic listener on port 443 with enabled reuseport - keep comment to disable IPv6
    # http3 on;                                 # uncomment to enable HTTP/3 / QUIC - supported on nginx v1.25.0+
    # quic_gso on;                              # uncomment to enable HTTP/3 / QUIC - supported on nginx v1.25.0+
    # quic_retry on;                            # uncomment to enable HTTP/3 / QUIC - supported on nginx v1.25.0+
    # quic_bpf on;                              # improves  HTTP/3 / QUIC - supported on nginx v1.25.0+, if nginx runs as a docker container you need to give it privileged permission to use this option
    # add_header Alt-Svc 'h3=":443"; ma=86400'; # uncomment to enable HTTP/3 / QUIC - supported on nginx v1.25.0+

    proxy_buffering off;
    proxy_request_buffering off;

    client_max_body_size 0;
    client_body_buffer_size 512k;
    # http3_stream_buffer_size 512k; # uncomment to enable HTTP/3 / QUIC - supported on nginx v1.25.0+
    proxy_read_timeout 86400s;

    server_name <your-nc-domain>;

    location / {
        proxy_pass http://127.0.0.1:11000$request_uri; # Adjust to match APACHE_PORT and APACHE_IP_BINDING. See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#adapting-the-sample-web-server-configurations-below

        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header Early-Data $ssl_early_data;

        # Websocket
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }

    # If running nginx on a subdomain (eg. nextcloud.example.com) of a domain that already has an wildcard ssl certificate from certbot on this machine, 
    # the <your-nc-domain> in the below lines should be replaced with just the domain (eg. example.com), not the subdomain. 
    # In this case the subdomain should already be secured without additional actions
    ssl_certificate /etc/letsencrypt/live/<your-nc-domain>/fullchain.pem;   # managed by certbot on host machine
    ssl_certificate_key /etc/letsencrypt/live/<your-nc-domain>/privkey.pem; # managed by certbot on host machine

    ssl_dhparam /etc/dhparam; # curl -L https://ssl-config.mozilla.org/ffdhe2048.txt -o /etc/dhparam

    ssl_early_data on;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ecdh_curve x25519:x448:secp521r1:secp384r1:secp256r1;

    ssl_prefer_server_ciphers on;
    ssl_conf_command Options PrioritizeChaCha;
    ssl_ciphers TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-RSA-AES128-GCM-SHA256;
}

```

⚠️ **Please note:** look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.

</details>

### NPMplus (Fork of Nginx-Proxy-Manager - NPM)

<details>

<summary>click here to expand</summary>

⚠️ **Please note:** This is not needed when running NPMplus as a community container.

First, make sure the environmental variables `PUID` and `PGID` in the `compose.yaml` file for NPM are either unset or set to `0`. <br>
If you need to change the GID/PID then please add `net.ipv4.ip_unprivileged_port_start=0` at the end of `/etc/sysctl.conf`. <br>
Note: this will cause that a non root user can bind privileged ports.

Second, see these screenshots for a working config:

![grafik](https://github.com/user-attachments/assets/c32c8fe8-7417-4f8f-9625-24b95651e630)

![grafik](https://github.com/user-attachments/assets/f14bba5c-69ce-4514-a2ac-5e5d7fb97792)

<!-- ![grafik](https://github.com/user-attachments/assets/a26c53fd-6cc8-4a6b-a86f-c2f94b70088f) -->

![grafik](https://github.com/user-attachments/assets/75d7f539-35d1-4a3e-8c51-43123f698893)

![grafik](https://github.com/user-attachments/assets/e494edb5-8b70-4d45-bc9b-374219230041)

`proxy_set_header Accept-Encoding $http_accept_encoding;`

⚠️ **Please note:** Nextcloud will complain that X-XXS-Protection is set to the wrong value, this is intended by NPMplus. <br>
⚠️ **Please note:** look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.

</details>

### Nginx-Proxy-Manager - NPM

<details>

<summary>click here to expand</summary>

**Hint:** You may have a look at [this guide](https://github.com/nextcloud/all-in-one/discussions/588#discussioncomment-3040493) for a more complete but possibly oudated example.

First, make sure the environmental variables `PUID` and `PGID` in the `compose.yaml` file for NPM are either unset or set to `0`. <br>
If you need to change the GID/PID then please add `net.ipv4.ip_unprivileged_port_start=0` at the end of `/etc/sysctl.conf`. <br>
Note: this will cause that a non root user can bind privileged ports.

Second, see these screenshots for a working config:

![grafik](https://user-images.githubusercontent.com/75573284/213889707-b7841ca0-3ea7-4321-acf6-50e1c1649442.png)

![grafik](https://user-images.githubusercontent.com/75573284/213889724-1ab32264-3e0c-4d83-b067-9fe9d1672fb2.png)

![grafik](https://github.com/nextcloud/all-in-one/assets/24786786/fecbb5ef-d2f4-4e0f-bc4b-82207e2c2809)

![grafik](https://user-images.githubusercontent.com/75573284/213889746-87dbe8c5-4d1f-492f-b251-bbf82f1510d0.png)

```
client_body_buffer_size 512k;
proxy_read_timeout 86400s;
client_max_body_size 0;
```

⚠️ **Please note:** look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.
Also change `<you>@<your-mail-provider-domain>` to a mail address of yours.

</details>

### Nginx-Proxy

<details>

<summary>click here to expand</summary>

Unfortunately, it is not possible to configure Nginx-proxy in a way that works because it completely relies on environmental variables of the docker containers itself. Providing these variables does not work as stated above.

If you really want to use AIO, we recommend you to switch to caddy. It is simply amazing!<br>

Apart from that, there is a [manual-install](https://github.com/nextcloud/all-in-one/tree/main/manual-install).

</details>

### Node.js with Express

<details>

<summary>click here to expand</summary>

**Disclaimer:** it might be possible that the config below is not working 100% correctly, yet. Improvements to it are very welcome!

For Node.js, we will use the npm package `http-proxy`. WebSockets must be handled separately.

This example only uses `http`, but if your Express server already uses a `https` server, then follow the same instructions for `https`.

```js
const HttpProxy = require('http-proxy');
const express = require('express');
const http = require('http');

const app = express();
const proxy = HttpProxy.createProxyServer({
	target: 'http://localhost:11000', // Adjust to match APACHE_PORT and APACHE_IP_BINDING. See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#adapting-the-sample-web-server-configurations-below
	// Timeout can be changed to your liking.
	timeout: 1000 * 60 * 3,
	proxyTimeout: 1000 * 60 * 3,
	// Not 100% certain whether autoRewrite is necessary, but enabling it SEEMS to make it behave more stably.
	autoRewrite: true,
	// Do not enable followRedirects.
	followRedirects: false,
});

// Handle errors with proxy.web and proxy.ws
function onProxyError(err, req, res, target) {
	// Handle errors however you like. Here's an example:
	if (err.code === 'ECONNREFUSED') {
		return res.status(503).send('Nextcloud server is currently not running. It may be down for temporary maintenance.');
	}
	// other errors
	else {
		console.error(err);
		return res.status(500).send(String(err));
	}
}

app.use((req, res) => {
	proxy.web(req, res, {}, onProxyError);
});

const httpServer = http.createServer(app);
httpServer.listen('80');

// Listen for an upgrade to a WebSocket connection.
httpServer.on('upgrade', (req, socket, head) => {
	proxy.ws(req, socket, head, {}, onProxyError);
});
```

If you are using the Express package `vhost` for your app, you can use `proxy.web` inside the vhosted express function (see the following code snippet), but `proxy.ws` still needs to be done "globally" on your http server. Nextcloud should automatically ignore websocket requests for other domains.

```js
const HttpProxy = require('http-proxy');
const express = require('express');
const http = require('http');

const myNextcloudApp = express();
const myOtherApp = express();
const vhost = express();

// Definitions for proxy and onProxyError unchanged. (see above)

myNextcloudApp.use((req, res) => {
	proxy.web(req, res, {}, onProxyError);
});

vhost.use(vhostFunc('<your-nc-domain>', myNextcloudApp));

const httpServer = http.createServer(app);
httpServer.listen('80');

// Listen for an upgrade to a WebSocket connection.
httpServer.on('upgrade', (req, socket, head) => {
	proxy.ws(req, socket, head, {}, onProxyError);
});
```

⚠️ **Please note:** look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.

</details>

### Synology Reverse Proxy

<details>

<summary>click here to expand</summary>

**Disclaimer:** it might be possible that the config below is not working 100% correctly, yet. Improvements to it are very welcome!

See these screenshots for a working config:

![image](https://user-images.githubusercontent.com/89748315/192525606-48cab54b-866e-4964-90a8-15e71bd362fb.png)

![image](https://user-images.githubusercontent.com/70434961/213193789-fa936edc-e307-4e6a-9a53-ae26d1bf2f42.jpg)

⚠️ **Please note:** look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.

</details>

### Traefik 2

<details>

<summary>click here to expand</summary>

**Hint:** You may have a look at [this video](https://www.youtube.com/watch?v=VLPSRrLMDmA) for a more complete but possibly outdated example.

**Disclaimer:** it might be possible that the config below is not working 100% correctly, yet. Improvements to it are very welcome!

Traefik's building blocks (router, service, middlewares) need to be defined using dynamic configuration similar to [this](https://doc.traefik.io/traefik/providers/file/#configuration-examples) official Traefik configuration example. Using **docker labels _won't work_** because of the nature of the project.

The examples below define the dynamic configuration in YAML files. If you rather prefer TOML, use a YAML to TOML converter.

1. In Traefik's static configuration define a [file provider](https://doc.traefik.io/traefik/providers/file/) for dynamic providers:

    ```yml
    # STATIC CONFIGURATION
   
    entryPoints:
      https:
        address: ":443" # Create an entrypoint called "https" that uses port 443
        # If you want to enable HTTP/3 support, uncomment the line below
        # http3: {}
    
    certificatesResolvers:
      # Define "letsencrypt" certificate resolver
      letsencrypt:
        acme:
          storage: /letsencrypt/acme.json # Defines the path where certificates should be stored
          email: <your-email-address> # Where LE sends notification about certificates expiring
          tlschallenge: true
   
    providers:
      file:
        directory: "/path/to/dynamic/conf" # Adjust the path according your needs.
        watch: true

    # Enable HTTP/3 feature by uncommenting the lines below. Don't forget to route 443 UDP to Traefik (Firewall\NAT\Traefik Container)
    # experimental:
      # http3: true
    ```

1. Declare the router, service and middlewares for Nextcloud in `/path/to/dynamic/conf/nextcloud.yml`:

    ```yml
    http:
      routers:
        nextcloud:
          rule: "Host(`<your-nc-domain>`)"
          entrypoints:
            - "https"
          service: nextcloud
          middlewares:
            - nextcloud-chain
          tls:
            certresolver: "letsencrypt"

      services:
        nextcloud:
          loadBalancer:
            servers:
              - url: "http://localhost:11000" # Adjust to match APACHE_PORT and APACHE_IP_BINDING. See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#adapting-the-sample-web-server-configurations-below

      middlewares:
        nextcloud-secure-headers:
          headers:
            hostsProxyHeaders:
              - "X-Forwarded-Host"
            referrerPolicy: "same-origin"

        https-redirect:
          redirectscheme:
            scheme: https 

        nextcloud-chain:
          chain:
            middlewares:
              # - ... (e.g. rate limiting middleware)
              - https-redirect
              - nextcloud-secure-headers
    ```

---

⚠️ **Please note:** look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.

</details>

### Traefik 3

<details>

<summary>click here to expand</summary>

**Disclaimer:** it might be possible that the config below is not working 100% correctly, yet. Improvements to it are very welcome!

Traefik's building blocks (router, service, middlewares) need to be defined using dynamic configuration similar to [this](https://doc.traefik.io/traefik/providers/file/#configuration-examples) official Traefik configuration example. Using **docker labels _won't work_** because of the nature of the project.

The examples below define the dynamic configuration in YAML files. If you rather prefer TOML, use a YAML to TOML converter.

1. In Traefik's static configuration define a [file provider](https://doc.traefik.io/traefik/providers/file/) for dynamic providers:

    ```yml
    # STATIC CONFIGURATION
   
    entryPoints:
      https:
        address: ":443" # Create an entrypoint called "https" that uses port 443
        # If you want to enable HTTP/3 support, uncomment the line below
        # http3: {}
    
    certificatesResolvers:
      # Define "letsencrypt" certificate resolver
      letsencrypt:
        acme:
          storage: /letsencrypt/acme.json # Defines the path where certificates should be stored
          email: <your-email-address> # Where LE sends notification about certificates expiring
          tlschallenge: true
   
    providers:
      file:
        directory: "/path/to/dynamic/conf" # Adjust the path according your needs.
        watch: true
    ```

2. Declare the router, service and middlewares for Nextcloud in `/path/to/dynamic/conf/nextcloud.yml`:

    ```yml
    http:
      routers:
        nextcloud:
          rule: "Host(`<your-nc-domain>`)"
          entrypoints:
            - "https"
          service: nextcloud
          middlewares:
            - nextcloud-chain
          tls:
            certresolver: "letsencrypt"

      services:
        nextcloud:
          loadBalancer:
            servers:
              - url: "http://localhost:11000" # Adjust to match APACHE_PORT and APACHE_IP_BINDING. See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#adapting-the-sample-web-server-configurations-below

      middlewares:
        nextcloud-secure-headers:
          headers:
            hostsProxyHeaders:
              - "X-Forwarded-Host"
            referrerPolicy: "same-origin"

        https-redirect:
          redirectscheme:
            scheme: https 

        nextcloud-chain:
          chain:
            middlewares:
              # - ... (e.g. rate limiting middleware)
              - https-redirect
              - nextcloud-secure-headers
    ```

---

⚠️ **Please note:** look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.

</details>

### IIS with ARR and URL Rewrite

<details>

<summary>click here to expand</summary>

**Disclaimer:** It might be possible that the config below is not working 100% correctly, yet. Improvements to it are very welcome!

**Please note:** Using IIS as a reverse proxy comes with some limitations:
- A maximum upload size of 4GiB, in the example configuration below the limit is set to 2GiB.


#### Prerequisites
1. **Windows Server** with IIS installed.
2. [**Application Request Routing (ARR)**](https://www.iis.net/downloads/microsoft/application-request-routing) and [**URL Rewrite**](https://www.iis.net/downloads/microsoft/url-rewrite) modules installed.
3. [**WebSocket Protocol**](https://learn.microsoft.com/en-us/iis/configuration/system.webserver/websocket) feature enabled.

For information on how to set up IIS as a reverse proxy please refer to [this](https://learn.microsoft.com/en-us/iis/extensions/url-rewrite-module/reverse-proxy-with-url-rewrite-v2-and-application-request-routing).
There are also information on how to use the IIS Manager [here](https://learn.microsoft.com/en-us/iis/).

The following configuration example assumes the following:
* A site has been created that is configured with HTTPS support and the desired host name.
* A server farm named `nc-server-farm` has been created and is pointing to the Nextcloud server.
* No global Rewrite Rules has been created for the `nc-server-farm` server farm.

Add the following `web.config` file to the root of the site you created as the reverse proxy.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.web>
    <!-- Allow all urls -->
    <httpRuntime requestValidationMode="2.0" requestPathInvalidCharacters="" />
  </system.web>
  <system.webServer>
    <rewrite>
      <!-- useOriginalURLEncoding needs to be set to false, otherwise IIS will double encode urls causing all files with spaces or special characters to be inaccessible -->
      <rules useOriginalURLEncoding="false">
        <!-- Force https -->
        <rule name="Https" stopProcessing="true">
          <match url="(.*)" />
          <conditions>
            <add input="{HTTPS}" pattern="^OFF$" />
          </conditions>
          <action type="Redirect" url="https://{HTTP_HOST}/{REQUEST_URI}" appendQueryString="false" />
        </rule>
        <!-- Redirect to internal nextcloud server -->
        <rule name="To nextcloud" stopProcessing="true">
          <match url="(.*)" />
          <conditions>
            <add input="{HTTPS}" pattern="^ON$" />
          </conditions>
          <!-- Note that {UNENCODED_URL} already contains starting slash, so we must add it directly after the port number without additional slash -->
          <action type="Rewrite" url="http://nc-server-farm:11000{UNENCODED_URL}" appendQueryString="false" />
        </rule>
      </rules>
    </rewrite>
    <security>
      <!-- Increase upload limit to 2GiB -->
      <requestFiltering allowDoubleEscaping="true">
        <requestLimits maxAllowedContentLength="2147483648" />
      </requestFiltering>
    </security>
  </system.webServer>
</configuration>

```
⚠️ **Please note:** Look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.

</details>

### Tailscale

<details>

<summary>click here to expand</summary>

For a reverse proxy example guide for Tailscale, see this guide by @flll: https://github.com/nextcloud/all-in-one/discussions/5439

</details>


### Others

<details>

<summary>click here to expand</summary>

Config examples for other reverse proxies are currently not documented. Pull requests are welcome!

</details>

## 2. Use this startup command

After adjusting your reverse proxy config, use the following command to start AIO:<br>

(For a `compose.yaml` example, see the example further [below](#inspiration-for-a-docker-compose-file).)

```
# For Linux:
sudo docker run \
--init \
--sig-proxy=false \
--name nextcloud-aio-mastercontainer \
--restart always \
--publish 8080:8080 \
--env APACHE_PORT=11000 \
--env APACHE_IP_BINDING=0.0.0.0 \
--env APACHE_ADDITIONAL_NETWORK="" \
--env SKIP_DOMAIN_VALIDATION=false \
--volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
--volume /var/run/docker.sock:/var/run/docker.sock:ro \
ghcr.io/nextcloud-releases/all-in-one:latest
```

Note: you may be interested in adjusting Nextcloud’s datadir to store the files in a different location than the default docker volume. See [this documentation](https://github.com/nextcloud/all-in-one#how-to-change-the-default-location-of-nextclouds-datadir) on how to do it.

You should also think about limiting the Apache container to listen only on localhost in case the reverse proxy is running on the same host and in the host network, by providing an additional environmental variable to this docker run command. See [point 3](#3-limit-the-access-to-the-apache-container).

On macOS see https://github.com/nextcloud/all-in-one#how-to-run-aio-on-macos.

<details>

<summary>Command for Windows</summary>

On Windows, install [Docker Desktop](https://www.docker.com/products/docker-desktop/) (and don't forget to [enable ipv6](https://github.com/nextcloud/all-in-one/blob/main/docker-ipv6-support.md) if you should need that) and run the following command in the command prompt:

```
docker run ^
--init ^
--sig-proxy=false ^
--name nextcloud-aio-mastercontainer ^
--restart always ^
--publish 8080:8080 ^
--env APACHE_PORT=11000 ^
--env APACHE_IP_BINDING=0.0.0.0 ^
--env APACHE_ADDITIONAL_NETWORK="" ^
--env SKIP_DOMAIN_VALIDATION=false ^
--volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config ^
--volume //var/run/docker.sock:/var/run/docker.sock:ro ^
ghcr.io/nextcloud-releases/all-in-one:latest
```

Also, you may be interested in adjusting Nextcloud's Datadir to store the files on the host system. See [this documentation](https://github.com/nextcloud/all-in-one#how-to-change-the-default-location-of-nextclouds-datadir) on how to do it.

</details>

On Synology DSM see https://github.com/nextcloud/all-in-one#how-to-run-aio-on-synology-dsm

### Inspiration for a docker-compose file

Simply translate the docker run command into a docker-compose file. You can have a look at [this file](https://github.com/nextcloud/all-in-one/blob/main/compose.yaml) for some inspiration but you will need to modify it either way. You can find further examples here: https://github.com/nextcloud/all-in-one/discussions/588

## 3. Limit the access to the Apache container

Use this environment variable during the initial startup of the mastercontainer to make the apache container only listen on localhost: `--env APACHE_IP_BINDING=127.0.0.1`. **Attention:** This is only recommended to be set if you use `localhost` in your reverse proxy config to connect to your AIO instance. If you use an ip-address instead of localhost, you should set it to `0.0.0.0`.

## 4. Open the AIO interface

After starting AIO, you should be able to access the AIO Interface via `https://ip.address.of.the.host:8080` and type in and validate the domain that you have configured.<br>
⚠️ **Important:** do always use an ip-address if you access this port and not a domain as HSTS might block access to it later! (It is also expected that this port uses a self-signed certificate due to security concerns which you need to accept in your browser)<br>
Enter your domain in the AIO interface that you've used in the reverse proxy config and you should be done. Please do not forget to open/forward port `3478/TCP` and `3478/UDP` in your firewall/router for the Talk container!

## 5. Optional: get a valid certificate for the AIO interface

If you want to also access your AIO interface publicly with a valid certificate, you can add e.g. the following config to your Caddyfile:

```
https://<your-nc-domain>:8443 {
    reverse_proxy https://localhost:8080 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}
```
⚠️ **Please note:** Look into [this](#adapting-the-sample-web-server-configurations-below) to adapt the above example configuration.

Afterwards should the AIO interface be accessible via `https://ip.address.of.the.host:8443`. You can alternatively change the domain to a different subdomain by using `https://<your-alternative-domain>:443` instead of `https://<your-nc-domain>:8443` in the Caddyfile and use that to access the AIO interface.

## 6. How to debug things?

If something does not work, follow the steps below:
1. Make sure to exactly follow the whole reverse proxy documentation step-for-step from top to bottom!
1. Make sure that you used the `docker run` command that is described in this reverse proxy documentation. **Hint:** make sure that you have set the `APACHE_PORT` via e.g. `--env APACHE_PORT=11000` during the docker run command!
1. Make sure to set the `APACHE_IP_BINDING` variable correctly. If in doubt, set it to `--env APACHE_IP_BINDING=0.0.0.0`
1. Make sure that all ports to which your reverse proxy is pointing match the chosen `APACHE_PORT`.
1. Make sure to follow [this](#adapting-the-sample-web-server-configurations-below) to adapt the example configurations to your specific setup!
1. Make sure that the mastercontainer is able to spawn other containers. You can do so by checking that the mastercontainer indeed has access to the Docker socket which might not be positioned in one of the suggested directories like `/var/run/docker.sock` but in a different directory, based on your OS and the way how you installed Docker. The mastercontainer logs should help figuring this out. You can have a look at them by running `sudo docker logs nextcloud-aio-mastercontainer` after the container is started the first time.
1. Check if after the mastercontainer was started, the reverse proxy if running inside a container, can reach the provided apache port. You can test this by running `nc -z localhost 11000; echo $?` from inside the reverse proxy container. If the output is `0`, everything works. Alternatively you can of course use instead of `localhost` the ip-address of the host here for the test.
1. Make sure that you are not behind CGNAT. If that is the case, you will not be able to open ports properly. In that case you might use a Cloudflare Tunnel!
1. If you use Cloudflare, you might need to skip the domain validation anyways since it is known that Cloudflare might block the validation attempts. In that case, see the last option below!
1. If your reverse proxy is configured to use the host network (as recommended in the above docs) or running on the host, make sure that you've configured your firewall to open port 443 (and 80)!
1. Check if you have a public IPv4- and public IPv6-address. If you only have a public IPv6-address (e.g. due to DS-Lite), make sure to enable IPv6 in Docker and your whole networking infrastructure (e.g. also by adding an AAAA DNS-entry to your domain)!
1. [Enable Hairpin NAT in your router](https://github.com/nextcloud/all-in-one/discussions/5849) or [set up a local DNS server and add a custom dns-record](https://github.com/nextcloud/all-in-one#how-can-i-access-nextcloud-locally) that allows the server to reach itself locally
1. Try to configure everything from scratch - if it still does not work by following https://github.com/nextcloud/all-in-one#how-to-properly-reset-the-instance.
1. As last resort, you may disable the domain validation by adding `--env SKIP_DOMAIN_VALIDATION=true` to the docker run command. But only use this if you are completely sure that you've correctly configured everything!
