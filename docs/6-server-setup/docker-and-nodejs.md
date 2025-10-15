---
layout: default
title: Docker and Node.js
nav_order: 4
parent: Server Setup
---

# Docker and Node.js

[&laquo; Return to Server Setup Chapter](index.md)

## Introduction

In this chapter, we'll learn how to containerize a Node.js Express application using Docker and set up nginx as a reverse proxy. This is a common production setup where nginx handles static files, SSL termination, and load balancing while forwarding dynamic requests to your Node.js application.

We'll cover:
- **Creating a Node.js Express application** - Building a web API with Express
- **Containerizing the Express app** - Creating Docker images for Node.js applications
- **Setting up nginx as a reverse proxy** - Configuring nginx to forward requests to your app
- **Using Docker Compose** - Orchestrating multiple containers together
- **Production considerations** - Security, performance, and deployment best practices

## What is a Reverse Proxy?

A **reverse proxy** is a server that sits between clients and your application servers. Instead of clients connecting directly to your Node.js app, they connect to nginx, which then forwards requests to your app.

**Benefits of using nginx as a reverse proxy:**
- **Load Balancing** - Distribute requests across multiple app instances
- **SSL Termination** - Handle HTTPS certificates at the proxy level
- **Static File Serving** - Serve CSS, images, and JavaScript files efficiently
- **Caching** - Cache responses to improve performance
- **Security** - Hide your application servers behind the proxy

## Creating a Node.js Express Application

Let's start by creating a simple Express application that we'll containerize.

### Step 1: Set Up the Project Structure

```bash
mkdir -p ~/my-nodejs-app
cd ~/my-nodejs-app
```

### Step 2: Create package.json

In your project folder, use `npm init` to initialize the npm project, then install express:
```bash
npm init -y
npm i express
```

This will create a basic `package.json` file. You can edit it with nano if needed:
```bash
nano package.json
```

Make sure there's a start script in the scripts section:
```json
"scripts": {
    "start": "node server.js"
}
```


### Step 3: Create a basic express server.js

Create the server file:
```bash
nano server.js
```

Add the following code to the file:
```javascript
const express = require('express');

const app = express();
const PORT = process.env.PORT || 3000;  //you don't need to use your special IP anymore!


// Body parsing middleware
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Routes
app.get('/', (req, res) => {
  res.json({
    message: 'Welcome to my Node.js Express app!',
    timestamp: new Date().toISOString(),
  });
});


// 404 handler
app.use((req, res) => {
  res.status(404).json({ error: 'Route not found' });
});

// Start server
// Note: We use '0.0.0.0' instead of 'localhost' because Docker containers
// need to bind to all network interfaces to accept connections from outside the container
app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server is running on port ${PORT}`);
});
```

> ## Things to observe
>
> You'll notice that the server is giving json info at the root. In this case, the express server will operate purely as a backend.
> Nginx will route traffic to it based on the address, and will go to the static 


### Step 4: Create a Public Directory with Static Files

Create the public directory:
```bash
mkdir public
```

Create an HTML file:
```bash
nano public/index.html
```

Add the following HTML content:
```html
<html>
<head>
    <title>My App</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <h1>Welcome to My App!</h1>
    <p>This is a static page.</p>
</body>
</html>
```

Create a CSS file:
```bash
nano public/style.css
```

Add the following CSS content:
```css
body {
  font-family: Arial, sans-serif;
  margin: 20px;
  background-color:rgb(110, 112, 231);
}

h1 {
  color: #333;
}

.container {
  background-color: white;
  padding: 20px;
}
```

## Containerizing the Express Application

Now we'll create a Dockerfile to containerize our Express application. Make sure you're in your project directory (`~/my-nodejs-app`) where your `server.js`, `package.json`, and `public` folder are located.

### Step 1: Create a Dockerfile

Create the Dockerfile in your project root directory:
```bash
nano Dockerfile
```

Add the following content:
```dockerfile
# Use the official Node.js runtime as the base image
FROM node:24

# Set the working directory inside the container. Think of this as a virtual enviornment.
WORKDIR /app

# Copy package.json and package-lock.json (if available)
COPY package*.json ./

