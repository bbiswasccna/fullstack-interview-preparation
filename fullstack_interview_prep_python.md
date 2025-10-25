# Python Senior Backend Developer Interview Preparation Guide
## Complete Guide for Senior-Level Python Backend Interviews

---

## 1. PYTHON CORE CONCEPTS

### Q1: Explain Python's GIL (Global Interpreter Lock) and how to work around it.
**Answer:**

**What is GIL:**
- A mutex that protects access to Python objects
- Prevents multiple threads from executing Python bytecode simultaneously
- Only one thread can execute Python code at a time (even on multi-core systems)

**Why it exists:**
- Simplifies memory management (reference counting)
- Makes CPython implementation simpler
- Protects non-thread-safe C extensions

**Performance Impact:**
```python
import threading
import time

def cpu_bound_task():
    count = 0
    for i in range(10_000_000):
        count += i
    return count

# Multi-threading (GIL limits performance)
start = time.time()
threads = []
for _ in range(4):
    t = threading.Thread(target=cpu_bound_task)
    threads.append(t)
    t.start()

for t in threads:
    t.join()
print(f"Threading time: {time.time() - start:.2f}s")  # ~2.5s

# Multi-processing (bypasses GIL)
from multiprocessing import Process

start = time.time()
processes = []
for _ in range(4):
    p = Process(target=cpu_bound_task)
    processes.append(p)
    p.start()

for p in processes:
    p.join()
print(f"Multiprocessing time: {time.time() - start:.2f}s")  # ~0.7s
```

**Workarounds:**

**1. Multiprocessing:**
```python
from multiprocessing import Pool

def process_data(chunk):
    return sum(x * x for x in chunk)

if __name__ == '__main__':
    data = range(10_000_000)
    chunks = [data[i:i+2500000] for i in range(0, len(data), 2500000)]
    
    with Pool(processes=4) as pool:
        results = pool.map(process_data, chunks)
    
    print(f"Total: {sum(results)}")
```

**2. AsyncIO for I/O-bound tasks:**
```python
import asyncio
import aiohttp

async def fetch_url(session, url):
    async with session.get(url) as response:
        return await response.text()

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
    return results

# GIL doesn't matter for I/O operations
```

**3. Use C extensions:**
```python
import numpy as np  # Releases GIL for numerical operations

# NumPy operations release GIL
large_array = np.random.rand(10_000_000)
result = np.sum(large_array)  # Efficient, GIL released
```

**4. Alternative Python Implementations:**
- **Jython:** No GIL (runs on JVM)
- **IronPython:** No GIL (runs on .NET)
- **PyPy:** Has GIL but JIT compilation makes it faster

**When GIL doesn't matter:**
- I/O-bound applications (web servers, APIs)
- Single-threaded applications
- Tasks that release GIL (NumPy, Pandas operations)

---

### Q2: Explain decorators, metaclasses, and when to use them.
**Answer:**

**Decorators:**

**Basic Function Decorator:**
```python
import time
from functools import wraps

def timing_decorator(func):
    @wraps(func)  # Preserves original function metadata
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} took {end - start:.2f}s")
        return result
    return wrapper

@timing_decorator
def slow_function():
    time.sleep(1)
    return "Done"
```

**Decorator with Arguments:**
```python
def retry(max_attempts=3, delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            attempts = 0
            while attempts < max_attempts:
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    attempts += 1
                    if attempts >= max_attempts:
                        raise
                    print(f"Attempt {attempts} failed: {e}. Retrying in {delay}s...")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry(max_attempts=3, delay=2)
def unstable_api_call():
    # Simulated API call that might fail
    import random
    if random.random() < 0.7:
        raise Exception("API Error")
    return "Success"
```

**Class Decorator:**
```python
def singleton(cls):
    instances = {}
    @wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

@singleton
class DatabaseConnection:
    def __init__(self):
        print("Creating database connection")
        self.connection = "Connected"

# Only creates one instance
db1 = DatabaseConnection()  # Creates connection
db2 = DatabaseConnection()  # Returns same instance
print(db1 is db2)  # True
```

**Method Decorators (property, staticmethod, classmethod):**
```python
class User:
    def __init__(self, first_name, last_name):
        self._first_name = first_name
        self._last_name = last_name
        self._age = None
    
    @property
    def full_name(self):
        """Computed property"""
        return f"{self._first_name} {self._last_name}"
    
    @property
    def age(self):
        return self._age
    
    @age.setter
    def age(self, value):
        if value < 0:
            raise ValueError("Age cannot be negative")
        self._age = value
    
    @staticmethod
    def is_adult(age):
        """Utility function, doesn't need instance"""
        return age >= 18
    
    @classmethod
    def from_dict(cls, data):
        """Alternative constructor"""
        return cls(data['first_name'], data['last_name'])

# Usage
user = User("John", "Doe")
print(user.full_name)  # John Doe
user.age = 25  # Calls setter
print(User.is_adult(25))  # True
user2 = User.from_dict({'first_name': 'Jane', 'last_name': 'Smith'})
```

**Metaclasses:**

**What are Metaclasses:**
- Classes that create classes
- `type` is the default metaclass
- Control class creation behavior

```python
# Every class is an instance of type
class MyClass:
    pass

print(type(MyClass))  # <class 'type'>
print(type(int))      # <class 'type'>
print(type(str))      # <class 'type'>
```

**Custom Metaclass:**
```python
class ValidationMeta(type):
    """Metaclass that validates class attributes"""
    
    def __new__(mcs, name, bases, attrs):
        # Validate all methods have docstrings
        for attr_name, attr_value in attrs.items():
            if callable(attr_value) and not attr_name.startswith('_'):
                if not attr_value.__doc__:
                    raise TypeError(
                        f"Method {attr_name} in class {name} must have a docstring"
                    )
        
        # Create the class
        return super().__new__(mcs, name, bases, attrs)
    
    def __init__(cls, name, bases, attrs):
        # Additional initialization
        print(f"Creating class: {name}")
        super().__init__(name, bases, attrs)

class APIClient(metaclass=ValidationMeta):
    def get_user(self, user_id):
        """Fetch user by ID"""
        return f"User {user_id}"
    
    def create_user(self, data):
        """Create a new user"""
        return "User created"
    
    # This would raise TypeError:
    # def bad_method(self):
    #     return "No docstring!"
```

