# inception — 42KL

> A Docker-based web infrastructure built entirely from custom Dockerfiles — no pre-built images — running NGINX, WordPress, and MariaDB in isolated, networked containers.

## Overview

Modern software deployment increasingly relies on containers: isolated, reproducible environments that bundle an application with its dependencies. `inception` is a hands-on infrastructure project that requires building a small but complete web hosting stack from scratch using Docker and Docker Compose.

The twist is that convenience images like `wordpress:latest` or `nginx:latest` are **forbidden**. Every container must be built from a raw `debian:bullseye` or `alpine` base, with all software installed, configured, and initialised via shell scripts inside the Dockerfile. This forces you to understand what those convenience images do for you — and to do it yourself.

## The Challenge

Build and orchestrate three services using Docker Compose:

1. **NGINX** — the only container exposed to the outside world. Handles HTTPS (TLS 1.2/1.3 only, no HTTP) and proxies PHP requests to WordPress.
2. **WordPress** — a PHP-FPM application server (no Apache) initialised with WP-CLI. Not directly exposed to the internet.
3. **MariaDB** — a database server initialised with a non-root user, accessible only by WordPress over the internal Docker network.

Additional requirements: passwords must be passed via Docker secrets (files mounted at runtime), not hardcoded in environment variables. Data must persist across container restarts via named volumes. Containers must restart automatically on failure.

## Concepts Introduced

- **Docker images and layers**: how a `Dockerfile` builds a filesystem layer by layer
- **Docker Compose**: defining multi-container applications with dependencies, volumes, and networks in a single YAML file
- **Container networking**: isolated bridge networks, DNS resolution between services by container name, no direct host access
- **Docker secrets**: mounting sensitive data (passwords) as files rather than environment variables, preventing secret leakage in `docker inspect`
- **Named volumes**: persisting database and WordPress file data across container restarts and rebuilds
- **TLS termination**: generating self-signed certificates with OpenSSL and configuring NGINX to accept only modern TLS
- **PHP-FPM**: how NGINX and PHP communicate via the FastCGI protocol over a Unix socket / TCP port
- **Initialisation scripting**: using shell scripts as `ENTRYPOINT` or `CMD` to configure services at first run

## Learning Outcomes

After completing this project you will have:
- Built real Docker images from scratch, understanding every layer and package install
- Configured NGINX as a reverse proxy with TLS, serving a dynamic PHP application
- Set up a WordPress + MariaDB stack entirely from the command line using WP-CLI
- Managed secrets securely in a containerised environment
- Understood the difference between build-time (`ARG`) and runtime (`ENV`, secrets) configuration
- Written reusable infrastructure that can be torn down and rebuilt deterministically with a single command

## Architecture

```
Host machine (port 443)
       │ HTTPS
   ┌───┴────────────────────────────────────────┐
   │  NGINX container                           │
   │  - TLS 1.2/1.3 termination                │
   │  - Serves static files directly            │
   │  - Proxies PHP requests via FastCGI        │
   └───────────────────┬────────────────────────┘
                       │ FastCGI (port 9000)
   ┌───────────────────┴────────────────────────┐
   │  WordPress (PHP-FPM) container             │
   │  - Runs php-fpm, no web server             │
   │  - WP-CLI used for initial setup           │
   └───────────────────┬────────────────────────┘
                       │ TCP (port 3306)
   ┌───────────────────┴────────────────────────┐
   │  MariaDB container                         │
   │  - Initialised with non-root DB user       │
   │  - Data stored in a named volume           │
   └────────────────────────────────────────────┘

Networks:   wordpress_network (internal bridge)
Volumes:    wordpress_data   (wp files)
            db_data          (mariadb data directory)
Secrets:    db_root_password, db_user_password,
            wp_admin_password, wp_user_password
```

## Prerequisites

- Docker (20.x+) and Docker Compose (v2)
- `make`
- OpenSSL (for generating the self-signed TLS certificate)
- Linux or WSL2 on Windows

## How to Deploy

**1. Clone the repository and set up configuration:**

```bash
git clone https://github.com/yapjienfei/inception.git
cd inception
```

**2. Create your `.env` file** (copy from `srcs/sample_.env` and fill in values):

```env
DOMAIN_NAME=yourlogin.42.fr
DB_NAME=wordpress
DB_USER=wpuser
WP_TITLE=My Site
WP_ADMIN_USER=admin
WP_USER=editor
WP_USER_EMAIL=editor@example.com
```

**3. Create your secrets** (copy from `srcs/sample_secrets/`):

```bash
mkdir -p secrets
echo "strongrootpassword"  > secrets/db_root_password.txt
echo "stronguserpassword"  > secrets/db_user_password.txt
echo "adminpassword"       > secrets/wp_admin_password.txt
echo "editorpassword"      > secrets/wp_user_password.txt
```

**4. Start the stack:**

```bash
make        # builds all images and starts containers
```

**5. Access the site:**

Add an entry to `/etc/hosts` if using a local domain:
```
127.0.0.1  yourlogin.42.fr
```
Then open `https://yourlogin.42.fr` in a browser.

## Makefile Commands

| Command | Effect |
|---------|--------|
| `make` | Build all images and start containers |
| `make down` | Stop all containers |
| `make clean` | Stop containers and remove images |
| `make fclean` | Full teardown including volumes (destroys all data) |
| `make re` | Full rebuild from scratch |
| `make logs` | Tail logs of all running containers |

## Useful Docker Commands

```bash
# Check container status
docker compose -f srcs/docker-compose.yml ps

# Shell into a running container
docker exec -it wordpress bash
docker exec -it mariadb bash

# Inspect named volumes
docker volume ls
docker volume inspect inception_db_data

# View logs for a specific service
docker compose -f srcs/docker-compose.yml logs -f nginx
```
