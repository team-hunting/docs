
# Self Host (Django, Postgres, S3) with Docker, Portainer, Tailscale Funnel, and Nginx

Pre-implementation:  
Download proxmox iso and create bootable USB -> Install proxmox -> Configure proxmox with static IP


**VM Setup**  
ISO link from https://releases.ubuntu.com/noble/  
https://releases.ubuntu.com/noble/ubuntu-24.04.3-desktop-amd64.iso  
Download into proxmox from "local" -> ISO Images


**Create VM** (Proxmox)  
Storage 256 GiB, Memory balloon from 4096->12000  
Select ubuntu ISO downloaded just previously


**Check VM IP and set manually at router:**
```
# Run in VM terminal
hostname -I  # 192.168.1.29

# Manually setting to 192.168.1.2
# Reboot VM

hostname -I # Check that it's the IP you manually set
```


**Enable SSH**  
Settings -> System -> Secure Shell -> Toggle On (Unsure if this is needed when installing openssh-server)
```
sudo apt update  
sudo apt upgrade -y
sudo apt install openssh-server -y
sudo systemctl enable ssh  
sudo systemctl start ssh

```


**Install Docker** (Ubuntu)  
https://docs.docker.com/desktop/setup/install/linux/ubuntu/
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Check status
sudo systemctl status docker
sudo systemctl enable docker

# Add user to docker group (fix permissions)
sudo usermod -aG docker $USER
newgrp docker
```


**Setup Git** (optional)
```
# Persist credentials. Optional 

git config --global credential.helper store
```

**Setup Portainer**
```
mkdir utils
cd utils
git clone https://github.com/team-hunting/portainer.git 

cd portainer
docker compose up -d
```

Access portainer at: https://192.168.1.2:9443/  
Need to use https, and make sure to use the correct IP of the host.

You can use portainer to setup GitOps automated deployments that watch git repositories and point towards a docker compose file.  
I'm using portainer to automatically deploy my main service (django app) as well as the tailscale sidecar. (In addition to other things)

**Setup Postgres**
```
sudo apt install postgresql postgresql-contrib
sudo systemctl status postgresql
sudo systemctl enable postgresql

# Set password
sudo -u postgres psql
ALTER USER postgres WITH PASSWORD 'your_new_password'

# Configure networking - update postgresql.conf
sudo nano /etc/postgresql/16/main/postgresql.conf

# update listen_addresses (near the top) to listen_addresses = 'localhost,192.168.1.2'

# Get docker subnet
docker network inspect bridge | grep Subnet # 172.17.0.0/16

# Update pg_hba.conf
sudo nano /etc/postgresql/16/main/pg_hba.conf

# Near the bottom, add:  (use md5 instead of scram-sha-256 if that's what the other entries use)
host all all 192.168.1.0/24 scram-sha-256  # local network
host all all 172.0.0.0/8 scram-sha-256     # docker subnets

```


**Create Postgres DB and User for Django**
```
Create login/role (for django to use)
Create database (for django to use - make the owner the login/role created just before)

# can do this visually by connecting with pgAdmin, or can run commands:

psql -U postgres
CREATE ROLE myuser WITH LOGIN PASSWORD 'mypassword';
CREATE DATABASE mydb OWNER myuser;
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;

# The values for "myuser", "mypassword", and "mydb" will all be passed as environment variables to django

```


**Run Portainer**
```
git clone https://github.com/team-hunting/portainer.git
cd portainer
docker compose up -d

# Alternately, here is the docker compose file:

> BOF

services:
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:lts
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    ports:
      - 9443:9443
      # - 8000:8000  # Remove if you do not intend to use Edge Agents

volumes:
  portainer_data:
    name: portainer_data

EOF

