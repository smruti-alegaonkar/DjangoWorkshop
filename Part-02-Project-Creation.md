# Part 2: Creating Your Django Project and First App

## Introduction

In this part, we'll create the Django project structure and understand how Django organizes code through projects and apps.

## Understanding Projects vs Apps

**Project**: Your entire website/application (the faculty leave management system)

**App**: A component within your project that does one specific thing (like handling leave requests, user authentication, etc.)

**Think of it like this:**
- **Project** = University
- **Apps** = Different departments (Admin, HR, Academics)

A project can contain multiple apps, and apps can be reused across different projects.

## Step 1: Create the Django Project

Make sure your virtual environment is activated (you should see `(venv)` in your terminal).

```bash
# Create the project
django-admin startproject leave_management_system .
```

**Note the dot (.) at the end!** This creates the project in the current directory instead of creating an extra nested folder.

### Project Structure Created

```
faculty-leave-management/
├── venv/
├── manage.py                      # Command-line tool
├── leave_management_system/       # Project configuration
│   ├── __init__.py
│   ├── settings.py                # Project settings
│   ├── urls.py                    # URL patterns
│   ├── asgi.py                    # Async server config
│   └── wsgi.py                    # Web server config
└── requirements.txt
```

### Understanding Key Files

**manage.py**
- Command-line utility to interact with your project
- Run server, create migrations, create superuser, etc.
- Never modify this file directly

**settings.py**
- All project configuration
- Database settings, installed apps, middleware, templates, etc.
- This is where you'll make most configuration changes

