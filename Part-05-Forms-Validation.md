# Part 5: Forms and Validation

## Introduction

In this part, we'll create Django forms for leave applications, user registration, and learn about form validation.

## Django Forms

Django forms handle:
- **Rendering** HTML form fields
- **Validation** of user input
- **Cleaning** and normalizing data
- **Security** (CSRF protection)

### Two Types of Forms

**Form**: Basic form from scratch
```python
from django import forms

class ContactForm(forms.Form):
    name = forms.CharField(max_length=100)
    email = forms.EmailField()
```

**ModelForm**: Automatically created from a model
```python
from django import forms
from .models import LeaveRequest

class LeaveRequestForm(forms.ModelForm):
    class Meta:
        model = LeaveRequest
        fields = ['leave_type', 'start_date', 'end_date', 'reason']
```

## Step 1: Create Forms File

Create `leaves/forms.py`:

```python
from django import forms
from django.contrib.auth.models import User
from django.contrib.auth.forms import UserCreationForm
from django.core.exceptions import ValidationError
from .models import LeaveRequest, FacultyProfile
from datetime import date
from django import forms


class ProfileUpdateForm(forms.ModelForm):

    first_name = forms.CharField(max_length=150)
    last_name = forms.CharField(max_length=150)
    email = forms.EmailField()

    class Meta:
        model = FacultyProfile
        fields = [
            "first_name",
            "last_name",
            "email",
            "department",
            "designation",
            "phone_number",
            "profile_picture",
        ]

    def __init__(self, *args, **kwargs):
        self.user = kwargs.pop("user")
        super().__init__(*args, **kwargs)

        # Prefill user data
        self.fields["first_name"].initial = self.user.first_name
        self.fields["last_name"].initial = self.user.last_name
        self.fields["email"].initial = self.user.email

        for field in self.fields:
            self.fields[field].widget.attrs.update({
                "class": "form-control"
            })

    def save(self, commit=True):
        faculty = super().save(commit=False)

        # Update User model
        self.user.first_name = self.cleaned_data["first_name"]
        self.user.last_name = self.cleaned_data["last_name"]
        self.user.email = self.cleaned_data["email"]
        self.user.save()

        if commit:
            faculty.save()

        return faculty

class LeaveRequestForm(forms.ModelForm):
    """
    Form for faculty to apply for leave
    """
    class Meta:
        model = LeaveRequest
        fields = ['leave_type', 'start_date', 'end_date', 'reason', 'supporting_document']
        widgets = {
            'start_date': forms.DateInput(attrs={
                'type': 'date',
                'class': 'form-control',
                'min': date.today().isoformat()
            }),
            'end_date': forms.DateInput(attrs={
                'type': 'date', 
                'class': 'form-control',
                'min': date.today().isoformat()
            }),
            'reason': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 4,
                'placeholder': 'Please provide reason for leave...'
            }),
            'leave_type': forms.Select(attrs={
                'class': 'form-select'
            }),
            'supporting_document': forms.FileInput(attrs={
                'class': 'form-control'
            })
        }
        labels = {
            'leave_type': 'Type of Leave',
            'start_date': 'From Date',
            'end_date': 'To Date',
            'reason': 'Reason for Leave',
            'supporting_document': 'Supporting Document (if any)'
        }
        help_texts = {
            'start_date': 'Leave start date',
            'end_date': 'Leave end date',
            'supporting_document': 'Upload medical certificate or other documents (PDF, JPG, PNG - Max 5MB)'
        }
    
    def clean(self):
        """
        Custom validation for the entire form
        """
        cleaned_data = super().clean()
        start_date = cleaned_data.get('start_date')
        end_date = cleaned_data.get('end_date')
        leave_type = cleaned_data.get('leave_type')
        
        # Validate dates
        if start_date and end_date:
            # Check if end date is after start date
            if end_date < start_date:
                raise ValidationError('End date must be after start date.')
            
            # Check if dates are not in the past
            if start_date < date.today():
                raise ValidationError('Cannot apply for leave in the past.')
            
            # Calculate number of days
            delta = end_date - start_date
            number_of_days = delta.days + 1
            
            # Check if exceeds maximum days for leave type
            if leave_type and number_of_days > leave_type.max_days_per_year:
                raise ValidationError(
                    f'{leave_type.name} allows maximum {leave_type.max_days_per_year} days. '
                    f'You are requesting {number_of_days} days.'
                )
        
        return cleaned_data
    
    def clean_supporting_document(self):
        """
        Validate uploaded document
        """
        document = self.cleaned_data.get('supporting_document')
        
        if document:
            # Check file size (5MB limit)
            if document.size > 5 * 1024 * 1024:
                raise ValidationError('File size cannot exceed 5MB.')
            
            # Check file extension
            allowed_extensions = ['.pdf', '.jpg', '.jpeg', '.png']
            file_extension = document.name[document.name.rfind('.'):].lower()
            
            if file_extension not in allowed_extensions:
                raise ValidationError(
                    f'Only {", ".join(allowed_extensions)} files are allowed.'
                )
        
        return document


class LeaveReviewForm(forms.ModelForm):
    """
    Form for admin to review leave requests
    """
    class Meta:
        model = LeaveRequest
        fields = ['status', 'admin_remarks']
        widgets = {
            'status': forms.Select(attrs={
                'class': 'form-select'
            }),
            'admin_remarks': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 3,
                'placeholder': 'Add remarks (optional)...'
            })
        }
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # Limit status choices to approved/rejected only
        self.fields['status'].choices = [
            ('approved', 'Approved'),
            ('rejected', 'Rejected'),
        ]


class FacultyRegistrationForm(UserCreationForm):
    """
    Extended registration form for faculty
    """
    # User fields
    email = forms.EmailField(
        required=True,
        widget=forms.EmailInput(attrs={'class': 'form-control'})
    )
    first_name = forms.CharField(
        max_length=30,
        required=True,
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    last_name = forms.CharField(
        max_length=30,
        required=True,
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    
    # Faculty profile fields
    employee_id = forms.CharField(
        max_length=20,
        required=True,
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    department = forms.CharField(
        max_length=100,
        required=True,
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    designation = forms.CharField(
        max_length=100,
        required=True,
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    phone_number = forms.CharField(
        max_length=15,
        required=False,
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    date_of_joining = forms.DateField(
        required=True,
        widget=forms.DateInput(attrs={
            'type': 'date',
            'class': 'form-control'
        })
    )
    profile_picture = forms.ImageField(
        required=False,
        widget=forms.FileInput(attrs={'class': 'form-control'})
    )
    
    class Meta:
        model = User
        fields = ['username', 'email', 'first_name', 'last_name', 'password1', 'password2']
        widgets = {
            'username': forms.TextInput(attrs={'class': 'form-control'}),
        }
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

        # Only pre-fill if instance is FacultyProfile
        if isinstance(self.instance, FacultyProfile) and self.instance.pk:
            if self.instance.user:
                self.fields['first_name'].initial = self.instance.user.first_name
                self.fields['last_name'].initial = self.instance.user.last_name
                self.fields['email'].initial = self.instance.user.email
    
    def clean_email(self):
        """
        Validate that email is unique
        """
        email = self.cleaned_data.get('email')
        if User.objects.filter(email=email).exists():
            raise ValidationError('This email is already registered.')
        return email
    
    def clean_employee_id(self):
        """
        Validate that employee ID is unique
        """
        employee_id = self.cleaned_data.get('employee_id')
        if FacultyProfile.objects.filter(employee_id=employee_id).exists():
            raise ValidationError('This employee ID is already registered.')
        return employee_id


class FacultyProfileUpdateForm(forms.ModelForm):
    """
    Form to update faculty profile
    """
    # Include User fields
    first_name = forms.CharField(
        max_length=30,
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    last_name = forms.CharField(
        max_length=30,
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    email = forms.EmailField(
        widget=forms.EmailInput(attrs={'class': 'form-control'})
    )
    
    class Meta:
        model = FacultyProfile
        fields = ['department', 'designation', 'phone_number', 'profile_picture']
        widgets = {
            'department': forms.TextInput(attrs={'class': 'form-control'}),
            'designation': forms.TextInput(attrs={'class': 'form-control'}),
            'phone_number': forms.TextInput(attrs={'class': 'form-control'}),
            'profile_picture': forms.FileInput(attrs={'class': 'form-control'}),
        }
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # Pre-populate user fields if instance exists
        if self.instance and self.instance.user:
            self.fields['first_name'].initial = self.instance.user.first_name
            self.fields['last_name'].initial = self.instance.user.last_name
            self.fields['email'].initial = self.instance.user.email
```

