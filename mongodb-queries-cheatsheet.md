# MongoDB Queries Cheatsheet

## Database Operations

### Basic Commands
```javascript
// Show all databases
show dbs

// Create/Switch to database
use mydb

// Show current database
db

// Drop database
db.dropDatabase()

// Database stats
db.stats()
```

## Collection Operations

### Create & Drop Collections
```javascript
// Create collection
db.createCollection("users")

// Create with options
db.createCollection("users", {
    capped: true,
    size: 5242880,  // 5MB
    max: 5000
})

// Drop collection
db.users.drop()

// Show all collections
show collections
db.getCollectionNames()

// Collection stats
db.users.stats()

// Rename collection
db.users.renameCollection("customers")
```

## Insert Documents

### Insert Operations
```javascript
// Insert one document
db.users.insertOne({
    name: "John Doe",
    email: "john@example.com",
    age: 30,
    created_at: new Date()
})

// Insert multiple documents
db.users.insertMany([
    { name: "John", email: "john@example.com", age: 30 },
    { name: "Jane", email: "jane@example.com", age: 25 },
    { name: "Bob", email: "bob@example.com", age: 35 }
])

// Insert with specific _id
db.users.insertOne({
    _id: ObjectId("507f1f77bcf86cd799439011"),
    name: "John Doe"
})

// Insert (deprecated, use insertOne/insertMany)
db.users.insert({ name: "John" })
```

## Find Documents

### Basic Queries
```javascript
// Find all documents
db.users.find()

// Find with pretty print
db.users.find().pretty()

// Find one document
db.users.findOne()

// Find with condition
db.users.find({ age: 30 })

// Find with multiple conditions (AND)
db.users.find({ age: 30, name: "John" })

// Find with specific fields (projection)
db.users.find({}, { name: 1, email: 1 })  // Include fields
db.users.find({}, { age: 0 })  // Exclude fields
db.users.find({}, { name: 1, _id: 0 })  // Exclude _id

// Count documents
db.users.countDocuments()
db.users.countDocuments({ age: { $gt: 30 } })
db.users.estimatedDocumentCount()  // Faster, approximate

// Check if exists
db.users.findOne({ email: "john@example.com" }) !== null
```

### Comparison Operators
```javascript
// Equal
db.users.find({ age: 30 })

// Not equal
db.users.find({ age: { $ne: 30 } })

// Greater than
db.users.find({ age: { $gt: 30 } })

// Greater than or equal
db.users.find({ age: { $gte: 30 } })

// Less than
db.users.find({ age: { $lt: 30 } })

// Less than or equal
db.users.find({ age: { $lte: 30 } })

// In array
db.users.find({ age: { $in: [25, 30, 35] } })

// Not in array
db.users.find({ age: { $nin: [25, 30, 35] } })

// Between (range)
db.users.find({ age: { $gte: 25, $lte: 35 } })
```

### Logical Operators
```javascript
// AND (implicit)
db.users.find({ age: 30, name: "John" })

// AND (explicit)
db.users.find({ $and: [
    { age: { $gt: 25 } },
    { age: { $lt: 35 } }
]})

// OR
db.users.find({ $or: [
    { name: "John" },
    { name: "Jane" }
]})

// NOT
db.users.find({ age: { $not: { $gt: 30 } } })

// NOR (not any condition is true)
db.users.find({ $nor: [
    { age: { $lt: 25 } },
    { age: { $gt: 35 } }
]})

// Complex query
db.users.find({
    $or: [
        { name: "John" },
        { name: "Jane" }
    ],
    age: { $gte: 25 }
})
```

### Element Operators
```javascript
// Field exists
db.users.find({ email: { $exists: true } })

// Field doesn't exist
db.users.find({ email: { $exists: false } })

// Type check
db.users.find({ age: { $type: "number" } })
db.users.find({ age: { $type: "int" } })
db.users.find({ name: { $type: "string" } })

// Multiple types
db.users.find({ age: { $type: ["int", "double"] } })
```

### String Pattern Matching
```javascript
// Regex - contains
db.users.find({ name: /john/i })  // Case insensitive
db.users.find({ name: { $regex: "john", $options: "i" } })

// Starts with
db.users.find({ name: /^John/ })

// Ends with
db.users.find({ name: /Doe$/ })

// Contains
db.users.find({ name: /oh/ })

// Text search (requires text index)
db.users.createIndex({ name: "text", bio: "text" })
db.users.find({ $text: { $search: "john doe" } })
db.users.find({ $text: { $search: "\"exact phrase\"" } })
```

