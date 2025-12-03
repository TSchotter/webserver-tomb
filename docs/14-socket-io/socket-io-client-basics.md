---
layout: default
title: Socket.IO Client Basics
nav_order: 1
parent: Real-Time Communication with Socket.IO
---

# Socket.IO Client Basics

[&laquo; Return to Socket.IO Chapter](index.md)

## Introduction to Socket.IO

Socket.IO is a JavaScript library that enables real-time, bidirectional communication between web clients and servers. Unlike traditional HTTP requests where the client must initiate each communication, Socket.IO maintains a persistent connection that allows both the client and server to send data at any time.

### Why Use Socket.IO?

**Traditional HTTP (Request-Response):**
- Client sends a request → Server responds → Connection closes
- Client must constantly poll the server to check for updates
- Inefficient for real-time features

**Socket.IO (Persistent Connection):**
- Client connects once → Connection stays open
- Server can push data to client instantly
- Client can send data to server instantly
- Perfect for real-time features like chat, notifications, live updates

### Key Concepts

- **Connection**: A persistent connection between client and server
- **Events**: Messages sent between client and server (like "message", "data", "update")
- **Emit**: Sending an event (client emits to server, server emits to client)
- **Listen**: Receiving an event (client listens for server events, server listens for client events)

## Setting Up Socket.IO on the Client

### Step 1: Include Socket.IO Client Library

You can include Socket.IO in your HTML page using a CDN (Content Delivery Network). Add this script tag before your closing `</body>` tag:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Socket.IO Client Example</title>
</head>
<body>
    <h1>Socket.IO Client</h1>
    <button id="requestData">Request Data from Server</button>
    <div id="response"></div>

    <!-- Include Socket.IO client library -->
    <script src="https://cdn.socket.io/4.8.1/socket.io.min.js"></script>
    <script>
        // Your Socket.IO code will go here
    </script>
</body>
</html>
```

### Step 2: Connect to the Server

Create a connection to your Socket.IO server:

```javascript
// Connect to the Socket.IO server
// Replace '[domain name]' with your server's domain name
const socket = io('http://[domain name]');

// Listen for connection confirmation
socket.on('connect', () => {
    console.log('Connected to server!');
    console.log('Socket ID:', socket.id);
});
```

**What's happening:**
- `io('http://[domain name]')` creates a connection to the server
- The `connect` event fires when the connection is established
- `socket.id` is a unique identifier for this connection

### Step 3: Listen for Server Events

Set up listeners to receive data from the server:

```javascript
// Listen for a 'response' event from the server
socket.on('response', (data) => {
    console.log('Received from server:', data);
    
    // Display the JSON response
    const responseDiv = document.getElementById('response');
    responseDiv.innerHTML = '<pre>' + JSON.stringify(data, null, 2) + '</pre>';
});

// Listen for any errors
socket.on('error', (error) => {
    console.error('Socket error:', error);
});
```

### Step 4: Send Events to the Server

Emit events to the server when the button is clicked:

```javascript
// Get the button element
const button = document.getElementById('requestData');

// Add click event listener
button.addEventListener('click', () => {
    console.log('Button clicked! Sending request to server...');
    
    // Emit an event to the server
    socket.emit('requestData', {
        message: 'Hello from client!',
        timestamp: new Date().toISOString()
    });
});
```

## Complete Example

Here's a complete HTML page that demonstrates Socket.IO client-side communication:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Socket.IO Client Example</title>
</head>
<body>
    <h1>Socket.IO Client Example</h1>
    
    <div>
        <button id="requestData">Request Data from Server</button>
        <p id="status">Status: Not connected</p>
    </div>
    
    <div>
        <h2>Server Response:</h2>
        <pre id="response">No data received yet...</pre>
    </div>

    <!-- Include Socket.IO client library -->
    <script src="https://cdn.socket.io/4.8.1/socket.io.min.js"></script>
    <script>
        // Connect to the Socket.IO server
        const socket = io('http://[domain name]');

        // Update status when connected
        socket.on('connect', () => {
            console.log('Connected to server!');
            document.getElementById('status').textContent = 'Status: Connected (ID: ' + socket.id + ')';
        });

        // Update status when disconnected
        socket.on('disconnect', () => {
            console.log('Disconnected from server');
            document.getElementById('status').textContent = 'Status: Disconnected';
        });

        // Listen for 'response' event from server
        socket.on('response', (data) => {
            console.log('Received JSON response:', data);
            
            // Display the JSON response in a readable format
            const responseDiv = document.getElementById('response');
            responseDiv.textContent = JSON.stringify(data, null, 2);
        });

        // Listen for errors
        socket.on('error', (error) => {
            console.error('Socket error:', error);
            document.getElementById('response').textContent = 'Error: ' + error.message;
        });

        // Handle button click
        const button = document.getElementById('requestData');
        button.addEventListener('click', () => {
            console.log('Requesting data from server...');
            
            // Emit 'requestData' event to server with some data
            socket.emit('requestData', {
                message: 'Hello from client!',
                timestamp: new Date().toISOString(),
                clientId: socket.id
            });
        });
    </script>
</body>
</html>
```

