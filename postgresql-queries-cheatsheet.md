# PostgreSQL Queries Cheatsheet

## Database Operations

### Create & Drop Database
```sql
-- Create database
CREATE DATABASE mydb;
CREATE DATABASE mydb WITH ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8';

-- Drop database
DROP DATABASE mydb;
DROP DATABASE IF EXISTS mydb;

-- List databases
\l
SELECT datname FROM pg_database;

-- Connect to database
\c mydb
```

## Table Operations

### Create Table
```sql
-- Basic table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    age INTEGER CHECK (age >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Table with foreign key
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    content TEXT,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create table from query
CREATE TABLE users_backup AS SELECT * FROM users;

-- Create temporary table
CREATE TEMP TABLE temp_users (id INTEGER, name VARCHAR(100));

-- Create table if not exists
CREATE TABLE IF NOT EXISTS users (id SERIAL PRIMARY KEY);
```

### Alter Table
```sql
-- Add column
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ALTER TABLE users ADD COLUMN IF NOT EXISTS phone VARCHAR(20);

-- Drop column
ALTER TABLE users DROP COLUMN phone;
ALTER TABLE users DROP COLUMN IF EXISTS phone;

-- Rename column
ALTER TABLE users RENAME COLUMN name TO full_name;

-- Change column type
ALTER TABLE users ALTER COLUMN age TYPE BIGINT;

-- Add constraint
ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE (email);
ALTER TABLE users ADD CONSTRAINT check_age CHECK (age >= 18);
ALTER TABLE posts ADD FOREIGN KEY (user_id) REFERENCES users(id);

-- Drop constraint
ALTER TABLE users DROP CONSTRAINT unique_email;

-- Rename table
ALTER TABLE users RENAME TO customers;
```

### Drop & Truncate Table
```sql
-- Drop table
DROP TABLE users;
DROP TABLE IF EXISTS users;
DROP TABLE users CASCADE;  -- Drop with dependent objects

-- Truncate table (delete all rows, faster than DELETE)
TRUNCATE TABLE users;
TRUNCATE TABLE users RESTART IDENTITY;  -- Reset auto-increment
TRUNCATE TABLE users CASCADE;  -- Truncate dependent tables
```

### Table Info
```sql
-- List tables
\dt
SELECT table_name FROM information_schema.tables WHERE table_schema = 'public';

-- Describe table
\d users
\d+ users  -- More details

-- Show table structure
SELECT column_name, data_type, character_maximum_length, is_nullable
FROM information_schema.columns
WHERE table_name = 'users';
```

## SELECT Queries

### Basic SELECT
```sql
-- Select all columns
SELECT * FROM users;

-- Select specific columns
SELECT name, email FROM users;

-- Select with alias
SELECT name AS full_name, email AS email_address FROM users;

-- Select distinct values
SELECT DISTINCT age FROM users;

-- Select with LIMIT
SELECT * FROM users LIMIT 10;

-- Select with OFFSET (pagination)
SELECT * FROM users LIMIT 10 OFFSET 20;
```

### WHERE Clause
```sql
-- Equal
SELECT * FROM users WHERE age = 25;

-- Not equal
SELECT * FROM users WHERE age != 25;
SELECT * FROM users WHERE age <> 25;

-- Comparison operators
SELECT * FROM users WHERE age > 18;
SELECT * FROM users WHERE age >= 18;
SELECT * FROM users WHERE age < 65;
SELECT * FROM users WHERE age <= 65;

-- BETWEEN
SELECT * FROM users WHERE age BETWEEN 18 AND 65;

-- IN
SELECT * FROM users WHERE id IN (1, 2, 3, 4, 5);
SELECT * FROM users WHERE name IN ('John', 'Jane', 'Bob');

-- NOT IN
SELECT * FROM users WHERE id NOT IN (1, 2, 3);

-- IS NULL / IS NOT NULL
SELECT * FROM users WHERE email IS NULL;
SELECT * FROM users WHERE email IS NOT NULL;

-- LIKE (pattern matching)
SELECT * FROM users WHERE name LIKE 'John%';  -- Starts with
SELECT * FROM users WHERE name LIKE '%Doe';   -- Ends with
SELECT * FROM users WHERE name LIKE '%oh%';   -- Contains

-- ILIKE (case-insensitive)
SELECT * FROM users WHERE name ILIKE '%john%';

-- NOT LIKE
SELECT * FROM users WHERE name NOT LIKE 'John%';

-- SIMILAR TO (regex-like)
SELECT * FROM users WHERE name SIMILAR TO '(John|Jane)%';

-- Regular expressions
SELECT * FROM users WHERE name ~ '^J';      -- Case sensitive
SELECT * FROM users WHERE name ~* '^j';     -- Case insensitive
SELECT * FROM users WHERE name !~ '^J';     -- Not match
SELECT * FROM users WHERE name !~* '^j';    -- Not match (case insensitive)
```

