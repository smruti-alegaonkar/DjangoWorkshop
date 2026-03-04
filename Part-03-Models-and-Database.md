# Part 3: Models and Database Design

## Introduction

In this part, we'll design our database structure using Django's Object-Relational Mapping (ORM). We'll create models for our Leave Management System.

## What are Django Models?

**Models** are Python classes that represent database tables. Each attribute of the model represents a database field.

### Why Use an ORM?

Instead of writing SQL:
```sql
CREATE TABLE faculty (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);
```

You write Python:
```python
class Faculty(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField()
```

**Benefits:**
- Write Python instead of SQL
- Database-agnostic (works with SQLite, PostgreSQL, MySQL)
- Automatic primary keys
- Built-in validation
- Easy relationships between tables

## Planning Our Database Structure

For our Leave Management System, we need:

1. **Faculty Profile** - Store additional faculty information
2. **Leave Type** - Different types of leaves (Casual, Sick, Earned)
3. **Leave Request** - Individual leave applications
4. **Leave Balance** - Track remaining leaves for each faculty

### Entity Relationship

```
User (Django built-in)
  ↓
FacultyProfile (extends User)
  ↓
LeaveBalance → LeaveType
  ↓
LeaveRequest → LeaveType
```

## Step 1: Understanding Django's User Model

Django provides a built-in User model with:
- Username
- Password (automatically hashed)
- Email
- First name / Last name
- Is staff / Is superuser
- Date joined

We'll extend this with additional faculty-specific information.

## Step 2: Create the Models

Open `leaves/models.py` and replace its contents:

```python
from django.db import models
from django.contrib.auth.models import User
from django.core.validators import MinValueValidator
from django.utils import timezone
from decimal import Decimal


class FacultyProfile(models.Model):
    """
    Extended profile for faculty members
    Linked to Django's built-in User model
    """
    user = models.OneToOneField(
    User,
    on_delete=models.CASCADE,
    related_name='faculty_profile'
    )
    employee_id = models.CharField(max_length=20, unique=True)
    department = models.CharField(max_length=100)
    designation = models.CharField(max_length=100)
    phone_number = models.CharField(max_length=15, blank=True)
    date_of_joining = models.DateField()
    profile_picture = models.ImageField(upload_to='profile_pics/', blank=True, null=True)
    email_notifications = models.BooleanField(default=True, help_text="Receive email notifications")

    def __str__(self):
        return f"{self.user.get_full_name()} ({self.employee_id})"
    
    class Meta:
        verbose_name = "Faculty Profile"
        verbose_name_plural = "Faculty Profiles"
        ordering = ['user__first_name']


class LeaveType(models.Model):
    """
    Different types of leaves available
    """
    name = models.CharField(max_length=50, unique=True)
    description = models.TextField(blank=True)
    max_days_per_year = models.IntegerField(validators=[MinValueValidator(1)])
    requires_document = models.BooleanField(default=False)
    is_active = models.BooleanField(default=True)
    
    def __str__(self):
        return self.name
    
    class Meta:
        verbose_name = "Leave Type"
        verbose_name_plural = "Leave Types"
        ordering = ['name']


class LeaveBalance(models.Model):
    """
    Track leave balance for each faculty member
    """
    faculty = models.ForeignKey(FacultyProfile, on_delete=models.CASCADE, related_name='leave_balances')
    leave_type = models.ForeignKey(LeaveType, on_delete=models.CASCADE)
    year = models.IntegerField()
    total_leaves = models.IntegerField(validators=[MinValueValidator(0)])
    used_leaves = models.DecimalField(max_digits=4, decimal_places=1, default=0)
    
    def remaining_leaves(self):
        """Calculate remaining leaves"""
        return self.total_leaves - float(self.used_leaves)
    
    def __str__(self):
        return f"{self.faculty.user.username} - {self.leave_type.name} ({self.year})"
    
    class Meta:
        verbose_name = "Leave Balance"
        verbose_name_plural = "Leave Balances"
        unique_together = ['faculty', 'leave_type', 'year']
        ordering = ['-year', 'leave_type']


class LeaveRequest(models.Model):
    """
    Individual leave request submitted by faculty
    """
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('approved', 'Approved'),
        ('rejected', 'Rejected'),
        ('cancelled', 'Cancelled'),
    ]
    
    faculty = models.ForeignKey(FacultyProfile, on_delete=models.CASCADE, related_name='leave_requests')
    leave_type = models.ForeignKey(LeaveType, on_delete=models.PROTECT)
    start_date = models.DateField()
    end_date = models.DateField()
    number_of_days = models.DecimalField(max_digits=4, decimal_places=1)
    reason = models.TextField()
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default='pending')
    applied_on = models.DateTimeField(auto_now_add=True)
    reviewed_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True, blank=True, related_name='reviewed_leaves')
    reviewed_on = models.DateTimeField(null=True, blank=True)
    admin_remarks = models.TextField(blank=True)
    supporting_document = models.FileField(upload_to='leave_documents/', blank=True, null=True)
    
    def __str__(self):
        return f"{self.faculty.user.username} - {self.leave_type.name} ({self.start_date} to {self.end_date})"
    
    def save(self, *args, **kwargs):
        """Override save to calculate number of days"""
        if self.start_date and self.end_date:
            delta = self.end_date - self.start_date
            self.number_of_days = Decimal(delta.days + 1)
        super().save(*args, **kwargs)
        
    class Meta:
        verbose_name = "Leave Request"
        verbose_name_plural = "Leave Requests"
        ordering = ['-applied_on']
        get_latest_by = 'applied_on'
```

