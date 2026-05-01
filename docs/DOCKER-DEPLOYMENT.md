# Docker Deployment Guide — dcm4chee-arc-light

Complete guide to deploy dcm4chee Archive 5.x with Docker Compose and Nginx reverse proxy.

## Prerequisites

- Linux server (Ubuntu 22.04+ recommended)
- Docker Engine 24+ and Docker Compose v2+
- Nginx (system package, not containerized)
- 4GB+ RAM available, 20GB+ disk
- Port 80 open (HTTP)
- Port 11112 open (DICOM, optional — only if receiving DICOM from external modalities)

## Quick Start

```bash
ssh user@server
mkdir -p ~/dcm4chee && cd ~/dcm4chee
```

### 1. Create docker-compose.yml

```yaml
services:
  ldap:
    image: dcm4che/slapd-dcm4chee:2.6.10-35.0
    container_name: dcm4chee-ldap
    volumes:
      - ldap-data:/var/lib/openldap/openldap-data
      - ldap-config:/etc/openldap/slapd.d
    restart: unless-stopped

  db:
    image: dcm4che/postgres-dcm4chee:17.4-34
    container_name: dcm4chee-db
    environment:
      POSTGRES_DB: pacsdb
      POSTGRES_USER: pacs
      POSTGRES_PASSWORD: pacs
    volumes:
      - db-data:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pacs -d pacsdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  arc:
    image: dcm4che/dcm4chee-arc-psql:5.34.3
    container_name: dcm4chee-arc
    ports:
      - "127.0.0.1:8080:8080"
      - "127.0.0.1:8443:8443"
      - "127.0.0.1:9990:9990"
      - "0.0.0.0:11112:11112"
      - "0.0.0.0:2762:2762"
      - "0.0.0.0:2575:2575"
      - "0.0.0.0:12575:12575"
    environment:
      POSTGRES_DB: pacsdb
      POSTGRES_USER: pacs
      POSTGRES_PASSWORD: pacs
      POSTGRES_HOST: db
      WILDFLY_CHOWN: /opt/wildfly/standalone /storage
      WILDFLY_WAIT_FOR: ldap:389 db:5432
      JAVA_OPTS: -Xms256m -Xmx1g -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m
    depends_on:
      ldap:
        condition: service_started
      db:
        condition: service_healthy
    volumes:
      - wildfly-data:/opt/wildfly/standalone
      - storage-data:/storage
    restart: unless-stopped

volumes:
  ldap-data:
  ldap-config:
  db-data:
  wildfly-data:
  storage-data:
```

Key design decisions:
- HTTP ports (8080, 8443, 9990) bind to `127.0.0.1` only — not accessible from outside
- DICOM port (11112) binds to `0.0.0.0` — accessible from modalities on the network
- HL7 ports (2575, 12575) and DICOM TLS (2762) exposed for integration
- PostgreSQL and LDAP have no exposed ports — internal only

### 2. Start the stack

```bash
cd ~/dcm4chee
docker compose up -d
```

WildFly takes 30-60 seconds to start. Monitor with:

```bash
docker logs dcm4chee-arc -f
# Wait for: "WFLYSRV0025: WildFly Full ... started in XXXXXms"
```

### 3. Verify containers

```bash
docker ps
# Should show 3 containers: dcm4chee-arc, dcm4chee-db (healthy), dcm4chee-ldap
```

### 4. Test REST API (from the server)

```bash
curl -s http://localhost:8080/dcm4chee-arc/aets | python3 -m json.tool
# Should return 8 AE titles: DCM4CHEE, AS_RECEIVED, WORKLIST, IOCM_*
```

## Nginx Reverse Proxy (port 80)

### 5. Create Nginx config

```bash
sudo tee /etc/nginx/sites-available/dcm4chee > /dev/null << 'EOF'
server {
    listen 80;
    server_name dcm4chee.yourdomain.com;

    client_max_body_size 0;
    proxy_read_timeout 600s;
    proxy_send_timeout 600s;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
EOF
```

Configuration notes:
- `client_max_body_size 0` — no upload limit (DICOM files can be large)
- `proxy_read_timeout 600s` — WADO-RS retrievals can be slow for large studies
- WebSocket headers for streaming support