```


-------

# **Tailscale**

Set up a tailscale account and sign into the admin panel  
Generate an auth key (or use oauth key instead)  
This auth key will be used to set up autonomous tailscale sidecars

Run this sidecar using docker compose directly on the host, or run it with something like portainer.  
Make sure to inject the environment variables for TS_HOST_NAME and TS_AUTH_KEY and APP_PORT.  
TS_HOST_NAME will become the subdomain of your tailscale link, i.e. https://my-host-name.tail8367fd.ts.net/

If running your app (that will be exposed) in a docker container as well, make sure to set network_mode: "host" on the service that will be exposed.  
Setting network_mode: "host" doesn't work correctly on Windows hosts.  
To make this setup work on windows, you must include tailscale in the same docker compose file as the service you want to expose (and have them use the same network), or define an external docker network separately and set both services to use it.  
(Or run tailscale on the windows host directly.)  

Watch the logs for the tailscale container.  
Sometimes it may fail to get an SSL cert on first run and you can re-deploy without changing anything to fix it. Give it a few minutes before attempting.

```
git clone https://github.com/team-hunting/portainer.git
cd stacks

# Pass env vars in to the docker compose up command
# Base command = docker compose -f tailscale.yml up -d

TS_HOST_NAME=my-host-name TS_AUTH_KEY=my-auth-key APP_PORT=3000 docker compose -f tailscale.yml up -d

# OR - create .env file locally

docker compose --env-file ./.env -f tailscale.yml up -d


#####################

# Docker compose file
https://github.com/team-hunting/portainer/blob/master/stacks/tailscale.yml

> BOF

services:
  tailscale-sidecar:
    image: tailscale/tailscale:latest
    container_name: tailscale-sidecar
    # MUST use network_mode: "host" to see the 'web' service
    network_mode: "host"
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
    volumes:
      # Named volume to persist Tailscale state (IP, keys, etc.)
      - tailscale_state:/var/lib/tailscale
      - /var/run/tailscale:/var/run/tailscale
    environment:
      # *** REQUIRED: Replace this with your actual Tailscale Auth Key ***
      TS_AUTH_KEY: ${TS_AUTH_KEY}
      # Forces the container to use userspace networking mode
      TS_USERSPACE: "true"
      # Optional: Set a recognizable hostname for the node
      TS_HOSTNAME: ${TS_HOST_NAME}
    
    # This command funnels traffic to 127.0.0.1:{APP_PORT}
    # Because both containers are in "host" mode, 127.0.0.1
    # is the host's loopback address, where 'web' is listening.
    command: |
      sh -c "
        tailscaled --state=/var/lib/tailscale/tailscaled.state --socket=/var/run/tailscale/tailscaled.sock &

        # Give it a few seconds just to be safe
        sleep 5

        tailscale --socket=/var/run/tailscale/tailscaled.sock up --authkey=${TS_AUTH_KEY} --hostname=${TS_HOST_NAME} --accept-routes=false

        tailscale --socket=/var/run/tailscale/tailscaled.sock funnel ${APP_PORT}
      "
    restart: unless-stopped

volumes:
  tailscale_state: {}
  
EOF
```

Some resources say:
```
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
```
But:
```
If your host already has the `tun` kernel module loaded (which most modern systems do), Tailscale works perfectly fine without `sys_module`.
```

---
**Django Specifics:**  
Make sure to set the correct values for 'allowed_hosts'.
For example:
1. running at https://my-host-name.tail8367fd.ts.net/
2. accessible on localhost
3. Accessible on local network at 192.168.1.99
```
ALLOWED_HOSTS = localhost,127.0.0.1,192.168.1.99,my-host-name.tail8367fd.ts.net
```

---
# Tailscale can be run in a sidecar, or directly on the host.  

---

### Tailscale on Host - Optional  
  
**Install Tailscale on VM (not in container)**
```
curl -fsSL https://tailscale.com/install.sh | sh

sudo tailscale up
sudo systemctl enable --now tailscaled
```

**Start Tailscale Funnel manually**
```
# foreground (can help find your URL if you don't know it)
sudo tailscale funnel 8000