### AND, OR, NOT
```sql
-- AND
SELECT * FROM users WHERE age > 18 AND age < 65;

-- OR
SELECT * FROM users WHERE name = 'John' OR name = 'Jane';

-- NOT
SELECT * FROM users WHERE NOT age < 18;

-- Combine
SELECT * FROM users 
WHERE (name = 'John' OR name = 'Jane') AND age > 18;
```

### ORDER BY
```sql
-- Ascending (default)
SELECT * FROM users ORDER BY name;
SELECT * FROM users ORDER BY name ASC;

-- Descending
SELECT * FROM users ORDER BY age DESC;

-- Multiple columns
SELECT * FROM users ORDER BY age DESC, name ASC;

-- NULLS FIRST / NULLS LAST
SELECT * FROM users ORDER BY email NULLS FIRST;
SELECT * FROM users ORDER BY email NULLS LAST;

-- Order by expression
SELECT * FROM users ORDER BY LENGTH(name) DESC;
```

### GROUP BY
```sql
-- Basic grouping
SELECT age, COUNT(*) FROM users GROUP BY age;

-- Multiple columns
SELECT age, city, COUNT(*) FROM users GROUP BY age, city;

-- With aggregate functions
SELECT age, COUNT(*), AVG(salary), MAX(salary), MIN(salary)
FROM users GROUP BY age;

-- HAVING (filter groups)
SELECT age, COUNT(*) FROM users 
GROUP BY age 
HAVING COUNT(*) > 5;

-- GROUP BY with ORDER BY
SELECT age, COUNT(*) as count 
FROM users 
GROUP BY age 
ORDER BY count DESC;
```

## Aggregate Functions

```sql
-- COUNT
SELECT COUNT(*) FROM users;
SELECT COUNT(DISTINCT age) FROM users;

-- SUM
SELECT SUM(salary) FROM users;

-- AVG
SELECT AVG(age) FROM users;

-- MIN / MAX
SELECT MIN(age), MAX(age) FROM users;

-- STRING_AGG (concatenate strings)
SELECT STRING_AGG(name, ', ') FROM users;
SELECT STRING_AGG(name, ', ' ORDER BY name) FROM users;

-- ARRAY_AGG (aggregate to array)
SELECT ARRAY_AGG(name) FROM users;

-- JSON_AGG / JSONB_AGG
SELECT JSON_AGG(name) FROM users;
```

## JOIN Operations

```sql
-- INNER JOIN
SELECT users.name, posts.title
FROM users
INNER JOIN posts ON users.id = posts.user_id;

-- Shorthand (implicit join)
SELECT users.name, posts.title
FROM users, posts
WHERE users.id = posts.user_id;

-- LEFT JOIN (LEFT OUTER JOIN)
SELECT users.name, posts.title
FROM users
LEFT JOIN posts ON users.id = posts.user_id;

-- RIGHT JOIN (RIGHT OUTER JOIN)
SELECT users.name, posts.title
FROM users
RIGHT JOIN posts ON users.id = posts.user_id;

-- FULL JOIN (FULL OUTER JOIN)
SELECT users.name, posts.title
FROM users
FULL JOIN posts ON users.id = posts.user_id;

-- CROSS JOIN (Cartesian product)
SELECT users.name, posts.title
FROM users
CROSS JOIN posts;

-- SELF JOIN
SELECT u1.name AS user1, u2.name AS user2
FROM users u1
JOIN users u2 ON u1.id < u2.id;

-- Multiple joins
SELECT u.name, p.title, c.content
FROM users u
JOIN posts p ON u.id = p.user_id
JOIN comments c ON p.id = c.post_id;
```

