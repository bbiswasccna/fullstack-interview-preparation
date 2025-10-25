# certfile = '/path/to/cert.pem'

# Run with: gunicorn -c gunicorn_config.py myproject.wsgi:application
```

**3. Nginx Configuration:**

```nginx
# /etc/nginx/sites-available/myapp
upstream django_app {
    server 127.0.0.1:8000;
    # Add more servers for load balancing
    # server 127.0.0.1:8001;
    # server 127.0.0.1:8002;
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS server
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;
    
    # SSL certificates
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_stapling on;
    ssl_stapling_verify on;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Client body size
    client_max_body_size 10M;
    
    # Timeouts
    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
    
    # Static files
    location /static/ {
        alias /var/www/myapp/staticfiles/;
        expires 1y;
        access_log off;
        add_header Cache-Control "public, immutable";
        
        # Gzip compression
        gzip on;
        gzip_types text/css application/javascript application/json image/svg+xml;
        gzip_vary on;
    }
    
    # Media files
    location /media/ {
        alias /var/www/myapp/media/;
        expires 7d;
        access_log off;
    }
    
    # Django app
    location / {
        proxy_pass http://django_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        
        # WebSocket support (if needed)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    
    # Health check endpoint
    location /health/ {
        access_log off;
        proxy_pass http://django_app;
    }
}
```

**4. Docker Configuration:**

```dockerfile
# Dockerfile
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Install system dependencies
RUN apt-get update && apt-get install -y \
    postgresql-client \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Set work directory
WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy project
COPY . .

# Collect static files
RUN python manage.py collectstatic --noinput

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Run gunicorn
CMD ["gunicorn", "-c", "gunicorn_config.py", "myproject.wsgi:application"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    command: gunicorn -c gunicorn_config.py myproject.wsgi:application
    volumes:
      - static_volume:/app/staticfiles
      - media_volume:/app/media
    expose:
      - 8000
    env_file:
      - .env
    depends_on:
      - db
      - redis
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - static_volume:/var/www/staticfiles
      - media_volume:/var/www/media
      - ./ssl:/etc/nginx/ssl
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - web
    restart: unless-stopped

  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  celery_worker:
    build: .
    command: celery -A myproject worker -l info -c 4
    volumes:
      - .:/app
    env_file:
      - .env
    depends_on:
      - db
      - redis
    restart: unless-stopped

  celery_beat:
    build: .
    command: celery -A myproject beat -l info
    volumes:
      - .:/app
    env_file:
      - .env
    depends_on:
      - db
      - redis
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  static_volume:
  media_volume:
```

**5. Celery for Async Tasks:**

```python
# myproject/celery.py
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings.production')

app = Celery('myproject')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()

# myapp/tasks.py
from celery import shared_task
from django.core.mail import send_mail
from django.conf import settings

@shared_task(bind=True, max_retries=3)
def send_email_task(self, subject, message, recipient_list):
    """Send email asynchronously"""
    try:
        send_mail(
            subject,
            message,
            settings.DEFAULT_FROM_EMAIL,
            recipient_list,
            fail_silently=False,
        )
        return f"Email sent to {recipient_list}"
    except Exception as exc:
        # Retry after 5 minutes
        raise self.retry(exc=exc, countdown=300)

@shared_task
def cleanup_old_sessions():
    """Clean up expired sessions"""
    from django.contrib.sessions.models import Session
    from django.utils import timezone
    
    Session.objects.filter(expire_date__lt=timezone.now()).delete()

@shared_task
def generate_report(user_id, report_type):
    """Generate heavy report"""
    from .models import Report
    import time
    
    # Simulate long-running task
    time.sleep(10)
    
    report = Report.objects.create(
        user_id=user_id,
        type=report_type,
        status='completed'
    )
    
    return f"Report {report.id} generated"

# In views
from .tasks import send_email_task

def register_user(request):
    # ... user creation logic ...
    
    # Send welcome email asynchronously
    send_email_task.delay(
        subject='Welcome!',
        message='Welcome to our platform',
        recipient_list=[user.email]
    )
    
    return redirect('home')
```

**6. Database Optimization for Scale:**

```python
# Read replicas
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST'),
        'PORT': env('DB_PORT'),
    },
    'replica1': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_REPLICA1_HOST'),
        'PORT': env('DB_PORT'),
    },
}

# Database router
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):
        """Read from replica"""
        return 'replica1'
    
    def db_for_write(self, model, **hints):
        """Write to primary"""
        return 'default'
    
    def allow_relation(self, obj1, obj2, **hints):
        return True
    
    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == 'default'

DATABASE_ROUTERS = ['myproject.routers.PrimaryReplicaRouter']

# Connection pooling with pgbouncer
# settings.py
DATABASES['default']['OPTIONS'] = {
    'pool': True,
    'pool_size': 20,
}
```

**7. Horizontal Scaling:**

```python
# Load balancer configuration (HAProxy example)
# /etc/haproxy/haproxy.cfg
global
    maxconn 4096
    
defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend http_front
    bind *:80
    default_backend django_backend

backend django_backend
    balance roundrobin
    option httpchk GET /health/
    
    server app1 10.0.1.10:8000 check
    server app2 10.0.1.11:8000 check
    server app3 10.0.1.12:8000 check
    server app4 10.0.1.13:8000 check
```

**8. Monitoring & Logging:**

```python
# Using Sentry for error tracking
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

sentry_sdk.init(
    dsn=env('SENTRY_DSN'),
    integrations=[DjangoIntegration()],
    traces_sample_rate=0.1,  # 10% of transactions
    send_default_pii=True,
    environment=env('ENVIRONMENT', default='production'),
)

# Prometheus metrics
from prometheus_client import Counter, Histogram
import time

request_count = Counter(
    'django_request_count',
    'Total request count',
    ['method', 'endpoint', 'status']
)

request_duration = Histogram(
    'django_request_duration_seconds',
    'Request duration',
    ['method', 'endpoint']
)

class PrometheusMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        start = time.time()
        
        response = self.get_response(request)
        
        duration = time.time() - start
        
        request_count.labels(
            method=request.method,
            endpoint=request.path,
            status=response.status_code
        ).inc()
        
        request_duration.labels(
            method=request.method,
            endpoint=request.path
        ).observe(duration)
        
        return response
```

**9. Auto-scaling (AWS example):**

```yaml
# AWS ECS Task Definition
{
  "family": "myapp",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "myapp:latest",
      "cpu": 512,
      "memory": 1024,
      "essential": true,
      "environment": [
        {"name": "DJANGO_SETTINGS_MODULE", "value": "myproject.settings.production"}
      ],
      "secrets": [
        {"name": "SECRET_KEY", "valueFrom": "arn:aws:secretsmanager:..."}
      ],
      "portMappings": [
        {"containerPort": 8000, "protocol": "tcp"}
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8000/health/ || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3
      }
    }
  ]
}

# Auto Scaling Policy
{
  "TargetValue": 70.0,
  "PredefinedMetricType": "ECSServiceAverageCPUUtilization",
  "ScaleInCooldown": 300,
  "ScaleOutCooldown": 60
}
```

**10. CI/CD Pipeline (GitHub Actions):**

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
    
    - name: Run tests
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
      run: |
        python manage.py test
        coverage run --source='.' manage.py test
        coverage report
    
    - name: Run linting
      run: |
        flake8 .
        black --check .
  
  build:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
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
    
    steps:
    - name: Deploy to server
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.SERVER_HOST }}
        username: ${{ secrets.SERVER_USER }}
        key: ${{ secrets.SSH_PRIVATE_KEY }}
        script: |
          cd /var/www/myapp
          docker-compose pull
          docker-compose up -d
          docker exec myapp_web python manage.py migrate
          docker exec myapp_web python manage.py collectstatic --noinput
```

**Scaling Checklist:**
- ✅ Use Gunicorn/uWSGI with multiple workers
- ✅ Nginx as reverse proxy
- ✅ Redis for caching and session storage
- ✅ Celery for background tasks
- ✅ Database read replicas
- ✅ Connection pooling (pgbouncer)
- ✅ CDN for static files
- ✅ Load balancer for multiple app servers
- ✅ Auto-scaling based on metrics
- ✅ Monitoring and alerting (Sentry, Prometheus)
- ✅ CI/CD pipeline
- ✅ Database indexing and query optimization

---

## 7. ADVANCED DJANGO TOPICS

### Q10: Explain Django Channels and WebSocket implementation.
**Answer:**

**Django Channels** extends Django to handle WebSockets, HTTP2, and other protocols beyond HTTP.

**Installation:**
```bash
pip install channels channels-redis
```

**Configuration:**

```python
# settings.py
INSTALLED_APPS = [
    'daphne',  # Must be before django.contrib.staticfiles
    'django.contrib.staticfiles',
    # ... other apps
    'channels',
]

ASGI_APPLICATION = 'myproject.asgi.application'

CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            "hosts": [('127.0.0.1', 6379)],
        },
    },
}

# myproject/asgi.py
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
from channels.security.websocket import AllowedHostsOriginValidator

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

django_asgi_app = get_asgi_application()

from chat import routing

application = ProtocolTypeRouter({
    "http": django_asgi_app,
    "websocket": AllowedHostsOriginValidator(
        AuthMiddlewareStack(
            URLRouter(
                routing.websocket_urlpatterns
            )
        )
    ),
})
```

**Consumer (WebSocket Handler):**

```python
# chat/consumers.py
import json
from channels.generic.websocket import AsyncWebsocketConsumer
from channels.db import database_sync_to_async
from django.contrib.auth.models import User
from .models import Message, ChatRoom

class ChatConsumer(AsyncWebsocketConsumer):
    """WebSocket consumer for chat"""
    
    async def connect(self):
        """Called when WebSocket connects"""
        self.room_id = self.scope['url_route']['kwargs']['room_id']
        self.room_group_name = f'chat_{self.room_id}'
        
        # Join room group
        await self.channel_layer.group_add(
            self.room_group_name,
            self.channel_name
        )
        
        await self.accept()
        
        # Send user list
        users = await self.get_room_users()
        await self.send(text_data=json.dumps({
            'type': 'user_list',
            'users': users
        }))
    
    async def disconnect(self, close_code):
        """Called when WebSocket disconnects"""
        # Leave room group
        await self.channel_layer.group_discard(
            self.room_group_name,
            self.channel_name
        )
    
    async def receive(self, text_data):
        """Receive message from WebSocket"""
        data = json.loads(text_data)
        message_type = data.get('type')
        
        if message_type == 'chat_message':
            message = data['message']
            user = self.scope['user']
            
            # Save message to database
            saved_message = await self.save_message(
                self.room_id,
                user,
                message
            )
            
            # Send message to room group
            await self.channel_layer.group_send(
                self.room_group_name,
                {
                    'type': 'chat_message',
                    'message': message,
                    'user': user.username,
                    'timestamp': saved_message.created_at.isoformat()
                }
            )
        
        elif message_type == 'typing':
            # Broadcast typing indicator
            await self.channel_layer.group_send(
                self.room_group_name,
                {
                    'type': 'user_typing',
                    'user': self.scope['user'].username
                }
            )
    
    async def chat_message(self, event):
        """Send message to WebSocket"""
        await self.send(text_data=json.dumps({
            'type': 'chat_message',
            'message': event['message'],
            'user': event['user'],
            'timestamp': event['timestamp']
        }))
    
    async def user_typing(self, event):
        """Send typing indicator"""
        await self.send(text_data=json.dumps({
            'type': 'typing',
            'user': event['user']
        }))
    
    @database_sync_to_async
    def save_message(self, room_id, user, message):
        """Save message to database"""
        room = ChatRoom.objects.get(id=room_id)
        return Message.objects.create(
            room=room,
            user=user,
            content=message
        )
    
    @database_sync_to_async
    def get_room_users(self):
        """Get list of users in room"""
        room = ChatRoom.objects.get(id=self.room_id)
        return list(room.users.values_list('username', flat=True))

# chat/routing.py
from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(r'ws/chat/(?P<room_id>\w+)/_FRAME_OPTIONS = 'DENY'  # Prevent clickjacking

# HTTPS redirect
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# HSTS (HTTP Strict Transport Security)
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Referrer policy
SECURE_REFERRER_POLICY = 'same-origin'
```

**6. Input Validation & Sanitization:**

```python
from django import forms
from django.core.validators import EmailValidator, URLValidator, RegexValidator

class ArticleForm(forms.ModelForm):
    # Field validation
    title = forms.CharField(
        max_length=200,
        validators=[
            RegexValidator(
                regex=r'^[a-zA-Z0-9\s\-]+### Q5: Explain DRF ViewSets, Generic Views, and their differences.
**Answer:**

**1. Function-Based Views (FBV):**
```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated

@api_view(['GET', 'POST'])
@permission_classes([IsAuthenticated])
def article_list(request):
    """List articles or create new article"""
    if request.method == 'GET':
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

@api_view(['GET', 'PUT', 'DELETE'])
def article_detail(request, pk):
    """Retrieve, update or delete article"""
    try:
        article = Article.objects.get(pk=pk)
    except Article.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)
    
    if request.method == 'GET':
        serializer = ArticleSerializer(article)
        return Response(serializer.data)
    
    elif request.method == 'PUT':
        serializer = ArticleSerializer(article, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    elif request.method == 'DELETE':
        article.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**2. Class-Based Views (CBV):**
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class ArticleList(APIView):
    """List all articles or create new article"""
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True, context={'request': request})
        return Response(serializer.data)
    
    def post(self, request):
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class ArticleDetail(APIView):
    """Retrieve, update or delete article"""
    
    def get_object(self, pk):
        try:
            return Article.objects.get(pk=pk)
        except Article.DoesNotExist:
            raise Http404
    
    def get(self, request, pk):
        article = self.get_object(pk)
        serializer = ArticleSerializer(article)
        return Response(serializer.data)
    
    def put(self, request, pk):
        article = self.get_object(pk)
        serializer = ArticleSerializer(article, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def delete(self, request, pk):
        article = self.get_object(pk)
        article.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**3. Generic Views:**
```python
from rest_framework import generics
from rest_framework.permissions import IsAuthenticatedOrReadOnly

# List and Create
class ArticleList(generics.ListCreateAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def perform_create(self, serializer):
        """Customize creation"""
        serializer.save(author=self.request.user)

# Retrieve, Update, Delete
class ArticleDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]

# Other generic views:
# - ListAPIView: Read-only list
# - CreateAPIView: Create only
# - RetrieveAPIView: Read-only single object
# - UpdateAPIView: Update only
# - DestroyAPIView: Delete only
# - RetrieveUpdateAPIView: Read and update
```

**4. ViewSets:**
```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

class ArticleViewSet(viewsets.ModelViewSet):
    """Complete CRUD operations"""
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filterset_fields = ['author', 'is_published']
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'title']
    
    def get_queryset(self):
        """Customize queryset"""
        queryset = super().get_queryset()
        
        # Optimize queries
        queryset = queryset.select_related('author')
        queryset = queryset.prefetch_related('comments')
        
        # Filter by user for non-staff
        if not self.request.user.is_staff:
            queryset = queryset.filter(
                models.Q(is_published=True) | models.Q(author=self.request.user)
            )
        
        return queryset
    
    def get_serializer_class(self):
        """Use different serializers for different actions"""
        if self.action == 'list':
            return ArticleListSerializer
        elif self.action == 'retrieve':
            return ArticleDetailSerializer
        return ArticleSerializer
    
    def perform_create(self, serializer):
        """Set author on creation"""
        serializer.save(author=self.request.user)
    
    def perform_update(self, serializer):
        """Custom update logic"""
        serializer.save(updated_by=self.request.user)
    
    # Custom actions
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """Publish article"""
        article = self.get_object()
        article.is_published = True
        article.published_at = timezone.now()
        article.save()
        
        serializer = self.get_serializer(article)
        return Response(serializer.data)
    
    @action(detail=False, methods=['get'])
    def my_articles(self, request):
        """Get current user's articles"""
        articles = self.get_queryset().filter(author=request.user)
        serializer = self.get_serializer(articles, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['get'])
    def comments(self, request, pk=None):
        """Get article comments"""
        article = self.get_object()
        comments = article.comments.all()
        serializer = CommentSerializer(comments, many=True)
        return Response(serializer.data)

# URLs for ViewSet
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'articles', ArticleViewSet, basename='article')

urlpatterns = router.urls

# Generated URLs:
# GET    /articles/              -> list
# POST   /articles/              -> create
# GET    /articles/{pk}/         -> retrieve
# PUT    /articles/{pk}/         -> update
# PATCH  /articles/{pk}/         -> partial_update
# DELETE /articles/{pk}/         -> destroy
# POST   /articles/{pk}/publish/ -> publish (custom action)
# GET    /articles/my_articles/  -> my_articles (custom action)
```

**5. ReadOnlyModelViewSet:**
```python
class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    """Read-only viewset - only list and retrieve"""
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    
    # Only provides:
    # - list()
    # - retrieve()
```

**6. Custom ViewSet:**
```python
from rest_framework import viewsets, mixins

class CreateListRetrieveViewSet(
    mixins.CreateModelMixin,
    mixins.ListModelMixin,
    mixins.RetrieveModelMixin,
    viewsets.GenericViewSet
):
    """ViewSet that only allows create, list, and retrieve"""
    pass

class CommentViewSet(CreateListRetrieveViewSet):
    """Comments can only be created and viewed, not updated or deleted"""
    queryset = Comment.objects.all()
    serializer_class = CommentSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def perform_create(self, serializer):
        serializer.save(user=self.request.user)
```

**7. Advanced ViewSet Features:**
```python
from rest_framework import viewsets, filters
from django_filters.rest_framework import DjangoFilterBackend

class AdvancedArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    # Filtering
    filter_backends = [
        DjangoFilterBackend,
        filters.SearchFilter,
        filters.OrderingFilter
    ]
    filterset_fields = {
        'author': ['exact'],
        'is_published': ['exact'],
        'created_at': ['gte', 'lte'],
        'title': ['icontains']
    }
    search_fields = ['title', 'content', 'author__name']
    ordering_fields = ['created_at', 'title', 'updated_at']
    ordering = ['-created_at']
    
    # Pagination
    pagination_class = PageNumberPagination
    
    def get_permissions(self):
        """Different permissions for different actions"""
        if self.action in ['list', 'retrieve']:
            return [permissions.AllowAny()]
        elif self.action in ['create']:
            return [permissions.IsAuthenticated()]
        elif self.action in ['update', 'partial_update', 'destroy']:
            return [permissions.IsAuthenticated(), IsOwnerOrReadOnly()]
        return super().get_permissions()
    
    def get_throttles(self):
        """Different throttles for different actions"""
        if self.action == 'create':
            return [UserRateThrottle()]
        return super().get_throttles()
    
    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def like(self, request, pk=None):
        """Like an article"""
        article = self.get_object()
        user = request.user
        
        if article.likes.filter(id=user.id).exists():
            article.likes.remove(user)
            return Response({'status': 'unliked'})
        else:
            article.likes.add(user)
            return Response({'status': 'liked'})
    
    @action(detail=False, methods=['get'])
    def trending(self, request):
        """Get trending articles"""
        from django.db.models import Count
        
        articles = self.get_queryset().annotate(
            like_count=Count('likes')
        ).order_by('-like_count')[:10]
        
        serializer = self.get_serializer(articles, many=True)
        return Response(serializer.data)
```

**Comparison:**

| Feature | FBV | APIView | GenericView | ViewSet |
|---------|-----|---------|-------------|---------|
| Code Amount | Most | Moderate | Less | Least |
| Flexibility | Highest | High | Moderate | Low |
| Reusability | Low | Moderate | High | Highest |
| Best For | Custom logic | Custom APIs | Standard CRUD | REST APIs |
| URL Routing | Manual | Manual | Manual | Automatic |

**When to Use What:**

**Function-Based Views:**
- Simple, one-off endpoints
- Very custom logic
- Learning/prototyping

**APIView:**
- Need full control
- Custom HTTP methods
- Complex business logic

**Generic Views:**
- Standard CRUD operations
- Want some customization
- Don't need all methods

**ViewSets:**
- Full REST API for a model
- Standard CRUD with minimal customization
- Need automatic URL routing
- Want consistent API structure

---

## 3. DJANGO PERFORMANCE & OPTIMIZATION

### Q6: How do you optimize Django application performance?
**Answer:**

**1. Database Query Optimization:**

```python
# BAD: N+1 Query Problem
def get_articles_bad():
    articles = Article.objects.all()
    for article in articles:
        print(article.author.name)  # N queries
        print(article.category.name)  # N queries

# GOOD: Use select_related
def get_articles_good():
    articles = Article.objects.select_related(
        'author', 'category'
    ).all()  # 1 query with JOINs
    
    for article in articles:
        print(article.author.name)  # No extra query
        print(article.category.name)  # No extra query

# GOOD: Use prefetch_related for reverse relations
def get_authors_with_articles():
    authors = Author.objects.prefetch_related('articles').all()
    
    for author in authors:
        for article in author.articles.all():  # No extra queries
            print(article.title)

# Advanced prefetch
from django.db.models import Prefetch

def get_authors_with_published_articles():
    published_articles = Article.objects.filter(is_published=True)
    
    authors = Author.objects.prefetch_related(
        Prefetch('articles', queryset=published_articles, to_attr='published_articles')
    ).all()
    
    for author in authors:
        for article in author.published_articles:  # Use prefetched data
            print(article.title)
```

**2. Caching Strategies:**

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'KEY_PREFIX': 'myapp',
        'TIMEOUT': 300,
    }
}

# Low-level cache API
from django.core.cache import cache

def get_article(article_id):
    cache_key = f'article_{article_id}'
    article = cache.get(cache_key)
    
    if article is None:
        article = Article.objects.select_related('author').get(id=article_id)
        cache.set(cache_key, article, timeout=3600)  # 1 hour
    
    return article

# Invalidate cache on update
from django.db.models.signals import post_save, post_delete

@receiver([post_save, post_delete], sender=Article)
def invalidate_article_cache(sender, instance, **kwargs):
    cache_key = f'article_{instance.id}'
    cache.delete(cache_key)

# Template fragment caching
{% load cache %}
{% cache 3600 sidebar %}
    <!-- Expensive sidebar rendering -->
    {% for category in categories %}
        <li>{{ category.name }}</li>
    {% endfor %}
{% endcache %}

# Per-site cache
MIDDLEWARE = [
    'django.middleware.cache.UpdateCacheMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.cache.FetchFromCacheMiddleware',
]

CACHE_MIDDLEWARE_ALIAS = 'default'
CACHE_MIDDLEWARE_SECONDS = 600
CACHE_MIDDLEWARE_KEY_PREFIX = 'myapp'

# Per-view cache
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # Cache for 15 minutes
def article_list(request):
    articles = Article.objects.all()
    return render(request, 'articles/list.html', {'articles': articles})

# Cache with conditions
from django.views.decorators.vary import vary_on_cookie

@cache_page(60 * 15)
@vary_on_cookie
def my_view(request):
    # Cache varies based on cookie
    pass
```

**3. Database Indexing:**

```python
class Article(models.Model):
    title = models.CharField(max_length=200, db_index=True)  # Simple index
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    published_date = models.DateTimeField()
    is_published = models.BooleanField(default=False)
    slug = models.SlugField(unique=True)  # Automatically indexed
    
    class Meta:
        indexes = [
            # Composite index for common queries
            models.Index(fields=['author', '-published_date']),
            
            # Covering index
            models.Index(fields=['is_published', 'published_date', 'title']),
            
            # Partial index (PostgreSQL)
            models.Index(
                fields=['title'],
                name='published_title_idx',
                condition=models.Q(is_published=True)
            ),
            
            # Expression index (PostgreSQL)
            models.Index(
                Lower('title').desc(),
                name='title_lower_idx'
            ),
        ]

# Find missing indexes
python manage.py sqlmigrate myapp 0001 | grep "CREATE INDEX"

# Analyze queries
from django.db import connection

def show_queries():
    for query in connection.queries:
        print(query['sql'])
```

**4. Query Optimization:**

```python
# Use only() to fetch specific fields
articles = Article.objects.only('id', 'title', 'author_id')

# Use defer() to exclude heavy fields
articles = Article.objects.defer('content')

# Use values() or values_list() for simple data
titles = Article.objects.values_list('title', flat=True)

# Use exists() instead of count()
if Article.objects.filter(author=user).exists():  # Fast
    pass

# Use iterator() for large datasets
for article in Article.objects.iterator(chunk_size=1000):
    process_article(article)  # Memory efficient

# Use bulk operations
articles = [Article(title=f'Article {i}') for i in range(1000)]
Article.objects.bulk_create(articles, batch_size=100)

# Use update() instead of save() for updates
Article.objects.filter(author=user).update(is_published=True)

# Use annotate for computed fields
from django.db.models import Count, Avg

authors = Author.objects.annotate(
    article_count=Count('articles'),
    avg_rating=Avg('articles__rating')
)

# Use aggregation
from django.db.models import Sum

total_views = Article.objects.aggregate(total=Sum('view_count'))
```

**5. Lazy Loading & Eager Loading:**

```python
# Lazy loading (queries execute when needed)
articles = Article.objects.filter(is_published=True)  # No query yet
for article in articles:  # Query executes here
    print(article.title)

# Force evaluation
articles = list(Article.objects.all())  # Execute now

# Eager loading with select_related
article = Article.objects.select_related('author', 'category').get(id=1)

# Eager loading with prefetch_related
authors = Author.objects.prefetch_related(
    'articles',
    'articles__comments'
).all()
```

**6. Asynchronous Processing:**

```python
# Use Celery for background tasks
from celery import shared_task

@shared_task
def send_newsletter(article_id):
    article = Article.objects.get(id=article_id)
    users = User.objects.filter(subscribed=True)
    
    for user in users:
        send_email(user.email, article)

# In view
def publish_article(request, pk):
    article = Article.objects.get(pk=pk)
    article.is_published = True
    article.save()
    
    # Send newsletter asynchronously
    send_newsletter.delay(article.id)
    
    return JsonResponse({'status': 'published'})

# Periodic tasks
from celery.schedules import crontab

@app.task
def cleanup_old_data():
    threshold = timezone.now() - timedelta(days=30)
    Article.objects.filter(created_at__lt=threshold, is_published=False).delete()

# Celery beat schedule
app.conf.beat_schedule = {
    'cleanup-every-day': {
        'task': 'myapp.tasks.cleanup_old_data',
        'schedule': crontab(hour=2, minute=0),
    },
}
```

**7. Connection Pooling:**

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'mypassword',
        'HOST': 'localhost',
        'PORT': '5432',
        'CONN_MAX_AGE': 600,  # Connection pooling (10 minutes)
        'OPTIONS': {
            'connect_timeout': 10,
            'options': '-c statement_timeout=30000'  # 30 seconds
        }
    }
}
```

**8. Middleware Optimization:**

```python
# Custom middleware for performance
class PerformanceMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        import time
        from django.db import connection
        
        # Start timing
        start_time = time.time()
        start_queries = len(connection.queries)
        
        # Process request
        response = self.get_response(request)
        
        # Calculate stats
        end_time = time.time()
        total_time = end_time - start_time
        num_queries = len(connection.queries) - start_queries
        
        # Add headers
        response['X-Response-Time'] = f'{total_time:.3f}s'
        response['X-Query-Count'] = str(num_queries)
        
        # Log slow requests
        if total_time > 1.0:  # Over 1 second
            logger.warning(
                f'Slow request: {request.path} took {total_time:.3f}s '
                f'with {num_queries} queries'
            )
        
        return response
```

**9. Static Files & CDN:**

```python
# settings.py
STATIC_URL = 'https://cdn.example.com/static/'
MEDIA_URL = 'https://cdn.example.com/media/'

# Use WhiteNoise for static files
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # Add this
    # ... other middleware
]

STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

**10. Monitoring & Profiling:**

```python
# Django Debug Toolbar
INSTALLED_APPS = [
    # ...
    'debug_toolbar',
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    # ...
]

INTERNAL_IPS = ['127.0.0.1']

# Django Silk for profiling
INSTALLED_APPS = [
    # ...
    'silk',
]

MIDDLEWARE = [
    'silk.middleware.SilkyMiddleware',
    # ...
]

# Custom profiling
import cProfile
import pstats

def profile_view(func):
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()
        
        result = func(*args, **kwargs)
        
        profiler.disable()
        stats = pstats.Stats(profiler)
        stats.sort_stats('cumulative')
        stats.print_stats(20)
        
        return result
    return wrapper

@profile_view
def expensive_view(request):
    # Your view logic
    pass
