# Docker Networking

Docker networking is the mechanism that allows:

- Container -> Container communication
- Container -> Host communication
- Container -> Internet communication
- Containers -> External services such as RDS, Redis, and APIs

A useful mental model:

```text
										Docker Host
┌───────────────────────────────────────────────────────┐
│                                                       │
│                  Docker Engine                        │
│                       │                               │
│              ┌────────┴────────┐                      │
│              │ Docker Network  │                      │
│              │   172.x.x.x     │                      │
│              └───────┬─────────┘                      │
│                      │                                │
│          ┌───────────┼───────────┐                    │
│          │           │           │                    │
│          ▼           ▼           ▼                    │
│       ┌─────┐     ┌──────┐    ┌──────┐               │
│       │ API │     │ DB   │    │Redis │               │
│       │     │     │      │    │      │               │
│       └─────┘     └──────┘    └──────┘               │
│                                                       │
└───────────────────────────────────────────────────────┘
```

The containers can communicate through the Docker network.

## 1. Why Docker networking matters

Suppose you have:

```text
										Internet
											 │
											 ▼
										ALB
											 │
											 ▼
								 .NET API
									Container
											 │
								 Docker Network
											 │
							┌────────┴────────┐
							▼                 ▼
					PostgreSQL          Redis
					 Container          Container
```

Without networking:

```text
API   X   PostgreSQL
API   X   Redis
```

With networking:

```text
API ─────────→ PostgreSQL
 │
 └───────────→ Redis
```

## 2. Docker networking architecture

At a high level:

```text
												Docker Host
┌───────────────────────────────────────────────────┐
│                                                   │
│                    Docker Engine                  │
│                         │                         │
│                  Network Driver                   │
│                         │                         │
│              ┌──────────┴──────────┐              │
│              │                     │              │
│          docker0                 Custom           │
│          bridge                  network          │
│              │                     │              │
│          ┌───┴───┐             ┌───┴────┐         │
│          │       │             │        │         │
│         API     DB            API      Redis      │
│                                                   │
└───────────────────────────────────────────────────┘
```

Most important network drivers:

1. `bridge`
2. `host`
3. `none`
4. `overlay`
5. `macvlan`

For normal Docker deployments, `bridge` is the first one to master.

## 3. Bridge network

Default networking mode is usually bridge.

```bash
docker network ls
```

Example:

```text
NETWORK ID     NAME      DRIVER
xxxx           bridge    bridge
xxxx           host      host
xxxx           none      null
```

Architecture:

```text
								 Docker Host
┌────────────────────────────────────┐
│                                    │
│          bridge network            │
│          172.17.0.0/16             │
│                                    │
│       ┌────────┐   ┌────────┐      │
│       │ API    │   │ Redis  │      │
│       │172.17. │   │172.17. │      │
│       │ 0.2    │   │ 0.3    │      │
│       └────────┘   └────────┘      │
│                                    │
└────────────────────────────────────┘
```

## 4. Create a custom bridge network

In production-like setups, prefer a custom network over default bridge.

```bash
docker network create app-network
docker network ls
docker network inspect app-network
```

## 5. Run containers on the same network

Run PostgreSQL:

```bash
docker run -d \
	--name postgres \
	--network app-network \
	postgres
```

Run API:

```bash
docker run -d \
	--name api \
	--network app-network \
	my-api:1.0
```

```text
						 app-network
									│
					┌───────┴────────┐
					│                │
				API             PostgreSQL
			 Container         Container
```

The API can connect to database by container name:

```text
Server=postgres
Port=5432
```

## 6. Container name becomes hostname

If container name is `postgres`, API should connect to `postgres:5432`, not `localhost:5432`.

Inside API container, `localhost` means the API container itself.

```text
API container
localhost:5432
			↓
API itself ❌
```

Correct:

```text
API container
		 │
		 │ app-network
		 ▼
postgres:5432
		 │
		 ▼
PostgreSQL container ✅
```

## 7. Docker DNS

Custom Docker networks provide internal DNS.

If `api`, `postgres`, and `redis` are on `app-network`, Docker resolves:

```text
postgres -> PostgreSQL container IP
redis    -> Redis container IP
```