### Array Operators
```javascript
// Array contains value
db.users.find({ tags: "mongodb" })

// Array contains all values
db.users.find({ tags: { $all: ["mongodb", "database"] } })

// Array size
db.users.find({ tags: { $size: 3 } })

// Element match (for array of objects)
db.users.find({
    addresses: {
        $elemMatch: { 
            city: "New York",
            zip: { $gte: 10000 }
        }
    }
})

// Array element at position
db.users.find({ "tags.0": "mongodb" })

// Array slice (projection)
db.users.find({}, { tags: { $slice: 2 } })  // First 2
db.users.find({}, { tags: { $slice: -2 } })  // Last 2
db.users.find({}, { tags: { $slice: [1, 3] } })  // Skip 1, limit 3
```

### Sorting, Limiting, Skipping
```javascript
// Sort ascending
db.users.find().sort({ age: 1 })

// Sort descending
db.users.find().sort({ age: -1 })

// Sort by multiple fields
db.users.find().sort({ age: -1, name: 1 })

// Limit results
db.users.find().limit(5)

// Skip results
db.users.find().skip(10)

// Pagination
db.users.find().skip(10).limit(5)

// Sort + Limit + Skip
db.users.find()
    .sort({ age: -1 })
    .skip(10)
    .limit(5)
```

### Nested Documents
```javascript
// Query nested field
db.users.find({ "address.city": "New York" })

// Query nested array
db.users.find({ "addresses.0.city": "New York" })

// Dot notation
db.users.find({ "profile.bio": { $regex: /developer/i } })
```

## Update Documents

### Update Operations
```javascript
// Update one document
db.users.updateOne(
    { name: "John" },
    { $set: { age: 31 } }
)

// Update multiple documents
db.users.updateMany(
    { age: { $lt: 30 } },
    { $set: { status: "young" } }
)

// Replace entire document
db.users.replaceOne(
    { name: "John" },
    { name: "John Doe", age: 31, email: "john@example.com" }
)

// Update (deprecated, use updateOne/updateMany)
db.users.update({ name: "John" }, { $set: { age: 31 } })

// Find and modify
db.users.findOneAndUpdate(
    { name: "John" },
    { $set: { age: 31 } },
    { returnNewDocument: true }
)

// Upsert (insert if not exists)
db.users.updateOne(
    { email: "john@example.com" },
    { $set: { name: "John", age: 30 } },
    { upsert: true }
)
```

### Update Operators
```javascript
// Set field
db.users.updateOne(
    { name: "John" },
    { $set: { age: 31, email: "newemail@example.com" } }
)

// Unset (remove field)
db.users.updateOne(
    { name: "John" },
    { $unset: { age: "" } }
)

// Increment
db.users.updateOne(
    { name: "John" },
    { $inc: { age: 1 } }
)

// Multiply
db.users.updateOne(
    { name: "John" },
    { $mul: { price: 1.1 } }
)

// Rename field
db.users.updateOne(
    { name: "John" },
    { $rename: { "name": "full_name" } }
)

// Set on insert (only during upsert)
db.users.updateOne(
    { email: "john@example.com" },
    {
        $set: { name: "John" },
        $setOnInsert: { created_at: new Date() }
    },
    { upsert: true }
)

// Current date
db.users.updateOne(
    { name: "John" },
    { $currentDate: { last_modified: true } }
)

// Min (update if new value is less)
db.users.updateOne(
    { name: "John" },
    { $min: { age: 25 } }
)

// Max (update if new value is greater)
db.users.updateOne(
    { name: "John" },
    { $max: { age: 40 } }
)
```

### Array Update Operators
```javascript
// Add to array
db.users.updateOne(
    { name: "John" },
    { $push: { tags: "mongodb" } }
)

// Add multiple to array
db.users.updateOne(
    { name: "John" },
    { $push: { tags: { $each: ["mongodb", "database"] } } }
)

// Add to set (no duplicates)
db.users.updateOne(
    { name: "John" },
    { $addToSet: { tags: "mongodb" } }
)

// Remove from array
db.users.updateOne(
    { name: "John" },
    { $pull: { tags: "mongodb" } }
)

// Remove multiple from array
db.users.updateOne(
    { name: "John" },
    { $pull: { tags: { $in: ["mongodb", "database"] } } }
)

// Remove first/last element
db.users.updateOne(
    { name: "John" },
    { $pop: { tags: 1 } }  // Remove last
)
db.users.updateOne(
    { name: "John" },
    { $pop: { tags: -1 } }  // Remove first
)

// Update array element by position
db.users.updateOne(
    { name: "John" },
    { $set: { "tags.0": "updated" } }
)

// Update matched array element ($ operator)
db.users.updateOne(
    { "tags": "mongodb" },
    { $set: { "tags.$": "MongoDB" } }
)

// Update all array elements ($[] operator)
db.users.updateOne(
    { name: "John" },
    { $inc: { "scores.$[]": 10 } }
)

// Update filtered array elements ($[identifier])
db.users.updateOne(
    { name: "John" },
    { $inc: { "scores.$[elem]": 10 } },
    { arrayFilters: [{ elem: { $gte: 80 } }] }
)

// Pull all (remove multiple)
db.users.updateOne(
    { name: "John" },
    { $pullAll: { tags: ["mongodb", "database"] } }
)
```

