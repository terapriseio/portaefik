# Portaefik

A lightweight template for [Portainer](https://github.com/portainer/portainer) and [Traefik](https://github.com/traefik/traefik) - a modern reverse proxy and load balancer. This template focuses on exposing Docker services securely with automatic SSL certificate management through [Let's Encrypt](https://letsencrypt.org/).

## Key Features
- Automatic SSL/TLS certificate generation and renewal
- Secure Docker service exposure
- Dashboard access for monitoring


## Prerequisites
- docker
- git

Create Non-root user for security
```
# Create non root user
adduser leaf

# Add The New User to sudo Group 
usermod -aG sudo leaf

# Switch to new user
su leaf
```
Install required dependencies 
```
sudo curl -fsSL https://get.docker.com | bash
sudo usermod -aG docker leaf
sudo apt install git -y
sudo apt update -y
sudo apt upgrade -y
```
Log out and back in to refresh groups
```
# login to root
su root
```
```
# log back into non root
su leaf
```
```
# change to home dir
cd ~
```

## 1. Clone Repo
```
git clone https://github.com/terapriseio/portaefik.git
```

Then CD into the new directory
```
cd portaefik
```

## 2. Create and Edit `.env` file
### Copy the `.env.example`
There is an example .env in the repo. You can copy it to made adding the env variables easier.

From the `~/portaefik` directory, run:
```
cp .env.example .env # copies contents of .env.example to new .env file
```

### Generate Password Hash
Generate a password hash for Traefik dashboard. This is required to access your 
```
echo "HASHED_TRAEFIK_PASSWORD='$(openssl passwd -apr1 your-secure-password)'"  >> .env
```

### Add Your Env Vars
Change env values in the new `.env` file.
You should delete the placeholder "`HASHED_TRAEFIK_PASSWORD=hashed_password_here`"
```
nano .env # opens new .env file to edit
```

## 3. Add DNS Records
Add DNS A records pointing the correct domain names to the IP addresses
![Dns settings screenshot](/readme-images/dns-settings.png)

> ⚠️ The ssl certs will fail if the dns records aren't added and propagated before starting the containers

> ℹ️ I have had mixed results while running this behind Cloudflare's proxy. If you have issues, i recommend disabling any proxy, but I have services running just find behind a proxy as well.

## 4. Start Container
Start the container by the below command from the `~/portaefik` directory:
```
docker compose up -d
```

## 5. Access Portainer and Traefik Sites
Visit the site and set up your Portainer.

## 6. Connecting Services to Traefik
Now, you can set up and portainer stack or compose file to use traefik by adding the traefik config to the compose files.

### Add Required Labels
Add these labels to any container you want to expose through Traefik:
```
labels:
    # Enable Traefik for this container
    - "traefik.enable=true"
    # Route traffic based on domain name
    - "traefik.http.routers.${APP_NAME}.rule=Host(`${APP_DOMAIN}`)"
    # Enable automatic SSL
    - "traefik.http.routers.${APP_NAME}.tls.certresolver=letsencrypt"
    # Port your application runs on
    - "traefik.http.services.${APP_NAME}.loadbalancer.server.port=${APP_PORT}"
```

### Specify "proxy" Network
Add network to container (under container config):
```
  networks:
    - proxy
```
Declare network (at root level):
```
networks:
  proxy:
    external: true
```
### Complete Example
Here's a full docker-compose.yaml example using Uptime Kuma:
```
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    volumes:
      - uptime-kuma:/app/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.${APP_NAME}.rule=Host(`${APP_DOMAIN}`)"
      - "traefik.http.routers.${APP_NAME}.tls.certresolver=letsencrypt"
      - "traefik.http.services.${APP_NAME}.loadbalancer.server.port=${APP_PORT}"
    networks:
      - proxy
    restart: always

volumes:
  uptime-kuma:

networks:
  proxy:
    external: true
```
Then you need to define the env vars `APP_DOMAIN`, `APP_PORT`, and `APP_NAME` like:
```
APP_DOMAIN=whatsup.yourDomain.com # what your set in the dns record
APP_PORT=3001 # must be what your app actually uses
APP_NAME=uptime # any name unique in your traefik config
```