## Understanding the Models

### Field Types Used

**CharField**: Short text (has max_length)
```python
name = models.CharField(max_length=50)
```

**TextField**: Long text (no max length needed)
```python
reason = models.TextField()
```

**EmailField**: Validates email format
```python
email = models.EmailField()
```

**IntegerField**: Stores integers
```python
year = models.IntegerField()
```

**DecimalField**: For precise decimals (like 0.5 days leave)
```python
used_leaves = models.DecimalField(max_digits=4, decimal_places=1)
```

**DateField**: Stores dates only
```python
start_date = models.DateField()
```

**DateTimeField**: Stores date and time
```python
applied_on = models.DateTimeField(auto_now_add=True)
```

**BooleanField**: True/False values
```python
is_active = models.BooleanField(default=True)
```

**FileField/ImageField**: For file uploads
```python
profile_picture = models.ImageField(upload_to='profile_pics/')
```

### Relationship Fields

**OneToOneField**: 1-to-1 relationship
```python
user = models.OneToOneField(User, on_delete=models.CASCADE)
```
*Each faculty profile links to exactly one user*

**ForeignKey**: Many-to-1 relationship
```python
faculty = models.ForeignKey(FacultyProfile, on_delete=models.CASCADE)
```
*Many leave requests can belong to one faculty*

**on_delete options:**
- `CASCADE`: Delete related objects too
- `PROTECT`: Prevent deletion if related objects exist
- `SET_NULL`: Set to null when deleted

### Field Options

**blank=True**: Field can be empty in forms
**null=True**: Field can be NULL in database
**unique=True**: Value must be unique
**default**: Default value if not provided
**choices**: Predefined options
**validators**: Custom validation rules
**auto_now_add=True**: Set to current time on creation
**auto_now=True**: Update to current time on every save

### Special Methods

**__str__(self)**: String representation (what shows in admin)
```python
def __str__(self):
    return self.name
```

**save()**: Override to add custom logic before saving
```python
def save(self, *args, **kwargs):
    # Custom logic here
    super().save(*args, **kwargs)
```

**Custom methods**: Add your own methods
```python
def remaining_leaves(self):
    return self.total_leaves - float(self.used_leaves)
```

### Meta Class Options

```python
class Meta:
    verbose_name = "Leave Request"  # Singular name
    verbose_name_plural = "Leave Requests"  # Plural name
    ordering = ['-applied_on']  # Default ordering
    unique_together = ['faculty', 'leave_type', 'year']  # Composite unique constraint
    get_latest_by = 'applied_on'  # Field to use for latest()
```

## Step 3: Create Migrations

Migrations are Django's way of propagating model changes to the database.

```bash
python manage.py makemigrations
```

You should see:

```
Migrations for 'leaves':
  leaves/migrations/0001_initial.py
    - Create model FacultyProfile
    - Create model LeaveType
    - Create model LeaveBalance
    - Create model LeaveRequest
```

**What just happened?**
Django created a migration file that contains the instructions to create these tables in the database.

### View the Migration

Look at `leaves/migrations/0001_initial.py` - it contains the SQL-like operations Django will perform.

You can see the SQL that will be executed:
```bash
python manage.py sqlmigrate leaves 0001
```

## Step 4: Apply Migrations

```bash
python manage.py migrate
```

This creates the actual database tables.

```
Running migrations:
  Applying leaves.0001_initial... OK
```

Your database now has tables for all your models!

## Step 5: Register Models in Admin

