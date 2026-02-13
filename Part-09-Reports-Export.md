# Part 9: Reports, Analytics, and Data Export

## Introduction

We'll add reporting features, data visualization, and export capabilities (CSV, PDF).

## Step 1: Install Required Packages

```bash
pip install reportlab  # For PDF generation
pip install openpyxl   # For Excel export
```

## Step 2: Create Reports Views

Add to `leaves/views.py`:

```python
from django.http import HttpResponse
from django.contrib.auth.decorators import login_required, user_passes_test
from django.db.models import Count, Q, Sum
from django.utils import timezone
import csv
from datetime import datetime, timedelta

@login_required
@user_passes_test(lambda u: u.is_staff)
def reports(request):
    """Generate leave reports and analytics"""
    current_year = datetime.now().year
    
    # Leave statistics by type
    leave_by_type = LeaveRequest.objects.filter(
        start_date__year=current_year
    ).values('leave_type__name').annotate(
        total=Count('id'),
        approved=Count('id', filter=Q(status='approved')),
        rejected=Count('id', filter=Q(status='rejected')),
        pending=Count('id', filter=Q(status='pending'))
    )
    
    # Leave statistics by department
    leave_by_dept = LeaveRequest.objects.filter(
        start_date__year=current_year
    ).values('faculty__department').annotate(
        total=Count('id'),
        total_days=Sum('number_of_days')
    ).order_by('-total')
    
    # Monthly leave trend
    monthly_leaves = []
    for month in range(1, 13):
        count = LeaveRequest.objects.filter(
            start_date__year=current_year,
            start_date__month=month,
            status='approved'
        ).count()
        monthly_leaves.append(count)
    
    # Top leave requesters
    top_requesters = LeaveRequest.objects.filter(
        start_date__year=current_year
    ).values(
        'faculty__user__first_name',
        'faculty__user__last_name',
        'faculty__department'
    ).annotate(
        total_requests=Count('id'),
        total_days=Sum('number_of_days')
    ).order_by('-total_requests')[:10]
    
    context = {
        'leave_by_type': leave_by_type,
        'leave_by_dept': leave_by_dept,
        'monthly_leaves': monthly_leaves,
        'top_requesters': top_requesters,
        'current_year': current_year,
    }
    return render(request, 'leaves/reports.html', context)


@login_required
@user_passes_test(lambda u: u.is_staff)
def export_leaves_csv(request):
    """Export leave requests to CSV"""
    # Get filter parameters
    status = request.GET.get('status', '')
    department = request.GET.get('department', '')
    year = request.GET.get('year', datetime.now().year)
    
    # Build queryset
    leaves = LeaveRequest.objects.filter(start_date__year=year)
    if status:
        leaves = leaves.filter(status=status)
    if department:
        leaves = leaves.filter(faculty__department=department)
    
    # Create CSV response
    response = HttpResponse(content_type='text/csv')
    response['Content-Disposition'] = f'attachment; filename="leave_requests_{year}.csv"'
    
    writer = csv.writer(response)
    writer.writerow([
        'Faculty Name', 'Employee ID', 'Department', 'Leave Type',
        'Start Date', 'End Date', 'Days', 'Status', 'Applied On',
        'Reviewed By', 'Reviewed On', 'Remarks'
    ])
    
    for leave in leaves:
        writer.writerow([
            leave.faculty.user.get_full_name(),
            leave.faculty.employee_id,
            leave.faculty.department,
            leave.leave_type.name,
            leave.start_date,
            leave.end_date,
            leave.number_of_days,
            leave.get_status_display(),
            leave.applied_on.strftime('%Y-%m-%d %H:%M'),
            leave.reviewed_by.get_full_name() if leave.reviewed_by else '',
            leave.reviewed_on.strftime('%Y-%m-%d %H:%M') if leave.reviewed_on else '',
            leave.admin_remarks or ''
        ])
    
    return response


@login_required
@user_passes_test(lambda u: u.is_staff)
def export_leaves_pdf(request):
    """Export leave summary to PDF"""
    from reportlab.lib import colors
    from reportlab.lib.pagesizes import letter, A4
    from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph, Spacer
    from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
    from reportlab.lib.units import inch
    from io import BytesIO
    
    # Create PDF buffer
    buffer = BytesIO()
    doc = SimpleDocTemplate(buffer, pagesize=A4)
    elements = []
    styles = getSampleStyleSheet()
    
    # Title
    title_style = ParagraphStyle(
        'CustomTitle',
        parent=styles['Heading1'],
        fontSize=24,
        textColor=colors.HexColor('#0d6efd'),
        spaceAfter=30,
        alignment=1  # Center
    )
    elements.append(Paragraph('Leave Management Report', title_style))
    elements.append(Paragraph(f'Generated on: {datetime.now().strftime("%B %d, %Y")}', styles['Normal']))
    elements.append(Spacer(1, 0.5*inch))
    
    # Statistics
    current_year = datetime.now().year
    total = LeaveRequest.objects.filter(start_date__year=current_year).count()
    pending = LeaveRequest.objects.filter(start_date__year=current_year, status='pending').count()
    approved = LeaveRequest.objects.filter(start_date__year=current_year, status='approved').count()
    rejected = LeaveRequest.objects.filter(start_date__year=current_year, status='rejected').count()
    
    stats_data = [
        ['Statistics', 'Count'],
        ['Total Requests', str(total)],
        ['Pending', str(pending)],
        ['Approved', str(approved)],
        ['Rejected', str(rejected)],
    ]
    
    stats_table = Table(stats_data, colWidths=[3*inch, 1.5*inch])
    stats_table.setStyle(TableStyle([
        ('BACKGROUND', (0, 0), (-1, 0), colors.grey),
        ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
        ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
        ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
        ('FONTSIZE', (0, 0), (-1, 0), 14),
        ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
        ('BACKGROUND', (0, 1), (-1, -1), colors.beige),
        ('GRID', (0, 0), (-1, -1), 1, colors.black)
    ]))
    
    elements.append(stats_table)
    elements.append(Spacer(1, 0.5*inch))
    
    # Department-wise summary
    elements.append(Paragraph('Department-wise Leave Summary', styles['Heading2']))
    elements.append(Spacer(1, 0.2*inch))
    
    dept_data = [['Department', 'Total Requests', 'Total Days']]
    dept_summary = LeaveRequest.objects.filter(
        start_date__year=current_year
    ).values('faculty__department').annotate(
        total=Count('id'),
        days=Sum('number_of_days')
    ).order_by('-total')
    
    for dept in dept_summary:
        dept_data.append([
            dept['faculty__department'],
            str(dept['total']),
            str(dept['days'])
        ])
    
    dept_table = Table(dept_data, colWidths=[2.5*inch, 1.5*inch, 1.5*inch])
    dept_table.setStyle(TableStyle([
        ('BACKGROUND', (0, 0), (-1, 0), colors.grey),
        ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
        ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
        ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
        ('FONTSIZE', (0, 0), (-1, 0), 12),
        ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
        ('BACKGROUND', (0, 1), (-1, -1), colors.lightblue),
        ('GRID', (0, 0), (-1, -1), 1, colors.black)
    ]))
    
    elements.append(dept_table)
    
    # Build PDF
    doc.build(elements)
    
    # Return PDF
    pdf = buffer.getvalue()
    buffer.close()
    
    response = HttpResponse(content_type='application/pdf')
    response['Content-Disposition'] = f'attachment; filename="leave_report_{current_year}.pdf"'
    response.write(pdf)
    
    return response


@login_required
def my_leave_report(request):
    """Personal leave report for faculty"""
    try:
        faculty = request.user.faculty_profile
    except FacultyProfile.DoesNotExist:
        messages.error(request, 'Profile not found.')
        return redirect('leaves:home')
    
    current_year = datetime.now().year
    
    # Personal statistics
    total_leaves = LeaveRequest.objects.filter(
        faculty=faculty,
        start_date__year=current_year
    ).count()
    
    approved_leaves = LeaveRequest.objects.filter(
        faculty=faculty,
        start_date__year=current_year,
        status='approved'
    ).aggregate(total=Sum('number_of_days'))['total'] or 0
    
    pending_leaves = LeaveRequest.objects.filter(
        faculty=faculty,
        status='pending'
    ).count()
    
    # Leave balance
    leave_balances = LeaveBalance.objects.filter(
        faculty=faculty,
        year=current_year
    )
    
    # Monthly breakdown
    monthly_data = []
    for month in range(1, 13):
        count = LeaveRequest.objects.filter(
            faculty=faculty,
            start_date__year=current_year,
            start_date__month=month,
            status='approved'
        ).aggregate(days=Sum('number_of_days'))['days'] or 0
        monthly_data.append({
            'month': datetime(current_year, month, 1).strftime('%B'),
            'days': count
        })
    
    context = {
        'total_leaves': total_leaves,
        'approved_leaves': approved_leaves,
        'pending_leaves': pending_leaves,
        'leave_balances': leave_balances,
        'monthly_data': monthly_data,
        'current_year': current_year,
    }
    return render(request, 'leaves/my_leave_report.html', context)
```