## Delete Documents

```javascript
// Delete one document
db.users.deleteOne({ name: "John" })

// Delete multiple documents
db.users.deleteMany({ age: { $lt: 30 } })

// Delete all documents
db.users.deleteMany({})

// Find and delete
db.users.findOneAndDelete({ name: "John" })

// Remove (deprecated, use deleteOne/deleteMany)
db.users.remove({ name: "John" })
```

## Aggregation Pipeline

### Basic Aggregation
```javascript
// Simple aggregation
db.users.aggregate([
    { $match: { age: { $gte: 30 } } },
    { $group: { _id: "$city", count: { $sum: 1 } } }
])

// Count by group
db.users.aggregate([
    { $group: { _id: "$age", count: { $sum: 1 } } },
    { $sort: { count: -1 } }
])

// Average
db.users.aggregate([
    { $group: { 
        _id: null, 
        avgAge: { $avg: "$age" } 
    }}
])

// Sum
db.orders.aggregate([
    { $group: { 
        _id: "$user_id", 
        total: { $sum: "$amount" } 
    }}
])

// Min / Max
db.users.aggregate([
    { $group: { 
        _id: "$department", 
        minSalary: { $min: "$salary" },
        maxSalary: { $max: "$salary" }
    }}
])
```

### Aggregation Stages
```javascript
// $match - Filter documents
db.users.aggregate([
    { $match: { age: { $gte: 30 } } }
])

// $group - Group documents
db.users.aggregate([
    { $group: { _id: "$age", count: { $sum: 1 } } }
])

// $project - Select/reshape fields
db.users.aggregate([
    { $project: { name: 1, age: 1, _id: 0 } }
])

// $sort - Sort documents
db.users.aggregate([
    { $sort: { age: -1 } }
])

// $limit - Limit results
db.users.aggregate([
    { $limit: 10 }
])

// $skip - Skip documents
db.users.aggregate([
    { $skip: 10 }
])

// $unwind - Deconstruct array
db.users.aggregate([
    { $unwind: "$tags" }
])

// $lookup - Left outer join
db.orders.aggregate([
    {
        $lookup: {
            from: "users",
            localField: "user_id",
            foreignField: "_id",
            as: "user_info"
        }
    }
])

// $addFields - Add new fields
db.users.aggregate([
    { $addFields: { fullName: { $concat: ["$firstName", " ", "$lastName"] } } }
])

// $count - Count documents
db.users.aggregate([
    { $match: { age: { $gte: 30 } } },
    { $count: "total" }
])

// $out - Write results to collection
db.users.aggregate([
    { $match: { age: { $gte: 30 } } },
    { $out: "adult_users" }
])

// $merge - Merge results to collection
db.users.aggregate([
    { $group: { _id: "$age", count: { $sum: 1 } } },
    { $merge: { into: "age_stats", on: "_id", whenMatched: "replace" } }
])

// $facet - Multiple aggregation pipelines
db.products.aggregate([
    {
        $facet: {
            priceStats: [
                { $group: { _id: null, avgPrice: { $avg: "$price" } } }
            ],
            categoryCount: [
                { $group: { _id: "$category", count: { $sum: 1 } } }
            ]
        }
    }
])

// $bucket - Categorize into buckets
db.users.aggregate([
    {
        $bucket: {
            groupBy: "$age",
            boundaries: [0, 18, 30, 50, 100],
            default: "Other",
            output: { count: { $sum: 1 } }
        }
    }
])
```