So use names, not fixed IP addresses.

## 8. Test Docker DNS and connectivity

Enter API container:

```bash
docker exec -it api sh
```

Resolve host:

```bash
getent hosts postgres
```

Example:

```text
172.20.0.2 postgres
```

Test port:

```bash
nc -zv postgres 5432
```

Success example:

```text
Connection to postgres 5432 port [tcp/*] succeeded
```

## 9. Port mapping

If API listens on 8080 inside container:

```bash
docker run -d \
	--name api \
	-p 8080:8080 \
	my-api
```

```text
						 Host
					localhost:8080
								│
								│ -p 8080:8080
								▼
				┌────────────────┐
				│ API Container  │
				│                │
				│ Port 8080      │
				└────────────────┘
```

Syntax:

```text
-p HOST_PORT:CONTAINER_PORT
```

Example `-p 80:8080` means host port 80 forwards to container port 8080.

## 10. EXPOSE vs -p

Dockerfile:

```dockerfile
EXPOSE 8080
```

`EXPOSE` only documents metadata. It does not publish a host port.

Actual publishing:

```bash
docker run -p 8080:8080 my-api
```

```text
EXPOSE -> Documentation/metadata
-p     -> Actual host-to-container mapping
```

## 11. Container-to-container does not require -p

If both containers are on same Docker network, API can reach `postgres:5432` without publishing database port to host.

You need `-p` only when something outside that Docker network must access the service.

## 12. Core Docker network commands

```bash
docker network ls
docker network create app-network
docker network inspect app-network
docker network connect app-network api
docker network disconnect app-network api
docker network rm app-network
docker network prune
```

Use `prune` carefully in production.

## 13. Inspecting a network

This command is essential:

```bash
docker network inspect app-network
```

It shows network name, driver, subnet, gateway, and connected containers.

## 14. Network drivers overview

### Bridge

Most common for standalone Docker and Compose.

Use for APIs, databases, Redis, and local development.

### Host

```bash
docker run --network host nginx
```

Container shares host network namespace. Less isolation.

### None

```bash
docker run --network none my-api
```

No external network connectivity.

### Overlay

Used across multiple Docker hosts (especially with Swarm).

```text
Docker Host 1                 Docker Host 2

	 API  ─────── Overlay ─────── DB
		 │                             │
		 └──────── Docker Network ─────┘
```

For AWS-focused learning, prioritize ECS and EKS networking over Swarm.

## 15. Docker Compose networking

Example:

```yaml
services:
	api:
		image: my-api
		ports:
			- "8080:8080"

	postgres:
		image: postgres:17

	redis:
		image: redis:7
```

Compose automatically creates a default app network.

```text
								 app_default
										 │
					┌──────────┼──────────┐
					│          │          │
				 API      PostgreSQL   Redis
```

API can connect to `postgres:5432` and `redis:6379`.

## 16. Local architecture example

```text
										 Your Laptop
┌────────────────────────────────────────────┐
│                                            │
│              Docker Engine                 │
│                                            │
│              app-network                   │
│                   │                        │
│       ┌───────────┼───────────┐            │
│       │           │           │            │
│       ▼           ▼           ▼            │
│     .NET API   PostgreSQL    Redis         │
│     :8080       :5432        :6379         │
│       │                                    │
└───────┼────────────────────────────────────┘
				│
				▼
	 localhost:8080
```

## 17. AWS production architecture

```text
												Internet
													 │
													 ▼
										┌─────────────┐
										│     ALB     │
										└──────┬──────┘
													 │
													 ▼
										┌─────────────┐
										│ ECS Service │
										│             │
										│ .NET API    │
										│ Container   │
										└──────┬──────┘
													 │
									VPC Networking
													 │
								┌──────────┴──────────┐
								▼                     ▼
						 RDS MySQL              Redis
```

Docker networking and AWS networking interact, but they are not the same thing.

## 18. Docker networking vs AWS networking

Docker networking controls container-to-container communication.

```text
Container
	 ↓
Docker Network
	 ↓
Container
```

AWS VPC networking controls communication between AWS resources.

