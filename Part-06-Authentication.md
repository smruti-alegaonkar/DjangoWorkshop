# Part 6: User Authentication

## Introduction

We'll implement user registration, login, logout, and profile management.

## Step 1: Configure Authentication URLs

Update `leave_management_system/urls.py`:

```python
from django.contrib import admin
from django.urls import path, include
from django.contrib.auth import views as auth_views
from django.conf import settings
from django.conf.urls.static import static


urlpatterns = [
    path('admin/', admin.site.urls),

    path(
        'login/',
        auth_views.LoginView.as_view(
            template_name='registration/login.html',
            redirect_authenticated_user=True
        ),
        name='login'
    ),

    path(
        'logout/',
        auth_views.LogoutView.as_view(next_page='login'),
        name='logout'
    ),
    path('', include('leaves.urls')),
    path('password-reset/', auth_views.PasswordResetView.as_view(template_name='registration/password_reset.html'), name='password_reset'),
    path('password-reset/done/', auth_views.PasswordResetDoneView.as_view(template_name='registration/password_reset_done.html'), name='password_reset_done'),
    path('reset/<uidb64>/<token>/', auth_views.PasswordResetConfirmView.as_view(template_name='registration/password_reset_confirm.html'), name='password_reset_confirm'),
    path('reset/done/', auth_views.PasswordResetCompleteView.as_view(template_name='registration/password_reset_complete.html'), name='password_reset_complete'),
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

## Step 2: Configure Settings

Add to `leave_management_system/settings.py`:

```python
# Authentication
LOGIN_URL = 'login'
LOGIN_REDIRECT_URL = 'leaves:dashboard'
LOGOUT_REDIRECT_URL = 'leaves:home'

# Email backend for development (prints to console)
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

## Step 3: Create Registration View

Add to `leaves/views.py`:

```python
from django.shortcuts import render, redirect
from django.contrib import messages
from django.contrib.auth import login
from .forms import FacultyRegistrationForm
from .models import FacultyProfile, LeaveBalance, LeaveType
from datetime import datetime

def register(request):
    if request.method == 'POST':
        form = FacultyRegistrationForm(request.POST, request.FILES)

        if form.is_valid():
            user = form.save(commit=False)
            user.email = form.cleaned_data['email']
            user.first_name = form.cleaned_data['first_name']
            user.last_name = form.cleaned_data['last_name']
            user.save()

            FacultyProfile.objects.create(
                user=user,
                employee_id=form.cleaned_data['employee_id'],
                department=form.cleaned_data['department'],
                designation=form.cleaned_data['designation'],
                phone_number=form.cleaned_data['phone_number'],
                date_of_joining=form.cleaned_data['date_of_joining'],
                profile_picture=form.cleaned_data.get('profile_picture')
            )

            login(request, user)
            messages.success(request, "Registration successful!")
            return redirect('leaves:dashboard')

        else:
            print(form.errors)   # VERY IMPORTANT FOR DEBUG

    else:
        form = FacultyRegistrationForm()

    return render(request, 'registration/register.html', {'form': form})


@login_required
def profile(request):

    faculty = FacultyProfile.objects.get(user=request.user)

    if request.method == "POST":
        form = ProfileUpdateForm(
            request.POST,
            request.FILES,
            instance=faculty,
            user=request.user
        )

        if form.is_valid():
            form.save()
            return redirect("leaves:profile")

    else:
        form = ProfileUpdateForm(instance=faculty, user=request.user)

    return render(request, "leaves/profile.html", {
        "faculty": faculty,
        "form": form
    })
```

## Step 4: Create Login Template

Create `leaves/templates/registration/login.html`:

```html
{% extends 'leaves/base.html' %}

{% block title %}Login - Leave Management{% endblock %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-6 col-lg-4">
        <div class="card shadow">
            <div class="card-header" style="background: linear-gradient(90deg, #0d6efd, #0b5ed7); color: white;">
                <h4 class="mb-0"><i class="bi bi-box-arrow-in-right"></i> Faculty Login</h4>
            </div>
            <div class="card-body p-4">
                {% if form.errors %}
                    <div class="alert alert-danger">
                        Invalid username or password. Please try again.
                    </div>
                {% endif %}

                <form method="post">
                    {% csrf_token %}
                    <div class="mb-3">
                        <label for="id_username" class="form-label">Username</label>
                        <input type="text" name="username" class="form-control" id="id_username" required autofocus>
                    </div>
                    
                    <div class="mb-3">
                        <label for="id_password" class="form-label">Password</label>
                        <input type="password" name="password" class="form-control" id="id_password" required>
                    </div>
                    
                    <div class="d-grid">
                        <button type="submit" class="btn btn-primary">
                            <i class="bi bi-box-arrow-in-right"></i> Login
                        </button>
                    </div>
                </form>

                <hr>
                
                <div class="text-center">
                    <p class="mb-2">
                        <a href="{% url 'password_reset' %}">Forgot Password?</a>
                    </p>
                    <p class="mb-0">
                        Don't have an account? <a href="{% url 'leaves:register' %}">Register here</a>
                    </p>
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

## Step 5: Create Registration Template

Create `leaves/templates/registration/register.html`:

```html
{% extends 'leaves/base.html' %}

