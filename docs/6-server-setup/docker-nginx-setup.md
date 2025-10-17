---
layout: default
title: Setting Up nginx with Docker and Node.js
nav_exclude: true
parent: Server Setup
---

# Setting Up nginx with Docker and Node.js

This chapter builds upon the Docker and Node.js setup from the previous chapter. We'll configure nginx as a reverse proxy for our Node.js application and use Docker Compose to orchestrate both services.

Effectively, we'll have two docker containers. One container that is the backend express which is for api requests, and a container nginx acting as both the reverse proxy to route traffic to the backend and to handle the static pages when it doesn't match the api request url.

## Project Structure

We'll organize our project with separate folders for each container to maintain clear separation of concerns:

```
project-root/
├── backend/                 # Express.js backend container
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── nginx/                  # nginx reverse proxy container
│   ├── Dockerfile
│   ├── default.conf
│   └── public/             # Static files served by nginx
├── docker-compose.yml      # Orchestrates both containers
└── docker-compose.dev.yml  # Development configuration
```

This structure makes it clear that each folder represents one container and helps maintain separation between the backend API and the reverse proxy.

## Modifying the Backend for nginx Integration

Since nginx will now handle static files, we need to modify the Express.js backend to focus only on API endpoints.

### Step 1: Update server.js

Modify your `backend/server.js` file to remove static file serving:

```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Remove static file serving - nginx will handle this
// app.use(express.static('public')); // Remove this line

// API Routes
app.get('/api/', (req, res) => {
    res.json({ 
        message: 'Hello from the API!',
        timestamp: new Date().toISOString()
    });
});

app.get('/api/health', (req, res) => {
    res.json({ 
        status: 'healthy',
        service: 'nodejs-backend'
    });
});

// Start server
app.listen(PORT, '0.0.0.0', () => {
    console.log(`Server running on port ${PORT}`);
});
```

### Step 2: Update Backend Dockerfile

Modify your `backend/Dockerfile` to remove static file copying:

```dockerfile
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy source code
COPY . .

# Remove static file copying - nginx will handle this
# COPY public/ ./public/  # Remove this line if it exists

# Expose port (internal to container network)
EXPOSE 3000

# Start the application
CMD ["node", "server.js"]
```

**Key changes:**
- Removed `app.use(express.static('public'))` from server.js
- Removed static file copying from Dockerfile
- Backend now focuses purely on API endpoints
- nginx will handle all static file serving

**About the EXPOSE directive:**
- `EXPOSE 3000` in the Dockerfile documents which port the container uses
- This is different from `ports:` in docker-compose.yml which exposes ports to the host
- nginx can still reach the backend on port 3000 via the internal Docker network
- The EXPOSE directive is for documentation and internal container communication

## Prerequisites

Before starting this chapter, make sure you have:
- Completed the [Docker and Node.js setup](docker-and-nodejs.md)
- A working Node.js application containerized with Docker
- Docker Compose installed

## Setting Up nginx as a Reverse Proxy

Now let's configure nginx to act as a reverse proxy for our Node.js application.

### Step 1: Create nginx Configuration

Create the nginx configuration directory:
```bash
mkdir -p nginx
```

Create the nginx configuration file:
```bash
nano nginx/default.conf
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
nano nginx/Dockerfile
```

Add the following content:
```dockerfile
FROM nginx:alpine

# Copy custom nginx configuration
COPY default.conf /etc/nginx/conf.d/default.conf

# Copy static files to nginx html directory
COPY public/ /usr/share/nginx/html/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Step 3: Create Static Files Directory

Create the public directory for static files within the nginx folder:
```bash
mkdir -p nginx/public
```

Create a sample index.html file:
```bash
nano nginx/public/index.html
```

Add the following content:
```html
<!DOCTYPE html>
<head>
    <title>Double Container Test</title>
</head>
<body>
    <h1>Welcome to My Application</h1>
    <p>This is served by nginx as a static file.</p>
    <p>API requests to <code>/api/</code> will be proxied to the Node.js backend.</p>
</body>
</html>
```

## Using Docker Compose

Docker Compose makes it easy to run multiple containers together. Let's create a configuration that runs both our Node.js app and nginx.

### Step 1: Create docker-compose.yml

Create the Docker Compose file (this will be at the root of the project):
```bash
nano docker-compose.yml
```

Add the following configuration:
```yaml
services:
  # Node.js Express application
  nodejs-app:
    build: ./backend
    container_name: my-nodejs-app
    restart: unless-stopped
    environment:
      - NODE_ENV=production
      - PORT=3000
    # No ports exposed - only nginx can access this container
    networks:
      - app-network

  # nginx reverse proxy
  nginx:
    build: ./nginx
    container_name: my-nginx-proxy
    restart: unless-stopped
    ports:
      - "80:80"  # Only nginx exposes ports to the host
    depends_on:
      - nodejs-app
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