```text
ECS
 ↓
VPC
 ↓
Subnet
 ↓
Security Group
 ↓
RDS
```

In production, both layers often apply.

## 19. Production issue: API cannot connect to database

Common error:

```text
Connection refused
```

Debug sequence:

```bash
docker ps
docker network inspect app-network
docker exec -it api sh
getent hosts postgres
nc -zv postgres 5432
docker logs postgres
```

Possible root causes:

- Wrong hostname
- Wrong port
- DNS failure
- Different Docker networks
- Database not running
- Database not ready

## 20. Production issue: using localhost

If API config has:

```text
DB_HOST=localhost
```

inside API container, it points to itself, not PostgreSQL.

Fix:

```text
DB_HOST=postgres
```

## 21. Production issue: containers on different networks

If API and PostgreSQL are on different networks, they cannot communicate directly.

Check:

```bash
docker network inspect network-a
docker network inspect network-b
```

Fix example:

```bash
docker network connect network-a postgres
```

## 22. Production issue: port mapping confusion

If app listens on container port 8080 but run command is:

```bash
docker run -p 80:8080 my-api
```

you must access host on `localhost:80`, not `localhost:8080`.

## 23. Production issue: app listens only on localhost

For containerized .NET apps, bind to `0.0.0.0:8080` rather than `127.0.0.1:8080`.

ASP.NET Core example:

```text
ASPNETCORE_URLS=http://+:8080
```

## 24. Production issue: DNS failure

Error:

```text
Could not resolve host
```

Debug:

```bash
docker exec -it api sh
cat /etc/resolv.conf
getent hosts postgres
```

If name resolution fails, investigate Docker DNS, network membership, and service naming.

## 25. Production issue: can reach DB but not Internet

Pattern:

```text
API -> PostgreSQL ✅
API -> External API ❌
```

May be host or cloud routing/firewall/NAT issue, not container-to-container networking.

In AWS private subnets, validate route table, NAT gateway, and internet gateway path.

## 26. Production issue: ECS networking path

If service is unreachable from internet, check:

- ALB listener
- Target group
- ECS task and container port mapping
- Application port binding
- Security group
- Subnet
- Route table
- Network ACL
- Health checks

A healthy container can still be unreachable due to AWS networking.

## 27. Recommended production debugging flow

```text
							Network Problem
										│
										▼
						 Is container running?
										│
								docker ps
										│
										▼
					Are both containers on same network?
										│
					docker network inspect
										│
										▼
							 DNS working?
										│
					 getent hosts <service>
										│
										▼
							 Port reachable?
										│
					 nc -zv <host> <port>
										│
										▼
					Application listening?
										│
						 ss -lnt / netstat
										│
										▼
						Check application logs
										│
										▼
		 Check AWS and infrastructure networking
```

## 28. Commands to memorize

Network commands:

```bash
docker network ls
docker network create app-network
docker network inspect app-network
docker network connect app-network api
docker network disconnect app-network api
docker network rm app-network
docker network prune
```

Troubleshooting commands:

```bash
docker exec -it api sh
getent hosts postgres
nc -zv postgres 5432
docker port api
docker inspect api
docker logs api
```

## 29. Most important mental model

Debug in five layers:

```text
┌──────────────────────────┐
│ 1. Application           │
│ Is app listening?        │
└────────────┬─────────────┘
						 ↓
┌──────────────────────────┐
│ 2. Container             │
│ Is container running?    │
└────────────┬─────────────┘
						 ↓
┌──────────────────────────┐
│ 3. Docker Network        │
│ Can containers talk?     │
└────────────┬─────────────┘
						 ↓
┌──────────────────────────┐
│ 4. Host / AWS Network    │
│ VPC, SG, routes, NAT     │
└────────────┬─────────────┘
						 ↓
┌──────────────────────────┐
│ 5. External Dependency   │
│ RDS, API, Internet       │
└──────────────────────────┘
```

For AWS DevOps learning, keep this distinction clear:

- Docker networking: Container -> Container
- AWS networking: ECS/EC2 -> VPC -> Subnet -> Security Group -> RDS/Internet
- Production troubleshooting order: Application -> Container -> Docker Network -> VPC -> AWS Service