# Install dependencies
RUN npm i

# Copy the rest of the application code
COPY . .

# Create a non-root user for security
# addgroup creates a new group called 'nodejs' with group ID 1001
# -g 1001: sets the group ID to 1001
# -S: creates a system group (used for system services)
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Change ownership of the app directory to the nodejs user
RUN chown -R nodejs:nodejs /app
USER nodejs

# Expose the port the app runs on
EXPOSE 3000

# Define the command to run the application
CMD ["npm", "start"]
```

### Step 2: Create a .dockerignore File

Create the .dockerignore file:
```bash
nano .dockerignore
```

Add the following content:
```
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.nyc_output
coverage
.DS_Store
*.log
```
> The docker ignore operatates like the git ignore, when considering what goes into the container it will ignore the files you add to the ignore file. This is important if you want to test files outside the docker (which often involves creating a node_modules).


### Step 3: Build the Docker Image

```bash
docker build -t my-nodejs-app .
```

### Step 4: Test the Express Server

Before setting up nginx, let's test that our Express server is working correctly in the container:

```bash
# Run the container (we'll turn this into a compose file later)
docker run -d --name my-nodejs-container -p 3000:3000 my-nodejs-app

# Check if it's running
docker ps

# Test the application
curl http://localhost:3000
```

You should see a JSON response like:
```json
{
  "message": "Welcome to my Node.js Express app!",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

If you see this response, your Express server is working correctly in the container!

### Step 5: Clean Up for nginx Setup

Now that we've confirmed the Express server works, let's clean up and prepare for the nginx setup:

```bash
# Stop and remove the test container
docker stop my-nodejs-container
docker rm my-nodejs-container
```

## Setting Up nginx as a Reverse Proxy

Now let's configure nginx to act as a reverse proxy for our Node.js application.

### Step 1: Create nginx Configuration

Create the nginx configuration directory:
```bash
mkdir -p ~/nginx-config
```

Create the nginx configuration file:
```bash
nano ~/nginx-config/default.conf
```

Add the following nginx configuration:
```nginx
# Upstream defines where nginx should send requests
# nodejs_backend is a name we give to our backend server
# server nodejs-app:3000 means "send requests to the container named 'nodejs-app' on port 3000"
upstream nodejs_backend {
    server nodejs-app:3000;
}

server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # Serve static files
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API requests to Node.js app
    location /api/ {
        proxy_pass http://nodejs_backend;
    }
}
```

### Step 2: Create a Custom nginx Dockerfile

Create the nginx Dockerfile:
```bash
nano ~/nginx-config/Dockerfile
```

Add the following content:
```dockerfile
FROM nginx:alpine

# Copy custom nginx configuration
COPY default.conf /etc/nginx/conf.d/default.conf

# Copy static files from the Node.js app
# We'll mount the public directory from the Node.js container
VOLUME ["/usr/share/nginx/html"]

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

## Using Docker Compose

Docker Compose makes it easy to run multiple containers together. Let's create a configuration that runs both our Node.js app and nginx.

### Step 1: Create docker-compose.yml

Create the Docker Compose file:
```bash
nano docker-compose.yml
```

Add the following configuration:
```yaml
services:
  # Node.js Express application
  nodejs-app:
    build: .
    container_name: my-nodejs-app
    restart: unless-stopped
    environment:
      - NODE_ENV=production
      - PORT=3000
    networks:
      - app-network

  # nginx reverse proxy
  nginx:
    build: ~/nginx-config
    container_name: my-nginx-proxy
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./public:/usr/share/nginx/html
    depends_on:
      - nodejs-app
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

### Step 2: Start the Services

```bash
# Stop any existing containers
docker stop my-nodejs-container 2>/dev/null || true
docker rm my-nodejs-container 2>/dev/null || true

# Start the services with Docker Compose
docker-compose up -d

# Check the status
docker-compose ps
```

### Step 3: Test the Setup

```bash
# Test through nginx (port 80)
# Test static files (served by nginx)
curl http://localhost
curl http://localhost/index.html

# Test API (proxied to Node.js)
curl http://localhost/api/

# Check logs
docker-compose logs nodejs-app
docker-compose logs nginx
```

## Development Setup

For development, you might want to mount your source code as a volume so changes are reflected immediately.

### Development docker-compose.yml

Create the development Docker Compose file:
```bash
nano docker-compose.dev.yml
```

Add the following configuration:
```yaml
version: '3.8'

services:
  # Node.js Express application (development)
  nodejs-app:
    build: .
    container_name: my-nodejs-app-dev
    restart: unless-stopped
    environment:
      - NODE_ENV=development
      - PORT=3000
    volumes:
      - .:/app
      - /app/node_modules
    networks:
      - app-network
    command: npm run dev

  # nginx reverse proxy
  nginx:
    build: ~/nginx-config
    container_name: my-nginx-proxy-dev
    restart: unless-stopped
    ports:
      - "80:80"
    depends_on:
      - nodejs-app
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

Start development environment:

```bash
docker-compose -f docker-compose.dev.yml up -d
```

## Production Considerations

### Environment Variables

Create a `.env` file for production configuration:
```bash
nano .env
```

Add the following environment variables:
```
NODE_ENV=production
PORT=3000
DB_HOST=your-database-host
DB_PORT=5432
DB_NAME=your-database-name
JWT_SECRET=your-jwt-secret
API_RATE_LIMIT=100
```

Create a production Docker Compose file:
```bash
nano docker-compose.prod.yml
```

Add the following configuration:
```yaml
version: '3.8'

services:
  nodejs-app:
    build: .
    container_name: my-nodejs-app-prod
    restart: unless-stopped
    env_file:
      - .env
    networks:
      - app-network

  nginx:
    build: ~/nginx-config
    container_name: my-nginx-proxy-prod
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - nodejs-app
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

### Security Best Practices

1. **Use non-root users** in containers
2. **Keep images updated** with security patches
3. **Use specific image tags** instead of `latest`
4. **Implement proper logging** and monitoring
5. **Use secrets management** for sensitive data
6. **Configure firewall rules** appropriately

### Monitoring and Logging

```bash
# View logs from all services
docker-compose logs -f

# View logs from specific service
docker-compose logs -f nodejs-app

# Monitor resource usage
docker stats

# Check container health
docker-compose ps
```

## Scaling Your Application

### Horizontal Scaling

You can scale your Node.js application by running multiple instances:

```bash
# Scale the nodejs-app service to 3 instances
docker-compose up -d --scale nodejs-app=3

# Check running containers
docker-compose ps
```

Update your nginx configuration to handle multiple backend instances:
```bash
nano ~/nginx-config/nginx-scaled.conf
```

Add the following configuration:
```nginx
upstream nodejs_backend {
    server nodejs-app_1:3000;
    server nodejs-app_2:3000;
    server nodejs-app_3:3000;
}

# Rest of the configuration remains the same...
```

## Troubleshooting

### Common Issues

#### Container Won't Start
```bash
# Check logs
docker-compose logs nodejs-app

# Check if port is already in use
sudo netstat -tlnp | grep :80
```

#### nginx Can't Connect to Node.js App
```bash
# Check if containers are on the same network
docker network ls
docker network inspect my-nodejs-app_app-network

# Test connectivity between containers
docker exec my-nginx-proxy ping nodejs-app
```

#### Performance Issues
```bash
# Monitor resource usage
docker stats

# Check nginx access logs
docker exec my-nginx-proxy tail -f /var/log/nginx/access.log
```

## Cleanup

When you're done testing:

```bash
# Stop and remove all containers
docker-compose down

# Remove images
docker rmi my-nodejs-app
docker rmi my-nodejs-app_nginx

# Remove unused resources
docker system prune -a
```

## Summary

In this chapter, you've learned how to:

- Create a Node.js Express application with proper structure and security
- Containerize the application using Docker with best practices
- Set up nginx as a reverse proxy to handle requests
- Use Docker Compose to orchestrate multiple containers
- Configure the setup for both development and production
- Implement monitoring, logging, and scaling strategies

This setup provides a solid foundation for deploying Node.js applications in production with proper separation of concerns, security, and scalability.

**[Return to Server Setup Chapter →](index.md)**
