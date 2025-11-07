# Django ORM Query Cheatsheet

## Basic Queries

### Retrieving Objects
```python
# Get all objects
Model.objects.all()

# Get single object (raises DoesNotExist if not found)
Model.objects.get(id=1)

# Get first/last object
Model.objects.first()
Model.objects.last()

# Get or create
obj, created = Model.objects.get_or_create(name="John", defaults={'age': 30})

# Update or create
obj, created = Model.objects.update_or_create(name="John", defaults={'age': 31})
```

## Filtering

### Basic Filtering
```python
# Filter objects
Model.objects.filter(name="John")
Model.objects.filter(age__gte=18)  # Greater than or equal

# Exclude objects
Model.objects.exclude(name="John")

# Chain filters
Model.objects.filter(name="John").filter(age__gte=18)
Model.objects.filter(name="John", age__gte=18)  # Same as above
```

### Field Lookups
```python
# Exact match
Model.objects.filter(name__exact="John")
Model.objects.filter(name="John")  # Same as exact

# Case-insensitive exact match
Model.objects.filter(name__iexact="john")

# Contains
Model.objects.filter(name__contains="oh")
Model.objects.filter(name__icontains="OH")  # Case-insensitive

# Starts with / Ends with
Model.objects.filter(name__startswith="J")
Model.objects.filter(name__istartswith="j")  # Case-insensitive
Model.objects.filter(name__endswith="n")
Model.objects.filter(name__iendswith="N")  # Case-insensitive

# In
Model.objects.filter(id__in=[1, 2, 3])
Model.objects.filter(name__in=['John', 'Jane'])

# Greater than / Less than
Model.objects.filter(age__gt=18)   # Greater than
Model.objects.filter(age__gte=18)  # Greater than or equal
Model.objects.filter(age__lt=65)   # Less than
Model.objects.filter(age__lte=65)  # Less than or equal

# Range
Model.objects.filter(age__range=(18, 65))

# Null checks
Model.objects.filter(name__isnull=True)
Model.objects.filter(name__isnull=False)

# Regex
Model.objects.filter(name__regex=r'^J')
Model.objects.filter(name__iregex=r'^j')  # Case-insensitive
```

### Date/Time Lookups
```python
# Date
Model.objects.filter(created_at__date=date(2025, 1, 1))
Model.objects.filter(created_at__year=2025)
Model.objects.filter(created_at__month=1)
Model.objects.filter(created_at__day=1)
Model.objects.filter(created_at__week=1)
Model.objects.filter(created_at__week_day=2)  # 1=Sunday, 7=Saturday

# Time
Model.objects.filter(created_at__time=time(14, 30))
Model.objects.filter(created_at__hour=14)
Model.objects.filter(created_at__minute=30)
Model.objects.filter(created_at__second=0)
```

## Complex Queries

### Q Objects (OR, NOT)
```python
from django.db.models import Q

# OR
Model.objects.filter(Q(name="John") | Q(name="Jane"))

# AND (default behavior)
Model.objects.filter(Q(name="John") & Q(age=30))

# NOT
Model.objects.filter(~Q(name="John"))

# Complex queries
Model.objects.filter(
    Q(name="John") | Q(name="Jane"),
    age__gte=18
)
```

### F Objects (Field References)
```python
from django.db.models import F

# Compare fields
Model.objects.filter(start_date__lt=F('end_date'))

# Update with field reference
Model.objects.update(views=F('views') + 1)

# Arithmetic operations
Model.objects.filter(discounted_price__lt=F('price') * 0.9)
```

## Ordering

```python
# Order by field (ascending)
Model.objects.order_by('name')

# Order by field (descending)
Model.objects.order_by('-name')

# Multiple fields
Model.objects.order_by('name', '-age')

# Random order
Model.objects.order_by('?')

# Reverse order
Model.objects.order_by('name').reverse()
```

## Limiting Results

```python
# Limit to first 5
Model.objects.all()[:5]

# Offset and limit (skip 5, get next 5)
Model.objects.all()[5:10]

# Get specific index
Model.objects.all()[0]  # First object
```

## Aggregation

```python
from django.db.models import Count, Sum, Avg, Max, Min

# Count
Model.objects.count()
Model.objects.filter(age__gte=18).count()

# Aggregate
Model.objects.aggregate(Avg('age'))
Model.objects.aggregate(total=Sum('price'), average=Avg('price'))

# Annotate (add calculated field to each object)
Model.objects.annotate(num_items=Count('items'))
Model.objects.annotate(total_price=Sum('items__price'))
```

