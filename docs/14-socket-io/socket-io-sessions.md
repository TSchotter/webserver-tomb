---
layout: default
title: Connecting Express-Session to Socket.IO
nav_order: 2
parent: Real-Time Communication with Socket.IO
---

# Connecting Express-Session to Socket.IO

[&laquo; Return to Socket.IO Chapter](index.md)

Now it's time to have the io socket interact with the session.

Some of the advantages of having these two aspects communicate with each other:
- **Identify users** in Socket.IO connections (who is connected?)
- **Authenticate Socket.IO events** (is this user allowed to do this?)
- **Access user data** from sessions (username, user ID, permissions, etc.)
- **Maintain consistency** between HTTP requests and WebSocket connections

By connecting express-session to Socket.IO, you can access the same session data in both your Express routes and Socket.IO event handlers.

## Prerequisites

Before starting, make sure you have:
- Completed [Socket.IO Client Basics](socket-io-client-basics.md)
- Understanding of [Server-Side Sessions](../7-cookies-sessions/server-side-sessions.md)
- An Express server with express-session configured

## How Session Sharing Works

Socket.IO connections don't automatically have access to Express sessions. The session cookie is sent during the initial handshake, but you need to configure Socket.IO to read and use that session data.

There are two main approaches:
1. **Official method using `io.engine.use()`** - The recommended approach from the official Socket.IO documentation. Simple and straightforward.
2. **Manual session parsing** - More control but requires manually parsing cookies and loading sessions from the session store.

We'll cover both approaches, starting with the official method which is simpler and recommended.

## Method 1: Official Method (Recommended)

This is the official method recommended by Socket.IO. 

https://socket.io/how-to/use-with-express-session

It's simple and doesn't require any additional packages.

### Step 1: Install Required Packages

```bash
npm install express express-session socket.io
```

### Step 2: Configure Session Sharing

```javascript
// server.js
const express = require('express');
const http = require('http');
const session = require('express-session');
const { Server } = require('socket.io');

const app = express();
const server = http.createServer(app);

// Configure express-session
const sessionMiddleware = session({
    secret: 'your-secret-key-change-this-in-production',
    resave: false,
    saveUninitialized: false,
    cookie: {
        secure: false,
        maxAge: 24 * 60 * 60 * 1000
    }
});

app.use(sessionMiddleware);

// Your Express routes
app.use(express.json());
app.use(express.static('public'));

app.post('/api/login', (req, res) => {
    const { username, password } = req.body;
    
    req.session.userId = 123;
    req.session.username = username;
    req.session.isLoggedIn = true;
    
    res.json({ success: true, message: 'Logged in' });
});

// Create Socket.IO server
const io = new Server(server, {
    cors: {
        origin: "*",
        methods: ["GET", "POST"]
    }
});

// Share session with Socket.IO (official method)
io.engine.use(sessionMiddleware);

// Now session is available in socket.request.session
io.on('connection', (socket) => {
    const session = socket.request.session;
    const userId = session.userId;
    const username = session.username;
    const isLoggedIn = session.isLoggedIn;
    
    console.log('Client connected:', socket.id);
    console.log('User:', username, 'ID:', userId);
    
    // Authentication check
    if (!isLoggedIn) {
        socket.emit('error', { message: 'Authentication required' });
        socket.disconnect();
        return;
    }
    
    // Listen for events
    socket.on('requestData', (data) => {
        socket.emit('response', {
            success: true,
            message: `Hello ${username}!`,
            userId: userId,
            data: data
        });
    });
    
    socket.on('disconnect', () => {
        console.log('Client disconnected:', socket.id);
    });
});

const PORT = 3000;
server.listen(PORT, () => {
    console.log(`Server running on http://[domain name]`);
});
```

### Step 3: Update Client-Side HTML

When using sessions with Socket.IO, you need to ensure the client sends cookies with the Socket.IO connection. Update your HTML to include `withCredentials: true` in the connection options:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Socket.IO with Sessions</title>
</head>
<body>
    <h1>Socket.IO with Session Authentication</h1>
    
    <div>
        <button id="loginBtn">Login</button>
        <button id="connectBtn">Connect to Socket.IO</button>
        <button id="requestDataBtn">Request Data</button>
    </div>
    
    <div id="status">Not connected</div>
    <div id="response"></div>

    <script src="https://cdn.socket.io/4.8.1/socket.io.min.js"></script>
    <script>
        let socket = null;
        
        // Login function
        document.getElementById('loginBtn').addEventListener('click', async () => {
            const response = await fetch('/api/login', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                credentials: 'include', // Important: include cookies
                body: JSON.stringify({
                    username: 'testuser',
                    password: 'password123'
                })
            });
            
            const data = await response.json();
            document.getElementById('status').textContent = data.message;
        });
        
        // Connect to Socket.IO with credentials
        document.getElementById('connectBtn').addEventListener('click', () => {
            // Important: Set withCredentials to true so cookies are sent
            socket = io('http://[domain name]', { // CHANGE THIS TO YOUR URL
                withCredentials: true
            });
            
            socket.on('connect', () => {
                document.getElementById('status').textContent = 'Connected!';
            });
            
            socket.on('error', (error) => {
                document.getElementById('status').textContent = 'Error: ' + error.message;
            });
            
            socket.on('response', (data) => {
                document.getElementById('response').innerHTML = 
                    '<pre>' + JSON.stringify(data, null, 2) + '</pre>';
            });
        });
        
        // Request data
        document.getElementById('requestDataBtn').addEventListener('click', () => {
            if (socket) {
                socket.emit('requestData', {
                    message: 'Hello from client!'
                });
            }
        });
    </script>
</body>
</html>
```

