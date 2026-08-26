# Docker Commands — Complete Practical Reference

You do not need to memorize every obscure Docker command. For a DevOps Engineer, focus on command categories and when to use each command.

## 1. Docker version and information

| Command | Purpose |
|---|---|
| docker --version | Shows installed Docker version |
| docker version | Shows client and server/engine versions |
| docker info | Shows Docker Engine/system information |
| docker help | Shows available Docker commands |
| docker <command> --help | Shows help for a specific command |

Examples:

```bash
docker --version
docker version
docker info
docker container --help
```

## 2. Docker images

Images are templates used to create containers.

### List images

```bash
docker images
docker image ls
```

### Build an image

```bash
docker build -t myapp:1.0 .
```

Flow:

```text
Dockerfile
	↓
docker build
	↓
Docker Image
```

### Build without cache

```bash
docker build --no-cache -t myapp:1.0 .
```

Useful when Docker is reusing an old cached layer.

### Remove image

```bash
docker rmi myapp:1.0
```

### Force remove image

```bash
docker rmi -f myapp:1.0
```

### Inspect image

```bash
docker image inspect myapp:1.0
```

Shows image configuration, layers, architecture, environment variables, entrypoint, and command.

### Image history

```bash
docker history myapp:1.0
```

Useful for finding why an image is large.

### Pull image

```bash
docker pull nginx:latest
```

### Push image

```bash
docker push username/myapp:1.0
```

### Tag image

```bash
docker tag myapp:1.0 username/myapp:1.0
```

Useful before pushing to Docker Hub, ECR, and similar registries.

## 3. Docker containers

These are the commands used most frequently.

### List running containers

```bash
docker ps
```

Example:

```text
CONTAINER ID   IMAGE       STATUS
abc123         myapp:1.0   Up 10 minutes
```

### List all containers

```bash
docker ps -a
```

Also shows exited, created, and restarting containers.

### Create a container

```bash
docker create nginx
```

Creates a container but does not start it.

```text
Image
 ↓
docker create
 ↓
Stopped container
```

### Start a container

```bash
docker start mycontainer
```

### Stop a container

```bash
docker stop mycontainer
```

Graceful shutdown.

### Kill a container

```bash
docker kill mycontainer
```

Immediate termination. Use carefully in production.

```text
docker stop  -> Graceful shutdown
docker kill  -> Immediate termination
```

### Restart container

```bash
docker restart mycontainer
```

Stops and starts the container again.

### Remove container

```bash
docker rm mycontainer
```

Removes a stopped container.

### Force remove container

```bash
docker rm -f mycontainer
```

### Run a container

```bash
docker run nginx
```

Creates and starts a container from an image.

Common detached mode:

```bash
docker run -d nginx
```

### Run container with a name

```bash
docker run -d --name my-nginx nginx
```

### Port mapping

```bash
docker run -d -p 8080:80 nginx
```

```text
Host              Container
8080       --->      80
```

localhost:8080 forwards to container port 80.

### Environment variables

```bash
docker run -d \
  -e ASPNETCORE_ENVIRONMENT=Production \
  my-api
```

Multiple variables:

```bash
docker run -d \
  -e DB_HOST=database \
  -e DB_PORT=5432 \
  my-api
```

### Pass environment file

```bash
docker run --env-file .env my-api
```

Useful for development and testing.

### Execute command inside running container

```bash
docker exec my-api ls
```

### Open shell inside container

```bash
docker exec -it my-api sh
docker exec -it my-api bash
```

Most important production debugging pattern:

```bash
docker exec -it my-api sh
ls
pwd
env
whoami
```

### View container logs

```bash
docker logs my-api
docker logs --tail 100 my-api
docker logs -f my-api
docker logs -t my-api
docker logs --tail 100 -f my-api
```

### Inspect container

```bash
docker inspect my-api
```

Investigate network, IP, mounts, environment, ports, image, restart policy, and health status.

### Container resource usage

```bash
docker stats
docker stats my-api
```

Shows CPU, memory, network, block I/O, and PIDs.

### Container processes

```bash
docker top my-api
```

Useful when CPU or memory is high or the app seems stuck.

### Container port mapping details

```bash
docker port my-api
```

Example:

```text
8080/tcp -> 0.0.0.0:8080
```

### Copy files

Container to host:

```bash
docker cp my-api:/app/log.txt .
```

Host to container:

```bash
docker cp config.json my-api:/app/
```

### Rename container

```bash
docker rename old-name new-name
```

### Pause and unpause

```bash
docker pause my-api
docker unpause my-api
```

## 4. Docker networks

```bash
docker network ls
docker network create app-network
docker network inspect app-network
docker network connect app-network my-api
docker network disconnect app-network my-api
docker network rm app-network
```

