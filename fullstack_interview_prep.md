# Full-Stack Engineer Interview Preparation Guide
## Focus: End-to-End Applications with AI Integration

---

## 1. FRONT-END DEVELOPMENT

### Q1: Explain the Virtual DOM and how React optimizes rendering.
**Answer:** The Virtual DOM is a lightweight JavaScript representation of the actual DOM. React uses it to:
- Create a virtual copy of the UI in memory
- When state changes, React creates a new Virtual DOM tree
- Uses a diffing algorithm to compare old and new trees
- Only updates the changed parts in the real DOM (reconciliation)
- Batch updates for better performance
- This makes React faster than directly manipulating the DOM for each change

**Follow-up capability:** Can explain React Fiber, concurrent rendering, and React 18's automatic batching.

### Q2: What are React Hooks and why were they introduced?
**Answer:** Hooks are functions that let you use state and lifecycle features in functional components.

**Key hooks:**
- `useState` - for state management
- `useEffect` - for side effects (replaces componentDidMount, componentDidUpdate, componentDidUnmount)
- `useContext` - for consuming context
- `useReducer` - for complex state logic
- `useMemo` & `useCallback` - for performance optimization

**Why introduced:**
- Reuse stateful logic without wrapper components
- Avoid "wrapper hell" and HOC complexity
- No need for class components and `this` binding
- Better code organization by related concerns
- Easier to test and maintain

### Q3: Explain the difference between controlled and uncontrolled components.
**Answer:**

**Controlled Components:**
- Form data handled by React component state
- Input value controlled by React via `value` prop
- Changes handled through `onChange` event
```javascript
const [name, setName] = useState('');
<input value={name} onChange={(e) => setName(e.target.value)} />
```

**Uncontrolled Components:**
- Form data handled by DOM itself
- Use `ref` to access current values
- Useful for file inputs or integrating with non-React libraries
```javascript
const inputRef = useRef();
<input ref={inputRef} />
// Access via inputRef.current.value
```

**Best practice:** Use controlled components for better control and validation.

### Q4: How would you optimize a React application's performance?
**Answer:**

1. **Code Splitting:** Use `React.lazy()` and `Suspense` for route-based splitting
2. **Memoization:** 
   - `React.memo()` for component memoization
   - `useMemo()` for expensive calculations
   - `useCallback()` for function references
3. **Virtual Scrolling:** For large lists (react-window, react-virtualized)
4. **Debouncing/Throttling:** For search inputs and scroll events
5. **Image Optimization:** Lazy loading, WebP format, responsive images
6. **Bundle Size:** Analyze with webpack-bundle-analyzer, tree-shaking
7. **State Management:** Keep state close to where it's used
8. **Avoid Inline Functions:** In render methods (causes re-renders)
9. **Key Props:** Proper key usage in lists
10. **Production Build:** Always use optimized production builds

---

## 2. BACK-END DEVELOPMENT

### Q5: Explain RESTful API design principles and best practices.
**Answer:**

**REST Principles:**
- **Stateless:** Each request contains all information needed
- **Client-Server:** Separation of concerns
- **Cacheable:** Responses should indicate if they're cacheable
- **Uniform Interface:** Consistent URL structure and HTTP methods

**Best Practices:**
```
GET    /api/users          - Get all users
GET    /api/users/:id      - Get specific user
POST   /api/users          - Create user
PUT    /api/users/:id      - Update entire user
PATCH  /api/users/:id      - Partial update
DELETE /api/users/:id      - Delete user
```

**Additional practices:**
- Use proper HTTP status codes (200, 201, 400, 401, 404, 500)
- Versioning: `/api/v1/users`
- Filtering: `/api/users?role=admin&status=active`
- Pagination: `/api/users?page=2&limit=20`
- HATEOAS: Include links to related resources
- Consistent error responses
- Security: Authentication, rate limiting, input validation

### Q6: What is the difference between authentication and authorization?
**Answer:**

**Authentication (AuthN):** "Who are you?"
- Verifies identity of a user
- Methods: Username/password, OAuth, JWT, biometrics
- Happens first in the security flow

**Authorization (AuthZ):** "What can you do?"
- Determines what resources a user can access
- Happens after authentication
- Methods: Role-Based Access Control (RBAC), Access Control Lists (ACL)

**Example:**
```javascript
// Authentication - Verify user
const token = jwt.sign({ userId: user.id }, SECRET_KEY);

// Authorization - Check permissions
const authorize = (requiredRole) => {
  return (req, res, next) => {
    if (req.user.role !== requiredRole) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
};
```

### Q7: Explain database indexing and when to use it.
**Answer:**

**What is indexing:**
- Data structure (usually B-tree or Hash) that improves query speed
- Creates a separate lookup table with pointers to actual data
- Trade-off: Faster reads, slower writes

**When to use:**
- Columns frequently used in WHERE clauses
- Foreign keys for JOIN operations
- Columns used in ORDER BY or GROUP BY
- Unique constraints

**When NOT to use:**
- Small tables (full scan might be faster)
- Columns with frequent updates
- Low cardinality columns (few unique values)

**Example (MongoDB):**
```javascript
// Single field index
db.users.createIndex({ email: 1 });

// Compound index
db.orders.createIndex({ userId: 1, createdAt: -1 });

// Text index for search
db.products.createIndex({ name: "text", description: "text" });
```

### Q8: How do you handle errors in a Node.js/Express application?
**Answer:**

**Approach:**

1. **Try-Catch for Async/Await:**
```javascript
app.get('/api/users/:id', async (req, res, next) => {
  try {
    const user = await User.findById(req.params.id);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    res.json(user);
  } catch (error) {
    next(error); // Pass to error handler
  }
});
```

2. **Custom Error Classes:**
```javascript
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
  }
}
```

3. **Global Error Handler:**
```javascript
app.use((err, req, res, next) => {
  const statusCode = err.statusCode || 500;
  const message = err.message || 'Internal Server Error';
  
  // Log error
  console.error(err.stack);
  
  res.status(statusCode).json({
    status: 'error',
    statusCode,
    message,
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
  });
});
```

4. **Unhandled Promise Rejections:**
```javascript
process.on('unhandledRejection', (err) => {
  console.error('Unhandled Rejection:', err);
  process.exit(1);
});
```

---

## 3. API DESIGN & INTEGRATION

### Q9: What is GraphQL and how does it differ from REST?
**Answer:**

**GraphQL:**
- Query language for APIs
- Single endpoint (`/graphql`)
- Client specifies exactly what data it needs
- Strongly typed schema

**Key Differences:**

| Aspect | REST | GraphQL |
|--------|------|---------|
| Endpoints | Multiple endpoints | Single endpoint |
| Over-fetching | Common | Client controls data |
| Under-fetching | Multiple requests | Single request |
| Versioning | URL versioning | Schema evolution |
| Caching | HTTP caching | More complex |

**GraphQL Example:**
```graphql
query {
  user(id: "123") {
    name
    email
    posts {
      title
      comments {
        text
      }
    }
  }
}
```

**When to use GraphQL:**
- Complex data relationships
- Multiple clients with different needs
- Need to minimize network requests

**When to use REST:**
- Simple CRUD operations
- Need HTTP caching
- File uploads/downloads

### Q10: How would you design a rate limiting system for an API?
**Answer:**

**Approaches:**

1. **Token Bucket Algorithm:**
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
  message: 'Too many requests, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
});

app.use('/api/', limiter);
```

2. **Redis-based (Distributed System):**
```javascript
const redis = require('redis');
const client = redis.createClient();

const rateLimiter = async (req, res, next) => {
  const key = `rate_limit:${req.ip}`;
  const requests = await client.incr(key);
  
  if (requests === 1) {
    await client.expire(key, 60); // 60 seconds window
  }
  
  if (requests > 10) {
    return res.status(429).json({ error: 'Rate limit exceeded' });
  }
  
  next();
};
```

3. **Different Limits for Different Routes:**
```javascript
const strictLimiter = rateLimit({ windowMs: 60000, max: 5 });
const normalLimiter = rateLimit({ windowMs: 60000, max: 100 });

app.post('/api/auth/login', strictLimiter, loginController);
app.get('/api/users', normalLimiter, getUsersController);
```

**Considerations:**
- Return appropriate headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`
- Different limits for authenticated vs anonymous users
- Whitelist trusted IPs
- Use Redis for distributed systems

### Q11: Explain CORS and how to handle it in a full-stack application.
**Answer:**

**CORS (Cross-Origin Resource Sharing):**
- Security feature that restricts web pages from making requests to a different domain
- Browser blocks requests if server doesn't explicitly allow them

**Same-Origin Policy:**
- Protocol, domain, and port must match
- `http://example.com:80` ≠ `https://example.com:443`

**Implementation:**

**Backend (Express):**
```javascript
const cors = require('cors');

// Simple usage (allow all)
app.use(cors());

// Production configuration
const corsOptions = {
  origin: ['https://myapp.com', 'https://admin.myapp.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true, // Allow cookies
  maxAge: 86400 // Preflight cache duration
};

app.use(cors(corsOptions));

// Dynamic origin
app.use(cors({
  origin: (origin, callback) => {
    const allowedOrigins = process.env.ALLOWED_ORIGINS.split(',');
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  }
}));
```

**Frontend (React with credentials):**
```javascript
fetch('https://api.example.com/data', {
  method: 'GET',
  credentials: 'include', // Send cookies
  headers: {
    'Content-Type': 'application/json',
  }
});
```

**Preflight Requests:**
- Browser sends OPTIONS request first for complex requests
- Server must respond with allowed methods and headers

---

## 4. AI INTEGRATION & BACK-END AI LOGIC

### Q12: How would you integrate an AI model (like a sentiment analysis API) into your application?
**Answer:**

**Architecture:**

1. **API Wrapper Service:**
```javascript
// ai-service.js
const axios = require('axios');

class AIService {
  constructor() {
    this.apiKey = process.env.AI_API_KEY;
    this.baseURL = 'https://api.ai-provider.com';
  }

  async analyzeSentiment(text) {
    try {
      const response = await axios.post(
        `${this.baseURL}/sentiment`,
        { text },
        { 
          headers: { 'Authorization': `Bearer ${this.apiKey}` },
          timeout: 10000 // 10 second timeout
        }
      );
      return response.data;
    } catch (error) {
      console.error('AI Service Error:', error);
      throw new Error('Sentiment analysis failed');
    }
  }

  async batchAnalyze(texts) {
    // Implement batching for efficiency
    const batchSize = 10;
    const results = [];
    
    for (let i = 0; i < texts.length; i += batchSize) {
      const batch = texts.slice(i, i + batchSize);
      const batchResults = await Promise.all(
        batch.map(text => this.analyzeSentiment(text))
      );
      results.push(...batchResults);
    }
    
    return results;
  }
}

module.exports = new AIService();
```

2. **Controller with Caching:**
```javascript
// sentiment-controller.js
const aiService = require('./ai-service');
const redis = require('redis');
const client = redis.createClient();

exports.analyzeSentiment = async (req, res) => {
  try {
    const { text } = req.body;
    
    // Check cache first
    const cacheKey = `sentiment:${Buffer.from(text).toString('base64')}`;
    const cached = await client.get(cacheKey);
    
    if (cached) {
      return res.json({ 
        result: JSON.parse(cached), 
        cached: true 
      });
    }
    
    // Call AI service
    const result = await aiService.analyzeSentiment(text);
    
    // Cache for 1 hour
    await client.setex(cacheKey, 3600, JSON.stringify(result));
    
    res.json({ result, cached: false });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

3. **Queue for Long-Running Tasks:**
```javascript
// Using Bull queue
const Queue = require('bull');
const aiQueue = new Queue('ai-processing', {
  redis: { host: 'localhost', port: 6379 }
});

// Add job to queue
exports.queueAIAnalysis = async (req, res) => {
  const job = await aiQueue.add({
    userId: req.user.id,
    data: req.body.data
  });
  
  res.json({ jobId: job.id, status: 'queued' });
};

// Process job
aiQueue.process(async (job) => {
  const result = await aiService.analyze(job.data.data);
  await saveResultToDatabase(job.data.userId, result);
  return result;
});
```

**Best Practices:**
- Implement retry logic with exponential backoff
- Rate limiting to AI API
- Fallback mechanisms
- Monitor API usage and costs
- Cache frequently requested results
- Use webhooks for async processing
- Store raw inputs and outputs for debugging

### Q13: Explain how you would implement a real-time AI-powered feature (e.g., live chat with AI suggestions).
**Answer:**

**Architecture:**

1. **WebSocket Connection:**
```javascript
// Backend (Socket.io)
const io = require('socket.io')(server, {
  cors: { origin: process.env.CLIENT_URL }
});

io.use(async (socket, next) => {
  // Authenticate
  const token = socket.handshake.auth.token;
  const user = await verifyToken(token);
  socket.user = user;
  next();
});

io.on('connection', (socket) => {
  console.log(`User connected: ${socket.user.id}`);
  
  socket.on('message', async (data) => {
    try {
      // Store message
      const message = await Message.create({
        userId: socket.user.id,
        text: data.text,
        timestamp: Date.now()
      });
      
      // Broadcast to room
      io.to(data.roomId).emit('new-message', message);
      
      // Get AI suggestions
      const suggestions = await aiService.getSuggestions(data.text);
      socket.emit('ai-suggestions', suggestions);
      
    } catch (error) {
      socket.emit('error', { message: error.message });
    }
  });
  
  socket.on('disconnect', () => {
    console.log(`User disconnected: ${socket.user.id}`);
  });
});
```

2. **Frontend (React):**
```javascript
import { useEffect, useState } from 'react';
import io from 'socket.io-client';