**Key changes on the client side:**
1. **`withCredentials: true`** - Must be set in the Socket.IO connection options to send cookies
2. **`credentials: 'include'`** - Must be set in fetch requests to send cookies with login

Without these settings, the session cookie won't be sent, and Socket.IO won't have access to the session.

### Modifying Session Data

When you modify session data in Socket.IO, you need to manually reload and save the session since it's not bound to a single HTTP request:

```javascript
io.on('connection', (socket) => {
    const req = socket.request;
    
    socket.on('updateCount', () => {
        // Reload session to get latest data
        req.session.reload((err) => {
            if (err) {
                return socket.disconnect();
            }
            
            // Modify session
            req.session.count = (req.session.count || 0) + 1;
            
            // Save session
            req.session.save();
            
            // Emit updated count
            socket.emit('countUpdated', { count: req.session.count });
        });
    });
});
```

### Using Session ID for Rooms

You can use the session ID to create rooms for each user session:

```javascript
io.on('connection', (socket) => {
    const sessionId = socket.request.session.id;
    
    // Join a room based on session ID
    socket.join(sessionId);
    
    // Now you can send messages to this specific session
    // from your Express routes
});

// In an Express route, you can emit to a specific session:
app.post('/api/notify', (req, res) => {
    const sessionId = req.session.id;
    io.to(sessionId).emit('notification', {
        message: 'You have a new notification!'
    });
    res.json({ success: true });
});
```

### Handling Session Expiration

You may want to periodically reload the session to check if it has expired:

```javascript
const SESSION_RELOAD_INTERVAL = 30 * 1000; // 30 seconds

io.on('connection', (socket) => {
    const req = socket.request;
    
    // Periodically reload session to check expiration
    const timer = setInterval(() => {
        req.session.reload((err) => {
            if (err) {
                // Session expired or error - force reconnect
                socket.conn.close();
                clearInterval(timer);
            }
        });
    }, SESSION_RELOAD_INTERVAL);
    
    socket.on('disconnect', () => {
        clearInterval(timer);
    });
});
```

## Method 2: Manual Session Access

This method manually parses the session cookie and loads the session data in Socket.IO connection handlers.

### Step 1: Install Required Packages

```bash
npm install express express-session socket.io cookie-parser
```

### Step 2: Set Up Express with Sessions

```javascript
// server.js
const express = require('express');
const http = require('http');
const session = require('express-session');
const cookieParser = require('cookie-parser');
const { Server } = require('socket.io');

const app = express();
const server = http.createServer(app);

// Cookie parser middleware (needed to read session cookie)
app.use(cookieParser());

// Session middleware configuration
app.use(session({
    secret: 'its-a-secret',
    resave: false,
    saveUninitialized: false,
    cookie: {
        secure: false, // Set to true if using HTTPS
        maxAge: 24 * 60 * 60 * 1000 // 24 hours
    }
}));

// Your Express routes
app.use(express.json());
app.use(express.static('public'));

// Example: Login route that sets session
app.post('/api/login', (req, res) => {
    const { username, password } = req.body;
    
    // Your authentication logic here...
    // For this example, we'll just set the session
    
    req.session.userId = 123;
    req.session.username = username;
    req.session.isLoggedIn = true;
    
    res.json({ success: true, message: 'Logged in' });
});

// Create Socket.IO server
const io = new Server(server, {
    cors: {
        origin: "*",
        methods: ["GET", "POST"]
    }
});
```

