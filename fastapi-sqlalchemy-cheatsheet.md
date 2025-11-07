# FastAPI SQLAlchemy Query Cheatsheet

## Setup

```python
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session, relationship
from fastapi import FastAPI, Depends

# Database setup
SQLALCHEMY_DATABASE_URL = "sqlite:///./sql_app.db"
engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Dependency
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Model example
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    email = Column(String, unique=True, index=True)
    age = Column(Integer)
```

## Basic Queries

### Retrieving Objects
```python
# Get all objects
db.query(User).all()

# Get single object by primary key
db.query(User).get(1)

# Get first object
db.query(User).first()

# Get one (raises if multiple found)
db.query(User).one()

# Get one or none
db.query(User).one_or_none()

# Filter and get first
db.query(User).filter(User.name == "John").first()

# Count
db.query(User).count()

# Check if exists
db.query(User).filter(User.name == "John").first() is not None
# Or using scalar
db.query(User.id).filter(User.name == "John").scalar() is not None
```

## Filtering

### Basic Filtering
```python
# Equal
db.query(User).filter(User.name == "John").all()

# Not equal
db.query(User).filter(User.name != "John").all()

# Multiple conditions (AND)
db.query(User).filter(User.name == "John", User.age > 18).all()
db.query(User).filter(User.name == "John").filter(User.age > 18).all()

# IN
db.query(User).filter(User.id.in_([1, 2, 3])).all()

# NOT IN
db.query(User).filter(~User.id.in_([1, 2, 3])).all()

# IS NULL
db.query(User).filter(User.name.is_(None)).all()
db.query(User).filter(User.name == None).all()

# IS NOT NULL
db.query(User).filter(User.name.isnot(None)).all()
db.query(User).filter(User.name != None).all()
```

### Comparison Operators
```python
from sqlalchemy import and_, or_, not_

# Greater than / Less than
db.query(User).filter(User.age > 18).all()
db.query(User).filter(User.age >= 18).all()
db.query(User).filter(User.age < 65).all()
db.query(User).filter(User.age <= 65).all()

# Between
db.query(User).filter(User.age.between(18, 65)).all()

# AND
db.query(User).filter(and_(User.name == "John", User.age > 18)).all()

# OR
db.query(User).filter(or_(User.name == "John", User.name == "Jane")).all()

# NOT
db.query(User).filter(not_(User.name == "John")).all()
db.query(User).filter(~(User.name == "John")).all()
```

### String Operations
```python
# LIKE
db.query(User).filter(User.name.like("%John%")).all()

# ILIKE (case-insensitive)
db.query(User).filter(User.name.ilike("%john%")).all()

# Starts with
db.query(User).filter(User.name.startswith("J")).all()

# Ends with
db.query(User).filter(User.name.endswith("n")).all()

# Contains
db.query(User).filter(User.name.contains("oh")).all()

# Match (regex - database dependent)
db.query(User).filter(User.name.match("^J")).all()
```

## Ordering

```python
# Order by (ascending)
db.query(User).order_by(User.name).all()

# Order by (descending)
db.query(User).order_by(User.name.desc()).all()

# Multiple columns
db.query(User).order_by(User.name, User.age.desc()).all()

# Using asc() explicitly
db.query(User).order_by(User.name.asc()).all()

# Nulls first/last
db.query(User).order_by(User.name.asc().nullsfirst()).all()
db.query(User).order_by(User.name.asc().nullslast()).all()
```

## Limiting and Offsetting

```python
# Limit
db.query(User).limit(5).all()

# Offset
db.query(User).offset(10).all()

# Offset and limit (pagination)
db.query(User).offset(10).limit(5).all()

# Slice notation
db.query(User).slice(10, 15).all()

# Using slice directly
db.query(User).all()[10:15]
```

## Selecting Specific Columns

```python
# Select specific columns
db.query(User.name, User.email).all()

# Returns tuples
results = db.query(User.name, User.email).all()
# [("John", "john@example.com"), ...]

# Using with_entities
db.query(User).with_entities(User.name, User.email).all()

# Load only specific columns (optimization)
from sqlalchemy.orm import load_only
db.query(User).options(load_only(User.name, User.email)).all()

# Defer columns (load everything except specified)
from sqlalchemy.orm import defer
db.query(User).options(defer(User.email)).all()
```

## Aggregation

```python
from sqlalchemy import func

# Count
db.query(func.count(User.id)).scalar()
db.query(func.count()).select_from(User).scalar()

# Count with filter
db.query(func.count(User.id)).filter(User.age > 18).scalar()

# Sum
db.query(func.sum(User.age)).scalar()

# Average
db.query(func.avg(User.age)).scalar()

# Max / Min
db.query(func.max(User.age)).scalar()
db.query(func.min(User.age)).scalar()

# Multiple aggregations
db.query(
    func.count(User.id).label('total'),
    func.avg(User.age).label('avg_age'),
    func.max(User.age).label('max_age')
).first()
```

## Grouping

