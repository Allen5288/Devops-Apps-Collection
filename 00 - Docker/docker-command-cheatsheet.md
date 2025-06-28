# Docker Command Cheatsheet

This document contains useful Docker commands and URLs for quick reference, organized by topic. Each command is explained in one line for clarity.

---

## 01 Prepare Your Environment

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)  
  Install Docker Desktop for container management.
- [VS Code](https://code.visualstudio.com/download)  
  Install Visual Studio Code for code editing.

---

## 02 Docker Build

- [docker build docs](https://docs.docker.com/engine/reference/commandline/build/)  
  Official documentation for `docker build`.

```bash
docker build [OPTIONS] PATH | URL | -   # Build an image from a Dockerfile
docker build .   # Build using Dockerfile in current directory
docker build -f [dockerfile] .   # Specify a Dockerfile
docker build --force-rm=true .   # Always remove intermediate containers
docker build --rm=true .   # Remove intermediate containers after a successful build
docker build --no-cache .   # Build without cache
docker build --help   # Show build options
```

---

## 03 Docker Search

```bash
docker search python   # Search Docker Hub for images related to python
docker search --filter is-official=true python   # Only show official images
docker search --filter stars=100 python   # Images with at least 100 stars
docker search --filter is-automated=true python   # Only show automated builds
docker search --filter is-official=true --filter stars=10 --filter is-automated=false python   # Combine filters
docker search --limit 4 python   # Limit results to 4
docker search --no-trunc python   # Show full output without truncation
docker search --format "{{.Name}}: {{.Description}}" --no-trunc python   # Custom output format
docker search --format "table {{.Name}}\t{{.IsAutomated}}\t{{.IsOfficial}}\t{{.Description}}\t{{.StarCount}}" python   # Table format
docker search --format "table {{.Name}}\t{{.IsOfficial}}\t{{.Description}}\t{{.StarCount}}" --filter is-official=true --filter stars=10 --no-trunc python   # Table with filters
docker search --help   # Show help for search
```

- [python official image](https://hub.docker.com/_/python)  
  Python image on Docker Hub.

```bash
docker image pull python   # Pull the latest python image
docker image pull python:3.12-rc-bookworm   # Pull a specific python image
docker image pull --help   # Show help for pull
```

---

## 04 Docker Image Listing

- [docker image ls docs](https://docs.docker.com/engine/reference/commandline/image_ls/)  
  Official documentation for `docker image ls`.

```bash
docker image ls [OPTIONS] [REPOSITORY[:TAG]]   # List images
docker image ls --all   # List all images
docker image ls --no-trunc   # Do not truncate output
docker image ls --quiet   # Only show numeric IDs
docker image ls --digests   # Show digests
docker image ls --filter reference=python*   # Filter by reference
docker image ls --filter before=python   # Images created before python
docker image ls --filter since=python   # Images created since python
docker image ls --filter dangling=true   # Only dangling images
docker image ls --filter label=python*   # Filter by label
docker image ls --format "table {{.ID}}\t{{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedSince}}\t{{.CreatedAt}}\t{{.Digest}}"   # Custom table format
docker image ls --help   # Show help for image ls
```

---

## 05 Docker Image Management

```bash
docker build -t SOURCE_IMAGE[:TAG] .   # Build and tag an image
docker build -t big-star-collectibles .   # Build and tag the image as big-star-collectibles
docker build --no-cache -t big-star-collectibles:v1 -t big-star-collectibles .   # Build without cache and tag
docker image ls   # List images
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]   # Tag an image
docker tag big-star-collectibles:v1 big-star-collectibles:v1-python   # Tag an image with a new tag
docker build --no-cache -t big-star-collectibles:v2 .   # Build a new version without cache
docker tag SOURCE_IMAGE_ID TARGET_IMAGE[:TAG]   # Tag an image by ID
LABEL <key>=<value> <key>=<value> <key>=<value> ...   # Add labels to an image
docker image ls --filter label="com.example.vendor"="Big Star Collectibles"   # Filter images by label
```

---

## 06 Docker Hub Interaction

- [Docker Hub](https://hub.docker.com/)  
  Docker's official cloud-based registry service.

```bash
docker login   # Log in to Docker Hub
docker tag big-star-collectibles:v2 sbenhoff/big-star-collectibles-repo:big-star-collectibles   # Tag image for Docker Hub
docker push sbenhoff/big-star-collectibles-repo:big-star-collectibles   # Push image to Docker Hub
docker pull sbenhoff/big-star-collectibles-repo:big-star-collectibles   # Pull image from Docker Hub
```

---

## 07 Docker Image Inspection

```bash
docker image ls   # List images
docker image inspect IMAGE_ID   # Inspect an image by ID
docker image inspect --format='{{json .Config.Labels}}' big-star-collectibles:v2   # Show labels of an image
```

---

## 08 Docker Image Removal

```bash
docker image ls   # List images
docker rmi [OPTIONS] IMAGE [IMAGE...]   # Remove one or more images
docker rmi -f big-star-collectibles:v2   # Force remove an image
docker rmi -f IMAGE_ID   # Force remove an image by ID
docker images --digests   # Show image digests
docker rmi -f sbenhoff/big-star-collectibles@[DIGEST]   # Remove image by digest
```

---

## 09 Docker Container Management

- [docker run docs](https://docs.docker.com/engine/reference/commandline/run/)  
  Official documentation for `docker run`.

```bash
docker start [OPTIONS] CONTAINER [CONTAINER...]   # Start a stopped container
docker start big-star-collectibles:v2   # Start a specific container
docker run [OPTIONS] IMAGE[:TAG] [COMMAND] [ARG...]   # Run a command in a new container
docker run -it -d -p 5000:5000 -v ${PWD}:/app/code big-star-collectibles   # Run a container in detached mode with port and volume mapping
http://localhost:5000   # Access the application in a web browser
docker stop [OPTIONS] CONTAINER [CONTAINER...]   # Stop a running container
docker stop big-star-collectibles:v2   # Stop a specific container
docker kill [OPTIONS] CONTAINER [CONTAINER...]   # Kill a running container
docker kill big-star-collectibles:v2   # Forcefully stop a specific container
```

---

## 10 Docker Container Listing

- [docker ps docs](https://docs.docker.com/engine/reference/commandline/ps/)  
  Official documentation for `docker ps`.

```bash
docker ps [OPTIONS]   # List containers
docker ps   # List running containers
docker ps -a   # List all containers
docker ps -n 1   # Show the last created container
docker ps -q   # Only show container IDs
docker ps -s   # Show container sizes
docker ps -l   # Show the latest created container (includes stopped ones)
docker ps --no-trunc   # Show full output without truncation
docker ps --filter label="com.example.vendor"="Big Star Collectibles"   # Filter containers by label
docker ps -a --filter 'exited=0'   # Show all exited containers
docker ps --filter ancestor=big-star-collectibles   # Show containers based on an image
docker ps --filter label="com.example.vendor"="Big Star Collectibles" --format json   # Filter and format output as JSON
docker ps --filter label="com.example.vendor"="Big Star Collectibles" --format "table {{.ID}}\t{{.Names}}"   # Filter and format output as table
```

- COMMON DOCKER CONTAINER EXIT CODES *
Exit Code 0     Purposely stopped
Exit Code 1     Application error
Exit Code 125 Container failed to run error
Exit Code 126 Command invoke error
Exit Code 127 File or directory not found
Exit Code 128 Invalid argument used on exit
Exit Code 134 Abnormal termination (SIGABRT)
Exit Code 137 Immediate termination (SIGKILL)
Exit Code 139 Segmentation fault (SIGSEGV)
Exit Code 143 Graceful termination (SIGTERM)
Exit Code 255 Exit Status Out Of Range

---

## 11 Docker Container Inspection

- [docker inspect docs](https://docs.docker.com/engine/reference/commandline/inspect/)  
  Official documentation for `docker inspect`.

```bash
docker inspect [OPTIONS] NAME|ID [NAME|ID...]   # Return low-level information about a container
docker ps   # List containers
docker inspect [ID]   # Inspect a container by ID
docker inspect --format='{{.Config.Image}}' [NAME|ID]   # Show the image of a container
docker inspect --format='{{.Id}}' [NAME|ID]   # Show the ID of a container
docker inspect --format='{{.LogPath}}' [NAME|ID]   # Show the log path of a container
docker inspect --format='{{json .Config}}' [NAME|ID]   # Show the configuration of a container in JSON format
```

---

## 12 Docker Container Logs

- [docker logs docs](https://docs.docker.com/engine/reference/commandline/logs/)  
  Official documentation for `docker logs`.

```bash
docker logs [OPTIONS] CONTAINER   # Fetch the logs of a container
docker logs --tail 1000 -f [NAME|ID]   # Tail the logs
docker logs --tail 1000 -f --details [NAME|ID]   # Tail the logs with details
# UTC Format
docker logs --tail 1000 -f --since 2023-12-01T01:00:00Z [NAME|ID]   # Logs since a specific UTC time
docker logs --tail 1000 -f --until 2023-12-01T01:00:00Z [NAME|ID]   # Logs until a specific UTC time
# Relative Format
docker logs --tail 1000 -f --since 60 [NAME|ID]   # Logs since 60 seconds ago
docker logs --tail 1000 -f --until 60 [NAME|ID]   # Logs until 60 seconds ago
docker logs --tail 1000 -f --timestamps [NAME|ID]   # Show timestamps in logs
```

---

## 13 Docker Volume Management

```bash
docker volume create big-star-collectibles-vol   # Create a new volume
docker volume ls   # List volumes
docker volume inspect big-star-collectibles-vol   # Inspect a volume
docker run -it -d -p 5000:5000 -v big-star-collectibles-vol:/app/data sbenhoff/big-star-collectibles-repo:big-star-collectibles   # Run a container with a volume
docker volume rm big-star-collectibles-vol   # Remove a volume
docker run -it -d -p 5000:5000 -v "$(PWD)/test"/target:/app/test sbenhoff/big-star-collectibles-repo:big-star-collectibles   # Run a container with a bind mount
docker run -it -d -p 5000:5000 --mount type=bind,source="$(PWD)/test",target=/app/test sbenhoff/big-star-collectibles-repo:big-star-collectibles   # Run a container with a bind mount (alternative syntax)
```

---

## 14 Docker System Prune

```bash
docker image prune   # Remove unused images
docker image prune -a -f   # Remove all unused images without confirmation
docker image prune -a -f --filter label="com.example.vendor"="Big Star Collectibles"   # Prune images with a specific label
docker container prune   # Remove stopped containers
docker volume prune -f --filter label="com.example.vendor"="Big Star Collectibles"   # Prune volumes with a specific label
docker system prune --volumes -f   # Prune unused data, including volumes
```