```

**Performance Checklist:**
- ✅ Use `select_related()` and `prefetch_related()`
- ✅ Add database indexes
- ✅ Implement caching (Redis)
- ✅ Use `only()` and `defer()` appropriately
- ✅ Bulk operations for mass updates
- ✅ Async tasks for heavy operations (Celery)
- ✅ Connection pooling
- ✅ CDN for static files
- ✅ Database query optimization
- ✅ Monitor with Django Debug Toolbar
- ✅ Use `iterator()` for large datasets
- ✅ Pagination for list views

---

## 4. DJANGO SECURITY

### Q7: Explain Django security best practices and common vulnerabilities.
**Answer:**

**1. SQL Injection Prevention:**

```python
# BAD - Vulnerable to SQL injection
def search_articles_bad(request):
    query = request.GET.get('q')
    articles = Article.objects.raw(
        f"SELECT * FROM articles WHERE title LIKE '%{query}%'"
    )
    return render(request, 'articles.html', {'articles': articles})

# GOOD - Use parameterized queries
def search_articles_good(request):
    query = request.GET.get('q')
    articles = Article.objects.raw(
        "SELECT * FROM articles WHERE title LIKE %s",
        [f'%{query}%']
    )
    return render(request, 'articles.html', {'articles': articles})

# BEST - Use ORM
def search_articles_best(request):
    query = request.GET.get('q')
    articles = Article.objects.filter(title__icontains=query)
    return render(request, 'articles.html', {'articles': articles})
```

**2. Cross-Site Scripting (XSS) Prevention:**

```python
# Django templates auto-escape by default
{% autoescape on %}
    {{ user_input }}  # Automatically escaped
{% endautoescape %}

# Explicitly mark as safe (use cautiously)
from django.utils.safestring import mark_safe

def render_html(content):
    # Sanitize first!
    import bleach
    clean_content = bleach.clean(
        content,
        tags=['p', 'b', 'i', 'u', 'a'],
        attributes={'a': ['href', 'title']},
        strip=True
    )
    return mark_safe(clean_content)

# In template
{{ content|safe }}  # Only if you're sure it's safe!

# JSON responses
from django.http import JsonResponse

def api_view(request):
    data = {'user_input': request.GET.get('input')}
    return JsonResponse(data)  # Automatically escapes
```

**3. Cross-Site Request Forgery (CSRF) Protection:**

```python
# Django CSRF protection is enabled by default
MIDDLEWARE = [
    'django.middleware.csrf.CsrfViewMiddleware',  # Required
    # ...
]

# In forms
<form method="post">
    {% csrf_token %}
    <!-- form fields -->
</form>

# AJAX requests
// Get CSRF token
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

const csrftoken = getCookie('csrftoken');

// Include in AJAX
fetch('/api/endpoint/', {
    method: 'POST',
    headers: {
        'X-CSRFToken': csrftoken,
        'Content-Type': 'application/json',
    },
    body: JSON.stringify(data)
});

# Exempt specific views (use cautiously)
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def webhook_view(request):
    # For third-party webhooks
    pass

# DRF CSRF with Session Authentication
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
}
```

**4. Authentication & Authorization:**

```python
# Strong password validation
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {'min_length': 12}
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]

# Custom password validator
from django.core.exceptions import ValidationError

class SpecialCharacterValidator:
    def validate(self, password, user=None):
        if not any(char in '!@#$%^&*()' for char in password):
            raise ValidationError(
                "Password must contain at least one special character",
                code='password_no_special',
            )
    
    def get_help_text(self):
        return "Your password must contain at least one special character (!@#$%^&*())"

# Secure authentication views
from django.contrib.auth import authenticate, login
from django.contrib.auth.decorators import login_required
from django.contrib.auth.mixins import LoginRequiredMixin

@login_required
def protected_view(request):
    return render(request, 'protected.html')

class ProtectedView(LoginRequiredMixin, View):
    login_url = '/login/'
    redirect_field_name = 'next'
    
    def get(self, request):
        return render(request, 'protected.html')

# Permission-based access
from django.contrib.auth.decorators import permission_required

@permission_required('myapp.can_publish', raise_exception=True)
def publish_article(request, pk):
    article = Article.objects.get(pk=pk)
    article.is_published = True
    article.save()
    return redirect('article_detail', pk=pk)

# Custom permissions
class Article(models.Model):
    # ... fields ...
    
    class Meta:
        permissions = [
            ("can_publish", "Can publish articles"),
            ("can_feature", "Can feature articles"),
        ]
```

**5. Secure Session Management:**

```python
# settings.py

# Session security
SESSION_COOKIE_SECURE = True  # HTTPS only
SESSION_COOKIE_HTTPONLY = True  # Not accessible via JavaScript
SESSION_COOKIE_SAMESITE = 'Strict'  # CSRF protection
SESSION_COOKIE_AGE = 3600  # 1 hour

# CSRF security
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_HTTPONLY = True
CSRF_COOKIE_SAMESITE = 'Strict'

# Security headers
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X# Django Senior Backend Developer Interview Preparation Guide
## Comprehensive Guide for Senior Django Developer Interviews

---

## 1. DJANGO CORE CONCEPTS

### Q1: Explain Django's MTV architecture and how it differs from MVC.
**Answer:**

**Django MTV (Model-Template-View):**
- **Model:** Data layer - defines database structure
- **Template:** Presentation layer - HTML with template tags
- **View:** Business logic - processes requests and returns responses

**Traditional MVC (Model-View-Controller):**
- **Model:** Data layer
- **View:** Presentation layer
- **Controller:** Business logic

**Key Difference:**
Django's "View" is equivalent to MVC's "Controller", and Django's "Template" is equivalent to MVC's "View". Django calls it MTV but it's essentially MVC with different naming.

```python
# models.py (Model)
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    published = models.BooleanField(default=False)
    
    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['published', '-created_at']),
        ]
    
    def __str__(self):
        return self.title

# views.py (View - Controller logic)
from django.shortcuts import render, get_object_or_404
from django.http import JsonResponse
from .models import Article

def article_list(request):
    """List all published articles"""
    articles = Article.objects.filter(published=True).select_related('author')
    return render(request, 'articles/list.html', {'articles': articles})

def article_detail(request, pk):
    """Display single article"""
    article = get_object_or_404(Article, pk=pk, published=True)
    return render(request, 'articles/detail.html', {'article': article})

# templates/articles/list.html (Template - Presentation)
{% extends 'base.html' %}

{% block content %}
<h1>Articles</h1>
{% for article in articles %}
    <div class="article">
        <h2>{{ article.title }}</h2>
        <p>By {{ article.author.username }} on {{ article.created_at|date:"F d, Y" }}</p>
        <a href="{% url 'article_detail' article.pk %}">Read more</a>
    </div>
{% endfor %}
{% endblock %}

# urls.py (URL Configuration)
from django.urls import path
from . import views

urlpatterns = [
    path('articles/', views.article_list, name='article_list'),
    path('articles/<int:pk>/', views.article_detail, name='article_detail'),
]
```

---

### Q2: Explain Django's ORM and query optimization techniques.
**Answer:**

**Django ORM Basics:**

```python
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    bio = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name='books')
    published_date = models.DateField()
    isbn = models.CharField(max_length=13, unique=True)
    price = models.DecimalField(max_digits=6, decimal_places=2)
    
    class Meta:
        ordering = ['-published_date']
        indexes = [
            models.Index(fields=['author', '-published_date']),
        ]

class Review(models.Model):
    book = models.ForeignKey(Book, on_delete=models.CASCADE, related_name='reviews')
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    rating = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
    comment = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ['book', 'user']
```

**Query Optimization Techniques:**

**1. N+1 Query Problem:**
```python
# BAD - N+1 queries
books = Book.objects.all()  # 1 query
for book in books:
    print(book.author.name)  # N queries (one per book)

# GOOD - select_related (for ForeignKey, OneToOne)
books = Book.objects.select_related('author').all()  # 1 query with JOIN
for book in books:
    print(book.author.name)  # No additional queries

# Generated SQL:
# SELECT book.*, author.* FROM book 
# INNER JOIN author ON book.author_id = author.id

# GOOD - prefetch_related (for ManyToMany, reverse ForeignKey)
authors = Author.objects.prefetch_related('books').all()
for author in authors:
    for book in author.books.all():  # No additional queries
        print(book.title)

# Uses 2 queries:
# 1. SELECT * FROM author
# 2. SELECT * FROM book WHERE author_id IN (1, 2, 3, ...)
```

**2. Only/Defer:**
```python
# Only fetch specific fields
books = Book.objects.only('title', 'price')  # Fetch only title and price

# Defer heavy fields
books = Book.objects.defer('description')  # Fetch all except description

# With relations
books = Book.objects.select_related('author').only(
    'title', 'price', 'author__name'
)
```

**3. Aggregation:**
```python
from django.db.models import Count, Avg, Sum, Max, Min, F, Q

# Count books per author
authors = Author.objects.annotate(
    book_count=Count('books'),
    avg_price=Avg('books__price')
).filter(book_count__gt=0)

for author in authors:
    print(f"{author.name}: {author.book_count} books, avg price: {author.avg_price}")

# Complex aggregation
stats = Book.objects.aggregate(
    total_books=Count('id'),
    avg_price=Avg('price'),
    max_price=Max('price'),
    min_price=Min('price'),
    total_revenue=Sum('price')
)

# Annotate with conditions
books = Book.objects.annotate(
    high_ratings_count=Count('reviews', filter=Q(reviews__rating__gte=4))
)
```

**4. F() expressions:**
```python
# Update using database-level operations
from django.db.models import F

# Atomic update
Book.objects.filter(id=1).update(price=F('price') * 1.1)  # Increase by 10%

# Comparison
expensive_books = Book.objects.filter(price__gt=F('author__avg_book_price'))

# Arithmetic
books = Book.objects.annotate(
    discount_price=F('price') * 0.9,
    days_since_published=Now() - F('published_date')
)
```

**5. Q objects (Complex queries):**
```python
from django.db.models import Q

# OR queries
books = Book.objects.filter(
    Q(title__icontains='python') | Q(title__icontains='django')
)

# Complex conditions
books = Book.objects.filter(
    Q(price__lt=30) & (Q(author__name='John') | Q(published_date__year=2023))
)

# NOT
books = Book.objects.filter(~Q(author__name='John'))
```

**6. Bulk Operations:**
```python
# Bulk create
books = [
    Book(title=f'Book {i}', author=author, price=10 + i)
    for i in range(1000)
]
Book.objects.bulk_create(books, batch_size=100)

# Bulk update
books = Book.objects.filter(published_date__year=2023)
for book in books:
    book.price *= 1.1
Book.objects.bulk_update(books, ['price'], batch_size=100)

# Update all at once
Book.objects.filter(published_date__year=2023).update(price=F('price') * 1.1)

# Bulk delete
Book.objects.filter(published_date__year__lt=2020).delete()
```

**7. Raw SQL & Database Functions:**
```python
from django.db.models.functions import Concat, Lower, Upper, Coalesce

# Database functions
authors = Author.objects.annotate(
    full_name=Concat('first_name', models.Value(' '), 'last_name'),
    email_lower=Lower('email')
)

# Raw SQL (when necessary)
books = Book.objects.raw('SELECT * FROM book WHERE price > %s', [50])

# Execute raw SQL
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("UPDATE book SET price = price * 1.1 WHERE author_id = %s", [author_id])
```

**8. Database Indexes:**
```python
class Book(models.Model):
    title = models.CharField(max_length=200, db_index=True)  # Simple index
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    published_date = models.DateField()
    
    class Meta:
        indexes = [
            # Composite index
            models.Index(fields=['author', '-published_date']),
            
            # Partial index (PostgreSQL)
            models.Index(
                fields=['title'],
                name='title_active_idx',
                condition=Q(is_active=True)
            ),
            
            # Expression index (PostgreSQL)
            models.Index(
                Lower('title').desc(),
                name='title_lower_idx'
            ),
        ]
```

**9. Query Performance Tips:**
```python
# Use exists() instead of count()
if Book.objects.filter(author=author).exists():  # Fast
    pass
# vs
if Book.objects.filter(author=author).count() > 0:  # Slower

# Use iterator() for large querysets
for book in Book.objects.iterator(chunk_size=1000):
    process_book(book)  # Memory efficient

# Use values() or values_list() for simple data
book_titles = Book.objects.values_list('title', flat=True)
# Returns: ['Book 1', 'Book 2', ...]

# Use explain() to analyze queries
print(Book.objects.filter(price__gt=50).explain())
```

**10. Caching:**
```python
from django.core.cache import cache
from django.views.decorators.cache import cache_page

# Query result caching
def get_books():
    cache_key = 'all_books'
    books = cache.get(cache_key)
    
    if books is None:
        books = list(Book.objects.select_related('author').all())
        cache.set(cache_key, books, timeout=3600)  # 1 hour
    
    return books

# View caching
@cache_page(60 * 15)  # Cache for 15 minutes
def book_list(request):
    books = Book.objects.all()
    return render(request, 'books/list.html', {'books': books})

# Template fragment caching
{% load cache %}
{% cache 3600 book_list %}
    <!-- Expensive template rendering -->
{% endcache %}
```

---

### Q3: Explain Django signals and when to use them.
**Answer:**

**What are Django Signals:**
Signals allow decoupled applications to get notified when actions occur elsewhere in the framework.

**Built-in Signals:**

```python
from django.db.models.signals import (
    pre_save, post_save,
    pre_delete, post_delete,
    m2m_changed
)
from django.contrib.auth.signals import (
    user_logged_in, user_logged_out, user_login_failed
)
from django.core.signals import request_started, request_finished
from django.dispatch import receiver

# models.py
class Profile(models.Model):
    user = models.OneToOneField('auth.User', on_delete=models.CASCADE)
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', null=True)
    created_at = models.DateTimeField(auto_now_add=True)

# signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth.models import User

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    """Automatically create profile when user is created"""
    if created:
        Profile.objects.create(user=instance)
        print(f"Profile created for {instance.username}")

@receiver(post_save, sender=User)
def save_user_profile(sender, instance, **kwargs):
    """Save profile when user is saved"""
    if hasattr(instance, 'profile'):
        instance.profile.save()

# Alternative: Connect without decorator
def user_logged_in_handler(sender, request, user, **kwargs):
    """Track user login"""
    LoginLog.objects.create(
        user=user,
        ip_address=request.META.get('REMOTE_ADDR'),
        user_agent=request.META.get('HTTP_USER_AGENT')
    )

user_logged_in.connect(user_logged_in_handler)
```

**Signal Types:**

**1. Model Signals:**
```python
from django.db.models.signals import pre_save, post_save, pre_delete, post_delete

class Article(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    content = models.TextField()
    status = models.CharField(max_length=20)

@receiver(pre_save, sender=Article)
def generate_slug(sender, instance, **kwargs):
    """Generate slug before saving"""
    if not instance.slug:
        from django.utils.text import slugify
        instance.slug = slugify(instance.title)

@receiver(post_save, sender=Article)
def notify_on_publish(sender, instance, created, **kwargs):
    """Send notification when article is published"""
    if not created and instance.status == 'published':
        # Check if status changed
        try:
            old_instance = Article.objects.get(pk=instance.pk)
            if old_instance.status != 'published':
                send_publication_notification(instance)
        except Article.DoesNotExist:
            pass

@receiver(pre_delete, sender=Article)
def backup_before_delete(sender, instance, **kwargs):
    """Backup article before deletion"""
    ArticleBackup.objects.create(
        original_id=instance.id,
        title=instance.title,
        content=instance.content,
        deleted_at=timezone.now()
    )

@receiver(post_delete, sender=Article)
def cleanup_files(sender, instance, **kwargs):
    """Delete associated files"""
    if instance.image:
        instance.image.delete(save=False)
```

**2. Many-to-Many Signals:**
```python
from django.db.models.signals import m2m_changed

class Course(models.Model):
    title = models.CharField(max_length=200)
    students = models.ManyToManyField('auth.User', related_name='courses')

@receiver(m2m_changed, sender=Course.students.through)
def notify_course_enrollment(sender, instance, action, pk_set, **kwargs):
    """Notify when students are added/removed"""
    if action == 'post_add':
        # Students added
        students = User.objects.filter(pk__in=pk_set)
        for student in students:
            send_enrollment_email(student, instance)
    
    elif action == 'post_remove':
        # Students removed
        students = User.objects.filter(pk__in=pk_set)
        for student in students:
            send_unenrollment_email(student, instance)
    
    elif action == 'pre_clear':
        # All students about to be removed
        print(f"Clearing all students from {instance.title}")
```

**3. Request/Response Signals:**
```python
from django.core.signals import request_started, request_finished

@receiver(request_started)
def log_request_started(sender, environ, **kwargs):
    """Log when request starts"""
    print(f"Request started: {environ.get('PATH_INFO')}")

@receiver(request_finished)
def log_request_finished(sender, **kwargs):
    """Log when request finishes"""
    print("Request finished")
```

**Custom Signals:**

```python
# signals.py
from django.dispatch import Signal

# Define custom signal
payment_completed = Signal()  # No providing_args in Django 4.0+

# models.py
class Order(models.Model):
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)

class Payment(models.Model):
    order = models.ForeignKey(Order, on_delete=models.CASCADE)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)

# views.py
from .signals import payment_completed

def process_payment(request):
    # ... payment processing ...
    
    if payment.status == 'completed':
        # Send signal
        payment_completed.send(
            sender=Payment,
            payment=payment,
            order=payment.order
        )
    
    return JsonResponse({'status': 'success'})

# handlers.py
from django.dispatch import receiver
from .signals import payment_completed

@receiver(payment_completed)
def update_order_status(sender, payment, order, **kwargs):
    """Update order status when payment completes"""
    order.status = 'paid'
    order.save()

@receiver(payment_completed)
def send_receipt(sender, payment, order, **kwargs):
    """Send receipt email"""
    send_email(
        to=order.user.email,
        subject='Payment Receipt',
        template='emails/receipt.html',
        context={'payment': payment, 'order': order}
    )

@receiver(payment_completed)
def update_inventory(sender, payment, order, **kwargs):
    """Update inventory after payment"""
    for item in order.items.all():
        item.product.stock -= item.quantity
        item.product.save()
```

**When to Use Signals:**

**Good Use Cases:**
1. Creating related objects (Profile when User created)
2. Logging and auditing
3. Cache invalidation
4. Sending notifications
5. Triggering background tasks
6. Keeping data in sync across apps

**When NOT to Use Signals:**
1. Simple operations that can be in save() method
2. When direct code is clearer
3. Heavy processing (use Celery instead)
4. When testing becomes difficult

**Better Alternatives:**

```python
# Instead of signal for simple operations
class User(models.Model):
    email = models.EmailField()
    
    def save(self, *args, **kwargs):
        # Direct operation in save()
        self.email = self.email.lower()
        super().save(*args, **kwargs)
        
        # Create profile if needed
        if not hasattr(self, 'profile'):
            Profile.objects.create(user=self)

# Instead of signal for heavy operations, use Celery
from celery import shared_task

@shared_task
def send_welcome_email(user_id):
    user = User.objects.get(id=user_id)
    # Send email (async)

# In view
def register_user(request):
    user = User.objects.create(...)
    send_welcome_email.delay(user.id)  # Background task
```

**Signal Best Practices:**

```python
# 1. Always disconnect in tests
from django.test import TestCase
from django.db.models.signals import post_save

class MyTestCase(TestCase):
    def setUp(self):
        post_save.disconnect(create_user_profile, sender=User)
    
    def tearDown(self):
        post_save.connect(create_user_profile, sender=User)

# 2. Use dispatch_uid to prevent duplicate signals
@receiver(post_save, sender=User, dispatch_uid='create_user_profile')
def create_user_profile(sender, instance, created, **kwargs):
    pass

# 3. Be careful with exceptions
@receiver(post_save, sender=User)
def safe_signal_handler(sender, instance, **kwargs):
    try:
        # Your code
        pass
    except Exception as e:
        logger.error(f"Signal failed: {e}")
        # Don't let signal errors break the save

# 4. Register signals in AppConfig
# apps.py
from django.apps import AppConfig

class MyAppConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'myapp'
    
    def ready(self):
        import myapp.signals  # Import signals
```

---

## 2. DJANGO REST FRAMEWORK (DRF)

### Q4: Explain DRF serializers and their types.
**Answer:**

**Serializer Types:**

**1. Basic Serializer:**
```python
from rest_framework import serializers

class ArticleSerializer(serializers.Serializer):
    """Manual field definition"""
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    author = serializers.CharField(max_length=100)
    published_date = serializers.DateTimeField()
    is_published = serializers.BooleanField(default=False)
    
    def create(self, validated_data):
        """Create new instance"""
        return Article.objects.create(**validated_data)
    
    def update(self, instance, validated_data):
        """Update existing instance"""
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.is_published = validated_data.get('is_published', instance.is_published)
        instance.save()
        return instance
```

**2. ModelSerializer (Most Common):**
```python
from rest_framework import serializers
from .models import Article, Author, Comment

class AuthorSerializer(serializers.ModelSerializer):
    """Automatically creates fields from model"""
    book_count = serializers.SerializerMethodField()
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'email', 'bio', 'book_count']
        read_only_fields = ['id']
    
    def get_book_count(self, obj):
        """Custom method field"""
        return obj.books.count()

class ArticleSerializer(serializers.ModelSerializer):
    # Custom fields
    author_name = serializers.CharField(source='author.name', read_only=True)
    comment_count = serializers.SerializerMethodField()
    
    # Nested serializer
    author = AuthorSerializer(read_only=True)
    author_id = serializers.IntegerField(write_only=True)
    
    class Meta:
        model = Article
        fields = [
            'id', 'title', 'content', 'author', 'author_id',
            'author_name', 'published_date', 'is_published',
            'comment_count', 'created_at', 'updated_at'
        ]
        read_only_fields = ['id', 'created_at', 'updated_at']
        extra_kwargs = {
            'content': {'write_only': True},  # Don't return in response
            'title': {'required': True, 'allow_blank': False}
        }
    
    def get_comment_count(self, obj):
        return obj.comments.count()
    
    def validate_title(self, value):
        """Field-level validation"""
        if len(value) < 5:
            raise serializers.ValidationError("Title must be at least 5 characters")
        return value
    
    def validate(self, data):
        """Object-level validation"""
        if data.get('is_published') and not data.get('content'):
            raise serializers.ValidationError("Published articles must have content")
        return data

class CommentSerializer(serializers.ModelSerializer):
    user_name = serializers.CharField(source='user.username', read_only=True)
    
    class Meta:
        model = Comment
        fields = ['id', 'article', 'user', 'user_name', 'text', 'created_at']
        read_only_fields = ['id', 'created_at', 'user']
```

**3. Nested Serializers:**
```python
class ArticleDetailSerializer(serializers.ModelSerializer):
    """Detailed article with nested comments"""
    author = AuthorSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'author', 'comments', 'published_date']

# Usage
article = Article.objects.prefetch_related('comments', 'comments__user').get(pk=1)
serializer = ArticleDetailSerializer(article)
print(serializer.data)
# Output:
# {
#     'id': 1,
#     'title': 'My Article',
#     'content': '...',
#     'author': {'id': 1, 'name': 'John Doe', ...},
#     'comments': [
#         {'id': 1, 'text': 'Great article!', ...},
#         {'id': 2, 'text': 'Thanks for sharing', ...}
#     ]
# }
```

**4. Writable Nested Serializers:**
```python
class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'isbn']

class AuthorWithBooksSerializer(serializers.ModelSerializer):
    books = BookSerializer(many=True)
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'email', 'books']
    
    def create(self, validated_data):
        """Handle nested creation"""
        books_data = validated_data.pop('books')
        author = Author.objects.create(**validated_data)
        
        for book_data in books_data:
            Book.objects.create(author=author, **book_data)
        
        return author
    
    def update(self, instance, validated_data):
        """Handle nested updates"""
        books_data = validated_data.pop('books', None)
        
        # Update author fields
        instance.name = validated_data.get('name', instance.name)
        instance.email = validated_data.get('email', instance.email)
        instance.save()
        
        # Update books
        if books_data is not None:
            # Simple approach: delete and recreate
            instance.books.all().delete()
            for book_data in books_data:
                Book.objects.create(author=instance, **book_data)
        
        return instance
```

**5. Dynamic Fields:**
```python
class DynamicFieldsSerializer(serializers.ModelSerializer):
    """Serializer that can exclude fields"""
    
    def __init__(self, *args, **kwargs):
        # Extract fields argument
        fields = kwargs.pop('fields', None)
        exclude = kwargs.pop('exclude', None)
        
        super().__init__(*args, **kwargs)
        
        if fields is not None:
            # Drop fields not in `fields`
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)
        
        if exclude is not None:
            # Drop fields in `exclude`
            for field_name in exclude:
                self.fields.pop(field_name, None)

class ArticleSerializer(DynamicFieldsSerializer):
    class Meta:
        model = Article
        fields = '__all__'

# Usage
# Only include specific fields
serializer = ArticleSerializer(article, fields=['id', 'title'])

# Exclude specific fields
serializer = ArticleSerializer(article, exclude=['content'])
```

**6. Context and Custom Methods:**
```python
class ArticleSerializer(serializers.ModelSerializer):
    is_owner = serializers.SerializerMethodField()
    can_edit = serializers.SerializerMethodField()
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'author', 'is_owner', 'can_edit']
    
    def get_is_owner(self, obj):
        """Check if current user is owner"""
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.author == request.user
        return False
    
    def get_can_edit(self, obj):
        """Check if user can edit"""
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.author == request.user or request.user.is_staff
        return False

# In view
serializer = ArticleSerializer(article, context={'request': request})
```

**7. Validation:**
```python
class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = '__all__'
    
    def validate_title(self, value):
        """Validate single field"""
        if 'badword' in value.lower():
            raise serializers.ValidationError("Title contains inappropriate content")
        return value
    
    def validate(self, data):
        """Validate multiple fields together"""
        if data.get('is_published'):
            if not data.get('content'):
                raise serializers.ValidationError("Published articles must have content")
            if not data.get('author'):
                raise serializers.ValidationError("Published articles must have an author")
        
        # Check uniqueness with custom logic
        if Article.objects.filter(
            title=data.get('title'),
            author=data.get('author')
        ).exclude(pk=self.instance.pk if self.instance else None).exists():
            raise serializers.ValidationError("Article with this title already exists for this author")
        
        return data
    
    def validate_published_date(self, value):
        """Validate date"""
        from django.utils import timezone
        if value > timezone.now():
            raise serializers.ValidationError("Published date cannot be in the future")
        return value
```

**8. Performance Optimization:**
```python
class OptimizedArticleSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'author', 'created_at']
    
    @classmethod
    def setup_eager_loading(cls, queryset):
        """Optimize queryset for serializer"""
        queryset = queryset.select_related('author')
        queryset = queryset.prefetch_related('comments')
        return queryset

# In viewset
class ArticleViewSet(viewsets.ModelViewSet):
    serializer_class = OptimizedArticleSerializer
    
    def get_queryset(self):
        queryset = Article.objects.all()
        # Apply eager loading
        queryset = self.get_serializer_class().setup_eager_loading(queryset)
        return queryset
```

---

### Q5: Explain DRF ViewS,
                message='Title can only contain letters, numbers, spaces, and hyphens'
            )
        ]
    )
    
    email = forms.EmailField(validators=[EmailValidator()])
    website = forms.URLField(validators=[URLValidator()])
    
    class Meta:
        model = Article
        fields = ['title', 'content', 'email', 'website']
    
    def clean_title(self):
        """Custom field validation"""
        title = self.cleaned_data['title']
        
        # Check for bad words
        bad_words = ['spam', 'scam']
        if any(word in title.lower() for word in bad_words):
            raise forms.ValidationError("Title contains prohibited words")
        
        return title
    
    def clean(self):
        """Form-wide validation"""
        cleaned_data = super().clean()
        
        # Cross-field validation
        if cleaned_data.get('is_published') and not cleaned_data.get('content'):
            raise forms.ValidationError("Published articles must have content")
        
        return cleaned_data

# Sanitize HTML input
import bleach

ALLOWED_TAGS = ['p', 'br', 'strong', 'em', 'u', 'a', 'ul', 'ol', 'li']
ALLOWED_ATTRIBUTES = {'a': ['href', 'title']}

def sanitize_html(content):
    return bleach.clean(
        content,
        tags=ALLOWED_TAGS,
        attributes=ALLOWED_ATTRIBUTES,
        strip=True
    )

# In view
def create_article(request):
    if request.method == 'POST':
        form = ArticleForm(request.POST)
        if form.is_valid():
            article = form.save(commit=False)
            article.content = sanitize_html(article.content)
            article.save()
            return redirect('article_detail', pk=article.pk)
    else:
        form = ArticleForm()
    
    return render(request, 'article_form.html', {'form': form})
```

**7. File Upload Security:**

```python
import os
from django.core.validators import FileExtensionValidator
from django.core.exceptions import ValidationError

def validate_file_size(file):
    """Limit file size to 5MB"""
    max_size = 5 * 1024 * 1024
    if file.size > max_size:
        raise ValidationError(f"File size cannot exceed {max_size/1024/1024}MB")