**Singleton using Metaclass:**
```python
class SingletonMeta(type):
    _instances = {}
    
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    def __init__(self):
        self.connection = "Connected"

db1 = Database()
db2 = Database()
print(db1 is db2)  # True
```

**ORM-like Metaclass:**
```python
class ModelMeta(type):
    def __new__(mcs, name, bases, attrs):
        # Collect field definitions
        fields = {}
        for key, value in list(attrs.items()):
            if isinstance(value, Field):
                fields[key] = value
                attrs.pop(key)  # Remove from class attrs
        
        attrs['_fields'] = fields
        return super().__new__(mcs, name, bases, attrs)

class Field:
    def __init__(self, field_type, required=True):
        self.field_type = field_type
        self.required = required

class Model(metaclass=ModelMeta):
    def __init__(self, **kwargs):
        for field_name, field in self._fields.items():
            value = kwargs.get(field_name)
            if field.required and value is None:
                raise ValueError(f"{field_name} is required")
            setattr(self, field_name, value)

class User(Model):
    username = Field(str)
    email = Field(str)
    age = Field(int, required=False)

# Usage
user = User(username="john", email="john@example.com", age=30)
print(user.username)  # john
print(User._fields)   # {'username': <Field>, 'email': <Field>, 'age': <Field>}
```

**When to use:**

**Decorators:**
- Logging, timing, caching
- Authentication/authorization
- Input validation
- Rate limiting
- Retry logic

**Metaclasses:**
- ORM implementations (Django, SQLAlchemy)
- API frameworks (automatic serialization)
- Singleton pattern
- Enforcing coding standards
- Auto-registration of plugins

**Note:** "Metaclasses are deeper magic than 99% of users should ever worry about." - Tim Peters

---

### Q3: Explain context managers and the `with` statement.
**Answer:**

**What are Context Managers:**
- Objects that define `__enter__` and `__exit__` methods
- Manage resource setup and teardown
- Guarantee cleanup even if exceptions occur

**Basic Implementation:**
```python
class DatabaseConnection:
    def __init__(self, host, port):
        self.host = host
        self.port = port
        self.connection = None
    
    def __enter__(self):
        """Called when entering the 'with' block"""
        print(f"Connecting to {self.host}:{self.port}")
        self.connection = f"Connection to {self.host}"
        return self.connection
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """Called when exiting the 'with' block"""
        print("Closing connection")
        self.connection = None
        
        # Handle exceptions
        if exc_type is not None:
            print(f"Exception occurred: {exc_type.__name__}: {exc_val}")
            return False  # Re-raise exception
        
        return True  # Suppress exception if True

# Usage
with DatabaseConnection("localhost", 5432) as conn:
    print(f"Using {conn}")
    # Connection automatically closed after block
```

**Using `contextlib`:**

**1. contextmanager decorator:**
```python
from contextlib import contextmanager
import time

@contextmanager
def timing(label):
    """Simple timing context manager"""
    start = time.time()
    try:
        yield  # Code block executes here
    finally:
        end = time.time()
        print(f"{label}: {end - start:.2f}s")

# Usage
with timing("Database query"):
    time.sleep(1)
    result = "Query result"
```

**2. File handling:**
```python
@contextmanager
def open_file(filename, mode='r'):
    """Safe file handling"""
    file = None
    try:
        file = open(filename, mode)
        yield file
    except Exception as e:
        print(f"Error: {e}")
        raise
    finally:
        if file:
            file.close()

with open_file('data.txt', 'w') as f:
    f.write("Hello, World!")
```

**3. Multiple context managers:**
```python
with open('input.txt', 'r') as infile, open('output.txt', 'w') as outfile:
    data = infile.read()
    outfile.write(data.upper())
```

**Real-World Examples:**

**Database Transaction:**
```python
from contextlib import contextmanager
import psycopg2

@contextmanager
def transaction(connection):
    """Database transaction context manager"""
    cursor = connection.cursor()
    try:
        yield cursor
        connection.commit()
        print("Transaction committed")
    except Exception as e:
        connection.rollback()
        print(f"Transaction rolled back: {e}")
        raise
    finally:
        cursor.close()

# Usage
conn = psycopg2.connect("dbname=test user=postgres")
try:
    with transaction(conn) as cursor:
        cursor.execute("INSERT INTO users (name) VALUES (%s)", ("John",))
        cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE user_id = 1")
        # Both queries committed together
finally:
    conn.close()
```

**Lock Management:**
```python
import threading

@contextmanager
def acquire_lock(lock, timeout=10):
    """Thread lock with timeout"""
    acquired = lock.acquire(timeout=timeout)
    try:
        if not acquired:
            raise TimeoutError("Could not acquire lock")
        yield
    finally:
        if acquired:
            lock.release()

lock = threading.Lock()
with acquire_lock(lock, timeout=5):
    # Critical section
    shared_resource += 1
```

**Temporary Directory:**
```python
import tempfile
import shutil
from pathlib import Path

@contextmanager
def temp_directory():
    """Create and cleanup temporary directory"""
    temp_dir = tempfile.mkdtemp()
    try:
        yield Path(temp_dir)
    finally:
        shutil.rmtree(temp_dir)

with temp_directory() as tmp:
    # Create files in tmp
    (tmp / "test.txt").write_text("Hello")
    # Automatically deleted after block
```