function ChatComponent() {
  const [socket, setSocket] = useState(null);
  const [messages, setMessages] = useState([]);
  const [suggestions, setSuggestions] = useState([]);
  
  useEffect(() => {
    const newSocket = io('http://localhost:5000', {
      auth: { token: localStorage.getItem('token') }
    });
    
    newSocket.on('new-message', (message) => {
      setMessages(prev => [...prev, message]);
    });
    
    newSocket.on('ai-suggestions', (suggestions) => {
      setSuggestions(suggestions);
    });
    
    setSocket(newSocket);
    
    return () => newSocket.close();
  }, []);
  
  const sendMessage = (text) => {
    socket.emit('message', { text, roomId: 'room1' });
  };
  
  return (
    <div>
      {/* Chat UI */}
      {suggestions.length > 0 && (
        <div className="suggestions">
          {suggestions.map(s => (
            <button key={s.id} onClick={() => sendMessage(s.text)}>
              {s.text}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

3. **Debouncing AI Calls:**
```javascript
let aiDebounceTimer;

socket.on('typing', async (data) => {
  clearTimeout(aiDebounceTimer);
  
  aiDebounceTimer = setTimeout(async () => {
    const suggestions = await aiService.getSuggestions(data.text);
    socket.emit('ai-suggestions', suggestions);
  }, 500); // Wait 500ms after user stops typing
});
```

**Considerations:**
- Use message queues for high traffic
- Implement connection pooling
- Handle reconnection logic
- Stream responses for large AI outputs
- Monitor WebSocket performance
- Implement presence indicators

### Q14: How would you handle AI model deployment and versioning?
**Answer:**

**Deployment Strategies:**

1. **Containerized Deployment (Docker):**
```dockerfile
# Dockerfile for AI Model
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY model/ ./model/
COPY app.py .

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

2. **API Gateway Pattern:**
```javascript
// model-router.js
const express = require('express');
const router = express.Router();

const modelVersions = {
  'v1': 'http://model-v1:8000',
  'v2': 'http://model-v2:8000',
  'v3': 'http://model-v3:8000'
};

router.post('/predict', async (req, res) => {
  const version = req.header('X-Model-Version') || 'v2';
  const modelUrl = modelVersions[version];
  
  if (!modelUrl) {
    return res.status(400).json({ error: 'Invalid model version' });
  }
  
  const response = await axios.post(`${modelUrl}/predict`, req.body);
  res.json(response.data);
});
```

3. **A/B Testing:**
```javascript
const assignModelVersion = (userId) => {
  // Consistent assignment based on user ID
  const hash = crypto.createHash('md5').update(userId).digest('hex');
  const value = parseInt(hash.substring(0, 8), 16);
  
  // 80% v2, 20% v3
  return value % 100 < 80 ? 'v2' : 'v3';
};

router.post('/predict', async (req, res) => {
  const version = assignModelVersion(req.user.id);
  const modelUrl = modelVersions[version];
  
  // Log for analysis
  await analytics.track({
    userId: req.user.id,
    event: 'model_prediction',
    properties: { version }
  });
  
  const response = await axios.post(`${modelUrl}/predict`, req.body);
  res.json({ ...response.data, modelVersion: version });
});
```

4. **Canary Deployment:**
```javascript
const getModelEndpoint = () => {
  const random = Math.random();
  
  if (random < 0.05) { // 5% traffic to new model
    return modelVersions.v3;
  }
  return modelVersions.v2;
};
```

**Monitoring:**
```javascript
const prometheus = require('prom-client');

const predictionLatency = new prometheus.Histogram({
  name: 'model_prediction_latency_seconds',
  help: 'Model prediction latency',
  labelNames: ['version', 'status']
});

const predictWithMonitoring = async (version, data) => {
  const end = predictionLatency.startTimer({ version });
  
  try {
    const result = await callModel(version, data);
    end({ status: 'success' });
    return result;
  } catch (error) {
    end({ status: 'error' });
    throw error;
  }
};
```

---

## 5. CI/CD & DEVOPS

### Q15: Explain a complete CI/CD pipeline for a full-stack application.
**Answer:**

**Pipeline Stages:**

1. **Source Control (Git):**
```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: |
        npm ci
        cd client && npm ci
    
    - name: Run linter
      run: npm run lint
    
    - name: Run tests
      run: npm test -- --coverage
    
    - name: Upload coverage
      uses: codecov/codecov-action@v2

  build:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Build Docker image
      run: |
        docker build -t myapp:${{ github.sha }} .
    
    - name: Push to registry
      run: |
        echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
        docker push myapp:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Deploy to production
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.PROD_HOST }}
        username: ${{ secrets.PROD_USER }}
        key: ${{ secrets.SSH_PRIVATE_KEY }}
        script: |
          cd /app
          docker-compose pull
          docker-compose up -d
          docker system prune -f
```

2. **Docker Configuration:**
```dockerfile
# Multi-stage build
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# Production image
FROM node:18-alpine

WORKDIR /app

COPY --from=builder /app/build ./build
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

EXPOSE 3000

USER node

CMD ["node", "server.js"]
```

3. **Docker Compose:**
```yaml
version: '3.8'

services:
  app:
    image: myapp:latest
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DB_HOST=mongo
    depends_on:
      - mongo
      - redis
    restart: unless-stopped

  mongo:
    image: mongo:5
    volumes:
      - mongo-data:/data/db
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app

volumes:
  mongo-data:
```

4. **Health Checks:**
```javascript
// healthcheck.js
app.get('/health', async (req, res) => {
  const health = {
    uptime: process.uptime(),
    timestamp: Date.now(),
    status: 'OK'
  };

  try {
    // Check database
    await mongoose.connection.db.admin().ping();
    health.database = 'OK';

    // Check Redis
    await redis.ping();
    health.redis = 'OK';

    res.status(200).json(health);
  } catch (error) {
    health.status = 'ERROR';
    health.error = error.message;
    res.status(503).json(health);
  }
});
```

**Best Practices:**
- Automated testing at each stage
- Environment-specific configurations
- Secrets management (never commit secrets)
- Rolling deployments with zero downtime
- Automated rollbacks on failure
- Database migrations as part of pipeline
- Monitoring and alerting integration

### Q16: How do you ensure security in a full-stack application?
**Answer:**

**Frontend Security:**

1. **XSS Prevention:**
```javascript
// React automatically escapes by default
<div>{userInput}</div> // Safe

// Dangerous (avoid):
<div dangerouslySetInnerHTML={{__html: userInput}} />

// If needed, sanitize:
import DOMPurify from 'dompurify';
const clean = DOMPurify.sanitize(userInput);
```

2. **CSRF Protection:**
```javascript
// Backend - Generate CSRF token
const csrf = require('csurf');
const csrfProtection = csrf({ cookie: true });

app.get('/form', csrfProtection, (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});

// Frontend - Send token
fetch('/api/data', {
  method: 'POST',
  headers: {
    'CSRF-Token': csrfToken
  }
});
```

**Backend Security:**

1. **Input Validation:**
```javascript
const { body, validationResult } = require('express-validator');

app.post('/api/users',
  [
    body('email').isEmail().normalizeEmail(),
    body('password').isLength({ min: 8 }).matches(/^(?=.*[A-Z])(?=.*[0-9])/),
    body('name').trim().escape()
  ],
  (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    // Process valid data
  }
);
```

2. **Authentication with JWT:**
```javascript
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');

// Login
app.post('/api/login', async (req, res) => {
  const { email, password } = req.body;
  
  const user = await User.findOne({ email });
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  const validPassword = await bcrypt.compare(password, user.password);
  if (!validPassword) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  const token = jwt.sign(
    { userId: user._id, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: '1h' }
  );
  
  // Send as httpOnly cookie
  res.cookie('token', token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 3600000
  });
  
  res.json({ user: { id: user._id, email: user.email } });
});

// Middleware
const authenticate = (req, res, next) => {
  const token = req.cookies.token;
  
  if (!token) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};
```

3. **SQL Injection Prevention:**
```javascript
// NEVER do this:
const query = `SELECT * FROM users WHERE email = '${email}'`;

// Use parameterized queries (Mongoose does this automatically)
const user = await User.findOne({ email: email });

// For raw SQL:
const result = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);
```

4. **Security Headers:**
```javascript
const helmet = require('helmet');

app.use(helmet());
// Sets: X-Frame-Options, X-Content-Type-Options, 
// Strict-Transport-Security, etc.

// Custom CSP
app.use(
  helmet.contentSecurityPolicy({
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'", "trusted-cdn.com"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
    },
  })
);
```

5. **Environment Variables:**
```javascript
// .env (NEVER commit this)
DB_CONNECTION=mongodb://localhost:27017/myapp
JWT_SECRET=your-super-secret-key-here
API_KEY=external-api-key

// Load in app
require('dotenv').config();
const dbConnection = process.env.DB_CONNECTION;
```

**Other Security Measures:**
- Regular dependency updates (`npm audit`)
- Rate limiting on sensitive endpoints
- File upload validation (type, size)
- Logging and monitoring
- Regular security audits
- HTTPS in production
- Database encryption at rest

---

## 6. DATABASE & DATA MANAGEMENT

### Q17: Explain the difference between SQL and NoSQL databases. When would you use each?
**Answer:**

**SQL (Relational) - e.g., PostgreSQL, MySQL:**

**Characteristics:**
- Structured data with predefined schema
- Tables with rows and columns
- ACID compliant (Atomicity, Consistency, Isolation, Durability)
- Strong relationships with foreign keys
- SQL query language
- Vertical scaling (scale up)

**When to use:**
- Complex queries and joins needed
- Data integrity is critical (banking, e-commerce)
- Structured data with relationships
- Need ACID transactions
- Example: User profiles with orders, payments, addresses

**NoSQL - e.g., MongoDB, Redis, Cassandra:**

**Characteristics:**
- Flexible/dynamic schema
- Document, key-value, column, or graph based
- Eventually consistent (BASE)
- Horizontal scaling (scale out)
- Optimized for specific access patterns

**When to use:**
- Rapidly changing data structure
- Need horizontal scaling
- Large volumes of unstructured data
- Real-time applications
- Example: Social media feeds, logging, session storage

**Comparison Example:**

```javascript
// SQL (PostgreSQL with Sequelize)
const User = sequelize.define('User', {
  id: { type: DataTypes.INTEGER, primaryKey: true },
  email: { type: DataTypes.STRING, unique: true },
  name: DataTypes.STRING
});

const Post = sequelize.define('Post', {
  title: DataTypes.STRING,
  content: DataTypes.TEXT,
  userId: {
    type: DataTypes.INTEGER,
    references: { model: User, key: 'id' }
  }
});

// Query with join
const userWithPosts = await User.findOne({
  where: { email: 'user@example.com' },
  include: [Post]
});

// NoSQL (MongoDB with Mongoose)
const UserSchema = new mongoose.Schema({
  email: { type: String, unique: true },
  name: String,
  posts: [{
    title: String,
    content: String,
    createdAt: Date
  }]
});

// Embedded documents - single query
const user = await User.findOne({ email: 'user@example.com' });
```

**Hybrid Approach:**
Use both! SQL for transactional data, NoSQL for caching/sessions.

### Q18: What are database transactions and when are they important?
**Answer:**

**Transaction:** A sequence of database operations that execute as a single unit.

**ACID Properties:**
- **Atomicity:** All or nothing - if one operation fails, all rollback
- **Consistency:** Database moves from one valid state to another
- **Isolation:** Concurrent transactions don't interfere
- **Durability:** Committed changes are permanent

**Example Scenario - Money Transfer:**
```javascript
// Without transaction (DANGEROUS!)
async function transferMoney(fromId, toId, amount) {
  const fromAccount = await Account.findById(fromId);
  fromAccount.balance -= amount;
  await fromAccount.save(); // What if this succeeds but next fails?
  
  const toAccount = await Account.findById(toId);
  toAccount.balance += amount;
  await toAccount.save(); // CRASH! Money disappeared!
}

// With transaction (SAFE)
async function transferMoney(fromId, toId, amount) {
  const session = await mongoose.startSession();
  session.startTransaction();
  
  try {
    const fromAccount = await Account.findById(fromId).session(session);
    if (fromAccount.balance < amount) {
      throw new Error('Insufficient funds');
    }
    
    fromAccount.balance -= amount;
    await fromAccount.save({ session });
    
    const toAccount = await Account.findById(toId).session(session);
    toAccount.balance += amount;
    await toAccount.save({ session });
    
    await session.commitTransaction(); // All succeed together
    session.endSession();
    
    return { success: true };
  } catch (error) {
    await session.abortTransaction(); // All rollback together
    session.endSession();
    throw error;
  }
}
```

**SQL Transaction:**
```javascript
const sequelize = require('./database');

async function createUserWithProfile(userData, profileData) {
  const transaction = await sequelize.transaction();
  
  try {
    const user = await User.create(userData, { transaction });
    
    profileData.userId = user.id;
    await Profile.create(profileData, { transaction });
    
    await transaction.commit();
    return user;
  } catch (error) {
    await transaction.rollback();
    throw error;
  }
}
```

**When to use:**
- Financial transactions
- Multi-step operations that must succeed together
- Data consistency is critical
- Inventory management
- Booking systems

### Q19: How do you handle database migrations in production?
**Answer:**

**Migration Strategy:**

1. **Using Migration Tools (Sequelize example):**
```javascript
// migrations/20240101-create-users.js
module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('Users', {
      id: {
        type: Sequelize.INTEGER,
        primaryKey: true,
        autoIncrement: true
      },
      email: {
        type: Sequelize.STRING,
        unique: true,
        allowNull: false
      },
      createdAt: Sequelize.DATE,
      updatedAt: Sequelize.DATE
    });
    
    // Add index
    await queryInterface.addIndex('Users', ['email']);
  },
  
  down: async (queryInterface, Sequelize) => {
    await queryInterface.dropTable('Users');
  }
};

// Run migration
npx sequelize-cli db:migrate

// Rollback
npx sequelize-cli db:migrate:undo
```

2. **MongoDB Migrations:**
```javascript
// migrations/001-add-user-roles.js
module.exports = {
  async up(db) {
    await db.collection('users').updateMany(
      { role: { $exists: false } },
      { $set: { role: 'user' } }
    );
    
    await db.collection('users').createIndex({ role: 1 });
  },
  
  async down(db) {
    await db.collection('users').updateMany(
      {},
      { $unset: { role: '' } }
    );
    
    await db.collection('users').dropIndex('role_1');
  }
};
```

**Best Practices:**

1. **Always write reversible migrations** (up and down)
2. **Test migrations on staging** before production
3. **Backup database** before major migrations
4. **Version control** all migration files
5. **Zero-downtime migrations:**
   - Add new column (nullable)
   - Deploy code that writes to both old and new
   - Migrate data
   - Deploy code that only uses new column
   - Remove old column

**Example Zero-Downtime:**
```javascript
// Step 1: Add new column
await queryInterface.addColumn('Users', 'full_name', {
  type: Sequelize.STRING,
  allowNull: true // Initially nullable
});