### Aggregation Operators
```javascript
// Arithmetic
db.products.aggregate([
    { $project: { 
        name: 1,
        discountedPrice: { $multiply: ["$price", 0.9] }
    }}
])

// String operations
db.users.aggregate([
    { $project: { 
        upperName: { $toUpper: "$name" },
        lowerName: { $toLower: "$name" },
        nameLength: { $strLenCP: "$name" },
        firstName: { $arrayElemAt: [{ $split: ["$name", " "] }, 0] }
    }}
])

// Date operations
db.orders.aggregate([
    { $project: { 
        year: { $year: "$created_at" },
        month: { $month: "$created_at" },
        day: { $dayOfMonth: "$created_at" }
    }}
])

// Conditional
db.users.aggregate([
    { $project: { 
        name: 1,
        ageGroup: {
            $cond: {
                if: { $gte: ["$age", 18] },
                then: "Adult",
                else: "Minor"
            }
        }
    }}
])

// Switch (multiple conditions)
db.users.aggregate([
    { $project: { 
        name: 1,
        ageCategory: {
            $switch: {
                branches: [
                    { case: { $lt: ["$age", 18] }, then: "Minor" },
                    { case: { $lt: ["$age", 65] }, then: "Adult" },
                ],
                default: "Senior"
            }
        }
    }}
])

// Array operations
db.users.aggregate([
    { $project: { 
        name: 1,
        tagCount: { $size: "$tags" },
        firstTag: { $arrayElemAt: ["$tags", 0] },
        hasTag: { $in: ["mongodb", "$tags"] }
    }}
])
```

### Complex Aggregation Examples
```javascript
// Group by multiple fields
db.orders.aggregate([
    {
        $group: {
            _id: { user_id: "$user_id", year: { $year: "$created_at" } },
            total: { $sum: "$amount" },
            count: { $sum: 1 }
        }
    }
])

// Accumulator operators
db.orders.aggregate([
    {
        $group: {
            _id: "$user_id",
            orders: { $push: "$$ROOT" },  // All documents
            orderIds: { $push: "$_id" },  // Just IDs
            firstOrder: { $first: "$amount" },
            lastOrder: { $last: "$amount" },
            avgAmount: { $avg: "$amount" }
        }
    }
])

// Pipeline with multiple stages
db.orders.aggregate([
    { $match: { status: "completed" } },
    { $lookup: {
        from: "users",
        localField: "user_id",
        foreignField: "_id",
        as: "user"
    }},
    { $unwind: "$user" },
    { $group: {
        _id: "$user.city",
        totalRevenue: { $sum: "$amount" },
        orderCount: { $sum: 1 }
    }},
    { $sort: { totalRevenue: -1 } },
    { $limit: 10 }
])
```

## Indexes

### Create Indexes
```javascript
// Create single field index
db.users.createIndex({ email: 1 })  // Ascending
db.users.createIndex({ age: -1 })   // Descending

// Create compound index
db.users.createIndex({ name: 1, age: -1 })

// Create unique index
db.users.createIndex({ email: 1 }, { unique: true })

// Create sparse index (only indexes documents with the field)
db.users.createIndex({ email: 1 }, { sparse: true })

// Create partial index (index subset of documents)
db.users.createIndex(
    { age: 1 },
    { partialFilterExpression: { age: { $gte: 18 } } }
)

// Create TTL index (auto-delete after time)
db.sessions.createIndex(
    { created_at: 1 },
    { expireAfterSeconds: 3600 }  // 1 hour
)

// Text index
db.articles.createIndex({ title: "text", content: "text" })

// Wildcard index
db.users.createIndex({ "userMetadata.$**": 1 })

// 2dsphere index (for geospatial queries)
db.places.createIndex({ location: "2dsphere" })

// Hashed index
db.users.createIndex({ user_id: "hashed" })
```

### Manage Indexes
```javascript
// List all indexes
db.users.getIndexes()

// Drop index
db.users.dropIndex("email_1")
db.users.dropIndex({ email: 1 })

// Drop all indexes (except _id)
db.users.dropIndexes()

// Rebuild indexes
db.users.reIndex()

// Index statistics
db.users.stats().indexSizes
```

### Index Usage
```javascript
// Explain query execution
db.users.find({ email: "john@example.com" }).explain("executionStats")

// Hint (force index usage)
db.users.find({ age: 30 }).hint({ age: 1 })

// Check if index is being used
db.users.find({ email: "john@example.com" }).explain().queryPlanner.winningPlan
```

## Geospatial Queries

```javascript
// Create geospatial index
db.places.createIndex({ location: "2dsphere" })

// Insert location data
db.places.insertOne({
    name: "Central Park",
    location: {
        type: "Point",
        coordinates: [-73.968285, 40.785091]  // [longitude, latitude]
    }
})

// Find near a point
db.places.find({
    location: {
        $near: {
            $geometry: {
                type: "Point",
                coordinates: [-73.968285, 40.785091]
            },
            $maxDistance: 1000  // meters
        }
    }
})

// Find within a polygon
db.places.find({
    location: {
        $geoWithin: {
            $geometry: {
                type: "Polygon",
                coordinates: [[
                    [-73.97, 40.78],
                    [-73.96, 40.78],
                    [-73.96, 40.79],
                    [-73.97, 40.79],
                    [-73.97, 40.78]
                ]]
            }
        }
    }
})

// Find within a circle
db.places.find({
    location: {
        $geoWithin: {
            $centerSphere: [[-73.968285, 40.785091], 1 / 3963.2]  // 1 mile radius
        }
    }
})
```