{% if form.errors %}
<div class="alert alert-danger">
    <ul>
        {% for field in form %}
            {% for error in field.errors %}
                <li>{{ field.label }} - {{ error }}</li>
            {% endfor %}
        {% endfor %}
        {% for error in form.non_field_errors %}
            <li>{{ error }}</li>
        {% endfor %}
    </ul>
</div>
{% endif %}

{% block title %}Register{% endblock %}

{% block content %}
<div class="container mt-5">
    <div class="card shadow">
        <div class="card-header bg-primary text-white">
            <h4>Faculty Registration</h4>
        </div>
        <div class="card-body">

            <!-- Show Non Field Errors -->
            {% if form.non_field_errors %}
                <div class="alert alert-danger">
                    {{ form.non_field_errors }}
                </div>
            {% endif %}

            <form method="POST" enctype="multipart/form-data">
                {% csrf_token %}

                <h5>Account Information</h5>
                <div class="row">

                    <div class="col-md-6 mb-3">
                        {{ form.username.label_tag }}
                        {{ form.username }}
                        <div class="text-danger">{{ form.username.errors }}</div>
                    </div>

                    <div class="col-md-6 mb-3">
                        {{ form.email.label_tag }}
                        {{ form.email }}
                        <div class="text-danger">{{ form.email.errors }}</div>
                    </div>

                    <div class="col-md-6 mb-3">
                        {{ form.first_name.label_tag }}
                        {{ form.first_name }}
                        <div class="text-danger">{{ form.first_name.errors }}</div>
                    </div>

                    <div class="col-md-6 mb-3">
                        {{ form.last_name.label_tag }}
                        {{ form.last_name }}
                        <div class="text-danger">{{ form.last_name.errors }}</div>
                    </div>

                    <div class="col-md-6 mb-3">
                        {{ form.password1.label_tag }}
                        {{ form.password1 }}
                        <small class="form-text text-muted">
                            Password must be at least 8 characters and not entirely numeric.
                        </small>
                        <div class="text-danger">{{ form.password1.errors }}</div>
                    </div>

                    <div class="col-md-6 mb-3">
                        {{ form.password2.label_tag }}
                        {{ form.password2 }}
                        <div class="text-danger">{{ form.password2.errors }}</div>
                    </div>

                </div>

                <hr>

                <h5>Faculty Information</h5>
                <div class="row">

                    <div class="col-md-6 mb-3">
                        {{ form.employee_id.label_tag }}
                        {{ form.employee_id }}
                        <div class="text-danger">{{ form.employee_id.errors }}</div>
                    </div>

                    <div class="col-md-6 mb-3">
                        {{ form.department.label_tag }}
                        {{ form.department }}
                        <div class="text-danger">{{ form.department.errors }}</div>
                    </div>

                    <div class="col-md-6 mb-3">
                        {{ form.designation.label_tag }}
                        {{ form.designation }}
                        <div class="text-danger">{{ form.designation.errors }}</div>
                    </div>

                    <div class="col-md-6 mb-3">
                        {{ form.phone_number.label_tag }}
                        {{ form.phone_number }}
                        <div class="text-danger">{{ form.phone_number.errors }}</div>
                    </div>

                    <div class="col-md-6 mb-3">
                        {{ form.date_of_joining.label_tag }}
                        {{ form.date_of_joining }}
                        <div class="text-danger">{{ form.date_of_joining.errors }}</div>
                    </div>

                    <div class="col-md-6 mb-3">
                        {{ form.profile_picture.label_tag }}
                        {{ form.profile_picture }}
                        <div class="text-danger">{{ form.profile_picture.errors }}</div>
                    </div>

                </div>

                <button type="submit" class="btn btn-primary w-100 mt-3">
                    Register
                </button>
            </form>

            <div class="mt-3 text-center">
                Already have an account?
                <a href="{% url 'login' %}">Login here</a>
            </div>

        </div>
    </div>