This becomes even more important when moving from Docker Compose to ECS/ECR and then Kubernetes/EKS.

## 30. Practical example: .NET API + PostgreSQL with Docker Compose

This example keeps networking practical: Docker Compose creates a network so the API can reach PostgreSQL by service name.

### Project structure

```text
MyApp/
├── MyApi/
│   ├── MyApi.csproj
│   ├── Program.cs
│   └── appsettings.json
├── Dockerfile
├── docker-compose.yml
└── .dockerignore
```

Architecture:

```text
										Docker Host
┌─────────────────────────────────────────────┐
│                                             │
│              app-network                    │
│                                             │
│     ┌─────────────────┐                     │
│     │   .NET API      │                     │
│     │   Container     │                     │
│     │   :8080         │                     │
│     └────────┬────────┘                     │
│              │                              │
│              │ postgres:5432                │
│              ▼                              │
│     ┌─────────────────┐                     │
│     │   PostgreSQL    │                     │
│     │   Container     │                     │
│     │   :5432         │                     │
│     └─────────────────┘                     │
│                                             │
└─────────────────────────────────────────────┘
						 │
						 │ :8080
						 ▼
				Your Browser
```

Key point: API and PostgreSQL communicate over the Docker network.

### Dockerfile

The Dockerfile builds the API image only. PostgreSQL is a separate container.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build

WORKDIR /src

COPY ["MyApi/MyApi.csproj", "MyApi/"]

RUN dotnet restore "MyApi/MyApi.csproj"

COPY . .

WORKDIR "/src/MyApi"

RUN dotnet publish "MyApi.csproj" \
		-c Release \
		-o /app/publish \
		/p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS final

WORKDIR /app

COPY --from=build /app/publish .

EXPOSE 8080

ENTRYPOINT ["dotnet", "MyApi.dll"]
```

### docker-compose.yml

```yaml
services:
	api:
		build:
			context: .
			dockerfile: Dockerfile
		container_name: my-api
		ports:
			- "8080:8080"
		environment:
			ConnectionStrings__DefaultConnection: "Host=postgres;Port=5432;Database=mydb;Username=postgres;Password=password"
		depends_on:
			- postgres
		networks:
			- app-network

	postgres:
		image: postgres:17
		container_name: postgres
		environment:
			POSTGRES_DB: mydb
			POSTGRES_USER: postgres
			POSTGRES_PASSWORD: password
		volumes:
			- postgres-data:/var/lib/postgresql/data
		networks:
			- app-network

networks:
	app-network:

volumes:
	postgres-data:
```

### Why Host=postgres matters

Use `Host=postgres` because Docker DNS resolves service name `postgres` to the PostgreSQL container.

Do not use `Host=localhost` from inside the API container, because `localhost` points to the API container itself.

### Port behavior

- API exposes `8080` to host via `ports`.
- PostgreSQL has no host `ports` entry, which is fine when only API needs DB access.
- API still reaches DB internally using `postgres:5432`.

### Start and verify

Start:

```bash
docker compose up -d
```

Compose flow:

1. Build API image
2. Pull PostgreSQL image
3. Create network
4. Create PostgreSQL container
5. Create API container
6. Attach both to same network

Check containers:

```bash
docker compose ps
docker ps
```

Check network:

```bash
docker network ls
docker network inspect app-network
```

Test API:

```bash
curl http://localhost:8080
```

Request flow:

```text
Browser
	 │
	 │ localhost:8080
	 ▼
API Container
	 │
	 │ postgres:5432
	 ▼
PostgreSQL Container
```

### Compose vs manual commands

Without Compose, you manually create network, start DB container, build API image, and start API container with networking and port flags.

With Compose, all of that lives in one file and starts with:

```bash
docker compose up -d
```

### Responsibilities to remember

```text
Dockerfile      -> How to build API image
Docker Compose  -> How to run full multi-container app
Docker Network  -> How containers communicate
Docker Volume   -> How database data persists
Port mapping    -> How host traffic reaches a container
```

Production pattern:

API container -> Docker network -> database container

Only the API is externally exposed. In AWS this maps conceptually to:

ALB -> ECS task/container -> RDS