// Step 2: Backfill data
await queryInterface.sequelize.query(`
  UPDATE Users 
  SET full_name = CONCAT(first_name, ' ', last_name)
  WHERE full_name IS NULL
`);

// Step 3: Make it required (after code deployment)
await queryInterface.changeColumn('Users', 'full_name', {
  type: Sequelize.STRING,
  allowNull: false
});

// Step 4: Remove old columns (after verification)
await queryInterface.removeColumn('Users', 'first_name');
await queryInterface.removeColumn('Users', 'last_name');
```

---

## 7. SYSTEM DESIGN & ARCHITECTURE

### Q20: How would you design a scalable URL shortener service?
**Answer:**

**Requirements:**
- Shorten long URLs to unique short codes
- Redirect short URLs to original URLs
- Track click analytics
- Handle high read/write traffic

**Architecture:**

```
Client → Load Balancer → API Servers → Cache (Redis) → Database (PostgreSQL)
                                     ↓
                                  Analytics Queue → Analytics Service
```

**Database Schema:**
```sql
CREATE TABLE urls (
  id BIGSERIAL PRIMARY KEY,
  short_code VARCHAR(10) UNIQUE NOT NULL,
  original_url TEXT NOT NULL,
  user_id BIGINT,
  created_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP,
  clicks BIGINT DEFAULT 0
);

CREATE INDEX idx_short_code ON urls(short_code);
CREATE INDEX idx_user_id ON urls(user_id);
```

**Implementation:**

1. **Generate Short Code:**
```javascript
const crypto = require('crypto');

class URLShortener {
  constructor() {
    this.baseUrl = 'https://short.ly/';
    this.alphabet = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
  }
  
  // Base62 encoding
  encodeBase62(num) {
    if (num === 0) return this.alphabet[0];
    
    let encoded = '';
    while (num > 0) {
      encoded = this.alphabet[num % 62] + encoded;
      num = Math.floor(num / 62);
    }
    return encoded;
  }
  
  async createShortURL(originalUrl, userId = null) {
    // Check if URL already exists
    const existing = await redis.get(`url:${originalUrl}`);
    if (existing) return existing;
    
    // Generate unique ID
    const id = await this.getNextId();
    const shortCode = this.encodeBase62(id);
    
    // Store in database
    await URL.create({
      id,
      shortCode,
      originalUrl,
      userId
    });
    
    // Cache mapping (both directions)
    await redis.setex(`short:${shortCode}`, 3600, originalUrl);
    await redis.setex(`url:${originalUrl}`, 3600, shortCode);
    
    return `${this.baseUrl}${shortCode}`;
  }
  
  async redirect(shortCode) {
    // Check cache first
    let originalUrl = await redis.get(`short:${shortCode}`);
    
    if (!originalUrl) {
      // Cache miss - query database
      const url = await URL.findOne({ where: { shortCode } });
      if (!url) throw new Error('URL not found');
      
      originalUrl = url.originalUrl;
      
      // Update cache
      await redis.setex(`short:${shortCode}`, 3600, originalUrl);
    }
    
    // Async: increment click counter
    this.trackClick(shortCode);
    
    return originalUrl;
  }
  
  async trackClick(shortCode) {
    // Add to queue for async processing
    await analyticsQueue.add({
      shortCode,
      timestamp: Date.now(),
      // Additional metadata: IP, user agent, etc.
    });
    
    // Increment Redis counter
    await redis.incr(`clicks:${shortCode}`);
  }
  
  async getNextId() {
    // Use Redis for distributed ID generation
    return await redis.incr('url:id:counter');
  }
}
```

2. **API Endpoints:**
```javascript
const shortener = new URLShortener();

// Create short URL
app.post('/api/shorten', authenticate, async (req, res) => {
  try {
    const { url } = req.body;
    
    // Validate URL
    if (!isValidURL(url)) {
      return res.status(400).json({ error: 'Invalid URL' });
    }
    
    const shortUrl = await shortener.createShortURL(url, req.user.id);
    res.json({ shortUrl });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Redirect
app.get('/:shortCode', async (req, res) => {
  try {
    const originalUrl = await shortener.redirect(req.params.shortCode);
    res.redirect(301, originalUrl);
  } catch (error) {
    res.status(404).send('URL not found');
  }
});

// Analytics
app.get('/api/analytics/:shortCode', authenticate, async (req, res) => {
  const clicks = await redis.get(`clicks:${req.params.shortCode}`) || 0;
  res.json({ clicks: parseInt(clicks) });
});
```

**Scaling Strategies:**
1. **Caching:** Redis for hot URLs
2. **Database sharding:** Partition by short_code first character
3. **CDN:** Cache redirect responses
4. **Rate limiting:** Prevent abuse
5. **Analytics queue:** Async processing to avoid blocking
6. **Read replicas:** For analytics queries
7. **Monitoring:** Track cache hit rate, response times

### Q21: Design a real-time notification system.
**Answer:**

**Requirements:**
- Push notifications to web/mobile clients
- Support multiple notification types
- Handle offline users
- Ensure delivery

**Architecture:**

```
Event Source → Message Queue (RabbitMQ/Kafka) → Notification Service
                                                        ↓
                                                  WebSocket Server
                                                        ↓
                                                    Clients
                                                        ↓
                                                  Push Service (FCM/APNs)
```

**Implementation:**

1. **Notification Service:**
```javascript
const amqp = require('amqplib');
const io = require('socket.io');

class NotificationService {
  constructor() {
    this.activeConnections = new Map(); // userId -> socket
  }
  
  async initialize() {
    // Connect to message queue
    this.connection = await amqp.connect(process.env.RABBITMQ_URL);
    this.channel = await this.connection.createChannel();
    
    await this.channel.assertQueue('notifications', { durable: true });
    
    // Consume notifications
    this.channel.consume('notifications', async (msg) => {
      const notification = JSON.parse(msg.content.toString());
      await this.processNotification(notification);
      this.channel.ack(msg);
    });
  }
  
  async processNotification(notification) {
    const { userId, type, data } = notification;
    
    // Store in database
    await Notification.create({
      userId,
      type,
      data,
      read: false,
      createdAt: new Date()
    });
    
    // Send to online user
    const socket = this.activeConnections.get(userId);
    if (socket) {
      socket.emit('notification', notification);
    } else {
      // User offline - send push notification
      await this.sendPushNotification(userId, notification);
    }
  }
  
  async sendPushNotification(userId, notification) {
    const user = await User.findById(userId);
    
    if (user.pushToken) {
      // Firebase Cloud Messaging
      await admin.messaging().send({
        token: user.pushToken,
        notification: {
          title: notification.title,
          body: notification.body
        },
        data: notification.data
      });
    }
  }
  
  registerConnection(userId, socket) {
    this.activeConnections.set(userId, socket);
    
    socket.on('disconnect', () => {
      this.activeConnections.delete(userId);
    });
  }
}

const notificationService = new NotificationService();
notificationService.initialize();
```

2. **WebSocket Server:**
```javascript
const server = require('http').createServer(app);
const io = require('socket.io')(server, {
  cors: { origin: process.env.CLIENT_URL }
});

io.use(async (socket, next) => {
  const token = socket.handshake.auth.token;
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    socket.userId = decoded.userId;
    next();
  } catch (error) {
    next(new Error('Authentication error'));
  }
});

io.on('connection', (socket) => {
  console.log(`User ${socket.userId} connected`);
  
  // Register connection
  notificationService.registerConnection(socket.userId, socket);
  
  // Send unread notifications
  Notification.find({
    userId: socket.userId,
    read: false
  }).limit(50).then(notifications => {
    socket.emit('unread-notifications', notifications);
  });
  
  // Mark as read
  socket.on('mark-read', async (notificationId) => {
    await Notification.updateOne(
      { _id: notificationId },
      { read: true }
    );
  });
});
```

3. **Publishing Notifications:**
```javascript
// When user creates a comment
app.post('/api/comments', authenticate, async (req, res) => {
  const comment = await Comment.create({
    postId: req.body.postId,
    userId: req.user.id,
    text: req.body.text
  });
  
  // Get post author
  const post = await Post.findById(req.body.postId);
  
  // Publish notification
  await channel.sendToQueue(
    'notifications',
    Buffer.from(JSON.stringify({
      userId: post.authorId,
      type: 'new_comment',
      title: 'New Comment',
      body: `${req.user.name} commented on your post`,
      data: {
        commentId: comment.id,
        postId: post.id
      }
    }))
  );
  
  res.json(comment);
});
```

4. **Frontend (React):**
```javascript
import { useEffect, useState } from 'react';
import io from 'socket.io-client';

function useNotifications() {
  const [notifications, setNotifications] = useState([]);
  const [socket, setSocket] = useState(null);
  
  useEffect(() => {
    const newSocket = io('http://localhost:5000', {
      auth: { token: localStorage.getItem('token') }
    });
    
    newSocket.on('notification', (notification) => {
      setNotifications(prev => [notification, ...prev]);
      
      // Show browser notification
      if (Notification.permission === 'granted') {
        new Notification(notification.title, {
          body: notification.body,
          icon: '/icon.png'
        });
      }
    });
    
    newSocket.on('unread-notifications', (unread) => {
      setNotifications(unread);
    });
    
    setSocket(newSocket);
    
    return () => newSocket.close();
  }, []);
  
  const markAsRead = (id) => {
    socket.emit('mark-read', id);
    setNotifications(prev => 
      prev.map(n => n.id === id ? { ...n, read: true } : n)
    );
  };
  
  return { notifications, markAsRead };
}
```

**Scalability:**
- Multiple WebSocket servers behind load balancer (sticky sessions)
- Redis pub/sub for inter-server communication
- Message queue for reliability
- Database for persistence and offline delivery
- Push notifications for mobile/offline users

---

## 8. CODING CHALLENGES

### Q22: Implement a debounce function.
**Answer:**

```javascript
function debounce(func, delay) {
  let timeoutId;
  
  return function(...args) {
    // Clear previous timeout
    clearTimeout(timeoutId);
    
    // Set new timeout
    timeoutId = setTimeout(() => {
      func.apply(this, args);
    }, delay);
  };
}

// Usage
const searchAPI = debounce((query) => {
  fetch(`/api/search?q=${query}`)
    .then(res => res.json())
    .then(data => console.log(data));
}, 500);

// User types: 'h', 'he', 'hel', 'hell', 'hello'
// API called only once after 500ms of last keystroke
input.addEventListener('input', (e) => {
  searchAPI(e.target.value);
});
```

**React Hook version:**
```javascript
import { useEffect, useState } from 'react';

function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
    
    return () => clearTimeout(handler);
  }, [value, delay]);
  
  return debouncedValue;
}

// Usage
function SearchComponent() {
  const [searchTerm, setSearchTerm] = useState('');
  const debouncedSearch = useDebounce(searchTerm, 500);
  
  useEffect(() => {
    if (debouncedSearch) {
      // API call
      fetch(`/api/search?q=${debouncedSearch}`)
        .then(res => res.json())
        .then(data => console.log(data));
    }
  }, [debouncedSearch]);
  
  return (
    <input 
      value={searchTerm}
      onChange={(e) => setSearchTerm(e.target.value)}
    />
  );
}
```

### Q23: Implement a rate limiter middleware.
**Answer:**

```javascript
class RateLimiter {
  constructor(maxRequests, windowMs) {
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
    this.requests = new Map(); // ip -> [timestamps]
  }
  
  middleware() {
    return (req, res, next) => {
      const ip = req.ip;
      const now = Date.now();
      
      // Get existing requests for this IP
      if (!this.requests.has(ip)) {
        this.requests.set(ip, []);
      }
      
      const userRequests = this.requests.get(ip);
      
      // Remove old requests outside the window
      const validRequests = userRequests.filter(
        timestamp => now - timestamp < this.windowMs
      );
      
      // Check if limit exceeded
      if (validRequests.length >= this.maxRequests) {
        const oldestRequest = validRequests[0];
        const retryAfter = Math.ceil(
          (this.windowMs - (now - oldestRequest)) / 1000
        );
        
        res.setHeader('Retry-After', retryAfter);
        res.setHeader('X-RateLimit-Limit', this.maxRequests);
        res.setHeader('X-RateLimit-Remaining', 0);
        
        return res.status(429).json({
          error: 'Too many requests',
          retryAfter: `${retryAfter}s`
        });
      }
      
      // Add current request
      validRequests.push(now);
      this.requests.set(ip, validRequests);
      
      // Set headers
      res.setHeader('X-RateLimit-Limit', this.maxRequests);
      res.setHeader('X-RateLimit-Remaining', 
        this.maxRequests - validRequests.length
      );
      
      next();
    };
  }
  
  // Cleanup old entries periodically
  cleanup() {
    const now = Date.now();
    for (const [ip, timestamps] of this.requests.entries()) {
      const valid = timestamps.filter(
        t => now - t < this.windowMs
      );
      if (valid.length === 0) {
        this.requests.delete(ip);
      } else {
        this.requests.set(ip, valid);
      }
    }
  }
}

// Usage
const limiter = new RateLimiter(100, 60000); // 100 req/min
app.use('/api/', limiter.middleware());

// Cleanup every 5 minutes
setInterval(() => limiter.cleanup(), 5 * 60 * 1000);
```

### Q24: Implement a simple pub/sub system.
**Answer:**

```javascript
class EventEmitter {
  constructor() {
    this.events = {};
  }
  
  // Subscribe to event
  on(event, callback) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(callback);
    