## Understanding Form Components

### Form Fields

Common field types:
```python
CharField()           # Text input
EmailField()         # Email validation
IntegerField()       # Numbers only
DecimalField()       # Decimal numbers
DateField()          # Date picker
DateTimeField()      # Date and time
BooleanField()       # Checkbox
ChoiceField()        # Dropdown
FileField()          # File upload
ImageField()         # Image upload
```

### Field Arguments

```python
field = forms.CharField(
    max_length=100,        # Maximum length
    min_length=3,          # Minimum length
    required=True,         # Must be filled
    label='Name',          # Display label
    initial='Default',     # Default value
    help_text='Help',      # Help text
    widget=forms.TextInput(attrs={'class': 'form-control'}),
    validators=[...]       # Custom validators
)
```

### Widgets

Widgets control how fields are rendered:

```python
forms.TextInput(attrs={'class': 'form-control'})
forms.Textarea(attrs={'rows': 4})
forms.Select(attrs={'class': 'form-select'})
forms.CheckboxInput(attrs={'class': 'form-check-input'})
forms.DateInput(attrs={'type': 'date'})
forms.FileInput(attrs={'accept': '.pdf,.jpg'})
```

## Step 2: Create Leave Application View

Update `leaves/views.py`:

```python
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from .models import LeaveRequest, LeaveBalance, FacultyProfile
from .forms import LeaveRequestForm, LeaveReviewForm
from datetime import datetime

@login_required
def apply_leave(request):

    if request.user.is_staff:
        return redirect('leaves:pending_requests')

    try:
        faculty = request.user.faculty_profile
    except FacultyProfile.DoesNotExist:
        return redirect('leaves:register')

    if request.method == 'POST':
        form = LeaveRequestForm(request.POST, request.FILES)
        if form.is_valid():
            leave = form.save(commit=False)
            leave.faculty = faculty
            leave.save()
            messages.success(request, "Leave applied successfully.")
            return redirect('leaves:dashboard')
    else:
        form = LeaveRequestForm()

    return render(request, 'leaves/apply_leave.html', {'form': form})


@login_required
def leave_history(request):
    faculty_profile = FacultyProfile.objects.get(user=request.user)

    leave_requests = LeaveRequest.objects.filter(
        faculty=faculty_profile
    ).order_by('-applied_on')

    return render(request, 'leaves/leave_history.html', {
        'leave_requests': leave_requests
    })
```

