# Django Senior Backend Developer Interview Preparation Guide
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