# Part 4: Templates and Frontend Design

## Introduction

In this part, we'll create the user interface for our Leave Management System using Django's template system. We'll build professional-looking pages with Bootstrap.

## Django Template System

Django templates combine HTML with Django Template Language (DTL) to create dynamic web pages.

### Template Syntax Basics

**Variables**: `{{ variable }}`
```django
<h1>Welcome, {{ user.first_name }}</h1>
```

**Tags**: `{% tag %}`
```django
{% if user.is_authenticated %}
    <p>Hello, logged in user!</p>
{% endif %}
```

**Filters**: `{{ variable|filter }}`
```django
{{ text|lower }}
{{ date|date:"M d, Y" }}
```

**Comments**: `{# comment #}`
```django
{# This won't be rendered #}
```

## Step 1: Template Directory Structure

Create the templates directory structure:

```bash
# In your project root
mkdir -p leaves/templates/leaves
mkdir -p leaves/templates/registration
mkdir -p leaves/static/leaves/css
mkdir -p leaves/static/leaves/js
mkdir -p leaves/static/leaves/images
```

**Why this structure?**
- Django looks for templates in `app_name/templates/app_name/`
- The double `leaves` prevents namespace conflicts
- Static files go in `app_name/static/app_name/`

## Step 2: Configure Static Files

Static files are CSS, JavaScript, images, etc.

In `leave_management_system/settings.py`, verify/add:

```python
# Static files (CSS, JavaScript, Images)
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

# Media files (user uploads)
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

Update `leave_management_system/urls.py` to serve media files in development:

```python
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('leaves.urls')),
]

# Serve media files in development
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
    urlpatterns += static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)
```

## Step 3: Create Base Template

Create `leaves/templates/leaves/base.html` - this will be the parent template:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Faculty Leave Management{% endblock %}</title>
    
    <!-- Bootstrap CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    
    <!-- Bootstrap Icons -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.10.0/font/bootstrap-icons.css" rel="stylesheet">
    
    <!-- Custom CSS -->
    {% load static %}
    <link rel="stylesheet" href="{% static 'leaves/css/style.css' %}">
    
    {% block extra_css %}{% endblock %}
</head>
<body>
    <!-- Navigation Bar -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
        <div class="container-fluid">
            <a class="navbar-brand" href="{% url 'leaves:dashboard' %}">
                <i class="bi bi-calendar-check"></i> Leave Management
            </a>
            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav me-auto">
                    <ul class="navbar-nav me-auto">
                    {% if user.is_authenticated %}

                        {% if user.is_staff %}
                            <!-- ADMIN NAV -->
                            <li class="nav-item">
                                <a class="nav-link" href="{% url 'leaves:pending_requests' %}">
                                    Pending Applications
                                </a>
                            </li>

                            <li class="nav-item">
                                <a class="nav-link" href="{% url 'leaves:reports' %}">
                                    Reports & Analytics
                                </a>
                            </li>

                        {% else %}
                            <!-- FACULTY NAV -->
                            <li class="nav-item">
                                <a class="nav-link" href="{% url 'leaves:dashboard' %}">
                                    Dashboard
                                </a>
                            </li>

                            <li class="nav-item">
                                <a class="nav-link" href="{% url 'leaves:apply_leave' %}">
                                    Apply Leave
                                </a>
                            </li>

                            <li class="nav-item">
                                <a class="nav-link" href="{% url 'leaves:reports' %}">
                                    Reports
                                </a>
                            </li>

                        {% endif %}

                    {% endif %}
                </ul>
                
                <ul class="navbar-nav">
                    {% if user.is_authenticated %}
                        <li class="nav-item dropdown">
                            <a class="nav-link dropdown-toggle" href="#" id="userDropdown" role="button" 
                               data-bs-toggle="dropdown">
                                <i class="bi bi-person-circle"></i> {{ user.get_full_name|default:user.username }}
                            </a>
                            <ul class="dropdown-menu dropdown-menu-end">
                                <li><a class="dropdown-item" href="{% url 'leaves:profile' %}">Profile</a></li>
                                <li><hr class="dropdown-divider"></li>
                                <li>
                                    <form method="POST" action="{% url 'logout' %}" style="display:inline;">
                                        {% csrf_token %}
                                        <button type="submit" class="dropdown-item">
                                            Logout
                                        </button>
                                    </form>
                                </li>
                            </ul>
                        </li>
                    {% else %}
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'login' %}">Login</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'leaves:register' %}">Register</a>
                        </li>
                    {% endif %}
                </ul>
            </div>
        </div>
    </nav>

    <!-- Messages -->
    {% if messages %}
        <div class="container mt-3">
            {% for message in messages %}
                <div class="alert alert-{{ message.tags }} alert-dismissible fade show" role="alert">
                    {{ message }}
                    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                </div>
            {% endfor %}
        </div>
    {% endif %}

    <!-- Main Content -->
    <main class="container my-4">
        {% block content %}
        {% endblock %}
    </main>

    <!-- Footer -->
    <footer class="bg-light text-center text-lg-start mt-5">
        <div class="container p-4">
            <div class="text-center">
                <p class="text-muted">
                    © 2026 Faculty Leave Management System. All rights reserved.
                </p>
            </div>
        </div>
    </footer>

    <!-- Bootstrap JS -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    
    <!-- Custom JS -->
    <script src="{% static 'leaves/js/script.js' %}"></script>
    
    {% block extra_js %}{% endblock %}
</body>
</html>
```