**urls.py**
- URL routing for the entire project
- Maps URLs to views (we'll understand this better later)

**__init__.py**
- Empty file that tells Python this directory is a Python package

## Step 2: Run the Development Server

Let's see if everything works:

```bash
python manage.py runserver
```

You should see output like:

```
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).

You have 18 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Run 'python manage.py migrate' to apply them.

February 13, 2026 - 10:30:00
Django version 5.0.x, using settings 'leave_management_system.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CTRL-BREAK.
```

### Visit Your Site

Open your browser and go to: `http://127.0.0.1:8000/` or `http://localhost:8000/`

You should see the Django welcome page! 🎉

**To stop the server**: Press `Ctrl+C` in the terminal

## Step 3: Apply Initial Migrations

Django comes with built-in apps (admin, auth, sessions, etc.) that need database tables. Let's create them:

```bash
python manage.py migrate
```

This creates a `db.sqlite3` file (your database) and sets up necessary tables.

**What just happened?**
- Django created tables for user authentication
- Created tables for the admin interface
- Set up session management tables
- Created content type tracking tables

## Step 4: Create a Superuser

The superuser is an admin account with full permissions:

```bash
python manage.py createsuperuser
```

You'll be prompted for:
- **Username**: (e.g., admin)
- **Email**: (e.g., admin@example.com) - optional, can press Enter
- **Password**: (won't be visible as you type)
- **Password confirmation**: (type the same password)

**Remember these credentials!** You'll need them to access the admin panel.

## Step 5: Access Django Admin

1. Start the server: `python manage.py runserver`
2. Go to: `http://127.0.0.1:8000/admin/`
3. Login with your superuser credentials

You'll see the Django administration interface with:
- Users
- Groups

This is Django's built-in admin panel - extremely powerful for managing your application!

## Step 6: Create Your First Django App

Now let's create an app for handling leave requests:

```bash
python manage.py startapp leaves
```

### App Structure Created

```
leaves/
├── migrations/           # Database migration files
│   └── __init__.py
├── __init__.py
├── admin.py             # Register models for admin panel
├── apps.py              # App configuration
├── models.py            # Database models
├── tests.py             # Unit tests
└── views.py             # View functions/classes
```

### Understanding App Files

**models.py**
- Define your database structure using Python classes
- Each model becomes a database table

**views.py**
- Handle HTTP requests and return responses
- Contains your business logic

**admin.py**
- Register models to appear in Django admin
- Customize admin interface behavior

**tests.py**
- Write automated tests for your app

**migrations/**
- Stores database migration files
- Django's way of version controlling your database schema

## Step 7: Register the App

To use your new app, you must register it in `settings.py`:

Open `leave_management_system/settings.py` and find `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'leaves',  # Add this line
]
```

**Important**: Add your app name to the list. The comma at the end is a Python convention for lists.

## Step 8: Create a Simple View

Let's create your first view. Open `leaves/views.py`:

```python
from django.shortcuts import render
from django.http import HttpResponse

# Create your views here.

def home(request):
    """
    Home page view - displays welcome message
    """
    return HttpResponse("<h1>Welcome to Faculty Leave Management System</h1>")

def about(request):
    """
    About page view
    """
    return HttpResponse("<h1>About Our Leave Management System</h1><p>This system helps faculty manage their leave requests efficiently.</p>")
```

### Understanding the View

- **request**: An object containing information about the current web request
- **HttpResponse**: Returns a simple HTTP response with content
- Each view function takes at least one parameter: `request`

## Step 9: Configure URLs

Now we need to map URLs to our views.

### Create App URLs

Create a new file `leaves/urls.py`:

```python
from django.urls import path
from . import views

# App namespace
app_name = 'leaves'

urlpatterns = [
    path('', views.home, name='home'),
    path('about/', views.about, name='about'),
]
```

**Explanation:**
- `path()`: Defines a URL pattern
- First argument: URL path (empty string '' means root)
- Second argument: Which view function to call
- `name=`: A name to reference this URL in templates

### Include App URLs in Project

Open `leave_management_system/urls.py`:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('leaves.urls')),  # Add this line
]
```

**What's happening?**
- We're including all URLs from the `leaves` app
- Empty string means these URLs start at the root of the site
- `include()` allows you to reference other URL configurations

## Step 10: Test Your Views

1. Start the server: `python manage.py runserver`
2. Visit:
   - `http://127.0.0.1:8000/` - Should show "Welcome to Faculty Leave Management System"
   - `http://127.0.0.1:8000/about/` - Should show the about page

Congratulations! You've created your first working Django views! 🎊

## Understanding the Request-Response Flow

Let's trace what happens when you visit `http://127.0.0.1:8000/`:

1. **Browser sends request** to Django server
2. **Django's URL dispatcher** looks at `leave_management_system/urls.py`
3. **Finds matching pattern** and includes `leaves.urls`
4. **Looks in leaves/urls.py** for matching path (empty string)
5. **Calls the view function** `home(request)`
6. **View returns** an HttpResponse
7. **Django sends response** back to browser

```
Browser → URL Dispatcher → View → Response → Browser
```

## Project Structure Now

```
faculty-leave-management/
├── venv/
├── db.sqlite3                      # Database file
├── manage.py
├── leave_management_system/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py                     # Modified
│   ├── asgi.py
│   └── wsgi.py
├── leaves/                         # New app
│   ├── migrations/
│   │   └── __init__.py
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── models.py
│   ├── tests.py
│   ├── urls.py                     # New file
│   └── views.py                    # Modified
└── requirements.txt
```

## Common Issues and Solutions

### Issue: "ModuleNotFoundError: No module named 'leaves'"
**Solution**: Make sure you added 'leaves' to INSTALLED_APPS in settings.py

### Issue: "Page not found (404)"
**Solution**: Check your URL patterns match exactly (including trailing slashes)

### Issue: Changes not appearing
**Solution**: 
1. Make sure you saved all files
2. Django auto-reloads, but sometimes you need to restart the server
3. Clear browser cache (Ctrl+Shift+R)

### Issue: "That port is already in use"
**Solution**: Either:
- Stop the other server running
- Use a different port: `python manage.py runserver 8080`

## Best Practices Learned

1. **Always use a virtual environment** - Keeps dependencies isolated
2. **One app per functionality** - Makes code organized and reusable
3. **Meaningful names** - Use descriptive names for apps, views, and URLs
4. **URL naming** - Always use the `name` parameter in path()
5. **Documentation** - Add docstrings to your views

## Exercise: Practice Tasks

Try these on your own:

1. Create a new view called `dashboard` that returns "Faculty Dashboard"
2. Add a URL pattern for `/dashboard/`
3. Create a view for `/contact/` with contact information
4. Add a view that shows the current date and time

<details>
<summary>Click to see solution</summary>

```python
# In leaves/views.py
from datetime import datetime

def dashboard(request):
    return HttpResponse("<h1>Faculty Dashboard</h1>")

def contact(request):
    return HttpResponse("<h1>Contact Us</h1><p>Email: support@university.edu</p>")

def current_time(request):
    now = datetime.now()
    return HttpResponse(f"<h1>Current time: {now}</h1>")

# In leaves/urls.py
urlpatterns = [
    path('', views.home, name='home'),
    path('about/', views.about, name='about'),
    path('dashboard/', views.dashboard, name='dashboard'),
    path('contact/', views.contact, name='contact'),
    path('time/', views.current_time, name='current_time'),
]
```
</details>

## Git Setup (Optional but Recommended)

Initialize Git for version control:

```bash
git init
```

Create a `.gitignore` file:

```
# Python
*.pyc
__pycache__/
venv/

# Django
*.log
db.sqlite3
media/

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db
```

Make your first commit:

```bash
git add .
git commit -m "Initial Django project setup with leaves app"
```

## Next Steps

In Part 3, we'll:
- Create our database models (Faculty, LeaveRequest)
- Understand Django ORM
- Work with the database
- Register models in the admin panel

## Checklist

Before moving to Part 3, ensure you:
- [ ] Created Django project successfully
- [ ] Created the 'leaves' app
- [ ] Added 'leaves' to INSTALLED_APPS
- [ ] Created and accessed superuser account
- [ ] Created basic views and URLs
- [ ] Can access home page and admin panel
- [ ] Understand the request-response flow

---

*Workshop Materials | Faculty Leave Management System | Part 2 of 10*
