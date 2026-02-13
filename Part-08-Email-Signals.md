# Part 8: Email Notifications and Django Signals

## Introduction

We'll implement email notifications when leave requests are approved/rejected using Django signals.

## Step 1: Configure Email Settings

For development, add to `leave_management_system/settings.py`:

```python
# Email Configuration (Development - Console Backend)
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'

# For production with Gmail (example):
# EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
# EMAIL_HOST = 'smtp.gmail.com'
# EMAIL_PORT = 587
# EMAIL_USE_TLS = True
# EMAIL_HOST_USER = 'your-email@gmail.com'
# EMAIL_HOST_PASSWORD = 'your-app-password'
# DEFAULT_FROM_EMAIL = 'Leave Management System <noreply@example.com>'
```

## Step 2: Create Signals

Create `leaves/signals.py`:

```python
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.core.mail import send_mail
from django.conf import settings
from .models import LeaveRequest

@receiver(post_save, sender=LeaveRequest)
def leave_request_notification(sender, instance, created, **kwargs):
    """
    Send email notifications for leave requests
    """
    if created:
        # New leave request - notify admin
        send_leave_application_email(instance)
    else:
        # Leave request updated - check if status changed
        if instance.status in ['approved', 'rejected']:
            send_leave_decision_email(instance)


def send_leave_application_email(leave_request):
    """
    Notify admin when new leave request is submitted
    """
    subject = f'New Leave Request from {leave_request.faculty.user.get_full_name()}'
    
    message = f"""
    A new leave request has been submitted:
    
    Faculty: {leave_request.faculty.user.get_full_name()}
    Employee ID: {leave_request.faculty.employee_id}
    Department: {leave_request.faculty.department}
    
    Leave Type: {leave_request.leave_type.name}
    Period: {leave_request.start_date} to {leave_request.end_date}
    Number of Days: {leave_request.number_of_days}
    
    Reason: {leave_request.reason}
    
    Please review this request at your earliest convenience.
    """
    
    # Get admin emails
    from django.contrib.auth.models import User
    admin_emails = list(User.objects.filter(is_staff=True).values_list('email', flat=True))
    admin_emails = [email for email in admin_emails if email]  # Remove empty emails
    
    if admin_emails:
        send_mail(
            subject=subject,
            message=message,
            from_email=settings.DEFAULT_FROM_EMAIL if hasattr(settings, 'DEFAULT_FROM_EMAIL') else 'noreply@example.com',
            recipient_list=admin_emails,
            fail_silently=True,
        )


def send_leave_decision_email(leave_request):
    """
    Notify faculty when their leave request is approved/rejected
    """
    faculty_email = leave_request.faculty.user.email
    
    if not faculty_email:
        return
    
    status_text = 'APPROVED' if leave_request.status == 'approved' else 'REJECTED'
    
    subject = f'Leave Request {status_text}'
    
    message = f"""
    Dear {leave_request.faculty.user.get_full_name()},
    
    Your leave request has been {status_text.lower()}.
    
    Leave Details:
    - Leave Type: {leave_request.leave_type.name}
    - Period: {leave_request.start_date} to {leave_request.end_date}
    - Number of Days: {leave_request.number_of_days}
    
    Reviewed By: {leave_request.reviewed_by.get_full_name() if leave_request.reviewed_by else 'Admin'}
    Reviewed On: {leave_request.reviewed_on}
    """
    
    if leave_request.admin_remarks:
        message += f"\nRemarks: {leave_request.admin_remarks}"
    
    message += "\n\nThank you,\nLeave Management System"
    
    send_mail(
        subject=subject,
        message=message,
        from_email=settings.DEFAULT_FROM_EMAIL if hasattr(settings, 'DEFAULT_FROM_EMAIL') else 'noreply@example.com',
        recipient_list=[faculty_email],
        fail_silently=True,
    )
```

## Step 3: Register Signals

Create/update `leaves/apps.py`:

```python
from django.apps import AppConfig


class LeavesConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'leaves'

    def ready(self):
        """Import signals when app is ready"""
        import leaves.signals
```

Make sure your app config is used in `leaves/__init__.py`:

```python
default_app_config = 'leaves.apps.LeavesConfig'
```

## Step 4: Create Email Templates (Optional - HTML emails)

Create `leaves/utils.py` for HTML email support:

```python
from django.core.mail import EmailMultiAlternatives
from django.template.loader import render_to_string
from django.conf import settings


def send_html_email(subject, template_name, context, recipient_list):
    """
    Send HTML email with text fallback
    """
    # Render HTML email
    html_content = render_to_string(f'leaves/emails/{template_name}.html', context)
    text_content = render_to_string(f'leaves/emails/{template_name}.txt', context)
    
    email = EmailMultiAlternatives(
        subject=subject,
        body=text_content,
        from_email=settings.DEFAULT_FROM_EMAIL if hasattr(settings, 'DEFAULT_FROM_EMAIL') else 'noreply@example.com',
        to=recipient_list
    )
    
    email.attach_alternative(html_content, "text/html")
    email.send(fail_silently=True)
```

## Understanding Django Signals

Signals allow decoupled applications to get notified when certain actions occur.

### Common Signals