class Document(models.Model):
    file = models.FileField(
        upload_to='documents/%Y/%m/%d/',
        validators=[
            FileExtensionValidator(
                allowed_extensions=['pdf', 'doc', 'docx', 'txt']
            ),
            validate_file_size
        ]
    )
    
    def save(self, *args, **kwargs):
        # Sanitize filename
        if self.file:
            name = os.path.basename(self.file.name)
            name = name.replace(' ', '_')
            # Remove special characters
            import re
            name = re.sub(r'[^a-zA-Z0-9._-]', '', name)
            self.file.name = name
        
        super().save(*args, **kwargs)

# In view
from django.core.files.storage import FileSystemStorage

def upload_file(request):
    if request.method == 'POST' and request.FILES.get('file'):
        uploaded_file = request.FILES['file']
        
        # Validate file type (check actual content, not just extension)
        import magic
        file_type = magic.from_buffer(uploaded_file.read(1024), mime=True)
        uploaded_file.seek(0)  # Reset file pointer
        
        allowed_types = ['application/pdf', 'text/plain']
        if file_type not in allowed_types:
            return JsonResponse({'error': 'Invalid file type'}, status=400)
        
        # Scan for malware (if using ClamAV)
        # import pyclamd
        # cd = pyclamd.ClamdUnixSocket()
        # result = cd.scan_stream(uploaded_file.read())
        
        fs = FileSystemStorage()
        filename = fs.save(uploaded_file.name, uploaded_file)
        
        return JsonResponse({'filename': filename})
```

**8. API Security (DRF):**

```python
# settings.py
REST_FRAMEWORK = {
    # Authentication
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    
    # Permissions
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    
    # Throttling
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
    },
    
    # Pagination
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}

# JWT Authentication
from rest_framework_simplejwt.tokens import RefreshToken

def get_tokens_for_user(user):
    refresh = RefreshToken.for_user(user)
    return {
        'refresh': str(refresh),
        'access': str(refresh.access_token),
    }

# Custom permission
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """Only owner can edit"""
    
    def has_object_permission(self, request, view, obj):
        # Read permissions allowed to any request
        if request.method in permissions.SAFE_METHODS:
            return True
        
        # Write permissions only to owner
        return obj.author == request.user

# Rate limiting per user
from rest_framework.throttling import UserRateThrottle

class BurstRateThrottle(UserRateThrottle):
    rate = '60/min'

class SustainedRateThrottle(UserRateThrottle):
    rate = '1000/day'

# In viewset
class ArticleViewSet(viewsets.ModelViewSet):
    permission_classes = [IsOwnerOrReadOnly]
    throttle_classes = [BurstRateThrottle, SustainedRateThrottle]
```

**9. Environment Variables & Secret Management:**

```python
# Never commit secrets!
# .env file
SECRET_KEY=your-secret-key-here
DATABASE_URL=postgresql://user:pass@localhost/dbname
AWS_ACCESS_KEY_ID=your-aws-key
AWS_SECRET_ACCESS_KEY=your-aws-secret

# settings.py
import os
from pathlib import Path
import environ

# Initialize environ
env = environ.Env(
    DEBUG=(bool, False)
)

# Read .env file
environ.Env.read_env(os.path.join(Path(__file__).resolve().parent.parent, '.env'))

# Get values
SECRET_KEY = env('SECRET_KEY')
DEBUG = env('DEBUG')
DATABASE_URL = env('DATABASE_URL')

DATABASES = {
    'default': env.db()
}

# .gitignore
.env
*.pyc
__pycache__/
db.sqlite3
```

**10. Logging & Monitoring:**

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'WARNING',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/django/security.log',
            'maxBytes': 1024*1024*15,  # 15MB
            'backupCount': 10,
            'formatter': 'verbose',
        },
        'mail_admins': {
            'level': 'ERROR',
            'class': 'django.utils.log.AdminEmailHandler',
        },
    },
    'loggers': {
        'django.security': {
            'handlers': ['file', 'mail_admins'],
            'level': 'WARNING',
            'propagate': False,
        },
    },
}

# Log security events
import logging

logger = logging.getLogger('django.security')

def login_view(request):
    username = request.POST.get('username')
    password = request.POST.get('password')
    
    user = authenticate(request, username=username, password=password)
    
    if user is not None:
        login(request, user)
        logger.info(f'Successful login: {username} from {request.META.get("REMOTE_ADDR")}')
    else:
        logger.warning(f'Failed login attempt: {username} from {request.META.get("REMOTE_ADDR")}')
```

**Security Checklist:**
- ✅ Keep Django and dependencies updated
- ✅ Use HTTPS in production
- ✅ Enable CSRF protection
- ✅ Validate and sanitize all user input
- ✅ Use parameterized queries (ORM)
- ✅ Implement proper authentication
- ✅ Use strong password policies
- ✅ Set secure cookie flags
- ✅ Enable security headers
- ✅ Implement rate limiting
- ✅ Use environment variables for secrets
- ✅ Log security events
- ✅ Keep SECRET_KEY secret
- ✅ Disable DEBUG in production
- ✅ Use allowed_hosts properly

---

## 5. DJANGO TESTING

### Q8: Explain Django testing strategies and best practices.
**Answer:**

**1. Unit Tests:**

```python
from django.test import TestCase
from django.contrib.auth.models import User
from .models import Article

class ArticleModelTest(TestCase):
    """Test Article model"""
    
    @classmethod
    def setUpTestData(cls):
        """Set up data for the whole TestCase"""
        cls.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        cls.article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=cls.user
        )
    
    def test_article_creation(self):
        """Test article is created correctly"""
        self.assertEqual(self.article.title, 'Test Article')
        self.assertEqual(self.article.author, self.user)
        self.assertFalse(self.article.is_published)
    
    def test_article_str(self):
        """Test string representation"""
        self.assertEqual(str(self.article), 'Test Article')
    
    def test_get_absolute_url(self):
        """Test get_absolute_url method"""
        expected_url = f'/articles/{self.article.id}/'
        self.assertEqual(self.article.get_absolute_url(), expected_url)
    
    def test_article_slug_generation(self):
        """Test slug is auto-generated"""
        article = Article.objects.create(
            title='New Article',
            content='Content',
            author=self.user
        )
        self.assertEqual(article.slug, 'new-article')
    
    def test_published_articles_manager(self):
        """Test custom manager"""
        Article.objects.create(
            title='Published',
            content='Content',
            author=self.user,
            is_published=True
        )
        
        published_count = Article.published.count()
        self.assertEqual(published_count, 1)
```

**2. View Tests:**

```python
from django.test import TestCase, Client
from django.urls import reverse

class ArticleViewTest(TestCase):
    """Test Article views"""
    
    def setUp(self):
        """Set up test client and data"""
        self.client = Client()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.user,
            is_published=True
        )
    
    def test_article_list_view(self):
        """Test article list view"""
        response = self.client.get(reverse('article_list'))
        
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, 'Test Article')
        self.assertTemplateUsed(response, 'articles/list.html')
    
    def test_article_detail_view(self):
        """Test article detail view"""
        url = reverse('article_detail', kwargs={'pk': self.article.pk})
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, self.article.title)
        self.assertEqual(response.context['article'], self.article)
    
    def test_article_create_view_authenticated(self):
        """Test creating article when logged in"""
        self.client.login(username='testuser', password='testpass123')
        
        data = {
            'title': 'New Article',
            'content': 'New content'
        }
        response = self.client.post(reverse('article_create'), data)
        
        self.assertEqual(response.status_code, 302)  # Redirect after success
        self.assertTrue(Article.objects.filter(title='New Article').exists())
    
    def test_article_create_view_unauthenticated(self):
        """Test creating article requires authentication"""
        data = {
            'title': 'New Article',
            'content': 'New content'
        }
        response = self.client.post(reverse('article_create'), data)
        
        self.assertEqual(response.status_code, 302)  # Redirect to login
        self.assertFalse(Article.objects.filter(title='New Article').exists())
    
    def test_article_update_view_owner(self):
        """Test owner can update article"""
        self.client.login(username='testuser', password='testpass123')
        
        url = reverse('article_update', kwargs={'pk': self.article.pk})
        data = {
            'title': 'Updated Title',
            'content': 'Updated content'
        }
        response = self.client.post(url, data)
        
        self.article.refresh_from_db()
        self.assertEqual(self.article.title, 'Updated Title')
    
    def test_article_delete_view(self):
        """Test deleting article"""
        self.client.login(username='testuser', password='testpass123')
        
        url = reverse('article_delete', kwargs={'pk': self.article.pk})
        response = self.client.post(url)
        
        self.assertFalse(Article.objects.filter(pk=self.article.pk).exists())
```

**3. Form Tests:**

```python
class ArticleFormTest(TestCase):
    """Test Article form"""
    
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
    
    def test_form_valid_data(self):
        """Test form with valid data"""
        form = ArticleForm(data={
            'title': 'Test Article',
            'content': 'Test content',
            'author': self.user.id
        })
        
        self.assertTrue(form.is_valid())
    
    def test_form_missing_title(self):
        """Test form validation with missing title"""
        form = ArticleForm(data={
            'content': 'Test content',
            'author': self.user.id
        })
        
        self.assertFalse(form.is_valid())
        self.assertIn('title', form.errors)
    
    def test_form_title_too_short(self):
        """Test custom validation"""
        form = ArticleForm(data={
            'title': 'abc',  # Too short
            'content': 'Test content',
            'author': self.user.id
        })
        
        self.assertFalse(form.is_valid())
        self.assertIn('title', form.errors)
    
    def test_form_save(self):
        """Test form save creates article"""
        form = ArticleForm(data={
            'title': 'Test Article',
            'content': 'Test content',
            'author': self.user.id
        })
        
        self.assertTrue(form.is_valid())
        article = form.save()
        
        self.assertEqual(article.title, 'Test Article')
        self.assertEqual(article.author, self.user)
```

**4. API Tests (DRF):**

```python
from rest_framework.test import APITestCase, APIClient
from rest_framework import status

class ArticleAPITest(APITestCase):
    """Test Article API"""
    
    def setUp(self):
        """Set up test client and data"""
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.user
        )
    
    def test_get_article_list(self):
        """Test GET /api/articles/"""
        response = self.client.get('/api/articles/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 1)
    
    def test_get_article_detail(self):
        """Test GET /api/articles/{id}/"""
        url = f'/api/articles/{self.article.id}/'
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], 'Test Article')
    
    def test_create_article_authenticated(self):
        """Test POST /api/articles/ when authenticated"""
        self.client.force_authenticate(user=self.user)
        
        data = {
            'title': 'New Article',
            'content': 'New content'
        }
        response = self.client.post('/api/articles/', data)
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Article.objects.count(), 2)
    
    def test_create_article_unauthenticated(self):
        """Test POST /api/articles/ requires authentication"""
        data = {
            'title': 'New Article',
            'content': 'New content'
        }
        response = self.client.post('/api/articles/', data)
        
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
    
    def test_update_article_owner(self):
        """Test PUT /api/articles/{id}/ by owner"""
        self.client.force_authenticate(user=self.user)
        
        url = f'/api/articles/{self.article.id}/'
        data = {
            'title': 'Updated Title',
            'content': 'Updated content'
        }
        response = self.client.put(url, data)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.article.refresh_from_db()
        self.assertEqual(self.article.title, 'Updated Title')
    
    def test_update_article_non_owner(self):
        """Test non-owner cannot update"""
        other_user = User.objects.create_user(
            username='otheruser',
            password='testpass123'
        )
        self.client.force_authenticate(user=other_user)
        
        url = f'/api/articles/{self.article.id}/'
        data = {
            'title': 'Updated Title',
            'content': 'Updated content'
        }
        response = self.client.put(url, data)
        
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
    
    def test_delete_article(self):
        """Test DELETE /api/articles/{id}/"""
        self.client.force_authenticate(user=self.user)
        
        url = f'/api/articles/{self.article.id}/'
        response = self.client.delete(url)
        
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertEqual(Article.objects.count(), 0)
```

**5. Factory Pattern (using factory_boy):**

```python
import factory
from factory.django import DjangoModelFactory

class UserFactory(DjangoModelFactory):
    class Meta:
        model = User
    
    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.LazyAttribute(lambda obj: f'{obj.username}@example.com')
    first_name = factory.Faker('first_name')
    last_name = factory.Faker('last_name')

class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article
    
    title = factory.Faker('sentence', nb_words=4)
    content = factory.Faker('paragraph', nb_sentences=5)
    author = factory.SubFactory(UserFactory)
    is_published = True

# Usage in tests
class ArticleTestCase(TestCase):
    def test_with_factory(self):
        # Create single article
        article = ArticleFactory()
        
        # Create multiple articles
        articles = ArticleFactory.create_batch(10)
        
        # Create with custom attributes
        article = ArticleFactory(title='Custom Title', is_published=False)
        
        # Create with related objects
        user = UserFactory(username='john')
        article = ArticleFactory(author=user)
```

**6. Mocking:**

```python
from unittest.mock import patch, Mock

class EmailTestCase(TestCase):
    """Test email sending"""
    
    @patch('myapp.tasks.send_email')
    def test_send_welcome_email(self, mock_send_email):
        """Test welcome email is sent"""
        user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        
        # Trigger action that sends email
        send_welcome_email(user.id)
        
        # Assert email was sent
        mock_send_email.assert_called_once_with(
            to='test@example.com',
            subject='Welcome!',
            template='emails/welcome.html'
        )
    
    @patch('requests.get')
    def test_fetch_external_data(self, mock_get):
        """Test fetching from external API"""
        # Mock response
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {'data': 'test'}
        mock_get.return_value = mock_response
        
        # Call function
        result = fetch_external_data('https://api.example.com/data')
        
        # Assert
        self.assertEqual(result, {'data': 'test'})
        mock_get.assert_called_once_with('https://api.example.com/data')
```

**7. Database Testing:**

```python
from django.test import TransactionTestCase

class ArticleTransactionTest(TransactionTestCase):
    """Tests that need database transactions"""
    
    def test_concurrent_updates(self):
        """Test handling concurrent updates"""
        from django.db import transaction
        
        article = Article.objects.create(
            title='Test',
            content='Content',
            author=self.user
        )
        
        with transaction.atomic():
            article.view_count += 1
            article.save()
        
        article.refresh_from_db()
        self.assertEqual(article.view_count, 1)
```

**8. Coverage:**

```python
# Install coverage
# pip install coverage

# Run tests with coverage
# coverage run --source='.' manage.py test
# coverage report
# coverage html

# .coveragerc
[run]
omit =
    */migrations/*
    */tests/*
    */venv/*
    manage.py

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
    if __name__ == .__main__.:
```

**Testing Best Practices:**
- ✅ Test models, views, forms, and APIs
- ✅ Use factories for test data
- ✅ Mock external dependencies
- ✅ Test edge cases and error conditions
- ✅ Use descriptive test names
- ✅ Aim for high coverage (>80%)
- ✅ Keep tests fast and isolated
- ✅ Use `setUpTestData` for class-level data
- ✅ Test permissions and authentication
- ✅ Use `TransactionTestCase` when needed

---

## 6. DJANGO DEPLOYMENT & SCALABILITY

### Q9: How do you deploy and scale Django applications?
**Answer:**

**1. Production Settings:**

```python
# settings/base.py
from pathlib import Path
import environ

env = environ.Env()

BASE_DIR = Path(__file__).resolve().parent.parent.parent

INSTALLED_APPS = [
    # Django apps
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # Third-party apps
    'rest_framework',
    'corsheaders',
    'django_filters',
    
    # Local apps
    'myapp',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # Static files
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

# settings/production.py
from .base import *

DEBUG = False

ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')

# Database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST'),
        'PORT': env('DB_PORT', default='5432'),
        'CONN_MAX_AGE': 600,
        'OPTIONS': {
            'connect_timeout': 10,
        }
    }
}

# Cache
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': env('REDIS_URL'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'CONNECTION_POOL_KWARGS': {'max_connections': 50}
        }
    }
}

# Static files
STATIC_ROOT = BASE_DIR / 'staticfiles'
STATIC_URL = '/static/'
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

# Media files
MEDIA_ROOT = BASE_DIR / 'media'
MEDIA_URL = '/media/'

# Security
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Email
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = env('EMAIL_HOST')
EMAIL_PORT = env.int('EMAIL_PORT', default=587)
EMAIL_USE_TLS = True
EMAIL_HOST_USER = env('EMAIL_HOST_USER')
EMAIL_HOST_PASSWORD = env('EMAIL_HOST_PASSWORD')

# Celery
CELERY_BROKER_URL = env('CELERY_BROKER_URL')
CELERY_RESULT_BACKEND = env('CELERY_RESULT_BACKEND')
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'

# Logging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'ERROR',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/django/error.log',
            'maxBytes': 1024*1024*15,  # 15MB
            'backupCount': 10,
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['file'],
            'level': 'ERROR',
            'propagate': True,
        },
    },
}
```

**2. Gunicorn Configuration:**

```python
# gunicorn_config.py
import multiprocessing

# Server socket
bind = '0.0.0.0:8000'
backlog = 2048

# Worker processes
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = 'sync'
worker_connections = 1000
timeout = 30
keepalive = 2

# Logging
accesslog = '/var/log/gunicorn/access.log'
errorlog = '/var/log/gunicorn/error.log'
loglevel = 'info'

# Process naming
proc_name = 'myapp'

# Server mechanics
daemon = False
pidfile = '/var/run/gunicorn/pid'
user = 'www-data'
group = 'www-data'
tmp_upload_dir = None

# SSL (if terminating SSL at Gunicorn)
# keyfile = '/path/to/key.pem'
# certfile### Q5: Explain DRF ViewSets, Generic Views, and their differences.
**Answer:**

**1. Function-Based Views (FBV):**
```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated

@api_view(['GET', 'POST'])
@permission_classes([IsAuthenticated])
def article_list(request):
    """List articles or create new article"""
    if request.method == 'GET':
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

@api_view(['GET', 'PUT', 'DELETE'])
def article_detail(request, pk):
    """Retrieve, update or delete article"""
    try:
        article = Article.objects.get(pk=pk)
    except Article.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)
    
    if request.method == 'GET':
        serializer = ArticleSerializer(article)
        return Response(serializer.data)
    
    elif request.method == 'PUT':
        serializer = ArticleSerializer(article, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    elif request.method == 'DELETE':
        article.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**2. Class-Based Views (CBV):**
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class ArticleList(APIView):
    """List all articles or create new article"""
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True, context={'request': request})
        return Response(serializer.data)
    
    def post(self, request):
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class ArticleDetail(APIView):
    """Retrieve, update or delete article"""
    
    def get_object(self, pk):
        try:
            return Article.objects.get(pk=pk)
        except Article.DoesNotExist:
            raise Http404
    
    def get(self, request, pk):
        article = self.get_object(pk)
        serializer = ArticleSerializer(article)
        return Response(serializer.data)
    
    def put(self, request, pk):
        article = self.get_object(pk)
        serializer = ArticleSerializer(article, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def delete(self, request, pk):
        article = self.get_object(pk)
        article.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**3. Generic Views:**
```python
from rest_framework import generics
from rest_framework.permissions import IsAuthenticatedOrReadOnly

# List and Create
class ArticleList(generics.ListCreateAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def perform_create(self, serializer):
        """Customize creation"""
        serializer.save(author=self.request.user)

# Retrieve, Update, Delete
class ArticleDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]

# Other generic views:
# - ListAPIView: Read-only list
# - CreateAPIView: Create only
# - RetrieveAPIView: Read-only single object
# - UpdateAPIView: Update only
# - DestroyAPIView: Delete only
# - RetrieveUpdateAPIView: Read and update
```

**4. ViewSets:**
```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

class ArticleViewSet(viewsets.ModelViewSet):
    """Complete CRUD operations"""
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filterset_fields = ['author', 'is_published']
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'title']
    
    def get_queryset(self):
        """Customize queryset"""
        queryset = super().get_queryset()
        
        # Optimize queries
        queryset = queryset.select_related('author')
        queryset = queryset.prefetch_related('comments')
        
        # Filter by user for non-staff
        if not self.request.user.is_staff:
            queryset = queryset.filter(
                models.Q(is_published=True) | models.Q(author=self.request.user)
            )
        
        return queryset
    
    def get_serializer_class(self):
        """Use different serializers for different actions"""
        if self.action == 'list':
            return ArticleListSerializer
        elif self.action == 'retrieve':
            return ArticleDetailSerializer
        return ArticleSerializer
    
    def perform_create(self, serializer):
        """Set author on creation"""
        serializer.save(author=self.request.user)
    
    def perform_update(self, serializer):
        """Custom update logic"""
        serializer.save(updated_by=self.request.user)
    
    # Custom actions
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """Publish article"""
        article = self.get_object()
        article.is_published = True
        article.published_at = timezone.now()
        article.save()
        
        serializer = self.get_serializer(article)
        return Response(serializer.data)
    
    @action(detail=False, methods=['get'])
    def my_articles(self, request):
        """Get current user's articles"""
        articles = self.get_queryset().filter(author=request.user)
        serializer = self.get_serializer(articles, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['get'])
    def comments(self, request, pk=None):
        """Get article comments"""
        article = self.get_object()
        comments = article.comments.all()
        serializer = CommentSerializer(comments, many=True)
        return Response(serializer.data)

# URLs for ViewSet
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'articles', ArticleViewSet, basename='article')

urlpatterns = router.urls

# Generated URLs:
# GET    /articles/              -> list
# POST   /articles/              -> create
# GET    /articles/{pk}/         -> retrieve
# PUT    /articles/{pk}/         -> update
# PATCH  /articles/{pk}/         -> partial_update
# DELETE /articles/{pk}/         -> destroy
# POST   /articles/{pk}/publish/ -> publish (custom action)
# GET    /articles/my_articles/  -> my_articles (custom action)
```

**5. ReadOnlyModelViewSet:**
```python
class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    """Read-only viewset - only list and retrieve"""
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    
    # Only provides:
    # - list()
    # - retrieve()
```

**6. Custom ViewSet:**
```python
from rest_framework import viewsets, mixins

class CreateListRetrieveViewSet(
    mixins.CreateModelMixin,
    mixins.ListModelMixin,
    mixins.RetrieveModelMixin,
    viewsets.GenericViewSet
):
    """ViewSet that only allows create, list, and retrieve"""
    pass

class CommentViewSet(CreateListRetrieveViewSet):
    """Comments can only be created and viewed, not updated or deleted"""
    queryset = Comment.objects.all()
    serializer_class = CommentSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def perform_create(self, serializer):
        serializer.save(user=self.request.user)
```

**7. Advanced ViewSet Features:**
```python
from rest_framework import viewsets, filters
from django_filters.rest_framework import DjangoFilterBackend

class AdvancedArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    # Filtering
    filter_backends = [
        DjangoFilterBackend,
        filters.SearchFilter,
        filters.OrderingFilter
    ]
    filterset_fields = {
        'author': ['exact'],
        'is_published': ['exact'],
        'created_at': ['gte', 'lte'],
        'title': ['icontains']
    }
    search_fields = ['title', 'content', 'author__name']
    ordering_fields = ['created_at', 'title', 'updated_at']
    ordering = ['-created_at']
    
    # Pagination
    pagination_class = PageNumberPagination
    
    def get_permissions(self):
        """Different permissions for different actions"""
        if self.action in ['list', 'retrieve']:
            return [permissions.AllowAny()]
        elif self.action in ['create']:
            return [permissions.IsAuthenticated()]
        elif self.action in ['update', 'partial_update', 'destroy']:
            return [permissions.IsAuthenticated(), IsOwnerOrReadOnly()]
        return super().get_permissions()
    
    def get_throttles(self):
        """Different throttles for different actions"""
        if self.action == 'create':
            return [UserRateThrottle()]
        return super().get_throttles()
    
    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def like(self, request, pk=None):
        """Like an article"""
        article = self.get_object()
        user = request.user
        
        if article.likes.filter(id=user.id).exists():
            article.likes.remove(user)
            return Response({'status': 'unliked'})
        else:
            article.likes.add(user)
            return Response({'status': 'liked'})
    
    @action(detail=False, methods=['get'])
    def trending(self, request):
        """Get trending articles"""
        from django.db.models import Count
        
        articles = self.get_queryset().annotate(
            like_count=Count('likes')
        ).order_by('-like_count')[:10]
        
        serializer = self.get_serializer(articles, many=True)
        return Response(serializer.data)
```

**Comparison:**

| Feature | FBV | APIView | GenericView | ViewSet |
|---------|-----|---------|-------------|---------|
| Code Amount | Most | Moderate | Less | Least |
| Flexibility | Highest | High | Moderate | Low |
| Reusability | Low | Moderate | High | Highest |
| Best For | Custom logic | Custom APIs | Standard CRUD | REST APIs |
| URL Routing | Manual | Manual | Manual | Automatic |

**When to Use What:**

**Function-Based Views:**
- Simple, one-off endpoints
- Very custom logic
- Learning/prototyping

**APIView:**
- Need full control
- Custom HTTP methods
- Complex business logic

**Generic Views:**
- Standard CRUD operations
- Want some customization
- Don't need all methods

**ViewSets:**
- Full REST API for a model
- Standard CRUD with minimal customization
- Need automatic URL routing
- Want consistent API structure

---

## 3. DJANGO PERFORMANCE & OPTIMIZATION

### Q6: How do you optimize Django application performance?
**Answer:**

**1. Database Query Optimization:**

```python
# BAD: N+1 Query Problem
def get_articles_bad():
    articles = Article.objects.all()
    for article in articles:
        print(article.author.name)  # N queries
        print(article.category.name)  # N queries

# GOOD: Use select_related
def get_articles_good():
    articles = Article.objects.select_related(
        'author', 'category'
    ).all()  # 1 query with JOINs
    
    for article in articles:
        print(article.author.name)  # No extra query
        print(article.category.name)  # No extra query

# GOOD: Use prefetch_related for reverse relations
def get_authors_with_articles():
    authors = Author.objects.prefetch_related('articles').all()
    
    for author in authors:
        for article in author.articles.all():  # No extra queries
            print(article.title)

# Advanced prefetch
from django.db.models import Prefetch

def get_authors_with_published_articles():
    published_articles = Article.objects.filter(is_published=True)
    
    authors = Author.objects.prefetch_related(
        Prefetch('articles', queryset=published_articles, to_attr='published_articles')
    ).all()
    
    for author in authors:
        for article in author.published_articles:  # Use prefetched data
            print(article.title)
```

**2. Caching Strategies:**

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'KEY_PREFIX': 'myapp',
        'TIMEOUT': 300,
    }
}

# Low-level cache API
from django.core.cache import cache

def get_article(article_id):
    cache_key = f'article_{article_id}'
    article = cache.get(cache_key)
    
    if article is None:
        article = Article.objects.select_related('author').get(id=article_id)
        cache.set(cache_key, article, timeout=3600)  # 1 hour
    
    return article

# Invalidate cache on update
from django.db.models.signals import post_save, post_delete

@receiver([post_save, post_delete], sender=Article)
def invalidate_article_cache(sender, instance, **kwargs):
    cache_key = f'article_{instance.id}'
    cache.delete(cache_key)

# Template fragment caching
{% load cache %}
{% cache 3600 sidebar %}
    <!-- Expensive sidebar rendering -->
    {% for category in categories %}
        <li>{{ category.name }}</li>
    {% endfor %}
{% endcache %}

# Per-site cache
MIDDLEWARE = [
    'django.middleware.cache.UpdateCacheMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.cache.FetchFromCacheMiddleware',
]

CACHE_MIDDLEWARE_ALIAS = 'default'
CACHE_MIDDLEWARE_SECONDS = 600
CACHE_MIDDLEWARE_KEY_PREFIX = 'myapp'

# Per-view cache
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # Cache for 15 minutes
def article_list(request):
    articles = Article.objects.all()
    return render(request, 'articles/list.html', {'articles': articles})

# Cache with conditions
from django.views.decorators.vary import vary_on_cookie

@cache_page(60 * 15)
@vary_on_cookie
def my_view(request):
    # Cache varies based on cookie
    pass
```

**3. Database Indexing:**

```python
class Article(models.Model):
    title = models.CharField(max_length=200, db_index=True)  # Simple index
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    published_date = models.DateTimeField()
    is_published = models.BooleanField(default=False)
    slug = models.SlugField(unique=True)  # Automatically indexed
    
    class Meta:
        indexes = [
            # Composite index for common queries
            models.Index(fields=['author', '-published_date']),
            
            # Covering index
            models.Index(fields=['is_published', 'published_date', 'title']),
            
            # Partial index (PostgreSQL)
            models.Index(
                fields=['title'],
                name='published_title_idx',
                condition=models.Q(is_published=True)
            ),
            
            # Expression index (PostgreSQL)
            models.Index(
                Lower('title').desc(),
                name='title_lower_idx'
            ),
        ]

