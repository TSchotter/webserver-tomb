---
layout: default
title: Setting Up MySQL with Docker
nav_order: 2
parent: MySQL Server
---

# Setting Up MySQL with Docker

MySQL is one of the most popular relational database management systems. In the previous chapter, we learned how to install MySQL directly on a Linux server. Now, we'll explore an alternative approach: running MySQL in a Docker container, which provides an isolated, portable, and easily manageable database environment that's perfect for development and production.

## Introduction

In this guide, we'll set up a MySQL server using Docker. This approach offers several advantages:

- **Isolation** - MySQL runs in its own container, separate from other applications
- **Portability** - Easy to move between different environments
- **Version Control** - Specify exact MySQL versions
- **Easy Cleanup** - Remove everything by stopping and removing the container
- **Reproducibility** - Same setup every time

## Prerequisites

Before we begin, make sure you have:
- Docker installed and running on your system
- Basic familiarity with Docker commands (from [Docker and Node.js](../6-server-setup/docker-and-nodejs.md))
- Understanding of basic terminal commands

## Step 1: Pull the MySQL Docker Image

First, let's pull the official MySQL Docker image. We'll use MySQL 8.0, which is the current stable version:

```bash
docker pull mysql:8.0
```

This command downloads the MySQL 8.0 image from Docker Hub. You only need to do this once, or when you want to update to a newer version.

> **Note:** You can use other MySQL versions like `mysql:8.0`, `mysql:5.7`, or `mysql:latest`. For production, it's best to specify a specific version number rather than `latest`.

## Step 2: Create a Directory for MySQL Data

It's a good practice to persist your database data outside the container. This way, if you remove the container, your data remains safe. Create a directory for MySQL data:

```bash
mkdir -p ~/mysql-data
```

This creates a directory where MySQL will store all database files. The `-p` flag creates parent directories if they don't exist.

## Step 3: Create a Docker Compose File

Instead of running a long command in the terminal, we'll use Docker Compose to manage our MySQL container. This makes it easier to configure and manage. Create a directory for your MySQL setup:

```bash
mkdir -p ~/mysql-docker
cd ~/mysql-docker
```

Now create a `docker-compose.yml` file:

```bash
nano docker-compose.yml
```

Add the following content:

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: mysql-server
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: mySecurePassword123
      MYSQL_DATABASE: myapp_db
      MYSQL_USER: app_user
      MYSQL_PASSWORD: app_password
    ports:
      - "3306:3306"
    volumes:
      - ~/mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p$MYSQL_ROOT_PASSWORD"]
      interval: 10s
      timeout: 5s
      retries: 5
```

Let's break down this configuration:

- `version: '3.8'` - Docker Compose file format version
- `services:` - Defines the containers to run
- `mysql:` - Service name (you can reference it later)
- `image: mysql:8.0` - The MySQL Docker image to use
- `container_name: mysql-server` - Names the container "mysql-server"
- `restart: unless-stopped` - Automatically restarts the container if it stops (unless manually stopped)
- `environment:` - Sets environment variables for MySQL:
  - `MYSQL_ROOT_PASSWORD` - Root user password (change this to something secure!)
  - `MYSQL_DATABASE` - Creates a database called "myapp_db" automatically
  - `MYSQL_USER` - Creates a MySQL user named "app_user"
  - `MYSQL_PASSWORD` - Sets the password for the app_user
- `ports:` - Maps port 3306 (MySQL's default port) from container to host
- `volumes:` - Mounts the data directory for persistence (data survives container removal)
- `healthcheck:` - Allows Docker to verify the MySQL server is healthy

> **Security Warning:** Never use simple passwords like "password123" in production. Always use strong, randomly generated passwords!

Save and exit the file (in nano: `Ctrl+X`, then `Y`, then `Enter`).

## Step 4: Run the MySQL Container

Now let's start the MySQL container using Docker Compose:

```bash
docker-compose up -d
```

This command:
- `docker-compose up` - Creates and starts the containers defined in docker-compose.yml
- `-d` - Runs in detached mode (in the background)

The `-d` flag runs the container in the background so you can continue using your terminal.

## Step 5: Verify the Container is Running

Check if your MySQL container is running:

```bash
docker ps
```

You should see a container named "mysql-server" in the list. If it's not running, check the logs:

```bash
docker-compose logs mysql
```

Or using docker directly:

```bash
docker logs mysql-server
```

## Step 6: Connect to MySQL

Now let's connect to our MySQL server. There are two main ways:

### Option A: Connect from the Host Machine

If you have MySQL client installed on your host machine:

```bash
mysql -h 127.0.0.1 -P 3306 -u root -p
```

Enter the root password when prompted (`mySecurePassword123` from our example).

### Option B: Connect from Inside the Container

Connect to the MySQL server running inside the Docker container:

```bash
docker exec -it mysql-server mysql -u root -p
```

This command:
- `docker exec` - Executes a command in a running container
- `-it` - Interactive terminal mode
- `mysql-server` - The name of the container
- `mysql -u root -p` - The MySQL command to run

Enter your root password when prompted.

## Step 7: Test the Connection

Once connected, try some basic MySQL commands:

```sql
-- Show all databases
SHOW DATABASES;