    // Return unsubscribe function
    return () => {
      this.events[event] = this.events[event].filter(
        cb => cb !== callback
      );
    };
  }
  
  // Subscribe once
  once(event, callback) {
    const wrapper = (...args) => {
      callback(...args);
      this.off(event, wrapper);
    };
    this.on(event, wrapper);
  }
  
  // Publish event
  emit(event, ...args) {
    if (!this.events[event]) return;
    
    this.events[event].forEach(callback => {
      callback(...args);
    });
  }
  
  // Unsubscribe
  off(event, callback) {
    if (!this.events[event]) return;
    
    this.events[event] = this.events[event].filter(
      cb => cb !== callback
    );
  }
  
  // Remove all listeners for event
  removeAllListeners(event) {
    if (event) {
      delete this.events[event];
    } else {
      this.events = {};
    }
  }
}

// Usage
const emitter = new EventEmitter();

// Subscribe
const unsubscribe = emitter.on('user:login', (user) => {
  console.log(`User ${user.name} logged in`);
});

emitter.on('user:login', (user) => {
  // Send welcome email
  sendEmail(user.email, 'Welcome!');
});

// Publish
emitter.emit('user:login', { name: 'John', email: 'john@example.com' });

// Unsubscribe
unsubscribe();
```

---

## 9. BEHAVIORAL & SITUATIONAL QUESTIONS

### Q25: Tell me about a challenging bug you fixed.
**Answer Structure:**

**Situation:** Describe the context
**Task:** What needed to be solved
**Action:** Steps you took
**Result:** Outcome and learning

**Example Answer:**
"In our e-commerce application, we had an intermittent bug where users' shopping carts would sometimes appear empty after login, despite items being added before authentication. This affected about 5% of users and was difficult to reproduce.

**Investigation:**
- Reviewed logs and found no errors
- Noticed pattern: only happened with certain browsers
- Discovered it was related to session storage vs localStorage
- Found that we were using sessionStorage for cart items, which gets cleared on redirect during OAuth flow

**Solution:**
- Migrated cart state to localStorage
- Implemented state persistence layer
- Added Redux middleware to sync with backend on login
- Added comprehensive logging for cart operations

**Result:**
- Bug completely resolved
- Improved user experience
- Learned importance of understanding browser storage lifecycles
- Documented the issue for team knowledge base"

### Q26: How do you prioritize tasks when everything is urgent?
**Answer:**

"I use a combination of impact analysis and communication:

**1. Assess Impact:**
- User-facing production bugs: Highest priority
- Security vulnerabilities: Immediate attention
- Feature deadlines: Based on business value
- Technical debt: Schedule dedicated time

**2. Communication:**
- Discuss with stakeholders to understand true urgency
- Set realistic expectations
- Break down large tasks into smaller chunks

**3. Time Management:**
- Use Eisenhower Matrix (Urgent/Important)
- Time-box tasks to avoid perfectionism
- Focus on MVP for new features

**Example:**
When we had a production bug, new feature deadline, and performance issues simultaneously, I:
- Fixed critical production bug first (2 hours)
- Deployed hotfix
- Delegated performance investigation to teammate
- Delivered MVP of feature by deadline
- Scheduled performance optimization for next sprint

The key is not letting 'urgent' override 'important' consistently."

---

## 10. ADVANCED TOPICS

### Q27: Explain microservices architecture and when to use it.
**Answer:**

**Microservices:**
- Application as collection of small, independent services
- Each service runs in its own process
- Communicates via APIs (REST, gRPC, message queues)
- Independently deployable

**vs Monolith:**
- Monolith: Single codebase, deployed as one unit
- Easier to start, harder to scale

**When to use Microservices:**
- Large team (>20 developers)
- Need independent scaling
- Different tech stacks needed
- Frequent deployments
- Clear bounded contexts

**When NOT to use:**
- Small team/application
- Unclear domain boundaries
- Network latency critical
- Limited DevOps resources

**Example Architecture:**
```
API Gateway (Kong/AWS API Gateway)
    ↓
    ├─ User Service (Node.js + PostgreSQL)
    ├─ Order Service (Python + MongoDB)
    ├─ Payment Service (Java + MySQL)
    ├─ Notification Service (Go + Redis)
    └─ Analytics Service (Python + ClickHouse)
```

**Challenges:**
- Distributed system complexity
- Data consistency (eventual consistency)
- Testing difficulty
- Monitoring and debugging
- Network overhead

**Solutions:**
- Service mesh (Istio, Linkerd)
- Distributed tracing (Jaeger, Zipkin)
- Circuit breakers (Hystrix)
- API gateway pattern
- Event-driven architecture

### Q28: What is serverless architecture?
**Answer:**

**Serverless:**
- Cloud provider manages infrastructure
- Functions execute in response to events
- Pay per execution
- Auto-scaling

**Example (AWS Lambda):**
```javascript
// Lambda function
exports.handler = async (event) => {
  const body = JSON.parse(event.body);
  
  const result = await processData(body);
  
  return {
    statusCode: 200,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(result)
  };
};
```

**Triggers:**
- HTTP requests (API Gateway)
- Database changes (DynamoDB Streams)
- File uploads (S3)
- Scheduled events (CloudWatch)
- Message queues (SQS)

**Pros:**
- No server management
- Auto-scaling
- Cost-effective for variable traffic
- Fast deployment

**Cons:**
- Cold starts (latency)
- Vendor lock-in
- Debugging complexity
- Limited execution time
- State management challenges

**Best Use Cases:**
- APIs with variable traffic
- Data processing pipelines
- Webhooks
- Scheduled tasks
- Image/video processing

**Not ideal for:**
- Long-running processes
- Real-time applications
- Stateful applications
- High-frequency, low-latency needs

---

## 11. SOFT SKILLS & BEST PRACTICES

### Q29: How do you ensure code quality in a team?
**Answer:**

**1. Code Reviews:**
- All code reviewed before merge
- Use PR templates
- Check for: logic, readability, tests, security
- Constructive feedback

**2. Automated Tools:**
- ESLint/Prettier for formatting
- Husky for pre-commit hooks
- CI/CD with automated tests
- SonarQube for code quality metrics

**3. Testing:**
- Unit tests (Jest)
- Integration tests
- E2E tests (Cypress, Playwright)
- Minimum 80% coverage for critical paths

**4. Documentation:**
- README with setup instructions
- API documentation (Swagger/OpenAPI)
- Code comments for complex logic
- Architecture decision records (ADRs)

**5. Standards:**
- Coding conventions documented
- Git commit message conventions
- Branching strategy (GitFlow, trunk-based)
- Definition of Done checklist

**Example PR Checklist:**
```markdown
- [ ] Code follows style guidelines
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] No console.logs or debug code
- [ ] Security considerations addressed
- [ ] Performance impact considered
- [ ] Backward compatible or migration plan
```

### Q30: How do you stay updated with technology trends?
**Answer:**

**Daily/Weekly:**
- Follow tech blogs (Dev.to, Medium, CSS-Tricks)
- Twitter tech community
- Newsletter subscriptions (JavaScript Weekly, Node Weekly)
- GitHub trending repositories

**Monthly:**
- Attend local meetups/webinars
- Read official documentation updates
- Experiment with new tools in side projects
- Tech podcasts during commute

**Continuous:**
- Online courses (Udemy, Pluralsight, Frontend Masters)
- Open source contributions
- Build personal projects
- Tech YouTube channels

**But also:**
- Focus on fundamentals over trends
- Evaluate if new tech solves real problems
- Consider adoption in team context
- Deep knowledge > surface-level many tools

---

## 12. PERFORMANCE OPTIMIZATION

### Q31: How do you optimize backend API performance?
**Answer:**

**1. Database Optimization:**
```javascript
// Bad: N+1 Query Problem
const users = await User.findAll();
for (const user of users) {
  user.posts = await Post.findAll({ where: { userId: user.id } });
}

// Good: Join/Include
const users = await User.findAll({
  include: [{ model: Post }]
});

// Indexing
db.users.createIndex({ email: 1 });
db.posts.createIndex({ userId: 1, createdAt: -1 });

// Pagination
const posts = await Post.find()
  .limit(20)
  .skip(page * 20)
  .sort({ createdAt: -1 });

// Select only needed fields
const users = await User.find({}, 'name email'); // Only name and email
```

**2. Caching Strategy:**
```javascript
const redis = require('redis');
const client = redis.createClient();

// Cache-Aside Pattern
async function getUser(userId) {
  const cacheKey = `user:${userId}`;
  
  // Try cache first
  const cached = await client.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // Cache miss - fetch from DB
  const user = await User.findById(userId);
  
  // Store in cache (1 hour TTL)
  await client.setex(cacheKey, 3600, JSON.stringify(user));
  
  return user;
}

// Cache Invalidation
async function updateUser(userId, data) {
  const user = await User.findByIdAndUpdate(userId, data);
  
  // Invalidate cache
  await client.del(`user:${userId}`);
  
  return user;
}
```

**3. Response Compression:**
```javascript
const compression = require('compression');

app.use(compression({
  filter: (req, res) => {
    if (req.headers['x-no-compression']) {
      return false;
    }
    return compression.filter(req, res);
  },
  level: 6 // Compression level (0-9)
}));
```

**4. Connection Pooling:**
```javascript
const mongoose = require('mongoose');

mongoose.connect(process.env.MONGODB_URI, {
  maxPoolSize: 10,
  minPoolSize: 2,
  socketTimeoutMS: 45000,
  serverSelectionTimeoutMS: 5000
});
```

**5. Async Processing:**
```javascript
// Bad: Blocking
app.post('/api/orders', async (req, res) => {
  const order = await Order.create(req.body);
  await sendEmail(order); // Blocks response
  await updateInventory(order); // Blocks response
  res.json(order);
});

// Good: Queue
app.post('/api/orders', async (req, res) => {
  const order = await Order.create(req.body);
  
  // Add to queue (non-blocking)
  await orderQueue.add({ orderId: order.id });
  
  res.json(order);
});

// Worker processes queue
orderQueue.process(async (job) => {
  const order = await Order.findById(job.data.orderId);
  await sendEmail(order);
  await updateInventory(order);
});
```

**6. Load Balancing:**
```nginx
# Nginx configuration
upstream backend {
  least_conn; # Load balancing method
  server backend1:3000;
  server backend2:3000;
  server backend3:3000;
}

server {
  listen 80;
  
  location /api/ {
    proxy_pass http://backend;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

**7. HTTP/2 and Keep-Alive:**
```javascript
const http2 = require('http2');
const fs = require('fs');

const server = http2.createSecureServer({
  key: fs.readFileSync('key.pem'),
  cert: fs.readFileSync('cert.pem')
});

server.on('stream', (stream, headers) => {
  stream.respond({
    'content-type': 'application/json',
    ':status': 200
  });
  stream.end(JSON.stringify({ data: 'response' }));
});
```

### Q32: Explain lazy loading and code splitting in React.
**Answer:**

**1. Route-Based Code Splitting:**
```javascript
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// Lazy load components
const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<div>Loading...</div>}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/profile" element={<Profile />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

**2. Component-Based Code Splitting:**
```javascript
import { lazy, Suspense } from 'react';

// Heavy component loaded only when needed
const HeavyChart = lazy(() => import('./components/HeavyChart'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);
  
  return (
    <div>
      <button onClick={() => setShowChart(true)}>
        Show Chart
      </button>
      
      {showChart && (
        <Suspense fallback={<Spinner />}>
          <HeavyChart data={chartData} />
        </Suspense>
      )}
    </div>
  );
}
```

**3. Image Lazy Loading:**
```javascript
function ImageGallery({ images }) {
  return (
    <div>
      {images.map(img => (
        <img 
          key={img.id}
          src={img.url}
          loading="lazy" // Native lazy loading
          alt={img.alt}
        />
      ))}
    </div>
  );
}

// Or with IntersectionObserver
function LazyImage({ src, alt }) {
  const [isLoaded, setIsLoaded] = useState(false);
  const imgRef = useRef();
  
  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsLoaded(true);
          observer.disconnect();
        }
      },
      { threshold: 0.1 }
    );
    
    if (imgRef.current) {
      observer.observe(imgRef.current);
    }
    
    return () => observer.disconnect();
  }, []);
  
  return (
    <img
      ref={imgRef}
      src={isLoaded ? src : 'placeholder.jpg'}
      alt={alt}
    />
  );
}
```

**4. Dynamic Imports:**
```javascript
// Load library only when needed
async function handleExport() {
  const ExcelJS = await import('exceljs');
  const workbook = new ExcelJS.Workbook();
  // ... export logic
}
```

**Benefits:**
- Smaller initial bundle size
- Faster page load
- Better user experience
- Reduced bandwidth usage

---

## 13. TESTING

### Q33: Write unit tests for a complex function.
**Answer:**

**Function to Test:**
```javascript
// userService.js
class UserService {
  constructor(database, emailService) {
    this.db = database;
    this.emailService = emailService;
  }
  
  async createUser(userData) {
    // Validation
    if (!userData.email || !userData.password) {
      throw new Error('Email and password required');
    }
    
    // Check if user exists
    const existing = await this.db.findUserByEmail(userData.email);
    if (existing) {
      throw new Error('User already exists');
    }
    
    // Hash password
    const hashedPassword = await bcrypt.hash(userData.password, 10);
    
    // Create user
    const user = await this.db.createUser({
      ...userData,
      password: hashedPassword,
      createdAt: new Date()
    });
    
    // Send welcome email
    await this.emailService.sendWelcome(user.email);
    
    return user;
  }
}
```

**Unit Tests (Jest):**
```javascript
// userService.test.js
const UserService = require('./userService');
const bcrypt = require('bcrypt');

// Mock dependencies
jest.mock('bcrypt');

