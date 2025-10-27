---
layout: default
title: Building a REST API Example
nav_order: 2
parent: REST API Principles
nav_exclude: true
---

# Building a REST API Example

Now let's build a complete REST API example - a simple Task Management API. This will demonstrate all the REST principles we learned in the previous section.

## Project Overview

We'll create a task management API that allows users to:
- Create, read, update, and delete tasks
- Organize tasks by categories
- Mark tasks as complete or incomplete

## Project Structure

```
task-api/
├── package.json
├── server.js
├── routes/
│   ├── tasks.js
│   └── categories.js
└── data/
    └── tasks.json
```

## Step 1: Initialize the Project

Create a new directory and initialize the project:

```bash
mkdir task-api
cd task-api
npm init -y
```

Install required dependencies:

```bash
npm install express cors
npm install --save-dev nodemon
```

## Step 2: Create package.json Scripts

```json
{
  "name": "task-api",
  "version": "1.0.0",
  "description": "A RESTful Task Management API",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

## Step 3: Create the Main Server

**server.js**
```javascript
const express = require('express');
const cors = require('cors');
const taskRoutes = require('./routes/tasks');
const categoryRoutes = require('./routes/categories');

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Routes
app.use('/api/tasks', taskRoutes);
app.use('/api/categories', categoryRoutes);

// Root endpoint
app.get('/', (req, res) => {
  res.json({
    message: 'Task Management API',
    version: '1.0.0',
    endpoints: {
      tasks: '/api/tasks',
      categories: '/api/categories'
    }
  });
});

// 404 handler
app.use('*', (req, res) => {
  res.status(404).json({
    error: 'Endpoint not found',
    message: `The endpoint ${req.method} ${req.originalUrl} does not exist`
  });
});

// Error handler
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({
    error: 'Internal server error',
    message: 'Something went wrong on the server'
  });
});

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

## Step 4: Create Sample Data

**data/tasks.json**
```json
{
  "tasks": [
    {
      "id": 1,
      "title": "Learn REST API principles",
      "description": "Study REST architecture and build examples",
      "completed": false,
      "category": "learning",
      "createdAt": "2024-01-15T10:00:00Z",
      "updatedAt": "2024-01-15T10:00:00Z"
    },
    {
      "id": 2,
      "title": "Buy groceries",
      "description": "Milk, bread, eggs, and vegetables",
      "completed": true,
      "category": "personal",
      "createdAt": "2024-01-14T15:30:00Z",
      "updatedAt": "2024-01-15T09:00:00Z"
    },
    {
      "id": 3,
      "title": "Review project proposal",
      "description": "Go through the Q1 project proposal and provide feedback",
      "completed": false,
      "category": "work",
      "createdAt": "2024-01-13T14:20:00Z",
      "updatedAt": "2024-01-13T14:20:00Z"
    }
  ],
  "categories": [
    {
      "id": "work",
      "name": "Work",
      "description": "Professional tasks and projects"
    },
    {
      "id": "personal",
      "name": "Personal",
      "description": "Personal tasks and errands"
    },
    {
      "id": "learning",
      "name": "Learning",
      "description": "Educational and skill development tasks"
    }
  ]
}
```

## Step 5: Create Task Routes