To manage these models through Django admin, register them in `leaves/admin.py`:

```python
from django.contrib import admin
from .models import FacultyProfile, LeaveType, LeaveBalance, LeaveRequest

@admin.register(FacultyProfile)
class FacultyProfileAdmin(admin.ModelAdmin):
    list_display = ['employee_id', 'get_full_name', 'department', 'designation', 'date_of_joining']
    list_filter = ['department', 'designation', 'date_of_joining']
    search_fields = ['employee_id', 'user__first_name', 'user__last_name', 'user__email']
    
    def get_full_name(self, obj):
        return obj.user.get_full_name()
    get_full_name.short_description = 'Full Name'


@admin.register(LeaveType)
class LeaveTypeAdmin(admin.ModelAdmin):
    list_display = ['name', 'max_days_per_year', 'requires_document', 'is_active']
    list_filter = ['is_active', 'requires_document']
    search_fields = ['name', 'description']


@admin.register(LeaveBalance)
class LeaveBalanceAdmin(admin.ModelAdmin):
    list_display = ['faculty', 'leave_type', 'year', 'total_leaves', 'used_leaves', 'get_remaining']
    list_filter = ['year', 'leave_type']
    search_fields = ['faculty__user__first_name', 'faculty__user__last_name', 'faculty__employee_id']
    
    def get_remaining(self, obj):
        return obj.remaining_leaves()
    get_remaining.short_description = 'Remaining Leaves'


@admin.register(LeaveRequest)
class LeaveRequestAdmin(admin.ModelAdmin):
    list_display = ['faculty', 'leave_type', 'start_date', 'end_date', 'number_of_days', 'status', 'applied_on']
    list_filter = ['status', 'leave_type', 'applied_on', 'start_date']
    search_fields = ['faculty__user__first_name', 'faculty__user__last_name', 'reason']
    readonly_fields = ['applied_on', 'number_of_days']
    date_hierarchy = 'applied_on'
    
    fieldsets = (
        ('Leave Information', {
            'fields': ('faculty', 'leave_type', 'start_date', 'end_date', 'number_of_days', 'reason')
        }),
        ('Status', {
            'fields': ('status', 'reviewed_by', 'reviewed_on', 'admin_remarks')
        }),
        ('Documents', {
            'fields': ('supporting_document',),
            'classes': ('collapse',)
        }),
    )
```

### Admin Customization Explained

**list_display**: Columns to show in the list view
**list_filter**: Add sidebar filters
**search_fields**: Fields to search
**readonly_fields**: Cannot be edited
**date_hierarchy**: Date-based drill-down navigation
**fieldsets**: Group fields in the edit form

## Step 6: Test in Admin Panel

1. Start the server: `python manage.py runserver`
2. Go to: `http://127.0.0.1:8000/admin/`
3. Login with your superuser credentials

You should now see:
- Faculty Profiles
- Leave Types
- Leave Balances
- Leave Requests

### Add Test Data

**Create Leave Types:**
1. Click "Leave Types" → "Add Leave Type"
2. Add these types:
   - Name: Casual Leave, Max days: 12
   - Name: Sick Leave, Max days: 10
   - Name: Earned Leave, Max days: 30

**Create a Faculty Profile:**
1. First create a user: Users → Add user
   - Username: john.doe
   - Password: (set a password)
   - First name: John, Last name: Doe
   
2. Then create profile: Faculty Profiles → Add Faculty Profile
   - User: john.doe
   - Employee ID: EMP001
   - Department: Computer Science
   - Designation: Assistant Professor
   - Date of joining: (select a date)

## Step 7: Django Shell - Querying the Database

The Django shell lets you interact with your database using Python:

```bash
python manage.py shell
```

### Basic Queries

```python
# Import models
from leaves.models import FacultyProfile, LeaveType, LeaveRequest
from django.contrib.auth.models import User

# Get all leave types
leave_types = LeaveType.objects.all()
print(leave_types)

# Get a specific leave type
casual = LeaveType.objects.get(name='Casual Leave')
print(casual.max_days_per_year)

# Filter leave types
active_types = LeaveType.objects.filter(is_active=True)

# Get all faculty profiles
faculties = FacultyProfile.objects.all()

# Get a specific faculty
faculty = FacultyProfile.objects.get(employee_id='EMP001')
print(faculty.user.email)

# Create a leave request
from datetime import date
leave_request = LeaveRequest.objects.create(
    faculty=faculty,
    leave_type=casual,
    start_date=date(2026, 3, 1),
    end_date=date(2026, 3, 3),
    reason="Personal work"
)

# Query leave requests
pending_leaves = LeaveRequest.objects.filter(status='pending')
approved_leaves = LeaveRequest.objects.filter(status='approved')

# Count records
total_requests = LeaveRequest.objects.count()
print(f"Total leave requests: {total_requests}")

# Order results
recent_requests = LeaveRequest.objects.order_by('-applied_on')[:5]

# Exit shell
exit()
```

