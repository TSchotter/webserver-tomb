---
layout: default
title: Building Your First Server
nav_order: 3
parent: NodeJS
---

# Building Your First Server

[&laquo; Return to NodeJS Chapter](index.md)

## Creating an HTTP Server

Let's build your first web server using NodeJS's built-in `http` module. This will help you understand the fundamentals of how web servers work.

It is highly recommended you create a seperate folder for your nodejs file structure.

### Basic Server Structure

```javascript
// server.js
const http = require('http');

// Create the server
const server = http.createServer((req, res) => {
    // Set response headers
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    
    // Send a response to the client (res.end() tells the client, 'no more is coming from the server')
    res.end('Hello from NodeJS! Your server is working correctly.');
});

// Start listening on a port
const PORT = 3000;
server.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
});
```
> ## Class related note
> 
> Use the port assigned to you (not 3000). This is the port that was on your static page.


### Running Your Server

Now that you have the basic server structure, let's run it:

1. **Save the code** above as `server.js`
2. **Open your terminal** and navigate to the directory containing `server.js`
3. **Run the server** with the following command:

```bash
node server.js
```

You should see output like:
```
Server running on http://localhost:[port]
```

**Test your server** 

If the server was running on your local machine, you could access the server by going to http://localhost:3000 (for port 3000). But if you're hosting on umainecos.org or toastcode.net, your server will be available in the node.js tab from the main site.

Currently, each of these sites is set up so that when you navigate to ```www.umainecos.org/[username]_node``` or ```www.toastcode.net/[username]_node```, it will redirect the request to 

```http://localhost:[your_port]``` on the server machine. We'll go over how to set up a server to make this sort of redirect in a future week.

**Important Notes:**
- The server will keep running until you stop it with `Ctrl+C` (or `Cmd+C` on Mac)
- If you see "Port 3000 is already in use", there are two common possibilities
  - Someone else is using your port by accident (or you're trying to use someone else's)
  - You have your server already running (either in another screen or another terminal).
- The server only responds to requests - without any route handling, you'll likely see an empty page or connection timeout

### Understanding Request and Response

Every HTTP request has:
- **Method**: GET, POST, PUT, DELETE, etc.
- **URL**: The path being requested
- **Headers**: Additional information
- **Body**: Data (for POST/PUT requests)

Every HTTP response has:
- **Status Code**: 200 (OK), 404 (Not Found), 500 (Error), etc.
- **Headers**: Content type, length, etc.
- **Body**: The actual content

### A Larger Example

```javascript
// server.js
const http = require('http');
const url = require('url');

// Route handler functions
function handleHome(req, res) {
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(`
        <html>
            <head><title>My First Server</title></head>
            <body>
                <h1>Welcome to My NodeJS Server!</h1>
                <p>This is your first web server.</p>
                <a href="/tschotter_node/about">About</a> | 
                <a href="/tschotter_node/contact">Contact</a>
            </body>
        </html>
    `);
}

function handleAbout(req, res) {
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(`
        <html>
            <head><title>About</title></head>
            <body>
                <h1>About This Server</h1>
                <p>This server was built with NodeJS!</p>
                <a href="/tschotter_node">Home</a> | 
                <a href="/tschotter_node/contact">Contact</a>
            </body>
        </html>
    `);
}

function handleContact(req, res) {
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(`
        <html>
            <head><title>Contact</title></head>
            <body>
                <h1>Contact Us</h1>
                <p>You can reach us at: contact@example.com</p>
                <a href="/tschotter_node">Home</a> | 
                <a href="/tschotter_node/about">About</a>
            </body>
        </html>
    `);
}

function handleApiHello(req, res) {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
        message: 'Hello from the API!',
        timestamp: new Date().toISOString()
    }));
}

function handleNotFound(req, res) {
    res.writeHead(404, { 'Content-Type': 'text/html' });
    res.end(`
        <html>
            <head><title>404 Not Found</title></head>
            <body>
                <h1>404 - Page Not Found</h1>
                <p>The page you're looking for doesn't exist.</p>
                <a href="/tschotter_node">Go Home</a>
            </body>
        </html>
    `);
}

// Main server logic
const server = http.createServer((req, res) => {
    // Parse the URL
    const parsedUrl = url.parse(req.url, true);
    const path = parsedUrl.pathname;
    const method = req.method;

    // Route to appropriate handler function
    if (path === '/' && method === 'GET') {
        handleHome(req, res);
    } else if (path === '/' && method === 'GET') {
        handleHome(req, res);
    }  else if (path === '/about' && method === 'GET') {
        handleAbout(req, res);
    } else if (path === '/contact' && method === 'GET') {
        handleContact(req, res);
    } else if (path === '/api/hello' && method === 'GET') {
        handleApiHello(req, res);
    } else {
        handleNotFound(req, res);
    }
});

const PORT = 3030;
server.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
});
```