### Understanding the Base Template

**Template Tags Used:**

`{% block title %}`: Define a block that child templates can override
`{% load static %}`: Load static files functionality
`{% static 'path' %}`: Generate URL for static files
`{% url 'name' %}`: Generate URL from URL name
`{% if condition %}`: Conditional rendering
`{% for item in list %}`: Loop through items
`{{ variable }}`: Output variable
`{{ var|filter }}`: Apply filter to variable

## Step 4: Create Custom CSS

Create `leaves/static/leaves/css/style.css`:

```css
:root {
    --bg: #f3f7fc;
    --surface: #ffffff;
    --surface-soft: #f8fbff;
    --text: #1d2a3b;
    --muted: #5f6f86;
    --primary: #1f6feb;
    --primary-dark: #184fa7;
    --accent: #2abf88;
    --danger: #dc3545;
    --border: #dbe5f0;
    --shadow-sm: 0 6px 18px rgba(30, 53, 86, 0.08);
    --shadow-md: 0 12px 30px rgba(30, 53, 86, 0.14);
    --radius: 14px;
    --radius-sm: 10px;
}

*,
*::before,
*::after {
    box-sizing: border-box;
}

body {
    margin: 0;
    color: var(--text);
    font-family: "Segoe UI", "Noto Sans", sans-serif;
    background:
        radial-gradient(circle at 8% -5%, rgba(31, 111, 235, 0.16), transparent 40%),
        radial-gradient(circle at 90% 2%, rgba(42, 191, 136, 0.12), transparent 35%),
        var(--bg);
    min-height: 100vh;
    line-height: 1.5;
}

a {
    color: var(--primary);
}

a:hover {
    color: var(--primary-dark);
}

.navbar.navbar-dark.bg-primary {
    background: linear-gradient(120deg, #1649a6, #1f6feb 52%, #20a4f3) !important;
    box-shadow: var(--shadow-sm);
}

.navbar-brand {
    font-weight: 700;
    letter-spacing: 0.2px;
}

.navbar .nav-link {
    border-radius: 999px;
    color: rgba(255, 255, 255, 0.94) !important;
    margin: 0 3px;
    padding: 8px 14px !important;
    transition: background-color 0.2s ease, transform 0.2s ease;
}

.navbar .nav-link:hover,
.navbar .nav-link:focus {
    background-color: rgba(255, 255, 255, 0.17);
    transform: translateY(-1px);
}

.dropdown-menu {
    border: 1px solid var(--border);
    border-radius: var(--radius-sm);
    box-shadow: var(--shadow-sm);
}

.dropdown-item:active {
    background-color: var(--primary);
}

main.container {
    margin-top: 2rem !important;
    margin-bottom: 2rem !important;
}

.card {
    border: 1px solid var(--border);
    border-radius: var(--radius);
    box-shadow: var(--shadow-sm);
    background: var(--surface);
    transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.card:hover {
    transform: translateY(-2px);
    box-shadow: var(--shadow-md);
}

.card-header {
    background: linear-gradient(180deg, #fdfefe, #f3f8ff);
    border-bottom: 1px solid var(--border);
    font-weight: 600;
}

.table {
    --bs-table-bg: transparent;
}

.table thead th {
    background: var(--surface-soft);
    border-bottom-width: 1px;
    color: #28405f;
    font-weight: 600;
}

.table td,
.table th {
    vertical-align: middle;
}

.btn {
    border-radius: 10px;
    font-weight: 600;
    letter-spacing: 0.15px;
}

.btn-primary {
    background: linear-gradient(120deg, #1d5fd1, #1f6feb);
    border: none;
}

.btn-primary:hover,
.btn-primary:focus {
    background: linear-gradient(120deg, #1a53b8, #195fcb);
}

.btn-outline-primary {
    border-color: #2f74ec;
    color: #2f74ec;
}

.btn-outline-primary:hover {
    background-color: #2f74ec;
    border-color: #2f74ec;
}

.form-control,
.form-select {
    border-radius: 10px;
    border: 1px solid #cfdceb;
    min-height: 42px;
}

.form-control:focus,
.form-select:focus {
    border-color: #8bb6ff;
    box-shadow: 0 0 0 0.2rem rgba(31, 111, 235, 0.14);
}

.alert {
    border-radius: 12px;
    border: 1px solid transparent;
    box-shadow: 0 5px 14px rgba(22, 40, 66, 0.08);
}

.alert-success {
    border-color: rgba(42, 191, 136, 0.25);
}

.alert-danger {
    border-color: rgba(220, 53, 69, 0.25);
}

.badge {
    border-radius: 999px;
    font-weight: 600;
    padding: 6px 10px;
}

footer.bg-light {
    background: #eef4fb !important;
    border-top: 1px solid var(--border);
}

footer .text-muted {
    color: var(--muted) !important;
    margin: 0;
}

/* Optional dashboard layout styles retained for pages that use sidebar markup */
.layout-wrapper {
    display: flex;
    min-height: calc(100vh - 56px);
}

.sidebar {
    width: 240px;
    height: 100%;
    background: var(--surface);
    border-right: 1px solid var(--border);
    padding: 20px 0;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
}

.sidebar-brand {
    font-size: 20px;
    font-weight: 700;
    padding: 0 20px 20px;
}

.sidebar-brand i {
    color: var(--primary);
    margin-right: 8px;
}

.sidebar-menu {
    list-style: none;
    margin: 0;
    padding: 0 10px;
}

.sidebar-menu li {
    margin: 5px 0;
}

.sidebar-menu a {
    display: block;
    color: #2b3b52;
    text-decoration: none;
    border-radius: 10px;
    padding: 10px 12px;
}

.sidebar-menu a:hover,
.sidebar-menu a.active {
    background: #e8f1ff;
    color: var(--primary-dark);
}

.sidebar-footer {
    padding: 16px 20px;
}

.logout-btn {
    width: 100%;
    border: none;
    background: none;
    color: var(--danger);
    text-align: left;
    font-weight: 600;
}

.main-content {
    flex: 1;
    padding: 24px;
}

.topbar {
    display: flex;
    justify-content: flex-end;
    margin-bottom: 20px;
}

.user-info {
    font-weight: 600;
}

@media (max-width: 991.98px) {
    main.container {
        margin-top: 1.25rem !important;
        margin-bottom: 1.5rem !important;
    }

    .layout-wrapper {
        flex-direction: column;
    }

    .sidebar {
        width: 100%;
        border-right: 0;
        border-bottom: 1px solid var(--border);
    }

    .main-content {
        padding: 16px;
    }

    .card:hover {
        transform: none;
    }
}
```