# Find missing indexes
python manage.py sqlmigrate myapp 0001 | grep "CREATE INDEX"

# Analyze queries
from django.db import connection

def show_queries():
    for query in connection.queries:
        print(query['sql'])
```

**4. Query Optimization:**

```python
# Use only() to fetch specific fields
articles = Article.objects.only('id', 'title', 'author_id')

# Use defer() to exclude heavy fields
articles = Article.objects.defer('content')

# Use values() or values_list() for simple data
titles = Article.objects.values_list('title', flat=True)

# Use exists() instead of count()
if Article.objects.filter(author=user).exists():  # Fast
    pass

# Use iterator() for large datasets
for article in Article.objects.iterator(chunk_size=1000):
    process_article(article)  # Memory efficient

# Use bulk operations
articles = [Article(title=f'Article {i}') for i in range(1000)]
Article.objects.bulk_create(articles, batch_size=100)

# Use update() instead of save() for updates
Article.objects.filter(author=user).update(is_published=True)

# Use annotate for computed fields
from django.db.models import Count, Avg

authors = Author.objects.annotate(
    article_count=Count('articles'),
    avg_rating=Avg('articles__rating')
)

# Use aggregation
from django.db.models import Sum

total_views = Article.objects.aggregate(total=Sum('view_count'))
```

**5. Lazy Loading & Eager Loading:**

```python
# Lazy loading (queries execute when needed)
articles = Article.objects.filter(is_published=True)  # No query yet
for article in articles:  # Query executes here
    print(article.title)

# Force evaluation
articles = list(Article.objects.all())  # Execute now

# Eager loading with select_related
article = Article.objects.select_related('author', 'category').get(id=1)

# Eager loading with prefetch_related
authors = Author.objects.prefetch_related(
    'articles',
    'articles__comments'
).all()
```

**6. Asynchronous Processing:**

```python
# Use Celery for background tasks
from celery import shared_task

@shared_task
def send_newsletter(article_id):
    article = Article.objects.get(id=article_id)
    users = User.objects.filter(subscribed=True)
    
    for user in users:
        send_email(user.email, article)

# In view
def publish_article(request, pk):
    article = Article.objects.get(pk=pk)
    article.is_published = True
    article.save()
    
    # Send newsletter asynchronously
    send_newsletter.delay(article.id)
    
    return JsonResponse({'status': 'published'})

# Periodic tasks
from celery.schedules import crontab

@app.task
def cleanup_old_data():
    threshold = timezone.now() - timedelta(days=30)
    Article.objects.filter(created_at__lt=threshold, is_published=False).delete()

# Celery beat schedule
app.conf.beat_schedule = {
    'cleanup-every-day': {
        'task': 'myapp.tasks.cleanup_old_data',
        'schedule': crontab(hour=2, minute=0),
    },
}
```

**7. Connection Pooling:**

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'mypassword',
        'HOST': 'localhost',
        'PORT': '5432',
        'CONN_MAX_AGE': 600,  # Connection pooling (10 minutes)
        'OPTIONS': {
            'connect_timeout': 10,
            'options': '-c statement_timeout=30000'  # 30 seconds
        }
    }
}
```

**8. Middleware Optimization:**

```python
# Custom middleware for performance
class PerformanceMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        import time
        from django.db import connection
        
        # Start timing
        start_time = time.time()
        start_queries = len(connection.queries)
        
        # Process request
        response = self.get_response(request)
        
        # Calculate stats
        end_time = time.time()
        total_time = end_time - start_time
        num_queries = len(connection.queries) - start_queries
        
        # Add headers
        response['X-Response-Time'] = f'{total_time:.3f}s'
        response['X-Query-Count'] = str(num_queries)
        
        # Log slow requests
        if total_time > 1.0:  # Over 1 second
            logger.warning(
                f'Slow request: {request.path} took {total_time:.3f}s '
                f'with {num_queries} queries'
            )
        
        return response
```

**9. Static Files & CDN:**

```python
# settings.py
STATIC_URL = 'https://cdn.example.com/static/'
MEDIA_URL = 'https://cdn.example.com/media/'

# Use WhiteNoise for static files
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # Add this
    # ... other middleware
]

STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

**10. Monitoring & Profiling:**

```python
# Django Debug Toolbar
INSTALLED_APPS = [
    # ...
    'debug_toolbar',
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    # ...
]

INTERNAL_IPS = ['127.0.0.1']

# Django Silk for profiling
INSTALLED_APPS = [
    # ...
    'silk',
]

MIDDLEWARE = [
    'silk.middleware.SilkyMiddleware',
    # ...
]

# Custom profiling
import cProfile
import pstats

def profile_view(func):
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()
        
        result = func(*args, **kwargs)
        
        profiler.disable()
        stats = pstats.Stats(profiler)
        stats.sort_stats('cumulative')
        stats.print_stats(20)
        
        return result
    return wrapper

@profile_view
def expensive_view(request):
    # Your view logic
    pass
```

**Performance Checklist:**
- ✅ Use `select_related()` and `prefetch_related()`
- ✅ Add database indexes
- ✅ Implement caching (Redis)
- ✅ Use `only()` and `defer()` appropriately
- ✅ Bulk operations for mass updates
- ✅ Async tasks for heavy operations (Celery)
- ✅ Connection pooling
- ✅ CDN for static files
- ✅ Database query optimization
- ✅ Monitor with Django Debug Toolbar
- ✅ Use `iterator()` for large datasets
- ✅ Pagination for list views

---

## 4. DJANGO SECURITY

### Q7: Explain Django security best practices and common vulnerabilities.
**Answer:**

**1. SQL Injection Prevention:**

```python
# BAD - Vulnerable to SQL injection
def search_articles_bad(request):
    query = request.GET.get('q')
    articles = Article.objects.raw(
        f"SELECT * FROM articles WHERE title LIKE '%{query}%'"
    )
    return render(request, 'articles.html', {'articles': articles})

# GOOD - Use parameterized queries
def search_articles_good(request):
    query = request.GET.get('q')
    articles = Article.objects.raw(
        "SELECT * FROM articles WHERE title LIKE %s",
        [f'%{query}%']
    )
    return render(request, 'articles.html', {'articles': articles})

# BEST - Use ORM
def search_articles_best(request):
    query = request.GET.get('q')
    articles = Article.objects.filter(title__icontains=query)
    return render(request, 'articles.html', {'articles': articles})
```

**2. Cross-Site Scripting (XSS) Prevention:**

```python
# Django templates auto-escape by default
{% autoescape on %}
    {{ user_input }}  # Automatically escaped
{% endautoescape %}

# Explicitly mark as safe (use cautiously)
from django.utils.safestring import mark_safe

def render_html(content):
    # Sanitize first!
    import bleach
    clean_content = bleach.clean(
        content,
        tags=['p', 'b', 'i', 'u', 'a'],
        attributes={'a': ['href', 'title']},
        strip=True
    )
    return mark_safe(clean_content)

# In template
{{ content|safe }}  # Only if you're sure it's safe!

# JSON responses
from django.http import JsonResponse

def api_view(request):
    data = {'user_input': request.GET.get('input')}
    return JsonResponse(data)  # Automatically escapes
```

**3. Cross-Site Request Forgery (CSRF) Protection:**

```python
# Django CSRF protection is enabled by default
MIDDLEWARE = [
    'django.middleware.csrf.CsrfViewMiddleware',  # Required
    # ...
]

# In forms
<form method="post">
    {% csrf_token %}
    <!-- form fields -->
</form>

# AJAX requests
// Get CSRF token
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

const csrftoken = getCookie('csrftoken');

// Include in AJAX
fetch('/api/endpoint/', {
    method: 'POST',
    headers: {
        'X-CSRFToken': csrftoken,
        'Content-Type': 'application/json',
    },
    body: JSON.stringify(data)
});

# Exempt specific views (use cautiously)
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def webhook_view(request):
    # For third-party webhooks
    pass

# DRF CSRF with Session Authentication
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
}
```

**4. Authentication & Authorization:**

```python
# Strong password validation
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {'min_length': 12}
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]

# Custom password validator
from django.core.exceptions import ValidationError

class SpecialCharacterValidator:
    def validate(self, password, user=None):
        if not any(char in '!@#$%^&*()' for char in password):
            raise ValidationError(
                "Password must contain at least one special character",
                code='password_no_special',
            )
    
    def get_help_text(self):
        return "Your password must contain at least one special character (!@#$%^&*())"

# Secure authentication views
from django.contrib.auth import authenticate, login
from django.contrib.auth.decorators import login_required
from django.contrib.auth.mixins import LoginRequiredMixin

@login_required
def protected_view(request):
    return render(request, 'protected.html')

class ProtectedView(LoginRequiredMixin, View):
    login_url = '/login/'
    redirect_field_name = 'next'
    
    def get(self, request):
        return render(request, 'protected.html')

# Permission-based access
from django.contrib.auth.decorators import permission_required

@permission_required('myapp.can_publish', raise_exception=True)
def publish_article(request, pk):
    article = Article.objects.get(pk=pk)
    article.is_published = True
    article.save()
    return redirect('article_detail', pk=pk)

# Custom permissions
class Article(models.Model):
    # ... fields ...
    
    class Meta:
        permissions = [
            ("can_publish", "Can publish articles"),
            ("can_feature", "Can feature articles"),
        ]
```

**5. Secure Session Management:**

```python
# settings.py

# Session security
SESSION_COOKIE_SECURE = True  # HTTPS only
SESSION_COOKIE_HTTPONLY = True  # Not accessible via JavaScript
SESSION_COOKIE_SAMESITE = 'Strict'  # CSRF protection
SESSION_COOKIE_AGE = 3600  # 1 hour

# CSRF security
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_HTTPONLY = True
CSRF_COOKIE_SAMESITE = 'Strict'

# Security headers
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X# Django Senior Backend Developer Interview Preparation Guide
## Comprehensive Guide for Senior Django Developer Interviews

---

## 1. DJANGO CORE CONCEPTS

### Q1: Explain Django's MTV architecture and how it differs from MVC.
**Answer:**

**Django MTV (Model-Template-View):**
- **Model:** Data layer - defines database structure
- **Template:** Presentation layer - HTML with template tags
- **View:** Business logic - processes requests and returns responses

**Traditional MVC (Model-View-Controller):**
- **Model:** Data layer
- **View:** Presentation layer
- **Controller:** Business logic

**Key Difference:**
Django's "View" is equivalent to MVC's "Controller", and Django's "Template" is equivalent to MVC's "View". Django calls it MTV but it's essentially MVC with different naming.

```python
# models.py (Model)
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    published = models.BooleanField(default=False)
    
    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['published', '-created_at']),
        ]
    
    def __str__(self):
        return self.title

# views.py (View - Controller logic)
from django.shortcuts import render, get_object_or_404
from django.http import JsonResponse
from .models import Article

def article_list(request):
    """List all published articles"""
    articles = Article.objects.filter(published=True).select_related('author')
    return render(request, 'articles/list.html', {'articles': articles})

def article_detail(request, pk):
    """Display single article"""
    article = get_object_or_404(Article, pk=pk, published=True)
    return render(request, 'articles/detail.html', {'article': article})

# templates/articles/list.html (Template - Presentation)
{% extends 'base.html' %}

{% block content %}
<h1>Articles</h1>
{% for article in articles %}
    <div class="article">
        <h2>{{ article.title }}</h2>
        <p>By {{ article.author.username }} on {{ article.created_at|date:"F d, Y" }}</p>
        <a href="{% url 'article_detail' article.pk %}">Read more</a>
    </div>
{% endfor %}
{% endblock %}

# urls.py (URL Configuration)
from django.urls import path
from . import views

urlpatterns = [
    path('articles/', views.article_list, name='article_list'),
    path('articles/<int:pk>/', views.article_detail, name='article_detail'),
]
```

---

### Q2: Explain Django's ORM and query optimization techniques.
**Answer:**

**Django ORM Basics:**

```python
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    bio = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name='books')
    published_date = models.DateField()
    isbn = models.CharField(max_length=13, unique=True)
    price = models.DecimalField(max_digits=6, decimal_places=2)
    
    class Meta:
        ordering = ['-published_date']
        indexes = [
            models.Index(fields=['author', '-published_date']),
        ]

class Review(models.Model):
    book = models.ForeignKey(Book, on_delete=models.CASCADE, related_name='reviews')
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    rating = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
    comment = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ['book', 'user']
```

**Query Optimization Techniques:**

**1. N+1 Query Problem:**
```python
# BAD - N+1 queries
books = Book.objects.all()  # 1 query
for book in books:
    print(book.author.name)  # N queries (one per book)

# GOOD - select_related (for ForeignKey, OneToOne)
books = Book.objects.select_related('author').all()  # 1 query with JOIN
for book in books:
    print(book.author.name)  # No additional queries

# Generated SQL:
# SELECT book.*, author.* FROM book 
# INNER JOIN author ON book.author_id = author.id

# GOOD - prefetch_related (for ManyToMany, reverse ForeignKey)
authors = Author.objects.prefetch_related('books').all()
for author in authors:
    for book in author.books.all():  # No additional queries
        print(book.title)

# Uses 2 queries:
# 1. SELECT * FROM author
# 2. SELECT * FROM book WHERE author_id IN (1, 2, 3, ...)
```

**2. Only/Defer:**
```python
# Only fetch specific fields
books = Book.objects.only('title', 'price')  # Fetch only title and price

# Defer heavy fields
books = Book.objects.defer('description')  # Fetch all except description

# With relations
books = Book.objects.select_related('author').only(
    'title', 'price', 'author__name'
)
```

**3. Aggregation:**
```python
from django.db.models import Count, Avg, Sum, Max, Min, F, Q

# Count books per author
authors = Author.objects.annotate(
    book_count=Count('books'),
    avg_price=Avg('books__price')
).filter(book_count__gt=0)

for author in authors:
    print(f"{author.name}: {author.book_count} books, avg price: {author.avg_price}")

# Complex aggregation
stats = Book.objects.aggregate(
    total_books=Count('id'),
    avg_price=Avg('price'),
    max_price=Max('price'),
    min_price=Min('price'),
    total_revenue=Sum('price')
)

# Annotate with conditions
books = Book.objects.annotate(
    high_ratings_count=Count('reviews', filter=Q(reviews__rating__gte=4))
)
```

**4. F() expressions:**
```python
# Update using database-level operations
from django.db.models import F

# Atomic update
Book.objects.filter(id=1).update(price=F('price') * 1.1)  # Increase by 10%

# Comparison
expensive_books = Book.objects.filter(price__gt=F('author__avg_book_price'))

# Arithmetic
books = Book.objects.annotate(
    discount_price=F('price') * 0.9,
    days_since_published=Now() - F('published_date')
)
```

**5. Q objects (Complex queries):**
```python
from django.db.models import Q

# OR queries
books = Book.objects.filter(
    Q(title__icontains='python') | Q(title__icontains='django')
)

# Complex conditions
books = Book.objects.filter(
    Q(price__lt=30) & (Q(author__name='John') | Q(published_date__year=2023))
)

# NOT
books = Book.objects.filter(~Q(author__name='John'))
```

**6. Bulk Operations:**
```python
# Bulk create
books = [
    Book(title=f'Book {i}', author=author, price=10 + i)
    for i in range(1000)
]
Book.objects.bulk_create(books, batch_size=100)

# Bulk update
books = Book.objects.filter(published_date__year=2023)
for book in books:
    book.price *= 1.1
Book.objects.bulk_update(books, ['price'], batch_size=100)

# Update all at once
Book.objects.filter(published_date__year=2023).update(price=F('price') * 1.1)

# Bulk delete
Book.objects.filter(published_date__year__lt=2020).delete()
```

**7. Raw SQL & Database Functions:**
```python
from django.db.models.functions import Concat, Lower, Upper, Coalesce

# Database functions
authors = Author.objects.annotate(
    full_name=Concat('first_name', models.Value(' '), 'last_name'),
    email_lower=Lower('email')
)

# Raw SQL (when necessary)
books = Book.objects.raw('SELECT * FROM book WHERE price > %s', [50])

# Execute raw SQL
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("UPDATE book SET price = price * 1.1 WHERE author_id = %s", [author_id])
```

**8. Database Indexes:**
```python
class Book(models.Model):
    title = models.CharField(max_length=200, db_index=True)  # Simple index
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    published_date = models.DateField()
    
    class Meta:
        indexes = [
            # Composite index
            models.Index(fields=['author', '-published_date']),
            
            # Partial index (PostgreSQL)
            models.Index(
                fields=['title'],
                name='title_active_idx',
                condition=Q(is_active=True)
            ),
            
            # Expression index (PostgreSQL)
            models.Index(
                Lower('title').desc(),
                name='title_lower_idx'
            ),
        ]
```

**9. Query Performance Tips:**
```python
# Use exists() instead of count()
if Book.objects.filter(author=author).exists():  # Fast
    pass
# vs
if Book.objects.filter(author=author).count() > 0:  # Slower

# Use iterator() for large querysets
for book in Book.objects.iterator(chunk_size=1000):
    process_book(book)  # Memory efficient

# Use values() or values_list() for simple data
book_titles = Book.objects.values_list('title', flat=True)
# Returns: ['Book 1', 'Book 2', ...]

# Use explain() to analyze queries
print(Book.objects.filter(price__gt=50).explain())
```

**10. Caching:**
```python
from django.core.cache import cache
from django.views.decorators.cache import cache_page

# Query result caching
def get_books():
    cache_key = 'all_books'
    books = cache.get(cache_key)
    
    if books is None:
        books = list(Book.objects.select_related('author').all())
        cache.set(cache_key, books, timeout=3600)  # 1 hour
    
    return books

# View caching
@cache_page(60 * 15)  # Cache for 15 minutes
def book_list(request):
    books = Book.objects.all()
    return render(request, 'books/list.html', {'books': books})

# Template fragment caching
{% load cache %}
{% cache 3600 book_list %}
    <!-- Expensive template rendering -->
{% endcache %}
```

---

### Q3: Explain Django signals and when to use them.
**Answer:**

**What are Django Signals:**
Signals allow decoupled applications to get notified when actions occur elsewhere in the framework.

**Built-in Signals:**

```python
from django.db.models.signals import (
    pre_save, post_save,
    pre_delete, post_delete,
    m2m_changed
)
from django.contrib.auth.signals import (
    user_logged_in, user_logged_out, user_login_failed
)
from django.core.signals import request_started, request_finished
from django.dispatch import receiver

# models.py
class Profile(models.Model):
    user = models.OneToOneField('auth.User', on_delete=models.CASCADE)
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', null=True)
    created_at = models.DateTimeField(auto_now_add=True)

# signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth.models import User

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    """Automatically create profile when user is created"""
    if created:
        Profile.objects.create(user=instance)
        print(f"Profile created for {instance.username}")

@receiver(post_save, sender=User)
def save_user_profile(sender, instance, **kwargs):
    """Save profile when user is saved"""
    if hasattr(instance, 'profile'):
        instance.profile.save()

# Alternative: Connect without decorator
def user_logged_in_handler(sender, request, user, **kwargs):
    """Track user login"""
    LoginLog.objects.create(
        user=user,
        ip_address=request.META.get('REMOTE_ADDR'),
        user_agent=request.META.get('HTTP_USER_AGENT')
    )

user_logged_in.connect(user_logged_in_handler)
```

**Signal Types:**

**1. Model Signals:**
```python
from django.db.models.signals import pre_save, post_save, pre_delete, post_delete

class Article(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    content = models.TextField()
    status = models.CharField(max_length=20)

@receiver(pre_save, sender=Article)
def generate_slug(sender, instance, **kwargs):
    """Generate slug before saving"""
    if not instance.slug:
        from django.utils.text import slugify
        instance.slug = slugify(instance.title)

@receiver(post_save, sender=Article)
def notify_on_publish(sender, instance, created, **kwargs):
    """Send notification when article is published"""
    if not created and instance.status == 'published':
        # Check if status changed
        try:
            old_instance = Article.objects.get(pk=instance.pk)
            if old_instance.status != 'published':
                send_publication_notification(instance)
        except Article.DoesNotExist:
            pass

@receiver(pre_delete, sender=Article)
def backup_before_delete(sender, instance, **kwargs):
    """Backup article before deletion"""
    ArticleBackup.objects.create(
        original_id=instance.id,
        title=instance.title,
        content=instance.content,
        deleted_at=timezone.now()
    )

@receiver(post_delete, sender=Article)
def cleanup_files(sender, instance, **kwargs):
    """Delete associated files"""
    if instance.image:
        instance.image.delete(save=False)
```

**2. Many-to-Many Signals:**
```python
from django.db.models.signals import m2m_changed

class Course(models.Model):
    title = models.CharField(max_length=200)
    students = models.ManyToManyField('auth.User', related_name='courses')

@receiver(m2m_changed, sender=Course.students.through)
def notify_course_enrollment(sender, instance, action, pk_set, **kwargs):
    """Notify when students are added/removed"""
    if action == 'post_add':
        # Students added
        students = User.objects.filter(pk__in=pk_set)
        for student in students:
            send_enrollment_email(student, instance)
    
    elif action == 'post_remove':
        # Students removed
        students = User.objects.filter(pk__in=pk_set)
        for student in students:
            send_unenrollment_email(student, instance)
    
    elif action == 'pre_clear':
        # All students about to be removed
        print(f"Clearing all students from {instance.title}")
```

**3. Request/Response Signals:**
```python
from django.core.signals import request_started, request_finished

@receiver(request_started)
def log_request_started(sender, environ, **kwargs):
    """Log when request starts"""
    print(f"Request started: {environ.get('PATH_INFO')}")

@receiver(request_finished)
def log_request_finished(sender, **kwargs):
    """Log when request finishes"""
    print("Request finished")
```

**Custom Signals:**

```python
# signals.py
from django.dispatch import Signal

# Define custom signal
payment_completed = Signal()  # No providing_args in Django 4.0+

# models.py
class Order(models.Model):
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)

class Payment(models.Model):
    order = models.ForeignKey(Order, on_delete=models.CASCADE)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)

# views.py
from .signals import payment_completed

def process_payment(request):
    # ... payment processing ...
    
    if payment.status == 'completed':
        # Send signal
        payment_completed.send(
            sender=Payment,
            payment=payment,
            order=payment.order
        )
    
    return JsonResponse({'status': 'success'})

# handlers.py
from django.dispatch import receiver
from .signals import payment_completed

@receiver(payment_completed)
def update_order_status(sender, payment, order, **kwargs):
    """Update order status when payment completes"""
    order.status = 'paid'
    order.save()

@receiver(payment_completed)
def send_receipt(sender, payment, order, **kwargs):
    """Send receipt email"""
    send_email(
        to=order.user.email,
        subject='Payment Receipt',
        template='emails/receipt.html',
        context={'payment': payment, 'order': order}
    )

@receiver(payment_completed)
def update_inventory(sender, payment, order, **kwargs):
    """Update inventory after payment"""
    for item in order.items.all():
        item.product.stock -= item.quantity
        item.product.save()
```

**When to Use Signals:**

**Good Use Cases:**
1. Creating related objects (Profile when User created)
2. Logging and auditing
3. Cache invalidation
4. Sending notifications
5. Triggering background tasks
6. Keeping data in sync across apps

**When NOT to Use Signals:**
1. Simple operations that can be in save() method
2. When direct code is clearer
3. Heavy processing (use Celery instead)
4. When testing becomes difficult

**Better Alternatives:**

```python
# Instead of signal for simple operations
class User(models.Model):
    email = models.EmailField()
    
    def save(self, *args, **kwargs):
        # Direct operation in save()
        self.email = self.email.lower()
        super().save(*args, **kwargs)
        
        # Create profile if needed
        if not hasattr(self, 'profile'):
            Profile.objects.create(user=self)

# Instead of signal for heavy operations, use Celery
from celery import shared_task

@shared_task
def send_welcome_email(user_id):
    user = User.objects.get(id=user_id)
    # Send email (async)

# In view
def register_user(request):
    user = User.objects.create(...)
    send_welcome_email.delay(user.id)  # Background task
```

**Signal Best Practices:**

```python
# 1. Always disconnect in tests
from django.test import TestCase
from django.db.models.signals import post_save

class MyTestCase(TestCase):
    def setUp(self):
        post_save.disconnect(create_user_profile, sender=User)
    
    def tearDown(self):
        post_save.connect(create_user_profile, sender=User)

# 2. Use dispatch_uid to prevent duplicate signals
@receiver(post_save, sender=User, dispatch_uid='create_user_profile')
def create_user_profile(sender, instance, created, **kwargs):
    pass

# 3. Be careful with exceptions
@receiver(post_save, sender=User)
def safe_signal_handler(sender, instance, **kwargs):
    try:
        # Your code
        pass
    except Exception as e:
        logger.error(f"Signal failed: {e}")
        # Don't let signal errors break the save

# 4. Register signals in AppConfig
# apps.py
from django.apps import AppConfig

class MyAppConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'myapp'
    
    def ready(self):
        import myapp.signals  # Import signals
```

---

## 2. DJANGO REST FRAMEWORK (DRF)

### Q4: Explain DRF serializers and their types.
**Answer:**

**Serializer Types:**

**1. Basic Serializer:**
```python
from rest_framework import serializers

class ArticleSerializer(serializers.Serializer):
    """Manual field definition"""
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    author = serializers.CharField(max_length=100)
    published_date = serializers.DateTimeField()
    is_published = serializers.BooleanField(default=False)
    
    def create(self, validated_data):
        """Create new instance"""
        return Article.objects.create(**validated_data)
    
    def update(self, instance, validated_data):
        """Update existing instance"""
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.is_published = validated_data.get('is_published', instance.is_published)
        instance.save()
        return instance
```

**2. ModelSerializer (Most Common):**
```python
from rest_framework import serializers
from .models import Article, Author, Comment

class AuthorSerializer(serializers.ModelSerializer):
    """Automatically creates fields from model"""
    book_count = serializers.SerializerMethodField()
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'email', 'bio', 'book_count']
        read_only_fields = ['id']
    
    def get_book_count(self, obj):
        """Custom method field"""
        return obj.books.count()

class ArticleSerializer(serializers.ModelSerializer):
    # Custom fields
    author_name = serializers.CharField(source='author.name', read_only=True)
    comment_count = serializers.SerializerMethodField()
    
    # Nested serializer
    author = AuthorSerializer(read_only=True)
    author_id = serializers.IntegerField(write_only=True)
    
    class Meta:
        model = Article
        fields = [
            'id', 'title', 'content', 'author', 'author_id',
            'author_name', 'published_date', 'is_published',
            'comment_count', 'created_at', 'updated_at'
        ]
        read_only_fields = ['id', 'created_at', 'updated_at']
        extra_kwargs = {
            'content': {'write_only': True},  # Don't return in response
            'title': {'required': True, 'allow_blank': False}
        }
    
    def get_comment_count(self, obj):
        return obj.comments.count()
    
    def validate_title(self, value):
        """Field-level validation"""
        if len(value) < 5:
            raise serializers.ValidationError("Title must be at least 5 characters")
        return value
    
    def validate(self, data):
        """Object-level validation"""
        if data.get('is_published') and not data.get('content'):
            raise serializers.ValidationError("Published articles must have content")
        return data

class CommentSerializer(serializers.ModelSerializer):
    user_name = serializers.CharField(source='user.username', read_only=True)
    
    class Meta:
        model = Comment
        fields = ['id', 'article', 'user', 'user_name', 'text', 'created_at']
        read_only_fields = ['id', 'created_at', 'user']
```

**3. Nested Serializers:**
```python
class ArticleDetailSerializer(serializers.ModelSerializer):
    """Detailed article with nested comments"""
    author = AuthorSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'author', 'comments', 'published_date']

# Usage
article = Article.objects.prefetch_related('comments', 'comments__user').get(pk=1)
serializer = ArticleDetailSerializer(article)
print(serializer.data)
# Output:
# {
#     'id': 1,
#     'title': 'My Article',
#     'content': '...',
#     'author': {'id': 1, 'name': 'John Doe', ...},
#     'comments': [
#         {'id': 1, 'text': 'Great article!', ...},
#         {'id': 2, 'text': 'Thanks for sharing', ...}
#     ]
# }
```

**4. Writable Nested Serializers:**
```python
class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'isbn']

class AuthorWithBooksSerializer(serializers.ModelSerializer):
    books = BookSerializer(many=True)
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'email', 'books']
    
    def create(self, validated_data):
        """Handle nested creation"""
        books_data = validated_data.pop('books')
        author = Author.objects.create(**validated_data)
        
        for book_data in books_data:
            Book.objects.create(author=author, **book_data)
        
        return author
    
    def update(self, instance, validated_data):
        """Handle nested updates"""
        books_data = validated_data.pop('books', None)
        
        # Update author fields
        instance.name = validated_data.get('name', instance.name)
        instance.email = validated_data.get('email', instance.email)
        instance.save()
        
        # Update books
        if books_data is not None:
            # Simple approach: delete and recreate
            instance.books.all().delete()
            for book_data in books_data:
                Book.objects.create(author=instance, **book_data)
        
        return instance
```

