# What is Docker Compose?

Docker Compose is a tool for defining and running one or more Docker containers using a YAML file.

Instead of writing a long `docker run` command for every container, you describe the application's services, ports, networks, volumes, and environment variables in one file. You can then start the complete application with one command.

```text
compose.yaml -> docker compose up -> Services -> Containers
```

Docker Compose is useful for applications that have multiple parts, such as:

- A frontend or web application
- An API server
- A database
- A cache such as Redis

Even a single-container application can use Compose to keep its configuration repeatable.

> Modern Docker uses the command `docker compose`. The older `docker-compose` command may still appear in tutorials.

## Dockerfile vs Docker Compose

The two files have different purposes:

| Dockerfile | Compose file |
| --- | --- |
| Defines how to build one Docker image | Defines how containers run and work together |
| Installs dependencies and copies application files | Configures services, ports, volumes, and networks |
| Used by `docker build` | Used by `docker compose` |

A Compose service can build an image from a Dockerfile or use an existing image from a registry.

## How to write a Compose file

Create a file named `compose.yaml` in the root of your project. `docker-compose.yml` is also supported, but `compose.yaml` is the preferred modern name.

For the Flask application in this project, the folder structure is:

```text
Docker/
|-- compose.yaml
`-- frontend/
    |-- app.py
    |-- Dockerfile
    |-- requirements.txt
    `-- templates/
```

Add the following content to `compose.yaml`:

```yaml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "5090:5090"
    environment:
      FLASK_ENV: development
    restart: unless-stopped
```

From the project root, validate the file:

```bash
docker compose config
```

Build and start the application:

```bash
docker compose up --build
```

Open `http://localhost:5090` in a browser.

Press `Ctrl+C` to stop the attached application, or start it in the background with:

```bash
docker compose up --build -d
```

## Compose file components

### services

`services` contains the containers that make up the application. Each service name should describe its role, such as `frontend`, `api`, `database`, or `redis`.

```yaml
services:
  frontend:
    image: nginx:alpine
```

### image

`image` tells Compose to use or download an existing image.

```yaml
services:
  database:
    image: postgres:17
```

Pin a specific image version instead of using `latest` so builds are predictable.

### build

`build` tells Compose to create an image from a Dockerfile.

```yaml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
```

- `context` is the directory sent to Docker during the build.
- `dockerfile` is the Dockerfile path relative to the build context.

### ports

`ports` publishes a container port on the host machine.

```yaml
ports:
  - "5090:5090"
```

The format is `HOST_PORT:CONTAINER_PORT`. Quoting port mappings is recommended because YAML may otherwise interpret some values unexpectedly.

### environment

`environment` passes environment variables to a container.

```yaml
environment:
  APP_ENV: development
  LOG_LEVEL: info
```

Do not commit passwords or API keys directly in the Compose file. A local `.env` file can supply variable values:

```dotenv
DATABASE_PASSWORD=change-me
```

Reference the variable in `compose.yaml`:

```yaml
environment:
  DATABASE_PASSWORD: ${DATABASE_PASSWORD}
```

Add `.env` to `.gitignore` when it contains secrets.

### volumes

Volumes preserve data after a container is removed.

```yaml
services:
  database:
    image: postgres:17
    volumes:
      - database_data:/var/lib/postgresql/data

volumes:
  database_data:
```

A bind mount maps a local directory into a container and is often useful during development:

```yaml
volumes:
  - ./frontend:/app
```

### networks

Compose automatically creates a network for the application. Services can reach each other by service name.

For example, an application connects to a service named `database` using the hostname `database`, not `localhost`. Inside a container, `localhost` refers to that same container.

### depends_on

`depends_on` controls service startup order.

```yaml
services:
  api:
    depends_on:
      database:
        condition: service_healthy
```

Startup order alone does not guarantee that a dependency is ready. Add a health check when another service must wait for it to accept connections.

### healthcheck

`healthcheck` defines how Docker checks whether a service is ready and working.

```yaml
services:
  database:
    image: postgres:17
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 5
```

### restart

`restart` defines when Docker restarts a stopped container.

```yaml
restart: unless-stopped
```

Common policies are `no`, `always`, `on-failure`, and `unless-stopped`.

## Multi-service example

The following example runs an application with PostgreSQL:

```yaml
services:
  frontend:
    build:
      context: ./frontend
    ports:
      - "5090:5090"
    environment:
      DATABASE_URL: postgresql://appuser:${DATABASE_PASSWORD}@database:5432/appdb
    depends_on:
      database:
        condition: service_healthy
    restart: unless-stopped

  database:
    image: postgres:17
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DATABASE_PASSWORD}
    volumes:
      - database_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  database_data:
```

Create a `.env` file beside `compose.yaml` before starting this example:

```dotenv
DATABASE_PASSWORD=use-a-strong-local-password
```

The Flask application would also need PostgreSQL client code that reads `DATABASE_URL` before using this larger example.

## Common Docker Compose commands

Run these commands from the directory containing `compose.yaml`.

```bash
# Validate and display the resolved configuration
docker compose config

# Build service images
docker compose build

# Create and start services
docker compose up

# Build images and start services in the background
docker compose up --build -d

# List running services
docker compose ps

# Follow logs from all services
docker compose logs -f

# Follow logs from one service
docker compose logs -f frontend

# Run a command inside a running service
docker compose exec frontend sh

# Stop and remove containers and networks
docker compose down

# Also remove named volumes and their stored data
docker compose down --volumes
```

Be careful with `docker compose down --volumes`: it permanently deletes data stored in the project's named volumes.

## Important YAML rules

- Use spaces for indentation, not tabs.
- Keep indentation consistent, usually two spaces per level.
- Add a space after each colon.
- Use a hyphen for list items such as ports and volumes.
- Quote port mappings.
- Service names must be unique within the file.
- A top-level `version` field is no longer required by modern Docker Compose.

Incorrect indentation:

```yaml
services:
frontend:
ports:
- "5090:5090"
```

Correct indentation:

```yaml
services:
  frontend:
    ports:
      - "5090:5090"
```

## Best practices

- Use specific image versions instead of `latest`.
- Keep passwords and secrets out of source control.
- Use named volumes for persistent database data.
- Add health checks for services that other services depend on.
- Use service names instead of fixed container IP addresses.
- Validate changes with `docker compose config`.
- Use a `.dockerignore` file to reduce the Docker build context.
- Commit `compose.yaml`, but do not commit secret `.env` files.

## Quick checklist

- Docker and Docker Compose are installed.
- The file is named `compose.yaml`.
- All YAML indentation uses spaces.
- Build paths are relative to the Compose file.
- Host and container ports are correct.
- Environment variables are defined.
- Persistent data uses named volumes.
- `docker compose config` succeeds.
- `docker compose up --build` starts the application.
- `docker compose down` stops and removes its containers.