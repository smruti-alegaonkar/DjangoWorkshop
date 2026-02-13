# Part 10: Testing, Best Practices, and Deployment

## Introduction

In this final part, we'll cover testing, security best practices, optimization, and deployment.

## Part A: Testing

### Step 1: Writing Unit Tests

Create comprehensive tests in `leaves/tests.py`:

```python
from django.test import TestCase, Client
from django.contrib.auth.models import User
from django.urls import reverse
from datetime import date, timedelta
from .models import FacultyProfile, LeaveType, LeaveRequest, LeaveBalance


class ModelTests(TestCase):
    """Test models"""
    
    def setUp(self):
        """Set up test data"""
        # Create user
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123',
            first_name='Test',
            last_name='User',
            email='test@example.com'
        )
        
        # Create faculty profile
        self.faculty = FacultyProfile.objects.create(
            user=self.user,
            employee_id='EMP001',
            department='Computer Science',
            designation='Assistant Professor',
            date_of_joining=date(2020, 1, 1)
        )
        
        # Create leave type
        self.leave_type = LeaveType.objects.create(
            name='Casual Leave',
            max_days_per_year=12,
            is_active=True
        )
        
        # Create leave balance
        self.leave_balance = LeaveBalance.objects.create(
            faculty=self.faculty,
            leave_type=self.leave_type,
            year=2026,
            total_leaves=12,
            used_leaves=0
        )
    
    def test_faculty_profile_creation(self):
        """Test faculty profile is created correctly"""
        self.assertEqual(self.faculty.employee_id, 'EMP001')
        self.assertEqual(self.faculty.user.username, 'testuser')
    
    def test_leave_balance_calculation(self):
        """Test remaining leaves calculation"""
        self.assertEqual(self.leave_balance.remaining_leaves(), 12)
        self.leave_balance.used_leaves = 3
        self.assertEqual(self.leave_balance.remaining_leaves(), 9)
    
    def test_leave_request_days_calculation(self):
        """Test leave request auto-calculates days"""
        leave_request = LeaveRequest.objects.create(
            faculty=self.faculty,
            leave_type=self.leave_type,
            start_date=date(2026, 3, 1),
            end_date=date(2026, 3, 5),
            reason='Test leave'
        )
        self.assertEqual(leave_request.number_of_days, 5)


class ViewTests(TestCase):
    """Test views"""
    
    def setUp(self):
        """Set up test client and data"""
        self.client = Client()
        
        # Create user and faculty
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123',
            email='test@example.com'
        )
        self.faculty = FacultyProfile.objects.create(
            user=self.user,
            employee_id='EMP001',
            department='CS',
            designation='Professor',
            date_of_joining=date(2020, 1, 1)
        )
        
        # Create leave type
        self.leave_type = LeaveType.objects.create(
            name='Casual Leave',
            max_days_per_year=12
        )
    
    def test_home_page(self):
        """Test home page loads"""
        response = self.client.get(reverse('leaves:home'))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, 'Leave Management')
    
    def test_dashboard_requires_login(self):
        """Test dashboard redirects if not logged in"""
        response = self.client.get(reverse('leaves:dashboard'))
        self.assertEqual(response.status_code, 302)  # Redirect
    
    def test_dashboard_after_login(self):
        """Test dashboard works after login"""
        self.client.login(username='testuser', password='testpass123')
        response = self.client.get(reverse('leaves:dashboard'))
        self.assertEqual(response.status_code, 200)
    
    def test_apply_leave_form(self):
        """Test leave application submission"""
        self.client.login(username='testuser', password='testpass123')
        response = self.client.post(reverse('leaves:apply_leave'), {
            'leave_type': self.leave_type.id,
            'start_date': '2026-03-01',
            'end_date': '2026-03-03',
            'reason': 'Personal work'
        })
        self.assertEqual(LeaveRequest.objects.count(), 1)


class FormTests(TestCase):
    """Test forms"""
    
    def setUp(self):
        self.leave_type = LeaveType.objects.create(
            name='Casual Leave',
            max_days_per_year=12
        )
    
    def test_valid_leave_request_form(self):
        """Test valid form data"""
        from .forms import LeaveRequestForm
        form_data = {
            'leave_type': self.leave_type.id,
            'start_date': date.today() + timedelta(days=1),
            'end_date': date.today() + timedelta(days=3),
            'reason': 'Personal work'
        }
        form = LeaveRequestForm(data=form_data)
        self.assertTrue(form.is_valid())
    
    def test_invalid_date_range(self):
        """Test form validation for invalid dates"""
        from .forms import LeaveRequestForm
        form_data = {
            'leave_type': self.leave_type.id,
            'start_date': date.today() + timedelta(days=5),
            'end_date': date.today() + timedelta(days=1),  # End before start
            'reason': 'Test'
        }
        form = LeaveRequestForm(data=form_data)
        self.assertFalse(form.is_valid())
```

