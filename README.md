# Containers

# Docker Multi-Frontend Project with Traefik

This project demonstrates the evolution of a Docker-based web application, starting from a basic setup and progressing to a more sophisticated configuration using Traefik as a reverse proxy.

## Project Evolution

### Phase 1: Basic Docker Setup
Initially, we set up a simple Docker environment with:
- A single Nginx frontend container
- Direct port mapping to host machine
- Basic static HTML content

Note: Nginx is installed and run as a Docker container in this setup. It is not installed directly on the host machine but instead runs within a containerized environment.

### Phase 2: Multiple Frontends
Added a second frontend to serve different content:
- Two separate Nginx containers
- Each serving different static content
- Direct port mapping for each service
- Manual port management required

### Phase 3: Traefik Integration
Implemented Traefik as a reverse proxy to improve:
- Routing management
- Service discovery
- Port consolidation

### Phase 4: Path-Based Routing
Enhanced the setup with path-based routing using Traefik:
- `/tom` path for first frontend
- `/peeters` path for second frontend
- Automatic path stripping

## Architecture

![alt text](image.png)

## Components

### 1. Basic Docker setup

#### Windows Users:
Before running this project, ensure that Docker Desktop is installed and running. If you haven't installed it yet, download it from:  
[https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)

Once installed, start Docker Desktop before running the following commands.

#### Linux Users:

**[Docker Installation Guide for Linux](https://docs.docker.com/desktop/setup/install/linux/ubuntu/)**  

Make sure to choose the appropriate guide for your Linux distribution.

#### 2. Frontend Services (Without Traefik)  

Each Nginx container serves different content under distinct paths. Initially, they are accessible via different ports.

##### Tom's Frontend (`/tom`)  

This frontend serves content for the `/tom` path and is accessible directly on port **8081**.

```yaml
frontend:
  image: nginx:alpine
  container_name: frontend
  ports:
    - "8081:80"  # Exposes the service on port 8081
  volumes:
    - ./frontend/index.html:/usr/share/nginx/html/index.html:ro
```

##### Peeters' Frontend (`/peeters`)  

This frontend serves content for the `/peeters` path and is accessible directly on port **8082**.

```yaml
frontend2:
  image: nginx:alpine
  container_name: frontend2
  ports:
    - "8082:80"  # Exposes the service on port 8082
  volumes:
    - ./frontend2/index.html:/usr/share/nginx/html/index.html:ro
```

##### Complete Docker file without traefik

```yml
version: "3.8"

services:
  # First frontend (Tom's service)
  frontend:
    image: nginx:alpine
    container_name: frontend
    ports:
      - "8081:80"  # Exposes Tom's frontend on port 8081
    volumes:
      - ./frontend/index.html:/usr/share/nginx/html/index.html:ro

  # Second frontend (Peeters' service)
  frontend2:
    image: nginx:alpine
    container_name: frontend2
    ports:
      - "8082:80"  # Exposes Peeters' frontend on port 8082
    volumes:
      - ./frontend2/index.html:/usr/share/nginx/html/index.html:ro
```

1. Ensure you have the necessary HTML files inside:

```bash
.
├── docker-compose.yml
├── frontend/
│   └── index.html    # Tom's content
├── frontend2/
│   └── index.html    # Peeters' content
```

Run the following command to start the services:

```sh
docker-compose up -d
```

At this stage, users can access:  
- **Tom's Frontend** at `http://<YOUR-IP>:8081/`  
- **Peeters' Frontend** at `http://<YOUR-IP>:8082/`

### 3. Introducing Traefik (Reverse Proxy)

Purpose: Traefik acts as a reverse proxy that manages routing for different frontend services. It consolidates multiple backend services under a single entry point, improving scalability and routing efficiency.

**Configuration Explanation:**

- traefik: This is the container running Traefik as a reverse proxy.
- build.context: Builds the image using the Dockerfile located in ./traefik.
- command: Configures Traefik to use a custom configuration file (traefik.yml).
- ports: Exposes HTTP (port 80) and the Traefik dashboard (port 8080) for monitoring.
- volumes: Grants Traefik access to Docker socket, allowing it to discover services running in the Docker environment.

```yaml
traefik:
  build:
    context: ./traefik
    dockerfile: Dockerfile
  container_name: traefik
  command:
    - "--configFile=/etc/traefik/traefik.yml"
  ports:
    - "80:80"      # Web traffic
    - "8080:8080"  # Dashboard
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
```

#### 4. Frontend Services (with traefik)
Purpose: Each Nginx container serves different content under distinct paths. Traefik routes traffic based on path prefixes to the appropriate container.

##### Tom's Frontend (/tom)
This frontend serves content for the /tom path.

**Configuration Explanation:**

- labels: Defines the routing rules for Traefik.
  - traefik.http.routers.frontend.rule: Routes traffic to the /tom path.
  - traefik.http.routers.frontend.middlewares: Uses stripPrefix middleware to remove the /tom prefix before forwarding the request to the frontend service.


```yaml
frontend:
  image: nginx:alpine
  container_name: frontend
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.frontend.rule=Host(`<YOUR-IP>`) && PathPrefix(`/tom`)"
    - "traefik.http.routers.frontend.middlewares=frontend-strip"
    - "traefik.http.middlewares.frontend-strip.stripPrefix.prefixes=/tom"
```
**IMPORTANT:** Replace `<YOUR-IP>` with your actual PC’s local IP address.  
Run `ipconfig` (Windows) or `ip a` (Linux/macOS) to find your local IP.

##### Peeters' Frontend (/peeters)
This frontend serves content for the /peeters path.

**Configuration Explanation:**

Similar to the first frontend, but routes traffic to the /peeters path.

```yaml
frontend2:
  image: nginx:alpine
  container_name: frontend2
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.frontend2.rule=Host(`<YOUR-IP>`) && PathPrefix(`/peeters`)"
    - "traefik.http.routers.frontend2.middlewares=frontend2-strip"
    - "traefik.http.middlewares.frontend2-strip.stripPrefix.prefixes=/peeters"
```
**IMPORTANT:** Replace `<YOUR-IP>` with your actual PC’s local IP address.

#### Configuration Files

##### Traefik Configuration (traefik.yml)

Purpose: Configures entry points, service discovery, and providers.

**Explanation:**

- entryPoints: Defines the ports Traefik will listen to (80 for web traffic, 8080 for the Traefik dashboard).
- providers.docker: Enables service discovery using Docker's API.

```yaml
entryPoints:
  web:
    address: ":80"
  api:
    insecure: true
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
```

##### Traefik Dockerfile

Purpose: Custom Dockerfile for building the Traefik image.

**Explanation:**

- Copies the custom traefik.yml configuration into the container.
- Exposes ports 80 and 8080.
- Defines the entry point to run Traefik with the custom configuration file.

```dockerfile
FROM traefik:latest
COPY traefik.yml /etc/traefik/traefik.yml
EXPOSE 80 8080
ENTRYPOINT ["traefik"]
CMD ["--configFile=/etc/traefik/traefik.yml"]
```

## Project Structure
```
.
├── docker-compose.yml
├── traefik/
│   ├── Dockerfile
│   └── traefik.yml
├── frontend/
│   └── index.html    # Tom's content
└── frontend2/
    └── index.html    # Peeters' content
```

## Complete docker-compose.yml file

```yml
version: "3.8"

services:
  # Traefik reverse proxy
  traefik:
    build:
      context: ./traefik
      dockerfile: Dockerfile
    container_name: traefik
    command: "--configFile=/etc/traefik/traefik.yml"
    ports:
      - "80:80"       # HTTP traffic
      - "8080:8080"   # Traefik dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    labels:
      - "traefik.enable=true"  # Enable this service in Traefik

  # First frontend (accessible via /tom)
  frontend:
    image: nginx:alpine
    container_name: frontend
    labels:
      - "traefik.enable=true"  # Enable Traefik for this container
      - "traefik.http.routers.frontend.rule=Host(`<YOUR-IP>`) && PathPrefix(`/tom`)"  # Define a router that matches requests to this host with the /tom prefix
      - "traefik.http.routers.frontend.middlewares=frontendares.frontend-strip.stripPrefix.prefixes=/tom"  # Middleware removes "/tom" from the request path before passing it to Nginx
      - "traefik.http.service-strip"  # Attach the 'frontend-strip' middleware to modify requests
      - "traefik.http.middlews.frontend.loadbalancer.server.port=80"  # Define the backend service port (Nginx listens on port 80)
    volumes:
      - ./frontend/index.html:/usr/share/nginx/html/index.html:ro

  # Second frontend (accessible via /peeters)
  frontend2:
    image: nginx:alpine
    container_name: frontend2
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.frontend2.rule=Host(`<YOUR-IP>`) && PathPrefix(`/peeters`)"  # Define a router that matches requests to this host with the /peeters prefix
      - "traefik.http.routers.frontend2.middlewares=frontend2-strip"  # Attach the 'frontend2-strip' middleware
      - "traefik.http.middlewares.frontend2-strip.stripPrefix.prefixes=/peeters"  # Middleware removes "/peeters" from the request path before passing it to Nginx
      - "traefik.http.services.frontend2.loadbalancer.server.port=80"  # Define the backend service port (Nginx listens on port 80)
    volumes:
      - ./frontend2/index.html:/usr/share/nginx/html/index.html:ro
```

## Usage

### Accessing the Services
- Tom's frontend: `http://<YOUR-IP>/tom`
- Peeters' frontend: `http://<YOUR-IP>/peeters`
- Traefik dashboard: `http://<YOUR-IP>:8080`

### Important Docker Commands

| **Action**                                  | **Command**                          |
|--------------------------------------------|--------------------------------------|
| **Start containers**                        | `docker-compose up -d`               |
| **Stop containers**                         | `docker-compose down`                |
| **Restart containers**                      | `docker-compose restart`             |
| **Stop a specific container**               | `docker stop <container_name>`       |
| **Start a specific container**              | `docker start <container_name>`      |
| **View running containers**                 | `docker ps`                          |
| **View all containers (including stopped)** | `docker ps -a`                       |
| **View logs of a specific container**       | `docker logs <container_name>`       |
| **View logs of all containers in docker-compose** | `docker-compose logs`         |
| **Follow real-time logs**                   | `docker logs -f <container_name>`    |
| **Find image IDs**                   | `docker images`    |
| **Remove Images**                   | `docker rmi IMAGE_ID`    |

## Key Features

### Path-Based Routing
- Each frontend is accessible through its own path
- Automatic path stripping via Traefik middleware
- Single port (80) handling all web traffic

### Traefik Dashboard
- Monitor service health
- View routing configuration
- Access metrics and logs
- Available on port 8080

### Benefits Over Initial Setup
1. Centralized routing management
2. Single port for multiple services
3. Automatic service discovery
4. Built-in monitoring
5. Easy path-based routing
6. Simple configuration through labels

## Future Improvements
- Add HTTPS support
- Implement authentication
- Add health checks
- Configure rate limiting
- Set up load balancing for scaling
