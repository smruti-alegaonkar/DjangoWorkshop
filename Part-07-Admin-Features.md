# Part 7: Admin Leave Approval and Management

## Introduction

We'll create functionality for administrators to review and approve/reject leave requests.

## Step 1: Create Admin Views

Add to `leaves/views.py`:

```python
from django.contrib.auth.decorators import login_required, user_passes_test
from django.shortcuts import render, redirect, get_object_or_404
from django.utils import timezone

def is_staff_user(user):
    return user.is_superuser

@user_passes_test(is_staff_user)
def pending_requests(request):
    """View all pending leave requests"""
    leave_requests = LeaveRequest.objects.filter(
        status='pending'
    ).order_by('-applied_on')
    
    context = {
        'leave_requests': leave_requests,
    }
    return render(request, 'leaves/pending_requests.html', context)


@login_required
@user_passes_test(is_staff_user)
def review_leave(request, leave_id):
    """Review and approve/reject leave request"""
    leave_request = get_object_or_404(LeaveRequest, id=leave_id)
    
    if request.method == 'POST':
        form = LeaveReviewForm(request.POST, instance=leave_request)
        if form.is_valid():
            leave = form.save(commit=False)
            leave.reviewed_by = request.user
            leave.reviewed_on = timezone.now()
            
            # Update leave balance if approved
            if leave.status == 'approved':
                try:
                    leave_balance = LeaveBalance.objects.get(
                        faculty=leave.faculty,
                        leave_type=leave.leave_type,
                        year=leave.start_date.year
                    )
                    leave_balance.used_leaves = leave_balance.used_leaves + leave.number_of_days
                    leave_balance.save()
                    
                    messages.success(request, f'Leave request approved for {leave.faculty.user.get_full_name()}.')
                except LeaveBalance.DoesNotExist:
                    messages.warning(request, 'Leave approved but balance not found.')
            else:
                messages.info(request, f'Leave request rejected for {leave.faculty.user.get_full_name()}.')
            
            leave.save()
            return redirect('leaves:pending_requests')
    else:
        form = LeaveReviewForm(instance=leave_request)
    
    context = {
        'form': form,
        'leave_request': leave_request,
    }
    return render(request, 'leaves/review_leave.html', context)


@login_required
@user_passes_test(is_staff_user)
def all_leaves(request):
    """View all leave requests (admin dashboard)"""
    # Get filter parameters
    status_filter = request.GET.get('status', '')
    department_filter = request.GET.get('department', '')
    
    # Base query
    leave_requests = LeaveRequest.objects.all()
    
    # Apply filters
    if status_filter:
        leave_requests = leave_requests.filter(status=status_filter)
    if department_filter:
        leave_requests = leave_requests.filter(faculty__department=department_filter)
    
    # Order by latest
    leave_requests = leave_requests.order_by('-applied_on')
    
    # Get statistics
    total_requests = LeaveRequest.objects.count()
    pending_count = LeaveRequest.objects.filter(status='pending').count()
    approved_count = LeaveRequest.objects.filter(status='approved').count()
    rejected_count = LeaveRequest.objects.filter(status='rejected').count()
    
    # Get unique departments for filter
    from .models import FacultyProfile
    departments = FacultyProfile.objects.values_list('department', flat=True).distinct()
    
    context = {
        'leave_requests': leave_requests,
        'total_requests': total_requests,
        'pending_count': pending_count,
        'approved_count': approved_count,
        'rejected_count': rejected_count,
        'departments': departments,
        'current_status': status_filter,
        'current_department': department_filter,
    }
    return render(request, 'leaves/all_leaves.html', context)
```

## Step 2: Update URLs

Update `leaves/urls.py`:

```python
from django.urls import path
from . import views
from .views import CustomLoginView

app_name = 'leaves'

urlpatterns = [
    path('login/', CustomLoginView.as_view(), name='login'),
    path('', views.dashboard, name='dashboard'),
    path('apply/', views.apply_leave, name='apply_leave'),
    path('history/', views.leave_history, name='leave_history'),
    path('profile/', views.profile, name='profile'),
    path('register/', views.register, name='register'),
    
    # Admin URLs
    path('pending/', views.pending_requests, name='pending_requests'),
    path('review/<int:leave_id>/', views.review_leave, name='review_leave'),
    path('all-leaves/', views.all_leaves, name='all_leaves'),
]
```

