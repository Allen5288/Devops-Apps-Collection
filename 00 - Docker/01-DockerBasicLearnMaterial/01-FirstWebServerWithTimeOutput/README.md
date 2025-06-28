# Project: First Web Server With Time Output

This project demonstrates how to build a custom web server Docker image using a Dockerfile and Bash script, run it as a container, and manage its lifecycle. You will also learn how to tag and push your image to Docker Hub, and how to use Docker volume mounting to serve static content with nginx.

## Key Concepts

- Building Docker images with a custom Dockerfile
- Running containers in detached mode
- Viewing container logs and managing containers
- Tagging and pushing images to Docker Hub
- Using Docker volumes to serve static files with nginx

---

## Command step by step

```bash
docker build -t our-web-server -f web-server.Dockerfile .

docker run -d --name our-web-server our-web-server

docker ps

docker logs our-web-server

docker rm -f our-web-server

docker run -d --name our-web-server -p 5001:5000 our-web-server
```

## command 2

```bash
docker login

# you create  your own repo on docker hub
docker tag our-web-server xxxx/our-web-server:0.0.1

ocker push xxxx/our-web-server:0.0.1

docker run --name website -v "$PWD/website:/usr/share/nginx/html" -p 8080:80 --rm nginx
```