## Subqueries

```sql
-- Subquery in WHERE
SELECT * FROM users 
WHERE age > (SELECT AVG(age) FROM users);

-- Subquery in SELECT
SELECT name, (SELECT COUNT(*) FROM posts WHERE user_id = users.id) AS post_count
FROM users;

-- Subquery in FROM
SELECT avg_age FROM (
    SELECT AVG(age) AS avg_age FROM users
) AS subquery;

-- EXISTS
SELECT * FROM users 
WHERE EXISTS (SELECT 1 FROM posts WHERE posts.user_id = users.id);

-- NOT EXISTS
SELECT * FROM users 
WHERE NOT EXISTS (SELECT 1 FROM posts WHERE posts.user_id = users.id);

-- IN with subquery
SELECT * FROM users 
WHERE id IN (SELECT user_id FROM posts WHERE title LIKE '%PostgreSQL%');

-- ANY / ALL
SELECT * FROM products WHERE price > ANY (SELECT price FROM products WHERE category = 'Electronics');
SELECT * FROM products WHERE price > ALL (SELECT price FROM products WHERE category = 'Electronics');
```

## UNION, INTERSECT, EXCEPT

```sql
-- UNION (combines results, removes duplicates)
SELECT name FROM users
UNION
SELECT name FROM customers;

-- UNION ALL (keeps duplicates)
SELECT name FROM users
UNION ALL
SELECT name FROM customers;

-- INTERSECT (common rows)
SELECT name FROM users
INTERSECT
SELECT name FROM customers;

-- EXCEPT (rows in first query but not in second)
SELECT name FROM users
EXCEPT
SELECT name FROM customers;
```

## INSERT Data

```sql
-- Insert single row
INSERT INTO users (name, email, age) 
VALUES ('John Doe', 'john@example.com', 30);

-- Insert multiple rows
INSERT INTO users (name, email, age) VALUES
    ('John Doe', 'john@example.com', 30),
    ('Jane Smith', 'jane@example.com', 25),
    ('Bob Johnson', 'bob@example.com', 35);

-- Insert and return inserted data
INSERT INTO users (name, email) 
VALUES ('John Doe', 'john@example.com')
RETURNING *;

INSERT INTO users (name, email) 
VALUES ('John Doe', 'john@example.com')
RETURNING id, name;

-- Insert from SELECT
INSERT INTO users_backup 
SELECT * FROM users WHERE age > 30;

-- Insert with default values
INSERT INTO users DEFAULT VALUES;

-- Insert or do nothing on conflict
INSERT INTO users (email, name) 
VALUES ('john@example.com', 'John')
ON CONFLICT (email) DO NOTHING;

-- Insert or update on conflict (UPSERT)
INSERT INTO users (email, name, age) 
VALUES ('john@example.com', 'John Doe', 30)
ON CONFLICT (email) 
DO UPDATE SET name = EXCLUDED.name, age = EXCLUDED.age;
```

## UPDATE Data

```sql
-- Update single column
UPDATE users SET age = 31 WHERE id = 1;

-- Update multiple columns
UPDATE users 
SET name = 'John Doe', age = 31 
WHERE id = 1;

-- Update all rows
UPDATE users SET active = true;

-- Update with expression
UPDATE products SET price = price * 1.1;

-- Update from another table
UPDATE users 
SET city = locations.city 
FROM locations 
WHERE users.location_id = locations.id;

-- Update with RETURNING
UPDATE users 
SET age = 31 
WHERE id = 1 
RETURNING *;

-- Conditional update
UPDATE users 
SET status = CASE 
    WHEN age < 18 THEN 'minor'
    WHEN age >= 65 THEN 'senior'
    ELSE 'adult'
END;
```

## DELETE Data