## Step 5: Create Dashboard Page

Create `leaves/templates/leaves/dashboard.html`:

```html
{% extends 'leaves/base.html' %}

{% block title %}Dashboard - Leave Management{% endblock %}

{% block content %}
<div class="container mt-4">

    <h1 class="mb-4">
        <i class="bi bi-speedometer2"></i> Dashboard
    </h1>

    <!-- Welcome -->
    <div class="alert alert-info">
        <h4 class="alert-heading">
            Welcome, {{ user.get_full_name|default:user.username }}!
        </h4>
        <p class="mb-0">Here's an overview of your leave status.</p>
    </div>

    <!-- Statistics -->
    <div class="row mb-4">

        <div class="col-md-3 mb-3">
            <div class="card shadow-sm">
                <div class="card-body text-center">
                    <h6 class="text-muted">Pending</h6>
                    <h2>{{ pending_count }}</h2>
                </div>
            </div>
        </div>

        <div class="col-md-3 mb-3">
            <div class="card shadow-sm">
                <div class="card-body text-center">
                    <h6 class="text-muted">Approved</h6>
                    <h2>{{ approved_count }}</h2>
                </div>
            </div>
        </div>

        <div class="col-md-3 mb-3">
            <div class="card shadow-sm">
                <div class="card-body text-center">
                    <h6 class="text-muted">Rejected</h6>
                    <h2>{{ rejected_count }}</h2>
                </div>
            </div>
        </div>

        <div class="col-md-3 mb-3">
            <div class="card shadow-sm">
                <div class="card-body text-center">
                    <h6 class="text-muted">Total</h6>
                    <h2>{{ total_count }}</h2>
                </div>
            </div>
        </div>

    </div>

    <!-- Leave Balance -->
    <h3 class="mb-3">Leave Balance ({{ now|date:"Y" }})</h3>

    {% if leave_balances %}
        <div class="row mb-4">
            {% for balance in leave_balances %}
            <div class="col-md-4 mb-3">
                <div class="card shadow-sm">
                    <div class="card-body">
                        <h5>{{ balance.leave_type.name }}</h5>
                        <p>Total: {{ balance.total_leaves }}</p>
                        <p>Used: {{ balance.used_leaves }}</p>
                        <p><strong>Remaining: {{ balance.remaining_leaves }}</strong></p>
                    </div>
                </div>
            </div>
            {% endfor %}
        </div>
    {% else %}
        <div class="alert alert-warning">
            No leave balance information available.
        </div>
    {% endif %}

    <!-- Recent Leaves -->
    <h3 class="mb-3">Recent Leave Requests</h3>

    <div class="card shadow-sm">
        <div class="card-body">

            {% if recent_leaves %}
                <div class="table-responsive">
                    <table class="table table-bordered">
                        <thead>
                            <tr>
                                <th>Type</th>
                                <th>From</th>
                                <th>To</th>
                                <th>Days</th>
                                <th>Status</th>
                            </tr>
                        </thead>
                        <tbody>
                            {% for leave in recent_leaves %}
                            <tr>
                                <td>{{ leave.leave_type.name }}</td>
                                <td>{{ leave.start_date }}</td>
                                <td>{{ leave.end_date }}</td>
                                <td>{{ leave.number_of_days }}</td>
                                <td>
                                    <span class="badge bg-secondary">
                                        {{ leave.get_status_display }}
                                    </span>
                                </td>
                            </tr>
                            {% endfor %}
                        </tbody>
                    </table>
                </div>

                <div class="text-center mt-3">
                    <a href="{% url 'leaves:leave_history' %}" class="btn btn-primary">
                        View All
                    </a>
                </div>

            {% else %}
                <p class="text-muted text-center">No leave requests yet.</p>
                <div class="text-center">
                    <a href="{% url 'leaves:apply_leave' %}" class="btn btn-primary">
                        Apply for Leave
                    </a>
                </div>
            {% endif %}

        </div>
    </div>

</div>
{% endblock %}
```

