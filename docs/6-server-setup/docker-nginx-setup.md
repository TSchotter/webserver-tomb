---
layout: default
title: Setting Up nginx with Docker and Node.js
nav_order: 5
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
// Note: We don't include '/api' in our routes because nginx strips it when forwarding
// nginx receives: http://localhost/api/users
// nginx forwards to: http://backend-nodejs:3000/users (without /api)
app.get('/', (req, res) => {
    res.json({ 
        message: 'Hello from the API!',
        timestamp: new Date().toISOString()
    });
});

app.get('/health', (req, res) => {
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
FROM node:lts

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

### Why We Don't Include `/api` in Express Routes

**The routing flow works like this:**

1. **Client makes request**: `http://localhost/api/users`
2. **nginx receives the request**: Sees the `/api/` prefix
3. **nginx forwards to Node.js**: Strips the `/api` prefix and sends `http://backend-nodejs:3000/users`
4. **Express receives**: Just `/users` (without `/api`)
5. **Express route matches**: `app.get('/users', ...)` handles the request

**Why nginx strips the prefix:**
- nginx configuration: `location /api/ { proxy_pass http://nodejs_backend; }`
- The trailing slash in `proxy_pass` tells nginx to strip the matched prefix
- This is a common pattern in reverse proxy setups

**About the EXPOSE directive:**
- `EXPOSE 3000` in the Dockerfile documents which port the container uses
- This is different from `ports:` in docker-compose.yml which exposes ports to the host
- nginx can still reach the backend on port 3000 via the internal Docker network
- The EXPOSE directive is for documentation and internal container communication



## Setting Up nginx as a Reverse Proxy

Now let's configure nginx to act as a reverse proxy for our Node.js application.

### Step 1: Create nginx Configuration

Create the nginx configuration directory (if you haven't already):
```bash
mkdir nginx
```

Create the nginx configuration file:
```bash
nano nginx/default.conf
```

Add the following nginx configuration:
```nginx
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
        proxy_pass http://backend-nodejs:3000/;
    }
}
```

### Understanding the nginx Configuration

**What each part does:**

1. **`server` block**:
   - Defines how nginx handles requests for this virtual server
   - `listen 80` means nginx listens on port 80 (standard HTTP port)
   - `server_name _` means "accept requests for any domain name". If we want to only accept requests based on a certain domain, we would state the domain here instead of an underscore.

2. **`root /usr/share/nginx/html`**:
   - Sets the directory where nginx looks for static files
   - When someone requests `/index.html`, nginx looks in `/usr/share/nginx/html/index.html`. Remember, this is based on the container's internal enviornment. Your server's files are not important or observed.

3. **`index index.html`**:
   - Sets the default file to serve when someone visits the root URL (`/`)
   - If someone visits `http://localhost/`, nginx serves `index.html`

4. **`location /`**:
   - Handles all requests that don't match other location blocks
   - `try_files $uri $uri/ /index.html` means:
     - First, try to serve the exact file requested (`$uri`)
     - If that fails, try to serve a directory (`$uri/`)
     - If that fails, serve `index.html` (useful for single-page applications)

5. **`location /api/`**:
   - Handles requests that start with `/api/`
   - `proxy_pass http://backend-nodejs:3000/` forwards these requests directly to the Node.js container
   - The `/api/` prefix gets stripped when forwarding (so `/api/users` becomes `/users`)

**How requests are handled:**
- `http://localhost/` → nginx serves static files from `/usr/share/nginx/html/`. We will be telling the container to look in the public directory for these files.
- `http://localhost/api/users` → nginx forwards to backend-nodejs container as `/users`
- `http://localhost/style.css` → nginx serves the CSS file directly


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

**Understanding the CMD instruction:**

The `CMD ["nginx", "-g", "daemon off;"]` command tells Docker what to run when the container starts:

- **`nginx`**: Starts the nginx web server
- **`-g`**: Allows passing global directives to nginx
- **`daemon off;`**: Tells nginx to run in the foreground (not as a background daemon)

**Why `daemon off;` is important:**
- By default, nginx runs as a background daemon (service)
- Docker containers need a process running in the foreground to stay alive
- If nginx runs as a daemon, it starts and immediately exits, causing the container to stop
- `daemon off;` keeps nginx running in the foreground, which keeps the container running

**What happens without `daemon off;`:**
- nginx would start as a background service
- The container would think its job is done and exit
- Your nginx container would stop immediately after starting

This is a common pattern in Docker containers - the main process must run in the foreground to keep the container alive.

### Step 3: Create Static Files Directory

Create the public directory for static files within the nginx folder (if you haven't already done so):
```bash
mkdir nginx/public
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
  backend-nodejs:
    build: ./backend
    container_name: backend-nodejs
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
      - backend-nodejs
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
- Containers can reach each other using their service names (`backend-nodejs` and `nginx`)

**Container communication:**
- nginx can send requests to `http://backend-nodejs:3000` (using the service name)
- The nginx config directly references `http://backend-nodejs:3000/` in the proxy_pass directive
- This is why the nginx configuration works - it can resolve the `backend-nodejs` hostname

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
docker compose logs backend-nodejs
docker compose logs nginx
```

**Important: If you get connection errors, you may need to rebuild the containers:**

If you made changes after the first build in any file that isn't being copied into the docker (like the static files), you need to rebuild the images:

```bash
# Stop the current containers
docker compose down

# Rebuild the images (this includes your updated config files)
docker compose build

# Start with the new images
docker compose up -d
```

**Why rebuilding is necessary:**
- Docker images are built from the files at build time
- If you change files after building, the running containers still use the old files
- Rebuilding ensures your updated files are included in the new images

**Common errors and solutions:**

1. **Error**: `curl: (56) Recv failure: Connection reset by peer`
   - **Cause**: Container was built with empty/incorrect config files
   - **Solution**: Rebuild the containers with the correct configuration

2. **Error**: `<pre>Cannot GET /api/</pre>`
   - **Cause**: nginx is forwarding the request, but there's a mismatch between what nginx sends and what Express expects
   - **Solution**: Check your nginx configuration and Express routes

   **Debugging steps:**
   
   a) **Check what nginx is actually sending** by looking at the nginx logs:
   ```bash
   docker compose logs nginx
   ```
   
   b) **Check what Express is receiving** by looking at the Node.js logs:
   ```bash
   docker compose logs backend-nodejs
   ```
   
   c) **Verify your nginx config** - it should be:
   ```nginx
   location /api/ {
       proxy_pass http://backend-nodejs:3000/; 
   }
   ```
   
   d) **Your Express routes should be** (without /api prefix):
   ```javascript
   app.get('/', (req, res) => {  // This handles /api/ -> /
       res.json({ message: 'Hello from the API!' });
   });
   ```

   **If the prefix isn't being stripped**, you may need to rebuild the nginx container:
   ```bash
   docker compose down
   docker compose build nginx
   docker compose up -d
   ```

3. **Error**: nginx logs show `"GET /api/ HTTP/1.1" 404` but Node.js logs show no incoming requests
   - **Cause**: nginx cannot reach the Node.js container (network/container name issue)
   - **Solution**: Check container networking and names

   **Debugging steps:**
   
   a) **Check if both containers are running**:
   ```bash
   docker compose ps
   ```
   
   b) **Check if containers can communicate**:
   ```bash
   # Test from nginx container to Node.js container
   docker exec -it <nginx-container-name> ping backend-nodejs
   ```
   
   c) **Verify container names match**:
   - Your nginx config references `http://backend-nodejs:3000/`
   - Your docker-compose.yml should have a service named `backend-nodejs`
   - Check that the service name in docker-compose.yml matches what's in nginx config
   
   d) **Check if Node.js container is listening on the right interface**:
   - Make sure your Express app uses `app.listen(PORT, '0.0.0.0', ...)`
   - Not `app.listen(PORT, 'localhost', ...)` or `app.listen(PORT, '127.0.0.1', ...)`
   
   e) **Rebuild both containers**:
   ```bash
   docker compose down
   docker compose build
   docker compose up -d
   ```


## Cleanup

When you're done testing:

```bash
# Stop and remove all containers
docker compose down

# Remove images
docker rmi backend-nodejs
docker rmi my-nginx-proxy

# Remove unused resources
docker system prune -a
```




**[Previous: Docker and Node.js Setup](docker-and-nodejs.md)** | **[Return to Server Setup Chapter](index.md)**