# Background
sudo tailscale funnel -bg 8000
```

**Persistent funnel using service**
```
sudo touch /etc/systemd/system/tailscale-django-funnel.service
sudo nano /etc/systemd/system/tailscale-django-funnel.service

# File content - change <port> for your port (8000)

[Unit]  
Description=Tailscale Funnel Service  
After=tailscaled.service  
Wants=tailscaled.service  
  
[Service]  
ExecStart=/usr/bin/tailscale funnel --bg <port>  
Restart=always  
User=root  
  
[Install]  
WantedBy=multi-user.target

# EOF


sudo systemctl enable --now tailscale-django-funnel
```
---

# **S3**  

Every app needs object storage these days. We'll host our own.  
I'm using Minio as a fully featured, open source S3 server.  
We'll run Minio through Portainer:  
```
services:
  minio:
    image: quay.io/minio/minio
    container_name: minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: ${S3_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${S3_ROOT_PASSWORD}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 10s
      retries: 5
    volumes:
      - s3_data:/data
    command: server /data --console-address ":9001"

  createbuckets:
    image: minio/mc
    container_name: initialize_minio
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
      /usr/bin/mc alias set myminio http://minio:9000 ${S3_ROOT_USER} ${S3_ROOT_PASSWORD};
      /usr/bin/mc mb myminio/${S3_USER_UPLOADS_BUCKET};
      /usr/bin/mc admin user add myminio ${S3_USER_UPLOADS_ACCESS_KEY} ${S3_USER_UPLOADS_SECRET_KEY};
      /usr/bin/mc admin policy attach myminio readwrite --user ${S3_USER_UPLOADS_ACCESS_KEY};
      exit 0;
      "

volumes:
  s3_data:
```

```
    # To troubleshoot minio, exec into the minio container and run:
    # mc alias set myminio http://127.0.0.1:9000
    # mc admin user -v -a myminio THEN enter ${S3_ROOT_USER} ${S3_ROOT_PASSWORD}
```

This stack includes an initialization sidecar to create a bucket and provision access for the main app.  

Minio has a user interface on port 9001, and the S3 API is served on port 9000.  
The way Minio and other S3 servers are designed to work is with pre-signed URLs.  
Your app can provide the client (end user) a pre-signed URL pointing to S3 that includes a temporary access credential. When your users can hit S3 directly, your app doesn't have to do streaming IO on every request by proxying every file and potentially bottlenecking your app.  

However, this means that we need to expose Minio as well as our app publicly.  
Since we're already running one tailscale funnel on this host, we'll opt to set up a reverse proxy and expose that using the tailscale funnel.   

Minio validates the host header in presigned URLs, so use nginx to rewrite the host to the internal IP for Minio. (see below)  

## **Reverse Proxy**  

Tailscale funnels can only serve 3 ports: 443, 8443, and 10000.  
This means you can only run 3 funnels on one host, maximum.  
Likely, you want your funnel listening on port 443 so that it can be reached easily using a web browser and HTTPS, as port 443 is the default port for HTTPS.  

There are a couple of options to get around this and expose more services.  
You can bake tailscale into every individual container you need to expose, and run every individual container as a tailscale node. If you need a different domain for each service, this could be an option.  
The recommended solution is to run a reverse proxy such as Nginx, Traefik, or Caddy, and point your tailscale funnel to the reverse proxy instead of directly at your app.  

With a configuration similar to the following, you can differentiate incoming requests and proxy them to the appropriate internal services:  
```
(nginx)

server {

    listen 80;

    # ----------- S3 ROUTE -----------
    location /s3/ {
        proxy_pass http://127.0.0.1:9000/;
        proxy_set_header Host 192.168.1.2:9000;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # ----------- APP ROUTE -----------
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```



---

Assorted notes:
Create Django Superuser for Admin Panel 
```
uv run python manage.py createsuperuser
# Follow prompts to create login (this login info will be stored in postgres)
```