**routes/tasks.js**
```javascript
const express = require('express');
const fs = require('fs').promises;
const path = require('path');

const router = express.Router();
const dataPath = path.join(__dirname, '../data/tasks.json');

// Helper function to read data
async function readData() {
  try {
    const data = await fs.readFile(dataPath, 'utf8');
    return JSON.parse(data);
  } catch (error) {
    console.error('Error reading data:', error);
    return { tasks: [], categories: [] };
  }
}

// Helper function to write data
async function writeData(data) {
  try {
    await fs.writeFile(dataPath, JSON.stringify(data, null, 2));
  } catch (error) {
    console.error('Error writing data:', error);
    throw error;
  }
}

// GET /api/tasks - Get all tasks
router.get('/', async (req, res) => {
  try {
    const data = await readData();
    const { completed, category } = req.query;
    
    let filteredTasks = data.tasks;
    
    // Filter by completion status
    if (completed !== undefined) {
      const isCompleted = completed === 'true';
      filteredTasks = filteredTasks.filter(task => task.completed === isCompleted);
    }
    
    // Filter by category
    if (category) {
      filteredTasks = filteredTasks.filter(task => task.category === category);
    }
    
    res.json({
      success: true,
      count: filteredTasks.length,
      data: filteredTasks
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Failed to fetch tasks'
    });
  }
});

// GET /api/tasks/:id - Get a specific task
router.get('/:id', async (req, res) => {
  try {
    const data = await readData();
    const taskId = parseInt(req.params.id);
    const task = data.tasks.find(t => t.id === taskId);
    
    if (!task) {
      return res.status(404).json({
        success: false,
        error: 'Task not found'
      });
    }
    
    res.json({
      success: true,
      data: task
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Failed to fetch task'
    });
  }
});

// POST /api/tasks - Create a new task
router.post('/', async (req, res) => {
  try {
    const data = await readData();
    const { title, description, category } = req.body;
    
    // Validation
    if (!title || !title.trim()) {
      return res.status(400).json({
        success: false,
        error: 'Title is required'
      });
    }
    
    // Check if category exists
    if (category && !data.categories.find(c => c.id === category)) {
      return res.status(400).json({
        success: false,
        error: 'Invalid category'
      });
    }
    
    // Create new task
    const newTask = {
      id: Math.max(...data.tasks.map(t => t.id), 0) + 1,
      title: title.trim(),
      description: description ? description.trim() : '',
      completed: false,
      category: category || 'personal',
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString()
    };
    
    data.tasks.push(newTask);
    await writeData(data);
    
    res.status(201).json({
      success: true,
      data: newTask
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Failed to create task'
    });
  }
});

// PUT /api/tasks/:id - Update an entire task
router.put('/:id', async (req, res) => {
  try {
    const data = await readData();
    const taskId = parseInt(req.params.id);
    const taskIndex = data.tasks.findIndex(t => t.id === taskId);
    
    if (taskIndex === -1) {
      return res.status(404).json({
        success: false,
        error: 'Task not found'
      });
    }
    
    const { title, description, completed, category } = req.body;
    
    // Validation
    if (!title || !title.trim()) {
      return res.status(400).json({
        success: false,
        error: 'Title is required'
      });
    }
    
    // Check if category exists
    if (category && !data.categories.find(c => c.id === category)) {
      return res.status(400).json({
        success: false,
        error: 'Invalid category'
      });
    }
    
    // Update task
    data.tasks[taskIndex] = {
      ...data.tasks[taskIndex],
      title: title.trim(),
      description: description ? description.trim() : '',
      completed: completed !== undefined ? completed : data.tasks[taskIndex].completed,
      category: category || data.tasks[taskIndex].category,
      updatedAt: new Date().toISOString()
    };
    
    await writeData(data);
    
    res.json({
      success: true,
      data: data.tasks[taskIndex]
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Failed to update task'
    });
  }
});

// PATCH /api/tasks/:id - Partially update a task
router.patch('/:id', async (req, res) => {
  try {
    const data = await readData();
    const taskId = parseInt(req.params.id);
    const taskIndex = data.tasks.findIndex(t => t.id === taskId);
    
    if (taskIndex === -1) {
      return res.status(404).json({
        success: false,
        error: 'Task not found'
      });
    }
    
    const updates = req.body;
    
    // Check if category exists
    if (updates.category && !data.categories.find(c => c.id === updates.category)) {
      return res.status(400).json({
        success: false,
        error: 'Invalid category'
      });
    }
    
    // Update only provided fields
    Object.keys(updates).forEach(key => {
      if (updates[key] !== undefined) {
        if (key === 'title' && updates[key].trim()) {
          data.tasks[taskIndex][key] = updates[key].trim();
        } else if (key !== 'title') {
          data.tasks[taskIndex][key] = updates[key];
        }
      }
    });
    
    data.tasks[taskIndex].updatedAt = new Date().toISOString();
    await writeData(data);
    
    res.json({
      success: true,
      data: data.tasks[taskIndex]
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Failed to update task'
    });
  }
});

// DELETE /api/tasks/:id - Delete a task
router.delete('/:id', async (req, res) => {
  try {
    const data = await readData();
    const taskId = parseInt(req.params.id);
    const taskIndex = data.tasks.findIndex(t => t.id === taskId);
    
    if (taskIndex === -1) {
      return res.status(404).json({
        success: false,
        error: 'Task not found'
      });
    }
    
    const deletedTask = data.tasks[taskIndex];
    data.tasks.splice(taskIndex, 1);
    await writeData(data);
    
    res.status(204).send();
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Failed to delete task'
    });
  }
});

module.exports = router;
```