## Step 3: Create Leave Application Template

Create `leaves/templates/leaves/apply_leave.html`:

```html
{% extends 'leaves/base.html' %}

{% block content %}
<div class="container mt-5">
    <div class="card shadow-lg mb-4">
        <div class="card-header" style="background: linear-gradient(90deg, #0d6efd, #0b5ed7); color: white;">
            <h4 class="mb-0">Apply for Leave</h4>
        </div>

        <div class="card-body">

            <form method="POST" enctype="multipart/form-data">
                {% csrf_token %}

                <!-- Show form errors -->
                {% if form.errors %}
                <div class="alert alert-danger">
                    Please correct the errors below.
                    {{ form.errors }}
                </div>
                {% endif %}

                <div class="row">

                    <div class="col-md-6 mb-3">
                        {{ form.leave_type.label_tag }}
                        {{ form.leave_type }}
                    </div>

                    <div class="col-md-6 mb-3">
                        {{ form.start_date.label_tag }}
                        {{ form.start_date }}
                    </div>

                    <div class="col-md-6 mb-3">
                        {{ form.end_date.label_tag }}
                        {{ form.end_date }}
                    </div>

                    <div class="col-md-12 mb-3">
                        {{ form.reason.label_tag }}
                        {{ form.reason }}
                    </div>

                    <div class="col-md-12 mb-3">
                        {{ form.supporting_document.label_tag }}
                        {{ form.supporting_document }}
                    </div>

                </div>

                <div class="text-end">
                    <button type="submit" class="btn btn-success">
                        Submit Application
                    </button>
                </div>

            </form>

        </div>
    </div>
</div>
{% endblock %}
```

