---
layout: default
title: Common MySQL Commands
nav_order: 2
parent: Database Fundamentals
---

# Common MySQL Commands

Now that you understand the fundamental concepts of databases, it's time to learn how to actually work with them using SQL commands. In this section, we'll explore the most common MySQL commands you'll use when building web applications.

We'll use a library management system as our example throughout this chapter, with three main tables: **Book**, **Author**, and **Genre**. This will help you see how these commands work in a realistic scenario.

## Our Library Database Example

Before we dive into the commands, let's establish our example database structure:

- **Author Table**: Stores information about book authors
- **Genre Table**: Stores different book genres/categories  
- **Book Table**: Stores information about books, with references to authors and genres

This structure demonstrates how tables relate to each other through foreign keys, which is essential for understanding how to properly insert and manage data.

## SQL Comments

Before we dive into the commands, it's important to know how to add comments to your SQL code. Comments help you document your code and make it easier to understand.

### Comment Syntax

SQL supports two types of comments:

#### Single-Line Comments
Use `--` (two hyphens) to create a single-line comment:

```sql
-- This is a single-line comment
SELECT * FROM Author;
```

#### Multi-Line Comments
Use `/* */` to create multi-line comments:

```sql
/* This is a multi-line comment
   that can span several lines
   and is useful for longer explanations */
SELECT * FROM Book;
```

### Using Comments in Practice

Comments are especially useful for:
- **Explaining complex queries**
- **Documenting table structures**
- **Adding notes for future reference**
- **Temporarily disabling code during testing**

```sql
-- Find all books published after 1990
SELECT * FROM Book
WHERE publication_year > 1990;

/* This query finds books by British authors
   We'll use LIKE to search for 'British' in the bio field */
SELECT * FROM Book
WHERE author_id IN (
    SELECT author_id 
    FROM Author 
    WHERE bio LIKE '%British%'
);
```

## Basic SQL Syntax

Now that you know how to add comments, let's understand the basic structure of SQL statements.

### Command Termination

**Every SQL command must end with a semicolon (`;`)**. The semicolon tells the database server that the command is complete and ready to be executed.

```sql
-- This command is complete and will execute
SELECT * FROM Author;

-- This command is NOT complete - missing semicolon
SELECT * FROM Author
```

### Line Breaks and Formatting

SQL commands can be written on a single line or spread across multiple lines. The choice is entirely up to the programmer and is based on readability preferences.

#### Single Line Format
```sql
SELECT first_name, last_name FROM Author WHERE birth_year > 1900;
```

#### Multi-Line Format
```sql
SELECT first_name, last_name 
FROM Author 
WHERE birth_year > 1900;
```

#### Multi-Line with Indentation
```sql
SELECT 
    first_name, 
    last_name 
FROM Author 
WHERE birth_year > 1900;
```

### Key Points

- **Semicolon is required**: Every command must end with `;`
- **Line breaks are optional**: You can format commands however you prefer
- **Readability matters**: Choose formatting that makes your code easy to read and understand
- **Consistency**: Use the same formatting style throughout your project

### Common Formatting Practices

While formatting is flexible, there are some widely accepted practices that make SQL code more readable:

#### Keywords in UPPERCASE
```sql
-- Common practice: SQL keywords in uppercase
SELECT first_name, last_name 
FROM Author 
WHERE birth_year > 1900;
```

#### Indentation for Clarity
```sql
-- Indent clauses for better readability
SELECT 
    first_name, 
    last_name,
    birth_year
FROM Author 
WHERE birth_year > 1900
    AND bio LIKE '%British%';
```

#### One Clause Per Line
```sql
-- Each major clause on its own line
SELECT first_name, last_name
FROM Author
WHERE birth_year > 1900
ORDER BY last_name;
```

#### Align Similar Elements
```sql
-- Align column names for easier reading
SELECT 
    first_name,
    last_name,
    birth_year,
    bio
FROM Author
WHERE birth_year > 1900;
```

These practices aren't required, but they make your code more professional and easier for others (and yourself) to read and maintain.

## Methods SQL commands will be run