**API Rate Limiting:**
```python
import time
from contextlib import contextmanager

class RateLimiter:
    def __init__(self, calls, period):
        self.calls = calls
        self.period = period
        self.call_times = []
    
    @contextmanager
    def limit(self):
        now = time.time()
        
        # Remove old calls outside the window
        self.call_times = [t for t in self.call_times if now - t < self.period]
        
        # Check if limit exceeded
        if len(self.call_times) >= self.calls:
            sleep_time = self.period - (now - self.call_times[0])
            print(f"Rate limit hit. Sleeping for {sleep_time:.2f}s")
            time.sleep(sleep_time)
        
        self.call_times.append(time.time())
        yield

# Usage
limiter = RateLimiter(calls=5, period=10)  # 5 calls per 10 seconds

for i in range(10):
    with limiter.limit():
        print(f"API call {i+1}")
        # Make API request
```

**Redis Connection Pool:**
```python
from contextlib import contextmanager
import redis

class RedisPool:
    def __init__(self, host='localhost', port=6379, max_connections=10):
        self.pool = redis.ConnectionPool(
            host=host,
            port=port,
            max_connections=max_connections
        )
    
    @contextmanager
    def connection(self):
        conn = redis.Redis(connection_pool=self.pool)
        try:
            yield conn
        finally:
            # Connection returned to pool
            pass

pool = RedisPool()

with pool.connection() as redis_client:
    redis_client.set('key', 'value')
    value = redis_client.get('key')
```

---

## 2. ADVANCED PYTHON CONCEPTS

### Q4: Explain async/await and asyncio in Python.
**Answer:**

**Basics:**

**Synchronous vs Asynchronous:**
```python
import time
import asyncio

# Synchronous (blocking)
def fetch_data_sync(n):
    print(f"Fetching {n}")
    time.sleep(2)  # Blocks entire program
    return f"Data {n}"

start = time.time()
results = [fetch_data_sync(i) for i in range(3)]
print(f"Sync time: {time.time() - start:.2f}s")  # ~6 seconds

# Asynchronous (non-blocking)
async def fetch_data_async(n):
    print(f"Fetching {n}")
    await asyncio.sleep(2)  # Doesn't block, switches to other tasks
    return f"Data {n}"

async def main():
    start = time.time()
    tasks = [fetch_data_async(i) for i in range(3)]
    results = await asyncio.gather(*tasks)
    print(f"Async time: {time.time() - start:.2f}s")  # ~2 seconds
    return results

asyncio.run(main())
```

**Core Concepts:**

**1. Coroutines:**
```python
async def my_coroutine():
    """Defined with async def, returns a coroutine object"""
    await asyncio.sleep(1)
    return "Done"

# Create coroutine (doesn't execute yet)
coro = my_coroutine()

# Execute it
result = asyncio.run(coro)
```

**2. Tasks:**
```python
async def background_task():
    while True:
        print("Background task running")
        await asyncio.sleep(2)

async def main():
    # Create task (starts execution immediately)
    task = asyncio.create_task(background_task())
    
    # Do other work
    await asyncio.sleep(5)
    
    # Cancel task
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("Task cancelled")

asyncio.run(main())
```

**3. gather vs wait:**
```python
async def task1():
    await asyncio.sleep(2)
    return "Task 1"

async def task2():
    await asyncio.sleep(1)
    return "Task 2"

async def task3():
    await asyncio.sleep(3)
    raise Exception("Task 3 failed")

# gather - waits for all, raises first exception
async def with_gather():
    try:
        results = await asyncio.gather(task1(), task2(), task3())
    except Exception as e:
        print(f"gather failed: {e}")

# gather with return_exceptions
async def with_gather_safe():
    results = await asyncio.gather(
        task1(), task2(), task3(),
        return_exceptions=True
    )
    # results = ["Task 1", "Task 2", Exception("Task 3 failed")]

# wait - more control
async def with_wait():
    tasks = [task1(), task2(), task3()]
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )
    
    for task in done:
        try:
            result = await task
            print(f"Result: {result}")
        except Exception as e:
            print(f"Error: {e}")
    
    # Cancel pending tasks
    for task in pending:
        task.cancel()
```

**Real-World Examples:**

**1. Async HTTP Requests:**
```python
import aiohttp
import asyncio

async def fetch_url(session, url):
    async with session.get(url) as response:
        return await response.text()

async def fetch_multiple_urls(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        return results

# Usage
urls = [
    'https://api.example.com/users/1',
    'https://api.example.com/users/2',
    'https://api.example.com/users/3'
]
results = asyncio.run(fetch_multiple_urls(urls))
```

**2. Async Database Operations:**
```python
import asyncpg
import asyncio

async def fetch_users():
    conn = await asyncpg.connect(
        user='user',
        password='password',
        database='mydb',
        host='localhost'
    )
    
    try:
        # Concurrent queries
        users, posts = await asyncio.gather(
            conn.fetch('SELECT * FROM users'),
            conn.fetch('SELECT * FROM posts')
        )
        return users, posts
    finally:
        await conn.close()

# Connection pool
async def with_pool():
    pool = await asyncpg.create_pool(
        user='user',
        password='password',
        database='mydb',
        host='localhost',
        min_size=5,
        max_size=20
    )
    
    try:
        async with pool.acquire() as conn:
            result = await conn.fetch('SELECT * FROM users')
            return result
    finally:
        await pool.close()
```

**3. Async Queue (Producer-Consumer):**
```python
import asyncio
import random

async def producer(queue, n):
    for i in range(n):
        item = f"item-{i}"
        await asyncio.sleep(random.uniform(0.1, 0.5))
        await queue.put(item)
        print(f"Produced: {item}")
    
    # Signal completion
    await queue.put(None)

async def consumer(queue, name):
    while True:
        item = await queue.get()
        if item is None:
            queue.task_done()
            break
        
        print(f"{name} consuming: {item}")
        await asyncio.sleep(random.uniform(0.2, 0.7))
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=10)
    
    # Create tasks
    prod = asyncio.create_task(producer(queue, 20))
    consumers = [
        asyncio.create_task(consumer(queue, f"Consumer-{i}"))
        for i in range(3)
    ]
    
    # Wait for producer
    await prod
    
    # Wait for queue to be empty
    await queue.join()
    
    # Cancel consumers
    for c in consumers:
        c.cancel()

asyncio.run(main())
```