```python
# Group by
db.query(User.age, func.count(User.id)).group_by(User.age).all()

# Group by with having
db.query(User.age, func.count(User.id))\
    .group_by(User.age)\
    .having(func.count(User.id) > 5)\
    .all()

# Multiple group by columns
db.query(User.name, User.age, func.count(User.id))\
    .group_by(User.name, User.age)\
    .all()
```

## Relationships

### Setup
```python
class Post(Base):
    __tablename__ = "posts"
    id = Column(Integer, primary_key=True)
    title = Column(String)
    user_id = Column(Integer, ForeignKey('users.id'))
    user = relationship("User", back_populates="posts")

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    posts = relationship("Post", back_populates="user")

# Many-to-Many
from sqlalchemy import Table

user_tags = Table('user_tags', Base.metadata,
    Column('user_id', Integer, ForeignKey('users.id')),
    Column('tag_id', Integer, ForeignKey('tags.id'))
)

class Tag(Base):
    __tablename__ = "tags"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    users = relationship("User", secondary=user_tags, back_populates="tags")

User.tags = relationship("Tag", secondary=user_tags, back_populates="users")
```

### Querying Relationships
```python
# Join (implicit)
db.query(User, Post).filter(User.id == Post.user_id).all()

# Join (explicit)
db.query(User).join(Post).all()
db.query(User).join(User.posts).all()

# Outer join
db.query(User).outerjoin(Post).all()

# Filter by related object
db.query(User).join(Post).filter(Post.title == "FastAPI").all()

# Filter using relationship
db.query(User).filter(User.posts.any(Post.title == "FastAPI")).all()

# Has (for one-to-many reverse)
db.query(Post).filter(Post.user.has(User.name == "John")).all()

# Contains (for many-to-many)
tag = db.query(Tag).first()
db.query(User).filter(User.tags.contains(tag)).all()
```

### Eager Loading
```python
from sqlalchemy.orm import joinedload, selectinload, subqueryload

# Joined load (single query with JOIN)
db.query(User).options(joinedload(User.posts)).all()

# Select in load (separate query with IN)
db.query(User).options(selectinload(User.posts)).all()

# Subquery load (separate query with subquery)
db.query(User).options(subqueryload(User.posts)).all()

# Nested eager loading
db.query(User).options(
    joinedload(User.posts).joinedload(Post.comments)
).all()

# Multiple relationships
db.query(User).options(
    joinedload(User.posts),
    joinedload(User.tags)
).all()
```

## Creating

```python
# Create single object
new_user = User(name="John", email="john@example.com", age=30)
db.add(new_user)
db.commit()
db.refresh(new_user)  # Get the ID and other defaults

# Create without commit
new_user = User(name="John", email="john@example.com")
db.add(new_user)
db.flush()  # Assign ID without committing

# Bulk insert
users = [
    User(name="John", age=30),
    User(name="Jane", age=25),
]
db.bulk_save_objects(users)
db.commit()

# Bulk insert with return_defaults (gets IDs)
db.add_all(users)
db.commit()
```

## Updating

```python
# Update single object
user = db.query(User).filter(User.id == 1).first()
user.name = "John Doe"
db.commit()

# Update with query
db.query(User).filter(User.id == 1).update({"name": "John Doe"})
db.commit()

# Update multiple objects
db.query(User).filter(User.age < 18).update({"status": "minor"})
db.commit()

# Increment value
db.query(User).filter(User.id == 1).update(
    {"age": User.age + 1},
    synchronize_session=False
)
db.commit()

# Bulk update
db.bulk_update_mappings(User, [
    {"id": 1, "name": "John"},
    {"id": 2, "name": "Jane"}
])
db.commit()
```

## Deleting

```python
# Delete single object
user = db.query(User).filter(User.id == 1).first()
db.delete(user)
db.commit()

# Delete with query
db.query(User).filter(User.id == 1).delete()
db.commit()

# Delete multiple
db.query(User).filter(User.age < 18).delete()
db.commit()

# Delete all
db.query(User).delete()
db.commit()
```

## Transactions

```python
from sqlalchemy.exc import SQLAlchemyError

# Basic commit/rollback
try:
    user = User(name="John")
    db.add(user)
    db.commit()
except SQLAlchemyError:
    db.rollback()
    raise

# Using context manager (FastAPI dependency handles this)
def create_user(db: Session):
    try:
        user = User(name="John")
        db.add(user)
        db.commit()
        db.refresh(user)
        return user
    except Exception:
        db.rollback()
        raise

# Nested transactions (savepoints)
from sqlalchemy.orm import sessionmaker
session = SessionLocal()
try:
    session.add(User(name="John"))
    
    savepoint = session.begin_nested()
    try:
        session.add(User(email="invalid"))
    except:
        savepoint.rollback()
    
    session.commit()
finally:
    session.close()
```

## Raw SQL

```python
from sqlalchemy import text

# Execute raw SQL
result = db.execute(text("SELECT * FROM users WHERE age > :age"), {"age": 18})
users = result.fetchall()

# Raw SQL with mapped objects
result = db.query(User).from_statement(
    text("SELECT * FROM users WHERE age > :age")
).params(age=18).all()

# Execute without ORM
from sqlalchemy import create_engine
engine = create_engine(SQLALCHEMY_DATABASE_URL)
with engine.connect() as conn:
    result = conn.execute(text("SELECT * FROM users"))
    for row in result:
        print(row)
```