### Directly

You can log into the sql database and run each command one at a time. Think of it like starting up the python environmnet and running python commands

### Loading a SQL file

You can tell the SQL database server to run all the commands in a file. It will attempt to execute each command in the file from top to bottom. This is extremely useful when first setting up your database with tables, that way you have a sql file that will be used to quickly set up the framework of your data structure.

### Sending commands to the database

This is the primary way that we will interact with the database with our webserver. The webserver will send SQL commands to the database server and the database server will respond to those commands. These responses will vary depending on what commands we send.



## CREATE TABLE

The `CREATE TABLE` command is used to create new tables in your database. This is where you define the structure of your data, including column names, data types, and constraints.

### Basic Syntax

```sql
CREATE TABLE table_name (
    column1 datatype constraints,
    column2 datatype constraints,
    column3 datatype constraints
);
```

### Data Types in MySQL

MySQL supports many data types. Here are the most common ones you'll use:

#### Text Data Types
- **VARCHAR(n)**: Variable-length string up to n characters
- **TEXT**: Large text data (up to 65,535 characters)
- **CHAR(n)**: Fixed-length string of exactly n characters

#### Numeric Data Types
- **INT**: Integer (whole numbers)
- **DECIMAL(p,s)**: Decimal numbers with p total digits and s decimal places
- **FLOAT**: Floating-point numbers

#### Date and Time Data Types
- **DATE**: Date in YYYY-MM-DD format
- **DATETIME**: Date and time in YYYY-MM-DD HH:MM:SS format
- **TIMESTAMP**: Automatic timestamp that updates when record is modified

### Keys and Constraints

#### Primary Key
A primary key uniquely identifies each row in a table. It cannot be NULL and must be unique.

```sql
PRIMARY KEY (column_name)
```

#### Foreign Key
A foreign key creates a relationship between tables by referencing the primary key of another table.

```sql
FOREIGN KEY (column_name) REFERENCES other_table(primary_key_column)
```

#### Other Common Constraints
- **NOT NULL**: Column cannot be empty
- **UNIQUE**: All values in column must be unique
- **AUTO_INCREMENT**: Automatically generates unique numbers (usually for primary keys)

### Creating Our Library Tables

Let's create our three example tables:

#### Author Table
```sql
CREATE TABLE Author (
    author_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    birth_year INT,
    bio TEXT
);
```

**Explanation**:
- `author_id` is our primary key that auto-increments
- `first_name` and `last_name` are required (NOT NULL) and limited to 50 characters
- `birth_year` and `bio` are optional

#### Genre Table
```sql
CREATE TABLE Genre (
    genre_id INT AUTO_INCREMENT PRIMARY KEY,
    genre_name VARCHAR(50) NOT NULL UNIQUE,
    description TEXT
);
```

**Explanation**:
- `genre_id` is our primary key
- `genre_name` must be unique (no duplicate genre names)
- `description` can store longer text

#### Book Table
```sql
CREATE TABLE Book (
    book_id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    isbn VARCHAR(13) UNIQUE,
    publication_year INT,
    author_id INT NOT NULL,
    genre_id INT NOT NULL,
    FOREIGN KEY (author_id) REFERENCES Author(author_id),
    FOREIGN KEY (genre_id) REFERENCES Genre(genre_id)
);
```

**Explanation**:
- `book_id` is our primary key
- `title` is required and can be up to 200 characters
- `isbn` should be unique (each book has a unique ISBN)
- `author_id` and `genre_id` are foreign keys that reference the Author and Genre tables
- The foreign key constraints ensure data integrity

### Important Notes About Foreign Keys

When creating tables with foreign keys, you must create the referenced tables first. In our example:
1. First create `Author` and `Genre` tables
2. Then create `Book` table (which references both)

This order is crucial because MySQL needs to verify that the referenced tables exist before creating the foreign key relationships.

## DROP TABLE

The `DROP TABLE` command removes entire tables from your database. This permanently deletes the table structure and all data within it.

### Basic Syntax

```sql
DROP TABLE table_name;
```

### Important Considerations