**4. Async Context Manager:**
```python
class AsyncDatabaseConnection:
    def __init__(self, host):
        self.host = host
        self.connection = None
    
    async def __aenter__(self):
        print("Connecting...")
        await asyncio.sleep(0.5)  # Simulate connection
        self.connection = f"Connected to {self.host}"
        return self.connection
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print("Closing connection...")
        await asyncio.sleep(0.2)  # Simulate cleanup
        self.connection = None

async def main():
    async with AsyncDatabaseConnection("localhost") as conn:
        print(f"Using {conn}")
        await asyncio.sleep(1)

asyncio.run(main())
```

**5. Async Iterator:**
```python
class AsyncRange:
    def __init__(self, start, stop):
        self.current = start
        self.stop = stop
    
    def __aiter__(self):
        return self
    
    async def __anext__(self):
        if self.current >= self.stop:
            raise StopAsyncIteration
        
        await asyncio.sleep(0.1)  # Simulate async operation
        value = self.current
        self.current += 1
        return value

async def main():
    async for i in AsyncRange(0, 5):
        print(i)

asyncio.run(main())
```

**Best Practices:**

**1. Use asyncio.run() for entry point:**
```python
# Good
async def main():
    # Your async code
    pass

if __name__ == "__main__":
    asyncio.run(main())

# Bad - manual event loop management
# loop = asyncio.get_event_loop()
# loop.run_until_complete(main())
```

**2. Always await coroutines:**
```python
# Bad - coroutine never executes
async def bad():
    my_coroutine()  # Missing await!

# Good
async def good():
    await my_coroutine()
```

**3. Use asyncio.create_task() for background tasks:**
```python
async def main():
    # Starts immediately
    task = asyncio.create_task(background_work())
    
    # Do other work
    await other_work()
    
    # Wait for background task
    await task
```

**4. Handle exceptions properly:**
```python
async def safe_task():
    try:
        await risky_operation()
    except Exception as e:
        logger.error(f"Task failed: {e}")
        # Don't let exceptions kill your event loop
```

**When to use async:**
- I/O-bound operations (API calls, database queries)
- Network operations
- File I/O (with aiofiles)
- Concurrent web scraping
- WebSocket connections
- Microservice communication

**When NOT to use async:**
- CPU-bound operations (use multiprocessing)
- Simple scripts
- When libraries don't support async
- Legacy code without async support

---

### Q5: Explain Python's memory management and garbage collection.
**Answer:**

**Memory Management Basics:**

**1. Reference Counting:**
```python
import sys

a = []
print(sys.getrefcount(a))  # 2 (a + getrefcount parameter)

b = a
print(sys.getrefcount(a))  # 3 (a, b + getrefcount parameter)

del b
print(sys.getrefcount(a))  # 2 (back to original)
```

**2. Object Allocation:**
```python
# Small integers are cached (-5 to 256)
a = 100
b = 100
print(a is b)  # True (same object)

a = 1000
b = 1000
print(a is b)  # False (different objects)

# String interning
s1 = "hello"
s2 = "hello"
print(s1 is s2)  # True (interned)

s1 = "hello world!"
s2 = "hello world!"
print(s1 is s2)  # False (not interned - has space and !)
```

**3. Garbage Collection:**

```python
import gc

# Check if GC is enabled
print(gc.isenabled())  # True

# Get GC stats
print(gc.get_stats())

# Manual garbage collection
gc.collect()  # Force collection

# Disable/Enable
gc.disable()
# ... code ...
gc.enable()
```

**Circular References:**
```python
import gc

class Node:
    def __init__(self, value):
        self.value = value
        self.next = None
    
    def __del__(self):
        print(f"Node {self.value} deleted")

# Create circular reference
node1 = Node(1)
node2 = Node(2)
node1.next = node2
node2.next = node1  # Circular reference

# Delete references
del node1
del node2

# Objects not deleted yet (circular reference prevents ref count from reaching 0)
print("After del")

# Force garbage collection
gc.collect()  # Now objects are deleted
print("After gc.collect()")
```

**Generations:**
```python
# Python uses generational GC
# Generation 0: newly created objects
# Generation 1: survived one collection
# Generation 2: survived multiple collections

# Get generation thresholds
print(gc.get_threshold())  # (700, 10, 10)
# 700 objects in gen0 triggers collection
# 10 gen0 collections trigger gen1 collection
# 10 gen1 collections trigger gen2 collection

# Get current counts
print(gc.get_count())  # (current gen0, gen1, gen2 counts)
```

**Memory Leaks:**

**1. Global Variables:**
```python
# Bad - holds reference forever
cache = {}

def process_data(key, data):
    cache[key] = data  # Never cleaned up
    # ... process ...

# Good - use weakref or limit cache size
from functools import lru_cache

@lru_cache(maxsize=128)
def process_data(key):
    # Automatically manages cache size
    pass
```

**2. Closures:**
```python
# Potential memory leak
def create_handlers():
    large_data = list(range(1000000))
    
    def handler():
        # Closure keeps reference to large_data
        print(large_data[0])
    
    return handler

# large_data stays in memory as long as handler exists

# Better
def create_handlers():
    large_data = list(range(1000000))
    first_element = large_data[0]  # Extract what you need
    del large_data  # Explicitly delete
    
    def handler():
        print(first_element)
    
    return handler
```

**3. Cyclic References with __del__:**
```python
# Problematic
class Resource:
    def __init__(self):
        self.data = []
    
    def __del__(self):
        # Cleanup code
        pass

# Better - use context manager
class Resource:
    def __enter__(self):
        return self
    
    def __exit__(self, *args):
        # Guaranteed cleanup
        pass

with Resource() as res:
    # Use resource
    pass
```