### Common Query Methods

```python
# Get all objects
Model.objects.all()

# Get one object (raises error if not found or multiple found)
Model.objects.get(field=value)

# Filter objects (returns QuerySet)
Model.objects.filter(field=value)

# Exclude objects
Model.objects.exclude(field=value)

# Count
Model.objects.count()

# Order
Model.objects.order_by('field')  # Ascending
Model.objects.order_by('-field')  # Descending

# Limit results
Model.objects.all()[:5]  # First 5

# Check if exists
Model.objects.filter(field=value).exists()

# Create object
Model.objects.create(field=value, ...)

# Update object
obj = Model.objects.get(id=1)
obj.field = new_value
obj.save()

# Delete object
obj = Model.objects.get(id=1)
obj.delete()
```

## Understanding Relationships in Queries

```python
# Access related objects (ForeignKey)
leave_request = LeaveRequest.objects.get(id=1)
print(leave_request.faculty.user.first_name)
print(leave_request.leave_type.name)

# Reverse relationship (related_name)
faculty = FacultyProfile.objects.get(employee_id='EMP001')
all_leaves = faculty.leave_requests.all()
pending = faculty.leave_requests.filter(status='pending')

# Spanning relationships with __
requests = LeaveRequest.objects.filter(faculty__department='Computer Science')
requests = LeaveRequest.objects.filter(faculty__user__first_name='John')
```

## Common Model Patterns

### Automatic Timestamps

```python
class MyModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

### Soft Delete (don't actually delete)

```python
class MyModel(models.Model):
    is_deleted = models.BooleanField(default=False)
    
    def delete(self):
        self.is_deleted = True
        self.save()
```

### Slug Fields (URL-friendly names)

```python
from django.utils.text import slugify

class MyModel(models.Model):
    name = models.CharField(max_length=100)
    slug = models.SlugField(unique=True)
    
    def save(self, *args, **kwargs):
        self.slug = slugify(self.name)
        super().save(*args, **kwargs)
```

## Best Practices

1. **Always use related_name** for ForeignKey relationships
2. **Add __str__ methods** to all models for better readability
3. **Use validators** for data validation
4. **Index frequently queried fields** with `db_index=True`
5. **Use choices** for fields with limited options
6. **Document your models** with docstrings
7. **Override save()** for custom logic
8. **Use Meta class** for model-level configuration

## Common Issues and Solutions

### Issue: "No migrations to apply"
**Solution**: You haven't made any model changes. Migrations only needed when models change.

### Issue: "You are trying to add a non-nullable field..."
**Solution**: Either add `null=True` or provide a `default` value.

### Issue: "Duplicate entry" error
**Solution**: Check your `unique=True` or `unique_together` constraints.

### Issue: Images not uploading
**Solution**: Make sure you installed `pillow`: `pip install pillow`

## Exercise: Practice Tasks

1. Add a `created_at` field to LeaveRequest model
2. Create a model for LeaveComment (for discussion on leave requests)
3. Add a method to LeaveRequest to check if leave is in the past
4. Create 3 leave types and 2 faculty profiles via admin
5. Practice shell queries to fetch pending leaves

<details>
<summary>Click to see LeaveComment model solution</summary>

```python
class LeaveComment(models.Model):
    leave_request = models.ForeignKey(LeaveRequest, on_delete=models.CASCADE, related_name='comments')
    commented_by = models.ForeignKey(User, on_delete=models.CASCADE)
    comment = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return f"Comment on {self.leave_request} by {self.commented_by}"
    
    class Meta:
        ordering = ['created_at']
```
</details>

## Next Steps

In Part 4, we'll:
- Create HTML templates
- Use Django Template Language
- Design the frontend interface
- Display data from our models

## Checklist

Before moving to Part 4, ensure you:
- [ ] Created all four models
- [ ] Understand field types and relationships
- [ ] Made and applied migrations successfully
- [ ] Registered models in admin
- [ ] Can add data through admin panel
- [ ] Can query data using Django shell
- [ ] Added some test data

---

*Workshop Materials | Faculty Leave Management System | Part 3 of 10*