Why it matters:

```text
API
 │
 │ Docker Network
 ↓
PostgreSQL
```

Your API should use service/container name (for example postgres) instead of localhost.

## 5. Docker volumes

Volumes provide persistent storage.

```bash
docker volume ls
docker volume create app-data
docker volume inspect app-data
docker volume rm app-data
```

Run with volume:

```bash
docker run -d \
  -v app-data:/app/data \
  my-api
```

```text
Container
	│
	▼
Volume
	│
	▼
Persistent data
```

### Bind mounts

```bash
docker run -v /host/data:/app/data my-api
docker run --mount type=bind,source=/host/data,target=/app/data my-api
```

## 6. Docker system maintenance

```bash
docker system df
docker container prune
docker image prune
docker network prune
docker volume prune
docker system prune
docker system prune -a
```

Use prune commands carefully in production.

## 7. Docker Compose

Compose runs multiple containers as one app stack (for example API + PostgreSQL + Redis + RabbitMQ) defined in docker-compose.yml.

```bash
docker compose up
docker compose up -d
docker compose up --build
docker compose stop
docker compose start
docker compose restart
docker compose down
docker compose down -v
docker compose ps
docker compose logs
docker compose logs api
docker compose logs -f api
docker compose exec api sh
docker compose build
docker compose pull
```

## 8. Docker registry commands

Works with Docker Hub, Amazon ECR, GitHub Container Registry, Azure Container Registry, and Google Artifact Registry.

```bash
docker login
docker logout
docker pull nginx:latest
docker push username/my-api:1.0
```

## 9. Docker Buildx

Buildx is used for advanced and multi-platform image builds.

```bash
docker buildx ls
docker buildx build -t my-api:1.0 .
docker buildx build --platform linux/amd64 -t my-api:1.0 .
docker buildx build --platform linux/amd64,linux/arm64 -t my-api:1.0 --push .
```

Especially useful on ARM64 Mac when deploying to x86 servers.

## 10. Docker context

Contexts allow working with different Docker engines.

```bash
docker context ls
docker context show
docker context create my-server
docker context use my-server
```

## 11. Docker plugins

```bash
docker plugin ls
```

## 12. Docker events

Useful for advanced troubleshooting.

```bash
docker events
```

Shows real-time Docker events such as container start, stop, die, network connect, and volume mount.

## 13. Docker secrets

Docker Swarm supports:

```bash
docker secret ls
docker secret create
docker secret inspect
docker secret rm
```

In AWS-focused environments, you will more commonly use AWS Secrets Manager, SSM Parameter Store, ECS Secrets, or Kubernetes Secrets.

## 14. Docker Swarm

Built-in Docker orchestration commands:

```bash
docker swarm init
docker swarm join
docker swarm leave
docker node ls
docker service ls
docker service create
docker service ps
docker service logs
docker service scale
docker service update
```

For AWS DevOps roles, prioritize ECS or EKS over Swarm.

## Most important commands for you

You do not need to memorize everything immediately.

### Level 1: Must know

```bash
docker --version
docker info
docker images
docker pull
docker build
docker run
docker ps
docker ps -a
docker stop
docker start
docker restart
docker rm
docker rmi
docker logs
docker exec
docker inspect
```

### Level 2: Production troubleshooting

```bash
docker stats
docker top
docker port
docker network ls
docker network inspect
docker volume ls
docker volume inspect
docker system df
docker events
```

### Level 3: Docker Compose

```bash
docker compose up
docker compose up -d
docker compose down
docker compose ps
docker compose logs
docker compose exec
docker compose build
docker compose pull
docker compose restart
```

### Level 4: Advanced DevOps

```bash
docker buildx
docker context
docker swarm
docker secret
docker system prune
```

## Workflow to memorize

This is more important than memorizing 100 commands:

```text
						  Developer
							  │
							  ▼
						Dockerfile
							  │
							  │ docker build
							  ▼
					  Docker Image
							  │
					  docker tag
							  │
							  ▼
						Container
							  │
				  docker run -d
							  │
							  ▼
					  Application
							  │
			 ┌────────────┼────────────┐
			 ▼            ▼            ▼
		 Network       Volume       Logs
			 │            │            │
			 ▼            ▼            ▼
		Database      Persistent    Monitoring
							Data
```

Typical AWS DevOps pipeline:

```text
GitHub
	│
	▼
CI/CD
	│
	│ docker build
	▼
Docker Image
	│
	│ docker push
	▼
Amazon ECR
	│
	▼
ECS / EKS
	│
	▼
Docker Container
	│
	├── ALB
	├── RDS
	├── CloudWatch
	└── Secrets Manager
```

If you understand this flow plus Level 1 and Level 2 commands, you already have a strong practical Docker foundation for a DevOps role.