**Memory Profiling:**

```python
import tracemalloc

# Start tracking
tracemalloc.start()

# Your code
data = [i for i in range(1000000)]

# Get memory usage
current, peak = tracemalloc.get_traced_memory()
print(f"Current: {current / 10**6:.2f} MB")
print(f"Peak: {peak / 10**6:.2f} MB")

# Get top memory consumers
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')

for stat in top_stats[:10]:
    print(stat)

tracemalloc.stop()
```

**Memory Optimization Techniques:**

**1. __slots__:**
```python
# Regular class - uses dict for attributes
class RegularPerson:
    def __init__(self, name, age):
        self.name = name
        self.age = age

# With __slots__ - no dict, fixed attributes
class SlottedPerson:
    __slots__ = ['name', 'age']
    
    def __init__(self, name, age):
        self.name = name
        self.age = age

# Memory comparison
import sys

regular = RegularPerson("John", 30)
slotted = SlottedPerson("John", 30)

print(f"Regular: {sys.getsizeof(regular.__dict__)} bytes")  # ~240 bytes
print(f"Slotted: {sys.getsizeof(slotted)} bytes")           # ~64 bytes

# Savings for millions of objects
regular_list = [RegularPerson(f"User{i}", i) for i in range(100000)]
slotted_list = [SlottedPerson(f"User{i}", i) for i in range(100000)]
```

**2. Generators vs Lists:**
```python
# Memory intensive
def get_numbers():
    return [i for i in range(1000000)]  # Creates entire list in memory

# Memory efficient
def get_numbers_gen():
    return (i for i in range(1000000))  # Yields one at a time

import sys
list_obj = get_numbers()
gen_obj = get_numbers_gen()

print(f"List: {sys.getsizeof(list_obj)} bytes")  # ~8MB
print(f"Generator: {sys.getsizeof(gen_obj)} bytes")  # ~128 bytes
```

**3. Weak References:**
```python
import weakref

class ExpensiveObject:
    def __init__(self, data):
        self.data = data

# Strong reference - prevents GC
cache = {}
obj = ExpensiveObject([1, 2, 3])
cache['key'] = obj  # obj won't be collected

# Weak reference - allows GC
weak_cache = {}
obj = ExpensiveObject([1, 2, 3])
weak_cache['key'] = weakref.ref(obj)

# Access weak reference
cached_obj = weak_cache['key']()
if cached_obj is not None:
    print(cached_obj.data)
else:
    print("Object was garbage collected")

del obj
gc.collect()
# Now weak reference returns None
```

---

## 3. DATABASE & ORM

### Q6: Explain N+1 query problem and how to solve it.
**Answer:**

**The Problem:**

```python
# models.py
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship

class Author(Base):
    __tablename__ = 'authors'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    books = relationship('Book', back_populates='author')

class Book(Base):
    __tablename__ = 'books'
    id = Column(Integer, primary_key=True)
    title = Column(String)
    author_id = Column(Integer, ForeignKey('authors.id'))
    author = relationship('Author', back_populates='books')

# BAD - N+1 Query Problem
def get_books_with_authors_bad():
    books = session.query(Book).all()  # 1 query
    
    for book in books:
        print(f"{book.title} by {book.author.name}")  # N queries (one per book)
    
    # Total: 1 + N queries where N = number of books

# If you have 100 books, this executes 101 queries!
```

**Solutions:**

**1. Eager Loading (joinedload):**
```python
from sqlalchemy.orm import joinedload

def get_books_with_authors_good():
    books = session.query(Book).options(
        joinedload(Book.author)
    ).all()  # Single query with JOIN
    
    for book in books:
        print(f"{book.title} by {book.author.name}")  # No additional queries
    
    # Total: 1 query

# Generated SQL:
# SELECT books.*, authors.* 
# FROM books 
# LEFT OUTER JOIN authors ON authors.id = books.author_id
```

**2. Subquery Load:**
```python
from sqlalchemy.orm import subqueryload

def get_books_with_authors():
    books = session.query(Book).options(
        subqueryload(Book.author)
    ).all()  # 2 queries total
    
    # Query 1: SELECT * FROM books
    # Query 2: SELECT * FROM authors WHERE id IN (author_ids from books)
```

**3. Select IN Loading:**
```python
from sqlalchemy.orm import selectinload

def get_authors_with_books():
    authors = session.query(Author).options(
        selectinload(Author.books)
    ).all()
    
    for author in authors:
        for book in author.books:
            print(f"{author.name}: {book.title}")
    
    # Query 1: SELECT * FROM authors
    # Query 2: SELECT * FROM books WHERE author_id IN (1, 2, 3, ...)
```

**Django ORM:**

```python
# Bad - N+1 problem
books = Book.objects.all()
for book in books:
    print(f"{book.title} by {book.author.name}")  # Hits DB each time

# Good - select_related (for ForeignKey, OneToOne)
books = Book.objects.select_related('author').all()
for book in books:
    print(f"{book.title} by {book.author.name}")  # No extra queries

# prefetch_related (for ManyToMany, reverse ForeignKey)
authors = Author.objects.prefetch_related('books').all()
for author in authors:
    for book in author.books.all():  # No extra queries
        print(f"{author.name}: {book.title}")

# Complex example
books = (
    Book.objects
    .select_related('author', 'publisher')  # ForeignKeys
    .prefetch_related('categories', 'reviews')  # ManyToMany
    .all()
)
```

**Manual Optimization:**

```python
# Raw SQL approach
def get_books_manual():
    query = """
        SELECT 
            b.id, b.title, 
            a.id as author_id, a.name as author_name
        FROM books b
        JOIN authors a ON b.author_id = a.id
    """
    
    results = session.execute(query).fetchall()
    
    books = []
    for row in results:
        book = {
            'id': row.id,
            'title': row.title,
            'author': {
                'id': row.author_id,
                'name': row.author_name
            }
        }
        books.append(book)
    
    return books
```

