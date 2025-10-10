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

## Docker vs Virtual Machines

You might be wondering: "Why use Docker instead of a traditional virtual machine?" Here's the key difference:

### Virtual Machines (VMs)
- **Full Operating System**: Each VM runs a complete operating system (Windows, Linux, etc.)
- **Heavy Resource Usage**: VMs consume significant CPU, memory, and disk space
- **Slow Startup**: Takes minutes to boot up a VM
- **Hardware Virtualization**: Uses a hypervisor to virtualize the entire hardware layer

### Docker Containers
- **Shared Operating System**: Containers share the host's operating system kernel
- **Lightweight**: Much smaller resource footprint - typically megabytes instead of gigabytes
- **Fast Startup**: Containers start in seconds, not minutes
- **Application-Level Isolation**: Isolates applications, not entire operating systems


> **Note for Windows Users**: While Docker containers are lightweight compared to full VMs, Docker Desktop on Windows actually runs a small Linux virtual machine in the background. This is because Docker containers are built on Linux kernel features, so Windows needs this small VM to provide the Linux environment that containers require. However, this VM is much smaller and more efficient than running full VMs for each application.

## Installing Docker

> These instructions are pulled from their official site here: https://docs.docker.com/engine/install/ubuntu/


### Step 1: Update Your System

First, make sure your system is up to date:

```bash
sudo apt update
sudo apt upgrade -y
```

### Step 2: Install Required Packages

Install packages that Docker needs:

```bash
sudo apt-get install ca-certificates curl
```
> This should be unessessary for our servers since the ubuntu version that our system are installed with should have these packages already installed.

> But there is also no harm in doing them to be safe.

### Step 3: Set Up Keyrings Directory

Create the directory for storing GPG keys:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

#### What are Keyrings?

A **keyring** is a secure storage location for cryptographic keys used to verify the authenticity of software packages (yes, even **more** security steps). Think of it as a digital "keychain" that holds the public keys needed to verify that software comes from trusted sources.

When you add a repository (like Docker's), your system needs the corresponding public key to verify that packages downloaded from that repository are legitimate. Without proper key verification, you could accidentally install malicious software that claims to be from Docker.

The `/etc/apt/keyrings` directory is the standard location where these verification keys are stored on modern Ubuntu systems.

### Step 4: Add Docker's Official GPG Key

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```

### Step 5: Add Docker Repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

> By default, docker isn't in our list of known packages that we see with `apt update`. What this long command above does is add it into our list of locations for ```apt update``` to look. There's a bunch of extra stuff in there to tie it to the keyring that was saved

And now one final update

```bash
sudo apt update
```

To pull from docker's official source a list of packages that they offer.

### Step 6: Install Docker Engine

Now that we have the list of repositories, including what docker has, we can install them.

```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

This is several packages, not just one giant package.

As a confirmation, lets check to see if docker is runnign properly. The Docker service *should* automatically start once the package is installed.

```
sudo systemctl status docker
```

If everything looks good and it shows the service as running, you're in the clear.

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

### Step 7: Add Your User to Docker Group

This allows you to run Docker commands without `sudo`:

```bash
sudo usermod -aG docker $USER
```

> **Important:** You'll need to log out and log back in (or restart your SSH session) for this change to take effect.

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

# Pull an image from Docker Hub (called nginx in this case)
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

> ### IMPORTANT
>
> Make sure your firewall allows the port.
> 

## Customizing nginx

### Creating a Custom HTML Page

Let's create a custom HTML page and serve it through nginx.

#### Step 1: Create a Directory for Your Website (or project)

```bash
mkdir ~/my-website
cd ~/my-website
```

#### Step 2: Create a Simple HTML File

Lets create an HTML file to be served by the nginx. In your folder, create `index.html`. Here is a simple html code to copy:


```bash
<!DOCTYPE html>
<html>
<body>
    <h1>Welcome to My Docker Website!</h1>
    <p>This page is being served from a Docker container running nginx!</p>
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


## Docker Compose (Optional)

For more complex setups, Docker Compose allows you to define multi-container applications using a YAML file.

This allows you to set up a docker container with options without having to remember the specific docker command like this:

```bash
docker run -d --name my-nginx -p 8080:80 -v ~/my-website:/usr/share/nginx/html nginx
```

Instead you would create a `docker-compose.yml` File in your project directory

```bash
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
> Make sure you're in the project directory with your docker compose file.


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


**[Return to Server Setup Chapter →](index.md)**