**Warning**: DROP TABLE permanently deletes the table and all its data. This action cannot be undone!

### Dropping Tables in Our Library Example

#### Dropping a Single Table
```sql
-- Drop the Book table (this will fail if other tables reference it)
DROP TABLE Book;
```

#### Dropping Multiple Tables
```sql
-- Drop multiple tables at once
DROP TABLE Book, Author, Genre;
```

### Foreign Key Constraints and DROP TABLE

When dropping tables with foreign key relationships, you must consider the order:

#### Safe Dropping Order
```sql
-- First drop tables that reference other tables (dependent tables)
DROP TABLE Book;

-- Then drop the referenced tables
DROP TABLE Author;
DROP TABLE Genre;
```

#### Dropping All Tables at Once
```sql
-- Drop all tables in the correct order
DROP TABLE Book, Author, Genre;
```

### When to Use DROP TABLE

- **Development and Testing**: When you need to recreate tables with different structures
- **Database Cleanup**: Removing tables that are no longer needed
- **Schema Changes**: When you need to completely restructure your database

### Alternative: TRUNCATE TABLE

If you want to remove all data but keep the table structure, use `TRUNCATE TABLE` instead:

```sql
-- Remove all data from Book table but keep the table structure
TRUNCATE TABLE Book;
```

**Key Differences**:
- `DROP TABLE`: Removes the table completely (structure + data)
- `TRUNCATE TABLE`: Removes all data but keeps the table structure
- `DELETE FROM table_name`: Removes specific rows based on WHERE conditions

## INSERT INTO

The `INSERT INTO` command adds new records (rows) to your tables. This is how you populate your database with data.

### Basic Syntax

```sql
INSERT INTO table_name (column1, column2, column3)
VALUES (value1, value2, value3);
```

### Inserting Data into Our Library Tables

Let's add some sample data to our tables:

#### Inserting Authors
```sql
INSERT INTO Author (first_name, last_name, birth_year, bio)
VALUES ('J.K.', 'Rowling', 1965, 'British author best known for the Harry Potter series');

INSERT INTO Author (first_name, last_name, birth_year, bio)
VALUES ('George', 'Orwell', 1903, 'English novelist and essayist, known for dystopian fiction');

INSERT INTO Author (first_name, last_name, birth_year, bio)
VALUES ('Harper', 'Lee', 1926, 'American novelist best known for To Kill a Mockingbird');
```

#### Inserting Genres
```sql
INSERT INTO Genre (genre_name, description)
VALUES ('Fantasy', 'Books with magical elements and imaginary worlds');

INSERT INTO Genre (genre_name, description)
VALUES ('Dystopian Fiction', 'Stories set in oppressive, futuristic societies');

INSERT INTO Genre (genre_name, description)
VALUES ('Literary Fiction', 'Character-driven stories with literary merit');
```

#### Inserting Books
```sql
INSERT INTO Book (title, isbn, publication_year, author_id, genre_id)
VALUES ('Harry Potter and the Philosopher''s Stone', '9780747532699', 1997, 1, 1);

INSERT INTO Book (title, isbn, publication_year, author_id, genre_id)
VALUES ('1984', '9780451524935', 1949, 2, 2);

INSERT INTO Book (title, isbn, publication_year, author_id, genre_id)
VALUES ('To Kill a Mockingbird', '9780061120084', 1960, 3, 3);
```

### Critical Rule: Foreign Key Requirements

**Before inserting a book, the referenced author and genre must already exist in their respective tables.**

This is why we:
1. First inserted authors (getting author_id values of 1, 2, 3)
2. Then inserted genres (getting genre_id values of 1, 2, 3)  
3. Finally inserted books using those existing IDs

If you try to insert a book with `author_id = 5` but no author with ID 5 exists, MySQL will reject the insert and show an error.

### Inserting Multiple Records

You can insert multiple records at once:

```sql
INSERT INTO Author (first_name, last_name, birth_year, bio)
VALUES 
    ('Agatha', 'Christie', 1890, 'British mystery writer, creator of Hercule Poirot'),
    ('Stephen', 'King', 1947, 'American horror and suspense novelist'),
    ('Toni', 'Morrison', 1931, 'American novelist and Nobel Prize winner');
```