</div>
{% endblock %}
```

## Step 6: Create Profile Template

Create `leaves/templates/leaves/profile.html`:

```html
{% extends 'leaves/base.html' %}

{% block title %}Profile - Leave Management{% endblock %}

{% block content %}

<div class="row">
    <div class="col-md-4 mb-4">
        <div class="card text-center">
            <div class="card-body">
                {% if faculty.profile_picture %}
                    <img src="{{ faculty.profile_picture.url }}" alt="Profile" class="profile-img mb-3">
                {% else %}
                    <div class="profile-img mb-3 d-flex align-items-center justify-content-center bg-light">
                        <i class="bi bi-person-circle" style="font-size: 100px; color: #ccc;"></i>
                    </div>
                {% endif %}
                <h4>{{ user.get_full_name }}</h4>
                <p class="text-muted">{{ faculty.designation }}</p>
                <p class="text-muted">{{ faculty.department }}</p>
                <p><strong>Employee ID:</strong> {{ faculty.employee_id }}</p>
            </div>
        </div>
    </div>
    
    <div class="col-md-8">
        <div class="card">
            <div class="card-header bg-primary text-white">
                <h5 class="mb-0"><i class="bi bi-pencil"></i> Update Profile</h5>
            </div>
            <div class="card-body">
                <form method="post" enctype="multipart/form-data">
                    {% csrf_token %}
                    
                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label for="{{ form.first_name.id_for_label }}" class="form-label">First Name</label>
                            {{ form.first_name }}
                            {% if form.first_name.errors %}
                                <div class="text-danger">{{ form.first_name.errors }}</div>
                            {% endif %}
                        </div>
                        
                        <div class="col-md-6 mb-3">
                            <label for="{{ form.last_name.id_for_label }}" class="form-label">Last Name</label>
                            {{ form.last_name }}
                            {% if form.last_name.errors %}
                                <div class="text-danger">{{ form.last_name.errors }}</div>
                            {% endif %}
                        </div>
                    </div>
                    
                    <div class="mb-3">
                        <label for="{{ form.email.id_for_label }}" class="form-label">Email</label>
                        {{ form.email }}
                        {% if form.email.errors %}
                            <div class="text-danger">{{ form.email.errors }}</div>
                        {% endif %}
                    </div>
                    
                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label for="{{ form.department.id_for_label }}" class="form-label">Department</label>
                            {{ form.department }}
                            {% if form.department.errors %}
                                <div class="text-danger">{{ form.department.errors }}</div>
                            {% endif %}
                        </div>
                        
                        <div class="col-md-6 mb-3">
                            <label for="{{ form.designation.id_for_label }}" class="form-label">Designation</label>
                            {{ form.designation }}
                            {% if form.designation.errors %}
                                <div class="text-danger">{{ form.designation.errors }}</div>
                            {% endif %}
                        </div>
                    </div>
                    
                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label for="{{ form.phone_number.id_for_label }}" class="form-label">Phone Number</label>
                            {{ form.phone_number }}
                            {% if form.phone_number.errors %}
                                <div class="text-danger">{{ form.phone_number.errors }}</div>
                            {% endif %}
                        </div>
                        
                        <div class="col-md-6 mb-3">
                            <label for="{{ form.profile_picture.id_for_label }}" class="form-label">Profile Picture</label>
                            {{ form.profile_picture }}
                            {% if form.profile_picture.errors %}
                                <div class="text-danger">{{ form.profile_picture.errors }}</div>
                            {% endif %}
                        </div>
                    </div>
                    
                    <button type="submit" class="btn btn-primary">
                        <i class="bi bi-check-circle"></i> Update Profile
                    </button>
                </form>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

## Authentication Decorators

```python
from django.contrib.auth.decorators import login_required, user_passes_test

# Require login
@login_required
def my_view(request):
    pass

# Require staff
@user_passes_test(lambda u: u.is_staff)
def admin_view(request):
    pass

# Custom test
def is_faculty(user):
    return hasattr(user, 'faculty_profile')

@user_passes_test(is_faculty)
def faculty_only_view(request):
    pass
```

## Next Steps

In Part 7, we'll implement admin leave approval functionality.

---

*Workshop Materials | Faculty Leave Management System | Part 6 of 10*
