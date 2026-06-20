# Nexlayer Build Failure Report

**Pipeline:** 19ee700c870
**Repository:** https://github.com/armondhonore/coolify
**Error category:** 
**Error summary:** pipeline: create job: create job: status 409: {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"jobs.batch \"pipeline-19ee700c-fix8\" already exists","reason":"AlreadyExists","details":{"name":"pipeline-19ee700c-fix8","group":"batch","kind":"jobs"},"code":409}

## Build log
```

```

## Repository build artifacts

These are the actual files from the repository. Use these to understand how the project
is SUPPOSED to be built — do not rely solely on the broken Dockerfile below.


### package.json
```
{
    "name": "coolify",
    "private": true,
    "type": "module",
    "scripts": {
        "dev": "vite",
        "build": "vite build"
    },
    "devDependencies": {
        "@tailwindcss/postcss": "4.1.18",
        "laravel-vite-plugin": "2.0.1",
        "postcss": "8.5.15",
        "tailwind-scrollbar": "4.0.2",
        "tailwindcss": "4.1.18",
        "vite": "7.3.2"
    },
    "dependencies": {
        "@tailwindcss/forms": "0.5.10",
        "@tailwindcss/typography": "0.5.16",
        "@xterm/addon-fit": "0.10.0",
        "@xterm/xterm": "5.5.0",
        "playwright": "^1.58.2"
    }
}

```

### docker-compose.yml
```
services:
    coolify:
        container_name: coolify
        restart: always
        working_dir: /var/www/html
        extra_hosts:
            - host.docker.internal:host-gateway
        networks:
            - coolify
        depends_on:
            - postgres
            - redis
            - soketi
    postgres:
        image: postgres:15-alpine
        container_name: coolify-db
        restart: always
        networks:
            - coolify
    redis:
        image: redis:7-alpine
        container_name: coolify-redis
        restart: always
        networks:
            - coolify
    soketi:
        container_name: coolify-realtime
        extra_hosts:
            - host.docker.internal:host-gateway
        restart: always
        networks:
            - coolify
networks:
    coolify:
        name: coolify
        driver: bridge
        external: false

```


## Last attempted Dockerfile
```dockerfile
FROM mirror.gcr.io/library/php:8.2-apache

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev \
    libzip-dev \
    unzip \
    git \
    curl \
    && docker-php-ext-install pdo pdo_pgsql zip

# Install Composer
COPY --from=mirror.gcr.io/library/composer:latest /usr/bin/composer /usr/bin/composer

# Install Node.js (using minimal installation to avoid timeout/resource bloat)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && apt-get install -y nodejs

WORKDIR /var/www/html

# Copy only dependency files first to leverage Docker cache
COPY package*.json composer*.json ./

# Install PHP dependencies without scripts or autoloader to prevent early failure
RUN composer install --no-dev --no-scripts --no-autoloader || true

# Install JS dependencies with --ignore-scripts to avoid husky/git hooks blocking the build
RUN npm install --ignore-scripts || true

# Copy the rest of the application
COPY . .

# Build assets - use a failure-tolerant approach for Vite
# We set NODE_OPTIONS to avoid OOM during the Vite build process
RUN NODE_OPTIONS="--max-old-space-size=4096" npm run build || true

# Finalize PHP autoloader
RUN composer dump-autoload --optimize || true

# Set permissions for Laravel storage and cache
RUN chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache

# Apache config for Laravel
ENV APACHE_DOCUMENT_ROOT /var/www/html/public
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf
RUN a2enmod rewrite

EXPOSE 80
ENV PORT=80
ENV HOSTNAME=0.0.0.0

CMD ["apache2-foreground"]
```

## Last attempted nexlayer.yaml
```yaml
application:
  name: coolify
  pods:
    - name: app
      image: "# filled by pipeline"
      servicePorts:
        - 80
      vars:
        APP_ENV: "production"
        APP_DEBUG: "false"
    - name: postgres
      image: mirror.gcr.io/library/postgres:15-alpine
      servicePorts:
        - 5432
      vars:
        POSTGRES_DB: coolify
        POSTGRES_USER: coolify
        POSTGRES_PASSWORD: password
    - name: redis
      image: mirror.gcr.io/library/redis:7-alpine
      servicePorts:
        - 6379
    - name: soketi
      image: mirror.gcr.io/library/soketi:latest
      servicePorts:
        - 6001
```

## Instructions for frontier model

CRITICAL: Before writing any fix, read the repository build artifacts above and answer:
1. What language/runtime does this project use? (go.mod, package.json, pom.xml, Cargo.toml, requirements.txt)
2. What is the actual build command? (package.json scripts.build, Makefile targets, pom.xml goals, gradle tasks)
3. What is the actual start command? (package.json scripts.start, Makefile run target, Procfile)
4. What port does it serve? (EXPOSE, ENV PORT=, --port flag, framework default)
5. What dependencies does it need at runtime? (docker-compose.yml services, .env.example vars)

Then create a correct Dockerfile from scratch based on your analysis:
- All FROM base images must be standard public images (library/, gcr.io, ghcr.io, etc.)
- Use `mirror.gcr.io/library/` prefix for Docker Hub official images (node:*, python:*, golang:*, etc.)
- DO NOT copy broken steps from the "last attempted Dockerfile" — build from what the repo actually needs

Fix nexlayer.yaml if needed:
- Inter-pod service references MUST use `<podName>.pod:<port>` addressing (resolved by the platform via DNS at deploy time)
- Example: `DATABASE_URL: postgresql://user:pass@postgres.pod:5432/db`

Create a file named `nexlayer_fix.md` on THIS branch (`nexlayer`) with this structure:

---
# Nexlayer Fix

## Fixed Dockerfile
```dockerfile
<your fixed Dockerfile>
```

## Fixed nexlayer.yaml
```yaml
<your fixed nexlayer.yaml>
```

## Notes
<explain: what build command you found, what was wrong with the previous Dockerfile, what you changed and why>
---

Nexlayer detects `nexlayer_fix.md` on the next pipeline run and applies your fixes automatically.