### Understanding Docker Networks

The `networks` section in Docker Compose creates a custom network that allows containers to communicate with each other. Here's what it does:

**Why we need networks:**
- By default, Docker containers are isolated and cannot communicate with each other
- Our nginx container needs to send requests to the Node.js container
- The custom network allows containers to find each other by name

**How it works:**
- `app-network` is the name of our custom network
- `driver: bridge` creates a bridge network (the default type)
- Both containers join this network, allowing them to communicate
- Containers can reach each other using their service names (`nodejs-app` and `nginx`)

**Container communication:**
- nginx can send requests to `http://nodejs-app:3000` (using the service name)
- The `upstream nodejs_backend` in nginx config references `server nodejs-app:3000`
- This is why the nginx configuration works - it can resolve the `nodejs-app` hostname

**Security benefits:**
- Only containers on the same network can communicate
- External traffic cannot directly access the Node.js container (only through nginx)
- This creates a secure internal network for your application

**Port exposure strategy:**
- **Backend container**: No ports exposed to host - only accessible via internal network
- **nginx container**: Exposes ports 80/443 to host - acts as the only entry point
- This ensures all traffic goes through nginx, providing better security and control

### Step 2: Start the Services

```bash
# Stop any existing containers (multiple approaches)

# Option 1: Stop all running containers
docker stop $(docker ps -q)

# Option 2: Stop containers by project name (recommended)
docker compose down
# you will need to be in the folder containing the docker compose file for this to work

# Option 3: Stop specific containers if you know their names
docker stop my-nodejs-container my-nginx-proxy
docker rm my-nodejs-container my-nginx-proxy
```

# Start the services with Docker Compose
docker compose up -d

# Check the status
docker compose ps
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
docker compose logs nodejs-app
docker compose logs nginx
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
services:
  # Node.js Express application (development)
  nodejs-app:
    build: ./backend
    container_name: my-nodejs-app-dev
    restart: unless-stopped
    environment:
      - NODE_ENV=development
      - PORT=3000
    volumes:
      - ./backend:/app
      - /app/node_modules
    # No ports exposed - only nginx can access this container
    networks:
      - app-network
    command: npm run dev

  # nginx reverse proxy
  nginx:
    build: ./nginx
    container_name: my-nginx-proxy-dev
    restart: unless-stopped
    ports:
      - "80:80"  # Only nginx exposes ports to the host
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
docker compose -f docker-compose.dev.yml up -d
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
services:
  nodejs-app:
    build: ./backend
    container_name: my-nodejs-app-prod
    restart: unless-stopped
    env_file:
      - .env
    # No ports exposed - only nginx can access this container
    networks:
      - app-network

  nginx:
    build: ./nginx
    container_name: my-nginx-proxy-prod
    restart: unless-stopped
    ports:
      - "80:80"   # HTTP
      - "443:443" # HTTPS
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
docker compose logs -f

# View logs from specific service
docker compose logs -f nodejs-app

# Monitor resource usage
docker stats

# Check container health
docker compose ps
```

## Scaling Your Application

### Horizontal Scaling

You can scale your Node.js application by running multiple instances:

```bash
# Scale the nodejs-app service to 3 instances
docker compose up -d --scale nodejs-app=3

# Check running containers
docker compose ps
```

Update your nginx configuration to handle multiple backend instances:
```bash
nano nginx/nginx-scaled.conf
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
docker compose logs nodejs-app

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
docker compose down

# Remove images
docker rmi my-nodejs-app
docker rmi my-nginx-proxy

# Remove unused resources
docker system prune -a
```

## Summary

In this chapter, you've learned how to:

- Set up nginx as a reverse proxy for Node.js applications
- Configure nginx to serve static files and proxy API requests
- Use Docker Compose to orchestrate multiple containers
- Set up development and production environments
- Implement monitoring, logging, and scaling strategies
- Troubleshoot common issues with containerized applications

This setup provides a solid foundation for deploying Node.js applications in production with proper separation of concerns, security, and scalability.

**[Previous: Docker and Node.js Setup](docker-and-nodejs.md)** | **[Return to Server Setup Chapter](index.md)**