```sql
-- Delete specific rows
DELETE FROM users WHERE id = 1;

-- Delete all rows
DELETE FROM users;

-- Delete with RETURNING
DELETE FROM users 
WHERE id = 1 
RETURNING *;

-- Delete with subquery
DELETE FROM users 
WHERE id IN (SELECT user_id FROM inactive_users);

-- Delete from multiple tables using USING
DELETE FROM posts 
USING users 
WHERE posts.user_id = users.id AND users.status = 'deleted';
```

## Window Functions

```sql
-- ROW_NUMBER
SELECT name, age, 
    ROW_NUMBER() OVER (ORDER BY age DESC) AS row_num
FROM users;

-- RANK (with gaps)
SELECT name, salary, 
    RANK() OVER (ORDER BY salary DESC) AS rank
FROM users;

-- DENSE_RANK (no gaps)
SELECT name, salary, 
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM users;

-- PARTITION BY
SELECT name, department, salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM users;

-- Running total
SELECT name, salary,
    SUM(salary) OVER (ORDER BY id) AS running_total
FROM users;

-- Moving average
SELECT name, salary,
    AVG(salary) OVER (ORDER BY id ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg
FROM users;

-- LAG / LEAD (access previous/next row)
SELECT name, salary,
    LAG(salary) OVER (ORDER BY id) AS prev_salary,
    LEAD(salary) OVER (ORDER BY id) AS next_salary
FROM users;

-- FIRST_VALUE / LAST_VALUE
SELECT name, salary,
    FIRST_VALUE(salary) OVER (ORDER BY salary DESC) AS highest_salary,
    LAST_VALUE(salary) OVER (ORDER BY salary DESC) AS lowest_salary
FROM users;
```

## Common Table Expressions (CTE)

```sql
-- Basic CTE
WITH high_earners AS (
    SELECT * FROM users WHERE salary > 100000
)
SELECT * FROM high_earners WHERE age > 30;

-- Multiple CTEs
WITH 
    high_earners AS (SELECT * FROM users WHERE salary > 100000),
    young_users AS (SELECT * FROM users WHERE age < 30)
SELECT * FROM high_earners
UNION
SELECT * FROM young_users;

-- Recursive CTE (hierarchical data)
WITH RECURSIVE subordinates AS (
    -- Anchor member
    SELECT id, name, manager_id, 1 AS level
    FROM employees WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive member
    SELECT e.id, e.name, e.manager_id, s.level + 1
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates;
```

## CASE Expressions

```sql
-- Simple CASE
SELECT name, age,
    CASE age
        WHEN 18 THEN 'Just Adult'
        WHEN 21 THEN 'Legal Drinking Age'
        ELSE 'Other'
    END AS age_category
FROM users;

-- Searched CASE
SELECT name, age,
    CASE 
        WHEN age < 18 THEN 'Minor'
        WHEN age BETWEEN 18 AND 64 THEN 'Adult'
        WHEN age >= 65 THEN 'Senior'
        ELSE 'Unknown'
    END AS age_group
FROM users;

-- CASE in WHERE
SELECT * FROM users
WHERE CASE 
    WHEN age < 18 THEN status = 'active'
    ELSE status IN ('active', 'pending')
END;
```

## String Functions

```sql
-- Concatenation
SELECT 'Hello' || ' ' || 'World';
SELECT CONCAT('Hello', ' ', 'World');
SELECT CONCAT_WS(', ', 'John', 'Doe', 'Jr');  -- With separator

-- Length
SELECT LENGTH('Hello');
SELECT CHAR_LENGTH('Hello');

-- Case conversion
SELECT UPPER('hello');
SELECT LOWER('HELLO');
SELECT INITCAP('hello world');  -- Hello World

-- Substring
SELECT SUBSTRING('Hello World' FROM 1 FOR 5);  -- Hello
SELECT SUBSTRING('Hello World', 7);  -- World

-- Trim
SELECT TRIM('  Hello  ');
SELECT LTRIM('  Hello');
SELECT RTRIM('Hello  ');
SELECT TRIM(BOTH 'x' FROM 'xxxHelloxxx');

-- Replace
SELECT REPLACE('Hello World', 'World', 'PostgreSQL');

-- Position
SELECT POSITION('World' IN 'Hello World');  -- 7

-- Split
SELECT SPLIT_PART('one,two,three', ',', 2);  -- two
SELECT STRING_TO_ARRAY('one,two,three', ',');

-- Reverse
SELECT REVERSE('Hello');  -- olleH

-- Repeat
SELECT REPEAT('Ha', 3);  -- HaHaHa

-- Padding
SELECT LPAD('Hello', 10, '*');  -- *****Hello
SELECT RPAD('Hello', 10, '*');  -- Hello*****
```