### Step 2: Running Tests

```bash
# Run all tests
python manage.py test

# Run specific app tests
python manage.py test leaves

# Run specific test class
python manage.py test leaves.tests.ModelTests

# Run with verbosity
python manage.py test --verbosity=2

# Keep test database
python manage.py test --keepdb

# Run in parallel
python manage.py test --parallel
```

### Step 3: Test Coverage

```bash
# Install coverage
pip install coverage

# Run tests with coverage
coverage run --source='.' manage.py test leaves

# Generate report
coverage report

# Generate HTML report
coverage html
# Open htmlcov/index.html in browser
```

## Part B: Security Best Practices

### 1. Production Settings

Create `leave_management_system/settings_production.py`:

```python
from .settings import *

DEBUG = False

ALLOWED_HOSTS = ['yourdomain.com', 'www.yourdomain.com']

# Security settings
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'

# HTTPS
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Database (use PostgreSQL in production)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'leave_management',
        'USER': 'dbuser',
        'PASSWORD': 'dbpassword',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}

# Static files
STATIC_ROOT = BASE_DIR / 'staticfiles'

# Media files
MEDIA_ROOT = BASE_DIR / 'media'
```

### 2. Environment Variables

Install python-decouple:
```bash
pip install python-decouple
```

Create `.env` file (never commit this):
```
SECRET_KEY=your-secret-key-here
DEBUG=False
DATABASE_URL=postgresql://user:password@localhost/dbname
EMAIL_HOST_USER=your-email@gmail.com
EMAIL_HOST_PASSWORD=your-app-password
```

Update `settings.py`:
```python
from decouple import config

SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
```

### 3. Additional Security

```python
# Rate limiting (install django-ratelimit)
from django_ratelimit.decorators import ratelimit

@ratelimit(key='user', rate='5/m')
def my_view(request):
    pass

# Input sanitization
from django.utils.html import escape
safe_text = escape(user_input)

# SQL injection prevention (Django ORM does this automatically)
# NEVER do this:
# User.objects.raw(f"SELECT * FROM users WHERE id = {user_id}")

# Instead:
User.objects.filter(id=user_id)
```

## Part C: Performance Optimization

### 1. Database Optimization

```python
# Use select_related for ForeignKey
leaves = LeaveRequest.objects.select_related(
    'faculty', 
    'leave_type'
).all()

# Use prefetch_related for reverse ForeignKey
faculties = FacultyProfile.objects.prefetch_related(
    'leave_requests'
).all()

# Add database indexes
class LeaveRequest(models.Model):
    status = models.CharField(max_length=10, db_index=True)
    start_date = models.DateField(db_index=True)

# Use only() to fetch specific fields
leaves = LeaveRequest.objects.only('id', 'status', 'start_date')

# Use values() for dictionaries (lighter)
leave_data = LeaveRequest.objects.values('id', 'status')
```

### 2. Caching

```python
# Install Redis
# pip install django-redis

# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}

# Cache view
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # Cache for 15 minutes
def my_view(request):
    pass

# Cache template fragment
{% load cache %}
{% cache 500 sidebar %}
    ... expensive sidebar content ...
{% endcache %}
```

## Part D: Deployment

### Option 1: Deploy to Heroku

```bash
# Install Heroku CLI
# heroku login

# Create Procfile
echo "web: gunicorn leave_management_system.wsgi" > Procfile

# Install gunicorn
pip install gunicorn

# Create runtime.txt
echo "python-3.11.0" > runtime.txt

# Update requirements.txt
pip freeze > requirements.txt

# Initialize git
git init
git add .
git commit -m "Initial commit"

# Create Heroku app
heroku create your-app-name

# Set environment variables
heroku config:set SECRET_KEY='your-secret-key'
heroku config:set DEBUG=False

# Add PostgreSQL
heroku addons:create heroku-postgresql:hobby-dev

# Deploy
git push heroku main

# Run migrations
heroku run python manage.py migrate

# Create superuser
heroku run python manage.py createsuperuser

# Open app
heroku open
```