## Step 6: Create Category Routes

**routes/categories.js**
```javascript
const express = require('express');
const fs = require('fs').promises;
const path = require('path');

const router = express.Router();
const dataPath = path.join(__dirname, '../data/tasks.json');

// Helper functions (same as in tasks.js)
async function readData() {
  try {
    const data = await fs.readFile(dataPath, 'utf8');
    return JSON.parse(data);
  } catch (error) {
    console.error('Error reading data:', error);
    return { tasks: [], categories: [] };
  }
}

async function writeData(data) {
  try {
    await fs.writeFile(dataPath, JSON.stringify(data, null, 2));
  } catch (error) {
    console.error('Error writing data:', error);
    throw error;
  }
}

// GET /api/categories - Get all categories
router.get('/', async (req, res) => {
  try {
    const data = await readData();
    res.json({
      success: true,
      count: data.categories.length,
      data: data.categories
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Failed to fetch categories'
    });
  }
});

// GET /api/categories/:id - Get a specific category
router.get('/:id', async (req, res) => {
  try {
    const data = await readData();
    const category = data.categories.find(c => c.id === req.params.id);
    
    if (!category) {
      return res.status(404).json({
        success: false,
        error: 'Category not found'
      });
    }
    
    res.json({
      success: true,
      data: category
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Failed to fetch category'
    });
  }
});

// POST /api/categories - Create a new category
router.post('/', async (req, res) => {
  try {
    const data = await readData();
    const { id, name, description } = req.body;
    
    // Validation
    if (!id || !name || !id.trim() || !name.trim()) {
      return res.status(400).json({
        success: false,
        error: 'ID and name are required'
      });
    }
    
    // Check if category already exists
    if (data.categories.find(c => c.id === id.trim())) {
      return res.status(409).json({
        success: false,
        error: 'Category already exists'
      });
    }
    
    // Create new category
    const newCategory = {
      id: id.trim(),
      name: name.trim(),
      description: description ? description.trim() : ''
    };
    
    data.categories.push(newCategory);
    await writeData(data);
    
    res.status(201).json({
      success: true,
      data: newCategory
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Failed to create category'
    });
  }
});

module.exports = router;
```

## Step 7: Test the API

Start the server:
```bash
npm run dev
```

### Test the API endpoints:

**1. Get all tasks:**
```bash
curl http://localhost:3000/api/tasks
```

**2. Get a specific task:**
```bash
curl http://localhost:3000/api/tasks/1
```

**3. Create a new task:**
```bash
curl -X POST http://localhost:3000/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Test REST API", "description": "Testing our new API", "category": "learning"}'
```

**4. Update a task:**
```bash
curl -X PUT http://localhost:3000/api/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"title": "Updated Task", "description": "This task was updated", "completed": true, "category": "work"}'
```

**5. Mark task as complete:**
```bash
curl -X PATCH http://localhost:3000/api/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"completed": true}'
```

**6. Delete a task:**
```bash
curl -X DELETE http://localhost:3000/api/tasks/1
```

**7. Get tasks by category:**
```bash
curl http://localhost:3000/api/tasks?category=work
```

**8. Get completed tasks:**
```bash
curl http://localhost:3000/api/tasks?completed=true
```

## REST Principles Demonstrated

This example demonstrates all the key REST principles:

1. **Stateless**: Each request contains all necessary information
2. **Resource-based URLs**: `/api/tasks`, `/api/categories`
3. **HTTP Methods**: GET, POST, PUT, PATCH, DELETE used appropriately
4. **Status Codes**: 200, 201, 204, 400, 404, 409, 500
5. **JSON Representation**: Data exchanged as JSON
6. **Uniform Interface**: Consistent response format

## API Response Format

All responses follow a consistent format:

```json
{
  "success": true,
  "data": { ... },
  "count": 5,
  "error": "Error message"
}
```

## Next Steps

- Add authentication and authorization
- Implement data validation middleware
- Add pagination for large datasets
- Include API documentation with Swagger
- Add rate limiting
- Implement proper error handling
- Add logging and monitoring

This REST API example provides a solid foundation for understanding and implementing RESTful web services!

---

**[Previous: What is REST?](what-is-rest.md)** | **[Next: Chapter 9](../9-next-chapter/index.md)**