### Step 3: Access Session in Socket.IO

To access the session in Socket.IO, you need to parse the session cookie from the handshake. Here's how:

```javascript
// Access session store from express-session
const sessionMiddleware = session({
    secret: 'its-a-secret',
    resave: false,
    saveUninitialized: false,
    cookie: {
        secure: false,
        maxAge: 24 * 60 * 60 * 1000
    }
});

app.use(sessionMiddleware);

// Convert express session middleware to Socket.IO middleware
io.use((socket, next) => {
    // Get the session cookie from the handshake
    const cookie = socket.handshake.headers.cookie;
    
    if (!cookie) {
        return next(new Error('No session cookie found'));
    }
    
    // Parse the cookie to get the session ID
    const sessionID = cookie.split('connect.sid=')[1]?.split(';')[0];
    
    if (!sessionID) {
        return next(new Error('No session ID found'));
    }
    
    // Load the session from the session store
    sessionMiddleware.sessionStore.get(sessionID, (err, session) => {
        if (err) {
            return next(new Error('Session error'));
        }
        
        if (!session) {
            return next(new Error('Session not found'));
        }
        
        // Attach session to socket
        socket.sessionID = sessionID;
        socket.session = session;
        
        next(); // This will call the next part of the chain
    });
});

// Now you can access session in Socket.IO handlers
io.on('connection', (socket) => {
    // Access session data
    const userId = socket.session?.userId; // the '?' here tests to see if the session exists first
    const username = socket.session?.username;
    const isLoggedIn = socket.session?.isLoggedIn;
    
    console.log('Client connected:', socket.id);
    console.log('User:', username, 'ID:', userId);
    
    // Only allow authenticated users
    if (!isLoggedIn) {
        socket.emit('error', { message: 'Authentication required' });
        socket.disconnect();
        return;
    }
    
    // Listen for events
    socket.on('requestData', (data) => {
        // You can use session data here
        socket.emit('response', {
            success: true,
            message: `Hello ${username}!`,
            userId: userId,
            data: data
        });
    });
    
    socket.on('disconnect', () => {
        console.log('Client disconnected:', socket.id);
    });
});

const PORT = 3000;
server.listen(PORT, () => {
    console.log(`Server running on http://[domain name]`);
});
```

This method manually parses the session cookie and loads the session data from the session store. It gives you more control but is more complex.

## Complete Example: Authenticated Socket.IO Connection

Here's a complete example that shows authentication and session usage:

```javascript
// server.js
const express = require('express');
const http = require('http');
const session = require('express-session');
const { Server } = require('socket.io');

const app = express();
const server = http.createServer(app);

// Session configuration
const sessionMiddleware = session({
    secret: 'your-secret-key-change-this-in-production',
    resave: false,
    saveUninitialized: false,
    cookie: {
        secure: false,
        maxAge: 24 * 60 * 60 * 1000
    }
});

app.use(sessionMiddleware);
app.use(express.json());
app.use(express.static('public'));

// Login route
app.post('/api/login', (req, res) => {
    const { username, password } = req.body;
    
    // Your authentication logic here
    // For this example, we'll just set them (client will send post)
    
    req.session.userId = Math.floor(Math.random() * 1000);
    req.session.username = username;
    req.session.isLoggedIn = true;
    req.session.loginTime = new Date().toISOString();
    
    res.json({ 
        success: true, 
        message: 'Logged in',
        username: username
    });
});

// Logout route
app.post('/api/logout', (req, res) => {
    req.session.destroy((err) => {
        if (err) {
            return res.status(500).json({ error: 'Logout failed' });
        }
        res.json({ success: true, message: 'Logged out' });
    });
});

// Socket.IO setup
const io = new Server(server, {
    cors: {
        origin: "*",
        methods: ["GET", "POST"]
    }
});

// Share session with Socket.IO (official method)
io.engine.use(sessionMiddleware);