## Subqueries

```python
from sqlalchemy import select

# Subquery
subq = db.query(
    Post.user_id,
    func.count(Post.id).label('post_count')
).group_by(Post.user_id).subquery()

# Use subquery
db.query(User, subq.c.post_count)\
    .outerjoin(subq, User.id == subq.c.user_id)\
    .all()

# Scalar subquery
subq = db.query(func.count(Post.id))\
    .filter(Post.user_id == User.id)\
    .correlate(User)\
    .as_scalar()

db.query(User.name, subq.label('post_count')).all()

# Exists
db.query(User).filter(
    db.query(Post.id).filter(Post.user_id == User.id).exists()
).all()
```

## Conditional Expressions

```python
from sqlalchemy import case

# Case statement
db.query(
    User.name,
    case(
        (User.age < 18, "Minor"),
        (User.age >= 65, "Senior"),
        else_="Adult"
    ).label("age_group")
).all()

# Case with column reference
db.query(User).filter(
    case(
        (User.status == "active", User.last_login > "2025-01-01"),
        else_=True
    )
).all()
```

## Window Functions

```python
from sqlalchemy import over

# Row number
db.query(
    User.name,
    func.row_number().over(order_by=User.age).label('row_num')
).all()

# Rank with partition
db.query(
    User.name,
    User.department,
    func.rank().over(
        partition_by=User.department,
        order_by=User.salary.desc()
    ).label('rank')
).all()

# Running total
db.query(
    User.name,
    func.sum(User.salary).over(
        order_by=User.hire_date,
        rows=(None, 0)  # Unbounded preceding to current row
    ).label('running_total')
).all()
```

## Database Functions

```python
from sqlalchemy import func

# String functions
db.query(func.lower(User.name)).all()
db.query(func.upper(User.name)).all()
db.query(func.length(User.name)).all()
db.query(func.concat(User.first_name, ' ', User.last_name)).all()

# Date functions
db.query(func.current_date()).scalar()
db.query(func.now()).scalar()
db.query(func.date_part('year', User.created_at)).all()

# Math functions
db.query(func.round(User.salary, 2)).all()
db.query(func.abs(User.balance)).all()

# Coalesce (return first non-null)
db.query(func.coalesce(User.nickname, User.name)).all()
```

## FastAPI Integration

```python
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel

app = FastAPI()

# Pydantic schema
class UserCreate(BaseModel):
    name: str
    email: str
    age: int

class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    
    class Config:
        from_attributes = True

# CREATE
@app.post("/users/", response_model=UserResponse)
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    db_user = User(**user.dict())
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

# READ
@app.get("/users/{user_id}", response_model=UserResponse)
def read_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user

# LIST with pagination
@app.get("/users/", response_model=list[UserResponse])
def list_users(skip: int = 0, limit: int = 10, db: Session = Depends(get_db)):
    users = db.query(User).offset(skip).limit(limit).all()
    return users

# UPDATE
@app.put("/users/{user_id}", response_model=UserResponse)
def update_user(user_id: int, user: UserCreate, db: Session = Depends(get_db)):
    db_user = db.query(User).filter(User.id == user_id).first()
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found")
    
    for key, value in user.dict().items():
        setattr(db_user, key, value)
    
    db.commit()
    db.refresh(db_user)
    return db_user

# DELETE
@app.delete("/users/{user_id}")
def delete_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    
    db.delete(user)
    db.commit()
    return {"message": "User deleted"}
```

## Query Optimization

```python
# Use indexes (in model definition)
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True)
    name = Column(String, index=True)

# Composite index
from sqlalchemy import Index
Index('idx_name_age', User.name, User.age)

# Only load needed columns
db.query(User.id, User.name).all()

# Use contains_eager when joining
from sqlalchemy.orm import contains_eager
db.query(User).join(User.posts).options(contains_eager(User.posts)).all()

# Yield per for large datasets
for user in db.query(User).yield_per(100):
    process(user)

# Disable autoflush for bulk operations
db.autoflush = False
# ... bulk operations
db.flush()
db.autoflush = True
```

## Common Patterns

```python
# Get or create
def get_or_create(db: Session, model, **kwargs):
    instance = db.query(model).filter_by(**kwargs).first()
    if instance:
        return instance, False
    else:
        instance = model(**kwargs)
        db.add(instance)
        db.commit()
        return instance, True

# Upsert (update or insert)
from sqlalchemy.dialects.postgresql import insert

stmt = insert(User).values(email="john@example.com", name="John")
stmt = stmt.on_conflict_do_update(
    index_elements=['email'],
    set_=dict(name="John Updated")
)
db.execute(stmt)
db.commit()

# Batch processing
def batch_process(db: Session, batch_size=1000):
    offset = 0
    while True:
        users = db.query(User).offset(offset).limit(batch_size).all()
        if not users:
            break
        
        for user in users:
            # Process user
            pass
        
        offset += batch_size
        db.commit()
```