## Transactions (Replica Set Required)

```javascript
// Start session
const session = db.getMongo().startSession()

// Start transaction
session.startTransaction()

try {
    const usersCol = session.getDatabase("mydb").users
    const ordersCol = session.getDatabase("mydb").orders
    
    // Perform operations
    usersCol.updateOne({ _id: userId }, { $inc: { balance: -100 } })
    ordersCol.insertOne({ user_id: userId, amount: 100 })
    
    // Commit transaction
    session.commitTransaction()
} catch (error) {
    // Abort transaction on error
    session.abortTransaction()
    throw error
} finally {
    session.endSession()
}

// With callback API
session.withTransaction(async () => {
    const usersCol = session.getDatabase("mydb").users
    const ordersCol = session.getDatabase("mydb").orders
    
    await usersCol.updateOne({ _id: userId }, { $inc: { balance: -100 } })
    await ordersCol.insertOne({ user_id: userId, amount: 100 })
})
```

## Bulk Operations

```javascript
// Bulk write
db.users.bulkWrite([
    { insertOne: { document: { name: "John", age: 30 } } },
    { updateOne: {
        filter: { name: "Jane" },
        update: { $set: { age: 26 } }
    }},
    { deleteOne: { filter: { name: "Bob" } } }
])

// Unordered bulk operations
const bulk = db.users.initializeUnorderedBulkOp()
bulk.insert({ name: "John" })
bulk.find({ name: "Jane" }).update({ $set: { age: 26 } })
bulk.find({ name: "Bob" }).remove()
bulk.execute()

// Ordered bulk operations
const bulk = db.users.initializeOrderedBulkOp()
bulk.insert({ name: "John" })
bulk.find({ name: "Jane" }).update({ $set: { age: 26 } })
bulk.execute()
```

## Utility Commands

### Database Info
```javascript
// Database statistics
db.stats()

// Collection statistics
db.users.stats()

// Storage size
db.users.storageSize()

// Total size
db.users.totalSize()

// Server status
db.serverStatus()

// Current operations
db.currentOp()

// Kill operation
db.killOp(opId)

// Get profiling level
db.getProfilingLevel()

// Set profiling level
db.setProfilingLevel(1)  // 0=off, 1=slow, 2=all

// Profiling data
db.system.profile.find().pretty()
```

### Data Import/Export
```bash
# Export collection (command line)
mongoexport --db=mydb --collection=users --out=users.json

# Export to CSV
mongoexport --db=mydb --collection=users --type=csv --fields=name,email --out=users.csv

# Import collection
mongoimport --db=mydb --collection=users --file=users.json

# Import CSV
mongoimport --db=mydb --collection=users --type=csv --headerline --file=users.csv

# Dump entire database
mongodump --db=mydb --out=/backup/

# Restore database
mongorestore --db=mydb /backup/mydb/
```

### User Management
```javascript
// Create user
db.createUser({
    user: "myuser",
    pwd: "mypassword",
    roles: [
        { role: "readWrite", db: "mydb" }
    ]
})

// Create admin user
use admin
db.createUser({
    user: "admin",
    pwd: "adminpassword",
    roles: ["root"]
})

// Drop user
db.dropUser("myuser")

// Update user
db.updateUser("myuser", {
    roles: [
        { role: "dbAdmin", db: "mydb" }
    ]
})

// Change password
db.changeUserPassword("myuser", "newpassword")

// Show users
db.getUsers()
show users
```

## Performance & Optimization

### Query Optimization
```javascript
// Use projection to limit fields
db.users.find({}, { name: 1, email: 1 })

// Use indexes
db.users.createIndex({ email: 1 })

// Use covered queries (all fields in index)
db.users.find({ email: "john@example.com" }, { email: 1, _id: 0 })

// Limit results
db.users.find().limit(100)

// Use aggregation for complex queries
db.users.aggregate([
    { $match: { age: { $gte: 30 } } },
    { $group: { _id: "$city", count: { $sum: 1 } } }
])

// Analyze slow queries
db.setProfilingLevel(1, { slowms: 100 })  // Log queries slower than 100ms
db.system.profile.find().sort({ ts: -1 }).limit(5)

// Connection pooling (driver level)
// Use in application code
```

### Monitoring
```javascript
// Current operations
db.currentOp()

// Server status
db.serverStatus()

// Collection stats
db.users.stats()

// Index stats
db.users.aggregate([{ $indexStats: {} }])

// Explain query
db.users.find({ email: "john@example.com" }).explain("executionStats")
```

## Data Validation

