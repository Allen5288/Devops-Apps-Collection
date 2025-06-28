# Kineteco Docker Compose Example

This project demonstrates a multi-container application using Docker Compose, simulating a simple e-commerce or business platform with a static storefront, a scheduler dashboard, and a MySQL database. It is designed for DevOps learners to practice orchestrating services, managing persistent data, and understanding service dependencies.

## Project Structure

- **docker-compose.yaml**: Main configuration file that defines all services, their build context, ports, dependencies, and shared volumes.
- **mysql/**: Contains:
  - `db.sql`: SQL script to initialize the database schema and data.
  - `env_vars`: Environment variables for MySQL root/user credentials and database name.
- **storefront/**: Nginx-based static website for the main storefront, with its own Dockerfile and static files.
- **scheduler/**: Nginx-based static dashboard for scheduling or admin, with its own Dockerfile and static files.

## Service Details

### database

- **Image**: MySQL (version set by the `TAG` environment variable, e.g., 5.7 or 8.0)
- **Initialization**: Loads schema/data from `mysql/db.sql` and uses credentials from `mysql/env_vars`.
- **Persistence**: Uses a named Docker volume (`kineteco`) to persist data across container restarts.

### storefront

- **Image**: Built from `storefront/` (Nginx + static HTML/JS/CSS)
- **Ports**: 80 (HTTP), 443 (HTTPS) mapped to host
- **Depends on**: database (ensures DB is up before starting)

### scheduler

- **Image**: Built from `scheduler/` (Nginx + static HTML/JS/CSS)
- **Port**: 81 (host) mapped to 80 (container)
- **Depends on**: database

## How to Run (Step by Step, Windows Bash)

1. **Open Git Bash** and navigate to the project directory:

   ```bash
   cd "/i/00_SoftwareDevopment/Devops-Apps-Collection/00 - Docker/03-DockerCompose-KinetecoExample/kineteco"
   ```

2. **Set the MySQL version tag**

   ```bash
   export TAG=8.0
   ```

   _This sets the MySQL version. You can use 5.7, 8.0, etc._
3. **Build and start all services with Docker Compose**:

   ```bash
   docker-compose up -d
   ```

   _This builds and starts the database, storefront, and scheduler containers in the background._
4. **Access the services in your browser**:
   - Storefront: [http://localhost:80](http://localhost:80)
   - Scheduler: [http://localhost:81](http://localhost:81)
5. **To stop and remove all containers, networks, and volumes**:

   ```bash
   docker-compose down -v
   ```

## How It Works

- The `database` service starts first, initializing MySQL with the provided schema and credentials.
- The `storefront` and `scheduler` services are built from their respective folders and wait for the database to be ready before starting.
- Static content is served by Nginx in both the storefront and scheduler containers.
- Data in MySQL is persisted using a Docker volume, so it is not lost when containers are stopped.

## Customization Tips

- You can edit the HTML files in `storefront/` and `scheduler/` to change the UI.
- To change database schema or seed data, edit `mysql/db.sql` and restart the stack.
- To use a different MySQL version, change the `TAG` environment variable before running `docker-compose up`.

---

This project is ideal for practicing Docker Compose, service orchestration, and basic DevOps workflows in a reproducible, local environment.