**Detection:**

```python
from sqlalchemy import event
from sqlalchemy.engine import Engine
import logging

# Log all SQL queries
logging.basicConfig()
logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)

# Count queries
query_count = 0

@event.listens_for(Engine, "before_cursor_execute")
def before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    global query_count
    query_count += 1
    print(f"Query {query_count}: {statement}")

# Django debug toolbar
# Shows all queries and highlights N+1 problems
```

**Best Practices:**

1. **Always use eager loading for related data**
2. **Use `only()` to select specific fields**
```python
books = Book.objects.only('title', 'author__name')
```

3. **Use `defer()` to exclude heavy fields**
```python
books = Book.objects.defer('description', 'content')
```

4. **Batch operations**
```python
# Bad
for user in users:
    user.is_active = True
    user.save()  # N queries

# Good
User.objects.filter(id__in=user_ids).update(is_active=True)  # 1 query
```

---

### Q7: Design a database schema for a complex system (e-commerce example).
**Answer:**

**Requirements:**
- Users can place orders
- Orders contain multiple products
- Products have variants (size, color)
- Track inventory
- Payment processing
- Order status tracking
- Reviews and ratings

**Schema Design:**

```python
from sqlalchemy import (
    Column, Integer, String, Float, DateTime, Boolean, 
    ForeignKey, Enum, Text, Numeric, UniqueConstraint, Index
)
from sqlalchemy.orm import relationship, declarative_base
from sqlalchemy.sql import func
from datetime import datetime
import enum

Base = declarative_base()

# Enums
class OrderStatus(enum.Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    PROCESSING = "processing"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class PaymentStatus(enum.Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"
    REFUNDED = "refunded"

# Users
class User(Base):
    __tablename__ = 'users'
    
    id = Column(Integer, primary_key=True)
    email = Column(String(255), unique=True, nullable=False, index=True)
    username = Column(String(100), unique=True, nullable=False)
    password_hash = Column(String(255), nullable=False)
    first_name = Column(String(100))
    last_name = Column(String(100))
    phone = Column(String(20))
    is_active = Column(Boolean, default=True)
    is_verified = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Relationships
    addresses = relationship('Address', back_populates='user', cascade='all, delete-orphan')
    orders = relationship('Order', back_populates='user')
    reviews = relationship('Review', back_populates='user')
    cart_items = relationship('CartItem', back_populates='user', cascade='all, delete-orphan')
    
    __table_args__ = (
        Index('ix_users_email_active', 'email', 'is_active'),
    )

# Addresses
class Address(Base):
    __tablename__ = 'addresses'
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('users.id'), nullable=False)
    address_type = Column(String(20))  # shipping, billing
    street_address = Column(String(255), nullable=False)
    city = Column(String(100), nullable=False)
    state = Column(String(100))
    postal_code = Column(String(20), nullable=False)
    country = Column(String(100), nullable=False)
    is_default = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Relationships
    user = relationship('User', back_populates='addresses')
    
    __table_args__ = (
        Index('ix_addresses_user_default', 'user_id', 'is_default'),
    )

# Categories
class Category(Base):
    __tablename__ = 'categories'
    
    id = Column(Integer, primary_key=True)
    name = Column(String(100), unique=True, nullable=False)
    slug = Column(String(100), unique=True, nullable=False, index=True)
    description = Column(Text)
    parent_id = Column(Integer, ForeignKey('categories.id'))
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Self-referential relationship for subcategories
    parent = relationship('Category', remote_side=[id], backref='subcategories')
    products = relationship('Product', back_populates='category')

# Products
class Product(Base):
    __tablename__ = 'products'
    
    id = Column(Integer, primary_key=True)
    name = Column(String(255), nullable=False)
    slug = Column(String(255), unique=True, nullable=False, index=True)
    description = Column(Text)
    category_id = Column(Integer, ForeignKey('categories.id'))
    base_price = Column(Numeric(10, 2), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Relationships
    category = relationship('Category', back_populates='products')
    variants = relationship('ProductVariant', back_populates='product', cascade='all, delete-orphan')
    images = relationship('ProductImage', back_populates='product', cascade='all, delete-orphan')
    reviews = relationship('Review', back_populates='product')
    
    __table_args__ = (
        Index('ix_products_category_active', 'category_id', 'is_active'),
    )

# Product Variants (size, color combinations)
class ProductVariant(Base):
    __tablename__ = 'product_variants'
    
    id = Column(Integer, primary_key=True)
    product_id = Column(Integer, ForeignKey('products.id'), nullable=False)
    sku = Column(String(100), unique=True, nullable=False, index=True)
    size = Column(String(50))
    color = Column(String(50))
    price = Column(Numeric(10, 2), nullable=False)
    stock_quantity = Column(Integer, default=0)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Relationships
    product = relationship('Product', back_populates='variants')
    order_items = relationship('OrderItem', back_populates='variant')
    
    __table_args__ = (
        UniqueConstraint('product_id', 'size', 'color', name='uq_variant'),
        Index('ix_variants_sku_active', 'sku', 'is_active'),
    )

# Product Images
class ProductImage(Base):
    __tablename__ = 'product_images'
    
    id = Column(Integer, primary_key=True)
    product_id = Column(Integer, ForeignKey('products.id'), nullable=False)
    image_url = Column(String(500), nullable=False)
    is_primary = Column(Boolean, default=False)
    display_order = Column(Integer, default=0)
    
    # Relationships
    product = relationship('Product', back_populates='images')

# Shopping Cart
class CartItem(Base):
    __tablename__ = 'cart_items'
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('users.id'), nullable=False)
    variant_id = Column(Integer, ForeignKey('product_variants.id'), nullable=False)
    quantity = Column(Integer, default=1)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Relationships
    user = relationship('User', back_populates='cart_items')
    variant = relationship('ProductVariant')
    
    __table_args__ = (
        UniqueConstraint('user_id', 'variant_id', name='uq_cart_item'),
        Index('ix_cart_user_created', 'user_id', 'created_at'),
    )

# Orders
class Order(Base):
    __tablename__ = 'orders'
    
    id = Column(Integer, primary_key=True)
    order_number = Column(String(50), unique=True, nullable=False, index=True)
    user_id = Column(Integer, ForeignKey('users.id'), nullable=False)
    status = Column(Enum(OrderStatus), default=OrderStatus.PENDING, nullable=False)
    
    # Amounts
    subtotal = Column(Numeric(10, 2), nullable=False)
    tax = Column(Numeric(10, 2), default=0)
    shipping_cost = Column(Numeric(10, 2), default=0)
    discount = Column(Numeric(10, 2), default=0)
    total = Column(Numeric(10, 2), nullable=False)
    
    # Addresses (denormalized for historical record)
    shipping_address = Column(Text, nullable=False)
    billing_address = Column(Text)
    
    # Timestamps
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    shipped_at = Column(DateTime)
    delivered_at = Column(DateTime)
    
    # Relationships
    user = relationship('User', back_populates='orders')
    items = relationship('OrderItem', back_populates='order', cascade='all, delete-orphan')
    payments = relationship('Payment', back_populates='order', cascade='all, delete-orphan')
    
    __table_args__ = (
        Index('ix_orders_user_status', 'user_id', 'status'),
        Index('ix_orders_created', 'created_at'),
    )

# Order Items
class OrderItem(Base):
    __tablename__ = 'order_items'
    
    id = Column(Integer, primary_key=True)
    order_id = Column(Integer, ForeignKey('orders.id'), nullable=False)
    variant_id = Column(Integer, ForeignKey('product_variants.id'), nullable=False)
    
    # Snapshot of product at time of order
    product_name = Column(String(255), nullable=False)
    variant_details = Column(String(255))  # size, color
    price = Column(Numeric(10, 2), nullable=False)
    quantity = Column(Integer, nullable=False)
    subtotal = Column(Numeric(10, 2), nullable=False)
    
    # Relationships
    order = relationship('Order', back_populates='items')
    variant = relationship('ProductVariant', back_populates='order_items')
    
    __table_args__ = (
        Index('ix_order_items_order', 'order_id'),
    )

# Payments
class Payment(Base):
    __tablename__ = 'payments'
    
    id = Column(Integer, primary_key=True)
    order_id = Column(Integer, ForeignKey('orders.id'), nullable=False)
    payment_method = Column(String(50), nullable=False)  # credit_card, paypal, etc.
    payment_status = Column(Enum(PaymentStatus), default=PaymentStatus.PENDING)
    amount = Column(Numeric(10, 2), nullable=False)
    transaction_id = Column(String(100), unique=True, index=True)
    payment_gateway_response = Column(Text)  # Store raw response
    created_at = Column(DateTime, default=datetime.utcnow)
    processed_at = Column(DateTime)
    
    # Relationships
    order = relationship('Order', back_populates='payments')
    
    __table_args__ = (
        Index('ix_payments_order_status', 'order_id', 'payment_status'),
    )

# Reviews
class Review(Base):
    __tablename__ = 'reviews'
    
    id = Column(Integer, primary_key=True)
    product_id = Column(Integer, ForeignKey('products.id'), nullable=False)
    user_id = Column(Integer, ForeignKey('users.id'), nullable=False)
    rating = Column(Integer, nullable=False)  # 1-5
    title = Column(String(255))
    comment = Column(Text)
    is_verified_purchase = Column(Boolean, default=False)
    is_approved = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Relationships
    product = relationship('Product', back_populates='reviews')
    user = relationship('User', back_populates='reviews')
    
    __table_args__ = (
        UniqueConstraint('product_id', 'user_id', name='uq_review'),
        Index('ix_reviews_product_approved', 'product_id', 'is_approved'),
    )
```