// Socket.IO connection handler
io.on('connection', (socket) => {
    const session = socket.request.session;
    
    // Check if user is authenticated
    if (!session.isLoggedIn) {
        socket.emit('error', { 
            message: 'Please log in to use Socket.IO features' 
        });
        socket.disconnect();
        return;
    }
    
    const username = session.username;
    const userId = session.userId;
    
    console.log(`User ${username} (ID: ${userId}) connected via Socket.IO`);
    
    // Send welcome message
    socket.emit('connected', {
        message: `Welcome ${username}!`,
        userId: userId,
        loginTime: session.loginTime
    });
    
    // Listen for authenticated requests
    socket.on('getUserInfo', () => {
        socket.emit('userInfo', {
            username: username,
            userId: userId,
            loginTime: session.loginTime
        });
    });
    
    socket.on('sendMessage', (data) => {
        // Broadcast message with user info
        io.emit('message', {
            username: username,
            userId: userId,
            message: data.message,
            timestamp: new Date().toISOString()
        });
    });
    
    socket.on('disconnect', () => {
        console.log(`User ${username} disconnected`);
    });
});

const PORT = 3000;
server.listen(PORT, () => {
    console.log(`Server running on http://[domain name]`);
});
```

## Client-Side Example

The client doesn't need any special changes - it automatically sends the session cookie:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Socket.IO with Sessions</title>
</head>
<body>
    <h1>Socket.IO with Session Authentication</h1>
    
    <div>
        <button id="loginBtn">Login</button>
        <button id="connectBtn">Connect to Socket.IO</button>
        <button id="getUserInfoBtn">Get User Info</button>
        <button id="sendMessageBtn">Send Message</button>
    </div>
    
    <div id="status">Not connected</div>
    <div id="messages"></div>

    <script src="https://cdn.socket.io/4.8.1/socket.io.min.js"></script>
    <script>
        let socket = null;
        
        // Login function
        document.getElementById('loginBtn').addEventListener('click', async () => {
            const response = await fetch('/api/login', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    username: 'testuser',
                    password: 'password123'
                })
            });
            
            const data = await response.json();
            document.getElementById('status').textContent = data.message;
        });
        
        // Connect to Socket.IO
        document.getElementById('connectBtn').addEventListener('click', () => {
            socket = io('http://[domain name]'); // CHANGE THIS
            
            socket.on('connect', () => {
                document.getElementById('status').textContent = 'Connected!';
            });
            
            socket.on('connected', (data) => {
                document.getElementById('status').textContent = data.message;
            });
            
            socket.on('error', (error) => {
                document.getElementById('status').textContent = 'Error: ' + error.message;
            });
            
            socket.on('userInfo', (data) => {
                document.getElementById('messages').innerHTML = 
                    '<pre>' + JSON.stringify(data, null, 2) + '</pre>';
            });
            
            socket.on('message', (data) => {
                const msgDiv = document.createElement('div');
                msgDiv.textContent = `${data.username}: ${data.message}`;
                document.getElementById('messages').appendChild(msgDiv);
            });
        });
        
        // Get user info
        document.getElementById('getUserInfoBtn').addEventListener('click', () => {
            if (socket) {
                socket.emit('getUserInfo');
            }
        });
        
        // Send message
        document.getElementById('sendMessageBtn').addEventListener('click', () => {
            if (socket) {
                socket.emit('sendMessage', {
                    message: 'Hello from client!'
                });
            }
        });
    </script>
</body>
</html>
```

## Key Points

- **Session cookie is automatically sent** - The browser sends the session cookie with the Socket.IO handshake
- **Session data is accessible** - You can read and modify session data in Socket.IO handlers
- **Authentication works** - You can check if users are logged in before allowing Socket.IO connections
- **User identification** - You know which user is connected via Socket.IO
- **Consistent state** - Session data is the same in Express routes and Socket.IO handlers

## Best Practices

1. **Always authenticate** - Check if users are logged in before allowing Socket.IO connections
2. **Validate permissions** - Check user permissions for specific Socket.IO events
3. **Handle session expiration** - Disconnect users if their session expires
4. **Use secure cookies** - Set `secure: true` when using HTTPS
5. **Store minimal data** - Only store necessary data in sessions


### Issue: CORS blocking session cookie

If your frontend domain is different from your backend domain, you need to configure CORS properly:

**Server side:**
```javascript
const cors = require('cors');

const corsOptions = {
    origin: ["http://[frontend-domain]"],
    credentials: true
};

// For Express
app.use(cors(corsOptions));

// For Socket.IO
const io = new Server(server, {
    cors: corsOptions
});
```

**Client side:**
```javascript
const socket = io('http://[domain name]', {
    withCredentials: true
});
```

**Solutions:**
1. Configure CORS properly in both Express and Socket.IO
2. Set `credentials: true` in CORS options
3. Set `withCredentials: true` in client connection options
4. Make sure the domain matches between Express and Socket.IO

---

**[Return to Socket.IO Chapter →](index.md)**