**5. Dynamic Fields:**
```python
class DynamicFieldsSerializer(serializers.ModelSerializer):
    """Serializer that can exclude fields"""
    
    def __init__(self, *args, **kwargs):
        # Extract fields argument
        fields = kwargs.pop('fields', None)
        exclude = kwargs.pop('exclude', None)
        
        super().__init__(*args, **kwargs)
        
        if fields is not None:
            # Drop fields not in `fields`
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)
        
        if exclude is not None:
            # Drop fields in `exclude`
            for field_name in exclude:
                self.fields.pop(field_name, None)

class ArticleSerializer(DynamicFieldsSerializer):
    class Meta:
        model = Article
        fields = '__all__'

# Usage
# Only include specific fields
serializer = ArticleSerializer(article, fields=['id', 'title'])

# Exclude specific fields
serializer = ArticleSerializer(article, exclude=['content'])
```

**6. Context and Custom Methods:**
```python
class ArticleSerializer(serializers.ModelSerializer):
    is_owner = serializers.SerializerMethodField()
    can_edit = serializers.SerializerMethodField()
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'author', 'is_owner', 'can_edit']
    
    def get_is_owner(self, obj):
        """Check if current user is owner"""
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.author == request.user
        return False
    
    def get_can_edit(self, obj):
        """Check if user can edit"""
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.author == request.user or request.user.is_staff
        return False

# In view
serializer = ArticleSerializer(article, context={'request': request})
```

**7. Validation:**
```python
class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = '__all__'
    
    def validate_title(self, value):
        """Validate single field"""
        if 'badword' in value.lower():
            raise serializers.ValidationError("Title contains inappropriate content")
        return value
    
    def validate(self, data):
        """Validate multiple fields together"""
        if data.get('is_published'):
            if not data.get('content'):
                raise serializers.ValidationError("Published articles must have content")
            if not data.get('author'):
                raise serializers.ValidationError("Published articles must have an author")
        
        # Check uniqueness with custom logic
        if Article.objects.filter(
            title=data.get('title'),
            author=data.get('author')
        ).exclude(pk=self.instance.pk if self.instance else None).exists():
            raise serializers.ValidationError("Article with this title already exists for this author")
        
        return data
    
    def validate_published_date(self, value):
        """Validate date"""
        from django.utils import timezone
        if value > timezone.now():
            raise serializers.ValidationError("Published date cannot be in the future")
        return value
```

**8. Performance Optimization:**
```python
class OptimizedArticleSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'author', 'created_at']
    
    @classmethod
    def setup_eager_loading(cls, queryset):
        """Optimize queryset for serializer"""
        queryset = queryset.select_related('author')
        queryset = queryset.prefetch_related('comments')
        return queryset

# In viewset
class ArticleViewSet(viewsets.ModelViewSet):
    serializer_class = OptimizedArticleSerializer
    
    def get_queryset(self):
        queryset = Article.objects.all()
        # Apply eager loading
        queryset = self.get_serializer_class().setup_eager_loading(queryset)
        return queryset
```

---

### Q5: Explain DRF ViewS, consumers.ChatConsumer.as_asgi()),
]
```

**Frontend (JavaScript):**

```javascript
// chat.js
const roomId = document.getElementById('room-id').value;
const chatSocket = new WebSocket(
    'ws://' + window.location.host + '/ws/chat/' + roomId + '/'
);

chatSocket.onopen = function(e) {
    console.log('WebSocket connected');
};

chatSocket.onmessage = function(e) {
    const data = JSON.parse(e.data);
    
    if (data.type === 'chat_message') {
        // Display message
        const messageDiv = document.createElement('div');
        messageDiv.className = 'message';
        messageDiv.innerHTML = `
            <strong>${data.user}</strong>: ${data.message}
            <span class="timestamp">${new Date(data.timestamp).toLocaleTimeString()}</span>
        `;
        document.getElementById('messages').appendChild(messageDiv);
        
        // Scroll to bottom
        document.getElementById('messages').scrollTop = 
            document.getElementById('messages').scrollHeight;
    }
    
    else if (data.type === 'typing') {
        // Show typing indicator
        document.getElementById('typing-indicator').textContent = 
            `${data.user} is typing...`;
        
        setTimeout(() => {
            document.getElementById('typing-indicator').textContent = '';
        }, 3000);
    }
    
    else if (data.type === 'user_list') {
        // Update user list
        const userList = document.getElementById('user-list');
        userList.innerHTML = '';
        data.users.forEach(user => {
            const li = document.createElement('li');
            li.textContent = user;
            userList.appendChild(li);
        });
    }
};

chatSocket.onclose = function(e) {
    console.error('WebSocket closed unexpectedly');
};

// Send message
document.getElementById('send-button').onclick = function(e) {
    const messageInput = document.getElementById('message-input');
    const message = messageInput.value;
    
    chatSocket.send(JSON.stringify({
        'type': 'chat_message',
        'message': message
    }));
    
    messageInput.value = '';
};

// Typing indicator
let typingTimeout;
document.getElementById('message-input').onkeyup = function(e) {
    clearTimeout(typingTimeout);
    
    chatSocket.send(JSON.stringify({
        'type': 'typing'
    }));
    
    typingTimeout = setTimeout(() => {
        // Stop typing indicator after 3 seconds
    }, 3000);
};
```

**Real-time Notifications:**

```python
# notifications/consumers.py
class NotificationConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.user = self.scope['user']
        
        if self.user.is_anonymous:
            await self.close()
            return
        
        self.notification_group_name = f'notifications_{self.user.id}'
        
        await self.channel_layer.group_add(
            self.notification_group_name,
            self.channel_name
        )
        
        await self.accept()
        
        # Send unread notifications
        unread_count = await self.get_unread_count()
        await self.send(text_data=json.dumps({
            'type': 'unread_count',
            'count': unread_count
        }))
    
    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(
            self.notification_group_name,
            self.channel_name
        )
    
    async def notification(self, event):
        """Send notification to user"""
        await self.send(text_data=json.dumps({
            'type': 'notification',
            'title': event['title'],
            'message': event['message'],
            'url': event.get('url'),
            'timestamp': event['timestamp']
        }))
    
    @database_sync_to_async
    def get_unread_count(self):
        return self.user.notifications.filter(is_read=False).count()

# Send notification from anywhere
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync

def send_notification(user_id, title, message, url=None):
    channel_layer = get_channel_layer()
    async_to_sync(channel_layer.group_send)(
        f'notifications_{user_id}',
        {
            'type': 'notification',
            'title': title,
            'message': message,
            'url': url,
            'timestamp': timezone.now().isoformat()
        }
    )

# Usage in views/signals
def create_post(request):
    post = Post.objects.create(...)
    
    # Notify followers
    for follower in request.user.followers.all():
        send_notification(
            follower.id,
            'New Post',
            f'{request.user.username} posted: {post.title}',
            url=f'/posts/{post.id}/'
        )
```

---

## FINAL TIPS FOR DJANGO INTERVIEWS

### Key Areas to Focus On:

**1. Core Django:**
- MTV architecture
- ORM and query optimization
- Forms and validation
- Class-based views vs function-based views
- Middleware
- Signals
- Custom management commands

**2. Django REST Framework:**
- Serializers (all types)
- ViewSets and Generic Views
- Authentication & Permissions
- Throttling and pagination
- Filtering and search

**3. Performance:**
- Database query optimization
- Caching strategies (Redis)
- select_related / prefetch_related
- Database indexing
- Async views (Django 3.1+)

**4. Security:**
- CSRF, XSS, SQL injection prevention
- Authentication and authorization
- Secure session management
- Input validation
- Security headers

**5. Testing:**
- Unit tests, integration tests
- Test fixtures and factories
- Mocking external dependencies
- Coverage

**6. Deployment:**
- Production settings
- Gunicorn/uWSGI
- Nginx configuration
- Docker and docker-compose
- CI/CD pipelines

**7. Scalability:**
- Load balancing
- Database replication
- Celery for background tasks
- Horizontal scaling
- Monitoring and logging

### Common Interview Questions:

1. **Explain Django's request-response cycle**
2. **What are middleware and custom middleware creation?**
3. **Difference between select_related and prefetch_related**
4. **How do you handle file uploads securely?**
5. **Explain Django's authentication system**
6. **How do you optimize database queries?**
7. **What are Django signals and when to use them?**
8. **Explain caching in Django**
9. **How do you deploy Django to production?**
10. **What are Django management commands?**

### Best Practices to Mention:

- Follow Django coding style (PEP 8)
- Use environment variables for sensitive data
- Write comprehensive tests
- Use version control (Git)
- Document your code
- Use Django's built-in features before third-party packages
- Keep dependencies updated
- Use virtual environments
- Follow security best practices
- Monitor application performance

### Good luck with your Django interview! 🚀_FRAME_OPTIONS = 'DENY'  # Prevent clickjacking

# HTTPS redirect
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# HSTS (HTTP Strict Transport Security)
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Referrer policy
SECURE_REFERRER_POLICY = 'same-origin'
```

**6. Input Validation & Sanitization:**

```python
from django import forms
from django.core.validators import EmailValidator, URLValidator, RegexValidator

class ArticleForm(forms.ModelForm):
    # Field validation
    title = forms.CharField(
        max_length=200,
        validators=[
            RegexValidator(
                regex=r'^[a-zA-Z0-9\s\-]+### Q5: Explain DRF ViewSets, Generic Views, and their differences.
**Answer:**

**1. Function-Based Views (FBV):**
```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated

@api_view(['GET', 'POST'])
@permission_classes([IsAuthenticated])
def article_list(request):
    """List articles or create new article"""
    if request.method == 'GET':
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

@api_view(['GET', 'PUT', 'DELETE'])
def article_detail(request, pk):
    """Retrieve, update or delete article"""
    try:
        article = Article.objects.get(pk=pk)
    except Article.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)
    
    if request.method == 'GET':
        serializer = ArticleSerializer(article)
        return Response(serializer.data)
    
    elif request.method == 'PUT':
        serializer = ArticleSerializer(article, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    elif request.method == 'DELETE':
        article.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**2. Class-Based Views (CBV):**
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class ArticleList(APIView):
    """List all articles or create new article"""
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True, context={'request': request})
        return Response(serializer.data)
    
    def post(self, request):
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class ArticleDetail(APIView):
    """Retrieve, update or delete article"""
    
    def get_object(self, pk):
        try:
            return Article.objects.get(pk=pk)
        except Article.DoesNotExist:
            raise Http404
    
    def get(self, request, pk):
        article = self.get_object(pk)
        serializer = ArticleSerializer(article)
        return Response(serializer.data)
    
    def put(self, request, pk):
        article = self.get_object(pk)
        serializer = ArticleSerializer(article, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def delete(self, request, pk):
        article = self.get_object(pk)
        article.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**3. Generic Views:**
```python
from rest_framework import generics
from rest_framework.permissions import IsAuthenticatedOrReadOnly

# List and Create
class ArticleList(generics.ListCreateAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def perform_create(self, serializer):
        """Customize creation"""
        serializer.save(author=self.request.user)

# Retrieve, Update, Delete
class ArticleDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]

# Other generic views:
# - ListAPIView: Read-only list
# - CreateAPIView: Create only
# - RetrieveAPIView: Read-only single object
# - UpdateAPIView: Update only
# - DestroyAPIView: Delete only
# - RetrieveUpdateAPIView: Read and update
```

**4. ViewSets:**
```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

class ArticleViewSet(viewsets.ModelViewSet):
    """Complete CRUD operations"""
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filterset_fields = ['author', 'is_published']
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'title']
    
    def get_queryset(self):
        """Customize queryset"""
        queryset = super().get_queryset()
        
        # Optimize queries
        queryset = queryset.select_related('author')
        queryset = queryset.prefetch_related('comments')
        
        # Filter by user for non-staff
        if not self.request.user.is_staff:
            queryset = queryset.filter(
                models.Q(is_published=True) | models.Q(author=self.request.user)
            )
        
        return queryset
    
    def get_serializer_class(self):
        """Use different serializers for different actions"""
        if self.action == 'list':
            return ArticleListSerializer
        elif self.action == 'retrieve':
            return ArticleDetailSerializer
        return ArticleSerializer
    
    def perform_create(self, serializer):
        """Set author on creation"""
        serializer.save(author=self.request.user)
    
    def perform_update(self, serializer):
        """Custom update logic"""
        serializer.save(updated_by=self.request.user)
    
    # Custom actions
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """Publish article"""
        article = self.get_object()
        article.is_published = True
        article.published_at = timezone.now()
        article.save()
        
        serializer = self.get_serializer(article)
        return Response(serializer.data)
    
    @action(detail=False, methods=['get'])
    def my_articles(self, request):
        """Get current user's articles"""
        articles = self.get_queryset().filter(author=request.user)
        serializer = self.get_serializer(articles, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['get'])
    def comments(self, request, pk=None):
        """Get article comments"""
        article = self.get_object()
        comments = article.comments.all()
        serializer = CommentSerializer(comments, many=True)
        return Response(serializer.data)

# URLs for ViewSet
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'articles', ArticleViewSet, basename='article')

urlpatterns = router.urls

# Generated URLs:
# GET    /articles/              -> list
# POST   /articles/              -> create
# GET    /articles/{pk}/         -> retrieve
# PUT    /articles/{pk}/         -> update
# PATCH  /articles/{pk}/         -> partial_update
# DELETE /articles/{pk}/         -> destroy
# POST   /articles/{pk}/publish/ -> publish (custom action)
# GET    /articles/my_articles/  -> my_articles (custom action)
```

**5. ReadOnlyModelViewSet:**
```python
class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    """Read-only viewset - only list and retrieve"""
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    
    # Only provides:
    # - list()
    # - retrieve()
```

**6. Custom ViewSet:**
```python
from rest_framework import viewsets, mixins

class CreateListRetrieveViewSet(
    mixins.CreateModelMixin,
    mixins.ListModelMixin,
    mixins.RetrieveModelMixin,
    viewsets.GenericViewSet
):
    """ViewSet that only allows create, list, and retrieve"""
    pass

class CommentViewSet(CreateListRetrieveViewSet):
    """Comments can only be created and viewed, not updated or deleted"""
    queryset = Comment.objects.all()
    serializer_class = CommentSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def perform_create(self, serializer):
        serializer.save(user=self.request.user)
```

**7. Advanced ViewSet Features:**
```python
from rest_framework import viewsets, filters
from django_filters.rest_framework import DjangoFilterBackend

class AdvancedArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    # Filtering
    filter_backends = [
        DjangoFilterBackend,
        filters.SearchFilter,
        filters.OrderingFilter
    ]
    filterset_fields = {
        'author': ['exact'],
        'is_published': ['exact'],
        'created_at': ['gte', 'lte'],
        'title': ['icontains']
    }
    search_fields = ['title', 'content', 'author__name']
    ordering_fields = ['created_at', 'title', 'updated_at']
    ordering = ['-created_at']
    
    # Pagination
    pagination_class = PageNumberPagination
    
    def get_permissions(self):
        """Different permissions for different actions"""
        if self.action in ['list', 'retrieve']:
            return [permissions.AllowAny()]
        elif self.action in ['create']:
            return [permissions.IsAuthenticated()]
        elif self.action in ['update', 'partial_update', 'destroy']:
            return [permissions.IsAuthenticated(), IsOwnerOrReadOnly()]
        return super().get_permissions()
    
    def get_throttles(self):
        """Different throttles for different actions"""
        if self.action == 'create':
            return [UserRateThrottle()]
        return super().get_throttles()
    
    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def like(self, request, pk=None):
        """Like an article"""
        article = self.get_object()
        user = request.user
        
        if article.likes.filter(id=user.id).exists():
            article.likes.remove(user)
            return Response({'status': 'unliked'})
        else:
            article.likes.add(user)
            return Response({'status': 'liked'})
    
    @action(detail=False, methods=['get'])
    def trending(self, request):
        """Get trending articles"""
        from django.db.models import Count
        
        articles = self.get_queryset().annotate(
            like_count=Count('likes')
        ).order_by('-like_count')[:10]
        
        serializer = self.get_serializer(articles, many=True)
        return Response(serializer.data)
```

**Comparison:**

| Feature | FBV | APIView | GenericView | ViewSet |
|---------|-----|---------|-------------|---------|
| Code Amount | Most | Moderate | Less | Least |
| Flexibility | Highest | High | Moderate | Low |
| Reusability | Low | Moderate | High | Highest |
| Best For | Custom logic | Custom APIs | Standard CRUD | REST APIs |
| URL Routing | Manual | Manual | Manual | Automatic |

**When to Use What:**

**Function-Based Views:**
- Simple, one-off endpoints
- Very custom logic
- Learning/prototyping

**APIView:**
- Need full control
- Custom HTTP methods
- Complex business logic

**Generic Views:**
- Standard CRUD operations
- Want some customization
- Don't need all methods

**ViewSets:**
- Full REST API for a model
- Standard CRUD with minimal customization
- Need automatic URL routing
- Want consistent API structure

---

## 3. DJANGO PERFORMANCE & OPTIMIZATION

### Q6: How do you optimize Django application performance?
**Answer:**

**1. Database Query Optimization:**

```python
# BAD: N+1 Query Problem
def get_articles_bad():
    articles = Article.objects.all()
    for article in articles:
        print(article.author.name)  # N queries
        print(article.category.name)  # N queries

# GOOD: Use select_related
def get_articles_good():
    articles = Article.objects.select_related(
        'author', 'category'
    ).all()  # 1 query with JOINs
    
    for article in articles:
        print(article.author.name)  # No extra query
        print(article.category.name)  # No extra query

# GOOD: Use prefetch_related for reverse relations
def get_authors_with_articles():
    authors = Author.objects.prefetch_related('articles').all()
    
    for author in authors:
        for article in author.articles.all():  # No extra queries
            print(article.title)

# Advanced prefetch
from django.db.models import Prefetch

def get_authors_with_published_articles():
    published_articles = Article.objects.filter(is_published=True)
    
    authors = Author.objects.prefetch_related(
        Prefetch('articles', queryset=published_articles, to_attr='published_articles')
    ).all()
    
    for author in authors:
        for article in author.published_articles:  # Use prefetched data
            print(article.title)
```

**2. Caching Strategies:**

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'KEY_PREFIX': 'myapp',
        'TIMEOUT': 300,
    }
}

# Low-level cache API
from django.core.cache import cache

def get_article(article_id):
    cache_key = f'article_{article_id}'
    article = cache.get(cache_key)
    
    if article is None:
        article = Article.objects.select_related('author').get(id=article_id)
        cache.set(cache_key, article, timeout=3600)  # 1 hour
    
    return article

# Invalidate cache on update
from django.db.models.signals import post_save, post_delete

@receiver([post_save, post_delete], sender=Article)
def invalidate_article_cache(sender, instance, **kwargs):
    cache_key = f'article_{instance.id}'
    cache.delete(cache_key)

# Template fragment caching
{% load cache %}
{% cache 3600 sidebar %}
    <!-- Expensive sidebar rendering -->
    {% for category in categories %}
        <li>{{ category.name }}</li>
    {% endfor %}
{% endcache %}

# Per-site cache
MIDDLEWARE = [
    'django.middleware.cache.UpdateCacheMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.cache.FetchFromCacheMiddleware',
]

CACHE_MIDDLEWARE_ALIAS = 'default'
CACHE_MIDDLEWARE_SECONDS = 600
CACHE_MIDDLEWARE_KEY_PREFIX = 'myapp'

# Per-view cache
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # Cache for 15 minutes
def article_list(request):
    articles = Article.objects.all()
    return render(request, 'articles/list.html', {'articles': articles})

# Cache with conditions
from django.views.decorators.vary import vary_on_cookie

@cache_page(60 * 15)
@vary_on_cookie
def my_view(request):
    # Cache varies based on cookie
    pass
```

**3. Database Indexing:**

```python
class Article(models.Model):
    title = models.CharField(max_length=200, db_index=True)  # Simple index
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    published_date = models.DateTimeField()
    is_published = models.BooleanField(default=False)
    slug = models.SlugField(unique=True)  # Automatically indexed
    
    class Meta:
        indexes = [
            # Composite index for common queries
            models.Index(fields=['author', '-published_date']),
            
            # Covering index
            models.Index(fields=['is_published', 'published_date', 'title']),
            
            # Partial index (PostgreSQL)
            models.Index(
                fields=['title'],
                name='published_title_idx',
                condition=models.Q(is_published=True)
            ),
            
            # Expression index (PostgreSQL)
            models.Index(
                Lower('title').desc(),
                name='title_lower_idx'
            ),
        ]

# Find missing indexes
python manage.py sqlmigrate myapp 0001 | grep "CREATE INDEX"

# Analyze queries
from django.db import connection

def show_queries():
    for query in connection.queries:
        print(query['sql'])
```

**4. Query Optimization:**

```python
# Use only() to fetch specific fields
articles = Article.objects.only('id', 'title', 'author_id')

# Use defer() to exclude heavy fields
articles = Article.objects.defer('content')

# Use values() or values_list() for simple data
titles = Article.objects.values_list('title', flat=True)

# Use exists() instead of count()
if Article.objects.filter(author=user).exists():  # Fast
    pass

# Use iterator() for large datasets
for article in Article.objects.iterator(chunk_size=1000):
    process_article(article)  # Memory efficient

# Use bulk operations
articles = [Article(title=f'Article {i}') for i in range(1000)]
Article.objects.bulk_create(articles, batch_size=100)

# Use update() instead of save() for updates
Article.objects.filter(author=user).update(is_published=True)

# Use annotate for computed fields
from django.db.models import Count, Avg

authors = Author.objects.annotate(
    article_count=Count('articles'),
    avg_rating=Avg('articles__rating')
)

# Use aggregation
from django.db.models import Sum

total_views = Article.objects.aggregate(total=Sum('view_count'))
```

**5. Lazy Loading & Eager Loading:**

```python
# Lazy loading (queries execute when needed)
articles = Article.objects.filter(is_published=True)  # No query yet
for article in articles:  # Query executes here
    print(article.title)

# Force evaluation
articles = list(Article.objects.all())  # Execute now

# Eager loading with select_related
article = Article.objects.select_related('author', 'category').get(id=1)

# Eager loading with prefetch_related
authors = Author.objects.prefetch_related(
    'articles',
    'articles__comments'
).all()
```

**6. Asynchronous Processing:**

```python
# Use Celery for background tasks
from celery import shared_task

@shared_task
def send_newsletter(article_id):
    article = Article.objects.get(id=article_id)
    users = User.objects.filter(subscribed=True)
    
    for user in users:
        send_email(user.email, article)

# In view
def publish_article(request, pk):
    article = Article.objects.get(pk=pk)
    article.is_published = True
    article.save()
    
    # Send newsletter asynchronously
    send_newsletter.delay(article.id)
    
    return JsonResponse({'status': 'published'})

# Periodic tasks
from celery.schedules import crontab

@app.task
def cleanup_old_data():
    threshold = timezone.now() - timedelta(days=30)
    Article.objects.filter(created_at__lt=threshold, is_published=False).delete()

# Celery beat schedule
app.conf.beat_schedule = {
    'cleanup-every-day': {
        'task': 'myapp.tasks.cleanup_old_data',
        'schedule': crontab(hour=2, minute=0),
    },
}
```

**7. Connection Pooling:**

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'mypassword',
        'HOST': 'localhost',
        'PORT': '5432',
        'CONN_MAX_AGE': 600,  # Connection pooling (10 minutes)
        'OPTIONS': {
            'connect_timeout': 10,
            'options': '-c statement_timeout=30000'  # 30 seconds
        }
    }
}
```

**8. Middleware Optimization:**

```python
# Custom middleware for performance
class PerformanceMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        import time
        from django.db import connection
        
        # Start timing
        start_time = time.time()
        start_queries = len(connection.queries)
        
        # Process request
        response = self.get_response(request)
        
        # Calculate stats
        end_time = time.time()
        total_time = end_time - start_time
        num_queries = len(connection.queries) - start_queries
        
        # Add headers
        response['X-Response-Time'] = f'{total_time:.3f}s'
        response['X-Query-Count'] = str(num_queries)
        
        # Log slow requests
        if total_time > 1.0:  # Over 1 second
            logger.warning(
                f'Slow request: {request.path} took {total_time:.3f}s '
                f'with {num_queries} queries'
            )
        
        return response
```

**9. Static Files & CDN:**

```python
# settings.py
STATIC_URL = 'https://cdn.example.com/static/'
MEDIA_URL = 'https://cdn.example.com/media/'

# Use WhiteNoise for static files
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # Add this
    # ... other middleware
]

STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

**10. Monitoring & Profiling:**

```python
# Django Debug Toolbar
INSTALLED_APPS = [
    # ...
    'debug_toolbar',
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    # ...
]

INTERNAL_IPS = ['127.0.0.1']

# Django Silk for profiling
INSTALLED_APPS = [
    # ...
    'silk',
]

MIDDLEWARE = [
    'silk.middleware.SilkyMiddleware',
    # ...
]

# Custom profiling
import cProfile
import pstats

def profile_view(func):
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()
        
        result = func(*args, **kwargs)
        
        profiler.disable()
        stats = pstats.Stats(profiler)
        stats.sort_stats('cumulative')
        stats.print_stats(20)
        
        return result
    return wrapper

@profile_view
def expensive_view(request):
    # Your view logic
    pass
```

**Performance Checklist:**
- ✅ Use `select_related()` and `prefetch_related()`
- ✅ Add database indexes
- ✅ Implement caching (Redis)
- ✅ Use `only()` and `defer()` appropriately
- ✅ Bulk operations for mass updates
- ✅ Async tasks for heavy operations (Celery)
- ✅ Connection pooling
- ✅ CDN for static files
- ✅ Database query optimization
- ✅ Monitor with Django Debug Toolbar
- ✅ Use `iterator()` for large datasets
- ✅ Pagination for list views

---

## 4. DJANGO SECURITY

### Q7: Explain Django security best practices and common vulnerabilities.
**Answer:**

**1. SQL Injection Prevention:**

```python
# BAD - Vulnerable to SQL injection
def search_articles_bad(request):
    query = request.GET.get('q')
    articles = Article.objects.raw(
        f"SELECT * FROM articles WHERE title LIKE '%{query}%'"
    )
    return render(request, 'articles.html', {'articles': articles})

# GOOD - Use parameterized queries
def search_articles_good(request):
    query = request.GET.get('q')
    articles = Article.objects.raw(
        "SELECT * FROM articles WHERE title LIKE %s",
        [f'%{query}%']
    )
    return render(request, 'articles.html', {'articles': articles})

# BEST - Use ORM
def search_articles_best(request):
    query = request.GET.get('q')
    articles = Article.objects.filter(title__icontains=query)
    return render(request, 'articles.html', {'articles': articles})
```

**2. Cross-Site Scripting (XSS) Prevention:**

```python
# Django templates auto-escape by default
{% autoescape on %}
    {{ user_input }}  # Automatically escaped
{% endautoescape %}

# Explicitly mark as safe (use cautiously)
from django.utils.safestring import mark_safe

def render_html(content):
    # Sanitize first!
    import bleach
    clean_content = bleach.clean(
        content,
        tags=['p', 'b', 'i', 'u', 'a'],
        attributes={'a': ['href', 'title']},
        strip=True
    )
    return mark_safe(clean_content)

# In template
{{ content|safe }}  # Only if you're sure it's safe!

# JSON responses
from django.http import JsonResponse

def api_view(request):
    data = {'user_input': request.GET.get('input')}
    return JsonResponse(data)  # Automatically escapes
```

**3. Cross-Site Request Forgery (CSRF) Protection:**

```python
# Django CSRF protection is enabled by default
MIDDLEWARE = [
    'django.middleware.csrf.CsrfViewMiddleware',  # Required
    # ...
]

# In forms
<form method="post">
    {% csrf_token %}
    <!-- form fields -->
</form>

# AJAX requests
// Get CSRF token
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

const csrftoken = getCookie('csrftoken');

// Include in AJAX
fetch('/api/endpoint/', {
    method: 'POST',
    headers: {
        'X-CSRFToken': csrftoken,
        'Content-Type': 'application/json',
    },
    body: JSON.stringify(data)
});

# Exempt specific views (use cautiously)
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def webhook_view(request):
    # For third-party webhooks
    pass

# DRF CSRF with Session Authentication
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
}
```

**4. Authentication & Authorization:**

```python
# Strong password validation
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {'min_length': 12}
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]

# Custom password validator
from django.core.exceptions import ValidationError

class SpecialCharacterValidator:
    def validate(self, password, user=None):
        if not any(char in '!@#$%^&*()' for char in password):
            raise ValidationError(
                "Password must contain at least one special character",
                code='password_no_special',
            )
    
    def get_help_text(self):
        return "Your password must contain at least one special character (!@#$%^&*())"