## Step 3: Create Pending Requests Template

Create `leaves/templates/leaves/pending_requests.html`:

```html
{% extends 'leaves/base.html' %}

{% block title %}Pending Requests - Leave Management{% endblock %}

{% block content %}
<div class="row mb-4">
    <div class="col-md-8">
        <h1><i class="bi bi-hourglass-split"></i> Pending Leave Requests</h1>
    </div>
    <div class="col-md-4 text-end">
        <a href="{% url 'leaves:all_leaves' %}" class="btn btn-outline-primary">
            <i class="bi bi-list"></i> View All Leaves
        </a>
    </div>
</div>

<div class="card">
    <div class="card-body">
        {% if leave_requests %}
        <div class="table-responsive">
            <table class="table table-hover">
                <thead class="table-light">
                    <tr>
                        <th>Faculty</th>
                        <th>Department</th>
                        <th>Leave Type</th>
                        <th>Period</th>
                        <th>Days</th>
                        <th>Applied On</th>
                        <th>Action</th>
                    </tr>
                </thead>
                <tbody>
                    {% for leave in leave_requests %}
                    <tr>
                        <td>
                            <strong>{{ leave.faculty.user.get_full_name }}</strong><br>
                            <small class="text-muted">{{ leave.faculty.employee_id }}</small>
                        </td>
                        <td>{{ leave.faculty.department }}</td>
                        <td>{{ leave.leave_type.name }}</td>
                        <td>
                            {{ leave.start_date|date:"M d" }} - {{ leave.end_date|date:"M d, Y" }}
                        </td>
                        <td>{{ leave.number_of_days }}</td>
                        <td>{{ leave.applied_on|date:"M d, Y" }}</td>
                        <td>
                            <a href="{% url 'leaves:review_leave' leave.id %}" class="btn btn-sm btn-primary">
                                <i class="bi bi-eye"></i> Review
                            </a>
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
        {% else %}
        <div class="text-center py-5">
            <i class="bi bi-check-circle" style="font-size: 4rem; color: #28a745;"></i>
            <p class="mt-3 text-muted">No pending leave requests!</p>
        </div>
        {% endif %}
    </div>
</div>
{% endblock %}
```

## Step 4: Create Review Leave Template

Create `leaves/templates/leaves/review_leave.html`:

```html
{% extends 'leaves/base.html' %}

{% block title %}Review Leave - Leave Management{% endblock %}

{% block content %}
<div class="row">
    <div class="col-lg-8 mx-auto">
        <div class="card">
            <div class="card-header bg-primary text-white">
                <h4 class="mb-0"><i class="bi bi-clipboard-check"></i> Review Leave Request</h4>
            </div>
            <div class="card-body">
                <!-- Leave Details -->
                <div class="alert alert-info">
                    <h5 class="alert-heading">Leave Request Details</h5>
                    <dl class="row mb-0">
                        <dt class="col-sm-4">Faculty:</dt>
                        <dd class="col-sm-8">{{ leave_request.faculty.user.get_full_name }}</dd>
                        
                        <dt class="col-sm-4">Employee ID:</dt>
                        <dd class="col-sm-8">{{ leave_request.faculty.employee_id }}</dd>
                        
                        <dt class="col-sm-4">Department:</dt>
                        <dd class="col-sm-8">{{ leave_request.faculty.department }}</dd>
                        
                        <dt class="col-sm-4">Designation:</dt>
                        <dd class="col-sm-8">{{ leave_request.faculty.designation }}</dd>
                        
                        <dt class="col-sm-4">Leave Type:</dt>
                        <dd class="col-sm-8">{{ leave_request.leave_type.name }}</dd>
                        
                        <dt class="col-sm-4">Period:</dt>
                        <dd class="col-sm-8">
                            {{ leave_request.start_date|date:"M d, Y" }} to {{ leave_request.end_date|date:"M d, Y" }}
                        </dd>
                        
                        <dt class="col-sm-4">Number of Days:</dt>
                        <dd class="col-sm-8">{{ leave_request.number_of_days }} days</dd>
                        
                        <dt class="col-sm-4">Applied On:</dt>
                        <dd class="col-sm-8">{{ leave_request.applied_on|date:"M d, Y H:i" }}</dd>
                        
                        <dt class="col-sm-4">Reason:</dt>
                        <dd class="col-sm-8">{{ leave_request.reason }}</dd>
                        
                        {% if leave_request.supporting_document %}
                        <dt class="col-sm-4">Document:</dt>
                        <dd class="col-sm-8">
                            <a href="{{ leave_request.supporting_document.url }}" target="_blank" class="btn btn-sm btn-outline-primary">
                                <i class="bi bi-download"></i> View Document
                            </a>
                        </dd>
                        {% endif %}
                    </dl>
                </div>
                
                <!-- Review Form -->
                <form method="post">
                    {% csrf_token %}
                    
                    <div class="mb-3">
                        <label for="{{ form.status.id_for_label }}" class="form-label">
                            Decision <span class="text-danger">*</span>
                        </label>
                        {{ form.status }}
                        {% if form.status.errors %}
                            <div class="text-danger">{{ form.status.errors }}</div>
                        {% endif %}
                    </div>
                    
                    <div class="mb-3">
                        <label for="{{ form.admin_remarks.id_for_label }}" class="form-label">
                            Remarks
                        </label>
                        {{ form.admin_remarks }}
                        {% if form.admin_remarks.errors %}
                            <div class="text-danger">{{ form.admin_remarks.errors }}</div>
                        {% endif %}
                        <small class="form-text text-muted">Optional remarks or feedback for the faculty</small>
                    </div>
                    
                    <div class="d-grid gap-2 d-md-flex justify-content-md-end">
                        <a href="{% url 'leaves:pending_requests' %}" class="btn btn-secondary">
                            <i class="bi bi-x-circle"></i> Cancel
                        </a>
                        <button type="submit" class="btn btn-primary">
                            <i class="bi bi-check-circle"></i> Submit Decision
                        </button>
                    </div>
                </form>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

## Step 5: Create All Leaves Template

Create `leaves/templates/leaves/all_leaves.html`:

```html
{% extends 'leaves/base.html' %}

{% block title %}All Leaves - Leave Management{% endblock %}

{% block content %}
<h1 class="mb-4"><i class="bi bi-list-check"></i> All Leave Requests</h1>

<!-- Statistics -->
<div class="row mb-4">
    <div class="col-md-3">
        <div class="card border-primary">
            <div class="card-body text-center">
                <h2>{{ total_requests }}</h2>
                <p class="mb-0">Total Requests</p>
            </div>
        </div>
    </div>
    <div class="col-md-3">
        <div class="card border-warning">
            <div class="card-body text-center">
                <h2>{{ pending_count }}</h2>
                <p class="mb-0">Pending</p>
            </div>
        </div>
    </div>
    <div class="col-md-3">
        <div class="card border-success">
            <div class="card-body text-center">
                <h2>{{ approved_count }}</h2>
                <p class="mb-0">Approved</p>
            </div>
        </div>
    </div>
    <div class="col-md-3">
        <div class="card border-danger">
            <div class="card-body text-center">
                <h2>{{ rejected_count }}</h2>
                <p class="mb-0">Rejected</p>
            </div>
        </div>
    </div>
</div>

<!-- Filters -->
<div class="card mb-4">
    <div class="card-body">
        <form method="get" class="row g-3">
            <div class="col-md-4">
                <label class="form-label">Status</label>
                <select name="status" class="form-select">
                    <option value="">All Statuses</option>
                    <option value="pending" {% if current_status == 'pending' %}selected{% endif %}>Pending</option>
                    <option value="approved" {% if current_status == 'approved' %}selected{% endif %}>Approved</option>
                    <option value="rejected" {% if current_status == 'rejected' %}selected{% endif %}>Rejected</option>
                    <option value="cancelled" {% if current_status == 'cancelled' %}selected{% endif %}>Cancelled</option>
                </select>
            </div>
            <div class="col-md-4">
                <label class="form-label">Department</label>
                <select name="department" class="form-select">
                    <option value="">All Departments</option>
                    {% for dept in departments %}
                    <option value="{{ dept }}" {% if current_department == dept %}selected{% endif %}>{{ dept }}</option>
                    {% endfor %}
                </select>
            </div>
            <div class="col-md-4">
                <label class="form-label">&nbsp;</label>
                <button type="submit" class="btn btn-primary d-block">
                    <i class="bi bi-funnel"></i> Apply Filters
                </button>
            </div>
        </form>
    </div>