### Option 2: Deploy to VPS (DigitalOcean, AWS, etc.)

```bash
# On server, install requirements
sudo apt update
sudo apt install python3-pip python3-venv nginx

# Clone repository
git clone your-repo-url
cd your-repo

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Install Gunicorn
pip install gunicorn

# Collect static files
python manage.py collectstatic

# Run migrations
python manage.py migrate

# Create systemd service file
sudo nano /etc/systemd/system/leave_management.service

# Add:
[Unit]
Description=Leave Management System
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/path/to/project
Environment="PATH=/path/to/venv/bin"
ExecStart=/path/to/venv/bin/gunicorn --workers 3 --bind unix:/path/to/project.sock leave_management_system.wsgi:application

[Install]
WantedBy=multi-user.target

# Start service
sudo systemctl start leave_management
sudo systemctl enable leave_management

# Configure Nginx
sudo nano /etc/nginx/sites-available/leave_management

# Add:
server {
    listen 80;
    server_name yourdomain.com;

    location = /favicon.ico { access_log off; log_not_found off; }
    
    location /static/ {
        root /path/to/project;
    }
    
    location /media/ {
        root /path/to/project;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/path/to/project.sock;
    }
}

# Enable site
sudo ln -s /etc/nginx/sites-available/leave_management /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl restart nginx

# Setup SSL with Let's Encrypt
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com
```

## Part E: Maintenance and Monitoring

### 1. Logging

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'file': {
            'level': 'ERROR',
            'class': 'logging.FileHandler',
            'filename': BASE_DIR / 'error.log',
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

# In views
import logging
logger = logging.getLogger(__name__)

def my_view(request):
    try:
        # code
        pass
    except Exception as e:
        logger.error(f'Error in my_view: {str(e)}')
```

### 2. Backup Strategy

```bash
# Backup database
python manage.py dumpdata > backup.json

# Backup specific app
python manage.py dumpdata leaves > leaves_backup.json

# Restore
python manage.py loaddata backup.json

# Automated backups (cron job)
0 2 * * * cd /path/to/project && /path/to/venv/bin/python manage.py dumpdata > /backups/backup_$(date +\%Y\%m\%d).json
```

## Final Checklist

### Before Deployment:
- [ ] All tests passing
- [ ] DEBUG = False
- [ ] SECRET_KEY in environment variable
- [ ] ALLOWED_HOSTS configured
- [ ] Database changed from SQLite to PostgreSQL/MySQL
- [ ] Static files collected
- [ ] Media files storage configured
- [ ] Email backend configured
- [ ] SSL certificate installed
- [ ] Security headers configured
- [ ] Error logging enabled
- [ ] Backup strategy in place

### After Deployment:
- [ ] Test all features in production
- [ ] Monitor error logs
- [ ] Set up automated backups
- [ ] Monitor performance
- [ ] Set up domain and SSL
- [ ] Create documentation

## Conclusion

Congratulations! You've built a complete Faculty Leave Management System with Django! 🎉

### What You've Learned:
1. Django project structure and MVT pattern
2. Models, migrations, and ORM
3. Forms and validation
4. User authentication and permissions
5. Templates and frontend design
6. Admin interface customization
7. Email notifications and signals
8. Reports and data export
9. Testing and security
10. Deployment strategies

### Next Steps:
- Add API with Django REST Framework
- Implement real-time notifications with WebSockets
- Add calendar integration
- Create mobile app
- Add advanced analytics with AI/ML
- Implement multi-tenant architecture

## Resources

- **Django Documentation**: https://docs.djangoproject.com/
- **Django REST Framework**: https://www.django-rest-framework.org/
- **Django Packages**: https://djangopackages.org/
- **Real Python Django**: https://realpython.com/tutorials/django/
- **Django Tutorial**: https://www.djangoproject.com/start/

## Thank You!

Thank you for completing this workshop! Keep building amazing things with Django! 🚀

---

*Workshop Materials | Faculty Leave Management System | Part 10 of 10 - COMPLETE*