# Secure authentication views
from django.contrib.auth import authenticate, login
from django.contrib.auth.decorators import login_required
from django.contrib.auth.mixins import LoginRequiredMixin

@login_required
def protected_view(request):
    return render(request, 'protected.html')

class ProtectedView(LoginRequiredMixin, View):
    login_url = '/login/'
    redirect_field_name = 'next'
    
    def get(self, request):
        return render(request, 'protected.html')

# Permission-based access
from django.contrib.auth.decorators import permission_required

@permission_required('myapp.can_publish', raise_exception=True)
def publish_article(request, pk):
    article = Article.objects.get(pk=pk)
    article.is_published = True
    article.save()
    return redirect('article_detail', pk=pk)

# Custom permissions
class Article(models.Model):
    # ... fields ...
    
    class Meta:
        permissions = [
            ("can_publish", "Can publish articles"),
            ("can_feature", "Can feature articles"),
        ]
```

**5. Secure Session Management:**

```python
# settings.py

# Session security
SESSION_COOKIE_SECURE = True  # HTTPS only
SESSION_COOKIE_HTTPONLY = True  # Not accessible via JavaScript
SESSION_COOKIE_SAMESITE = 'Strict'  # CSRF protection
SESSION_COOKIE_AGE = 3600  # 1 hour

# CSRF security
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_HTTPONLY = True
CSRF_COOKIE_SAMESITE = 'Strict'

# Security headers
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X# Django Senior Backend Developer Interview Preparation Guide
## Comprehensive Guide for Senior Django Developer Interviews

---

## 1. DJANGO CORE CONCEPTS

### Q1: Explain Django's MTV architecture and how it differs from MVC.
**Answer:**

**Django MTV (Model-Template-View):**
- **Model:** Data layer - defines database structure
- **Template:** Presentation layer - HTML with template tags
- **View:** Business logic - processes requests and returns responses

**Traditional MVC (Model-View-Controller):**
- **Model:** Data layer
- **View:** Presentation layer
- **Controller:** Business logic

**Key Difference:**
Django's "View" is equivalent to MVC's "Controller", and Django's "Template" is equivalent to MVC's "View". Django calls it MTV but it's essentially MVC with different naming.

```python
# models.py (Model)
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    published = models.BooleanField(default=False)
    
    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['published', '-created_at']),
        ]
    
    def __str__(self):
        return self.title

# views.py (View - Controller logic)
from django.shortcuts import render, get_object_or_404
from django.http import JsonResponse
from .models import Article

def article_list(request):
    """List all published articles"""
    articles = Article.objects.filter(published=True).select_related('author')
    return render(request, 'articles/list.html', {'articles': articles})

def article_detail(request, pk):
    """Display single article"""
    article = get_object_or_404(Article, pk=pk, published=True)
    return render(request, 'articles/detail.html', {'article': article})

# templates/articles/list.html (Template - Presentation)
{% extends 'base.html' %}

{% block content %}
<h1>Articles</h1>
{% for article in articles %}
    <div class="article">
        <h2>{{ article.title }}</h2>
        <p>By {{ article.author.username }} on {{ article.created_at|date:"F d, Y" }}</p>
        <a href="{% url 'article_detail' article.pk %}">Read more</a>
    </div>
{% endfor %}
{% endblock %}

# urls.py (URL Configuration)
from django.urls import path
from . import views

urlpatterns = [
    path('articles/', views.article_list, name='article_list'),
    path('articles/<int:pk>/', views.article_detail, name='article_detail'),
]
```

---

### Q2: Explain Django's ORM and query optimization techniques.
**Answer:**

**Django ORM Basics:**

```python
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    bio = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name='books')
    published_date = models.DateField()
    isbn = models.CharField(max_length=13, unique=True)
    price = models.DecimalField(max_digits=6, decimal_places=2)
    
    class Meta:
        ordering = ['-published_date']
        indexes = [
            models.Index(fields=['author', '-published_date']),
        ]

class Review(models.Model):
    book = models.ForeignKey(Book, on_delete=models.CASCADE, related_name='reviews')
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    rating = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
    comment = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ['book', 'user']
```

**Query Optimization Techniques:**

**1. N+1 Query Problem:**
```python
# BAD - N+1 queries
books = Book.objects.all()  # 1 query
for book in books:
    print(book.author.name)  # N queries (one per book)

# GOOD - select_related (for ForeignKey, OneToOne)
books = Book.objects.select_related('author').all()  # 1 query with JOIN
for book in books:
    print(book.author.name)  # No additional queries

# Generated SQL:
# SELECT book.*, author.* FROM book 
# INNER JOIN author ON book.author_id = author.id

# GOOD - prefetch_related (for ManyToMany, reverse ForeignKey)
authors = Author.objects.prefetch_related('books').all()
for author in authors:
    for book in author.books.all():  # No additional queries
        print(book.title)

# Uses 2 queries:
# 1. SELECT * FROM author
# 2. SELECT * FROM book WHERE author_id IN (1, 2, 3, ...)
```

**2. Only/Defer:**
```python
# Only fetch specific fields
books = Book.objects.only('title', 'price')  # Fetch only title and price

# Defer heavy fields
books = Book.objects.defer('description')  # Fetch all except description

# With relations
books = Book.objects.select_related('author').only(
    'title', 'price', 'author__name'
)
```

**3. Aggregation:**
```python
from django.db.models import Count, Avg, Sum, Max, Min, F, Q

# Count books per author
authors = Author.objects.annotate(
    book_count=Count('books'),
    avg_price=Avg('books__price')
).filter(book_count__gt=0)

for author in authors:
    print(f"{author.name}: {author.book_count} books, avg price: {author.avg_price}")

# Complex aggregation
stats = Book.objects.aggregate(
    total_books=Count('id'),
    avg_price=Avg('price'),
    max_price=Max('price'),
    min_price=Min('price'),
    total_revenue=Sum('price')
)

# Annotate with conditions
books = Book.objects.annotate(
    high_ratings_count=Count('reviews', filter=Q(reviews__rating__gte=4))
)
```

**4. F() expressions:**
```python
# Update using database-level operations
from django.db.models import F

# Atomic update
Book.objects.filter(id=1).update(price=F('price') * 1.1)  # Increase by 10%

# Comparison
expensive_books = Book.objects.filter(price__gt=F('author__avg_book_price'))

# Arithmetic
books = Book.objects.annotate(
    discount_price=F('price') * 0.9,
    days_since_published=Now() - F('published_date')
)
```

**5. Q objects (Complex queries):**
```python
from django.db.models import Q

# OR queries
books = Book.objects.filter(
    Q(title__icontains='python') | Q(title__icontains='django')
)

# Complex conditions
books = Book.objects.filter(
    Q(price__lt=30) & (Q(author__name='John') | Q(published_date__year=2023))
)

# NOT
books = Book.objects.filter(~Q(author__name='John'))
```

**6. Bulk Operations:**
```python
# Bulk create
books = [
    Book(title=f'Book {i}', author=author, price=10 + i)
    for i in range(1000)
]
Book.objects.bulk_create(books, batch_size=100)

# Bulk update
books = Book.objects.filter(published_date__year=2023)
for book in books:
    book.price *= 1.1
Book.objects.bulk_update(books, ['price'], batch_size=100)

# Update all at once
Book.objects.filter(published_date__year=2023).update(price=F('price') * 1.1)

# Bulk delete
Book.objects.filter(published_date__year__lt=2020).delete()
```

**7. Raw SQL & Database Functions:**
```python
from django.db.models.functions import Concat, Lower, Upper, Coalesce

# Database functions
authors = Author.objects.annotate(
    full_name=Concat('first_name', models.Value(' '), 'last_name'),
    email_lower=Lower('email')
)

# Raw SQL (when necessary)
books = Book.objects.raw('SELECT * FROM book WHERE price > %s', [50])

# Execute raw SQL
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("UPDATE book SET price = price * 1.1 WHERE author_id = %s", [author_id])
```

**8. Database Indexes:**
```python
class Book(models.Model):
    title = models.CharField(max_length=200, db_index=True)  # Simple index
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    published_date = models.DateField()
    
    class Meta:
        indexes = [
            # Composite index
            models.Index(fields=['author', '-published_date']),
            
            # Partial index (PostgreSQL)
            models.Index(
                fields=['title'],
                name='title_active_idx',
                condition=Q(is_active=True)
            ),
            
            # Expression index (PostgreSQL)
            models.Index(
                Lower('title').desc(),
                name='title_lower_idx'
            ),
        ]
```

**9. Query Performance Tips:**
```python
# Use exists() instead of count()
if Book.objects.filter(author=author).exists():  # Fast
    pass
# vs
if Book.objects.filter(author=author).count() > 0:  # Slower

# Use iterator() for large querysets
for book in Book.objects.iterator(chunk_size=1000):
    process_book(book)  # Memory efficient

# Use values() or values_list() for simple data
book_titles = Book.objects.values_list('title', flat=True)
# Returns: ['Book 1', 'Book 2', ...]

# Use explain() to analyze queries
print(Book.objects.filter(price__gt=50).explain())
```

**10. Caching:**
```python
from django.core.cache import cache
from django.views.decorators.cache import cache_page

# Query result caching
def get_books():
    cache_key = 'all_books'
    books = cache.get(cache_key)
    
    if books is None:
        books = list(Book.objects.select_related('author').all())
        cache.set(cache_key, books, timeout=3600)  # 1 hour
    
    return books

# View caching
@cache_page(60 * 15)  # Cache for 15 minutes
def book_list(request):
    books = Book.objects.all()
    return render(request, 'books/list.html', {'books': books})

# Template fragment caching
{% load cache %}
{% cache 3600 book_list %}
    <!-- Expensive template rendering -->
{% endcache %}
```

---

### Q3: Explain Django signals and when to use them.
**Answer:**

**What are Django Signals:**
Signals allow decoupled applications to get notified when actions occur elsewhere in the framework.

**Built-in Signals:**

```python
from django.db.models.signals import (
    pre_save, post_save,
    pre_delete, post_delete,
    m2m_changed
)
from django.contrib.auth.signals import (
    user_logged_in, user_logged_out, user_login_failed
)
from django.core.signals import request_started, request_finished
from django.dispatch import receiver

# models.py
class Profile(models.Model):
    user = models.OneToOneField('auth.User', on_delete=models.CASCADE)
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', null=True)
    created_at = models.DateTimeField(auto_now_add=True)

# signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth.models import User

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    """Automatically create profile when user is created"""
    if created:
        Profile.objects.create(user=instance)
        print(f"Profile created for {instance.username}")

@receiver(post_save, sender=User)
def save_user_profile(sender, instance, **kwargs):
    """Save profile when user is saved"""
    if hasattr(instance, 'profile'):
        instance.profile.save()

# Alternative: Connect without decorator
def user_logged_in_handler(sender, request, user, **kwargs):
    """Track user login"""
    LoginLog.objects.create(
        user=user,
        ip_address=request.META.get('REMOTE_ADDR'),
        user_agent=request.META.get('HTTP_USER_AGENT')
    )

user_logged_in.connect(user_logged_in_handler)
```

**Signal Types:**

**1. Model Signals:**
```python
from django.db.models.signals import pre_save, post_save, pre_delete, post_delete

class Article(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    content = models.TextField()
    status = models.CharField(max_length=20)

@receiver(pre_save, sender=Article)
def generate_slug(sender, instance, **kwargs):
    """Generate slug before saving"""
    if not instance.slug:
        from django.utils.text import slugify
        instance.slug = slugify(instance.title)

@receiver(post_save, sender=Article)
def notify_on_publish(sender, instance, created, **kwargs):
    """Send notification when article is published"""
    if not created and instance.status == 'published':
        # Check if status changed
        try:
            old_instance = Article.objects.get(pk=instance.pk)
            if old_instance.status != 'published':
                send_publication_notification(instance)
        except Article.DoesNotExist:
            pass

@receiver(pre_delete, sender=Article)
def backup_before_delete(sender, instance, **kwargs):
    """Backup article before deletion"""
    ArticleBackup.objects.create(
        original_id=instance.id,
        title=instance.title,
        content=instance.content,
        deleted_at=timezone.now()
    )

@receiver(post_delete, sender=Article)
def cleanup_files(sender, instance, **kwargs):
    """Delete associated files"""
    if instance.image:
        instance.image.delete(save=False)
```

**2. Many-to-Many Signals:**
```python
from django.db.models.signals import m2m_changed

class Course(models.Model):
    title = models.CharField(max_length=200)
    students = models.ManyToManyField('auth.User', related_name='courses')

@receiver(m2m_changed, sender=Course.students.through)
def notify_course_enrollment(sender, instance, action, pk_set, **kwargs):
    """Notify when students are added/removed"""
    if action == 'post_add':
        # Students added
        students = User.objects.filter(pk__in=pk_set)
        for student in students:
            send_enrollment_email(student, instance)
    
    elif action == 'post_remove':
        # Students removed
        students = User.objects.filter(pk__in=pk_set)
        for student in students:
            send_unenrollment_email(student, instance)
    
    elif action == 'pre_clear':
        # All students about to be removed
        print(f"Clearing all students from {instance.title}")
```

**3. Request/Response Signals:**
```python
from django.core.signals import request_started, request_finished

@receiver(request_started)
def log_request_started(sender, environ, **kwargs):
    """Log when request starts"""
    print(f"Request started: {environ.get('PATH_INFO')}")

@receiver(request_finished)
def log_request_finished(sender, **kwargs):
    """Log when request finishes"""
    print("Request finished")
```

**Custom Signals:**

```python
# signals.py
from django.dispatch import Signal

# Define custom signal
payment_completed = Signal()  # No providing_args in Django 4.0+

# models.py
class Order(models.Model):
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)

class Payment(models.Model):
    order = models.ForeignKey(Order, on_delete=models.CASCADE)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)

# views.py
from .signals import payment_completed

def process_payment(request):
    # ... payment processing ...
    
    if payment.status == 'completed':
        # Send signal
        payment_completed.send(
            sender=Payment,
            payment=payment,
            order=payment.order
        )
    
    return JsonResponse({'status': 'success'})

# handlers.py
from django.dispatch import receiver
from .signals import payment_completed

@receiver(payment_completed)
def update_order_status(sender, payment, order, **kwargs):
    """Update order status when payment completes"""
    order.status = 'paid'
    order.save()

@receiver(payment_completed)
def send_receipt(sender, payment, order, **kwargs):
    """Send receipt email"""
    send_email(
        to=order.user.email,
        subject='Payment Receipt',
        template='emails/receipt.html',
        context={'payment': payment, 'order': order}
    )

@receiver(payment_completed)
def update_inventory(sender, payment, order, **kwargs):
    """Update inventory after payment"""
    for item in order.items.all():
        item.product.stock -= item.quantity
        item.product.save()
```

**When to Use Signals:**

**Good Use Cases:**
1. Creating related objects (Profile when User created)
2. Logging and auditing
3. Cache invalidation
4. Sending notifications
5. Triggering background tasks
6. Keeping data in sync across apps

**When NOT to Use Signals:**
1. Simple operations that can be in save() method
2. When direct code is clearer
3. Heavy processing (use Celery instead)
4. When testing becomes difficult

**Better Alternatives:**

```python
# Instead of signal for simple operations
class User(models.Model):
    email = models.EmailField()
    
    def save(self, *args, **kwargs):
        # Direct operation in save()
        self.email = self.email.lower()
        super().save(*args, **kwargs)
        
        # Create profile if needed
        if not hasattr(self, 'profile'):
            Profile.objects.create(user=self)

# Instead of signal for heavy operations, use Celery
from celery import shared_task

@shared_task
def send_welcome_email(user_id):
    user = User.objects.get(id=user_id)
    # Send email (async)

# In view
def register_user(request):
    user = User.objects.create(...)
    send_welcome_email.delay(user.id)  # Background task
```

**Signal Best Practices:**

```python
# 1. Always disconnect in tests
from django.test import TestCase
from django.db.models.signals import post_save

class MyTestCase(TestCase):
    def setUp(self):
        post_save.disconnect(create_user_profile, sender=User)
    
    def tearDown(self):
        post_save.connect(create_user_profile, sender=User)

# 2. Use dispatch_uid to prevent duplicate signals
@receiver(post_save, sender=User, dispatch_uid='create_user_profile')
def create_user_profile(sender, instance, created, **kwargs):
    pass

# 3. Be careful with exceptions
@receiver(post_save, sender=User)
def safe_signal_handler(sender, instance, **kwargs):
    try:
        # Your code
        pass
    except Exception as e:
        logger.error(f"Signal failed: {e}")
        # Don't let signal errors break the save

# 4. Register signals in AppConfig
# apps.py
from django.apps import AppConfig

class MyAppConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'myapp'
    
    def ready(self):
        import myapp.signals  # Import signals
```

---

## 2. DJANGO REST FRAMEWORK (DRF)

### Q4: Explain DRF serializers and their types.
**Answer:**

**Serializer Types:**

**1. Basic Serializer:**
```python
from rest_framework import serializers

class ArticleSerializer(serializers.Serializer):
    """Manual field definition"""
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    author = serializers.CharField(max_length=100)
    published_date = serializers.DateTimeField()
    is_published = serializers.BooleanField(default=False)
    
    def create(self, validated_data):
        """Create new instance"""
        return Article.objects.create(**validated_data)
    
    def update(self, instance, validated_data):
        """Update existing instance"""
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.is_published = validated_data.get('is_published', instance.is_published)
        instance.save()
        return instance
```

**2. ModelSerializer (Most Common):**
```python
from rest_framework import serializers
from .models import Article, Author, Comment

class AuthorSerializer(serializers.ModelSerializer):
    """Automatically creates fields from model"""
    book_count = serializers.SerializerMethodField()
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'email', 'bio', 'book_count']
        read_only_fields = ['id']
    
    def get_book_count(self, obj):
        """Custom method field"""
        return obj.books.count()

class ArticleSerializer(serializers.ModelSerializer):
    # Custom fields
    author_name = serializers.CharField(source='author.name', read_only=True)
    comment_count = serializers.SerializerMethodField()
    
    # Nested serializer
    author = AuthorSerializer(read_only=True)
    author_id = serializers.IntegerField(write_only=True)
    
    class Meta:
        model = Article
        fields = [
            'id', 'title', 'content', 'author', 'author_id',
            'author_name', 'published_date', 'is_published',
            'comment_count', 'created_at', 'updated_at'
        ]
        read_only_fields = ['id', 'created_at', 'updated_at']
        extra_kwargs = {
            'content': {'write_only': True},  # Don't return in response
            'title': {'required': True, 'allow_blank': False}
        }
    
    def get_comment_count(self, obj):
        return obj.comments.count()
    
    def validate_title(self, value):
        """Field-level validation"""
        if len(value) < 5:
            raise serializers.ValidationError("Title must be at least 5 characters")
        return value
    
    def validate(self, data):
        """Object-level validation"""
        if data.get('is_published') and not data.get('content'):
            raise serializers.ValidationError("Published articles must have content")
        return data

class CommentSerializer(serializers.ModelSerializer):
    user_name = serializers.CharField(source='user.username', read_only=True)
    
    class Meta:
        model = Comment
        fields = ['id', 'article', 'user', 'user_name', 'text', 'created_at']
        read_only_fields = ['id', 'created_at', 'user']
```

**3. Nested Serializers:**
```python
class ArticleDetailSerializer(serializers.ModelSerializer):
    """Detailed article with nested comments"""
    author = AuthorSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'author', 'comments', 'published_date']

# Usage
article = Article.objects.prefetch_related('comments', 'comments__user').get(pk=1)
serializer = ArticleDetailSerializer(article)
print(serializer.data)
# Output:
# {
#     'id': 1,
#     'title': 'My Article',
#     'content': '...',
#     'author': {'id': 1, 'name': 'John Doe', ...},
#     'comments': [
#         {'id': 1, 'text': 'Great article!', ...},
#         {'id': 2, 'text': 'Thanks for sharing', ...}
#     ]
# }
```

**4. Writable Nested Serializers:**
```python
class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'isbn']

class AuthorWithBooksSerializer(serializers.ModelSerializer):
    books = BookSerializer(many=True)
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'email', 'books']
    
    def create(self, validated_data):
        """Handle nested creation"""
        books_data = validated_data.pop('books')
        author = Author.objects.create(**validated_data)
        
        for book_data in books_data:
            Book.objects.create(author=author, **book_data)
        
        return author
    
    def update(self, instance, validated_data):
        """Handle nested updates"""
        books_data = validated_data.pop('books', None)
        
        # Update author fields
        instance.name = validated_data.get('name', instance.name)
        instance.email = validated_data.get('email', instance.email)
        instance.save()
        
        # Update books
        if books_data is not None:
            # Simple approach: delete and recreate
            instance.books.all().delete()
            for book_data in books_data:
                Book.objects.create(author=instance, **book_data)
        
        return instance
```

**5. Dynamic Fields:**
```python
class DynamicFieldsSerializer(serializers.ModelSerializer):
    """Serializer that can exclude fields"""
    
    def __init__(self, *args, **kwargs):
        # Extract fields argument
        fields = kwargs.pop('fields', None)
        exclude = kwargs.pop('exclude', None)
        
        super().__init__(*args, **kwargs)
        
        if fields is not None:
            # Drop fields not in `fields`
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)
        
        if exclude is not None:
            # Drop fields in `exclude`
            for field_name in exclude:
                self.fields.pop(field_name, None)

class ArticleSerializer(DynamicFieldsSerializer):
    class Meta:
        model = Article
        fields = '__all__'

# Usage
# Only include specific fields
serializer = ArticleSerializer(article, fields=['id', 'title'])

# Exclude specific fields
serializer = ArticleSerializer(article, exclude=['content'])
```

**6. Context and Custom Methods:**
```python
class ArticleSerializer(serializers.ModelSerializer):
    is_owner = serializers.SerializerMethodField()
    can_edit = serializers.SerializerMethodField()
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'author', 'is_owner', 'can_edit']
    
    def get_is_owner(self, obj):
        """Check if current user is owner"""
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.author == request.user
        return False
    
    def get_can_edit(self, obj):
        """Check if user can edit"""
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.author == request.user or request.user.is_staff
        return False

# In view
serializer = ArticleSerializer(article, context={'request': request})
```

**7. Validation:**
```python
class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = '__all__'
    
    def validate_title(self, value):
        """Validate single field"""
        if 'badword' in value.lower():
            raise serializers.ValidationError("Title contains inappropriate content")
        return value
    
    def validate(self, data):
        """Validate multiple fields together"""
        if data.get('is_published'):
            if not data.get('content'):
                raise serializers.ValidationError("Published articles must have content")
            if not data.get('author'):
                raise serializers.ValidationError("Published articles must have an author")
        
        # Check uniqueness with custom logic
        if Article.objects.filter(
            title=data.get('title'),
            author=data.get('author')
        ).exclude(pk=self.instance.pk if self.instance else None).exists():
            raise serializers.ValidationError("Article with this title already exists for this author")
        
        return data
    
    def validate_published_date(self, value):
        """Validate date"""
        from django.utils import timezone
        if value > timezone.now():
            raise serializers.ValidationError("Published date cannot be in the future")
        return value
```

**8. Performance Optimization:**
```python
class OptimizedArticleSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'author', 'created_at']
    
    @classmethod
    def setup_eager_loading(cls, queryset):
        """Optimize queryset for serializer"""
        queryset = queryset.select_related('author')
        queryset = queryset.prefetch_related('comments')
        return queryset

# In viewset
class ArticleViewSet(viewsets.ModelViewSet):
    serializer_class = OptimizedArticleSerializer
    
    def get_queryset(self):
        queryset = Article.objects.all()
        # Apply eager loading
        queryset = self.get_serializer_class().setup_eager_loading(queryset)
        return queryset
```

---

### Q5: Explain DRF ViewS,
                message='Title can only contain letters, numbers, spaces, and hyphens'
            )
        ]
    )
    
    email = forms.EmailField(validators=[EmailValidator()])
    website = forms.URLField(validators=[URLValidator()])
    
    class Meta:
        model = Article
        fields = ['title', 'content', 'email', 'website']
    
    def clean_title(self):
        """Custom field validation"""
        title = self.cleaned_data['title']
        
        # Check for bad words
        bad_words = ['spam', 'scam']
        if any(word in title.lower() for word in bad_words):
            raise forms.ValidationError("Title contains prohibited words")
        
        return title
    
    def clean(self):
        """Form-wide validation"""
        cleaned_data = super().clean()
        
        # Cross-field validation
        if cleaned_data.get('is_published') and not cleaned_data.get('content'):
            raise forms.ValidationError("Published articles must have content")
        
        return cleaned_data

# Sanitize HTML input
import bleach

ALLOWED_TAGS = ['p', 'br', 'strong', 'em', 'u', 'a', 'ul', 'ol', 'li']
ALLOWED_ATTRIBUTES = {'a': ['href', 'title']}

def sanitize_html(content):
    return bleach.clean(
        content,
        tags=ALLOWED_TAGS,
        attributes=ALLOWED_ATTRIBUTES,
        strip=True
    )

# In view
def create_article(request):
    if request.method == 'POST':
        form = ArticleForm(request.POST)
        if form.is_valid():
            article = form.save(commit=False)
            article.content = sanitize_html(article.content)
            article.save()
            return redirect('article_detail', pk=article.pk)
    else:
        form = ArticleForm()
    
    return render(request, 'article_form.html', {'form': form})
```

**7. File Upload Security:**

```python
import os
from django.core.validators import FileExtensionValidator
from django.core.exceptions import ValidationError

def validate_file_size(file):
    """Limit file size to 5MB"""
    max_size = 5 * 1024 * 1024
    if file.size > max_size:
        raise ValidationError(f"File size cannot exceed {max_size/1024/1024}MB")

class Document(models.Model):
    file = models.FileField(
        upload_to='documents/%Y/%m/%d/',
        validators=[
            FileExtensionValidator(
                allowed_extensions=['pdf', 'doc', 'docx', 'txt']
            ),
            validate_file_size
        ]
    )
    
    def save(self, *args, **kwargs):
        # Sanitize filename
        if self.file:
            name = os.path.basename(self.file.name)
            name = name.replace(' ', '_')
            # Remove special characters
            import re
            name = re.sub(r'[^a-zA-Z0-9._-]', '', name)
            self.file.name = name
        
        super().save(*args, **kwargs)

# In view
from django.core.files.storage import FileSystemStorage

def upload_file(request):
    if request.method == 'POST' and request.FILES.get('file'):
        uploaded_file = request.FILES['file']
        
        # Validate file type (check actual content, not just extension)
        import magic
        file_type = magic.from_buffer(uploaded_file.read(1024), mime=True)
        uploaded_file.seek(0)  # Reset file pointer
        
        allowed_types = ['application/pdf', 'text/plain']
        if file_type not in allowed_types:
            return JsonResponse({'error': 'Invalid file type'}, status=400)
        
        # Scan for malware (if using ClamAV)
        # import pyclamd
        # cd = pyclamd.ClamdUnixSocket()
        # result = cd.scan_stream(uploaded_file.read())
        
        fs = FileSystemStorage()
        filename = fs.save(uploaded_file.name, uploaded_file)
        
        return JsonResponse({'filename': filename})
```

**8. API Security (DRF):**

```python
# settings.py
REST_FRAMEWORK = {
    # Authentication
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    
    # Permissions
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    
    # Throttling
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
    },
    
    # Pagination
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}

# JWT Authentication
from rest_framework_simplejwt.tokens import RefreshToken

def get_tokens_for_user(user):
    refresh = RefreshToken.for_user(user)
    return {
        'refresh': str(refresh),
        'access': str(refresh.access_token),
    }

# Custom permission
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """Only owner can edit"""
    
    def has_object_permission(self, request, view, obj):
        # Read permissions allowed to any request
        if request.method in permissions.SAFE_METHODS:
            return True
        
        # Write permissions only to owner
        return obj.author == request.user

# Rate limiting per user
from rest_framework.throttling import UserRateThrottle

class BurstRateThrottle(UserRateThrottle):
    rate = '60/min'

class SustainedRateThrottle(UserRateThrottle):
    rate = '1000/day'

# In viewset
class ArticleViewSet(viewsets.ModelViewSet):
    permission_classes = [IsOwnerOrReadOnly]
    throttle_classes = [BurstRateThrottle, SustainedRateThrottle]
```

**9. Environment Variables & Secret Management:**

```python
# Never commit secrets!
# .env file
SECRET_KEY=your-secret-key-here
DATABASE_URL=postgresql://user:pass@localhost/dbname
AWS_ACCESS_KEY_ID=your-aws-key
AWS_SECRET_ACCESS_KEY=your-aws-secret

# settings.py
import os
from pathlib import Path
import environ

# Initialize environ
env = environ.Env(
    DEBUG=(bool, False)
)

# Read .env file
environ.Env.read_env(os.path.join(Path(__file__).resolve().parent.parent, '.env'))

