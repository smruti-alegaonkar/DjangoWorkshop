# Part 1: Introduction to Django and Environment Setup

## Workshop Overview

Welcome to the Django Workshop! In this hands-on tutorial series, we'll build a **Faculty Leave Management System** - a real-world application that allows faculty members to submit leave requests and administrators to manage them.

## What You'll Build

By the end of this workshop, you'll have created a complete web application with:
- Faculty user registration and authentication
- Leave request submission form
- Leave history tracking
- Admin approval/rejection workflow
- Dashboard for both faculty and administrators
- Leave balance management

## What is Django?

Django is a high-level Python web framework that encourages rapid development and clean, pragmatic design. It follows the **MVT (Model-View-Template)** architectural pattern.

### MVT Pattern Explained

- **Model**: Represents your data structure (database tables)
- **View**: Contains the business logic (handles requests and returns responses)
- **Template**: Handles the presentation layer (HTML with Django template language)

### Why Django?

- **Batteries Included**: Comes with authentication, admin panel, ORM, and more
- **Secure**: Built-in protection against SQL injection, XSS, CSRF
- **Scalable**: Used by Instagram, Pinterest, Mozilla, and NASA
- **Great Documentation**: Comprehensive and beginner-friendly
- **Active Community**: Large ecosystem of packages and support

## Prerequisites

Before starting this workshop, you should have:
- Basic Python knowledge (variables, functions, classes)
- Understanding of HTML/CSS basics
- Familiarity with command line/terminal
- Basic understanding of databases (SQL concepts)

## Setting Up Your Development Environment

### Step 1: Check Python Installation

Django requires Python 3.8 or higher. Check your Python version:

```bash
python --version
# or
python3 --version
```

If you don't have Python installed, download it from [python.org](https://www.python.org/downloads/)

### Step 2: Create Project Directory

```bash
# Create a directory for your project
mkdir faculty-leave-management
cd faculty-leave-management
```

### Step 3: Set Up Virtual Environment

A virtual environment isolates your project dependencies from your system Python installation.

**On Windows:**
```bash
python -m venv venv
venv\Scripts\activate
```

**On macOS/Linux:**
```bash
python3 -m venv venv
source venv/bin/activate
```

You should see `(venv)` prefix in your terminal prompt, indicating the virtual environment is active.

### Step 4: Install Django

```bash
pip install django
```

Verify the installation:

```bash
django-admin --version
```

You should see the Django version (e.g., 5.0.x or 4.2.x)

### Step 5: Install Additional Dependencies

We'll need a few more packages for our project:

```bash
pip install pillow  # For handling image uploads (profile pictures)
```

### Step 6: Create Requirements File

This file lists all project dependencies:

```bash
pip freeze > requirements.txt
```

This allows others to install the same dependencies with:
```bash
pip install -r requirements.txt
```

## Understanding Django Project Structure

Before we create our project, let's understand what we'll get:

```
faculty-leave-management/
├── venv/                          # Virtual environment (don't commit to git)
├── manage.py                      # Command-line utility for Django
├── faculty_leave_system/          # Project configuration directory
│   ├── __init__.py
│   ├── settings.py                # Project settings
│   ├── urls.py                    # URL routing
│   ├── asgi.py                    # ASGI configuration
│   └── wsgi.py                    # WSGI configuration
└── requirements.txt               # Project dependencies
```

## Quick Django Commands Reference

Here are some essential Django commands you'll use throughout this workshop:

```bash
# Create a new Django project
django-admin startproject project_name

# Create a new Django app
python manage.py startapp app_name

# Run development server
python manage.py runserver

# Create database migrations
python manage.py makemigrations

# Apply migrations to database
python manage.py migrate

# Create superuser (admin)
python manage.py createsuperuser

# Open Django shell
python manage.py shell

# Collect static files
python manage.py collectstatic
```

## Development Tools (Optional but Recommended)

### Code Editor
- **VS Code** (Recommended): Free, with excellent Python/Django extensions
- **PyCharm**: Professional IDE with Django support
- **Sublime Text**: Lightweight alternative

### VS Code Extensions for Django
- Python (by Microsoft)
- Django (by Baptiste Darthenay)
- SQLite Viewer
- GitLens (for version control)

### Browser DevTools
- Chrome DevTools or Firefox Developer Tools for frontend debugging

## Database Choice

For this workshop, we'll use **SQLite** (Django's default database). It's:
- File-based (no separate server needed)
- Perfect for development and learning
- Included with Python

For production applications, you might use:
- PostgreSQL (recommended)
- MySQL
- Oracle

## Project Planning: Leave Management System

### Key Features We'll Implement

**For Faculty Members:**
1. Register and login
2. Submit leave requests (casual, sick, earned)
3. View leave balance
4. Check leave request status
5. View leave history

**For Administrators:**
1. View all leave requests
2. Approve or reject requests
3. View faculty leave summaries
4. Manage leave types and policies

### Database Models We'll Create

1. **User** (extends Django's built-in User model)
2. **Faculty Profile** (extra faculty information)
3. **Leave Request** (individual leave applications)
4. **Leave Balance** (track remaining leaves)

## Next Steps

In Part 2, we'll:
- Create our Django project
- Start our first Django app
- Run the development server
- Explore the Django admin interface

## Common Setup Issues and Solutions

### Issue: "django-admin command not found"
**Solution**: Make sure your virtual environment is activated and Django is installed in it.

### Issue: "Permission denied" errors
**Solution**: On macOS/Linux, you might need to use `python3` instead of `python`.

### Issue: Virtual environment not activating
**Solution**: 
- Windows: Try `venv\Scripts\activate.bat` in Command Prompt
- macOS/Linux: Ensure you use `source` before the activate script

### Issue: Port already in use (when running server)
**Solution**: Use a different port:
```bash
python manage.py runserver 8080
```

## Workshop Tips

1. **Type the code yourself** - Don't copy-paste. You'll learn better.
2. **Read error messages carefully** - They often tell you exactly what's wrong.
3. **Use Django documentation** - It's excellent: https://docs.djangoproject.com/
4. **Commit regularly to Git** - Save your progress frequently.
5. **Ask questions** - No question is too basic!

## Resources

- **Official Django Documentation**: https://docs.djangoproject.com/
- **Django Tutorial**: https://docs.djangoproject.com/en/stable/intro/tutorial01/
- **Django Girls Tutorial**: https://tutorial.djangogirls.org/
- **Mozilla Django Tutorial**: https://developer.mozilla.org/en-US/docs/Learn/Server-side/Django

---

**Ready to start?** Move on to Part 2 where we'll create our Django project and write our first code!

## Checklist

Before moving to Part 2, ensure you have:
- [ ] Python 3.8+ installed
- [ ] Virtual environment created and activated
- [ ] Django installed
- [ ] Code editor ready
- [ ] Terminal/command prompt comfortable

---

*Workshop Materials | Faculty Leave Management System | Part 1 of 10*