### Inserting with Default Values

If a column has a default value or allows NULL, you can omit it from the INSERT statement:

```sql
-- This will work if birth_year allows NULL
INSERT INTO Author (first_name, last_name, bio)
VALUES ('Ernest', 'Hemingway', 'American novelist and journalist, Nobel Prize winner');
```

## UPDATE

The `UPDATE` command modifies existing records in your tables. It's essential for keeping your data current and accurate.

### Basic Syntax

```sql
UPDATE table_name
SET column1 = new_value1, column2 = new_value2
WHERE condition;
```

### Important: Always Use WHERE

**Warning**: If you don't include a WHERE clause, UPDATE will modify ALL rows in the table. This is usually not what you want!

### Updating Our Library Data

#### Updating Author Information
```sql
-- Update J.K. Rowling's birth year (correcting a mistake)
UPDATE Author
SET birth_year = 1965
WHERE author_id = 1;

-- Update multiple columns for an author
UPDATE Author
SET first_name = 'Joanne', last_name = 'Rowling'
WHERE author_id = 1;
```

#### Updating Book Information
```sql
-- Update a book's publication year
UPDATE Book
SET publication_year = 1998
WHERE book_id = 1;

-- Update multiple columns
UPDATE Book
SET title = 'Harry Potter and the Sorcerer''s Stone', publication_year = 1998
WHERE book_id = 1;
```

#### Updating Based on Conditions
```sql
-- Update all books published before 1950 to have a note
UPDATE Book
SET title = CONCAT(title, ' (Classic)')
WHERE publication_year < 1950;
```


## SELECT

The `SELECT` command retrieves data from your tables. This is how you query your database to find specific information.

### Basic Syntax

```sql
SELECT column1, column2, column3
FROM table_name
WHERE condition;
```

### Selecting All Columns

```sql
-- Get all data from a table
SELECT * FROM Author;

-- Get all books
SELECT * FROM Book;
```

### Selecting Specific Columns

```sql
-- Get only author names
SELECT first_name, last_name FROM Author;

-- Get book titles and publication years
SELECT title, publication_year FROM Book;
```

### Using WHERE for Conditions

The WHERE clause lets you filter results based on specific criteria:

#### Simple Conditions
```sql
-- Find all authors with 'British' in their bio
SELECT * FROM Author
WHERE bio LIKE '%British%';

-- Find books published after 1990
SELECT * FROM Book
WHERE publication_year > 1990;

-- Find books by a specific author
SELECT * FROM Book
WHERE author_id = 1;
```

#### Multiple Conditions with AND/OR
```sql
-- Find authors with 'British' in their bio born after 1900
SELECT * FROM Author
WHERE bio LIKE '%British%' AND birth_year > 1900;

-- Find books published in 1997 OR 1998
SELECT * FROM Book
WHERE publication_year = 1997 OR publication_year = 1998;

-- Find books by author 1 OR author 2
SELECT * FROM Book
WHERE author_id = 1 OR author_id = 2;
```

#### Using IN for Multiple Values
```sql
-- Find books by multiple specific authors
SELECT * FROM Book
WHERE author_id IN (1, 2, 3);

-- Find books published in specific years
SELECT * FROM Book
WHERE publication_year IN (1949, 1960, 1997);
```

#### Pattern Matching with LIKE
The LIKE operator allows you to search for patterns in text data using wildcards:

- **`%`** (percent sign): Matches any sequence of characters (including zero characters)
- **`_`** (underscore): Matches exactly one character

```sql
-- Find authors with last names starting with 'R'
-- The % matches any characters after 'R'
SELECT * FROM Author
WHERE last_name LIKE 'R%';

-- Find books with 'Harry' in the title
-- The % before and after matches any characters before and after 'Harry'
SELECT * FROM Book
WHERE title LIKE '%Harry%';

-- Find books with titles ending in 'Stone'
-- The % before matches any characters before 'Stone'
SELECT * FROM Book
WHERE title LIKE '%Stone';
```

