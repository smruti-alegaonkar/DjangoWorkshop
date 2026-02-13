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
            <a class="navbar-brand" href="{% url 'leaves:home' %}">
                <i class="bi bi-calendar-check"></i> Leave Management
            </a>
            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav me-auto">
                    <li class="nav-item">
                        <a class="nav-link" href="{% url 'leaves:home' %}">Home</a>
                    </li>
                    {% if user.is_authenticated %}
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'leaves:dashboard' %}">Dashboard</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'leaves:apply_leave' %}">Apply Leave</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'leaves:leave_history' %}">My Leaves</a>
                        </li>
                        {% if user.is_staff %}
                            <li class="nav-item">
                                <a class="nav-link" href="{% url 'leaves:pending_requests' %}">Pending Requests</a>
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
                                <li><a class="dropdown-item" href="{% url 'logout' %}">Logout</a></li>
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
/* Custom styles for Leave Management System */

:root {
    --primary-color: #0d6efd;
    --secondary-color: #6c757d;
    --success-color: #198754;
    --danger-color: #dc3545;
    --warning-color: #ffc107;
}

body {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
}

main {
    flex: 1;
}

/* Card styles */
.card {
    box-shadow: 0 0.125rem 0.25rem rgba(0, 0, 0, 0.075);
    transition: transform 0.2s, box-shadow 0.2s;
}

.card:hover {
    transform: translateY(-5px);
    box-shadow: 0 0.5rem 1rem rgba(0, 0, 0, 0.15);
}

/* Dashboard cards */
.stat-card {
    border-left: 4px solid var(--primary-color);
}

.stat-card.pending {
    border-left-color: var(--warning-color);
}

.stat-card.approved {
    border-left-color: var(--success-color);
}

.stat-card.rejected {
    border-left-color: var(--danger-color);
}

/* Badge styles */
.badge-pending {
    background-color: var(--warning-color);
    color: #000;
}

.badge-approved {
    background-color: var(--success-color);
}

.badge-rejected {
    background-color: var(--danger-color);
}

.badge-cancelled {
    background-color: var(--secondary-color);
}

/* Table styles */
.table-hover tbody tr:hover {
    background-color: rgba(0, 0, 0, 0.02);
    cursor: pointer;
}

/* Profile image */
.profile-img {
    width: 150px;
    height: 150px;
    object-fit: cover;
    border-radius: 50%;
    border: 4px solid var(--primary-color);
}

/* Leave balance cards */
.leave-balance-card {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
}

/* Footer */
footer {
    margin-top: auto;
}

