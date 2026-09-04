# What is a Dockerfile?

A Dockerfile is a text file with step-by-step instructions that Docker uses to build an image.

Think of it like a recipe:

- The recipe is the Dockerfile.
- The prepared dish is the Docker image.
- The running dish on a plate is the container.

Flow:

```text
Dockerfile -> docker build -> Docker Image -> docker run -> Container
```

## How to write a Dockerfile

Create a file named Dockerfile (no extension) in your project folder.

A simple structure looks like this:

```dockerfile
FROM ubuntu:22.04

WORKDIR /app

COPY . .

RUN apt-get update && apt-get install -y curl

EXPOSE 8080

CMD ["sh", "-c", "echo App started && tail -f /dev/null"]
```

Then build and run:

```bash
docker build -t myapp:1.0 .
docker run -d -p 8080:8080 --name myapp-container myapp:1.0
```

## Dockerfile components explained

### FROM

Selects the base image used to start building your image.

Example:

```dockerfile
FROM ubuntu:22.04
```

### WORKDIR

Sets the working directory inside the image for the next instructions.

Example:

```dockerfile
WORKDIR /app
```

### COPY

Copies files from your machine into the image.

Example:

```dockerfile
COPY . .
```

### ADD

Similar to COPY, but can also fetch URLs and auto-extract local archives.

Example:

```dockerfile
ADD archive.tar.gz /app/
```

Use COPY by default; use ADD only when you need its extra behavior.

### RUN

Runs commands during image build time.

Example:

```dockerfile
RUN apt-get update && apt-get install -y curl
```

### EXPOSE

Documents the port the containerized app listens on.

Example:

```dockerfile
EXPOSE 8080
```

Important: EXPOSE does not publish the port to your laptop/server by itself.
You still need:

```bash
docker run -p 8080:8080 myapp:1.0
```

### ENV

Sets environment variables in the image/container.

Example:

```dockerfile
ENV APP_ENV=production
```

### ARG

Defines build-time variables (available only while building unless copied into ENV).

Example:

```dockerfile
ARG APP_VERSION=1.0.0
```

### CMD

Sets the default command when a container starts.

Example:

```dockerfile
CMD ["sh", "-c", "echo Hello"]
```

### ENTRYPOINT

Defines the main executable of the container.

Example:

```dockerfile
ENTRYPOINT ["/app/start.sh"]
```

### USER

Sets which user runs commands/container process.

Example:

```dockerfile
USER 1001
```

### HEALTHCHECK

Defines how Docker checks if the container is healthy.

Example:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:8080/health || exit 1
```

## RUN vs CMD vs ENTRYPOINT

- RUN: executes while building the image.
- CMD: default command at container start.
- ENTRYPOINT: fixed main process at container start.

Lifecycle:

```text
docker build -> RUN executes -> image created -> docker run -> ENTRYPOINT/CMD executes
```

## Best practices

- Use small base images when possible.
- Put frequently cached steps early.
- Combine related RUN commands to reduce layers.
- Avoid putting secrets directly in Dockerfile.
- Use .dockerignore to keep build context small.

Example .dockerignore:

```text
.git/
.vscode/
node_modules/
bin/
obj/
*.log
```

## Quick checklist

- Dockerfile exists in project root.
- Base image is correct.
- App files are copied.
- Build dependencies are installed.
- Correct port is exposed.
- Startup command is defined.
- Build and run commands are tested.

If you understand these components, you can read and write most Dockerfiles confidently.
