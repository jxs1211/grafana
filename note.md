### Deploying Grafana in Production Using Docker Compose: Best Practices

Grafana is a powerful open-source tool for monitoring and visualization. Deploying it in production with Docker Compose requires focusing on reliability, security, scalability, and data persistence. While Docker Compose is suitable for small to medium production setups, for large-scale environments, consider orchestrators like Docker Swarm or Kubernetes. Below is a step-by-step guide based on official recommendations and community best practices.

#### Prerequisites
- Docker and Docker Compose installed on your host machine.
- A domain or subdomain (e.g., `grafana.example.com`) for external access.
- NGINX or a similar reverse proxy for HTTPS termination.
- Certbot (or equivalent) for SSL certificates.
- Basic knowledge of YAML configuration.
- For production, avoid using the default SQLite database; use an external database like PostgreSQL for better performance and concurrency.

#### Step 1: Set Up Persistent Storage and User Permissions
Create a directory for Grafana's data to ensure persistence across container restarts. This prevents data loss for dashboards, users, and plugins.

```bash
mkdir -p grafana-data
```

Find your host user's UID to run the container as a non-root user, reducing security risks:

```bash
id -u
```

Note the output (e.g., 1000) for use in the Compose file.

#### Step 2: Configure an External Database (Recommended for Production)
For production, switch from the default embedded SQLite to PostgreSQL or MySQL to handle higher loads and enable features like high availability. Here's how to include PostgreSQL in your setup.

#### Step 3: Create the `docker-compose.yaml` File
Use the Grafana OSS or Enterprise image (Enterprise is free and includes OSS features). Specify a fixed version tag for stability. Include environment variables for configuration, and use Docker volumes or bind mounts for persistence.

Here's a production-ready example that includes Grafana and a PostgreSQL database:

```yaml
version: '3.8'

services:
  grafana:
    image: grafana/grafana-oss:11.2.2  # Use a specific version; replace with grafana/grafana-enterprise if needed
    container_name: grafana
    restart: unless-stopped
    user: "1000"  # Replace with your host UID to run as non-root
    depends_on:
      - db
    ports:
      - "3000:3000"  # Expose internally; use reverse proxy for external access
    volumes:
      - ./grafana-data:/var/lib/grafana  # Bind mount for persistence
    environment:
      - GF_SERVER_ROOT_URL=https://grafana.example.com  # Use your domain; enables correct URL rendering
      - GF_SECURITY_ADMIN_PASSWORD__FILE=/run/secrets/grafana_admin_password  # Use secrets for sensitive data
      - GF_DATABASE_TYPE=postgres
      - GF_DATABASE_HOST=db:5432
      - GF_DATABASE_NAME=grafana
      - GF_DATABASE_USER=grafana
      - GF_DATABASE_PASSWORD__FILE=/run/secrets/db_password
      - GF_SECURITY_ALLOW_EMBEDDING=false  # Disable embedding to prevent clickjacking
      - GF_SECURITY_COOKIE_SAMESITE=strict  # Mitigate CSRF
      - GF_LOG_LEVEL=info  # Set to debug for troubleshooting
    secrets:
      - grafana_admin_password
      - db_password

  db:
    image: postgres:16-alpine  # Lightweight Alpine variant for security
    container_name: grafana-db
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=grafana
      - POSTGRES_USER=grafana
      - POSTGRES_PASSWORD__FILE=/run/secrets/db_password
    secrets:
      - db_password

volumes:
  pgdata: {}  # Named volume for database persistence

secrets:
  grafana_admin_password:
    file: ./secrets/grafana_admin_password.txt  # Create this file with a strong password
  db_password:
    file: ./secrets/db_password.txt  # Create this file with a strong password
```

**Key Explanations:**
- **Image and Version:** Pin to a specific version to avoid breaking changes during updates.
- **Restart Policy:** `unless-stopped` ensures the container auto-starts unless manually stopped.
- **User:** Running as non-root prevents privilege escalation vulnerabilities.
- **Volumes:** Persist data outside the container to survive restarts or upgrades.
- **Environment Variables:** Override defaults for security and database settings. Use `__FILE` suffixes with Docker secrets for sensitive values like passwords.
- **Database Integration:** Connects Grafana to PostgreSQL for production-grade storage.
- **Secrets:** Store credentials in files mounted as secrets, avoiding exposure in logs or process lists.

Create the secret files:
```bash
echo "strong_admin_password" > ./secrets/grafana_admin_password.txt
echo "strong_db_password" > ./secrets/db_password.txt
chmod 600 ./secrets/*.txt  # Restrict permissions
```

#### Step 4: Start the Services
Validate the Compose file:
```bash
docker compose config
```

Start in detached mode:
```bash
docker compose up -d
```

Monitor logs for issues:
```bash
docker compose logs -f grafana
```

#### Step 5: Secure with a Reverse Proxy and HTTPS
Expose Grafana via a reverse proxy like NGINX for SSL, rate limiting, and additional security. Do not expose port 3000 directly to the internet.

Create `/etc/nginx/sites-available/grafana` with:
```
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen 80;
    server_name grafana.example.com;  # Your domain
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name grafana.example.com;

    ssl_certificate /etc/letsencrypt/live/grafana.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/grafana.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/live/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_pass http://localhost:3000;
    }
}
```

Enable and reload NGINX:
```bash
sudo ln -s /etc/nginx/sites-available/grafana /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Install SSL with Certbot:
```bash
sudo certbot --nginx -d grafana.example.com
```

This enforces HTTPS and supports WebSocket for real-time features.

#### Step 6: Access and Initial Setup
- Navigate to `https://grafana.example.com`.
- Log in with `admin` and the password from your secret file.
- Immediately change the admin password and set up additional users or OAuth/LDAP for authentication.
- Provision datasources and dashboards via YAML files in `/etc/grafana/provisioning` (mount this as a volume if needed).

#### Step 7: Maintenance and Scaling
```sh
# 每日凌晨 2 点备份
0 2 * * * docker exec grafana sh -c 'tar czf - /var/lib/grafana' | \
  aws s3 cp - s3://backup/grafana/$(date +\%F).tar.gz
# alert: Prometheus + Alertmanager
- alert: GrafanaDown
  expr: up{job="grafana"} == 0
  for: 1m
```

#### Additional Best Practices
- **Security Enhancements:** Enable brute-force protection, strict transport security, and content security policy in environment vars. Use least-privilege principles for the container.
- **Plugins:** Pre-install via `GF_INSTALL_PLUGINS` in a custom Dockerfile for consistency.
- **Monitoring and Logging:** Monitor the Grafana container itself with tools like Prometheus. Rotate logs and set appropriate levels.
- **Backups:** Regularly back up the volume (e.g., `grafana-data` and `pgdata`) using `rsync` or Docker volume backups. Schedule database dumps.
- **Updates:** Test updates in staging. Pull new images and recreate containers with `docker compose up -d --pull always`.
- **Scaling:** For high traffic, add replicas behind a load balancer, but ensure shared storage (e.g., via NFS) or use Grafana's clustering features with an external DB.
- **Performance:** Tune database connection pools and limit concurrent sessions. Use Alpine-based images for smaller footprints.
- **Common Pitfalls:** Avoid running as root, ensure secrets are not committed to version control, and validate configurations before deployment.

This setup provides a secure, persistent, and scalable Grafana deployment. For advanced features like high availability, refer to Grafana's Enterprise edition. If integrating with monitoring stacks (e.g., Prometheus), add those services to the Compose file.