## Date/Time Functions

```sql
-- Current date/time
SELECT CURRENT_DATE;
SELECT CURRENT_TIME;
SELECT CURRENT_TIMESTAMP;
SELECT NOW();

-- Date arithmetic
SELECT CURRENT_DATE + INTERVAL '1 day';
SELECT CURRENT_DATE - INTERVAL '1 week';
SELECT CURRENT_TIMESTAMP + INTERVAL '1 hour 30 minutes';

-- Extract parts
SELECT EXTRACT(YEAR FROM CURRENT_DATE);
SELECT EXTRACT(MONTH FROM CURRENT_DATE);
SELECT EXTRACT(DAY FROM CURRENT_DATE);
SELECT EXTRACT(HOUR FROM CURRENT_TIMESTAMP);

-- Date functions
SELECT DATE_PART('year', CURRENT_DATE);
SELECT DATE_TRUNC('month', CURRENT_TIMESTAMP);  -- First day of month
SELECT AGE(TIMESTAMP '2025-01-01', TIMESTAMP '2000-01-01');

-- Formatting
SELECT TO_CHAR(CURRENT_DATE, 'YYYY-MM-DD');
SELECT TO_CHAR(CURRENT_TIMESTAMP, 'HH24:MI:SS');
SELECT TO_CHAR(CURRENT_DATE, 'Day, DD Month YYYY');

-- Parsing
SELECT TO_DATE('2025-01-01', 'YYYY-MM-DD');
SELECT TO_TIMESTAMP('2025-01-01 14:30:00', 'YYYY-MM-DD HH24:MI:SS');
```

## Math Functions

```sql
-- Basic operations
SELECT ABS(-15);
SELECT CEIL(4.3);   -- 5
SELECT FLOOR(4.8);  -- 4
SELECT ROUND(4.567, 2);  -- 4.57
SELECT TRUNC(4.567, 2);  -- 4.56

-- Power and roots
SELECT POWER(2, 3);  -- 8
SELECT SQRT(16);     -- 4
SELECT CBRT(27);     -- 3

-- Trigonometric
SELECT SIN(1);
SELECT COS(1);
SELECT TAN(1);

-- Random
SELECT RANDOM();  -- 0 to 1
SELECT RANDOM() * 100;  -- 0 to 100
SELECT FLOOR(RANDOM() * 100 + 1);  -- 1 to 100

-- Logarithms
SELECT LN(10);   -- Natural log
SELECT LOG(100); -- Base 10 log
```

## JSON Operations

```sql
-- Create JSON
SELECT '{"name": "John", "age": 30}'::json;
SELECT '{"name": "John", "age": 30}'::jsonb;

-- Access JSON fields
SELECT data->>'name' FROM users;  -- Returns text
SELECT data->'age' FROM users;    -- Returns JSON

-- Nested access
SELECT data->'address'->>'city' FROM users;

-- JSON array elements
SELECT data->'tags'->0 FROM posts;

-- JSON functions
SELECT JSON_ARRAY_LENGTH('[1,2,3,4,5]');
SELECT JSONB_ARRAY_ELEMENTS('[1,2,3]');
SELECT JSONB_OBJECT_KEYS('{"a":1,"b":2}');

-- Build JSON
SELECT JSON_BUILD_OBJECT('name', name, 'age', age) FROM users;
SELECT JSON_BUILD_ARRAY(name, email) FROM users;

-- Aggregate to JSON
SELECT JSON_AGG(users) FROM users;
SELECT JSON_OBJECT_AGG(id, name) FROM users;

-- Query JSON
SELECT * FROM users WHERE data @> '{"country": "USA"}';
SELECT * FROM users WHERE data ? 'email';  -- Has key
SELECT * FROM users WHERE data ?| array['email', 'phone'];  -- Has any key
SELECT * FROM users WHERE data ?& array['email', 'phone'];  -- Has all keys
```

