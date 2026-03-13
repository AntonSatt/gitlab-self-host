# Self-Hosted GitLab CE (Docker)

A resource-optimized GitLab CE setup designed for small cloud servers (2-4 CPU cores, 4+ GB RAM). Runs GitLab in Docker behind an nginx reverse proxy with HTTPS via Let's Encrypt.

## Requirements

### Server
- **OS:** Ubuntu 22.04 or 24.04 LTS
- **CPU:** 2 cores minimum (4 recommended)
- **RAM:** 4 GB minimum (8+ recommended)
- **Disk:** 20 GB free minimum
- **Swap:** 2-4 GB recommended if RAM is limited

### Software (must be installed before setup)
- [Docker](https://docs.docker.com/engine/install/ubuntu/)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2, included with Docker Desktop or `docker-compose-plugin`)
- [Nginx](https://nginx.org/) (`sudo apt install nginx`)
- [Certbot](https://certbot.eff.org/) (`sudo apt install certbot python3-certbot-nginx`)
- A domain name pointed at your server's IP address

## Setup

### 1. Create swap (if you have less than 8 GB RAM)

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 2. Clone and configure

```bash
git clone https://github.com/YOUR_USERNAME/gitlab-self-hosted.git
cd gitlab-self-hosted
```

Edit `docker-compose.yml` and replace:
- `git.example.com` with your domain

### 3. Open firewall ports

```bash
# Open ports for web and SSH access
sudo iptables -I INPUT -p tcp --dport 8080 -m state --state NEW -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 2222 -m state --state NEW -j ACCEPT
sudo netfilter-persistent save
```

> **Cloud providers (AWS, Oracle, GCP, etc.):** You also need to open ports 8080 and 2222 in your cloud provider's security group / security list / firewall rules.

### 4. Start GitLab

```bash
docker compose up -d
```

GitLab takes 3-5 minutes to boot on a small server. Check progress with:

```bash
docker logs -f gitlab
```

### 5. Set up nginx reverse proxy

Copy the example config:

```bash
sudo cp nginx.conf.example /etc/nginx/sites-available/git.example.com
sudo ln -s /etc/nginx/sites-available/git.example.com /etc/nginx/sites-enabled/
```

Edit `/etc/nginx/sites-available/git.example.com` and replace `git.example.com` with your domain.

Test and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 6. Set up HTTPS with Let's Encrypt

```bash
sudo certbot --nginx -d git.example.com
```

Certbot will automatically configure SSL in your nginx config.

### 7. Get the initial root password

```bash
docker exec gitlab cat /etc/gitlab/initial_root_password
```

Log in at `https://your-domain.com` with:
- **Username:** `root`
- **Password:** (from the command above)

> **Important:** Change the root password immediately. This file is automatically deleted after 24 hours.

### 8. Create your user account

1. Log in as root
2. Go to **Admin Area** > **Overview** > **Users** > **New user**
3. Create your personal account
4. Optionally grant it Admin role
5. Use your personal account for day-to-day work

## What's optimized

This configuration disables several default GitLab services to reduce CPU and memory usage:

| Service | Status | Why |
|---------|--------|-----|
| Puma (web server) | Single mode | Avoids forking overhead on low-core servers |
| Prometheus monitoring | Disabled | Saves ~200 MB RAM |
| Grafana | Removed (deprecated) | No longer supported in latest GitLab |
| Container registry | Disabled | Enable when needed |
| All exporters | Disabled | Not needed without Prometheus |

## File structure

```
.
├── docker-compose.yml      # GitLab container configuration
├── nginx.conf.example      # Reverse proxy template
├── README.md
├── config/                  # GitLab config (created on first run)
├── logs/                    # GitLab logs (created on first run)
└── data/                    # GitLab data (created on first run)
```

## Useful commands

```bash
# Check GitLab status
docker exec gitlab gitlab-ctl status

# Restart GitLab
docker compose restart

# Stop GitLab
docker compose down

# View logs
docker logs -f gitlab

# Access GitLab Rails console
docker exec -it gitlab gitlab-rails console

# Reconfigure after changing settings
docker exec gitlab gitlab-ctl reconfigure
```

## Troubleshooting

**502 error on first boot**
Normal. GitLab takes 3-5 minutes to fully start on small servers. The page auto-refreshes.

**Puma keeps restarting**
If using `external_url` with a custom port (e.g., `http://example.com:8080`), Puma tries to bind to that port directly and conflicts with itself. Use a standard port in `external_url` and let nginx handle the mapping.

**SSH clone uses wrong port**
Set `gitlab_rails['gitlab_shell_ssh_port']` in `docker-compose.yml` to match your exposed SSH port.

**422 error on login**
Usually a CSRF/protocol mismatch. Make sure your nginx config passes `X-Forwarded-Proto https` and `X-Forwarded-Ssl on` headers.