**Advanced Queries:**

```python
from sqlalchemy import func, and_, or_

class EcommerceQueries:
    
    @staticmethod
    def get_user_orders(session, user_id, status=None):
        """Get user orders with items"""
        query = (
            session.query(Order)
            .options(
                joinedload(Order.items).joinedload(OrderItem.variant),
                joinedload(Order.payments)
            )
            .filter(Order.user_id == user_id)
        )
        
        if status:
            query = query.filter(Order.status == status)
        
        return query.order_by(Order.created_at.desc()).all()
    
    @staticmethod
    def get_product_with_variants(session, product_slug):
        """Get product with all variants and images"""
        return (
            session.query(Product)
            .options(
                joinedload(Product.variants),
                joinedload(Product.images),
                joinedload(Product.category)
            )
            .filter(Product.slug == product_slug)
            .first()
        )
    
    @staticmethod
    def get_products_by_category(session, category_id, page=1, per_page=20):
        """Get paginated products by category"""
        offset = (page - 1) * per_page
        
        return (
            session.query(Product)
            .filter(
                Product.category_id == category_id,
                Product.is_active == True
            )
            .options(
                joinedload(Product.images),
                selectinload(Product.variants)
            )
            .offset(offset)
            .limit(per_page)
            .all()
        )
    
    @staticmethod
    def get_product_reviews(session, product_id):
        """Get approved reviews with user info"""
        return (
            session.query(Review)
            .options(joinedload(Review.user))
            .filter(
                Review.product_id == product_id,
                Review.is_approved == True
            )
            .order_by(Review.created_at.desc())
            .all()
        )
    
    @staticmethod
    def get_low_stock_variants(session, threshold=10):
        """Get variants with low stock"""
        return (
            session.query(ProductVariant)
            .options(joinedload(ProductVariant.product))
            .filter(
                ProductVariant.stock_quantity <= threshold,
                ProductVariant.is_active == True
            )
            .all()
        )
    
    @staticmethod
    def get_sales_report(session, start_date, end_date):
        """Get sales statistics"""
        return (
            session.query(
                func.date(Order.created_at).label('date'),
                func.count(Order.id).label('total_orders'),
                func.sum(Order.total).label('total_revenue'),
                func.avg(Order.total).label('average_order_value')
            )
            .filter(
                Order.created_at.between(start_date, end_date),
                Order.status.in_([OrderStatus.DELIVERED, OrderStatus.SHIPPED])
            )
            .group_by(func.date(Order.created_at))
            .order_by(func.date(Order.created_at))
            .all()
        )
    
    @staticmethod
    def get_top_selling_products(session, limit=10):
        """Get best-selling products"""
        return (
            session.query(
                Product.id,
                Product.name,
                func.sum(OrderItem.quantity).label('total_sold'),
                func.sum(OrderItem.subtotal).label('total_revenue')
            )
            .join(ProductVariant, ProductVariant.product_id == Product.id)
            .join(OrderItem, OrderItem.variant_id == ProductVariant.id)
            .join(Order, Order.id == OrderItem.order_id)
            .filter(Order.status.in_([OrderStatus.DELIVERED, OrderStatus.SHIPPED]))
            .group_by(Product.id, Product.name)
            .order_by(func.sum(OrderItem.quantity).desc())
            .limit(limit)
            .all()
        )
```