-- Use the database we created
USE myapp_db;

-- Create a simple table
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert some test data
INSERT INTO users (name, email) VALUES 
    ('Luigi Mario', 'luigi@example.com'),
    ('Jane Smith', 'jane@example.com');

-- Query the data
SELECT * FROM users;

-- Exit MySQL
EXIT;
```

## Step 8: Container Management

Here are some useful commands for managing your MySQL container with Docker Compose:

```bash
# Start MySQL
docker-compose up -d

# Stop MySQL (removes containers but keeps volumes)
docker-compose down

# Stop MySQL and remove volumes (WARNING: This deletes your data!)
docker-compose down -v

# View logs
docker-compose logs mysql

# View recent logs (last 100 lines)
docker-compose logs --tail 100 mysql

# Follow logs in real-time
docker-compose logs -f mysql

# Restart the container
docker-compose restart mysql
```

You can also use standard Docker commands if you prefer:

```bash
# Stop the container
docker stop mysql-server

# Start a stopped container
docker start mysql-server

# View container logs
docker logs mysql-server
```

> **Important:** Using `docker-compose down` doesn't delete your data because of the volume mount. Your data persists in the `~/mysql-data` directory on your host machine. Only use `docker-compose down -v` if you want to completely remove everything including the data.

## Environment Variables Explained

The MySQL Docker image uses several environment variables:

- **MYSQL_ROOT_PASSWORD** - Sets the password for the root MySQL user (required)
- **MYSQL_DATABASE** - Creates a database with this name on container startup
- **MYSQL_USER** - Creates a new MySQL user with this name
- **MYSQL_PASSWORD** - Sets the password for the MYSQL_USER
- **MYSQL_ALLOW_EMPTY_PASSWORD** - Set to 'yes' to allow empty root password (not recommended)
- **MYSQL_RANDOM_ROOT_PASSWORD** - Set to 'yes' to generate a random root password (logs will show it)

## Security Best Practices

1. **Use Strong Passwords** - Always use complex passwords for production
2. **Limit Network Access** - Don't expose MySQL port to the public internet
3. **Use Non-Root Users** - Create application-specific users with limited privileges
4. **Regular Backups** - Backup your data regularly
5. **Update Regularly** - Keep MySQL and Docker images updated

## Troubleshooting

### Container Won't Start

Check the logs:
```bash
docker logs mysql-server
```

Common issues:
- Port 3306 already in use: Stop other MySQL instances or change the port mapping
- Permission issues with data directory: Check directory permissions

### Can't Connect to MySQL

1. Verify container is running: `docker ps`
2. Check if port is mapped correctly: `docker port mysql-server`
3. Try connecting from inside the container: `docker exec -it mysql-server mysql -u root -p`

### Forgot Root Password

If you need to reset the root password:

```bash
# Stop the container
docker stop mysql-server

# Start with --skip-grant-tables (temporary)
docker run --name mysql-server-temp \
  -e MYSQL_ROOT_PASSWORD=temporary \
  -v ~/mysql-data:/var/lib/mysql \
  mysql:8.0 mysqld --skip-grant-tables

# Connect and update password
docker exec -it mysql-server-temp mysql -u root
```

Then run:
```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'newSecurePassword123';
FLUSH PRIVILEGES;
```

## Next Steps

Now that you have MySQL running in Docker, you can:

- Learn to connect Node.js applications to MySQL
- Set up multiple databases for different projects
- Configure replication or clustering (advanced)
- Implement backup and restore procedures
- Connect to MySQL from other Docker containers

Your MySQL server is now ready to use! In the next sections, we'll learn how to connect to it from Node.js applications and perform database operations.

---

**[Previous: Setting Up MySQL Server](mysql-server-setup.md)** | **[Next: MySQL Server](index.md)**