describe('UserService', () => {
  let userService;
  let mockDatabase;
  let mockEmailService;
  
  beforeEach(() => {
    // Setup mocks
    mockDatabase = {
      findUserByEmail: jest.fn(),
      createUser: jest.fn()
    };
    
    mockEmailService = {
      sendWelcome: jest.fn()
    };
    
    userService = new UserService(mockDatabase, mockEmailService);
  });
  
  afterEach(() => {
    jest.clearAllMocks();
  });
  
  describe('createUser', () => {
    const validUserData = {
      email: 'test@example.com',
      password: 'Password123',
      name: 'Test User'
    };
    
    test('should create user successfully', async () => {
      // Arrange
      mockDatabase.findUserByEmail.mockResolvedValue(null);
      mockDatabase.createUser.mockResolvedValue({
        id: '123',
        ...validUserData,
        password: 'hashed'
      });
      bcrypt.hash.mockResolvedValue('hashed');
      
      // Act
      const result = await userService.createUser(validUserData);
      
      // Assert
      expect(mockDatabase.findUserByEmail).toHaveBeenCalledWith(validUserData.email);
      expect(bcrypt.hash).toHaveBeenCalledWith(validUserData.password, 10);
      expect(mockDatabase.createUser).toHaveBeenCalledWith(
        expect.objectContaining({
          email: validUserData.email,
          password: 'hashed',
          name: validUserData.name
        })
      );
      expect(mockEmailService.sendWelcome).toHaveBeenCalledWith(validUserData.email);
      expect(result).toHaveProperty('id', '123');
    });
    
    test('should throw error if email is missing', async () => {
      // Arrange
      const invalidData = { password: 'Password123' };
      
      // Act & Assert
      await expect(userService.createUser(invalidData))
        .rejects.toThrow('Email and password required');
      
      expect(mockDatabase.findUserByEmail).not.toHaveBeenCalled();
    });
    
    test('should throw error if user already exists', async () => {
      // Arrange
      mockDatabase.findUserByEmail.mockResolvedValue({ id: '456' });
      
      // Act & Assert
      await expect(userService.createUser(validUserData))
        .rejects.toThrow('User already exists');
      
      expect(mockDatabase.createUser).not.toHaveBeenCalled();
      expect(mockEmailService.sendWelcome).not.toHaveBeenCalled();
    });
    
    test('should not send email if user creation fails', async () => {
      // Arrange
      mockDatabase.findUserByEmail.mockResolvedValue(null);
      mockDatabase.createUser.mockRejectedValue(new Error('DB Error'));
      bcrypt.hash.mockResolvedValue('hashed');
      
      // Act & Assert
      await expect(userService.createUser(validUserData))
        .rejects.toThrow('DB Error');
      
      expect(mockEmailService.sendWelcome).not.toHaveBeenCalled();
    });
  });
});
```

**Integration Test:**
```javascript
// userService.integration.test.js
const request = require('supertest');
const app = require('../app');
const mongoose = require('mongoose');
const User = require('../models/User');

describe('User API Integration Tests', () => {
  beforeAll(async () => {
    await mongoose.connect(process.env.TEST_DB_URL);
  });
  
  afterAll(async () => {
    await mongoose.connection.close();
  });
  
  beforeEach(async () => {
    await User.deleteMany({});
  });
  
  test('POST /api/users - should create user', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({
        email: 'test@example.com',
        password: 'Password123',
        name: 'Test User'
      })
      .expect(201);
    
    expect(response.body).toHaveProperty('id');
    expect(response.body.email).toBe('test@example.com');
    expect(response.body).not.toHaveProperty('password');
    
    // Verify in database
    const user = await User.findOne({ email: 'test@example.com' });
    expect(user).toBeTruthy();
  });
  
  test('POST /api/users - should return 400 for duplicate email', async () => {
    // Create first user
    await User.create({
      email: 'test@example.com',
      password: 'hashed',
      name: 'First User'
    });
    
    // Try to create duplicate
    const response = await request(app)
      .post('/api/users')
      .send({
        email: 'test@example.com',
        password: 'Password123',
        name: 'Second User'
      })
      .expect(400);
    
    expect(response.body.error).toBe('User already exists');
  });
});
```

**E2E Test (Cypress):**
```javascript
// cypress/e2e/registration.cy.js
describe('User Registration', () => {
  beforeEach(() => {
    cy.visit('/register');
  });
  
  it('should register new user successfully', () => {
    cy.get('[data-testid="email-input"]')
      .type('test@example.com');
    
    cy.get('[data-testid="password-input"]')
      .type('Password123');
    
    cy.get('[data-testid="name-input"]')
      .type('Test User');
    
    cy.get('[data-testid="submit-button"]')
      .click();
    
    cy.url().should('include', '/dashboard');
    cy.contains('Welcome, Test User');
  });
  
  it('should show error for invalid email', () => {
    cy.get('[data-testid="email-input"]')
      .type('invalid-email');
    
    cy.get('[data-testid="submit-button"]')
      .click();
    
    cy.contains('Invalid email address');
  });
});
```

---

## 14. DOCKER & CONTAINERIZATION

### Q34: Create a complete Docker setup for a full-stack application.
**Answer:**

**1. Backend Dockerfile:**
```dockerfile
# backend/Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source
COPY . .

# Build if needed
RUN npm run build

# Production stage
FROM node:18-alpine

WORKDIR /app

# Copy from builder
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

USER nodejs

EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:5000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

CMD ["node", "dist/server.js"]
```

**2. Frontend Dockerfile:**
```dockerfile
# frontend/Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Production - Nginx
FROM nginx:alpine

# Copy built files
COPY --from=builder /app/build /usr/share/nginx/html

# Copy nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**3. Nginx Configuration:**
```nginx
# nginx.conf
server {
  listen 80;
  server_name localhost;
  
  root /usr/share/nginx/html;
  index index.html;
  
  # Gzip compression
  gzip on;
  gzip_types text/plain text/css application/json application/javascript;
  
  # Frontend routes
  location / {
    try_files $uri $uri/ /index.html;
  }
  
  # API proxy
  location /api/ {
    proxy_pass http://backend:5000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
  
  # Cache static assets
  location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
  }
}
```

**4. Docker Compose:**
```yaml
# docker-compose.yml
version: '3.8'

services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "5000:5000"
    environment:
      - NODE_ENV=production
      - DB_HOST=mongodb
      - REDIS_HOST=redis
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      mongodb:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - ./logs:/app/logs
    networks:
      - app-network
    restart: unless-stopped

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - app-network
    restart: unless-stopped

  mongodb:
    image: mongo:5
    ports:
      - "27017:27017"
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_PASSWORD}
    volumes:
      - mongo-data:/data/db
      - ./mongo-init.js:/docker-entrypoint-initdb.d/mongo-init.js
    networks:
      - app-network
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network
    command: redis-server --appendonly yes
    restart: unless-stopped

networks:
  app-network:
    driver: bridge

volumes:
  mongo-data:
  redis-data:
```

**5. .dockerignore:**
```
# .dockerignore
node_modules
npm-debug.log
.env
.git
.gitignore
README.md
.vscode
coverage
.DS_Store
*.log
dist
build
```

**6. Development Docker Compose:**
```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    ports:
      - "5000:5000"
      - "9229:9229" # Debug port
    environment:
      - NODE_ENV=development
    volumes:
      - ./backend:/app
      - /app/node_modules
    command: npm run dev
    
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    command: npm start
```

**Commands:**
```bash
# Build and start
docker-compose up -d

# View logs
docker-compose logs -f backend

# Scale service
docker-compose up -d --scale backend=3

# Stop
docker-compose down

# Remove volumes
docker-compose down -v

# Production
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 15. REAL-WORLD SCENARIOS

### Q35: Design a feature: Real-time collaborative document editing (like Google Docs).
**Answer:**

**Architecture:**

```
Client (React) ←→ WebSocket Server ←→ Operational Transform Service
                           ↓
                    Conflict Resolution
                           ↓
                    MongoDB (Document Store)
                           ↓
                    Redis (Presence & Cursors)
```

**Implementation:**

**1. Backend - WebSocket Server:**
```javascript
const io = require('socket.io')(server);
const Document = require('./models/Document');

class CollaborativeEditor {
  constructor() {
    this.activeUsers = new Map(); // documentId -> Set<userId>
    this.cursors = new Map(); // documentId -> Map<userId, position>
  }
  
  handleConnection(socket) {
    let currentDocId = null;
    
    socket.on('join-document', async (docId) => {
      currentDocId = docId;
      
      // Join room
      socket.join(`doc:${docId}`);
      
      // Track active user
      if (!this.activeUsers.has(docId)) {
        this.activeUsers.set(docId, new Set());
      }
      this.activeUsers.get(docId).add(socket.userId);
      
      // Load document
      const document = await Document.findById(docId);
      socket.emit('document-loaded', document);
      
      // Notify others
      socket.to(`doc:${docId}`).emit('user-joined', {
        userId: socket.userId,
        userName: socket.userName
      });
      
      // Send active users list
      socket.emit('active-users', Array.from(this.activeUsers.get(docId)));
    });
    
    socket.on('operation', async (operation) => {
      // Apply operational transform
      const transformed = await this.transformOperation(
        currentDocId,
        operation
      );
      
      // Save to database
      await Document.updateOne(
        { _id: currentDocId },
        { 
          $push: { operations: transformed },
          $set: { content: transformed.resultContent }
        }
      );
      
      // Broadcast to other clients
      socket.to(`doc:${currentDocId}`).emit('operation', {
        ...transformed,
        userId: socket.userId
      });
    });
    
    socket.on('cursor-move', (position) => {
      socket.to(`doc:${currentDocId}`).emit('cursor-update', {
        userId: socket.userId,
        userName: socket.userName,
        position
      });
    });
    
    socket.on('disconnect', () => {
      if (currentDocId) {
        this.activeUsers.get(currentDocId)?.delete(socket.userId);
        
        socket.to(`doc:${currentDocId}`).emit('user-left', {
          userId: socket.userId
        });
      }
    });
  }
  
  // Operational Transform Algorithm
  async transformOperation(docId, operation) {
    const { type, position, content, version } = operation;
    
    // Get operations after this version
    const document = await Document.findById(docId);
    const laterOps = document.operations.filter(op => op.version > version);
    
    let transformedOp = { ...operation };
    
    // Transform against each later operation
    for (const laterOp of laterOps) {
      transformedOp = this.transform(transformedOp, laterOp);
    }
    
    return {
      ...transformedOp,
      version: document.operations.length + 1,
      resultContent: this.applyOperation(document.content, transformedOp)
    };
  }
  
  transform(op1, op2) {
    // Insert vs Insert
    if (op1.type === 'insert' && op2.type === 'insert') {
      if (op2.position <= op1.position) {
        return {
          ...op1,
          position: op1.position + op2.content.length
        };
      }
    }
    
    // Insert vs Delete
    if (op1.type === 'insert' && op2.type === 'delete') {
      if (op2.position < op1.position) {
        return {
          ...op1,
          position: op1.position - op2.length
        };
      }
    }
    
    // Delete vs Insert
    if (op1.type === 'delete' && op2.type === 'insert') {
      if (op2.position <= op1.position) {
        return {
          ...op1,
          position: op1.position + op2.content.length
        };
      }
    }
    
    // Delete vs Delete
    if (op1.type === 'delete' && op2.type === 'delete') {
      if (op2.position < op1.position) {
        return {
          ...op1,
          position: op1.position - op2.length
        };
      }
    }
    
    return op1;
  }
  
  applyOperation(content, operation) {
    if (operation.type === 'insert') {
      return (
        content.slice(0, operation.position) +
        operation.content +
        content.slice(operation.position)
      );
    }
    
    if (operation.type === 'delete') {
      return (
        content.slice(0, operation.position) +
        content.slice(operation.position + operation.length)
      );
    }
    
    return content;
  }
}

const editor = new CollaborativeEditor();

io.use(authenticateSocket);

io.on('connection', (socket) => {
  editor.handleConnection(socket);
});
```

**2. Frontend - React Component:**
```javascript
import { useEffect, useState, useRef } from 'react';
import io from 'socket.io-client';