## Array Operations

```sql
-- Create array
SELECT ARRAY[1, 2, 3, 4, 5];
SELECT '{1,2,3,4,5}'::integer[];

-- Access elements
SELECT arr[1] FROM (SELECT ARRAY[1,2,3] AS arr) AS t;  -- 1 (1-indexed)

-- Array functions
SELECT ARRAY_LENGTH(ARRAY[1,2,3,4,5], 1);  -- 5
SELECT ARRAY_APPEND(ARRAY[1,2,3], 4);
SELECT ARRAY_PREPEND(0, ARRAY[1,2,3]);
SELECT ARRAY_CAT(ARRAY[1,2], ARRAY[3,4]);  -- Concatenate

-- Unnest array
SELECT UNNEST(ARRAY[1,2,3,4,5]);

-- Array operators
SELECT ARRAY[1,2,3] @> ARRAY[2];  -- Contains
SELECT ARRAY[1,2] <@ ARRAY[1,2,3];  -- Contained by
SELECT ARRAY[1,2,3] && ARRAY[2,3,4];  -- Overlap

-- ANY/ALL with arrays
SELECT * FROM products WHERE price > ANY(ARRAY[10, 20, 30]);
SELECT * FROM products WHERE price > ALL(ARRAY[10, 20, 30]);
```

## Indexes

```sql
-- Create index
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_name_age ON users(name, age);

-- Unique index
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Partial index
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- Expression index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- GIN index (for full-text search, JSONB, arrays)
CREATE INDEX idx_users_tags ON users USING GIN(tags);

-- GiST index (for geometric data)
CREATE INDEX idx_locations ON locations USING GIST(coordinates);

-- Drop index
DROP INDEX idx_users_email;
DROP INDEX IF EXISTS idx_users_email;

-- List indexes
\di
SELECT * FROM pg_indexes WHERE tablename = 'users';

-- Reindex
REINDEX TABLE users;
REINDEX INDEX idx_users_email;
```

## Constraints

```sql
-- Primary key
ALTER TABLE users ADD PRIMARY KEY (id);

-- Foreign key
ALTER TABLE posts ADD FOREIGN KEY (user_id) REFERENCES users(id);
ALTER TABLE posts ADD FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE;
ALTER TABLE posts ADD FOREIGN KEY (user_id) REFERENCES users(id) ON UPDATE CASCADE;

-- Unique constraint
ALTER TABLE users ADD UNIQUE (email);
ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE (email);

-- Check constraint
ALTER TABLE users ADD CHECK (age >= 0);
ALTER TABLE users ADD CONSTRAINT check_age CHECK (age >= 0 AND age <= 150);

-- Not null
ALTER TABLE users ALTER COLUMN email SET NOT NULL;
ALTER TABLE users ALTER COLUMN email DROP NOT NULL;

-- Default value
ALTER TABLE users ALTER COLUMN created_at SET DEFAULT CURRENT_TIMESTAMP;
ALTER TABLE users ALTER COLUMN status SET DEFAULT 'active';

-- Drop constraint
ALTER TABLE users DROP CONSTRAINT unique_email;
```

## Views

```sql
-- Create view
CREATE VIEW active_users AS
SELECT id, name, email FROM users WHERE active = true;

-- Materialized view (cached)
CREATE MATERIALIZED VIEW user_stats AS
SELECT age, COUNT(*) as count FROM users GROUP BY age;

-- Refresh materialized view
REFRESH MATERIALIZED VIEW user_stats;

-- Drop view
DROP VIEW active_users;
DROP MATERIALIZED VIEW user_stats;

-- Alter view
CREATE OR REPLACE VIEW active_users AS
SELECT id, name, email, age FROM users WHERE active = true;
```

## Transactions

