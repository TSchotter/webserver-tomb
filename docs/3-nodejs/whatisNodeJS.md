---
layout: default
title: What is NodeJS?
nav_order: 1
parent: NodeJS
---

# What is NodeJS?

[&laquo; Return to NodeJS Chapter](index.md)

## The JavaScript Runtime

NodeJS is a JavaScript runtime built on Chrome's V8 JavaScript engine. The original author, Ryan Dahl, developed it in 2009 but it has passed through several hands eventually ending up primarily maintained by the OpenJS Foundation. But what does using nodejs actually mean for a webserver developer?

### Other web server languages

You might use languages like:
- **PHP** - Processes requests one at a time
- **Python** - Synchronous by default
- **Java** - Thread-based concurrency
- **C#** - Thread-based concurrency

These languages typically handle one request per thread, which can become expensive when you have many concurrent users.

### NodeJS: Event-Driven and Non-Blocking

NodeJS takes a different approach:

```javascript
{% raw %}
// Traditional blocking approach (conceptual)
function handleRequest() {
    var data = readFileSync('large-file.txt'); // Blocks here
    var processed = processData(data);         // Blocks here
    return processed;                          // Finally returns
}

// NodeJS non-blocking approach
function handleRequest() {
    readFile('large-file.txt', function(err, data) {
        if (err) throw err;
        var processed = processData(data);
    });
    // Continue with other operations while file is being read
}
```
The first handle request function is forced to wait for each line before going to the next line.

But in the second example, the request to read a file doesn't stop the rest of the handle request from being processed.

## The Event Loop

NodeJS uses an event loop to handle asynchronous operations. This doesn't mean that NodeJS is multi-threaded, it's still operating on a single thread. But now server can do other things while it's waiting.


```javascript
console.log('Start');

setTimeout(() => {
    console.log('Timer 1');
}, 0);

setTimeout(() => {
    console.log('Timer 2');
}, 0);

console.log('End');
```

The above example will show:

```javascript
Start
End
Timer 1
Timer 2
```

## Why NodeJS for Web Servers?

### Advantages

1. **Fast I/O**: Excellent for handling many concurrent connections
2. **JavaScript Everywhere**: Same language on frontend and backend
3. **Rich Ecosystem**: NPM provides thousands of packages

### When to Use NodeJS

- **APIs and Web Services**: RESTful APIs, microservices
- **Real-time Applications**: Chat, gaming, live data feeds
- **Data Streaming**: Processing large amounts of data
- **Single Page Applications**: Serving static files and API endpoints

### When NOT to Use NodeJS

- **CPU-Intensive Tasks**: Image processing, complex calculations
- **Traditional Web Apps**: Simple CRUD applications might be better with PHP/Python
- **Legacy Systems**: When you need specific language features



## Getting Started

To check if NodeJS is installed on your server (this should be the case on both the toastcode.net and umainecos.org servers):

```bash
node --version
npm --version
```

If not installed, you can install it using:

```bash
# On Ubuntu/Debian
sudo apt update
sudo apt install nodejs npm
```

## Next Steps

That's enough of a backdrop, let's get to creating our first nodejs web application

**[Next: Building Your First Server →](firstServer.md)**