## Step 3: Create Reports Template

Create `leaves/templates/leaves/reports.html`:

```html
{% extends 'leaves/base.html' %}

{% block title %}Reports & Analytics - Leave Management{% endblock %}

{% block extra_css %}
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
{% endblock %}

{% block content %}
<div class="row mb-4">
    <div class="col-md-8">
        <h1><i class="bi bi-graph-up"></i> Leave Reports & Analytics</h1>
    </div>
    <div class="col-md-4 text-end">
        <div class="btn-group">
            <a href="{% url 'leaves:export_leaves_csv' %}?year={{ current_year }}" class="btn btn-success">
                <i class="bi bi-file-earmark-spreadsheet"></i> Export CSV
            </a>
            <a href="{% url 'leaves:export_leaves_pdf' %}" class="btn btn-danger">
                <i class="bi bi-file-earmark-pdf"></i> Export PDF
            </a>
        </div>
    </div>
</div>

<!-- Leave by Type -->
<div class="row mb-4">
    <div class="col-12">
        <div class="card">
            <div class="card-header">
                <h5 class="mb-0">Leave Requests by Type ({{ current_year }})</h5>
            </div>
            <div class="card-body">
                <canvas id="leaveTypeChart"></canvas>
            </div>
        </div>
    </div>
</div>

<!-- Leave by Department -->
<div class="row mb-4">
    <div class="col-md-6">
        <div class="card">
            <div class="card-header">
                <h5 class="mb-0">Leave by Department</h5>
            </div>
            <div class="card-body">
                <table class="table table-sm">
                    <thead>
                        <tr>
                            <th>Department</th>
                            <th>Requests</th>
                            <th>Total Days</th>
                        </tr>
                    </thead>
                    <tbody>
                        {% for dept in leave_by_dept %}
                        <tr>
                            <td>{{ dept.faculty__department }}</td>
                            <td>{{ dept.total }}</td>
                            <td>{{ dept.total_days }}</td>
                        </tr>
                        {% endfor %}
                    </tbody>
                </table>
            </div>
        </div>
    </div>
    
    <div class="col-md-6">
        <div class="card">
            <div class="card-header">
                <h5 class="mb-0">Monthly Trend</h5>
            </div>
            <div class="card-body">
                <canvas id="monthlyChart"></canvas>
            </div>
        </div>
    </div>
</div>

<!-- Top Requesters -->
<div class="row">
    <div class="col-12">
        <div class="card">
            <div class="card-header">
                <h5 class="mb-0">Top Leave Requesters</h5>
            </div>
            <div class="card-body">
                <table class="table table-striped">
                    <thead>
                        <tr>
                            <th>Faculty</th>
                            <th>Department</th>
                            <th>Total Requests</th>
                            <th>Total Days</th>
                        </tr>
                    </thead>
                    <tbody>
                        {% for faculty in top_requesters %}
                        <tr>
                            <td>{{ faculty.faculty__user__first_name }} {{ faculty.faculty__user__last_name }}</td>
                            <td>{{ faculty.faculty__department }}</td>
                            <td>{{ faculty.total_requests }}</td>
                            <td>{{ faculty.total_days }}</td>
                        </tr>
                        {% endfor %}
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</div>
{% endblock %}

{% block extra_js %}
<script>
// Leave by Type Chart
const typeCtx = document.getElementById('leaveTypeChart').getContext('2d');
const typeChart = new Chart(typeCtx, {
    type: 'bar',
    data: {
        labels: [{% for item in leave_by_type %}'{{ item.leave_type__name }}',{% endfor %}],
        datasets: [{
            label: 'Total',
            data: [{% for item in leave_by_type %}{{ item.total }},{% endfor %}],
            backgroundColor: 'rgba(54, 162, 235, 0.5)',
        }, {
            label: 'Approved',
            data: [{% for item in leave_by_type %}{{ item.approved }},{% endfor %}],
            backgroundColor: 'rgba(75, 192, 192, 0.5)',
        }, {
            label: 'Rejected',
            data: [{% for item in leave_by_type %}{{ item.rejected }},{% endfor %}],
            backgroundColor: 'rgba(255, 99, 132, 0.5)',
        }, {
            label: 'Pending',
            data: [{% for item in leave_by_type %}{{ item.pending }},{% endfor %}],
            backgroundColor: 'rgba(255, 206, 86, 0.5)',
        }]
    },
    options: {
        responsive: true,
        scales: {
            y: {
                beginAtZero: true
            }
        }
    }
});

// Monthly Trend Chart
const monthlyCtx = document.getElementById('monthlyChart').getContext('2d');
const monthlyChart = new Chart(monthlyCtx, {
    type: 'line',
    data: {
        labels: ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'],
        datasets: [{
            label: 'Approved Leaves',
            data: {{ monthly_leaves|safe }},
            borderColor: 'rgb(75, 192, 192)',
            backgroundColor: 'rgba(75, 192, 192, 0.2)',
            tension: 0.1
        }]
    },
    options: {
        responsive: true,
        scales: {
            y: {
                beginAtZero: true
            }
        }
    }
});
</script>
{% endblock %}
```

## Step 4: Update URLs

Add to `leaves/urls.py`:

```python
urlpatterns = [
    # ... existing URLs ...
    
    # Reports
    path('reports/', views.reports, name='reports'),
    path('reports/my/', views.my_leave_report, name='my_leave_report'),
    path('export/csv/', views.export_leaves_csv, name='export_leaves_csv'),
    path('export/pdf/', views.export_leaves_pdf, name='export_leaves_pdf'),
]
```

## Step 5: Add Chart.js for Visualization

Already included in the reports template via CDN.

## Export Data Best Practices

1. **Large datasets**: Use pagination or background tasks
2. **Security**: Check permissions before export
3. **File formats**: Offer multiple formats (CSV, Excel, PDF)
4. **Filters**: Allow users to filter before export
5. **Scheduling**: Consider automated scheduled reports

## Next Steps

In Part 10, we'll cover testing, deployment, and final touches.

---

*Workshop Materials | Faculty Leave Management System | Part 9 of 10*