## Step 6: Update Views

Update `leaves/views.py` to render these templates:

```python
from django.shortcuts import render, redirect
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from .models import LeaveRequest, LeaveBalance, FacultyProfile
from datetime import datetime


@login_required
def dashboard(request):

    # If admin → redirect to admin panel
    if request.user.is_superuser:
        return redirect("admin:index")

    # Ensure faculty profile exists
    if not hasattr(request.user, "faculty_profile"):
        return redirect("logout")

    faculty = request.user.faculty_profile
    current_year = datetime.now().year

    # Leave counts
    pending_count = LeaveRequest.objects.filter(
        faculty=faculty,
        status="pending"
    ).count()

    approved_count = LeaveRequest.objects.filter(
        faculty=faculty,
        status="approved"
    ).count()

    rejected_count = LeaveRequest.objects.filter(
        faculty=faculty,
        status="rejected"
    ).count()

    total_count = LeaveRequest.objects.filter(
        faculty=faculty
    ).count()

    # Leave balances
    leave_balances = LeaveBalance.objects.filter(
        faculty=faculty,
        year=current_year
    )

    # Recent leave requests (latest 5)
    recent_leaves = LeaveRequest.objects.filter(
        faculty=faculty
    ).order_by("-applied_on")[:5]

    context = {
        "pending_count": pending_count,
        "approved_count": approved_count,
        "rejected_count": rejected_count,
        "total_count": total_count,
        "leave_balances": leave_balances,
        "recent_leaves": recent_leaves,
    }

    return render(request, "leaves/dashboard.html", context)
```