**Optimization Strategies:**

**1. Indexing:**
```python
# Composite indexes for common queries
Index('ix_orders_user_status_created', 'user_id', 'status', 'created_at')
Index('ix_products_category_price', 'category_id', 'base_price')
```

**2. Partitioning:**
```sql
-- Partition orders by year
CREATE TABLE orders_2024 PARTITION OF orders
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

**3. Caching:**
```python
from functools import lru_cache
import redis

redis_client = redis.Redis(host='localhost', port=6379)

def get_product_cached(product_id):
    # Try cache first
    cache_key = f"product:{product_id}"
    cached = redis_client.get(cache_key)
    
    if cached:
        return json.loads(cached)
    
    # Fetch from DB
    product = session.query(Product).get(product_id)
    
    # Cache for 1 hour
    redis_client.setex(cache_key, 3600, json.dumps(product.to_dict()))
    
    return product
```

**4. Denormalization:**
```python
# Add computed columns
class Product(Base):
    # ... other fields ...
    average_rating = Column(Float)  # Computed from reviews
    review_count = Column(Integer, default=0)
    total_sold = Column(Integer, default=0)

# Update with triggers or scheduled jobs
```

---

## 4. API DESIGN & REST

### Q8: Design a RESTful API for a complex system.
**Answer:**

**API Structure:**

```python
from fastapi import FastAPI, Depends, HTTPException, status, Query
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel, EmailStr, Field, validator
from typing import List, Optional
from datetime import datetime, timedelta
import jwt

app = FastAPI(title="E-Commerce API", version="1.0.0")

# Pydantic Models (Request/Response schemas)

class UserCreate(BaseModel):
    email: EmailStr
    username: str = Field(..., min_length=3, max_length=50)
    password: str = Field(..., min_length=8)
    first_name: str
    last_name: str
    
    @validator('password')
    def validate_password(cls, v):
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain uppercase letter')
        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain digit')
        return v

class UserResponse(BaseModel):
    id: int
    email: str
    username: str
    first_name: str
    last_name: str
    is_active: bool
    created_at: datetime
    
    class Config:
        orm_mode = True

class ProductResponse(BaseModel):
    id: int
    name: str
    slug: str
    description: Optional[str]
    base_price: float
    category_id: int
    is_active: bool
    average_rating: Optional[float]
    review_count: int
    
    class Config:
        orm_mode = True

class ProductListResponse(BaseModel):
    items: List[ProductResponse]
    total: int
    page: int
    per_page: int
    total_pages: int

class OrderItemCreate(BaseModel):
    variant_id: int
    quantity: int = Field(..., gt=0)

class OrderCreate(BaseModel):
    items: List[OrderItemCreate]
    shipping_address_id: int
    billing_address_id: Optional[int]

class OrderResponse(BaseModel):
    id: int
    order_number: str
    status: str
    subtotal: float
    tax: float
    shipping_cost: float
    total: float
    created_at: datetime
    
    class Config:
        orm_mode = True

# Authentication
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"

def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(hours=24)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: int = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
    except jwt.PyJWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
    
    user = session.query(User).get(user_id)
    if user is None:
        raise HTTPException(status_code=401, detail="User not found")
    
    return user

# Endpoints

# Authentication
@app.post("/api/v1/auth/register", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def register(user_data: UserCreate):
    """Register a new user"""
    # Check if user exists
    existing = session.query(User).filter(
        or_(User.email == user_data.email, User.username == user_data.username)
    ).first()
    
    if existing:
        raise HTTPException(status_code=400, detail="User already exists")
    
    # Hash password
    from passlib.hash import bcrypt
    hashed_password = bcrypt.hash(user_data.password)
    
    # Create user
    user = User(
        email=user_data.email,
        username=user_data.username,
        password_hash=hashed_password,
        first_name=user_data.first_name,
        last_name=user_data.last_name
    )
    
    session.add(user)
    session.commit()
    session.refresh(user)
    
    return user

@app.post("/api/v1/auth/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """Login and get access token"""
    user = session.query(User).filter(User.username == form_data.username).first()
    
    if not user or not bcrypt.verify(form_data.password, user.password_hash):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    access_token = create_access_token(data={"sub": user.id})
    
    return {
        "access_token": access_token,
        "token_type": "bearer"
    }

# Products
@app.get("/api/v1/products", response_model=ProductListResponse)
async def get_products(
    category_id: Optional[int] = None,
    min_price: Optional[float] = None,
    