/* Animations */
@keyframes fadeIn {
    from {
        opacity: 0;
        transform: translateY(20px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.fade-in {
    animation: fadeIn 0.5s ease-in;
}

/* Responsive adjustments */
@media (max-width: 768px) {
    .stat-card {
        margin-bottom: 1rem;
    }
}
```

## Step 5: Create Home Page

Create `leaves/templates/leaves/home.html`:

```html
{% extends 'leaves/base.html' %}

{% block title %}Home - Leave Management{% endblock %}

{% block content %}
<div class="row">
    <!-- Hero Section -->
    <div class="col-12 mb-5">
        <div class="jumbotron bg-light p-5 rounded">
            <h1 class="display-4">
                <i class="bi bi-calendar-check text-primary"></i>
                Welcome to Faculty Leave Management System
            </h1>
            <p class="lead">Streamline your leave application process with our easy-to-use platform.</p>
            <hr class="my-4">
            <p>Apply for leaves, track your applications, and manage your leave balance - all in one place.</p>
            {% if not user.is_authenticated %}
                <a class="btn btn-primary btn-lg" href="{% url 'login' %}" role="button">
                    <i class="bi bi-box-arrow-in-right"></i> Login
                </a>
                <a class="btn btn-outline-primary btn-lg" href="{% url 'leaves:register' %}" role="button">
                    <i class="bi bi-person-plus"></i> Register
                </a>
            {% else %}
                <a class="btn btn-primary btn-lg" href="{% url 'leaves:dashboard' %}" role="button">
                    <i class="bi bi-speedometer2"></i> Go to Dashboard
                </a>
            {% endif %}
        </div>
    </div>

    <!-- Features Section -->
    <div class="col-12 mb-4">
        <h2 class="text-center mb-4">Key Features</h2>
    </div>

    <div class="col-md-4 mb-4">
        <div class="card h-100 text-center fade-in">
            <div class="card-body">
                <i class="bi bi-file-earmark-text text-primary" style="font-size: 3rem;"></i>
                <h5 class="card-title mt-3">Easy Application</h5>
                <p class="card-text">Submit leave requests with just a few clicks. Attach supporting documents if needed.</p>
            </div>
        </div>
    </div>

    <div class="col-md-4 mb-4">
        <div class="card h-100 text-center fade-in">
            <div class="card-body">
                <i class="bi bi-clock-history text-success" style="font-size: 3rem;"></i>
                <h5 class="card-title mt-3">Track Status</h5>
                <p class="card-text">Monitor your leave applications in real-time. Get notified when status changes.</p>
            </div>
        </div>
    </div>

    <div class="col-md-4 mb-4">
        <div class="card h-100 text-center fade-in">
            <div class="card-body">
                <i class="bi bi-graph-up text-warning" style="font-size: 3rem;"></i>
                <h5 class="card-title mt-3">Leave Balance</h5>
                <p class="card-text">View your remaining leave balance for different leave types at a glance.</p>
            </div>
        </div>
    </div>

    <!-- Leave Types -->
    <div class="col-12 mt-5">
        <h2 class="text-center mb-4">Leave Types Available</h2>
        <div class="row">
            <div class="col-md-4 mb-3">
                <div class="card border-primary">
                    <div class="card-header bg-primary text-white">
                        <h5 class="mb-0">Casual Leave</h5>
                    </div>
                    <div class="card-body">
                        <p class="card-text">For personal matters and unforeseen circumstances.</p>
                        <p class="mb-0"><strong>Maximum:</strong> 12 days per year</p>
                    </div>
                </div>
            </div>
            <div class="col-md-4 mb-3">
                <div class="card border-success">
                    <div class="card-header bg-success text-white">
                        <h5 class="mb-0">Sick Leave</h5>
                    </div>
                    <div class="card-body">
                        <p class="card-text">When you're unwell and need time to recover.</p>
                        <p class="mb-0"><strong>Maximum:</strong> 10 days per year</p>
                    </div>
                </div>
            </div>
            <div class="col-md-4 mb-3">
                <div class="card border-info">
                    <div class="card-header bg-info text-white">
                        <h5 class="mb-0">Earned Leave</h5>
                    </div>
                    <div class="card-body">
                        <p class="card-text">Earned through continuous service. Can be carried forward.</p>
                        <p class="mb-0"><strong>Maximum:</strong> 30 days per year</p>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

## Step 6: Create Dashboard Page

Create `leaves/templates/leaves/dashboard.html`:

```html
{% extends 'leaves/base.html' %}

{% block title %}Dashboard - Leave Management{% endblock %}

{% block content %}
<h1 class="mb-4">
    <i class="bi bi-speedometer2"></i> Dashboard
</h1>

<!-- Welcome Message -->
<div class="alert alert-info">
    <h4 class="alert-heading">Welcome, {{ user.get_full_name }}!</h4>
    <p class="mb-0">Here's an overview of your leave status.</p>
</div>

<!-- Statistics Cards -->
<div class="row mb-4">
    <div class="col-md-3 mb-3">
        <div class="card stat-card pending">
            <div class="card-body">
                <div class="d-flex justify-content-between align-items-center">
                    <div>
                        <h6 class="text-muted mb-1">Pending</h6>
                        <h2 class="mb-0">{{ pending_count }}</h2>
                    </div>
                    <div>
                        <i class="bi bi-hourglass-split text-warning" style="font-size: 2.5rem;"></i>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <div class="col-md-3 mb-3">
        <div class="card stat-card approved">
            <div class="card-body">
                <div class="d-flex justify-content-between align-items-center">
                    <div>
                        <h6 class="text-muted mb-1">Approved</h6>
                        <h2 class="mb-0">{{ approved_count }}</h2>
                    </div>
                    <div>
                        <i class="bi bi-check-circle text-success" style="font-size: 2.5rem;"></i>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <div class="col-md-3 mb-3">
        <div class="card stat-card rejected">
            <div class="card-body">
                <div class="d-flex justify-content-between align-items-center">
                    <div>
                        <h6 class="text-muted mb-1">Rejected</h6>
                        <h2 class="mb-0">{{ rejected_count }}</h2>
                    </div>
                    <div>
                        <i class="bi bi-x-circle text-danger" style="font-size: 2.5rem;"></i>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <div class="col-md-3 mb-3">
        <div class="card stat-card">
            <div class="card-body">
                <div class="d-flex justify-content-between align-items-center">
                    <div>
                        <h6 class="text-muted mb-1">Total Leaves</h6>
                        <h2 class="mb-0">{{ total_count }}</h2>
                    </div>
                    <div>
                        <i class="bi bi-calendar3 text-primary" style="font-size: 2.5rem;"></i>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>

<!-- Leave Balance -->
<div class="row mb-4">
    <div class="col-12">
        <h3 class="mb-3">Leave Balance</h3>
    </div>
    {% for balance in leave_balances %}
    <div class="col-md-4 mb-3">
        <div class="card leave-balance-card">
            <div class="card-body">
                <h5 class="card-title">{{ balance.leave_type.name }}</h5>
                <div class="progress mb-2" style="height: 25px;">
                    {% widthratio balance.remaining_leaves balance.total_leaves 100 as percentage %}
                    <div class="progress-bar bg-warning" role="progressbar" 
                         style="width: {{ percentage }}%">
                        {{ balance.remaining_leaves }} / {{ balance.total_leaves }}
                    </div>
                </div>
                <small>{{ balance.remaining_leaves }} days remaining</small>
            </div>
        </div>
    </div>
    {% empty %}
    <div class="col-12">
        <div class="alert alert-warning">
            No leave balance information available. Please contact admin.
        </div>
    </div>
    {% endfor %}
</div>

<!-- Recent Leave Requests -->
<div class="row">
    <div class="col-12">
        <h3 class="mb-3">Recent Leave Requests</h3>
        <div class="card">
            <div class="card-body">
                {% if recent_leaves %}
                <div class="table-responsive">
                    <table class="table table-hover">
                        <thead>
                            <tr>
                                <th>Leave Type</th>
                                <th>Start Date</th>
                                <th>End Date</th>
                                <th>Days</th>
                                <th>Status</th>
                                <th>Applied On</th>
                            </tr>
                        </thead>
                        <tbody>
                            {% for leave in recent_leaves %}
                            <tr>
                                <td>{{ leave.leave_type.name }}</td>
                                <td>{{ leave.start_date|date:"M d, Y" }}</td>
                                <td>{{ leave.end_date|date:"M d, Y" }}</td>
                                <td>{{ leave.number_of_days }}</td>
                                <td>
                                    <span class="badge badge-{{ leave.status }}">
                                        {{ leave.get_status_display }}
                                    </span>
                                </td>
                                <td>{{ leave.applied_on|date:"M d, Y" }}</td>
                            </tr>
                            {% endfor %}
                        </tbody>
                    </table>
                </div>
                <div class="text-center mt-3">
                    <a href="{% url 'leaves:leave_history' %}" class="btn btn-primary">
                        View All Leaves
                    </a>
                </div>
                {% else %}
                <p class="text-center text-muted mb-0">No leave requests yet.</p>
                <div class="text-center mt-3">
                    <a href="{% url 'leaves:apply_leave' %}" class="btn btn-primary">
                        Apply for Leave
                    </a>
                </div>
                {% endif %}
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

## Step 7: Update Views

Update `leaves/views.py` to render these templates:

```python
from django.shortcuts import render, redirect
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from .models import LeaveRequest, LeaveBalance, FacultyProfile
from datetime import datetime

def home(request):
    """Home page view"""
    return render(request, 'leaves/home.html')

@login_required
def dashboard(request):
    """Dashboard view - shows overview of user's leaves"""
    try:
        faculty = request.user.faculty_profile
        
        # Get leave statistics
        pending_count = LeaveRequest.objects.filter(
            faculty=faculty, 
            status='pending'
        ).count()
        
        approved_count = LeaveRequest.objects.filter(
            faculty=faculty, 
            status='approved'
        ).count()
        
        rejected_count = LeaveRequest.objects.filter(
            faculty=faculty, 
            status='rejected'
        ).count()
        
        total_count = LeaveRequest.objects.filter(faculty=faculty).count()
        
        # Get leave balances
        current_year = datetime.now().year
        leave_balances = LeaveBalance.objects.filter(
            faculty=faculty,
            year=current_year
        )
        
        # Get recent leave requests
        recent_leaves = LeaveRequest.objects.filter(
            faculty=faculty
        ).order_by('-applied_on')[:5]
        
        context = {
            'pending_count': pending_count,
            'approved_count': approved_count,
            'rejected_count': rejected_count,
            'total_count': total_count,
            'leave_balances': leave_balances,
            'recent_leaves': recent_leaves,
        }
        
        return render(request, 'leaves/dashboard.html', context)
    
    except FacultyProfile.DoesNotExist:
        messages.error(request, 'Faculty profile not found. Please contact admin.')
        return redirect('leaves:home')
```

## Step 8: Update URL Patterns

Update `leaves/urls.py`:

```python
from django.urls import path
from . import views

app_name = 'leaves'

urlpatterns = [
    path('', views.home, name='home'),
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
    ├── home.html (Child)
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