> ### Important Tip
>
> Take note that the urls in the ```href``` attributes still have ```tschotter_node``` in my example above. Just because the incoming path will have that stripped when the server interprets it means nothing to the client, who needs a url to lead to a specific section of the site. Whatever is in the html we hand the client must be from the perspective of the client.




## Error Handling

Error handling is crucial for building reliable web servers. Without proper error handling, your server could crash when something goes wrong, leaving users with no response.

```javascript
const server = http.createServer((req, res) => {
    try {
        // Your server logic here
        handleRequest(req, res);
    } catch (error) {
        console.error('Server error:', error);
        res.writeHead(500, { 'Content-Type': 'text/html' });
        res.end(`
            <html>
                <head><title>Server Error</title></head>
                <body>
                    <h1>500 - Internal Server Error</h1>
                    <p>Something went wrong on our end.</p>
                </body>
            </html>
        `);
    }
});

// Handle server errors
server.on('error', (error) => {
    if (error.code === 'EADDRINUSE') {
        console.error(`Port ${PORT} is already in use`);
    } else {
        console.error('Server error:', error);
    }
});

const PORT = 3000;
server.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
});
```

### What's Happening Here:

**Try-Catch Block:**
- The `try` block contains your main server logic
- If any error occurs (like a file not found, invalid JSON, etc.), it's caught by the `catch` block
- Instead of crashing, the server sends a proper 500 error response to the client

**Error Response:**
- `res.writeHead(500)` sets the HTTP status code to 500 (Internal Server Error)
- The server sends a user-friendly error page instead of crashing
- The client gets a proper response, even when something goes wrong

**Server-Level Error Handling:**
- `server.on('error', ...)` handles errors that occur at the server level
- `EADDRINUSE` is a specific error when the port is already in use
- This prevents the server from crashing when it can't start


## Creating a Project with NPM

Now that you understand how to build a server, let's learn how to properly set up a NodeJS project using NPM (Node Package Manager).

### Initializing a New Project

```bash
npm init
```

This command will ask you several questions about your project such as the project name, the description, author, etc. You can press Enter to accept the defaults, or type your own values.

### What NPM Init Creates

After running `npm init`, you'll have a `package.json` file that looks like this:

```json
{
  "name": "my-nodejs-project",
  "version": "1.0.0",
  "description": "The description goes here",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "The man with some name",
  "license": "ISC"
}
```

### Adding Scripts to package.json

You can add custom scripts to make running your server easier:

```json
{
  "name": "my-nodejs-project",
  "version": "1.0.0",
  "description": "My first NodeJS server project",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "The man with no name",
  "license": "ISC"
}
```

The above is adding a command for your project, "start". Make sure the command is from the perspective of the package.json. If your server file is inside of a directory, you need to have the command be something like: 
```bash
 "start": "node serverfiles/server.js"
 ```

Now you can run your server with:
```bash
npm start
```

Why go through npm? Because it makes installing additional packages (which we will be using) much easier as well as creating your own commands to run and test your server.

## Next Steps

Now that you've built your first server and learned how to set up a proper NPM project, let's learn about modules and NPM to organize your NodeJS applications.

**[Next: NodeJS Modules and NPM →](nodejsModules.md)**