## Grouping

```python
# Group by with values
Model.objects.values('category').annotate(count=Count('id'))

# Group by multiple fields
Model.objects.values('category', 'status').annotate(count=Count('id'))

# Values list (returns tuples)
Model.objects.values_list('name', 'age')
Model.objects.values_list('name', flat=True)  # Returns flat list
```

## Relationships

### Foreign Key
```python
# Forward relationship
Model.objects.filter(foreign_key__field="value")
Model.objects.select_related('foreign_key')  # Eager loading

# Reverse relationship
ParentModel.objects.filter(model__field="value")
ParentModel.objects.prefetch_related('model_set')  # Eager loading
```

### Many-to-Many
```python
# Filter by related objects
Model.objects.filter(tags__name="django")

# Prefetch related
Model.objects.prefetch_related('tags')

# Add/Remove relations
obj.tags.add(tag1, tag2)
obj.tags.remove(tag1)
obj.tags.clear()
obj.tags.set([tag1, tag2])
```

## Updating

```python
# Update single object
obj = Model.objects.get(id=1)
obj.name = "New Name"
obj.save()

# Update multiple objects
Model.objects.filter(category="old").update(category="new")

# Update with F object
Model.objects.update(views=F('views') + 1)

# Bulk update
objs = Model.objects.filter(category="tech")
for obj in objs:
    obj.views += 1
Model.objects.bulk_update(objs, ['views'])
```

## Deleting

```python
# Delete single object
obj = Model.objects.get(id=1)
obj.delete()

# Delete multiple objects
Model.objects.filter(status="inactive").delete()

# Delete all
Model.objects.all().delete()
```

## Creating

```python
# Create single object
obj = Model.objects.create(name="John", age=30)

# Create without saving
obj = Model(name="John", age=30)
obj.save()

# Bulk create
Model.objects.bulk_create([
    Model(name="John", age=30),
    Model(name="Jane", age=25),
])
```

## Raw SQL

```python
# Raw queries
Model.objects.raw('SELECT * FROM myapp_model WHERE age > %s', [18])

# Execute custom SQL
from django.db import connection
with connection.cursor() as cursor:
    cursor.execute("SELECT * FROM myapp_model WHERE age > %s", [18])
    rows = cursor.fetchall()
```

## Query Optimization

```python
# Select related (for foreign keys - SQL JOIN)
Model.objects.select_related('foreign_key')
Model.objects.select_related('fk1', 'fk2')
Model.objects.select_related('fk1__nested_fk')

# Prefetch related (for reverse FK and M2M - separate queries)
Model.objects.prefetch_related('many_to_many')
Model.objects.prefetch_related('reverse_fk_set')

# Only specific fields
Model.objects.only('name', 'age')

# Defer fields
Model.objects.defer('large_text_field')

# Exists (efficient check)
Model.objects.filter(name="John").exists()

# Distinct
Model.objects.values('category').distinct()
```

## Transactions

```python
from django.db import transaction

# Atomic decorator
@transaction.atomic
def my_view(request):
    Model.objects.create(name="John")
    Model.objects.create(name="Jane")

# Atomic context manager
with transaction.atomic():
    Model.objects.create(name="John")
    Model.objects.create(name="Jane")
```

## Subqueries

```python
from django.db.models import Subquery, OuterRef

# Subquery
newest = Comment.objects.filter(
    post=OuterRef('pk')
).order_by('-created_at')

Post.objects.annotate(
    newest_comment_id=Subquery(newest.values('id')[:1])
)
```

## Conditional Expressions

```python
from django.db.models import Case, When, Value, IntegerField

# Case/When
Model.objects.annotate(
    discount=Case(
        When(age__lt=18, then=Value(10)),
        When(age__gte=65, then=Value(20)),
        default=Value(0),
        output_field=IntegerField(),
    )
)
```

## Database Functions

```python
from django.db.models.functions import Lower, Upper, Length, Concat

# String functions
Model.objects.annotate(lower_name=Lower('name'))
Model.objects.filter(name__length__gt=5)

# Date functions
from django.db.models.functions import ExtractYear, TruncDate
Model.objects.annotate(year=ExtractYear('created_at'))

# Concat
Model.objects.annotate(
    full_name=Concat('first_name', Value(' '), 'last_name')
)
```