```python
from django.db.models.signals import (
    pre_save,    # Before model.save()
    post_save,   # After model.save()
    pre_delete,  # Before model.delete()
    post_delete, # After model.delete()
    m2m_changed, # When ManyToMany changes
)

from django.contrib.auth.signals import (
    user_logged_in,
    user_logged_out,
    user_login_failed,
)
```

### Creating a Signal Receiver

```python
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=MyModel)
def my_handler(sender, instance, created, **kwargs):
    """
    sender: Model class
    instance: Actual instance being saved
    created: True if new record, False if update
    """
    if created:
        # Do something with new instance
        pass
```

### Manual Signal Connection

```python
# Without decorator
def my_handler(sender, instance, created, **kwargs):
    pass

post_save.connect(my_handler, sender=MyModel)
```

## Step 5: Add Notification Preferences

Add to `leaves/models.py`:

```python
class FacultyProfile(models.Model):
    # ... existing fields ...
    
    # Notification preferences
    email_notifications = models.BooleanField(default=True, help_text="Receive email notifications")
    
    def __str__(self):
        return f"{self.user.get_full_name()} ({self.employee_id})"
```

Update signals to check preferences:

```python
def send_leave_decision_email(leave_request):
    """Notify faculty - respect preferences"""
    if not leave_request.faculty.email_notifications:
        return  # User opted out
    
    # ... rest of email code ...
```

## Step 6: Add Management Command for Testing

Create `leaves/management/commands/send_test_email.py`:

```python
from django.core.management.base import BaseCommand
from django.core.mail import send_mail

class Command(BaseCommand):
    help = 'Send a test email'

    def add_arguments(self, parser):
        parser.add_argument('email', type=str, help='Recipient email address')

    def handle(self, *args, **options):
        recipient = options['email']
        
        send_mail(
            subject='Test Email from Leave Management System',
            message='This is a test email. If you receive this, email configuration is working!',
            from_email='noreply@example.com',
            recipient_list=[recipient],
            fail_silently=False,
        )
        
        self.stdout.write(self.style.SUCCESS(f'Successfully sent test email to {recipient}'))
```

Run the command:
```bash
python manage.py send_test_email your-email@example.com
```

## Step 7: Custom Signals (Advanced)

Create custom signals in `leaves/signals.py`:

```python
from django.dispatch import Signal

# Custom signals
leave_approved = Signal()  # providing_args=['leave_request', 'approved_by']
leave_rejected = Signal()  # providing_args=['leave_request', 'rejected_by']

# In your view:
from .signals import leave_approved, leave_rejected

def review_leave(request, leave_id):
    # ... your code ...
    
    if leave.status == 'approved':
        leave_approved.send(sender=LeaveRequest, leave_request=leave, approved_by=request.user)
    elif leave.status == 'rejected':
        leave_rejected.send(sender=LeaveRequest, leave_request=leave, rejected_by=request.user)
```

## Email Templates Example

Create `leaves/templates/leaves/emails/leave_approved.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; }
        .container { max-width: 600px; margin: 0 auto; padding: 20px; }
        .header { background: #28a745; color: white; padding: 20px; text-align: center; }
        .content { padding: 20px; background: #f9f9f9; }
        .footer { text-align: center; padding: 20px; color: #666; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>Leave Request Approved</h1>
        </div>
        <div class="content">
            <p>Dear {{ leave_request.faculty.user.get_full_name }},</p>
            
            <p>Your leave request has been <strong>approved</strong>.</p>
            
            <table>
                <tr><td><strong>Leave Type:</strong></td><td>{{ leave_request.leave_type.name }}</td></tr>
                <tr><td><strong>Period:</strong></td><td>{{ leave_request.start_date }} to {{ leave_request.end_date }}</td></tr>
                <tr><td><strong>Days:</strong></td><td>{{ leave_request.number_of_days }}</td></tr>
            </table>
            
            {% if leave_request.admin_remarks %}
            <p><strong>Remarks:</strong> {{ leave_request.admin_remarks }}</p>
            {% endif %}
        </div>
        <div class="footer">
            <p>Leave Management System</p>
        </div>
    </div>
</body>
</html>
```

## Testing Signals

```python
# In Django shell
from leaves.models import LeaveRequest
from datetime import date

# Create a leave request (will trigger signal)
leave = LeaveRequest.objects.create(
    faculty=faculty,
    leave_type=leave_type,
    start_date=date(2026, 3, 1),
    end_date=date(2026, 3, 3),
    reason="Test"
)

# Update status (will trigger signal)
leave.status = 'approved'
leave.save()
```

## Best Practices

1. **Use fail_silently=True** in production to prevent email errors from breaking your app
2. **Queue emails** for large batches (use Celery)
3. **Test email templates** thoroughly
4. **Respect user preferences** for notifications
5. **Log email sending** for debugging
6. **Use email confirmation** for important actions

## Common Issues

### Emails not sending
- Check EMAIL_BACKEND setting
- Verify EMAIL_HOST and credentials
- Check spam folder
- Use console backend for testing

### Gmail not working
- Enable "Less secure app access" (not recommended)
- Use App Password with 2FA enabled (recommended)
- Check firewall/port 587

## Next Steps

In Part 9, we'll implement advanced features: reporting, export, and analytics.

---

*Workshop Materials | Faculty Leave Management System | Part 8 of 10*
