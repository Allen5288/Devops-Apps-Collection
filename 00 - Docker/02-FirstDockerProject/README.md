# Docker: Your First Project

This project demonstrates how to containerize a simple Python Flask web application using Docker. You will learn how to build a Docker image, run it as a container, and access the web app from your browser. The project is ideal for beginners who want hands-on experience with Docker and Python web development.

## Features

- Python Flask web app with multiple routes and HTML templates
- Dockerfile for building a reproducible image
- Example of port mapping and container management

## Prerequisites

- Docker Desktop
- VSCode
- Git Bash (or any Bash terminal on Windows)

## Quick Start: Run the App with Docker

1. Open Git Bash and navigate to the project directory:

   ```bash
   cd "/i/00_SoftwareDevopment/Devops-Apps-Collection/00 - Docker/02-FirstDockerProject"
   ```

2. Build the Docker image:

   ```bash
   docker build -t flask-demo-app -f dockerfile .
   ```

3. Run the Docker container:

   ```bash
   docker run -d --name flask-demo-app -p 5000:5000 flask-demo-app
   ```

4. Open your browser and visit: [http://localhost:5000](http://localhost:5000)

To stop and remove the container:

```bash
docker rm -f flask-demo-app
```

---

## Installing (Course Instructions)

1. To use these exercise files, you must have the following installed:
   - Docker Desktop
   - VSCode
   - Git Bash
2. Clone this repository into your local machine using the Git Bash command:

   ```bash
   git clone xxxxx
   ```

3. Switch between branches using:

   ```bash
   git checkout CHAPTER#_MOVIE#
   ```

4. View all available branches:

   ```bash
   git branch
   ```