```sql
-- Begin transaction
BEGIN;
-- or
START TRANSACTION;

-- Commit
COMMIT;

-- Rollback
ROLLBACK;

-- Savepoint
BEGIN;
INSERT INTO users (name) VALUES ('John');
SAVEPOINT sp1;
INSERT INTO users (name) VALUES ('Jane');
ROLLBACK TO SAVEPOINT sp1;  -- Rollback to savepoint
COMMIT;

-- Transaction isolation levels
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

## User & Permission Management

```sql
-- Create user
CREATE USER myuser WITH PASSWORD 'mypassword';

-- Create role
CREATE ROLE myrole;

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;
GRANT SELECT, INSERT, UPDATE ON users TO myuser;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO myuser;

-- Revoke privileges
REVOKE INSERT ON users FROM myuser;

-- Grant role
GRANT myrole TO myuser;

-- Drop user
DROP USER myuser;

-- List users
\du
SELECT * FROM pg_user;
```

## Backup & Restore

```bash
# Backup database (command line)
pg_dump mydb > backup.sql
pg_dump -U username -h localhost mydb > backup.sql

# Backup specific table
pg_dump -t users mydb > users_backup.sql

# Backup with custom format
pg_dump -Fc mydb > backup.dump

# Restore database
psql mydb < backup.sql
pg_restore -d mydb backup.dump

# Backup all databases
pg_dumpall > all_databases.sql
```

## Utility Commands

```sql
-- Show current database
SELECT current_database();

-- Show current user
SELECT current_user;

-- Show version
SELECT version();

-- Show table size
SELECT pg_size_pretty(pg_total_relation_size('users'));

-- Show database size
SELECT pg_size_pretty(pg_database_size('mydb'));

-- Vacuum (cleanup)
VACUUM users;
VACUUM FULL users;  -- More aggressive
VACUUM ANALYZE users;  -- Update statistics

-- Analyze (update statistics)
ANALYZE users;

-- Explain query plan
EXPLAIN SELECT * FROM users WHERE age > 30;
EXPLAIN ANALYZE SELECT * FROM users WHERE age > 30;  -- Execute and show actual time

-- Kill query
SELECT pg_cancel_backend(pid);
SELECT pg_terminate_backend(pid);

-- Show running queries
SELECT pid, query, state FROM pg_stat_activity WHERE state = 'active';
```

## Full-Text Search

```sql
-- Create tsvector column
ALTER TABLE posts ADD COLUMN tsv tsvector;

-- Update tsvector
UPDATE posts SET tsv = to_tsvector('english', title || ' ' || content);

-- Create index for full-text search
CREATE INDEX idx_posts_tsv ON posts USING GIN(tsv);

-- Search
SELECT * FROM posts WHERE tsv @@ to_tsquery('english', 'PostgreSQL & database');

-- Rank results
SELECT *, ts_rank(tsv, to_tsquery('PostgreSQL')) AS rank
FROM posts
WHERE tsv @@ to_tsquery('PostgreSQL')
ORDER BY rank DESC;

-- Highlight matches
SELECT ts_headline('english', content, to_tsquery('PostgreSQL')) FROM posts;
```

## Sequences

```sql
-- Create sequence
CREATE SEQUENCE user_id_seq START 1;

-- Next value
SELECT nextval('user_id_seq');

-- Current value
SELECT currval('user_id_seq');

-- Set value
SELECT setval('user_id_seq', 100);

-- Reset sequence
ALTER SEQUENCE user_id_seq RESTART WITH 1;

-- Drop sequence
DROP SEQUENCE user_id_seq;
```

## Performance Tips

```sql
-- Use EXPLAIN to analyze queries
EXPLAIN ANALYZE SELECT * FROM users WHERE age > 30;

-- Add indexes on frequently queried columns
CREATE INDEX ON users(email);

-- Use LIMIT for large result sets
SELECT * FROM users LIMIT 100;

-- Use prepared statements to prevent SQL injection
PREPARE get_user (int) AS SELECT * FROM users WHERE id = $1;
EXECUTE get_user(1);

-- Batch inserts instead of multiple single inserts
INSERT INTO users (name) VALUES ('John'), ('Jane'), ('Bob');

-- Use connection pooling (application level)

-- Monitor slow queries
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
```