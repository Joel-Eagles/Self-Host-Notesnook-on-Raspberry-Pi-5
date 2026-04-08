# Setting up a Notesnook Server on a Raspberry Pi 5

**What you will need:**
- Raspberry Pi 5 with Raspberry Pi OS x64 installed and a static LAN IP address assigned to it (you dont need static WAN for this setup)
- A domain or $10 to purchase a domain
  - This guide uses a Cloudflare Tunnel for external access. For this reason I recommend purchasing your domain through Cloudflare for simplicity

> [!NOTE]
> - While setting this up for myself I used these 2 references: [Notesnook sync server: a Noob-Friendly Setup Tutorial](https://sh.itjust.works/post/31407921) and [How to Self Host Notesnook Sync Server in 2026](https://fareedwarrad.substack.com/p/how-to-self-host-notesnook-sync-server)
> - Unlike Windows, Linux is case-sensitive. I will be listing directories and files in lowercase. If you capitalize folders like your Docker container root folder, you will need to adjust your commands to match.
> - This guide will focus on setting up the essentials first, then security hardening after. The essentials here include adequate security for most home users.

---

## Setting up Docker

1. Download and run the official installation script provided by Docker:
   ```bash
   curl -fsSL https://get.docker.com | sh
   ```
2. Verify Docker Compose was installed with `docker compose version`. If you don't see a version number, run:
   ```bash
   sudo apt-get install docker-compose-plugin
   ```
3. Create a Docker root folder. This is where all your Docker containers will live. Where you put this is up to you, but ideally you would put it in one of two places:
   - `/opt/docker/`
     - This is the technically correct place for this, especially if you plan on having multiple users access and manage the back-end, like in a production environment
   - `~/docker/` or `/home/<user>/docker/`
     - This is the easiest place to put it, because it simplifies permissions. You already have full access to everything in your user home folder
     - **This is what this guide will be using**
4. Create your Docker root folder:
   ```bash
   mkdir ~/docker/
   ```

> [!TIP]
> **Optional — Docker without `sudo`**
>
> By default, you can only interact with Docker with `sudo`. If you are running your containers in `/opt/docker/` and want to give multiple user accounts the ability to interact with docker without `sudo` (or you just don't want to use `sudo`), you can add the user to a docker user group.
>
> 1. Create the Docker user group: `sudo groupadd docker`
> 2. Add the user to the new group: `sudo usermod -aG docker <User>`, replacing `<User>` with the user account's username
> 3. Restart your Pi: `sudo reboot`
> 4. Once you are logged in, run `docker run hello-world` without `sudo` to verify you have permissions

---

## Setting up NGINX Proxy Manager in Docker

1. Create a Docker directory for NPM and some subdirectories:
   ```bash
   mkdir -p ~/docker/nginx-proxy-manager/data/
   mkdir -p ~/docker/nginx-proxy-manager/letsencrypt/
   ```
2. Create your Docker Compose setup file (YAML):
   ```bash
   touch ~/docker/nginx-proxy-manager/docker-compose.yml
   ```
3. Edit the file with `nano ~/docker/nginx-proxy-manager/docker-compose.yml` and paste the following:
   ```yaml
   services:
     nginx-proxy-manager:
       image: jc21/nginx-proxy-manager:latest
       container_name: nginx-proxy-manager
       ports:
         - 80:80      # HTTP
         - 443:443    # HTTPS
         - 81:81      # Admin web UI
       volumes:
         - ./data:/data
         - ./letsencrypt:/etc/letsencrypt
       restart: unless-stopped
   ```
4. Press `Ctrl+S` to save and `Ctrl+X` to exit
5. Go to the folder if you aren't already in it:
   ```bash
   cd ~/docker/nginx-proxy-manager/
   ```
6. Start the container:
   ```bash
   sudo docker compose up -d
   ```
7. Log into the web UI by opening a web browser and going to `xxx.xxx.xxx.xxx:81`, replacing the exes with your Pi's static IP address. For example, `192.168.2.42:81`
8. You will be presented with a login screen:
   - Enter your email and set a randomly generated password. Document in password manager
   - If needed, the default login is `admin@example.com` / `changeme`

---

## Cloudflare and Domain Setup

1. Go to [dash.cloudflare.com](https://dash.cloudflare.com) and create an account if you don't already have one
2. Make sure you use a secure password and set up 2FA
3. On the left, go to **Domains → Overview → Buy Domain**
4. Search for and purchase a domain
   - This can be basically anything as long as no one else owns it
   - Domains generally cost around $10 per year (like 85 cents per month)

> [!NOTE]
> If you like you can purchase domains for cheaper from third parties and transfer them to Cloudflare. This guide won't cover that.

### Set up Cloudflare Tunnel

Cloudflare Tunnel is a free product that creates an outbound-only encrypted link between your internal server (Raspberry Pi) and Cloudflare's external network. It eliminates the need for a public static IP address, port forwarding, or inbound firewall rules. This gives you a very secure way to expose internal services to the internet.

1. On the left, click on **Zero Trust**
2. It will ask you to create an account for Zero Trust. Enter the required info.
3. On the left, click on **Networks → Connectors**
4. Click **Create a Tunnel**
5. Select **cloudflared**
6. Set a name for your tunnel connection and click **Save/Next**
7. Select **Docker** as your operating system
8. Copy what it gives you. You will only need the raw token for this, not the full command

> [!NOTE]
> **Example**
>
> If Cloudflare gives you:
> ```
> docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token Cf9q93rJWFMdkdYUUVlMXLPxaMYGswRWEPUwaOP5p7oTl7C93zNUl7Mm13MNIKK2WNRDSgvQz5nVnYumESYgYShbJvYkuZC1bXQeSrTYMZtnvXR3CnF22xTlnYJvqIB1acahFe6KmNxHbIGXTnizTBNkfld355xSIqw8ju5lmgeGbABPPBnyoNQe
> ```
> You will only need:
> ```
> Cf9q93rJWFMdkdYUUVlMXLPxaMYGswRWEPUwaOP5p7oTl7C93zNUl7Mm13MNIKK2WNRDSgvQz5nVnYumESYgYShbJvYkuZC1bXQeSrTYMZtnvXR3CnF22xTlnYJvqIB1acahFe6KmNxHbIGXTnizTBNkfld355xSIqw8ju5lmgeGbABPPBnyoNQe
> ```

9. Back on your Raspberry Pi, create a new Docker directory for the Cloudflare tunnel:
   ```bash
   mkdir ~/docker/cloudflare-tunnel/
   ```
10. Create your Docker Compose file:
    ```bash
    touch ~/docker/cloudflare-tunnel/docker-compose.yml
    ```
11. Edit the file with `nano ~/docker/cloudflare-tunnel/docker-compose.yml` and paste the following. Replace `<your_token_here>` with the raw token from Cloudflare:
    ```yaml
    services:
      cloudflared:
        image: cloudflare/cloudflared:latest
        container_name: cloudflared
        restart: unless-stopped
        command: tunnel run
        environment:
          - TUNNEL_TOKEN=<your_token_here>
    ```
12. Press `Ctrl+S` to save and `Ctrl+X` to exit
13. Go to the directory:
    ```bash
    cd ~/docker/cloudflare-tunnel
    ```
14. Start the tunnel service:
    ```bash
    sudo docker compose up -d
    ```
15. Check the connection with `sudo docker logs cloudflared` — you should see `"Connected to Cloudflare"` somewhere in the output
16. Go back to the **Networks → Tunnels** page and click on your Connector
17. Click on the **Published Application Routes** tab
18. Add your routes. You will need 5 total:

    | Subdomain | Type | URL | HTTP Host Header | Origin Server Name |
    |---|---|---|---|---|
    | notesnook | HTTPS | xxx.xxx.xxx.xxx:443 | notesnook.yourdomain.com | notesnook.yourdomain.com |
    | notesnook-auth | HTTPS | xxx.xxx.xxx.xxx:443 | notesnook-auth.yourdomain.com | notesnook-auth.yourdomain.com |
    | notesnook-events | HTTPS | xxx.xxx.xxx.xxx:443 | notesnook-events.yourdomain.com | notesnook-events.yourdomain.com |
    | notesnook-monograph | HTTPS | xxx.xxx.xxx.xxx:443 | notesnook-monograph.yourdomain.com | notesnook-monograph.yourdomain.com |
    | notesnook-attachments | HTTP | xxx.xxx.xxx.xxx:9000 | [Not needed] | [Not needed] |

    - The domain should be the one you just purchased
    - Replace the Exes with the local static IP address of your Raspberry Pi
    - For each of the first four routes (not the attachments route) you will need to set the **HTTP Host Header** and the **Origin Server Name** to the FQDN of the route
      - You can find these settings by expanding **"Additional application settings"** and expanding **"TLS"** and **"HTTP Settings"**

### Set up Nginx Server for Cloudflare Domain

#### Cloudflare

1. Go back to [Dash.Cloudflare.com](https://dash.cloudflare.com)
2. Click on your account in the top right
3. On the left, select **API Tokens**
4. Click on **Create Token**
5. Select **Edit DNS Zone** template
   - Leave Permissions as is
   - Zone Resources should be "Include", "Specific Zone", and `<your domain>`
   - Continue to **Summary → Create Token** and copy the token immediately. You won't get another chance

#### NGINX

1. Log back into your NGINX Proxy Manager Server in your browser
2. Go to **Certificates**. Click on **Add Certificate**. Select **Let's Encrypt with DNS**
3. For your domains, enter both `*.yourdomain.com` and `yourdomain.com` (replace with your domain)
4. Key Type should be **ECDSA 256**
5. DNS provider should be **Cloudflare**
6. Paste the Cloudflare API key you created into the token field
7. Click **Save**
8. Go to **Hosts → Proxy Hosts → Add Proxy Host**. You will need to make 5 total
   - For each one, on the SSL tab, set the cert to the one you made and enable **Force SSL**, **HTTP/2**, and **HSTS**
   - Forward Hostname / IP is the local static IP address of your Raspberry Pi (for example `192.168.2.10`)

    | Domain Names | Scheme | Forward Hostname / IP | Forward Ports | Block Common Exploits? | Websocket Support |
    |---|---|---|---|---|---|
    | notesnook-auth.yourdomain.com | http | xxx.xxx.xxx.xxx | 8264 | on | off |
    | notesnook-events.yourdomain.com | http | xxx.xxx.xxx.xxx | 7264 | on | on |
    | notesnook-monograph.yourdomain.com | http | xxx.xxx.xxx.xxx | 6264 | on | off |
    | notesnook-attachments.yourdomain.com | http | xxx.xxx.xxx.xxx | 9000 | on | off |
    | notesnook.yourdomain.com | http | xxx.xxx.xxx.xxx | 5264 | on | on |

---

## Preparing for Notesnook Set Up

You will need a few pieces of information while setting up Notesnook.

### 1. SMTP Information

For emailing to you or your user account from the server for notifications and warnings.

- Most email providers allow you to set up SMTP through your existing email account for free
- If you have a paid SMTP service, you know what to do

**If you have a Microsoft email (Outlook):**
```bash
SMTP_USERNAME=<your email address>
SMTP_PASSWORD=<your email password>
SMTP_HOST=smtp.office365.com
```

**If you have a Google email (Gmail):**

You will need to generate a special app password. This is an extra security step that Google takes.

1. Go to [myaccount.google.com](http://myaccount.google.com)
2. Before going further, make sure you have 2-factor authentication turned on. It is required to create an app password
3. Type "App Passwords" into the search bar at the top and click on the result
4. Enter a descriptive app password name like `"Notesnook-Pi"`
5. Click **Create** and copy the 16 digit app password — this is your `SMTP_PASSWORD`

```bash
SMTP_USERNAME=<your gmail address>
SMTP_PASSWORD=<xxxx xxxx xxxx xxxx>    # the 16-char app password
SMTP_HOST=smtp.gmail.com
```

### 2. Notesnook API Secret

This is just a randomly generated string. Run the following and copy the 64 digit output string. This is your `NOTESNOOK_API_SECRET`:
```bash
openssl rand -hex 32
```

### 3. MinIO Root Password

This would be used to access the attachments directly, but we are not accessing it directly, so this password should be long and random. Generate one with the following and copy the 32 digit output. This is your `MINIO_ROOT_PASSWORD`:
```bash
openssl rand -hex 16
```

---

## Setting up Notesnook in Docker

1. Create a Docker directory for Notesnook and some subdirectories:
   ```bash
   mkdir -p ~/docker/notesnook/db
   mkdir -p ~/docker/notesnook/s3
   ```

   > [!WARNING]
   > Take note of these directories. If you ever need to transfer your data but your server is dead, `/db/` is your notes database, and `/s3/` are the attachments.

2. Create your environment variable file:
   ```bash
   touch ~/docker/notesnook/.env
   ```
3. Edit the file with `nano ~/docker/notesnook/.env` and paste the following:
   ```bash
   # Instance
   INSTANCE_NAME=               # This can be anything, I recommend something like "Timmy's Notesnook"
   DISABLE_SIGNUPS=false
   NOTESNOOK_API_SECRET=        # This is the 64 digit random string you generated above

   # SMTP — needed for account emails/2FA
   SMTP_USERNAME=               # your email address
   SMTP_PASSWORD=               # your email/app password
   SMTP_HOST=                   # e.g. smtp.gmail.com
   SMTP_PORT=587

   # Public URLs — replace yourdomain.com with your actual domain
   AUTH_SERVER_PUBLIC_URL=https://notesnook-auth.yourdomain.com
   NOTESNOOK_APP_PUBLIC_URL=https://app.notesnook.com
   MONOGRAPH_PUBLIC_URL=https://notesnook-monograph.yourdomain.com
   ATTACHMENTS_SERVER_PUBLIC_URL=https://notesnook-attachments.yourdomain.com

   # MinIO
   MINIO_ROOT_USER=admin        # You can change this, just document it.
   MINIO_ROOT_PASSWORD=         # This is the 32 digit random string you generated above
   ```
   - `INSTANCE_NAME` = Your Notesnook's Unique Name
   - `MINIO_ROOT_USER` = An admin username of your choice, recommended to rename it for security

4. Press `Ctrl+S` to save and `Ctrl+X` to exit
5. Create your Docker Compose Configuration file:
   ```bash
   touch ~/docker/notesnook/docker-compose.yml
   ```
6. Edit the file with `nano ~/docker/notesnook/docker-compose.yml` and paste the following:

   > [!IMPORTANT]
   > The current official Monograph Image does not work on Raspberry Pi due to a bug. Instead, we are using a [community made Monograph image made by codewhiz](https://hub.docker.com/r/codewhiz/notesnook-monograph-arm64).
   >
   > We are also using the [Autoheal image from willfarrell](https://hub.docker.com/r/willfarrell/autoheal/).

   ```yaml
   x-server-discovery: &server-discovery
     NOTESNOOK_SERVER_PORT: 5264
     NOTESNOOK_SERVER_HOST: notesnook-server
     IDENTITY_SERVER_PORT: 8264
     IDENTITY_SERVER_HOST: identity-server
     SSE_SERVER_PORT: 7264
     SSE_SERVER_HOST: sse-server
     SELF_HOSTED: 1
     IDENTITY_SERVER_URL: ${AUTH_SERVER_PUBLIC_URL}
     NOTESNOOK_APP_HOST: ${NOTESNOOK_APP_PUBLIC_URL}

   x-env-files: &env-files
     - .env

   services:
     validate:
       image: vandot/alpine-bash
       entrypoint: /bin/bash
       env_file: *env-files
       command:
         - -c
         - |
           required_vars=(
             "INSTANCE_NAME"
             "NOTESNOOK_API_SECRET"
             "DISABLE_SIGNUPS"
             "SMTP_USERNAME"
             "SMTP_PASSWORD"
             "SMTP_HOST"
             "SMTP_PORT"
             "AUTH_SERVER_PUBLIC_URL"
             "NOTESNOOK_APP_PUBLIC_URL"
             "MONOGRAPH_PUBLIC_URL"
             "ATTACHMENTS_SERVER_PUBLIC_URL"
           )
           for var in "${required_vars[@]}"; do
             if [ -z "${!var}" ]; then
               echo "Error: Required environment variable $$var is not set."
               exit 1
             fi
           done
           echo "All required environment variables are set."
       restart: "no"

     notesnook-db:
       image: mongo:7.0.12
       hostname: notesnook-db
       restart: unless-stopped
       volumes:
         - ./db:/data/db
         - ./db:/data/configdb
       networks:
         - notesnook
       command: --replSet rs0 --bind_ip_all
       depends_on:
         validate:
           condition: service_completed_successfully
       healthcheck:
         test: echo 'db.runCommand("ping").ok' | mongosh mongodb://localhost:27017 --quiet
         interval: 40s
         timeout: 30s
         retries: 3
         start_period: 60s

     initiate-rs0:
       image: mongo:7.0.12
       networks:
         - notesnook
       depends_on:
         - notesnook-db
       entrypoint: /bin/sh
       command:
         - -c
         - |
           mongosh mongodb://notesnook-db:27017 <<EOF
           rs.initiate();
           rs.status();
           EOF

     notesnook-s3:
       image: minio/minio:RELEASE.2024-07-29T22-14-52Z
       restart: unless-stopped
       ports:
         - 9000:9000
         - 9090:9090
       networks:
         - notesnook
       volumes:
         - ./s3:/data/s3
       environment:
         MINIO_BROWSER: "on"
       depends_on:
         validate:
           condition: service_completed_successfully
       env_file: *env-files
       command: server /data/s3 --console-address :9090
       healthcheck:
         test: timeout 5s bash -c ':> /dev/tcp/127.0.0.1/9000' || exit 1
         interval: 40s
         timeout: 30s
         retries: 3
         start_period: 60s

     setup-s3:
       image: minio/mc:RELEASE.2024-07-26T13-08-44Z
       depends_on:
         - notesnook-s3
       networks:
         - notesnook
       entrypoint: /bin/bash
       env_file: *env-files
       command:
         - -c
         - |
           until mc alias set minio http://notesnook-s3:9000/ ${MINIO_ROOT_USER:-minioadmin} ${MINIO_ROOT_PASSWORD:-minioadmin}; do
             sleep 1;
           done;
           mc mb minio/attachments -p

     identity-server:
       image: streetwriters/identity:latest
       restart: unless-stopped
       ports:
         - 8264:8264
       networks:
         - notesnook
       env_file: *env-files
       depends_on:
         - notesnook-db
       healthcheck:
         test: wget --tries=1 -nv -q http://localhost:8264/health -O- || exit 1
         interval: 40s
         timeout: 30s
         retries: 3
         start_period: 60s
       environment:
         <<: *server-discovery
         MONGODB_CONNECTION_STRING: mongodb://notesnook-db:27017/identity?replSet=rs0
         MONGODB_DATABASE_NAME: identity

     notesnook-server:
       image: streetwriters/notesnook-sync:latest
       restart: unless-stopped
       ports:
         - 5264:5264
       networks:
         - notesnook
       env_file: *env-files
       depends_on:
         - notesnook-s3
         - setup-s3
         - identity-server
       healthcheck:
         test: wget --tries=1 -nv -q http://localhost:5264/health -O- || exit 1
         interval: 40s
         timeout: 30s
         retries: 3
         start_period: 60s
       environment:
         <<: *server-discovery
         MONGODB_CONNECTION_STRING: mongodb://notesnook-db:27017/?replSet=rs0
         MONGODB_DATABASE_NAME: notesnook
         S3_INTERNAL_SERVICE_URL: "http://notesnook-s3:9000/"
         S3_INTERNAL_BUCKET_NAME: "attachments"
         S3_ACCESS_KEY_ID: "${MINIO_ROOT_USER:-minioadmin}"
         S3_ACCESS_KEY: "${MINIO_ROOT_PASSWORD:-minioadmin}"
         S3_SERVICE_URL: "${ATTACHMENTS_SERVER_PUBLIC_URL}"
         S3_REGION: "us-east-1"
         S3_BUCKET_NAME: "attachments"

     sse-server:
       image: streetwriters/sse:latest
       restart: unless-stopped
       ports:
         - 7264:7264
       env_file: *env-files
       depends_on:
         - identity-server
         - notesnook-server
       networks:
         - notesnook
       healthcheck:
         test: wget --tries=1 -nv -q http://localhost:7264/health -O- || exit 1
         interval: 40s
         timeout: 30s
         retries: 3
         start_period: 60s
       environment:
         <<: *server-discovery

     monograph-server:
       image: codewhiz/notesnook-monograph-arm64:latest
       restart: unless-stopped
       ports:
         - 6264:3000
       env_file: *env-files
       depends_on:
         - notesnook-server
       networks:
         - notesnook
       healthcheck:
         test: ["CMD-SHELL", "node -e \"require('http').get('http://localhost:3000/api/health', r => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))\""]
         interval: 40s
         timeout: 30s
         retries: 3
         start_period: 60s
       environment:
         <<: *server-discovery
         API_HOST: http://notesnook-server:5264/
         PUBLIC_URL: ${MONOGRAPH_PUBLIC_URL}

     autoheal:
       image: willfarrell/autoheal:latest
       restart: always
       environment:
         - AUTOHEAL_INTERVAL=60
         - AUTOHEAL_START_PERIOD=300
         - AUTOHEAL_DEFAULT_STOP_TIMEOUT=10
       depends_on:
         validate:
           condition: service_completed_successfully
       volumes:
         - /var/run/docker.sock:/var/run/docker.sock

   networks:
     notesnook:
   ```

7. Press `Ctrl+S` to save and `Ctrl+X` to exit
8. Go to the container's directory:
   ```bash
   cd ~/docker/notesnook/
   ```
9. Pull the images:
   ```bash
   sudo docker compose pull
   ```
10. Start the Notesnook Database service first:
    ```bash
    sudo docker compose up -d notesnook-db
    ```
    You will have to wait for the database to fully start up before starting the other services. This can take 2 minutes. You can check the current status with:
    ```bash
    sudo docker compose ps notesnook-db
    ```
11. Once that shows **healthy**, manually initialize the replica set:
    ```bash
    sudo docker compose exec notesnook-db mongosh --eval "rs.initiate()"
    ```
    Then verify it is running:
    ```bash
    sudo docker compose exec notesnook-db mongosh --eval "rs.status()"
    ```
    Look for `"stateStr" : "PRIMARY"` in the output.

12. Start the rest of the Notesnook services:
    ```bash
    sudo docker compose up -d
    ```
    "Already Initialized" errors are safe to ignore, since we started the replica set already.

13. Check the Notesnook services health. This can take up to 2 minutes for all services to fully start:
    ```bash
    sudo docker compose ps
    ```

14. Once the services are all healthy, test each of the routes:
    ```bash
    curl http://localhost:5264/health      # notesnook-server
    curl http://localhost:8264/health      # identity-server
    curl http://localhost:7264/health      # sse-server
    curl http://localhost:6264/api/health  # monograph-server
    curl http://localhost:9000/minio/health/live  # notesnook-s3
    ```

15. Then, check if they are accessible from the internet:
    ```bash
    curl https://notesnook-monograph.yourdomain.com/api/health
    curl https://notesnook.yourdomain.com/health
    curl https://notesnook-auth.yourdomain.com/health
    curl https://notesnook-events.yourdomain.com/health
    ```

**Connect your app to your server and you are done!**

> [!CAUTION]
> Right now, there is nothing to allow you to recover a damaged database, so you could lose all your data if there is an issue. You should be backing up your `~/docker/notesnook/db` and `/s3` directories nightly. If you have an existing backup solution, go for it, otherwise you can use this guide to utilize rsync for backup: https://linuxconfig.org/how-to-backup-raspberry-pi

---

## Security Hardening

Because you are using Cloudflare Tunnel, your setup is already quite secure from external threats. That said, there are some additional steps you should take.

### Ensure Your Raspberry Pi's Local User Account Has a Long Randomly Generated Password

14 digits or more with numbers, letters, capitals, and symbols. Save to password manager.

### Enable 2FA on Your Notesnook Account

Another easy one with no downsides. This is especially good because Notesnook prompts for 2FA before prompting for your password, which helps prevent brute-forcing of your password.

Expand the sandwich menu in the top left of your app and click on your profile → **Settings**. Select **Authentication** on the left and set up your 2FA.

### Disable Sign-Ups on Your Notesnook Authentication Server

Because your server is exposed to the internet, anyone who is able to find your instance can type in your server info to their Notesnook app and create an account using your server's resources.

After all Notesnook accounts have been created (yours and anyone you want to have access), block the ability to create a new account on your server by editing the environment variable file:
```bash
nano ~/docker/notesnook/.env
```
Change `DISABLE_SIGNUPS=false` to `DISABLE_SIGNUPS=true`.

> [!NOTE]
> Accounts made through Notesnook's official hosting are completely separate from any accounts made on your self-hosted authentication server. If you have an existing Notesnook account, you will still have to create a new account after connecting to your instance.

### Set up Rate-Limiting on the Authentication Server Tunnel

To combat the threat of brute-forcing your Notesnook password, we want to block anyone that attempts a password too many times in a short period of time.

1. At [dash.cloudflare.com](https://dash.cloudflare.com), go to **Domains → Overview → yourdomain.com**
2. On the left, go to **Security → Security rules**
3. Next to **Rate limiting rules**, click **Create rule**
4. Fill out the following:

   | Setting | Value |
   |---|---|
   | Rule Name | e.g. `Notesnook Auth Rate Limit - max 3 attempts per 10 seconds + 10 seconds time out` |
   | When incoming requests match | `(http.host eq "notesnook-auth.yourdomain.com")` |
   | With the same characteristics | IP |
   | Requests | 3 |
   | Period | 10 seconds |
   | Then take action | Block |
   | For duration | 10 seconds |
   | Place at | First |

5. Click **Save**

> [!NOTE]
> Most of these settings are just the only option due to Cloudflare's limitations on the free tier.
>
> In a more standard deployment in a professional environment, you might have your system exposed directly to the internet instead of through Cloudflare Tunnel. In that case you could use Fail2ban to watch the logs of your authentication server, and permanently block any IPs that fail too many times using iptables. We cannot use that method here because (on the Pi) all traffic is outbound to a single address (Cloudflare). If needed, you can upgrade to a higher tier Cloudflare subscription (or move to an alternative service) to have your rate limiting act on response codes, and ban an IP if the authentication server returns failed logins too many times.

### Geo-Blocking

Lots of people will tell you that geo-blocking is pointless for a number of reasons. But, as a home user you usually aren't going to be a direct target. Your threats are automated credential stuffing bots and opportunistic scanners. Geo-blocking eliminates the majority of that noise rather easily.

1. At [dash.cloudflare.com](https://dash.cloudflare.com), go to **Domains → Overview → yourdomain.com**
2. On the left, go to **Security → Security rules**
3. Click **+ Create rule**
4. You have 2 main options: you can either block all countries other than your own, or you can block specific locations.

**To only allow your country:**

| Setting | Value |
|---|---|
| Name | Geo-Blocking |
| Field | Country |
| Operator | does not equal |
| Value | Your country |
| Then take action | Block |

**To block specific countries:**

Click **Edit Expression** and paste the following (edit country codes as needed):
```
(ip.geoip.country in {"RU" "CN" "KP"})
```
This example would block Russia, China, and North Korea. The list is space-separated. For each country, enter its country code as defined by the ISO in quote marks. Then set **Then take action → Block** and click **Save**.

> [!NOTE]
> In the US, you can use the current US Sanctions list to decide which countries to block.

### Restrict Your Notesnook Environment Variable File Permissions

Your Notesnook `.env` file has your email and password, your API secret, and your MinIO admin password stored in plaintext for anyone to see.

Restrict the file permissions to your user account:
```bash
chmod 600 ~/docker/notesnook/.env
```

### Lock MinIO Console to Localhost

Currently, MinIO has its console port 9090 mapped for all interfaces, so anyone on your network can access the admin login panel. MinIO is completely managed by our Notesnook instance, so this is a needless security risk.

Lock your MinIO console to only be accessible locally on your Pi by editing the Docker Compose file:
```bash
nano ~/docker/notesnook/docker-compose.yml
```
Find the service entry labeled `notesnook-s3:`. Under `ports:`, change `- 9090:9090` to `- 127.0.0.1:9090:9090`.

### Force MinIO to Run as a Non-Root User

All other services either require root or already handle permissions dropping

First, check what your user account's PUID and PGID are. If you are using the default account you set up during OS setup, it should be `1000:1000`. You can find this by running `id` as the user (don't use `sudo`).

The following steps use the `1000:1000` IDs.

1. Make sure the MinIO s3 directory is owned by your user:
   ```bash
   sudo chown -R 1000:1000 ~/docker/notesnook/s3
   ```
2. Open the Docker Compose file:
   ```bash
   nano ~/docker/notesnook/docker-compose.yml
   ```
3. Find the service entry labeled `notesnook-s3:` and add `user: "1000:1000"` under `image: ...`:
   ```yaml
   notesnook-s3:
     image: minio/minio:RELEASE.2024-07-29T22-14-52Z
     user: "1000:1000"
     restart: unless-stopped
     ...
   ```
4. Press `Ctrl+S` to save and `Ctrl+X` to exit
5. Apply the changes by running the following one at a time:
   ```bash
   cd ~/docker/notesnook
   sudo docker compose up -d --force-recreate notesnook-s3
   ```

---

## Set up Docker Container Auto-Updates

Docker containers are super easy to update. Simply go to the container directory like `cd ~/docker/notesnook` and run `sudo docker compose pull` then `sudo docker compose up -d`. That still requires you to manually SSH into your server and update the images manually. Instead, we can set up "Watchtower" to update the images automatically.

1. Create a new Docker container directory:
   ```bash
   mkdir ~/docker/watchtower
   ```
2. Create a Docker Compose file with `touch ~/docker/watchtower/docker-compose.yml` and paste the following:
   ```yaml
   services:
     watchtower:
       image: containrrr/watchtower
       container_name: watchtower
       volumes:
         - /var/run/docker.sock:/var/run/docker.sock
       environment:
         - WATCHTOWER_CLEANUP=true
         - WATCHTOWER_SCHEDULE=0 0 3 * * *
         - DOCKER_API_VERSION=1.40
         - WATCHTOWER_LABEL_ENABLE=true
       restart: unless-stopped
   ```
   - `WATCHTOWER_SCHEDULE=0 0 3 * * *` means Watchtower will check for updates every day at 3 AM

3. Press `Ctrl+S` to save and `Ctrl+X` to exit
4. Go to the folder and start Watchtower:
   ```bash
   cd ~/docker/watchtower/
   sudo docker compose up -d
   ```
5. Now, to any service you would like to update automatically, add the following label to the service in its `docker-compose.yml` file:
   ```yaml
   labels:
     - com.centurylinklabs.watchtower.enable=true
   ```

> [!NOTE]
> **Example — identity-server**
> ```yaml
> identity-server:
>   image: streetwriters/identity:latest
>   ports:
>     - 8264:8264
>   networks:
>     - notesnook
>   env_file: *env-files
>   labels:
>     - com.centurylinklabs.watchtower.enable=true
>   depends_on:
>     - notesnook-db
> ```

- Recommended to add the label to Cloudflare Tunnel and Nginx Proxy Manager
  - Notesnook can be updated as well if you like. Personally I prefer to manually update Notesnook. Especially notesnook-db, because it interacts with the database of notes, and if there is a bad update, it could damage or wipe your notes (Version upgrades to the MongoDB need to be manual (e.g. 7.x->8.x)).
- You will need to add the label to each service in the `docker-compose.yml` file for the services to update
- It is best to do your Notesnook updates manually, but if you would like to do auto-updates:
  - Not every Notesnook service needs updated regularly. You can add the label to all of them if you want, but the recommended ones are:
    - notesnook-db
      - This can be risky to update automatically, but you do need to update it if and when security updates are released
    - notesnook-s3
      - Same risk as above, but not as important (to me at least) because this is just for the attachments
    - identity-server
    - notesnook-server
    - sse-server
    - monograph-server

---

## Set up OS Auto Updates

1. Install the Unattended Updates System:
   ```bash
   sudo apt install unattended-upgrades
   ```
2. Install this add-on package to allow auto-restarts:
   ```bash
   sudo apt install apt-config-auto-update
   ```
3. Check if the system was installed properly by running:
   ```bash
   sudo unattended-upgrades --dry-run --debug
   ```
   You can ignore many errors shown while this runs — we still need to configure it.
4. Enable the service (should be set automatically, but just to be sure):
   ```bash
   sudo systemctl enable unattended-upgrades
   ```

> [!TIP]
> **Optional — Auto-Update the Docker Application**
>
> Open the config file:
> ```bash
> sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
> ```
> In the section that says `Unattended-Upgrade::Origins-Pattern`, add the following:
> ```bash
> "origin=Docker,codename=${distro_codename}";
> ```
> This will allow updates to the Docker host application automatically (not the containers).

5. Now run through the rest of the config and set up the system how you like. You will need to uncomment (remove `//`) from anything you want enabled or changed. Recommended settings:
   ```
   Unattended-Upgrade::AutoFixInterruptedDpkg "true";
   Unattended-Upgrade::MinimalSteps "true";
   Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
   Unattended-Upgrade::Remove-Unused-Dependencies "true";
   Unattended-Upgrade::Automatic-Reboot "true";
   Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
   ```
6. To properly enable auto updates, run `sudo nano /etc/apt/apt.conf.d/20auto-upgrades` and make sure the following lines are added:
   ```
   APT::Periodic::Update-Package-Lists "1";
   APT::Periodic::Unattended-Upgrade "1";
   ```
7. Next, set up the update schedules in systemd:
   ```bash
   sudo systemctl edit apt-daily.timer
   ```
   Add the following above the line that says `"### Lines below this comment will be discarded"`:
   ```ini
   [Timer]
   OnCalendar=
   OnCalendar=01:00
   RandomizedDelaySec=0
   ```
   Press `Ctrl+S` and `Ctrl+X`.

8. Next run:
   ```bash
   sudo systemctl edit apt-daily-upgrade.timer
   ```
   Add the following above the discard line:
   ```ini
   [Timer]
   OnCalendar=
   OnCalendar=01:05
   RandomizedDelaySec=0
   ```
   Press `Ctrl+S` and `Ctrl+X`.

   This setup will give the system 5 minutes to download updates before upgrading.

9. Finally, restart the systemd services:
   ```bash
   sudo systemctl restart apt-daily.timer
   sudo systemctl restart apt-daily-upgrade.timer
   ```

10. Reboot and make sure everything is working:
    ```bash
    sudo reboot
    ```
