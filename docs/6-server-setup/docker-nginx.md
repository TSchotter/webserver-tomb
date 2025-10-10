---
layout: default
title: Docker and nginx
nav_order: 3
parent: Server Setup
---

# Docker and nginx

[&laquo; Return to Server Setup Chapter](index.md)

## Introduction to Docker

Docker is a containerization platform that allows you to package applications and their dependencies into lightweight, portable containers. This makes it easier to deploy applications consistently across different environments, from development to production.

In this chapter, we'll cover:
- **Installing Docker** - Setting up Docker on your Linux server
- **Basic Docker concepts** - Understanding containers, images, and the Docker ecosystem
- **Running nginx in a container** - Deploying a web server using Docker
- **Testing your setup** - Verifying that everything works correctly

## What is Docker?

Docker uses containers to package applications with all their dependencies. Think of a container as a lightweight virtual machine that includes:
- Your application code
- Runtime environment (Node.js, Python, etc.)
- System libraries and dependencies
- Configuration files

**Benefits of Docker:**
- **Consistency** - Applications run the same way everywhere
- **Isolation** - Containers don't interfere with each other
- **Portability** - Easy to move between development, testing, and production
- **Scalability** - Easy to scale applications up or down

## Installing Docker

### Step 1: Update Your System

First, make sure your system is up to date:

```bash
sudo apt update
sudo apt upgrade -y
```

### Step 2: Install Required Packages

Install packages that Docker needs:

```bash
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

### Step 3: Add Docker's Official GPG Key

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

### Step 4: Add Docker Repository

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 5: Install Docker Engine

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### Step 6: Add Your User to Docker Group

This allows you to run Docker commands without `sudo`:

```bash
sudo usermod -aG docker $USER
```

> **Important:** You'll need to log out and log back in (or restart your SSH session) for this change to take effect.

### Step 7: Verify Installation

Check if Docker is running:

```bash
sudo systemctl status docker
```

Start Docker if it's not running:

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

Test Docker with a simple command:

```bash
docker --version
```

You should see output like: `Docker version 24.0.7, build afdd53b`

## Basic Docker Concepts

### Images vs Containers

- **Image**: A read-only template that contains instructions for creating a container
- **Container**: A running instance of an image

Think of an image as a blueprint and a container as a house built from that blueprint.

### Common Docker Commands

```bash
# List running containers
docker ps

# List all containers (including stopped ones)
docker ps -a

# List downloaded images
docker images

# Pull an image from Docker Hub
docker pull nginx

# Run a container
docker run nginx

# Run a container in the background (detached mode)
docker run -d nginx

# Run a container with a custom name
docker run -d --name my-nginx nginx

# Stop a container
docker stop my-nginx

# Remove a container
docker rm my-nginx