#### NULL Values
```sql
-- Find authors without a birth year recorded
SELECT * FROM Author
WHERE birth_year IS NULL;

-- Find authors with birth year recorded
SELECT * FROM Author
WHERE birth_year IS NOT NULL;
```

### Sorting Results with ORDER BY

```sql
-- Sort authors by last name alphabetically
SELECT * FROM Author
ORDER BY last_name;

-- Sort books by publication year (newest first)
SELECT * FROM Book
ORDER BY publication_year DESC;

-- Sort by multiple columns
SELECT * FROM Author
ORDER BY last_name, first_name;
```

### Limiting Results

```sql
-- Get only the first 5 authors
SELECT * FROM Author
LIMIT 5;

-- Get the 3 most recently published books
SELECT * FROM Book
ORDER BY publication_year DESC
LIMIT 3;
```

## DELETE

The `DELETE` command removes records from your tables. Like UPDATE, it's crucial to use WHERE clauses to avoid deleting all your data.

### Basic Syntax

```sql
DELETE FROM table_name
WHERE condition;
```

### Important: Always Use WHERE

**Warning**: Without a WHERE clause, DELETE will remove ALL rows from the table!

### Deleting from Our Library Database

#### Deleting Specific Records
```sql
-- Delete a specific book
DELETE FROM Book
WHERE book_id = 3;

-- Delete all books by a specific author
DELETE FROM Book
WHERE author_id = 2;

-- Delete books published before 1950
DELETE FROM Book
WHERE publication_year < 1950;
```

#### Deleting Authors (with Considerations)
```sql
-- Delete a specific author
DELETE FROM Author
WHERE author_id = 3;
```

**Important Note**: If you try to delete an author who has books in the Book table, MySQL will reject the deletion because of the foreign key constraint. You must first delete all books by that author, then delete the author.

#### Safe Deletion Order
```sql
-- First, delete all books by the author
DELETE FROM Book
WHERE author_id = 3;

-- Then, delete the author
DELETE FROM Author
WHERE author_id = 3;
```

### Deleting All Records (Use with Caution)

```sql
-- Delete all books (be very careful!)
DELETE FROM Book;

-- Delete all authors (be very careful!)
DELETE FROM Author;
```

## Best Practices and Common Mistakes

### 1. Always Use WHERE Clauses
- **UPDATE** and **DELETE** without WHERE can affect all rows
- Double-check your WHERE conditions before running these commands

### 2. Test with SELECT First
Before running UPDATE or DELETE, test your WHERE condition with SELECT:

```sql
-- Test what you're about to update
SELECT * FROM Book WHERE author_id = 2;

-- If the results look correct, then update
UPDATE Book SET publication_year = 1950 WHERE author_id = 2;
```

### 3. Foreign Key Relationships Matter
- Always insert referenced data first (authors before books)
- Delete dependent data first (books before authors)
- Understand your table relationships before making changes

### 4. Use Transactions for Multiple Operations
When making multiple related changes, use transactions to ensure data consistency:

```sql
START TRANSACTION;
DELETE FROM Book WHERE author_id = 2;
DELETE FROM Author WHERE author_id = 2;
COMMIT;
```

### 5. Backup Before Major Changes
Always backup your data before running DELETE or UPDATE operations that affect many records.

## Summary

These five commands form the foundation of database operations:

- **CREATE TABLE**: Define your data structure with proper data types and relationships
- **INSERT INTO**: Add new data, remembering that foreign keys must reference existing records
- **UPDATE**: Modify existing data, always using WHERE clauses to target specific records
- **SELECT**: Retrieve data with filtering, sorting, and limiting capabilities
- **DELETE**: Remove data, being careful about foreign key constraints and always using WHERE

Understanding these commands and their proper usage is essential for building web applications that can effectively manage data. In the next section, we'll explore how to combine data from multiple tables using JOINs, which will allow you to create more powerful and useful queries.

---

**[← Back to Database Basics](databaseBasics.md)** | **[Next: Database Joins →]()**