```javascript
// Create collection with validation
db.createCollection("users", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["name", "email", "age"],
            properties: {
                name: {
                    bsonType: "string",
                    description: "must be a string and is required"
                },
                email: {
                    bsonType: "string",
                    pattern: "^.+@.+$",
                    description: "must be a valid email"
                },
                age: {
                    bsonType: "int",
                    minimum: 0,
                    maximum: 150,
                    description: "must be an integer between 0 and 150"
                }
            }
        }
    }
})

// Add validation to existing collection
db.runCommand({
    collMod: "users",
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["email"]
        }
    },
    validationLevel: "moderate",  // or "strict", "off"
    validationAction: "error"     // or "warn"
})
```

## Change Streams (Watch Collections)

```javascript
// Watch collection for changes
const changeStream = db.users.watch()

changeStream.on("change", (change) => {
    console.log(change)
})

// Watch with filter
const changeStream = db.users.watch([
    { $match: { "fullDocument.age": { $gte: 30 } } }
])

// Watch specific operations
const changeStream = db.users.watch([
    { $match: { operationType: "insert" } }
])

// Close change stream
changeStream.close()
```

## Backup & Restore Best Practices

```bash
# Full backup
mongodump --uri="mongodb://localhost:27017/mydb" --out=/backup/

# Backup with authentication
mongodump --uri="mongodb://user:pass@localhost:27017/mydb" --out=/backup/

# Backup specific collection
mongodump --db=mydb --collection=users --out=/backup/

# Compressed backup
mongodump --archive=backup.gz --gzip

# Restore full database
mongorestore --uri="mongodb://localhost:27017" /backup/

# Restore specific database
mongorestore --uri="mongodb://localhost:27017" --nsInclude="mydb.*" /backup/

# Restore with drop (replace existing)
mongorestore --drop --uri="mongodb://localhost:27017" /backup/

# Restore compressed backup
mongorestore --archive=backup.gz --gzip
```

## Schema Design Patterns

### Embedded Documents
```javascript
// One-to-One (Embed)
db.users.insertOne({
    name: "John Doe",
    email: "john@example.com",
    address: {
        street: "123 Main St",
        city: "New York",
        zip: "10001"
    }
})

// One-to-Few (Embed Array)
db.users.insertOne({
    name: "John Doe",
    email: "john@example.com",
    phones: [
        { type: "home", number: "555-1234" },
        { type: "work", number: "555-5678" }
    ]
})
```

### Referenced Documents
```javascript
// One-to-Many (Reference)
// Users collection
db.users.insertOne({
    _id: ObjectId("507f1f77bcf86cd799439011"),
    name: "John Doe"
})

// Posts collection (reference user)
db.posts.insertOne({
    title: "My First Post",
    content: "Hello World",
    user_id: ObjectId("507f1f77bcf86cd799439011")
})

// Query with reference
const user = db.users.findOne({ name: "John Doe" })
const posts = db.posts.find({ user_id: user._id })
```

### Two-Way References
```javascript
// Many-to-Many (Two-way references)
// Users collection
db.users.insertOne({
    _id: ObjectId("user1"),
    name: "John",
    group_ids: [ObjectId("group1"), ObjectId("group2")]
})

// Groups collection
db.groups.insertOne({
    _id: ObjectId("group1"),
    name: "Developers",
    user_ids: [ObjectId("user1"), ObjectId("user2")]
})
```

### Bucket Pattern (Time Series)
```javascript
// Bucket hourly data
db.sensor_data.insertOne({
    sensor_id: 123,
    timestamp: ISODate("2025-11-07T10:00:00Z"),
    measurements: [
        { time: ISODate("2025-11-07T10:00:00Z"), temp: 20.5 },
        { time: ISODate("2025-11-07T10:01:00Z"), temp: 20.6 },
        { time: ISODate("2025-11-07T10:02:00Z"), temp: 20.7 }
    ],
    count: 3,
    avg_temp: 20.6
})
```

### Computed Pattern
```javascript
// Pre-calculate and store frequently accessed data
db.products.insertOne({
    name: "Laptop",
    price: 1000,
    reviews: [
        { rating: 5, comment: "Great!" },
        { rating: 4, comment: "Good" }
    ],
    // Computed fields
    review_count: 2,
    avg_rating: 4.5
})

// Update computed fields
db.products.updateOne(
    { _id: productId },
    {
        $push: { reviews: newReview },
        $inc: { review_count: 1 },
        $set: { avg_rating: newAverage }
    }
)
```

## Advanced Query Techniques