## Step 4: Create Leave History Template

Create `leaves/templates/leaves/leave_history.html`:

```html
{% extends 'leaves/base.html' %}

{% block title %}Leave History - Leave Management{% endblock %}

{% block content %}
<div class="row mb-4">
    <div class="col-md-8">
        <h1><i class="bi bi-clock-history"></i> My Leave History</h1>
    </div>
    <div class="col-md-4 text-end">
        <a href="{% url 'leaves:apply_leave' %}" class="btn btn-primary">
            <i class="bi bi-plus-circle"></i> Apply New Leave
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
                        <th>Leave Type</th>
                        <th>Start Date</th>
                        <th>End Date</th>
                        <th>Days</th>
                        <th>Status</th>
                        <th>Applied On</th>
                        <th>Action</th>
                    </tr>
                </thead>
                <tbody>
                    {% for leave in leave_requests %}
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
                        <td>{{ leave.applied_on|date:"M d, Y H:i" }}</td>
                        <td>
                            <button type="button" class="btn btn-sm btn-info" 
                                    data-bs-toggle="modal" 
                                    data-bs-target="#leaveModal{{ leave.id }}">
                                <i class="bi bi-eye"></i> View
                            </button>
                        </td>
                    </tr>
                    
                    <!-- Modal for leave details -->
                    <div class="modal fade" id="leaveModal{{ leave.id }}" tabindex="-1">
                        <div class="modal-dialog">
                            <div class="modal-content">
                                <div class="modal-header">
                                    <h5 class="modal-title">Leave Request Details</h5>
                                    <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
                                </div>
                                <div class="modal-body">
                                    <dl class="row">
                                        <dt class="col-sm-4">Leave Type:</dt>
                                        <dd class="col-sm-8">{{ leave.leave_type.name }}</dd>
                                        
                                        <dt class="col-sm-4">Start Date:</dt>
                                        <dd class="col-sm-8">{{ leave.start_date|date:"M d, Y" }}</dd>
                                        
                                        <dt class="col-sm-4">End Date:</dt>
                                        <dd class="col-sm-8">{{ leave.end_date|date:"M d, Y" }}</dd>
                                        
                                        <dt class="col-sm-4">Number of Days:</dt>
                                        <dd class="col-sm-8">{{ leave.number_of_days }}</dd>
                                        
                                        <dt class="col-sm-4">Reason:</dt>
                                        <dd class="col-sm-8">{{ leave.reason }}</dd>
                                        
                                        <dt class="col-sm-4">Status:</dt>
                                        <dd class="col-sm-8">
                                            <span class="badge badge-{{ leave.status }}">
                                                {{ leave.get_status_display }}
                                            </span>
                                        </dd>
                                        
                                        <dt class="col-sm-4">Applied On:</dt>
                                        <dd class="col-sm-8">{{ leave.applied_on|date:"M d, Y H:i" }}</dd>
                                        
                                        {% if leave.reviewed_by %}
                                        <dt class="col-sm-4">Reviewed By:</dt>
                                        <dd class="col-sm-8">{{ leave.reviewed_by.get_full_name }}</dd>
                                        
                                        <dt class="col-sm-4">Reviewed On:</dt>
                                        <dd class="col-sm-8">{{ leave.reviewed_on|date:"M d, Y H:i" }}</dd>
                                        {% endif %}
                                        
                                        {% if leave.admin_remarks %}
                                        <dt class="col-sm-4">Remarks:</dt>
                                        <dd class="col-sm-8">{{ leave.admin_remarks }}</dd>
                                        {% endif %}
                                        
                                        {% if leave.supporting_document %}
                                        <dt class="col-sm-4">Document:</dt>
                                        <dd class="col-sm-8">
                                            <a href="{{ leave.supporting_document.url }}" target="_blank" class="btn btn-sm btn-outline-primary">
                                                <i class="bi bi-download"></i> Download
                                            </a>
                                        </dd>
                                        {% endif %}
                                    </dl>
                                </div>
                                <div class="modal-footer">
                                    <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
                                </div>
                            </div>
                        </div>
                    </div>
                    {% endfor %}
                </tbody>
            </table>
        </div>
        {% else %}
        <div class="text-center py-5">
            <i class="bi bi-inbox" style="font-size: 4rem; color: #ccc;"></i>
            <p class="mt-3 text-muted">No leave requests yet.</p>
            <a href="{% url 'leaves:apply_leave' %}" class="btn btn-primary">
                <i class="bi bi-plus-circle"></i> Apply for Your First Leave
            </a>
        </div>
        {% endif %}
    </div>
</div>
{% endblock %}
```

