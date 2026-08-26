# What is Docker?

Docker is a platform that packages an application together with everything it needs to run into a standardized unit called a container.

In simple terms:

Docker allows you to build an application once and run it consistently across different environments.

For example, a .NET API may need:

- .NET 9 runtime
- Specific libraries and packages
- Environment configuration
- OS-level dependencies
- Application code

Without Docker:

```text
Developer Laptop
   -> .NET 9 installed
   -> Correct dependencies
   -> Correct configuration
   -> Application works

Production Server
   -> Different .NET version
   -> Missing dependency
   -> Configuration difference
   -> Application fails
```

Docker solves much of this by packaging the application environment.

## Docker in simple terms

Think of Docker as a shipping container.

A physical shipping container can carry different goods, and the same container can be moved by different transport systems.

Similarly, a Docker container packages:

- Application
- Runtime
- Dependencies
- Configuration

And can run on:

- Developer laptop
- AWS EC2
- ECS
- EKS/Kubernetes
- On-premises servers

## Docker architecture

Basic flow:

```text
Dockerfile
   |
   | docker build
   v
Docker Image
   |
   | docker run
   v
Container
   |
   v
Application
```

Three key terms:

### 1) Dockerfile

A Dockerfile is a set of instructions that tells Docker how to build your application image.

Example:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0

WORKDIR /app

COPY ./publish .

EXPOSE 8080

ENTRYPOINT ["dotnet", "MyApi.dll"]
```

This means:

```text
Use .NET 9 runtime
Create /app
Copy application
Expose port 8080
Run MyApi.dll
```

### 2) Docker Image

An image is a packaged, read-only template from which containers are created.

Example image name:

```text
my-api:1.0
```

Build command:

```bash
docker build -t my-api:1.0 .
```

### 3) Docker Container

A container is a running instance of an image.

Run command:

```bash
docker run -d -p 8080:8080 my-api:1.0
```

Concept:

```text
Image
  |
  | docker run
  v
Container
  |
  +-- MyApi.dll running
```

You can create multiple containers from one image:

```text
	   my-api:1.0
	   /        \
	  v          v
Container 1   Container 2
	 |            |
	 v            v
   API #1       API #2
```

This is one reason Docker is useful for scaling applications.

## Docker vs Virtual Machine

This is a common DevOps interview topic.

### Virtual Machine

```text
Physical Server
	  |
	  v
  Hypervisor
   /      \
 VM        VM
 |         |
Guest OS  Guest OS
 |         |
App       App
```

Each VM has its own operating system.

### Docker

```text
Physical Server
	  |
	  v
 Docker Engine
   /      \
Container  Container
   |          |
  App        App
```

Containers share the host OS kernel rather than each carrying a full guest OS.

Therefore containers are generally:

- Faster to start
- More lightweight
- Easier to package
- Easier to scale

## Docker in an AWS DevOps architecture

Example flow for a .NET app:

```text
.NET Web API
	|
	v
Dockerfile
	|
	v
Docker Image
	|
	v
Amazon ECR
	|
	v
Amazon ECS
	|
	v
Running Containers
	|
	v
RDS MySQL
```

Important distinction:

- ECR stores Docker images.
- ECS runs Docker containers.

```text
Docker Image -> ECR -> ECS -> Container
```

## Docker's main purpose

The biggest reason companies use Docker is consistency.

Goal:

```text
Development
	|
	v
Same Docker Image
	|
	v
Testing
	|
	v
Same Docker Image
	|
	v
Production
```

Instead of:

```text
Developer environment != QA environment != Production environment
```

You want the same application image everywhere. Only configuration, secrets, resources, and infrastructure should vary.

## Interview-ready definition

If asked "What is Docker?", you can answer:

Docker is a containerization platform that packages an application along with its runtime and dependencies into a portable Docker image. That image can then be run as a container consistently across development, testing, and production environments. Docker helps solve environment inconsistencies and makes application deployment, scaling, and CI/CD easier.

Remember the foundation:

Dockerfile -> Image -> Container

After this, the next concepts to learn are Dockerfile best practices, image layers, containers, volumes, networks, Docker Compose, and registries such as Amazon ECR.