# Get values
SECRET_KEY = env('SECRET_KEY')
DEBUG = env('DEBUG')
DATABASE_URL = env('DATABASE_URL')

DATABASES = {
    'default': env.db()
}

# .gitignore
.env
*.pyc
__pycache__/
db.sqlite3
```

**10. Logging & Monitoring:**

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'WARNING',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/django/security.log',
            'maxBytes': 1024*1024*15,  # 15MB
            'backupCount': 10,
            'formatter': 'verbose',
        },
        'mail_admins': {
            'level': 'ERROR',
            'class': 'django.utils.log.AdminEmailHandler',
        },
    },
    'loggers': {
        'django.security': {
            'handlers': ['file', 'mail_admins'],
            'level': 'WARNING',
            'propagate': False,
        },
    },
}

# Log security events
import logging

logger = logging.getLogger('django.security')

def login_view(request):
    username = request.POST.get('username')
    password = request.POST.get('password')
    
    user = authenticate(request, username=username, password=password)
    
    if user is not None:
        login(request, user)
        logger.info(f'Successful login: {username} from {request.META.get("REMOTE_ADDR")}')
    else:
        logger.warning(f'Failed login attempt: {username} from {request.META.get("REMOTE_ADDR")}')
```

**Security Checklist:**
- ✅ Keep Django and dependencies updated
- ✅ Use HTTPS in production
- ✅ Enable CSRF protection
- ✅ Validate and sanitize all user input
- ✅ Use parameterized queries (ORM)
- ✅ Implement proper authentication
- ✅ Use strong password policies
- ✅ Set secure cookie flags
- ✅ Enable security headers
- ✅ Implement rate limiting
- ✅ Use environment variables for secrets
- ✅ Log security events
- ✅ Keep SECRET_KEY secret
- ✅ Disable DEBUG in production
- ✅ Use allowed_hosts properly

---

## 5. DJANGO TESTING

### Q8: Explain Django testing strategies and best practices.
**Answer:**

**1. Unit Tests:**

```python
from django.test import TestCase
from django.contrib.auth.models import User
from .models import Article

class ArticleModelTest(TestCase):
    """Test Article model"""
    
    @classmethod
    def setUpTestData(cls):
        """Set up data for the whole TestCase"""
        cls.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        cls.article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=cls.user
        )
    
    def test_article_creation(self):
        """Test article is created correctly"""
        self.assertEqual(self.article.title, 'Test Article')
        self.assertEqual(self.article.author, self.user)
        self.assertFalse(self.article.is_published)
    
    def test_article_str(self):
        """Test string representation"""
        self.assertEqual(str(self.article), 'Test Article')
    
    def test_get_absolute_url(self):
        """Test get_absolute_url method"""
        expected_url = f'/articles/{self.article.id}/'
        self.assertEqual(self.article.get_absolute_url(), expected_url)
    
    def test_article_slug_generation(self):
        """Test slug is auto-generated"""
        article = Article.objects.create(
            title='New Article',
            content='Content',
            author=self.user
        )
        self.assertEqual(article.slug, 'new-article')
    
    def test_published_articles_manager(self):
        """Test custom manager"""
        Article.objects.create(
            title='Published',
            content='Content',
            author=self.user,
            is_published=True
        )
        
        published_count = Article.published.count()
        self.assertEqual(published_count, 1)
```

**2. View Tests:**

```python
from django.test import TestCase, Client
from django.urls import reverse

class ArticleViewTest(TestCase):
    """Test Article views"""
    
    def setUp(self):
        """Set up test client and data"""
        self.client = Client()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.user,
            is_published=True
        )
    
    def test_article_list_view(self):
        """Test article list view"""
        response = self.client.get(reverse('article_list'))
        
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, 'Test Article')
        self.assertTemplateUsed(response, 'articles/list.html')
    
    def test_article_detail_view(self):
        """Test article detail view"""
        url = reverse('article_detail', kwargs={'pk': self.article.pk})
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, self.article.title)
        self.assertEqual(response.context['article'], self.article)
    
    def test_article_create_view_authenticated(self):
        """Test creating article when logged in"""
        self.client.login(username='testuser', password='testpass123')
        
        data = {
            'title': 'New Article',
            'content': 'New content'
        }
        response = self.client.post(reverse('article_create'), data)
        
        self.assertEqual(response.status_code, 302)  # Redirect after success
        self.assertTrue(Article.objects.filter(title='New Article').exists())
    
    def test_article_create_view_unauthenticated(self):
        """Test creating article requires authentication"""
        data = {
            'title': 'New Article',
            'content': 'New content'
        }
        response = self.client.post(reverse('article_create'), data)
        
        self.assertEqual(response.status_code, 302)  # Redirect to login
        self.assertFalse(Article.objects.filter(title='New Article').exists())
    
    def test_article_update_view_owner(self):
        """Test owner can update article"""
        self.client.login(username='testuser', password='testpass123')
        
        url = reverse('article_update', kwargs={'pk': self.article.pk})
        data = {
            'title': 'Updated Title',
            'content': 'Updated content'
        }
        response = self.client.post(url, data)
        
        self.article.refresh_from_db()
        self.assertEqual(self.article.title, 'Updated Title')
    
    def test_article_delete_view(self):
        """Test deleting article"""
        self.client.login(username='testuser', password='testpass123')
        
        url = reverse('article_delete', kwargs={'pk': self.article.pk})
        response = self.client.post(url)
        
        self.assertFalse(Article.objects.filter(pk=self.article.pk).exists())
```

**3. Form Tests:**

```python
class ArticleFormTest(TestCase):
    """Test Article form"""
    
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
    
    def test_form_valid_data(self):
        """Test form with valid data"""
        form = ArticleForm(data={
            'title': 'Test Article',
            'content': 'Test content',
            'author': self.user.id
        })
        
        self.assertTrue(form.is_valid())
    
    def test_form_missing_title(self):
        """Test form validation with missing title"""
        form = ArticleForm(data={
            'content': 'Test content',
            'author': self.user.id
        })
        
        self.assertFalse(form.is_valid())
        self.assertIn('title', form.errors)
    
    def test_form_title_too_short(self):
        """Test custom validation"""
        form = ArticleForm(data={
            'title': 'abc',  # Too short
            'content': 'Test content',
            'author': self.user.id
        })
        
        self.assertFalse(form.is_valid())
        self.assertIn('title', form.errors)
    
    def test_form_save(self):
        """Test form save creates article"""
        form = ArticleForm(data={
            'title': 'Test Article',
            'content': 'Test content',
            'author': self.user.id
        })
        
        self.assertTrue(form.is_valid())
        article = form.save()
        
        self.assertEqual(article.title, 'Test Article')
        self.assertEqual(article.author, self.user)
```

**4. API Tests (DRF):**

```python
from rest_framework.test import APITestCase, APIClient
from rest_framework import status

class ArticleAPITest(APITestCase):
    """Test Article API"""
    
    def setUp(self):
        """Set up test client and data"""
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.user
        )
    
    def test_get_article_list(self):
        """Test GET /api/articles/"""
        response = self.client.get('/api/articles/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 1)
    
    def test_get_article_detail(self):
        """Test GET /api/articles/{id}/"""
        url = f'/api/articles/{self.article.id}/'
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], 'Test Article')
    
    def test_create_article_authenticated(self):
        """Test POST /api/articles/ when authenticated"""
        self.client.force_authenticate(user=self.user)
        
        data = {
            'title': 'New Article',
            'content': 'New content'
        }
        response = self.client.post('/api/articles/', data)
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Article.objects.count(), 2)
    
    def test_create_article_unauthenticated(self):
        """Test POST /api/articles/ requires authentication"""
        data = {
            'title': 'New Article',
            'content': 'New content'
        }
        response = self.client.post('/api/articles/', data)
        
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
    
    def test_update_article_owner(self):
        """Test PUT /api/articles/{id}/ by owner"""
        self.client.force_authenticate(user=self.user)
        
        url = f'/api/articles/{self.article.id}/'
        data = {
            'title': 'Updated Title',
            'content': 'Updated content'
        }
        response = self.client.put(url, data)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.article.refresh_from_db()
        self.assertEqual(self.article.title, 'Updated Title')
    
    def test_update_article_non_owner(self):
        """Test non-owner cannot update"""
        other_user = User.objects.create_user(
            username='otheruser',
            password='testpass123'
        )
        self.client.force_authenticate(user=other_user)
        
        url = f'/api/articles/{self.article.id}/'
        data = {
            'title': 'Updated Title',
            'content': 'Updated content'
        }
        response = self.client.put(url, data)
        
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
    
    def test_delete_article(self):
        """Test DELETE /api/articles/{id}/"""
        self.client.force_authenticate(user=self.user)
        
        url = f'/api/articles/{self.article.id}/'
        response = self.client.delete(url)
        
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertEqual(Article.objects.count(), 0)
```

**5. Factory Pattern (using factory_boy):**

```python
import factory
from factory.django import DjangoModelFactory

class UserFactory(DjangoModelFactory):
    class Meta:
        model = User
    
    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.LazyAttribute(lambda obj: f'{obj.username}@example.com')
    first_name = factory.Faker('first_name')
    last_name = factory.Faker('last_name')

class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article
    
    title = factory.Faker('sentence', nb_words=4)
    content = factory.Faker('paragraph', nb_sentences=5)
    author = factory.SubFactory(UserFactory)
    is_published = True

# Usage in tests
class ArticleTestCase(TestCase):
    def test_with_factory(self):
        # Create single article
        article = ArticleFactory()
        
        # Create multiple articles
        articles = ArticleFactory.create_batch(10)
        
        # Create with custom attributes
        article = ArticleFactory(title='Custom Title', is_published=False)
        
        # Create with related objects
        user = UserFactory(username='john')
        article = ArticleFactory(author=user)
```

**6. Mocking:**

```python
from unittest.mock import patch, Mock

class EmailTestCase(TestCase):
    """Test email sending"""
    
    @patch('myapp.tasks.send_email')
    def test_send_welcome_email(self, mock_send_email):
        """Test welcome email is sent"""
        user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        
        # Trigger action that sends email
        send_welcome_email(user.id)
        
        # Assert email was sent
        mock_send_email.assert_called_once_with(
            to='test@example.com',
            subject='Welcome!',
            template='emails/welcome.html'
        )
    
    @patch('requests.get')
    def test_fetch_external_data(self, mock_get):
        """Test fetching from external API"""
        # Mock response
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {'data': 'test'}
        mock_get.return_value = mock_response
        
        # Call function
        result = fetch_external_data('https://api.example.com/data')
        
        # Assert
        self.assertEqual(result, {'data': 'test'})
        mock_get.assert_called_once_with('https://api.example.com/data')
```

**7. Database Testing:**

```python
from django.test import TransactionTestCase

class ArticleTransactionTest(TransactionTestCase):
    """Tests that need database transactions"""
    
    def test_concurrent_updates(self):
        """Test handling concurrent updates"""
        from django.db import transaction
        
        article = Article.objects.create(
            title='Test',
            content='Content',
            author=self.user
        )
        
        with transaction.atomic():
            article.view_count += 1
            article.save()
        
        article.refresh_from_db()
        self.assertEqual(article.view_count, 1)
```

**8. Coverage:**

```python
# Install coverage
# pip install coverage

# Run tests with coverage
# coverage run --source='.' manage.py test
# coverage report
# coverage html

# .coveragerc
[run]
omit =
    */migrations/*
    */tests/*
    */venv/*
    manage.py

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
    if __name__ == .__main__.:
```

**Testing Best Practices:**
- ✅ Test models, views, forms, and APIs
- ✅ Use factories for test data
- ✅ Mock external dependencies
- ✅ Test edge cases and error conditions
- ✅ Use descriptive test names
- ✅ Aim for high coverage (>80%)
- ✅ Keep tests fast and isolated
- ✅ Use `setUpTestData` for class-level data
- ✅ Test permissions and authentication
- ✅ Use `TransactionTestCase` when needed

---

## 6. DJANGO DEPLOYMENT & SCALABILITY

### Q9: How do you deploy and scale Django applications?
**Answer:**

**1. Production Settings:**

```python
# settings/base.py
from pathlib import Path
import environ

env = environ.Env()

BASE_DIR = Path(__file__).resolve().parent.parent.parent

INSTALLED_APPS = [
    # Django apps
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # Third-party apps
    'rest_framework',
    'corsheaders',
    'django_filters',
    
    # Local apps
    'myapp',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # Static files
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

# settings/production.py
from .base import *

DEBUG = False

ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')

# Database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST'),
        'PORT': env('DB_PORT', default='5432'),
        'CONN_MAX_AGE': 600,
        'OPTIONS': {
            'connect_timeout': 10,
        }
    }
}

# Cache
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': env('REDIS_URL'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'CONNECTION_POOL_KWARGS': {'max_connections': 50}
        }
    }
}

# Static files
STATIC_ROOT = BASE_DIR / 'staticfiles'
STATIC_URL = '/static/'
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

# Media files
MEDIA_ROOT = BASE_DIR / 'media'
MEDIA_URL = '/media/'

# Security
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Email
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = env('EMAIL_HOST')
EMAIL_PORT = env.int('EMAIL_PORT', default=587)
EMAIL_USE_TLS = True
EMAIL_HOST_USER = env('EMAIL_HOST_USER')
EMAIL_HOST_PASSWORD = env('EMAIL_HOST_PASSWORD')

# Celery
CELERY_BROKER_URL = env('CELERY_BROKER_URL')
CELERY_RESULT_BACKEND = env('CELERY_RESULT_BACKEND')
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'

# Logging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'ERROR',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/django/error.log',
            'maxBytes': 1024*1024*15,  # 15MB
            'backupCount': 10,
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['file'],
            'level': 'ERROR',
            'propagate': True,
        },
    },
}
```

**2. Gunicorn Configuration:**

```python
# gunicorn_config.py
import multiprocessing

# Server socket
bind = '0.0.0.0:8000'
backlog = 2048

# Worker processes
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = 'sync'
worker_connections = 1000
timeout = 30
keepalive = 2

# Logging
accesslog = '/var/log/gunicorn/access.log'
errorlog = '/var/log/gunicorn/error.log'
loglevel = 'info'

# Process naming
proc_name = 'myapp'

# Server mechanics
daemon = False
pidfile = '/var/run/gunicorn/pid'
user = 'www-data'
group = 'www-data'
tmp_upload_dir = None

# SSL (if terminating SSL at Gunicorn)
# keyfile = '/path/to/key.pem'
# certfile### Q5: Explain DRF ViewSets, Generic Views, and their differences.
**Answer:**

**1. Function-Based Views (FBV):**
```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated

@api_view(['GET', 'POST'])
@permission_classes([IsAuthenticated])
def article_list(request):
    """List articles or create new article"""
    if request.method == 'GET':
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

@api_view(['GET', 'PUT', 'DELETE'])
def article_detail(request, pk):
    """Retrieve, update or delete article"""
    try:
        article = Article.objects.get(pk=pk)
    except Article.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)
    
    if request.method == 'GET':
        serializer = ArticleSerializer(article)
        return Response(serializer.data)
    
    elif request.method == 'PUT':
        serializer = ArticleSerializer(article, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    elif request.method == 'DELETE':
        article.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**2. Class-Based Views (CBV):**
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class ArticleList(APIView):
    """List all articles or create new article"""
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True, context={'request': request})
        return Response(serializer.data)
    
    def post(self, request):
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class ArticleDetail(APIView):
    """Retrieve, update or delete article"""
    
    def get_object(self, pk):
        try:
            return Article.objects.get(pk=pk)
        except Article.DoesNotExist:
            raise Http404
    
    def get(self, request, pk):
        article = self.get_object(pk)
        serializer = ArticleSerializer(article)
        return Response(serializer.data)
    
    def put(self, request, pk):
        article = self.get_object(pk)
        serializer = ArticleSerializer(article, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def delete(self, request, pk):
        article = self.get_object(pk)
        article.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**3. Generic Views:**
```python
from rest_framework import generics
from rest_framework.permissions import IsAuthenticatedOrReadOnly

# List and Create
class ArticleList(generics.ListCreateAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def perform_create(self, serializer):
        """Customize creation"""
        serializer.save(author=self.request.user)

# Retrieve, Update, Delete
class ArticleDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]

# Other generic views:
# - ListAPIView: Read-only list
# - CreateAPIView: Create only
# - RetrieveAPIView: Read-only single object
# - UpdateAPIView: Update only
# - DestroyAPIView: Delete only
# - RetrieveUpdateAPIView: Read and update
```

**4. ViewSets:**
```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

class ArticleViewSet(viewsets.ModelViewSet):
    """Complete CRUD operations"""
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filterset_fields = ['author', 'is_published']
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'title']
    
    def get_queryset(self):
        """Customize queryset"""
        queryset = super().get_queryset()
        
        # Optimize queries
        queryset = queryset.select_related('author')
        queryset = queryset.prefetch_related('comments')
        
        # Filter by user for non-staff
        if not self.request.user.is_staff:
            queryset = queryset.filter(
                models.Q(is_published=True) | models.Q(author=self.request.user)
            )
        
        return queryset
    
    def get_serializer_class(self):
        """Use different serializers for different actions"""
        if self.action == 'list':
            return ArticleListSerializer
        elif self.action == 'retrieve':
            return ArticleDetailSerializer
        return ArticleSerializer
    
    def perform_create(self, serializer):
        """Set author on creation"""
        serializer.save(author=self.request.user)
    
    def perform_update(self, serializer):
        """Custom update logic"""
        serializer.save(updated_by=self.request.user)
    
    # Custom actions
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """Publish article"""
        article = self.get_object()
        article.is_published = True
        article.published_at = timezone.now()
        article.save()
        
        serializer = self.get_serializer(article)
        return Response(serializer.data)
    
    @action(detail=False, methods=['get'])
    def my_articles(self, request):
        """Get current user's articles"""
        articles = self.get_queryset().filter(author=request.user)
        serializer = self.get_serializer(articles, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['get'])
    def comments(self, request, pk=None):
        """Get article comments"""
        article = self.get_object()
        comments = article.comments.all()
        serializer = CommentSerializer(comments, many=True)
        return Response(serializer.data)

# URLs for ViewSet
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'articles', ArticleViewSet, basename='article')

urlpatterns = router.urls

# Generated URLs:
# GET    /articles/              -> list
# POST   /articles/              -> create
# GET    /articles/{pk}/         -> retrieve
# PUT    /articles/{pk}/         -> update
# PATCH  /articles/{pk}/         -> partial_update
# DELETE /articles/{pk}/         -> destroy
# POST   /articles/{pk}/publish/ -> publish (custom action)
# GET    /articles/my_articles/  -> my_articles (custom action)
```

**5. ReadOnlyModelViewSet:**
```python
class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    """Read-only viewset - only list and retrieve"""
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    
    # Only provides:
    # - list()
    # - retrieve()
```

**6. Custom ViewSet:**
```python
from rest_framework import viewsets, mixins

class CreateListRetrieveViewSet(
    mixins.CreateModelMixin,
    mixins.ListModelMixin,
    mixins.RetrieveModelMixin,
    viewsets.GenericViewSet
):
    """ViewSet that only allows create, list, and retrieve"""
    pass

class CommentViewSet(CreateListRetrieveViewSet):
    """Comments can only be created and viewed, not updated or deleted"""
    queryset = Comment.objects.all()
    serializer_class = CommentSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def perform_create(self, serializer):
        serializer.save(user=self.request.user)
```

**7. Advanced ViewSet Features:**
```python
from rest_framework import viewsets, filters
from django_filters.rest_framework import DjangoFilterBackend

class AdvancedArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    # Filtering
    filter_backends = [
        DjangoFilterBackend,
        filters.SearchFilter,
        filters.OrderingFilter
    ]
    filterset_fields = {
        'author': ['exact'],
        'is_published': ['exact'],
        'created_at': ['gte', 'lte'],
        'title': ['icontains']
    }
    search_fields = ['title', 'content', 'author__name']
    ordering_fields = ['created_at', 'title', 'updated_at']
    ordering = ['-created_at']
    
    # Pagination
    pagination_class = PageNumberPagination
    
    def get_permissions(self):
        """Different permissions for different actions"""
        if self.action in ['list', 'retrieve']:
            return [permissions.AllowAny()]
        elif self.action in ['create']:
            return [permissions.IsAuthenticated()]
        elif self.action in ['update', 'partial_update', 'destroy']:
            return [permissions.IsAuthenticated(), IsOwnerOrReadOnly()]
        return super().get_permissions()
    
    def get_throttles(self):
        """Different throttles for different actions"""
        if self.action == 'create':
            return [UserRateThrottle()]
        return super().get_throttles()
    
    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def like(self, request, pk=None):
        """Like an article"""
        article = self.get_object()
        user = request.user
        
        if article.likes.filter(id=user.id).exists():
            article.likes.remove(user)
            return Response({'status': 'unliked'})
        else:
            article.likes.add(user)
            return Response({'status': 'liked'})
    
    @action(detail=False, methods=['get'])
    def trending(self, request):
        """Get trending articles"""
        from django.db.models import Count
        
        articles = self.get_queryset().annotate(
            like_count=Count('likes')
        ).order_by('-like_count')[:10]
        
        serializer = self.get_serializer(articles, many=True)
        return Response(serializer.data)
```

**Comparison:**

| Feature | FBV | APIView | GenericView | ViewSet |
|---------|-----|---------|-------------|---------|
| Code Amount | Most | Moderate | Less | Least |
| Flexibility | Highest | High | Moderate | Low |
| Reusability | Low | Moderate | High | Highest |
| Best For | Custom logic | Custom APIs | Standard CRUD | REST APIs |
| URL Routing | Manual | Manual | Manual | Automatic |

**When to Use What:**

**Function-Based Views:**
- Simple, one-off endpoints
- Very custom logic
- Learning/prototyping

**APIView:**
- Need full control
- Custom HTTP methods
- Complex business logic

**Generic Views:**
- Standard CRUD operations
- Want some customization
- Don't need all methods

**ViewSets:**
- Full REST API for a model
- Standard CRUD with minimal customization
- Need automatic URL routing
- Want consistent API structure

---

## 3. DJANGO PERFORMANCE & OPTIMIZATION

### Q6: How do you optimize Django application performance?
**Answer:**

**1. Database Query Optimization:**

```python
# BAD: N+1 Query Problem
def get_articles_bad():
    articles = Article.objects.all()
    for article in articles:
        print(article.author.name)  # N queries
        print(article.category.name)  # N queries

# GOOD: Use select_related
def get_articles_good():
    articles = Article.objects.select_related(
        'author', 'category'
    ).all()  # 1 query with JOINs
    
    for article in articles:
        print(article.author.name)  # No extra query
        print(article.category.name)  # No extra query

# GOOD: Use prefetch_related for reverse relations
def get_authors_with_articles():
    authors = Author.objects.prefetch_related('articles').all()
    
    for author in authors:
        for article in author.articles.all():  # No extra queries
            print(article.title)

# Advanced prefetch
from django.db.models import Prefetch

def get_authors_with_published_articles():
    published_articles = Article.objects.filter(is_published=True)
    
    authors = Author.objects.prefetch_related(
        Prefetch('articles', queryset=published_articles, to_attr='published_articles')
    ).all()
    
    for author in authors:
        for article in author.published_articles:  # Use prefetched data
            print(article.title)
```

**2. Caching Strategies:**

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'KEY_PREFIX': 'myapp',
        'TIMEOUT': 300,
    }
}

# Low-level cache API
from django.core.cache import cache

def get_article(article_id):
    cache_key = f'article_{article_id}'
    article = cache.get(cache_key)
    
    if article is None:
        article = Article.objects.select_related('author').get(id=article_id)
        cache.set(cache_key, article, timeout=3600)  # 1 hour
    
    return article

# Invalidate cache on update
from django.db.models.signals import post_save, post_delete

@receiver([post_save, post_delete], sender=Article)
def invalidate_article_cache(sender, instance, **kwargs):
    cache_key = f'article_{instance.id}'
    cache.delete(cache_key)

# Template fragment caching
{% load cache %}
{% cache 3600 sidebar %}
    <!-- Expensive sidebar rendering -->
    {% for category in categories %}
        <li>{{ category.name }}</li>
    {% endfor %}
{% endcache %}

# Per-site cache
MIDDLEWARE = [
    'django.middleware.cache.UpdateCacheMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.cache.FetchFromCacheMiddleware',
]

CACHE_MIDDLEWARE_ALIAS = 'default'
CACHE_MIDDLEWARE_SECONDS = 600
CACHE_MIDDLEWARE_KEY_PREFIX = 'myapp'

# Per-view cache
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # Cache for 15 minutes
def article_list(request):
    articles = Article.objects.all()
    return render(request, 'articles/list.html', {'articles': articles})

# Cache with conditions
from django.views.decorators.vary import vary_on_cookie

@cache_page(60 * 15)
@vary_on_cookie
def my_view(request):
    # Cache varies based on cookie
    pass
```

**3. Database Indexing:**

```python
class Article(models.Model):
    title = models.CharField(max_length=200, db_index=True)  # Simple index
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    published_date = models.DateTimeField()
    is_published = models.BooleanField(default=False)
    slug = models.SlugField(unique=True)  # Automatically indexed
    
    class Meta:
        indexes = [
            # Composite index for common queries
            models.Index(fields=['author', '-published_date']),
            
            # Covering index
            models.Index(fields=['is_published', 'published_date', 'title']),
            
            # Partial index (PostgreSQL)
            models.Index(
                fields=['title'],
                name='published_title_idx',
                condition=models.Q(is_published=True)
            ),
            
            # Expression index (PostgreSQL)
            models.Index(
                Lower('title').desc(),
                name='title_lower_idx'
            ),
        ]

# Find missing indexes
python manage.py sqlmigrate myapp 0001 | grep "CREATE INDEX"

# Analyze queries
from django.db import connection

def show_queries():
    for query in connection.queries:
        print(query['sql'])
```

**4. Query Optimization:**

```python
# Use only() to fetch specific fields
articles = Article.objects.only('id', 'title', 'author_id')

# Use defer() to exclude heavy fields
articles = Article.objects.defer('content')

# Use values() or values_list() for simple data
titles = Article.objects.values_list('title', flat=True)

# Use exists() instead of count()
if Article.objects.filter(author=user).exists():  # Fast
    pass

# Use iterator() for large datasets
for article in Article.objects.iterator(chunk_size=1000):
    process_article(article)  # Memory efficient

# Use bulk operations
articles = [Article(title=f'Article {i}') for i in range(1000)]
Article.objects.bulk_create(articles, batch_size=100)

# Use update() instead of save() for updates
Article.objects.filter(author=user).update(is_published=True)

# Use annotate for computed fields
from django.db.models import Count, Avg

authors = Author.objects.annotate(
    article_count=Count('articles'),
    avg_rating=Avg('articles__rating')
)

# Use aggregation
from django.db.models import Sum

total_views = Article.objects.aggregate(total=Sum('view_count'))
```

**5. Lazy Loading & Eager Loading:**

```python
# Lazy loading (queries execute when needed)
articles = Article.objects.filter(is_published=True)  # No query yet
for article in articles:  # Query executes here
    print(article.title)

# Force evaluation
articles = list(Article.objects.all())  # Execute now

# Eager loading with select_related
article = Article.objects.select_related('author', 'category').get(id=1)

# Eager loading with prefetch_related
authors = Author.objects.prefetch_related(
    'articles',
    'articles__comments'
).all()
```

**6. Asynchronous Processing:**

```python
# Use Celery for background tasks
from celery import shared_task

@shared_task
def send_newsletter(article_id):
    article = Article.objects.get(id=article_id)
    users = User.objects.filter(subscribed=True)
    
    for user in users:
        send_email(user.email, article)

# In view
def publish_article(request, pk):
    article = Article.objects.get(pk=pk)
    article.is_published = True
    article.save()
    
    # Send newsletter asynchronously
    send_newsletter.delay(article.id)
    
    return JsonResponse({'status': 'published'})

# Periodic tasks
from celery.schedules import crontab

@app.task
def cleanup_old_data():
    threshold = timezone.now() - timedelta(days=30)
    Article.objects.filter(created_at__lt=threshold, is_published=False).delete()

# Celery beat schedule
app.conf.beat_schedule = {
    'cleanup-every-day': {
        'task': 'myapp.tasks.cleanup_old_data',
        'schedule': crontab(hour=2, minute=0),
    },
}
```

**7. Connection Pooling:**

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'mypassword',
        'HOST': 'localhost',
        'PORT': '5432',
        'CONN_MAX_AGE': 600,  # Connection pooling (10 minutes)
        'OPTIONS': {
            'connect_timeout': 10,
            'options': '-c statement_timeout=30000'  # 30 seconds
        }
    }
}
```

**8. Middleware Optimization:**