## Form Validation Methods

### Field-level Validation

```python
def clean_fieldname(self):
    data = self.cleaned_data['fieldname']
    # Validate
    if condition:
        raise ValidationError('Error message')
    return data
```

### Form-level Validation

```python
def clean(self):
    cleaned_data = super().clean()
    field1 = cleaned_data.get('field1')
    field2 = cleaned_data.get('field2')
    # Cross-field validation
    if field1 and field2:
        if condition:
            raise ValidationError('Error message')
    return cleaned_data
```

## Testing Forms in Django Shell

```python
python manage.py shell

from leaves.forms import LeaveRequestForm
from leaves.models import LeaveType
from datetime import date

# Get leave type
leave_type = LeaveType.objects.first()

# Test form with data
form_data = {
    'leave_type': leave_type.id,
    'start_date': date(2026, 3, 1),
    'end_date': date(2026, 3, 3),
    'reason': 'Personal work'
}

form = LeaveRequestForm(data=form_data)
print(form.is_valid())
print(form.errors)
print(form.cleaned_data)
```

## Common Form Patterns

### Custom Validator

```python
from django.core.exceptions import ValidationError

def validate_even(value):
    if value % 2 != 0:
        raise ValidationError(f'{value} is not an even number')

class MyForm(forms.Form):
    number = forms.IntegerField(validators=[validate_even])
```

### Dynamic Choices

```python
def __init__(self, *args, **kwargs):
    super().__init__(*args, **kwargs)
    self.fields['leave_type'].queryset = LeaveType.objects.filter(is_active=True)
```

## Next Steps

In Part 6, we'll:
- Create user authentication (login/logout/register)
- Protect views with decorators
- Handle user sessions
- Create password reset functionality

## Checklist

Before moving to Part 6, ensure you:
- [ ] Created forms.py with all forms
- [ ] Created apply_leave view and template
- [ ] Created leave_history view and template
- [ ] Can submit leave applications
- [ ] Form validation works correctly
- [ ] File uploads work properly
- [ ] Error messages display correctly

---

*Workshop Materials | Faculty Leave Management System | Part 5 of 10*