function CollaborativeEditor({ documentId }) {
  const [content, setContent] = useState('');
  const [activeUsers, setActiveUsers] = useState([]);
  const [cursors, setCursors] = useState({});
  const [localVersion, setLocalVersion] = useState(0);
  
  const socketRef = useRef();
  const editorRef = useRef();
  const pendingOps = useRef([]);
  
  useEffect(() => {
    const socket = io('http://localhost:5000', {
      auth: { token: localStorage.getItem('token') }
    });
    
    socketRef.current = socket;
    
    // Join document
    socket.emit('join-document', documentId);
    
    // Load document
    socket.on('document-loaded', (doc) => {
      setContent(doc.content);
      setLocalVersion(doc.operations.length);
    });
    
    // Receive operations
    socket.on('operation', (operation) => {
      applyRemoteOperation(operation);
    });
    
    // Active users
    socket.on('active-users', setActiveUsers);
    socket.on('user-joined', (user) => {
      setActiveUsers(prev => [...prev, user]);
    });
    socket.on('user-left', (user) => {
      setActiveUsers(prev => prev.filter(u => u.userId !== user.userId));
    });
    
    // Cursor updates
    socket.on('cursor-update', (cursor) => {
      setCursors(prev => ({
        ...prev,
        [cursor.userId]: cursor
      }));
    });
    
    return () => socket.close();
  }, [documentId]);
  
  const handleChange = (e) => {
    const newContent = e.target.value;
    const cursorPos = e.target.selectionStart;
    
    // Determine operation
    const operation = calculateOperation(content, newContent, cursorPos);
    
    if (operation) {
      // Apply locally
      setContent(newContent);
      
      // Send to server
      socketRef.current.emit('operation', {
        ...operation,
        version: localVersion
      });
      
      setLocalVersion(prev => prev + 1);
    }
  };
  
  const handleCursorMove = (e) => {
    const position = e.target.selectionStart;
    socketRef.current.emit('cursor-move', position);
  };
  
  const applyRemoteOperation = (operation) => {
    setContent(prevContent => {
      if (operation.type === 'insert') {
        return (
          prevContent.slice(0, operation.position) +
          operation.content +
          prevContent.slice(operation.position)
        );
      }
      
      if (operation.type === 'delete') {
        return (
          prevContent.slice(0, operation.position) +
          prevContent.slice(operation.position + operation.length)
        );
      }
      
      return prevContent;
    });
    
    setLocalVersion(operation.version);
  };
  
      const calculateOperation = (oldContent, newContent, cursorPos) => {
    if (newContent.length > oldContent.length) {
      // Insert
      const diff = newContent.length - oldContent.length;
      return {
        type: 'insert',
        position: cursorPos - diff,
        content: newContent.slice(cursorPos - diff, cursorPos)
      };
    } else if (newContent.length < oldContent.length) {
      // Delete
      const diff = oldContent.length - newContent.length;
      return {
        type: 'delete',
        position: cursorPos,
        length: diff
      };
    }
    return null;
  };
  
  return (
    <div className="editor-container">
      <div className="active-users">
        {activeUsers.map(user => (
          <div key={user.userId} className="user-badge">
            {user.userName}
          </div>
        ))}
      </div>
      
      <div className="editor-wrapper">
        <textarea
          ref={editorRef}
          value={content}
          onChange={handleChange}
          onSelect={handleCursorMove}
          className="editor"
        />
        
        {/* Render other users' cursors */}
        {Object.entries(cursors).map(([userId, cursor]) => (
          <div
            key={userId}
            className="remote-cursor"
            style={{
              left: `${cursor.position * 8}px`, // Approximate
              top: `${Math.floor(cursor.position / 80) * 20}px`
            }}
          >
            <span className="cursor-name">{cursor.userName}</span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Challenges & Solutions:**

1. **Conflict Resolution:** Operational Transform (OT) algorithm
2. **Performance:** Debounce operations, batch updates
3. **Offline Support:** Queue operations, sync on reconnect
4. **Cursor Tracking:** Send position updates (debounced)
5. **Versioning:** Track operation versions for conflict resolution

---

## 16. BLOCKCHAIN & WEB3 INTEGRATION

### Q36: How would you integrate Web3/Blockchain into a web application?
**Answer:**

Based on your package.json showing ethers.js, here's a complete integration:

**1. Wallet Connection:**
```javascript
import { ethers } from 'ethers';
import { useState, useEffect } from 'react';

function useWeb3() {
  const [account, setAccount] = useState(null);
  const [provider, setProvider] = useState(null);
  const [signer, setSigner] = useState(null);
  const [chainId, setChainId] = useState(null);
  
  useEffect(() => {
    if (window.ethereum) {
      // Listen for account changes
      window.ethereum.on('accountsChanged', handleAccountsChanged);
      window.ethereum.on('chainChanged', handleChainChanged);
    }
    
    return () => {
      if (window.ethereum) {
        window.ethereum.removeListener('accountsChanged', handleAccountsChanged);
        window.ethereum.removeListener('chainChanged', handleChainChanged);
      }
    };
  }, []);
  
  const connectWallet = async () => {
    try {
      if (!window.ethereum) {
        throw new Error('Please install MetaMask!');
      }
      
      // Request account access
      const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts'
      });
      
      // Create provider and signer
      const web3Provider = new ethers.providers.Web3Provider(window.ethereum);
      const web3Signer = web3Provider.getSigner();
      const network = await web3Provider.getNetwork();
      
      setAccount(accounts[0]);
      setProvider(web3Provider);
      setSigner(web3Signer);
      setChainId(network.chainId);
      
      return accounts[0];
    } catch (error) {
      console.error('Error connecting wallet:', error);
      throw error;
    }
  };
  
  const disconnectWallet = () => {
    setAccount(null);
    setProvider(null);
    setSigner(null);
    setChainId(null);
  };
  
  const handleAccountsChanged = (accounts) => {
    if (accounts.length === 0) {
      disconnectWallet();
    } else {
      setAccount(accounts[0]);
    }
  };
  
  const handleChainChanged = () => {
    window.location.reload();
  };
  
  const switchNetwork = async (targetChainId) => {
    try {
      await window.ethereum.request({
        method: 'wallet_switchEthereumChain',
        params: [{ chainId: ethers.utils.hexValue(targetChainId) }]
      });
    } catch (error) {
      // Network not added to MetaMask
      if (error.code === 4902) {
        // Add network logic here
      }
      throw error;
    }
  };
  
  return {
    account,
    provider,
    signer,
    chainId,
    connectWallet,
    disconnectWallet,
    switchNetwork
  };
}
```

**2. Smart Contract Interaction:**
```javascript
// Contract ABI (from compiled contract)
const NFT_ABI = [
  "function mint(address to, uint256 tokenId) public",
  "function balanceOf(address owner) public view returns (uint256)",
  "function tokenURI(uint256 tokenId) public view returns (string)",
  "function ownerOf(uint256 tokenId) public view returns (address)"
];

const NFT_CONTRACT_ADDRESS = "0x...";

function useNFTContract() {
  const { signer, provider, account } = useWeb3();
  const [contract, setContract] = useState(null);
  
  useEffect(() => {
    if (signer) {
      const nftContract = new ethers.Contract(
        NFT_CONTRACT_ADDRESS,
        NFT_ABI,
        signer
      );
      setContract(nftContract);
    }
  }, [signer]);
  
  const mintNFT = async (tokenId) => {
    try {
      if (!contract) throw new Error('Contract not initialized');
      
      // Estimate gas
      const gasEstimate = await contract.estimateGas.mint(account, tokenId);
      
      // Send transaction
      const tx = await contract.mint(account, tokenId, {
        gasLimit: gasEstimate.mul(120).div(100) // 20% buffer
      });
      
      // Wait for confirmation
      const receipt = await tx.wait();
      
      return {
        success: true,
        transactionHash: receipt.transactionHash,
        tokenId
      };
    } catch (error) {
      console.error('Mint error:', error);
      
      // Parse error message
      if (error.code === 'INSUFFICIENT_FUNDS') {
        throw new Error('Insufficient funds for gas');
      }
      
      throw error;
    }
  };
  
  const getNFTBalance = async (address = account) => {
    try {
      const balance = await contract.balanceOf(address);
      return balance.toNumber();
    } catch (error) {
      console.error('Balance error:', error);
      throw error;
    }
  };
  
  const getNFTMetadata = async (tokenId) => {
    try {
      const tokenURI = await contract.tokenURI(tokenId);
      
      // Fetch metadata from IPFS or server
      const response = await fetch(tokenURI);
      const metadata = await response.json();
      
      return metadata;
    } catch (error) {
      console.error('Metadata error:', error);
      throw error;
    }
  };
  
  return {
    contract,
    mintNFT,
    getNFTBalance,
    getNFTMetadata
  };
}
```

**3. Backend - Transaction Verification:**
```javascript
// backend/services/blockchainService.js
const { ethers } = require('ethers');

class BlockchainService {
  constructor() {
    this.provider = new ethers.providers.JsonRpcProvider(
      process.env.RPC_URL
    );
    
    this.contract = new ethers.Contract(
      process.env.NFT_CONTRACT_ADDRESS,
      NFT_ABI,
      this.provider
    );
  }
  
  async verifyTransaction(txHash) {
    try {
      const receipt = await this.provider.getTransactionReceipt(txHash);
      
      if (!receipt) {
        return { verified: false, error: 'Transaction not found' };
      }
      
      if (receipt.status === 0) {
        return { verified: false, error: 'Transaction failed' };
      }
      
      // Check confirmations
      const currentBlock = await this.provider.getBlockNumber();
      const confirmations = currentBlock - receipt.blockNumber;
      
      if (confirmations < 3) {
        return { 
          verified: false, 
          error: 'Insufficient confirmations',
          confirmations 
        };
      }
      
      return {
        verified: true,
        blockNumber: receipt.blockNumber,
        confirmations,
        gasUsed: receipt.gasUsed.toString()
      };
    } catch (error) {
      console.error('Verification error:', error);
      throw error;
    }
  }
  
  async getNFTOwner(tokenId) {
    try {
      const owner = await this.contract.ownerOf(tokenId);
      return owner;
    } catch (error) {
      if (error.message.includes('nonexistent token')) {
        return null;
      }
      throw error;
    }
  }
  
  async listenToEvents() {
    // Listen to Transfer events
    this.contract.on('Transfer', async (from, to, tokenId, event) => {
      console.log('NFT Transfer:', {
        from,
        to,
        tokenId: tokenId.toString(),
        txHash: event.transactionHash
      });
      
      // Update database
      await this.updateNFTOwnership(tokenId.toString(), to);
    });
  }
  
  async updateNFTOwnership(tokenId, newOwner) {
    await NFT.updateOne(
      { tokenId },
      { owner: newOwner, lastTransfer: new Date() }
    );
  }
}

module.exports = new BlockchainService();
```

**4. API Endpoints:**
```javascript
// routes/blockchain.js
const express = require('express');
const router = express.Router();
const blockchainService = require('../services/blockchainService');

// Verify transaction
router.post('/verify-transaction', async (req, res) => {
  try {
    const { txHash } = req.body;
    
    const result = await blockchainService.verifyTransaction(txHash);
    
    if (result.verified) {
      // Update user's NFT collection in database
      await User.updateOne(
        { _id: req.user.id },
        { $push: { nfts: { txHash, verifiedAt: new Date() } } }
      );
    }
    
    res.json(result);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get NFT metadata
router.get('/nft/:tokenId', async (req, res) => {
  try {
    const { tokenId } = req.params;
    
    const owner = await blockchainService.getNFTOwner(tokenId);
    if (!owner) {
      return res.status(404).json({ error: 'NFT not found' });
    }
    
    const nft = await NFT.findOne({ tokenId });
    
    res.json({
      tokenId,
      owner,
      metadata: nft?.metadata,
      lastTransfer: nft?.lastTransfer
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

**5. Frontend Component:**
```javascript
function NFTMinter() {
  const { account, connectWallet } = useWeb3();
  const { mintNFT, getNFTBalance } = useNFTContract();
  const [loading, setLoading] = useState(false);
  const [balance, setBalance] = useState(0);
  
  useEffect(() => {
    if (account) {
      loadBalance();
    }
  }, [account]);
  
  const loadBalance = async () => {
    const bal = await getNFTBalance();
    setBalance(bal);
  };
  
  const handleMint = async () => {
    try {
      setLoading(true);
      
      // Generate token ID (or get from backend)
      const tokenId = Date.now();
      
      // Mint NFT
      const result = await mintNFT(tokenId);
      
      // Verify on backend
      await fetch('/api/blockchain/verify-transaction', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ txHash: result.transactionHash })
      });
      
      alert('NFT minted successfully!');
      loadBalance();
    } catch (error) {
      alert('Error: ' + error.message);
    } finally {
      setLoading(false);
    }
  };
  
  if (!account) {
    return (
      <button onClick={connectWallet}>
        Connect Wallet
      </button>
    );
  }
  
  return (
    <div>
      <p>Connected: {account}</p>
      <p>NFT Balance: {balance}</p>
      <button onClick={handleMint} disabled={loading}>
        {loading ? 'Minting...' : 'Mint NFT'}
      </button>
    </div>
  );
}
```

**Security Considerations:**
- Never store private keys in code
- Validate all transactions on backend
- Use environment variables for contract addresses
- Implement rate limiting
- Check for sufficient gas before transactions
- Handle network errors gracefully

---

## 17. ADDITIONAL IMPORTANT TOPICS

### Q37: Explain WebSockets vs Server-Sent Events (SSE) vs Long Polling.
**Answer:**

**1. WebSockets:**
- **Bidirectional** full-duplex communication
- Persistent connection
- Low latency
- Best for: Chat, gaming, collaborative editing

```javascript
// Server
io.on('connection', (socket) => {
  socket.emit('message', 'Hello from server');
  socket.on('message', (data) => {
    console.log(data);
  });
});

// Client
socket.emit('message', 'Hello from client');
socket.on('message', (data) => {
  console.log(data);
});
```

**2. Server-Sent Events (SSE):**
- **Unidirectional** (server to client only)
- HTTP connection, auto-reconnects
- Best for: Live feeds, notifications, stock tickers

```javascript
// Server (Express)
app.get('/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  
  const sendEvent = (data) => {
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };
  
  const interval = setInterval(() => {
    sendEvent({ time: new Date() });
  }, 1000);
  
  req.on('close', () => {
    clearInterval(interval);
  });
});

// Client
const eventSource = new EventSource('/events');
eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log(data);
};
```

**3. Long Polling:**
- HTTP request held open until data available
- Client immediately reconnects
- Best for: When WebSocket/SSE not available

```javascript
// Server
app.get('/poll', async (req, res) => {
  const waitForData = () => {
    return new Promise((resolve) => {
      const checkData = () => {
        const data = getNewData();
        if (data) {
          resolve(data);
        } else {
          setTimeout(checkData, 100);
        }
      };
      checkData();
    });
  };
  
  const data = await waitForData();
  res.json(data);
});

// Client
async function poll() {
  try {
    const response = await fetch('/poll');
    const data = await response.json();
    handleData(data);
  } catch (error) {
    console.error(error);
  }
  
  poll(); // Immediately poll again
}
```

**Comparison:**

| Feature | WebSocket | SSE | Long Polling |
|---------|-----------|-----|--------------|
| Direction | Bidirectional | Server→Client | Bidirectional |
| Protocol | WS/WSS | HTTP | HTTP |
| Browser Support | Good | Good | Excellent |
| Auto Reconnect | Manual | Automatic | Manual |
| Firewall Friendly | Sometimes issues | Yes | Yes |
| Overhead | Low | Medium | High |

### Q38: How do you handle file uploads in a full-stack application?
**Answer:**

**1. Frontend - React:**
```javascript
import { useState } from 'react';

