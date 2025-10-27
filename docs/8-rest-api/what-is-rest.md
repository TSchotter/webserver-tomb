---
layout: default
title: What is REST?
nav_order: 1
parent: REST API Principles
---

# What is REST?

REST stands for **Representational State Transfer**. It's an architectural style for designing web services that was introduced by Roy Fielding in his 2000 doctoral dissertation. REST has become the standard approach for building web APIs because it's simple, scalable, and works well with HTTP.

## What REST Stands For

Let's break down each word in REST:

### **RE**presentational
- Data is represented in a specific format (usually JSON, XML, or HTML)
- The same resource can be represented in different formats
- The client and server exchange representations of resources, not the resources themselves

#### What are "Representations"?

A **representation** is how data appears when transferred between client and server. Think of it like different languages describing the same object.

**Example: A User Resource**

The actual user data exists in the database, but when we send it over HTTP, we create a representation of that data:

```javascript
// The actual user data in database might look like:
// Database Table: users
// id: 123, name: "Luigi Mario", email: "luigi@example.com", created_at: "2024-01-15"

// JSON Representation (most common)
{
  "id": 123,
  "name": "Luigi Mario",
  "email": "luigi@example.com",
  "createdAt": "2024-01-15T10:00:00Z"
}

// XML Representation
<?xml version="1.0" encoding="UTF-8"?>
<user>
  <id>123</id>
  <name>Luigi Mario</name>
  <email>luigi@example.com</email>
  <createdAt>2024-01-15T10:00:00Z</createdAt>
</user>

// HTML Representation
<div class="user">
  <h2>Luigi Mario</h2>
  <p>Email: luigi@example.com</p>
  <p>Member since: January 15, 2024</p>
</div>
```

**Key Points:**
- The **resource** (user) is the same, but the **representation** (format) changes
- Different clients can request different representations of the same resource
- The server doesn't send the actual database record - it sends a representation of it

### **S**tate
- Each request from client to server must contain all the information needed to understand the request
- The server doesn't store any context about the client between requests
- This makes REST APIs stateless and scalable

### **T**ransfer
- Data is transferred between client and server using standard HTTP methods
- The transfer happens over HTTP protocol
- Resources are transferred as representations

## Core REST Principles

### 1. **Stateless**
Every request from client to server must contain all the information needed to understand the request. The server doesn't store any client context between requests.

```javascript
// Not RESTful - server stores state
app.get('/user', (req, res) => {
  const userId = req.session.userId; // Server remembers user
  res.json(getUser(userId));
});

// RESTful - client provides all needed info
app.get('/user/:id', (req, res) => {
  const userId = req.params.id; // Client provides user ID
  res.json(getUser(userId));
});
```

> You may be asking yourself right now, "Wait a second! Haven't we been storing information about the client on the server in the form of sessions! Doesn't that violate Statelessness?
>
> Yes, it frequently does.
>
> REST is a guideline, not to rigidly follow without deviation. If your app can't function without keeping track of the client's information then you might need to violate an aspect of REST. The premise is you follow REST unless you can't.


### 2. **Client-Server Architecture**
- Clear separation between client and server
- Client handles user interface and user experience
- Server handles data storage and business logic
- They can evolve independently

### 3. **Uniform Interface**
All resources are accessed using standard HTTP methods and consistent URL patterns:

- **GET** - Retrieve data
- **POST** - Create new resources
- **PUT** - Update entire resources
- **PATCH** - Partial updates
- **DELETE** - Remove resources

> Technically, you can handle most things in a single type of request. For example, you can have your POST requests handle updating and removing of resources (like database entries). But then you would be violating REST. The internet at large agreed that each of these methods are for specific types of requests to keep each of their jobs seperate.

### 4. **Resource-Based URLs**
Resources are identified by URLs that represent nouns, not verbs:

```javascript
// Good RESTful URLs
GET    /api/users          // Get all users
GET    /api/users/123      // Get user with ID 123
POST   /api/users          // Create a new user
PUT    /api/users/123      // Update user 123
DELETE /api/users/123      // Delete user 123

// Not RESTful - verbs in URLs
GET    /api/getUsers
POST   /api/createUser
POST   /api/updateUser
POST   /api/deleteUser
```

### 5. **HTTP Status Codes**
Use appropriate HTTP status codes to indicate the result of operations:

- **200 OK** - Successful GET, PUT, PATCH
- **201 Created** - Successful POST
- **204 No Content** - Successful DELETE
- **400 Bad Request** - Client error
- **401 Unauthorized** - Authentication required
- **403 Forbidden** - Access denied
- **404 Not Found** - Resource doesn't exist
- **418 I'm a teapot** - Inform client that the server is a teapot
- **500 Internal Server Error** - Server error

## REST vs Other Approaches

### REST vs SOAP
- **REST**: Simple, uses HTTP, JSON/XML, stateless
- **SOAP**: Complex, uses XML, stateful, more overhead

### REST vs GraphQL
- **REST**: Multiple endpoints, fixed data structure
- **GraphQL**: Single endpoint, flexible queries

## Why Use REST?

### Advantages:
1. **Simple** - Easy to understand and implement
2. **Scalable** - Stateless nature allows horizontal scaling
3. **Cacheable** - HTTP caching can be leveraged
4. **Language Agnostic** - Works with any programming language
5. **Standard** - Uses standard HTTP methods and status codes

### When to Use REST:
- Building web APIs
- Mobile app backends
- Microservices architecture
- Public APIs

## Example: RESTful Resource Design

Let's look at how a blog API might be designed:

```javascript
// Blog Posts Resource
GET    /api/posts              // Get all posts
GET    /api/posts/123          // Get post 123
POST   /api/posts              // Create new post
PUT    /api/posts/123          // Update post 123
DELETE /api/posts/123          // Delete post 123

// Comments Resource (nested under posts)
GET    /api/posts/123/comments     // Get comments for post 123
POST   /api/posts/123/comments     // Add comment to post 123
GET    /api/comments/456           // Get specific comment
PUT    /api/comments/456           // Update comment 456
DELETE /api/comments/456           // Delete comment 456

// Users Resource
GET    /api/users              // Get all users
GET    /api/users/789          // Get user 789
POST   /api/users              // Create new user
PUT    /api/users/789          // Update user 789
DELETE /api/users/789          // Delete user 789
```

## Key Takeaways

1. **REST** = Representational State Transfer
2. **Stateless** - Each request is independent
3. **Resource-based** - URLs represent nouns (resources)
4. **HTTP methods** - Use GET, POST, PUT, DELETE appropriately
5. **Status codes** - Communicate results clearly
6. **Simple and scalable** - Easy to understand and scale

---

**[Previous: REST API Principles](index.md)** | **[Next: Chapter 9](../9-next-chapter/index.md)**
