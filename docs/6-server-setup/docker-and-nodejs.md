---
layout: default
title: Docker and Node.js
nav_order: 4
parent: Server Setup
---

# Docker and Node.js

[Return to Server Setup Chapter](index.md)

## Introduction

In this chapter, we'll learn how to containerize a Node.js Express application using Docker. This provides a solid foundation for deploying Node.js applications in a consistent, portable environment.

We'll cover:
- **Creating a Node.js Express application** - Building a web API with Express
- **Containerizing the Express app** - Creating Docker images for Node.js applications
- **Using Docker Compose** - Orchestrating containers for easy management
- **Testing and deployment** - Verifying your containerized application works correctly

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

// Serve static files from the public directory
app.use(express.static('public'));

// Routes
app.get('/api', (req, res) => {
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

**What is a Dockerfile?**

A Dockerfile is a text file that contains a series of instructions used to build a Docker image. Think of it as a list of commands that you want to run as the container is built. Each instruction in a Dockerfile creates a new layer in the image, and these layers are stacked on top of each other to form the final image. 

This is important for a nodejs image since we want the image to install all the packages that our project has, rather than installing the packages outside the container.

Create the Dockerfile in your project root directory:
```bash
nano Dockerfile
```

Add the following content:
```dockerfile
# Pick a node version, 
FROM node:lts

# Set the working directory inside the container. Think of this as a virtual enviornment.
WORKDIR /app

# Copy package.json and package-lock.json (if available)
COPY package*.json ./

# Install dependencies
RUN npm i

# Copy the rest of the application code
COPY . .

# Create a non-root user for security
# groupadd creates a new group called 'nodejs' with group ID 1001
# -g 1001: sets the group ID to 1001
# useradd creates a new user called 'nodejs' with user ID 1001
# -u 1001: sets the user ID to 1001
# -g nodejs: assigns the user to the nodejs group
# -s /bin/bash: sets the shell to bash
# -m: creates a home directory for the user
RUN groupadd -g 1001 nodejs && \
    useradd -u 1001 -g nodejs -s /bin/bash -m nodejs

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

**Note:** The `-t` flag stands for "tag" and assigns a name (`my-nodejs-app`) to the Docker image you're building. This creates a local image that you can reference later by name instead of using a random image ID. Without the `-t` flag, Docker would build the image but assign it a random ID, making it harder to reference in subsequent commands.

### Step 4: Test the Express Server

Before setting up nginx, let's test that our Express server is working correctly in the container:

```bash
# Run the container (we'll turn this into a compose file later)
docker run -d --name my-nodejs-container -p 3000:3000 my-nodejs-app

# Check if it's running
docker ps

# Test the application
curl http://localhost:3000/api
```

You should see a JSON response like:
```json
{
  "message": "Welcome to my Node.js Express app!",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

If you see this response, your Express server is working correctly in the container!

### Step 5: Create a Docker Compose File

Now let's create a Docker Compose file to make it easier to manage our container. This will replace the manual `docker run` command with a more manageable configuration.

Create a `docker-compose.yml` file in your project root:

```bash
nano docker-compose.yml
```

Add the following content:

```yaml
services:
  app:
    build: .
    container_name: my-nodejs-app
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    restart: unless-stopped
```

**What this Docker Compose file does:**

- **`services:`**: Defines the services (containers) to run
- **`app:`**: Name of our service
- **`build: .`**: Builds the image using the Dockerfile in the current directory
- **`container_name:`**: Gives the container a specific name
- **`ports:`**: Maps port 3000 from the container to port 3000 on the host
- **`environment:`**: Sets environment variables
- **`restart: unless-stopped`**: Automatically restarts the container if it stops (unless manually stopped)

Now you can start your application with:

```bash
docker compose up -d
```

The `-d` flag runs the container in detached mode (in the background).

To stop the application:

```bash
docker compose down
```

To view logs:

```bash
docker compose logs -f
```

**Note:** The `-f` flag stands for "follow" and continuously streams the log output in real-time. Without `-f`, the command shows logs once and exits. With `-f`, it keeps watching and displays new log entries as they appear. Press `Ctrl + C` to stop following the logs.

## Making Changes to Your Application

When you make changes to your `server.js` file or any other source code, you'll need to rebuild the container because Docker creates a snapshot of your code when building the image.

### Why Rebuilding is Necessary

The `COPY . .` command in your Dockerfile copies your current files into the image at build time. If you change your code after building, the container is still running the old version.

### How to Rebuild

You have a few options:

**Option 1: Rebuild and Restart (Current Setup)**
```bash
# Stop the current container
docker compose down

# Rebuild the image (this will include your changes)
docker compose build

# Start with the new image
docker compose up -d
```

**Option 2: One Command Rebuild**
```bash
# This stops, rebuilds, and starts in one command
docker compose up -d --build
```

### Development Alternative: Volume Mounting

For faster development, you can modify your `docker-compose.yml` to mount your source code as a volume, so changes are reflected immediately without rebuilding:

```yaml
services:
  app:
    build: .
    container_name: my-nodejs-app
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    volumes:
      - .:/app
      - /app/node_modules  # This prevents overwriting node_modules
    restart: unless-stopped
```

With volume mounting:
- Changes to your code are immediately reflected
- No need to rebuild the container
- Only need to restart if you change dependencies in `package.json`

### Production vs Development

- **Production**: Always rebuild to ensure consistency and security
- **Development**: Use volume mounting for faster iteration

### Step 6: Clean Up for nginx Setup

Now that we've confirmed the Express server works, let's clean up and prepare for the nginx setup:

```bash
# Stop and remove the test container
docker stop my-nodejs-container
docker rm my-nodejs-container
```

## Next Steps

You now have a working Node.js application running in a Docker container with Docker Compose! 

**What you've accomplished:**
- Created a Node.js Express application
- Containerized it with Docker
- Set up Docker Compose for easy management
- Tested the application successfully


## Development Setup with Volume Mounting

For active development, you'll want to set up volume mounting so that changes to your code are immediately reflected without rebuilding the container. This section shows you how to create a development-friendly setup.

### Step 1: Create a Development Docker Compose File

Create a separate development configuration:

```bash
nano docker-compose.dev.yml
```

Add the following content:

```yaml
services:
  app:
    build: .
    container_name: my-nodejs-app-dev
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    volumes:
      - .:/app
      - /app/node_modules
    restart: unless-stopped
    command: npm start
```

### Step 2: Understanding the Development Configuration

**Key differences from production setup:**

- **`volumes:`**: Mounts your current directory (`.`) to `/app` in the container
- **`/app/node_modules`**: Anonymous volume prevents your local `node_modules` from overwriting the container's dependencies
- **`NODE_ENV=development`**: Sets environment to development mode
- **`command: npm start`**: Explicitly runs the start command (useful for debugging)

### Step 3: Start Development Environment

```bash
# Start the development environment
docker compose -f docker-compose.dev.yml up -d

# Check that it's running
docker compose -f docker-compose.dev.yml ps

# View logs
docker compose -f docker-compose.dev.yml logs -f
```

### Step 4: Test Live Reloading

1. **Make a change** to your `server.js` file (e.g., change the message)
2. **Restart the Node.js process** inside the container:
   ```bash
   docker compose -f docker-compose.dev.yml restart app
   ```
3. **Test the change**:
   ```bash
   curl http://localhost:3000/api
   ```

### Step 5: Development Workflow

**For code changes:**
- Edit your files locally
- Restart the container: `docker compose -f docker-compose.dev.yml restart app`
- Test your changes immediately

**For dependency changes:**
- Edit `package.json`
- Rebuild the image: `docker compose -f docker-compose.dev.yml build`
- Restart: `docker compose -f docker-compose.dev.yml up -d`

### Step 6: Stop Development Environment

```bash
# Stop the development environment
docker compose -f docker-compose.dev.yml down
```

### Benefits of This Setup

- **Fast iteration**: No need to rebuild images for code changes
- **Immediate feedback**: Changes are reflected quickly
- **Dependency isolation**: Container's `node_modules` stays intact
- **Environment consistency**: Same Node.js version and dependencies as production

### When to Use Each Setup

- **Production setup** (`docker-compose.yml`): For final deployment and testing
- **Development setup** (`docker-compose.dev.yml`): For active coding and testing

## Summary

In this chapter, you've learned how to:

- Create a Node.js Express application with proper structure and security
- Containerize the application using Docker with best practices
- Use Docker Compose to orchestrate containers
- Test and manage your containerized application

This provides a solid foundation for your Node.js application. The next chapter will show you how to add nginx as a reverse proxy for production-ready deployments.

**[Return to Server Setup Chapter](index.md)**