function FileUpload() {
  const [file, setFile] = useState(null);
  const [progress, setProgress] = useState(0);
  const [preview, setPreview] = useState(null);
  
  const handleFileChange = (e) => {
    const selectedFile = e.target.files[0];
    
    // Validation
    const maxSize = 5 * 1024 * 1024; // 5MB
    const allowedTypes = ['image/jpeg', 'image/png', 'image/gif'];
    
    if (selectedFile.size > maxSize) {
      alert('File too large. Max 5MB');
      return;
    }
    
    if (!allowedTypes.includes(selectedFile.type)) {
      alert('Invalid file type');
      return;
    }
    
    setFile(selectedFile);
    
    // Create preview
    const reader = new FileReader();
    reader.onload = (e) => setPreview(e.target.result);
    reader.readAsDataURL(selectedFile);
  };
  
  const handleUpload = async () => {
    if (!file) return;
    
    const formData = new FormData();
    formData.append('file', file);
    formData.append('metadata', JSON.stringify({
      originalName: file.name,
      uploadedBy: 'userId'
    }));
    
    try {
      const xhr = new XMLHttpRequest();
      
      // Progress tracking
      xhr.upload.addEventListener('progress', (e) => {
        if (e.lengthComputable) {
          const percentComplete = (e.loaded / e.total) * 100;
          setProgress(percentComplete);
        }
      });
      
      xhr.addEventListener('load', () => {
        if (xhr.status === 200) {
          const response = JSON.parse(xhr.responseText);
          alert('Upload successful!');
          console.log(response);
        }
      });
      
      xhr.open('POST', '/api/upload');
      xhr.setRequestHeader('Authorization', `Bearer ${token}`);
      xhr.send(formData);
      
    } catch (error) {
      console.error('Upload error:', error);
    }
  };
  
  return (
    <div>
      <input 
        type="file" 
        onChange={handleFileChange}
        accept="image/*"
      />
      
      {preview && (
        <img src={preview} alt="Preview" style={{ maxWidth: '200px' }} />
      )}
      
      {file && (
        <div>
          <p>{file.name} - {(file.size / 1024 / 1024).toFixed(2)} MB</p>
          <button onClick={handleUpload}>Upload</button>
          {progress > 0 && (
            <div className="progress-bar">
              <div style={{ width: `${progress}%` }}>{progress.toFixed(0)}%</div>
            </div>
          )}
        </div>
      )}
    </div>
  );
}
```

**2. Backend - Express with Multer:**
```javascript
const express = require('express');
const multer = require('multer');
const path = require('path');
const crypto = require('crypto');

// Configure storage
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, 'uploads/');
  },
  filename: (req, file, cb) => {
    // Generate unique filename
    const uniqueName = crypto.randomBytes(16).toString('hex');
    const ext = path.extname(file.originalname);
    cb(null, `${uniqueName}${ext}`);
  }
});

// File filter
const fileFilter = (req, file, cb) => {
  const allowedTypes = ['image/jpeg', 'image/png', 'image/gif'];
  
  if (allowedTypes.includes(file.mimetype)) {
    cb(null, true);
  } else {
    cb(new Error('Invalid file type'), false);
  }
};

const upload = multer({
  storage,
  fileFilter,
  limits: {
    fileSize: 5 * 1024 * 1024 // 5MB
  }
});

// Single file upload
app.post('/api/upload', 
  authenticate,
  upload.single('file'),
  async (req, res) => {
    try {
      if (!req.file) {
        return res.status(400).json({ error: 'No file uploaded' });
      }
      
      const metadata = JSON.parse(req.body.metadata);
      
      // Save file info to database
      const fileRecord = await File.create({
        filename: req.file.filename,
        originalName: metadata.originalName,
        path: req.file.path,
        size: req.file.size,
        mimetype: req.file.mimetype,
        uploadedBy: req.user.id,
        uploadedAt: new Date()
      });
      
      res.json({
        success: true,
        file: {
          id: fileRecord.id,
          url: `/uploads/${req.file.filename}`,
          originalName: metadata.originalName
        }
      });
    } catch (error) {
      console.error('Upload error:', error);
      res.status(500).json({ error: error.message });
    }
  }
);

// Multiple files
app.post('/api/upload-multiple',
  authenticate,
  upload.array('files', 10), // Max 10 files
  async (req, res) => {
    try {
      const fileRecords = await Promise.all(
        req.files.map(file => 
          File.create({
            filename: file.filename,
            originalName: file.originalname,
            path: file.path,
            size: file.size,
            uploadedBy: req.user.id
          })
        )
      );
      
      res.json({
        success: true,
        files: fileRecords
      });
    } catch (error) {
      res.status(500).json({ error: error.message });
    }
  }
);

// Error handling
app.use((error, req, res, next) => {
  if (error instanceof multer.MulterError) {
    if (error.code === 'LIMIT_FILE_SIZE') {
      return res.status(400).json({ error: 'File too large' });
    }
    if (error.code === 'LIMIT_FILE_COUNT') {
      return res.status(400).json({ error: 'Too many files' });
    }
  }
  next(error);
});
```

**3. Cloud Storage (AWS S3):**
```javascript
const AWS = require('aws-sdk');
const multerS3 = require('multer-s3');

const s3 = new AWS.S3({
  accessKeyId: process.env.AWS_ACCESS_KEY,
  secretAccessKey: process.env.AWS_SECRET_KEY,
  region: process.env.AWS_REGION
});

const s3Upload = multer({
  storage: multerS3({
    s3: s3,
    bucket: process.env.S3_BUCKET,
    acl: 'public-read',
    metadata: (req, file, cb) => {
      cb(null, { fieldName: file.fieldname });
    },
    key: (req, file, cb) => {
      const uniqueName = crypto.randomBytes(16).toString('hex');
      const ext = path.extname(file.originalname);
      cb(null, `uploads/${Date.now()}-${uniqueName}${ext}`);
    }
  }),
  fileFilter,
  limits: { fileSize: 5 * 1024 * 1024 }
});

app.post('/api/upload-s3',
  authenticate,
  s3Upload.single('file'),
  (req, res) => {
    res.json({
      success: true,
      file: {
        url: req.file.location,
        key: req.file.key
      }
    });
  }
);

