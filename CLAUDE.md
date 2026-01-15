# Booklore Fork Project

## Project Overview
This is a fork of [booklore-app/booklore](https://github.com/booklore-app/booklore) with custom enhancements for auto-import functionality from the bookdrop folder.

## Infrastructure Details

### UNRAID Server (jack.server)
- **Host:** jack.server (192.168.1.141)
- **SSH User:** root
- **SSH Password:** ?xFQxYDd$PRaPr#4
- **SSH Command:** `sshpass -p '?xFQxYDd$PRaPr#4' ssh -o StrictHostKeyChecking=no root@jack.server`

### Gitea Server
- **Internal URL:** http://192.168.1.141:3030
- **External URL:** https://git.taranasus.xyz
- **Username:** gitea@taranasus.xyz
- **Password:** Fp?j8PDG?Pe4keJP
- **Database:** PostgreSQL on 192.168.1.141:5433
- **Repository:** https://git.taranasus.xyz/taranasus/booklore

### CI/CD Runner VM (unity-runner)
- **IP Address:** 192.168.1.250
- **SSH User:** runner
- **SSH Password:** zxcvbn
- **SSH Command:** `sshpass -p 'zxcvbn' ssh -o StrictHostKeyChecking=no runner@192.168.1.250`
- **OS:** Ubuntu 22.04.5 LTS
- **Runner Type:** Gitea Actions (act_runner v0.2.11)
- **Runner Label:** ubuntu-latest:host
- **Workspace:** /home/runner/workspace

### Current Booklore Docker Configuration
- **Container Name:** booklore
- **Image:** booklore/booklore:latest (currently v1.17.0)
- **Port:** 6060
- **Network:** bridge
- **Restart Policy:** unless-stopped

**Volume Mounts:**
- `/mnt/user/books` -> `/books` (main library)
- `/mnt/user/appdata/booklore/bookdrop` -> `/bookdrop` (auto-import folder)
- `/mnt/user/appdata/booklore/data` -> `/app/data` (application data)

**Environment Variables:**
- `DATABASE_URL=jdbc:mariadb://192.168.1.141:3306/booklore`
- `DATABASE_USERNAME=booklore`
- `DATABASE_PASSWORD=BookL0reDB2026!`
- `BOOKLORE_PORT=6060`
- `TZ=Europe/London`

**UNRAID Template Location:** `/boot/config/plugins/dockerMan/templates-user/my-booklore.xml`

## Technology Stack
- **Backend:** Java 21 with Spring Boot 3.5.8
- **Frontend:** Angular (Node.js 22)
- **Database:** MariaDB
- **Build Tool:** Gradle 8.14.3
- **Container Runtime:** Docker

## Repository Structure
- `booklore-api/` - Spring Boot backend (Gradle project)
- `booklore-ui/` - Angular frontend
- `Dockerfile` - Multi-stage build (Node -> Gradle -> JRE runtime)
- `nginx.conf` - Web server configuration
- `start.sh` - Container startup script

## Feature Goal: Auto-Import from Bookdrop
The goal is to add a toggle in settings that enables automatic import of books from the bookdrop folder when metadata is successfully found. Books without successful metadata matches should remain in bookdrop for manual review.

## Important Notes
- The upstream repository uses `develop` as the default branch
- Always keep the fork synchronized with upstream for PR compatibility
- CI/CD should build and deploy to UNRAID, replacing the existing container
- UNRAID requires XML template files at `/boot/config/plugins/dockerMan/templates-user/`