### Cursor Methods
```javascript
// Iterate cursor
db.users.find().forEach(function(doc) {
    print(doc.name)
})

// Map results
db.users.find().map(function(doc) {
    return doc.name
})

// Cursor size
db.users.find().size()

// Has next
const cursor = db.users.find()
if (cursor.hasNext()) {
    const doc = cursor.next()
}

// Batch size
db.users.find().batchSize(100)

// No timeout (prevent cursor timeout)
db.users.find().noCursorTimeout()

// Max time
db.users.find().maxTimeMS(5000)  // 5 seconds max

// Read concern
db.users.find().readConcern("majority")
```

### Distinct Values
```javascript
// Get distinct values
db.users.distinct("age")

// Distinct with query
db.users.distinct("city", { age: { $gte: 30 } })

// Count distinct
db.users.aggregate([
    { $group: { _id: "$age" } },
    { $count: "distinct_ages" }
])
```

### Sample Documents
```javascript
// Random sample
db.users.aggregate([
    { $sample: { size: 10 } }
])
```

### Collation (String Comparison)
```javascript
// Case-insensitive sorting
db.users.find().sort({ name: 1 }).collation({ locale: "en", strength: 2 })

// Create collection with default collation
db.createCollection("users", {
    collation: { locale: "en", strength: 2 }
})

// Create index with collation
db.users.createIndex({ name: 1 }, { collation: { locale: "en", strength: 2 } })
```

## Time Series Collections (MongoDB 5.0+)

```javascript
// Create time series collection
db.createCollection("weather", {
    timeseries: {
        timeField: "timestamp",
        metaField: "sensor_id",
        granularity: "hours"
    }
})

// Insert time series data
db.weather.insertMany([
    {
        sensor_id: 1,
        timestamp: ISODate("2025-11-07T10:00:00Z"),
        temperature: 20.5,
        humidity: 65
    },
    {
        sensor_id: 1,
        timestamp: ISODate("2025-11-07T11:00:00Z"),
        temperature: 21.0,
        humidity: 63
    }
])

// Query time series
db.weather.find({
    timestamp: {
        $gte: ISODate("2025-11-07T00:00:00Z"),
        $lt: ISODate("2025-11-08T00:00:00Z")
    }
})
```

## Capped Collections

```javascript
// Create capped collection (fixed size)
db.createCollection("logs", {
    capped: true,
    size: 5242880,  // 5MB
    max: 5000       // Max 5000 documents
})

// Check if capped
db.logs.isCapped()

// Convert to capped
db.runCommand({
    convertToCapped: "logs",
    size: 5242880
})

// Tailable cursor (like tail -f)
const cursor = db.logs.find().tailable().awaitData()
```

## GridFS (Store Large Files)

```javascript
// Store file (using MongoDB driver)
const bucket = new GridFSBucket(db)

// Upload file
const uploadStream = bucket.openUploadStream("myfile.txt")
fs.createReadStream("./myfile.txt").pipe(uploadStream)

// Download file
const downloadStream = bucket.openDownloadStreamByName("myfile.txt")
downloadStream.pipe(fs.createWriteStream("./downloaded.txt"))

// Delete file
bucket.delete(fileId)

// List files
db.fs.files.find()

// Find file by name
bucket.find({ filename: "myfile.txt" })
```

## Replica Set Commands

```javascript
// Check replica set status
rs.status()

// Initiate replica set
rs.initiate()

// Add member
rs.add("mongodb1.example.net:27017")

// Remove member
rs.remove("mongodb1.example.net:27017")

// Step down primary
rs.stepDown()

// Check if primary
db.isMaster()

// Read preference
db.getMongo().setReadPref("secondaryPreferred")

// Write concern
db.users.insertOne(
    { name: "John" },
    { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
)
```

## Sharding Commands

```javascript
// Enable sharding on database
sh.enableSharding("mydb")

// Shard collection
sh.shardCollection("mydb.users", { user_id: 1 })

// Shard with hashed key
sh.shardCollection("mydb.users", { user_id: "hashed" })

// Check sharding status
sh.status()

// Check if collection is sharded
db.users.getShardDistribution()

// Add shard
sh.addShard("mongodb1.example.net:27017")

// Remove shard
sh.removeShard("shard0001")

// Move chunk
sh.moveChunk("mydb.users", { user_id: 1000 }, "shard0001")
```

## Security Best Practices

```javascript
// Enable authentication
// In mongod.conf:
// security:
//   authorization: enabled

// Create admin user first
use admin
db.createUser({
    user: "admin",
    pwd: "strongpassword",
    roles: ["root"]
})

// Create database-specific user
use mydb
db.createUser({
    user: "appuser",
    pwd: "apppassword",
    roles: [
        { role: "readWrite", db: "mydb" }
    ]
})

// Built-in roles:
// - read
// - readWrite
// - dbAdmin
// - dbOwner
// - userAdmin
// - clusterAdmin
// - root

// Custom role
db.createRole({
    role: "customRole",
    privileges: [
        {
            resource: { db: "mydb", collection: "users" },
            actions: ["find", "insert", "update"]
        }
    ],
    roles: []
})

// Grant role
db.grantRolesToUser("appuser", ["customRole"])

// Revoke role
db.revokeRolesFromUser("appuser", ["customRole"])

// Enable field-level encryption
// Use MongoDB Client-Side Field Level Encryption (CSFLE)
```

