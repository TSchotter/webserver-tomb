---
layout: default
title: Setting Up nginx with Docker and Node.js
nav_exclude: true
parent: Server Setup
---

# Setting Up nginx with Docker and Node.js

This chapter builds upon the Docker and Node.js setup from the previous chapter. We'll configure nginx as a reverse proxy for our Node.js application and use Docker Compose to orchestrate both services.

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
docker rmi my-nodejs-app_nginx

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