## How It Works

1. **Page loads**: The Socket.IO client library is loaded from the CDN
2. **Connection established**: `io('http://[domain name]')` connects to the server
3. **Server confirms**: The `connect` event fires, confirming the connection
4. **User clicks button**: The button click handler emits a `requestData` event to the server
5. **Server responds**: The server receives the event and emits a `response` event back
6. **Client receives data**: The `response` event listener receives the JSON data and displays it

## Testing the Client

To test this client, you'll need a Socket.IO server running. **You only need ONE server file** - Socket.IO integrates with your existing Express server, so you don't need to run separate servers. This helps if you want to incorporate session information in the same javascript server.

This does not mean that you *can't* seperate the tasks (and place the socket server on another port). But keep in mind that if you're relying on session information (or user information), the socket server will also need to authenticate messages by the client on its own.


### Adding Socket.IO to Your Existing Express Server

```javascript
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');

const app = express();
// Create HTTP server from Express app
const server = http.createServer(app);

// Add Socket.IO to your existing server
const io = new Server(server, {
    cors: {
        origin: "*",
        methods: ["GET", "POST"]
    }
});

// Your existing Express routes
app.use(express.json());
app.use(express.static('public'));

app.get('/api/health', (req, res) => {
    res.json({ status: 'ok' });
});

// Add Socket.IO event handlers
io.on('connection', (socket) => {
    console.log('Client connected:', socket.id);

    socket.on('requestData', (data) => {
        socket.emit('response', {
            success: true,
            message: 'Data received successfully!',
            receivedData: data,
            serverTime: new Date().toISOString()
        });
    });

    socket.on('disconnect', () => {
        console.log('Client disconnected:', socket.id);
    });
});

// Use server.listen() instead of app.listen()
const PORT = 3000;
server.listen(PORT, () => {
    console.log(`Server running on http://[domain name]`);
});
```

**Key changes:**
- Import `http` and create a server from your Express app: `const server = http.createServer(app)`
- Create Socket.IO instance: `const io = new Server(server, {...})`
- Use `server.listen()` instead of `app.listen()`
- Your existing Express routes continue to work normally
- Socket.IO handles real-time connections on the same server

**You only need to run one server:** `node server.js` - this single server handles both your Express routes and Socket.IO connections!

## Key Points

- **Socket.IO maintains a persistent connection** between client and server
- **Events are the communication method** - both client and server can emit and listen to events
- **JSON data flows both ways** - you can send and receive any JSON data
- **Real-time updates** - the server can push data to clients instantly without them requesting it
- **Connection status** - you can track when clients connect and disconnect

## Common Patterns

### Sending Data with Events

```javascript
// Client sends data
socket.emit('eventName', { key: 'value' });

// Server receives data (on server side)
socket.on('eventName', (data) => {
    console.log(data); // { key: 'value' }
});
```

### Receiving Data from Server

```javascript
// Client listens for server event
socket.on('serverEvent', (data) => {
    console.log('Server sent:', data);
});

// Server sends data (on server side)
socket.emit('serverEvent', { message: 'Hello!' });
```

## Next Steps

Now that you understand the basics of Socket.IO on the client side, you can:
- Build real-time chat applications
- Create live notification systems
- Implement collaborative features
- Add real-time data updates to your applications

---

**[Return to Socket.IO Chapter →](index.md)**