</div>

<!-- Leave Requests Table -->
<div class="card">
    <div class="card-body">
        {% if leave_requests %}
        <div class="table-responsive">
            <table class="table table-hover">
                <thead class="table-light">
                    <tr>
                        <th>Faculty</th>
                        <th>Department</th>
                        <th>Leave Type</th>
                        <th>Period</th>
                        <th>Days</th>
                        <th>Status</th>
                        <th>Applied</th>
                        <th>Action</th>
                    </tr>
                </thead>
                <tbody>
                    {% for leave in leave_requests %}
                    <tr>
                        <td>
                            <strong>{{ leave.faculty.user.get_full_name }}</strong><br>
                            <small class="text-muted">{{ leave.faculty.employee_id }}</small>
                        </td>
                        <td>{{ leave.faculty.department }}</td>
                        <td>{{ leave.leave_type.name }}</td>
                        <td>{{ leave.start_date|date:"M d" }} - {{ leave.end_date|date:"M d" }}</td>
                        <td>{{ leave.number_of_days }}</td>
                        <td>
                            <span class="badge badge-{{ leave.status }}">
                                {{ leave.get_status_display }}
                            </span>
                        </td>
                        <td>{{ leave.applied_on|date:"M d, Y" }}</td>
                        <td>
                            {% if leave.status == 'pending' %}
                            <a href="{% url 'leaves:review_leave' leave.id %}" class="btn btn-sm btn-primary">
                                <i class="bi bi-eye"></i> Review
                            </a>
                            {% else %}
                            <button class="btn btn-sm btn-info" data-bs-toggle="modal" data-bs-target="#detailModal{{ leave.id }}">
                                <i class="bi bi-info-circle"></i> Details
                            </button>
                            {% endif %}
                        </td>
                    </tr>
                    
                    <!-- Detail Modal -->
                    <div class="modal fade" id="detailModal{{ leave.id }}">
                        <div class="modal-dialog">
                            <div class="modal-content">
                                <div class="modal-header">
                                    <h5 class="modal-title">Leave Details</h5>
                                    <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
                                </div>
                                <div class="modal-body">
                                    <dl class="row">
                                        <dt class="col-sm-5">Reason:</dt>
                                        <dd class="col-sm-7">{{ leave.reason }}</dd>
                                        
                                        {% if leave.reviewed_by %}
                                        <dt class="col-sm-5">Reviewed By:</dt>
                                        <dd class="col-sm-7">{{ leave.reviewed_by.get_full_name }}</dd>
                                        
                                        <dt class="col-sm-5">Reviewed On:</dt>
                                        <dd class="col-sm-7">{{ leave.reviewed_on|date:"M d, Y H:i" }}</dd>
                                        {% endif %}
                                        
                                        {% if leave.admin_remarks %}
                                        <dt class="col-sm-5">Admin Remarks:</dt>
                                        <dd class="col-sm-7">{{ leave.admin_remarks }}</dd>
                                        {% endif %}
                                    </dl>
                                </div>
                            </div>
                        </div>
                    </div>
                    {% endfor %}
                </tbody>
            </table>
        </div>
        {% else %}
        <p class="text-center text-muted">No leave requests found.</p>
        {% endif %}
    </div>
</div>
{% endblock %}
```

## Permission Decorators Explained

```python
# Must be logged in
@login_required
def view(request):
    pass

# Must pass custom test
@user_passes_test(lambda u: u.is_staff)
def admin_view(request):
    pass

# Combine decorators
from django.utils.decorators import method_decorator

@method_decorator([login_required, user_passes_test(is_staff)], name='dispatch')
class MyView(View):
    pass
```

## Next Steps

In Part 8, we'll add email notifications and advanced features.

---

*Workshop Materials | Faculty Leave Management System | Part 7 of 10*