```python
# Custom middleware for performance
class PerformanceMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        import time
        from django.db import connection
        
        # Start timing
        start_time = time.time()
        start_queries = len(connection.queries)
        
        # Process request
        response = self.get_response(request)
        
        # Calculate stats
        end_time = time.time()
        total_time = end_time - start_time
        num_queries = len(connection.queries) - start_queries
        
        # Add headers
        response['X-Response-Time'] = f'{total_time:.3f}s'
        response['X-Query-Count'] = str(num_queries)
        
        # Log slow requests
        if total_time > 1.0:  # Over 1 second
            logger.warning(
                f'Slow request: {request.path} took {total_time:.3f}s '
                f'with {num_queries} queries'
            )
        
        return response
```

**9. Static Files & CDN:**

```python
# settings.py
STATIC_URL = 'https://cdn.example.com/static/'
MEDIA_URL = 'https://cdn.example.com/media/'

# Use WhiteNoise for static files
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # Add this
    # ... other middleware
]

STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

**10. Monitoring & Profiling:**

```python
# Django Debug Toolbar
INSTALLED_APPS = [
    # ...
    'debug_toolbar',
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    # ...
]

INTERNAL_IPS = ['127.0.0.1']

# Django Silk for profiling
INSTALLED_APPS = [
    # ...
    'silk',
]

MIDDLEWARE = [
    'silk.middleware.SilkyMiddleware',
    # ...
]

# Custom profiling
import cProfile
import pstats

def profile_view(func):
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()
        
        result = func(*args, **kwargs)
        
        profiler.disable()
        stats = pstats.Stats(profiler)
        stats.sort_stats('cumulative')
        stats.print_stats(20)
        
        return result
    return wrapper

@profile_view
def expensive_view(request):
    # Your view logic
    pass
```

**Performance Checklist:**
- ✅ Use `select_related()` and `prefetch_related()`
- ✅ Add database indexes
- ✅ Implement caching (Redis)
- ✅ Use `only()` and `defer()` appropriately
- ✅ Bulk operations for mass updates
- ✅ Async tasks for heavy operations (Celery)
- ✅ Connection pooling
- ✅ CDN for static files
- ✅ Database query optimization
- ✅ Monitor with Django Debug Toolbar
- ✅ Use `iterator()` for large datasets
- ✅ Pagination for list views

---

## 4. DJANGO SECURITY

### Q7: Explain Django security best practices and common vulnerabilities.
**Answer:**

**1. SQL Injection Prevention:**

```python
# BAD - Vulnerable to SQL injection
def search_articles_bad(request):
    query = request.GET.get('q')
    articles = Article.objects.raw(
        f"SELECT * FROM articles WHERE title LIKE '%{query}%'"
    )
    return render(request, 'articles.html', {'articles': articles})

# GOOD - Use parameterized queries
def search_articles_good(request):
    query = request.GET.get('q')
    articles = Article.objects.raw(
        "SELECT * FROM articles WHERE title LIKE %s",
        [f'%{query}%']
    )
    return render(request, 'articles.html', {'articles': articles})

# BEST - Use ORM
def search_articles_best(request):
    query = request.GET.get('q')
    articles = Article.objects.filter(title__icontains=query)
    return render(request, 'articles.html', {'articles': articles})
```

**2. Cross-Site Scripting (XSS) Prevention:**

```python
# Django templates auto-escape by default
{% autoescape on %}
    {{ user_input }}  # Automatically escaped
{% endautoescape %}

# Explicitly mark as safe (use cautiously)
from django.utils.safestring import mark_safe

def render_html(content):
    # Sanitize first!
    import bleach
    clean_content = bleach.clean(
        content,
        tags=['p', 'b', 'i', 'u', 'a'],
        attributes={'a': ['href', 'title']},
        strip=True
    )
    return mark_safe(clean_content)

# In template
{{ content|safe }}  # Only if you're sure it's safe!

# JSON responses
from django.http import JsonResponse

def api_view(request):
    data = {'user_input': request.GET.get('input')}
    return JsonResponse(data)  # Automatically escapes
```

**3. Cross-Site Request Forgery (CSRF) Protection:**

```python
# Django CSRF protection is enabled by default
MIDDLEWARE = [
    'django.middleware.csrf.CsrfViewMiddleware',  # Required
    # ...
]

# In forms
<form method="post">
    {% csrf_token %}
    <!-- form fields -->
</form>

# AJAX requests
// Get CSRF token
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

const csrftoken = getCookie('csrftoken');

// Include in AJAX
fetch('/api/endpoint/', {
    method: 'POST',
    headers: {
        'X-CSRFToken': csrftoken,
        'Content-Type': 'application/json',
    },
    body: JSON.stringify(data)
});

# Exempt specific views (use cautiously)
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def webhook_view(request):
    # For third-party webhooks
    pass

# DRF CSRF with Session Authentication
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
}
```

**4. Authentication & Authorization:**

```python
# Strong password validation
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {'min_length': 12}
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]

# Custom password validator
from django.core.exceptions import ValidationError

class SpecialCharacterValidator:
    def validate(self, password, user=None):
        if not any(char in '!@#$%^&*()' for char in password):
            raise ValidationError(
                "Password must contain at least one special character",
                code='password_no_special',
            )
    
    def get_help_text(self):
        return "Your password must contain at least one special character (!@#$%^&*())"

# Secure authentication views
from django.contrib.auth import authenticate, login
from django.contrib.auth.decorators import login_required
from django.contrib.auth.mixins import LoginRequiredMixin

@login_required
def protected_view(request):
    return render(request, 'protected.html')

class ProtectedView(LoginRequiredMixin, View):
    login_url = '/login/'
    redirect_field_name = 'next'
    
    def get(self, request):
        return render(request, 'protected.html')

# Permission-based access
from django.contrib.auth.decorators import permission_required

@permission_required('myapp.can_publish', raise_exception=True)
def publish_article(request, pk):
    article = Article.objects.get(pk=pk)
    article.is_published = True
    article.save()
    return redirect('article_detail', pk=pk)

# Custom permissions
class Article(models.Model):
    # ... fields ...
    
    class Meta:
        permissions = [
            ("can_publish", "Can publish articles"),
            ("can_feature", "Can feature articles"),
        ]
```

**5. Secure Session Management:**

```python
# settings.py

# Session security
SESSION_COOKIE_SECURE = True  # HTTPS only
SESSION_COOKIE_HTTPONLY = True  # Not accessible via JavaScript
SESSION_COOKIE_SAMESITE = 'Strict'  # CSRF protection
SESSION_COOKIE_AGE = 3600  # 1 hour

# CSRF security
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_HTTPONLY = True
CSRF_COOKIE_SAMESITE = 'Strict'

# Security headers
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X# Django Senior Backend Developer Interview Preparation Guide
## Comprehensive Guide for Senior Django Developer Interviews

---

## 1. DJANGO CORE CONCEPTS

### Q1: Explain Django's MTV architecture and how it differs from MVC.
**Answer:**

**Django MTV (Model-Template-View):**
- **Model:** Data layer - defines database structure
- **Template:** Presentation layer - HTML with template tags
- **View:** Business logic - processes requests and returns responses

**Traditional MVC (Model-View-Controller):**
- **Model:** Data layer
- **View:** Presentation layer
- **Controller:** Business logic

**Key Difference:**
Django's "View" is equivalent to MVC's "Controller", and Django's "Template" is equivalent to MVC's "View". Django calls it MTV but it's essentially MVC with different naming.

```python
# models.py (Model)
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    published = models.BooleanField(default=False)
    
    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['published', '-created_at']),
        ]
    
    def __str__(self):
        return self.title

# views.py (View - Controller logic)
from django.shortcuts import render, get_object_or_404
from django.http import JsonResponse
from .models import Article

def article_list(request):
    """List all published articles"""
    articles = Article.objects.filter(published=True).select_related('author')
    return render(request, 'articles/list.html', {'articles': articles})

def article_detail(request, pk):
    """Display single article"""
    article = get_object_or_404(Article, pk=pk, published=True)
    return render(request, 'articles/detail.html', {'article': article})

# templates/articles/list.html (Template - Presentation)
{% extends 'base.html' %}

{% block content %}
<h1>Articles</h1>
{% for article in articles %}
    <div class="article">
        <h2>{{ article.title }}</h2>
        <p>By {{ article.author.username }} on {{ article.created_at|date:"F d, Y" }}</p>
        <a href="{% url 'article_detail' article.pk %}">Read more</a>
    </div>
{% endfor %}
{% endblock %}

# urls.py (URL Configuration)
from django.urls import path
from . import views

urlpatterns = [
    path('articles/', views.article_list, name='article_list'),
    path('articles/<int:pk>/', views.article_detail, name='article_detail'),
]
```

---

### Q2: Explain Django's ORM and query optimization techniques.
**Answer:**

**Django ORM Basics:**

```python
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    bio = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name='books')
    published_date = models.DateField()
    isbn = models.CharField(max_length=13, unique=True)
    price = models.DecimalField(max_digits=6, decimal_places=2)
    
    class Meta:
        ordering = ['-published_date']
        indexes = [
            models.Index(fields=['author', '-published_date']),
        ]

class Review(models.Model):
    book = models.ForeignKey(Book, on_delete=models.CASCADE, related_name='reviews')
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    rating = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
    comment = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ['book', 'user']
```

**Query Optimization Techniques:**

**1. N+1 Query Problem:**
```python
# BAD - N+1 queries
books = Book.objects.all()  # 1 query
for book in books:
    print(book.author.name)  # N queries (one per book)

# GOOD - select_related (for ForeignKey, OneToOne)
books = Book.objects.select_related('author').all()  # 1 query with JOIN
for book in books:
    print(book.author.name)  # No additional queries

# Generated SQL:
# SELECT book.*, author.* FROM book 
# INNER JOIN author ON book.author_id = author.id

# GOOD - prefetch_related (for ManyToMany, reverse ForeignKey)
authors = Author.objects.prefetch_related('books').all()
for author in authors:
    for book in author.books.all():  # No additional queries
        print(book.title)

# Uses 2 queries:
# 1. SELECT * FROM author
# 2. SELECT * FROM book WHERE author_id IN (1, 2, 3, ...)
```

**2. Only/Defer:**
```python
# Only fetch specific fields
books = Book.objects.only('title', 'price')  # Fetch only title and price

# Defer heavy fields
books = Book.objects.defer('description')  # Fetch all except description

# With relations
books = Book.objects.select_related('author').only(
    'title', 'price', 'author__name'
)
```

**3. Aggregation:**
```python
from django.db.models import Count, Avg, Sum, Max, Min, F, Q

# Count books per author
authors = Author.objects.annotate(
    book_count=Count('books'),
    avg_price=Avg('books__price')
).filter(book_count__gt=0)

for author in authors:
    print(f"{author.name}: {author.book_count} books, avg price: {author.avg_price}")

# Complex aggregation
stats = Book.objects.aggregate(
    total_books=Count('id'),
    avg_price=Avg('price'),
    max_price=Max('price'),
    min_price=Min('price'),
    total_revenue=Sum('price')
)

# Annotate with conditions
books = Book.objects.annotate(
    high_ratings_count=Count('reviews', filter=Q(reviews__rating__gte=4))
)
```

**4. F() expressions:**
```python
# Update using database-level operations
from django.db.models import F

# Atomic update
Book.objects.filter(id=1).update(price=F('price') * 1.1)  # Increase by 10%

# Comparison
expensive_books = Book.objects.filter(price__gt=F('author__avg_book_price'))

# Arithmetic
books = Book.objects.annotate(
    discount_price=F('price') * 0.9,
    days_since_published=Now() - F('published_date')
)
```

**5. Q objects (Complex queries):**
```python
from django.db.models import Q

# OR queries
books = Book.objects.filter(
    Q(title__icontains='python') | Q(title__icontains='django')
)

# Complex conditions
books = Book.objects.filter(
    Q(price__lt=30) & (Q(author__name='John') | Q(published_date__year=2023))
)

# NOT
books = Book.objects.filter(~Q(author__name='John'))
```

**6. Bulk Operations:**
```python
# Bulk create
books = [
    Book(title=f'Book {i}', author=author, price=10 + i)
    for i in range(1000)
]
Book.objects.bulk_create(books, batch_size=100)

# Bulk update
books = Book.objects.filter(published_date__year=2023)
for book in books:
    book.price *= 1.1
Book.objects.bulk_update(books, ['price'], batch_size=100)

# Update all at once
Book.objects.filter(published_date__year=2023).update(price=F('price') * 1.1)

# Bulk delete
Book.objects.filter(published_date__year__lt=2020).delete()
```

**7. Raw SQL & Database Functions:**
```python
from django.db.models.functions import Concat, Lower, Upper, Coalesce

# Database functions
authors = Author.objects.annotate(
    full_name=Concat('first_name', models.Value(' '), 'last_name'),
    email_lower=Lower('email')
)

# Raw SQL (when necessary)
books = Book.objects.raw('SELECT * FROM book WHERE price > %s', [50])

# Execute raw SQL
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("UPDATE book SET price = price * 1.1 WHERE author_id = %s", [author_id])
```

**8. Database Indexes:**
```python
class Book(models.Model):
    title = models.CharField(max_length=200, db_index=True)  # Simple index
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    published_date = models.DateField()
    
    class Meta:
        indexes = [
            # Composite index
            models.Index(fields=['author', '-published_date']),
            
            # Partial index (PostgreSQL)
            models.Index(
                fields=['title'],
                name='title_active_idx',
                condition=Q(is_active=True)
            ),
            
            # Expression index (PostgreSQL)
            models.Index(
                Lower('title').desc(),
                name='title_lower_idx'
            ),
        ]
```

**9. Query Performance Tips:**
```python
# Use exists() instead of count()
if Book.objects.filter(author=author).exists():  # Fast
    pass
# vs
if Book.objects.filter(author=author).count() > 0:  # Slower

# Use iterator() for large querysets
for book in Book.objects.iterator(chunk_size=1000):
    process_book(book)  # Memory efficient

# Use values() or values_list() for simple data
book_titles = Book.objects.values_list('title', flat=True)
# Returns: ['Book 1', 'Book 2', ...]

# Use explain() to analyze queries
print(Book.objects.filter(price__gt=50).explain())
```

**10. Caching:**
```python
from django.core.cache import cache
from django.views.decorators.cache import cache_page

# Query result caching
def get_books():
    cache_key = 'all_books'
    books = cache.get(cache_key)
    
    if books is None:
        books = list(Book.objects.select_related('author').all())
        cache.set(cache_key, books, timeout=3600)  # 1 hour
    
    return books

# View caching
@cache_page(60 * 15)  # Cache for 15 minutes
def book_list(request):
    books = Book.objects.all()
    return render(request, 'books/list.html', {'books': books})

# Template fragment caching
{% load cache %}
{% cache 3600 book_list %}
    <!-- Expensive template rendering -->
{% endcache %}
```

---

### Q3: Explain Django signals and when to use them.
**Answer:**

**What are Django Signals:**
Signals allow decoupled applications to get notified when actions occur elsewhere in the framework.

**Built-in Signals:**

```python
from django.db.models.signals import (
    pre_save, post_save,
    pre_delete, post_delete,
    m2m_changed
)
from django.contrib.auth.signals import (
    user_logged_in, user_logged_out, user_login_failed
)
from django.core.signals import request_started, request_finished
from django.dispatch import receiver

# models.py
class Profile(models.Model):
    user = models.OneToOneField('auth.User', on_delete=models.CASCADE)
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', null=True)
    created_at = models.DateTimeField(auto_now_add=True)

# signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth.models import User

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    """Automatically create profile when user is created"""
    if created:
        Profile.objects.create(user=instance)
        print(f"Profile created for {instance.username}")

@receiver(post_save, sender=User)
def save_user_profile(sender, instance, **kwargs):
    """Save profile when user is saved"""
    if hasattr(instance, 'profile'):
        instance.profile.save()

# Alternative: Connect without decorator
def user_logged_in_handler(sender, request, user, **kwargs):
    """Track user login"""
    LoginLog.objects.create(
        user=user,
        ip_address=request.META.get('REMOTE_ADDR'),
        user_agent=request.META.get('HTTP_USER_AGENT')
    )

user_logged_in.connect(user_logged_in_handler)
```

**Signal Types:**

**1. Model Signals:**
```python
from django.db.models.signals import pre_save, post_save, pre_delete, post_delete

class Article(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    content = models.TextField()
    status = models.CharField(max_length=20)

@receiver(pre_save, sender=Article)
def generate_slug(sender, instance, **kwargs):
    """Generate slug before saving"""
    if not instance.slug:
        from django.utils.text import slugify
        instance.slug = slugify(instance.title)

@receiver(post_save, sender=Article)
def notify_on_publish(sender, instance, created, **kwargs):
    """Send notification when article is published"""
    if not created and instance.status == 'published':
        # Check if status changed
        try:
            old_instance = Article.objects.get(pk=instance.pk)
            if old_instance.status != 'published':
                send_publication_notification(instance)
        except Article.DoesNotExist:
            pass

@receiver(pre_delete, sender=Article)
def backup_before_delete(sender, instance, **kwargs):
    """Backup article before deletion"""
    ArticleBackup.objects.create(
        original_id=instance.id,
        title=instance.title,
        content=instance.content,
        deleted_at=timezone.now()
    )

@receiver(post_delete, sender=Article)
def cleanup_files(sender, instance, **kwargs):
    """Delete associated files"""
    if instance.image:
        instance.image.delete(save=False)
```

**2. Many-to-Many Signals:**
```python
from django.db.models.signals import m2m_changed

class Course(models.Model):
    title = models.CharField(max_length=200)
    students = models.ManyToManyField('auth.User', related_name='courses')

@receiver(m2m_changed, sender=Course.students.through)
def notify_course_enrollment(sender, instance, action, pk_set, **kwargs):
    """Notify when students are added/removed"""
    if action == 'post_add':
        # Students added
        students = User.objects.filter(pk__in=pk_set)
        for student in students:
            send_enrollment_email(student, instance)
    
    elif action == 'post_remove':
        # Students removed
        students = User.objects.filter(pk__in=pk_set)
        for student in students:
            send_unenrollment_email(student, instance)
    
    elif action == 'pre_clear':
        # All students about to be removed
        print(f"Clearing all students from {instance.title}")
```

**3. Request/Response Signals:**
```python
from django.core.signals import request_started, request_finished

@receiver(request_started)
def log_request_started(sender, environ, **kwargs):
    """Log when request starts"""
    print(f"Request started: {environ.get('PATH_INFO')}")

@receiver(request_finished)
def log_request_finished(sender, **kwargs):
    """Log when request finishes"""
    print("Request finished")
```

**Custom Signals:**

```python
# signals.py
from django.dispatch import Signal

# Define custom signal
payment_completed = Signal()  # No providing_args in Django 4.0+

# models.py
class Order(models.Model):
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)

class Payment(models.Model):
    order = models.ForeignKey(Order, on_delete=models.CASCADE)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)

# views.py
from .signals import payment_completed

def process_payment(request):
    # ... payment processing ...
    
    if payment.status == 'completed':
        # Send signal
        payment_completed.send(
            sender=Payment,
            payment=payment,
            order=payment.order
        )
    
    return JsonResponse({'status': 'success'})

# handlers.py
from django.dispatch import receiver
from .signals import payment_completed

@receiver(payment_completed)
def update_order_status(sender, payment, order, **kwargs):
    """Update order status when payment completes"""
    order.status = 'paid'
    order.save()

@receiver(payment_completed)
def send_receipt(sender, payment, order, **kwargs):
    """Send receipt email"""
    send_email(
        to=order.user.email,
        subject='Payment Receipt',
        template='emails/receipt.html',
        context={'payment': payment, 'order': order}
    )

@receiver(payment_completed)
def update_inventory(sender, payment, order, **kwargs):
    """Update inventory after payment"""
    for item in order.items.all():
        item.product.stock -= item.quantity
        item.product.save()
```

**When to Use Signals:**

**Good Use Cases:**
1. Creating related objects (Profile when User created)
2. Logging and auditing
3. Cache invalidation
4. Sending notifications
5. Triggering background tasks
6. Keeping data in sync across apps

**When NOT to Use Signals:**
1. Simple operations that can be in save() method
2. When direct code is clearer
3. Heavy processing (use Celery instead)
4. When testing becomes difficult

**Better Alternatives:**

```python
# Instead of signal for simple operations
class User(models.Model):
    email = models.EmailField()
    
    def save(self, *args, **kwargs):
        # Direct operation in save()
        self.email = self.email.lower()
        super().save(*args, **kwargs)
        
        # Create profile if needed
        if not hasattr(self, 'profile'):
            Profile.objects.create(user=self)

# Instead of signal for heavy operations, use Celery
from celery import shared_task

@shared_task
def send_welcome_email(user_id):
    user = User.objects.get(id=user_id)
    # Send email (async)

# In view
def register_user(request):
    user = User.objects.create(...)
    send_welcome_email.delay(user.id)  # Background task
```

**Signal Best Practices:**

```python
# 1. Always disconnect in tests
from django.test import TestCase
from django.db.models.signals import post_save

class MyTestCase(TestCase):
    def setUp(self):
        post_save.disconnect(create_user_profile, sender=User)
    
    def tearDown(self):
        post_save.connect(create_user_profile, sender=User)

# 2. Use dispatch_uid to prevent duplicate signals
@receiver(post_save, sender=User, dispatch_uid='create_user_profile')
def create_user_profile(sender, instance, created, **kwargs):
    pass

# 3. Be careful with exceptions
@receiver(post_save, sender=User)
def safe_signal_handler(sender, instance, **kwargs):
    try:
        # Your code
        pass
    except Exception as e:
        logger.error(f"Signal failed: {e}")
        # Don't let signal errors break the save

# 4. Register signals in AppConfig
# apps.py
from django.apps import AppConfig

class MyAppConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'myapp'
    
    def ready(self):
        import myapp.signals  # Import signals
```

---

## 2. DJANGO REST FRAMEWORK (DRF)

### Q4: Explain DRF serializers and their types.
**Answer:**

**Serializer Types:**

**1. Basic Serializer:**
```python
from rest_framework import serializers

class ArticleSerializer(serializers.Serializer):
    """Manual field definition"""
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    author = serializers.CharField(max_length=100)
    published_date = serializers.DateTimeField()
    is_published = serializers.BooleanField(default=False)
    
    def create(self, validated_data):
        """Create new instance"""
        return Article.objects.create(**validated_data)
    
    def update(self, instance, validated_data):
        """Update existing instance"""
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.is_published = validated_data.get('is_published', instance.is_published)
        instance.save()
        return instance
```

**2. ModelSerializer (Most Common):**
```python
from rest_framework import serializers
from .models import Article, Author, Comment

class AuthorSerializer(serializers.ModelSerializer):
    """Automatically creates fields from model"""
    book_count = serializers.SerializerMethodField()
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'email', 'bio', 'book_count']
        read_only_fields = ['id']
    
    def get_book_count(self, obj):
        """Custom method field"""
        return obj.books.count()

class ArticleSerializer(serializers.ModelSerializer):
    # Custom fields
    author_name = serializers.CharField(source='author.name', read_only=True)
    comment_count = serializers.SerializerMethodField()
    
    # Nested serializer
    author = AuthorSerializer(read_only=True)
    author_id = serializers.IntegerField(write_only=True)
    
    class Meta:
        model = Article
        fields = [
            'id', 'title', 'content', 'author', 'author_id',
            'author_name', 'published_date', 'is_published',
            'comment_count', 'created_at', 'updated_at'
        ]
        read_only_fields = ['id', 'created_at', 'updated_at']
        extra_kwargs = {
            'content': {'write_only': True},  # Don't return in response
            'title': {'required': True, 'allow_blank': False}
        }
    
    def get_comment_count(self, obj):
        return obj.comments.count()
    
    def validate_title(self, value):
        """Field-level validation"""
        if len(value) < 5:
            raise serializers.ValidationError("Title must be at least 5 characters")
        return value
    
    def validate(self, data):
        """Object-level validation"""
        if data.get('is_published') and not data.get('content'):
            raise serializers.ValidationError("Published articles must have content")
        return data

class CommentSerializer(serializers.ModelSerializer):
    user_name = serializers.CharField(source='user.username', read_only=True)
    
    class Meta:
        model = Comment
        fields = ['id', 'article', 'user', 'user_name', 'text', 'created_at']
        read_only_fields = ['id', 'created_at', 'user']
```

**3. Nested Serializers:**
```python
class ArticleDetailSerializer(serializers.ModelSerializer):
    """Detailed article with nested comments"""
    author = AuthorSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'author', 'comments', 'published_date']

# Usage
article = Article.objects.prefetch_related('comments', 'comments__user').get(pk=1)
serializer = ArticleDetailSerializer(article)
print(serializer.data)
# Output:
# {
#     'id': 1,
#     'title': 'My Article',
#     'content': '...',
#     'author': {'id': 1, 'name': 'John Doe', ...},
#     'comments': [
#         {'id': 1, 'text': 'Great article!', ...},
#         {'id': 2, 'text': 'Thanks for sharing', ...}
#     ]
# }
```

**4. Writable Nested Serializers:**
```python
class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'isbn']

class AuthorWithBooksSerializer(serializers.ModelSerializer):
    books = BookSerializer(many=True)
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'email', 'books']
    
    def create(self, validated_data):
        """Handle nested creation"""
        books_data = validated_data.pop('books')
        author = Author.objects.create(**validated_data)
        
        for book_data in books_data:
            Book.objects.create(author=author, **book_data)
        
        return author
    
    def update(self, instance, validated_data):
        """Handle nested updates"""
        books_data = validated_data.pop('books', None)
        
        # Update author fields
        instance.name = validated_data.get('name', instance.name)
        instance.email = validated_data.get('email', instance.email)
        instance.save()
        
        # Update books
        if books_data is not None:
            # Simple approach: delete and recreate
            instance.books.all().delete()
            for book_data in books_data:
                Book.objects.create(author=instance, **book_data)
        
        return instance
```

**5. Dynamic Fields:**
```python
class DynamicFieldsSerializer(serializers.ModelSerializer):
    """Serializer that can exclude fields"""
    
    def __init__(self, *args, **kwargs):
        # Extract fields argument
        fields = kwargs.pop('fields', None)
        exclude = kwargs.pop('exclude', None)
        
        super().__init__(*args, **kwargs)
        
        if fields is not None:
            # Drop fields not in `fields`
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)
        
        if exclude is not None:
            # Drop fields in `exclude`
            for field_name in exclude:
                self.fields.pop(field_name, None)

class ArticleSerializer(DynamicFieldsSerializer):
    class Meta:
        model = Article
        fields = '__all__'

# Usage
# Only include specific fields
serializer = ArticleSerializer(article, fields=['id', 'title'])

# Exclude specific fields
serializer = ArticleSerializer(article, exclude=['content'])
```

**6. Context and Custom Methods:**
```python
class ArticleSerializer(serializers.ModelSerializer):
    is_owner = serializers.SerializerMethodField()
    can_edit = serializers.SerializerMethodField()
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'author', 'is_owner', 'can_edit']
    
    def get_is_owner(self, obj):
        """Check if current user is owner"""
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.author == request.user
        return False
    
    def get_can_edit(self, obj):
        """Check if user can edit"""
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.author == request.user or request.user.is_staff
        return False

# In view
serializer = ArticleSerializer(article, context={'request': request})
```

**7. Validation:**
```python
class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = '__all__'
    
    def validate_title(self, value):
        """Validate single field"""
        if 'badword' in value.lower():
            raise serializers.ValidationError("Title contains inappropriate content")
        return value
    
    def validate(self, data):
        """Validate multiple fields together"""
        if data.get('is_published'):
            if not data.get('content'):
                raise serializers.ValidationError("Published articles must have content")
            if not data.get('author'):
                raise serializers.ValidationError("Published articles must have an author")
        
        # Check uniqueness with custom logic
        if Article.objects.filter(
            title=data.get('title'),
            author=data.get('author')
        ).exclude(pk=self.instance.pk if self.instance else None).exists():
            raise serializers.ValidationError("Article with this title already exists for this author")
        
        return data
    
    def validate_published_date(self, value):
        """Validate date"""
        from django.utils import timezone
        if value > timezone.now():
            raise serializers.ValidationError("Published date cannot be in the future")
        return value
```

**8. Performance Optimization:**
```python
class OptimizedArticleSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'author', 'created_at']
    
    @classmethod
    def setup_eager_loading(cls, queryset):
        """Optimize queryset for serializer"""
        queryset = queryset.select_related('author')
        queryset = queryset.prefetch_related('comments')
        return queryset

# In viewset
class ArticleViewSet(viewsets.ModelViewSet):
    serializer_class = OptimizedArticleSerializer
    
    def get_queryset(self):
        queryset = Article.objects.all()
        # Apply eager loading
        queryset = self.get_serializer_class().setup_eager_loading(queryset)
        return queryset
```

---

### Q5: Explain DRF ViewS