## Step 7: Update URL Patterns

Update `leaves/urls.py`:

```python
from django.urls import path
from . import views
from .views import CustomLoginView

app_name = 'leaves'

urlpatterns = [
    path('login/', CustomLoginView.as_view(), name='login'),
    path('dashboard/', views.dashboard, name='dashboard'),
    # We'll add more URLs in next parts
    path('apply/', views.apply_leave, name='apply_leave'),
    path('history/', views.leave_history, name='leave_history'),
    path('pending/', views.pending_requests, name='pending_requests'),
    path('profile/', views.profile, name='profile'),
    path('register/', views.register, name='register'),
]
```

## Template Inheritance Explained

```
base.html (Parent)
    ├── dashboard.html (Child)
    └── other_pages.html (Child)
```

**Child templates extend parent:**
```django
{% extends 'leaves/base.html' %}

{% block content %}
    <!-- Your content here -->
{% endblock %}
```

## Common Template Filters

```django
{{ value|lower }}              <!-- Lowercase -->
{{ value|upper }}              <!-- Uppercase -->
{{ value|title }}              <!-- Title Case -->
{{ value|truncatewords:30 }}   <!-- Truncate to 30 words -->
{{ date|date:"M d, Y" }}       <!-- Format date -->
{{ number|floatformat:2 }}     <!-- 2 decimal places -->
{{ list|length }}              <!-- Length of list -->
{{ value|default:"N/A" }}      <!-- Default if empty -->
{{ text|linebreaks }}          <!-- Convert newlines to <p> tags -->
```

## Template Tags Reference

```django
{% if condition %}...{% endif %}
{% for item in list %}...{% empty %}...{% endfor %}
{% with name=value %}...{% endwith %}
{% url 'name' arg1 arg2 %}
{% static 'path/to/file' %}
{% load static %}
{% csrf_token %}  <!-- Required in all forms -->
{% include 'template_name.html' %}
```

## Next Steps

In Part 5, we'll:
- Create forms for leave application
- Handle form submission
- Validate user input
- Work with file uploads

## Checklist

Before moving to Part 5, ensure you:
- [ ] Created templates directory structure
- [ ] Created base.html template
- [ ] Added custom CSS
- [ ] Created home and dashboard templates
- [ ] Updated views to render templates
- [ ] Can see the home page properly
- [ ] Dashboard shows correct data
- [ ] Navigation works between pages

---

*Workshop Materials | Faculty Leave Management System | Part 4 of 10*