### 6. Enable and reload

```bash
sudo ln -sf /etc/nginx/sites-available/dcm4chee /etc/nginx/sites-enabled/dcm4chee
sudo nginx -t
sudo systemctl reload nginx
```

### 7. DNS / hosts entry

Add to `/etc/hosts` on the server:

```
127.0.0.1 dcm4chee.yourdomain.com
```

Add to `/etc/hosts` on client machines:

```
<SERVER_IP> dcm4chee.yourdomain.com
```

Or configure DNS if you have a domain.

### 8. Verify external access

```bash
curl -s http://dcm4chee.yourdomain.com/dcm4chee-arc/aets
curl -s http://dcm4chee.yourdomain.com/dcm4chee-arc/ui2/
```

## Endpoints

| Endpoint | URL | Protocol |
|----------|-----|----------|
| Web UI | `http://dcm4chee.yourdomain.com/dcm4chee-arc/ui2/` | HTTP (via Nginx) |
| REST API | `http://dcm4chee.yourdomain.com/dcm4chee-arc/aets` | HTTP (via Nginx) |
| WADO-RS | `http://dcm4chee.yourdomain.com/dcm4chee-arc/aets/DCM4CHEE/rs` | HTTP (via Nginx) |
| DICOM C-STORE | `DCM4CHEE@server:11112` | DICOM (direct) |
| DICOM C-FIND | `DCM4CHEE@server:11112` | DICOM (direct) |
| WildFly Admin | `http://localhost:9990` | HTTP (localhost only) |

## DICOM AE Titles

| AE Title | Purpose |
|----------|---------|
| DCM4CHEE | Primary — hide rejected instances |
| AS_RECEIVED | Retrieve all instances including rejected |
| WORKLIST | Modality and Unified Worklist |
| IOCM_QUALITY | Only rejected for Quality Reasons |
| IOCM_EXPIRED | Only rejected for Data Retention Expired |
| IOCM_PAT_SAFETY | Only rejected for Patient Safety |
| IOCM_WRONG_MWL | Only rejected for Incorrect MWL Entry |
| IOCM_REGULAR_USE | Show instances rejected for Quality Reasons |

## Operations

### Start / Stop

```bash
cd ~/dcm4chee
docker compose up -d      # start
docker compose down        # stop (preserves data)
docker compose down -v     # stop and DELETE all data
```

### Logs

```bash
docker logs dcm4chee-arc -f          # WildFly/archive logs
docker logs dcm4chee-db -f           # PostgreSQL logs
docker logs dcm4chee-ldap -f         # LDAP logs
```

### Health check

```bash
curl -s http://localhost:8080/dcm4chee-arc/aets | head -c 50
# Should return JSON array of AE titles
```

### Backup

```bash
# Database
docker exec dcm4chee-db pg_dump -U pacs pacsdb > backup-$(date +%Y%m%d).sql

# DICOM storage (files)
docker cp dcm4chee-arc:/storage ./storage-backup/

# LDAP configuration
docker cp dcm4chee-ldap:/var/lib/openldap/openldap-data ./ldap-backup/
```

### Restore

```bash
# Database
cat backup-20260501.sql | docker exec -i dcm4chee-db psql -U pacs pacsdb

# Restart after restore
docker compose restart arc
```

### Update to newer version

```bash
cd ~/dcm4chee
# Edit docker-compose.yml — change image tags
docker compose pull
docker compose up -d
```

## Troubleshooting

**WildFly won't start**: Check logs for LDAP/DB connectivity:
```bash
docker logs dcm4chee-arc 2>&1 | grep -i "error\|failed\|exception" | tail -20
```

**502 Bad Gateway from Nginx**: WildFly is still starting. Wait 60 seconds.

**DICOM C-STORE rejected**: Verify the sending AE title is configured:
```bash
curl -s http://localhost:8080/dcm4chee-arc/aets/DCM4CHEE/rs/config
```

**Out of memory**: Increase JAVA_OPTS in docker-compose.yml:
```yaml
JAVA_OPTS: -Xms512m -Xmx2g -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m
```

**Disk full**: Check DICOM storage usage:
```bash
docker exec dcm4chee-arc du -sh /storage
```