# Remove an image
docker rmi nginx
```

## Running nginx in a Container

nginx is a popular web server that's commonly used as a reverse proxy, load balancer, and HTTP cache. Let's deploy it using Docker.

### Step 1: Pull the nginx Image

```bash
docker pull nginx
```

This downloads the official nginx image from Docker Hub.

### Step 2: Run nginx Container

```bash
docker run -d --name my-nginx -p 8080:80 nginx
```

Let's break down this command:
- `-d`: Run in detached mode (in the background)
- `--name my-nginx`: Give the container a custom name
- `-p 8080:80`: Map port 8080 on your host to port 80 in the container
- `nginx`: The image to use

### Step 3: Verify the Container is Running

```bash
docker ps
```

You should see output like:
```
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS                  NAMES
abc123def456   nginx     "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8080->80/tcp   my-nginx
```

### Step 4: Test nginx

Open a web browser and navigate to:
```
http://your-server-ip:8080
```

Or test from the command line:

```bash
curl http://localhost:8080
```

You should see the nginx welcome page with "Welcome to nginx!"

## Customizing nginx

### Creating a Custom HTML Page

Let's create a custom HTML page and serve it through nginx.

#### Step 1: Create a Directory for Your Website

```bash
mkdir -p ~/my-website
cd ~/my-website
```

#### Step 2: Create a Simple HTML File

```bash
cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Docker Website</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            background-color: #f0f0f0;
        }
        .container {
            background-color: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        h1 {
            color: #333;
            text-align: center;
        }
        .info {
            background-color: #e8f4f8;
            padding: 15px;
            border-radius: 5px;
            margin: 20px 0;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🐳 Welcome to My Docker Website!</h1>
        
        <div class="info">
            <h2>Container Information</h2>
            <p><strong>Server:</strong> nginx running in Docker</p>
            <p><strong>Status:</strong> ✅ Successfully deployed</p>
            <p><strong>Date:</strong> <script>document.write(new Date().toLocaleDateString());</script></p>
        </div>
        
        <h2>What's Next?</h2>
        <ul>
            <li>Learn more about Docker containers</li>
            <li>Explore nginx configuration</li>
            <li>Deploy your own web applications</li>
            <li>Set up reverse proxies and load balancing</li>
        </ul>
        
        <p>This page is being served from a Docker container running nginx!</p>
    </div>
</body>
</html>
EOF
```

#### Step 3: Stop the Current Container

```bash
docker stop my-nginx
docker rm my-nginx
```

#### Step 4: Run nginx with Volume Mounting

```bash
docker run -d --name my-nginx -p 8080:80 -v ~/my-website:/usr/share/nginx/html nginx
```

This command:
- Mounts your local `~/my-website` directory to `/usr/share/nginx/html` in the container
- This allows nginx to serve your custom HTML files

#### Step 5: Test Your Custom Website

Visit `http://your-server-ip:8080` again. You should now see your custom HTML page instead of the default nginx welcome page.

## Managing Your nginx Container

### Viewing Container Logs

```bash
# View recent logs
docker logs my-nginx

# Follow logs in real-time
docker logs -f my-nginx
```

### Accessing the Container Shell

```bash
# Execute a command in the running container
docker exec -it my-nginx /bin/bash

# This opens a bash shell inside the container
# You can explore the nginx configuration files
# Type 'exit' to leave the container
```

### Updating Your Website

Since we mounted a volume, you can simply edit files in `~/my-website` and they'll be immediately available:

```bash
# Edit your HTML file
nano ~/my-website/index.html

# Add a new page
echo "<h1>About Page</h1><p>This is the about page.</p>" > ~/my-website/about.html
```

Visit `http://your-server-ip:8080/about.html` to see your new page.

## Docker Compose (Optional)

For more complex setups, Docker Compose allows you to define multi-container applications using a YAML file.

### Create a docker-compose.yml File

```bash
cat > ~/docker-compose.yml << 'EOF'
version: '3.8'

services:
  nginx:
    image: nginx
    container_name: my-nginx
    ports:
      - "8080:80"
    volumes:
      - ~/my-website:/usr/share/nginx/html
    restart: unless-stopped
EOF
```

### Using Docker Compose

```bash
# Start the service
docker-compose up -d

# Stop the service
docker-compose down

# View logs
docker-compose logs
```

## Troubleshooting

### Common Issues

#### Port Already in Use
If you get an error about port 8080 being in use:

```bash
# Find what's using the port
sudo netstat -tlnp | grep :8080

# Or use a different port
docker run -d --name my-nginx -p 8081:80 nginx
```

#### Permission Denied
If you get permission errors:

```bash
# Make sure you're in the docker group
groups $USER

# If docker is not listed, add yourself and restart your session
sudo usermod -aG docker $USER
# Then log out and log back in
```

#### Container Won't Start
Check the logs:

```bash
docker logs my-nginx
```

### Useful Commands for Debugging

```bash
# Check Docker daemon status
sudo systemctl status docker

# Check available disk space
df -h

# Check memory usage
free -h

# List all Docker networks
docker network ls

# Inspect a container
docker inspect my-nginx
```

## Security Considerations

### Basic Security Practices

1. **Keep Docker Updated**
   ```bash
   sudo apt update
   sudo apt upgrade docker-ce
   ```

2. **Don't Run as Root**
   - Always use your regular user account (which should be in the docker group)
   - Avoid running containers with `--privileged` unless absolutely necessary

3. **Use Specific Image Tags**
   - Instead of `nginx`, use `nginx:1.25` for a specific version
   - This prevents unexpected updates

4. **Limit Container Resources**
   ```bash
   docker run -d --name my-nginx --memory="512m" --cpus="1.0" -p 8080:80 nginx
   ```

## Next Steps

Now that you have Docker and nginx running, you can:

1. **Deploy Your Own Applications** - Containerize your Node.js, Python, or other applications
2. **Set Up Reverse Proxies** - Use nginx to route traffic to multiple applications
3. **Learn Docker Networking** - Connect multiple containers together
4. **Explore Docker Volumes** - Persistent data storage for your applications
5. **Study nginx Configuration** - Customize nginx for your specific needs

## Summary

In this chapter, you've learned:

- ✅ How to install Docker on a Linux server
- ✅ Basic Docker concepts (images, containers, commands)
- ✅ How to run nginx in a Docker container
- ✅ How to serve custom content through nginx
- ✅ Basic container management and troubleshooting
- ✅ Introduction to Docker Compose for multi-container applications

Docker provides a powerful way to package and deploy applications consistently. Combined with nginx, you have a solid foundation for serving web content and building more complex application architectures.

**[Return to Server Setup Chapter →](index.md)**