// Delete from S3
app.delete('/api/files/:key', authenticate, async (req, res) => {
  try {
    await s3.deleteObject({
      Bucket: process.env.S3_BUCKET,
      Key: req.params.key
    }).promise();
    
    res.json({ success: true });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

**4. Direct Upload to S3 (Presigned URL):**
```javascript
// Backend - Generate presigned URL
app.post('/api/get-upload-url', authenticate, async (req, res) => {
  const { fileName, fileType } = req.body;
  
  const key = `uploads/${Date.now()}-${fileName}`;
  
  const params = {
    Bucket: process.env.S3_BUCKET,
    Key: key,
    Expires: 60, // URL valid for 60 seconds
    ContentType: fileType,
    ACL: 'public-read'
  };
  
  const uploadURL = await s3.getSignedUrlPromise('putObject', params);
  
  res.json({
    uploadURL,
    key
  });
});

// Frontend - Direct upload
async function uploadToS3() {
  // Get presigned URL
  const { uploadURL, key } = await fetch('/api/get-upload-url', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      fileName: file.name,
      fileType: file.type
    })
  }).then(r => r.json());
  
  // Upload directly to S3
  await fetch(uploadURL, {
    method: 'PUT',
    body: file,
    headers: {
      'Content-Type': file.type
    }
  });
  
  console.log('File uploaded:', key);
}
```

---

## 18. FINAL TIPS & COMMON PITFALLS

### Q39: What are common mistakes developers make and how do you avoid them?
**Answer:**

**1. Not Handling Errors Properly:**
```javascript
// Bad
async function getUser(id) {
  const user = await User.findById(id);
  return user.name; // Crashes if user is null
}

// Good
async function getUser(id) {
  try {
    const user = await User.findById(id);
    if (!user) {
      throw new Error('User not found');
    }
    return user.name;
  } catch (error) {
    logger.error('Error fetching user:', error);
    throw error;
  }
}
```

**2. N+1 Query Problem:**
```javascript
// Bad - N+1 queries
const posts = await Post.findAll();
for (const post of posts) {
  post.author = await User.findById(post.authorId); // N queries
}

// Good - Single query with join
const posts = await Post.findAll({
  include: [{ model: User, as: 'author' }]
});
```

**3. Not Sanitizing User Input:**
```javascript
// Bad
app.post('/search', (req, res) => {
  const query = req.body.query;
  db.query(`SELECT * FROM products WHERE name LIKE '%${query}%'`);
});

// Good
app.post('/search', [
  body('query').trim().escape()
], (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ errors: errors.array() });
  }
  // Use parameterized query
});
```

**4. Storing Passwords in Plain Text:**
```javascript
// Bad
await User.create({ password: req.body.password });

// Good
const hashedPassword = await bcrypt.hash(req.body.password, 10);
await User.create({ password: hashedPassword });
```

**5. Not Using Environment Variables:**
```javascript
// Bad
const API_KEY = 'sk-12345...';

// Good
const API_KEY = process.env.API_KEY;
```

**6. Callback Hell:**
```javascript
// Bad
getData((data) => {
  processData(data, (result) => {
    saveData(result, (saved) => {
      notify(saved, () => {
        // ...
      });
    });
  });
});

// Good
const data = await getData();
const result = await processData(data);
const saved = await saveData(result);
await notify(saved);
```

**7. Not Closing Database Connections:**
```javascript
// Bad - Connection leak
app.get('/users', async (req, res) => {
  const db = await connectDB();
  const users = await db.query('SELECT * FROM users');
  res.json(users);
  // Connection not closed!
});

// Good - Use connection pooling or close properly
```

### Q40: How do you approach learning a new technology?
**Answer:**

**My Approach:**

**1. Understand the "Why":**
- What problem does it solve?
- When should I use it vs alternatives?
- What are the trade-offs?

**2. Learn Fundamentals:**
- Official documentation first
- Core concepts and principles
- Don't jump to frameworks immediately

**3. Build Something:**
- Start with a simple project
- Gradually increase complexity
- Apply what you learn immediately

**4. Read Others' Code:**
- Open source projects
- Best practices
- Common patterns

**5. Share Knowledge:**
- Write blog posts
- Create tutorials
- Teach others (best way to learn)

**Example - Learning GraphQL:**
1. Read official docs to understand concepts
2. Build a simple API (books, authors)
3. Compare with REST implementation
4. Add to existing project
5. Document learnings in team wiki

**Red Flags to Avoid:**
- Tutorial hell (watching but not building)
- Jumping to advanced topics too quickly
- Not understanding fundamentals
- Following outdated resources
- Not practicing regularly

---

## CONCLUSION & INTERVIEW STRATEGY

### Key Tips for Your Interview:

1. **Think Out Loud:** Explain your thought process
2. **Ask Clarifying Questions:** Show you understand requirements
3. **Consider Trade-offs:** Discuss pros/cons of different approaches
4. **Relate to Experience:** Connect to your project (Azumo-Game)
5. **Admit What You Don't Know:** Show willingness to learn
6. **Follow-up Questions:** Prepare to go deeper on any topic

### Your Project-Specific Talking Points (Based on package.json):

**1. Full-Stack Architecture:**
"In Azumo-Game, I built a full-stack application using React 17 on the frontend and Express/Node.js on the backend. We integrated MongoDB for user data and implemented JWT authentication for secure sessions."

**2. Blockchain Integration:**
"I integrated Web3 functionality using ethers.js v5 for connecting to Ethereum networks. Users can connect their MetaMask wallets, and we handle smart contract interactions for in-game NFT assets."

**3. Real-time Features:**
"While not in the current package.json, I would implement WebSocket connections using Socket.io for real-time multiplayer features, allowing players to interact in the same game session."

**4. Build Process:**
"I worked with Webpack 4 and custom configurations using Craco for the build pipeline. We had to use the --openssl-legacy-provider flag for Node.js 17+ compatibility."

**5. State Management:**
"We used React Context API and Apollo Client for state management, particularly for managing blockchain wallet connections and user session data."

**6. Testing:**
"I implemented testing with Jest and React Testing Library, ensuring core game logic and user authentication flows were properly tested."

### Technical Depth Areas to Highlight:

**Frontend:**
- React hooks and functional components
- Performance optimization techniques
- Responsive design implementation
- Integration with Web3 providers

**Backend:**
- RESTful API design and implementation
- Database schema design with Mongoose
- Authentication and authorization flows
- Error handling and validation

**DevOps:**
- Docker containerization (if applicable)
- Environment configuration management
- Build optimization strategies

**Problem Solving:**
- OpenSSL legacy provider issue resolution
- Cross-browser compatibility challenges
- Async blockchain transaction handling

---

## 19. BEHAVIORAL QUESTIONS - STAR FORMAT

### Q41: Tell me about a time you had to debug a difficult production issue.
**STAR Answer:**

**Situation:** 
"In Azumo-Game, we deployed a new feature that allowed users to mint NFTs. Within hours, we started receiving reports that some users' transactions were succeeding on-chain but failing to update in our database, causing users to lose track of their NFTs."

**Task:**
"I needed to identify the root cause quickly and implement a fix without requiring users to re-mint or lose their assets."

**Action:**
1. "First, I checked our error logs and found intermittent MongoDB connection timeouts during high traffic periods."
2. "I analyzed the blockchain transactions and confirmed they were successful on-chain."
3. "I discovered we weren't implementing retry logic for database writes after blockchain confirmations."
4. "I implemented a solution:
   - Added a message queue (Bull) to handle blockchain event processing
   - Created a background worker to sync blockchain state with database
   - Added idempotency checks to prevent duplicate entries
   - Implemented a reconciliation script to fix existing data gaps"
5. "I wrote a migration script that fetched all successful transactions from the blockchain and updated our database accordingly."

**Result:**
"We recovered all missing NFT records for affected users within 2 hours. The queue-based approach improved reliability to 99.9%, and we haven't had a sync issue since. I also documented the incident and created monitoring alerts for similar issues in the future."

**Key Learning:**
"This taught me the importance of handling distributed system failures gracefully. Now I always implement idempotency and retry logic when dealing with external systems, especially blockchain."

---

### Q42: Describe a time you had to learn a new technology quickly.
**STAR Answer:**

**Situation:**
"When starting the Azumo-Game project, I needed to integrate Web3 functionality, but I had minimal blockchain development experience at the time."

**Task:**
"I had 2 weeks to implement wallet connection, smart contract interactions, and transaction handling before our demo to investors."

**Action:**
1. "Day 1-3: I focused on fundamentals:
   - Read ethers.js documentation thoroughly
   - Studied how MetaMask works
   - Understood gas, transactions, and wallet signatures"

2. "Day 4-7: Built a simple proof of concept:
   - Created a basic NFT minting interface
   - Implemented wallet connection
   - Tested on testnet (Goerli)"

3. "Day 8-12: Integrated into main application:
   - Created reusable Web3 hooks
   - Added error handling for common issues
   - Implemented transaction status tracking"

4. "Day 13-14: Polish and testing:
   - Added loading states and user feedback
   - Tested edge cases (wallet not installed, wrong network)
   - Created user documentation"

**Result:**
"Successfully delivered the feature on time. The demo went well, and investors were impressed with the seamless Web3 integration. Since then, I've become the go-to person on the team for blockchain-related questions."

**Key Learning:**
"Breaking down the learning into focused phases worked well. I now apply this approach whenever learning new technologies: fundamentals first, then practical application, then integration and polish."

---

### Q43: Tell me about a time you disagreed with a team member about a technical decision.
**STAR Answer:**

**Situation:**
"During Azumo-Game development, a senior developer wanted to use Redux for all state management, including simple component-level state. I felt this was overengineering."

**Task:**
"I needed to express my concerns constructively while respecting their experience."

**Action:**
1. "I requested a meeting to discuss the state management strategy."
2. "I prepared data:
   - Bundle size impact of adding Redux
   - Code complexity comparison for simple use cases
   - Performance benchmarks"
3. "I proposed a hybrid approach:
   - Redux for global state (user auth, blockchain connections)
   - Context API for feature-specific state (game settings)
   - Local state (useState) for component-specific state"
4. "I created a proof of concept showing both approaches side-by-side."
5. "I emphasized I was open to their perspective and asked them to explain their reasoning."

**Result:**
"After discussing, they agreed the hybrid approach made sense. We implemented it, which kept our bundle size 20% smaller and made the codebase more maintainable. More importantly, we established a collaborative decision-making process."

**Key Learning:**
"Come prepared with data, not just opinions. Always approach disagreements as 'we're solving a problem together' rather than 'I'm right, you're wrong.' This experience taught me the value of technical discussions backed by evidence."

---

### Q44: Describe a time you improved an existing system or process.
**STAR Answer:**

**Situation:**
"In Azumo-Game, our API response times were degrading as our user base grew. Some endpoints were taking 3-5 seconds to respond during peak hours."

**Task:**
"I was tasked with improving performance without major architecture changes."

**Action:**
1. "Performance Analysis:
   - Used MongoDB explain() to analyze slow queries
   - Profiled API endpoints to identify bottlenecks
   - Checked for N+1 query problems"

2. "Implemented Optimizations:
   - Added database indexes on frequently queried fields (user email, NFT tokenId)
   - Implemented Redis caching for static data (game metadata, user profiles)
   - Fixed N+1 queries by using proper joins/population
   - Added pagination to endpoints returning large datasets
   - Implemented connection pooling for database"

3. "Monitoring:
   - Set up response time monitoring
   - Added performance budgets to CI/CD
   - Created alerts for slow queries"

**Result:**
"Average API response time dropped from 3-5 seconds to 200-300ms (90% improvement). Database query load decreased by 60%. User satisfaction scores improved, and we could handle 3x more concurrent users without performance degradation."

**Key Learning:**
"Measure before optimizing. I used to optimize based on assumptions, but this taught me to always profile first. Also, caching and proper indexing can solve most performance issues without complex solutions."

---

## 20. SYSTEM DESIGN QUESTIONS

### Q45: Design a URL shortener service (Complete System Design).
**Detailed Answer:**

**1. Requirements Clarification:**
- Functional: Create short URLs, redirect to original URLs
- Non-functional: High availability, low latency, scalable
- Scale: 100M URLs created per month, 10:1 read-write ratio

**2. Capacity Estimation:**
- Write: 100M / (30 days * 86400 sec) ≈ 40 writes/second
- Read: 400 reads/second
- Storage: 100M URLs/month * 500 bytes ≈ 50GB/month
- For 5 years: 50GB * 12 * 5 = 3TB

**3. High-Level Design:**

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
┌──────▼──────────┐
│  Load Balancer  │
└──────┬──────────┘
       │
┌──────▼──────────────────────────┐
│     Application Servers          │
│  (Stateless - Scale Horizontal)  │
└──────┬──────────────────────────┘
       │
   ┌───┴───┬──────────┬────────────┐
   │       │          │            │
┌──▼───┐ ┌─▼──────┐ ┌▼─────────┐ ┌▼────────┐
│Redis │ │Database│ │Analytics │ │ID Gen   │
│Cache │ │(Write) │ │  Queue   │ │Service  │
└──────┘ └────────┘ └──────────┘ └─────────┘
           │
    ┌──────┴──────┐
    │             │
┌───▼────┐   ┌───▼────┐
│DB Shard│   │DB Shard│
│   1    │   │   2    │
└────────┘   └────────┘
```

**4. API Design:**

```javascript
// Create short URL
POST /api/shorten
Request:
{
  "originalUrl": "https://example.com/very/long/url",
  "customAlias": "optional",  // Optional custom short code
  "expiresAt": "2024-12-31"   // Optional expiration
}

Response:
{
  "shortUrl": "https://short.ly/abc123",
  "shortCode": "abc123",
  "originalUrl": "https://example.com/very/long/url",
  "createdAt": "2024-01-15T10:30:00Z",
  "expiresAt": "2024-12-31T23:59:59Z"
}

// Redirect
GET /{shortCode}
Response: 301 Redirect to original URL

// Analytics
GET /api/analytics/{shortCode}
Response:
{
  "shortCode": "abc123",
  "clicks": 15234,
  "uniqueVisitors": 8721,
  "lastClicked": "2024-01-20T14:22:00Z",
  "clicksByCountry": {...},
  "clicksByDevice": {...}
}
```

**5. Database Schema:**

```sql
-- PostgreSQL
CREATE TABLE urls (
  id BIGSERIAL PRIMARY KEY,
  short_code VARCHAR(10) UNIQUE NOT NULL,
  original_url TEXT NOT NULL,
  user_id BIGINT,
  created_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP,
  is_active BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_short_code ON urls(short_code);
CREATE INDEX idx_user_id ON urls(user_id);
CREATE INDEX idx_created_at ON urls(created_at);

CREATE TABLE analytics (
  id BIGSERIAL PRIMARY KEY,
  short_code VARCHAR(10) NOT NULL,
  clicked_at TIMESTAMP DEFAULT NOW(),
  ip_address VARCHAR(45),
  user_agent TEXT,
  country VARCHAR(2),
  device_type VARCHAR(20)
);

CREATE INDEX idx_analytics_short_code ON analytics(short_code);
CREATE INDEX idx_analytics_clicked_at ON analytics(clicked_at);
```

**6. Short Code Generation:**

**Option A: Base62 Encoding (Chosen):**
```javascript
class ShortCodeGenerator {
  constructor() {
    this.chars = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
  }
  
  // Convert number to base62
  encode(num) {
    if (num === 0) return this.chars[0];
    
    let result = '';
    while (num > 0) {
      result = this.chars[num % 62] + result;
      num = Math.floor(num / 62);
    }
    return result;
  }
  
  // Generate short code from auto-increment ID
  async generateShortCode() {
    // Get next ID from counter
    const id = await redis.incr('url:id:counter');
    return this.encode(id);
  }
}

// With 6 characters: 62^6 = 56 billion possible URLs
// With 7 characters: 62^7 = 3.5 trillion possible URLs
```

**Option B: MD5 Hash (Alternative):**
```javascript
function generateShortCode(url) {
  const hash = crypto.createHash('md5').update(url).digest('hex');
  return hash.substring(0, 7); // Take first 7 characters
  // Need collision detection
}
```

**7. Caching Strategy:**

```javascript
class URLService {
  async getOriginalURL(shortCode) {
    // Try cache first (Redis)
    const cached = await redis.get(`url:${shortCode}`);
    if (cached) {
      // Async: increment click counter
      this.trackClick(shortCode);
      return cached;
    }
    
    // Cache miss - query database
    const url = await db.query(
      'SELECT original_url, expires_at FROM urls WHERE short_code = $1 AND is_active = TRUE',
      [shortCode]
    );
    
    if (!url) return null;
    
    // Check expiration
    if (url.expires_at && new Date() > url.expires_at) {
      return null;
    }
    
    // Cache for 1 hour
    await redis.setex(`url:${shortCode}`, 3600, url.original_url);
    
    this.trackClick(shortCode);
    
    return url.original_url;
  }
  
  async trackClick(shortCode) {
    // Add to queue for async processing
    await analyticsQueue.add({
      shortCode,
      timestamp: Date.now(),
      // Additional metadata
    });
  }
}
```

**8. Rate Limiting:**

```javascript
const rateLimit = require('express-rate-limit');

// Different limits for authenticated vs anonymous
const createLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: async (req) => {
    if (req.user) return 100; // Authenticated: 100 per 15 min
    return 10; // Anonymous: 10 per 15 min
  },
  standardHeaders: true
});

app.post('/api/shorten', createLimiter, shortenController);
```

**9. Scalability Considerations:**

**Horizontal Scaling:**
- Stateless application servers
- Load balancer distributes traffic
- Can add/remove servers based on load

**Database Sharding:**
```javascript
// Shard by first character of short code
function getShardNumber(shortCode) {
  const firstChar = shortCode[0];
  return firstChar.charCodeAt(0) % NUM_SHARDS;
}

// Route queries to appropriate shard
const shard = getShardNumber(shortCode);
const connection = dbConnections[shard];
```

**Caching:**
- Redis cluster for high availability
- Cache hot URLs (80/20 rule: 20% URLs get 80% traffic)
- Use consistent hashing for cache distribution

**10. High Availability:**

**Database:**
- Master-slave replication
- Automatic failover
- Regular backups

**Redis:**
- Redis Sentinel for automatic failover
- Redis Cluster for partitioning

**Application:**
- Multiple instances across availability zones
- Health checks and auto-recovery
- Circuit breakers for external dependencies

**11. Monitoring & Alerts:**

```javascript
// Prometheus metrics
const prometheus = require('prom-client');

const urlCreationCounter = new prometheus.Counter({
  name: 'urls_created_total',
  help: 'Total number of URLs created'
});

const redirectLatency = new prometheus.Histogram({
  name: 'redirect_latency_seconds',
  help: 'Redirect request latency',
  buckets: [0.01, 0.05, 0.1, 0.5, 1]
});

const cacheHitRate = new prometheus.Gauge({
  name: 'cache_hit_rate',
  help: 'Cache hit rate percentage'
});

// Track metrics
app.get('/:shortCode', async (req, res) => {
  const end = redirectLatency.startTimer();
  // ... handle redirect
  end();
});
```

**12. Security Considerations:**

```javascript
// Prevent malicious URLs
async function validateURL(url) {
  // Check against blacklist
  const blacklisted = await isBlacklisted(url);
  if (blacklisted) throw new Error('URL is blacklisted');
  
  // Check for phishing
  const safeBrowsing = await checkSafeBrowsing(url);
  if (!safeBrowsing) throw new Error('URL flagged as unsafe');
  
  // Validate format
  const urlPattern = /^https?:\/\/.+/;
  if (!urlPattern.test(url)) throw new Error('Invalid URL format');
  
  return true;
}

// Prevent abuse
async function checkUserLimits(userId) {
  const count = await redis.get(`user:${userId}:daily_count`);
  if (count > DAILY_LIMIT) {
    throw new Error('Daily limit exceeded');
  }
}
```

**Trade-offs & Decisions:**

1. **Base62 vs UUID:**
   - Chose Base62 for shorter codes (6-7 chars vs 36 chars)
   - Sequential IDs are predictable but shorter

2. **SQL vs NoSQL:**
   - Chose PostgreSQL for ACID compliance
   - URL creation is transactional
   - Need for analytics queries

3. **Synchronous vs Asynchronous Analytics:**
   - Chose async (queue) to not block redirects
   - Redirects need to be fast (<50ms)
   - Analytics can have delay

4. **Cache Expiration:**
   - 1 hour TTL balances freshness and cache hits
   - Expired URLs checked in database
   - Can adjust based on access patterns

This design handles 100M+ URLs/month, sub-100ms redirects, and can scale horizontally as needed.

---

## FINAL PREPARATION CHECKLIST

### Before the Interview:

**Technical Prep:**
- [ ] Review your Azumo-Game codebase thoroughly
- [ ] Be ready to explain architectural decisions
- [ ] Prepare examples of challenges you solved
- [ ] Review recent tech trends (AI integration, Web3)
- [ ] Practice coding on whiteboard/screen share

**Project-Specific:**
- [ ] Know your package.json dependencies and why you chose them
- [ ] Understand the build process (Webpack, Craco configuration)
- [ ] Be able to explain your state management approach
- [ ] Know how you implemented Web3 integration
- [ ] Understand your security implementations

**Behavioral Prep:**
- [ ] Prepare 3-4 STAR format stories
- [ ] Have examples of: debugging, learning, collaboration, conflict
- [ ] Prepare questions to ask interviewer
- [ ] Research the company and their tech stack

**Day Of:**
- [ ] Test your setup (internet, camera, mic)
- [ ] Have your code ready to share if needed
- [ ] Keep water nearby
- [ ] Have a notepad for notes
- [ ] Join 5 minutes early

### Questions to Ask Interviewer:

1. "What does a typical day look like for this role?"
2. "What are the biggest technical challenges the team is facing?"
3. "How do you approach technical debt and refactoring?"
4. "What's the team's approach to code reviews and testing?"
5. "How do you evaluate and adopt new technologies?"
6. "What does success look like for this position in the first 3-6 months?"
7. "Can you tell me about the team structure and collaboration process?"
8. "What AI integration are you currently working on or planning?"

---

## Good Luck, Biplab! 🚀

**Remember:**
- **Be confident** - You have real project experience
- **Be honest** - It's okay to say "I don't know, but here's how I'd find out"
- **Be curious** - Ask follow-up questions
- **Be collaborative** - Show you work well with others
- **Be yourself** - Let your passion for development shine through

**You've got this!** Your Azumo-Game project shows you have hands-on full-stack experience with modern technologies, blockchain integration, and you can build end-to-end solutions. That's exactly what they're looking for.

**Final Tip:** If you get stuck on a question, talk through your thought process. Interviewers want to see how you think, not just the final answer. Show them you can break down problems, consider trade-offs, and learn from challenges.

**All the best for your interview! 💪**