## Connection String Examples

```javascript
// Local connection
mongodb://localhost:27017/mydb

// With authentication
mongodb://username:password@localhost:27017/mydb

// Replica set
mongodb://host1:27017,host2:27017,host3:27017/mydb?replicaSet=rs0

// With options
mongodb://username:password@host1:27017,host2:27017/mydb?replicaSet=rs0&authSource=admin&retryWrites=true&w=majority

// MongoDB Atlas
mongodb+srv://username:password@cluster.mongodb.net/mydb

// With SSL/TLS
mongodb://username:password@host:27017/mydb?ssl=true

// Read preference
mongodb://host:27017/mydb?readPreference=secondaryPreferred

// Connection pooling
mongodb://host:27017/mydb?maxPoolSize=50&minPoolSize=10
```

## Common Errors & Solutions

```javascript
// Duplicate key error
// Solution: Check unique indexes and handle in code
try {
    db.users.insertOne({ email: "john@example.com" })
} catch (e) {
    if (e.code === 11000) {
        print("Email already exists")
    }
}

// Document too large (> 16MB)
// Solution: Use GridFS or normalize data

// Too many connections
// Solution: Use connection pooling, close unused connections

// Index too large
// Solution: Use partial indexes, review index strategy

// Slow queries
// Solution: Add indexes, use explain(), optimize queries

// Out of memory
// Solution: Add more RAM, use indexes, limit result sets
```

## Performance Tips

```javascript
// 1. Use indexes on frequently queried fields
db.users.createIndex({ email: 1 })

// 2. Use projection to limit returned fields
db.users.find({}, { name: 1, email: 1 })

// 3. Use covered queries
db.users.find({ email: "john@example.com" }, { email: 1, _id: 0 })

// 4. Limit result sets
db.users.find().limit(100)

// 5. Use aggregation instead of client-side processing
db.users.aggregate([
    { $group: { _id: "$age", count: { $sum: 1 } } }
])

// 6. Avoid $where operator (uses JavaScript, slow)
// Bad:
db.users.find({ $where: "this.age > 30" })
// Good:
db.users.find({ age: { $gt: 30 } })

// 7. Use bulk operations for multiple writes
const bulk = db.users.initializeUnorderedBulkOp()
bulk.insert({ name: "John" })
bulk.insert({ name: "Jane" })
bulk.execute()

// 8. Enable profiling to find slow queries
db.setProfilingLevel(1, { slowms: 100 })

// 9. Use proper schema design (embed vs reference)
// Embed for one-to-few, reference for one-to-many

// 10. Monitor and optimize indexes
db.users.aggregate([{ $indexStats: {} }])
```

## Data Types

```javascript
// String
{ name: "John Doe" }

// Number (Int32, Int64, Double)
{ age: 30, price: 19.99 }
{ count: NumberInt(100) }
{ bigNumber: NumberLong("9223372036854775807") }

// Boolean
{ active: true }

// Date
{ created_at: new Date() }
{ created_at: ISODate("2025-11-07T10:00:00Z") }

// ObjectId
{ _id: ObjectId() }
{ user_id: ObjectId("507f1f77bcf86cd799439011") }

// Array
{ tags: ["mongodb", "database", "nosql"] }

// Embedded Document
{ address: { street: "123 Main St", city: "New York" } }

// Null
{ middleName: null }

// Binary Data
{ data: BinData(0, "base64string==") }

// Regular Expression
{ pattern: /^John/i }

// JavaScript Code
{ code: function() { return 1; } }

// Timestamp (internal use)
{ ts: Timestamp(1634567890, 1) }

// Decimal128 (precise decimal)
{ price: NumberDecimal("19.99") }

// Min/Max Keys
{ minKey: MinKey(), maxKey: MaxKey() }
```

## MongoDB Shell Commands

```javascript
// Clear screen
cls  // Windows
clear  // Mac/Linux

// Exit shell
exit
quit()

// Show help
help

// Show database commands
db.help()

// Show collection commands
db.users.help()

// Load JavaScript file
load("script.js")

// Print JSON
printjson(db.users.findOne())

// Get hostname
hostname()

// Get version
version()

// Sleep
sleep(5000)  // 5 seconds
```