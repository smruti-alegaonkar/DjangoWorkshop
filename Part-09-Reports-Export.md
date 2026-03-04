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
from django.contrib.auth.decorators import login_required, user_passes_test
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib import messages
from django.utils import timezone
from .models import LeaveRequest, LeaveBalance, FacultyProfile
from .forms import LeaveReviewForm, LeaveRequestForm
from django.http import HttpResponse
from django.utils import timezone
import csv
from datetime import datetime, timedelta
from django.contrib.auth import login
from .forms import FacultyRegistrationForm
from django.db.models import Count, Q, Sum
from django.db import models
import json
from django.utils.timezone import now
from reportlab.pdfgen import canvas
import io
from django.contrib.auth.views import LoginView
from django.urls import reverse_lazy
from .forms import ProfileUpdateForm

class CustomLoginView(LoginView):
    template_name = "leaves/login.html"

    def get_success_url(self):
        user = self.request.user

        if user.is_superuser:
            return reverse_lazy('leaves:reports')   # Admin goes to reports
        else:
            return reverse_lazy('leaves:dashboard') # Faculty goes to dashboard

@login_required
def reports(request):

    current_year = now().year

    # 🔹 Role-based filtering
    if request.user.is_superuser:
        leaves = LeaveRequest.objects.filter(start_date__year=current_year)
    else:
        faculty_profile = FacultyProfile.objects.get(user=request.user)
        leaves = LeaveRequest.objects.filter(
            faculty=faculty_profile,
        )

    # 🔹 Leave type stats
    leave_type_data = leaves.values('leave_type__name').annotate(
        total=Count('id')
    )

    leave_type_labels = [item['leave_type__name'] for item in leave_type_data]
    leave_type_totals = [item['total'] for item in leave_type_data]

    leave_type_approved = []
    leave_type_rejected = []
    leave_type_pending = []

    for label in leave_type_labels:
        leave_type_approved.append(
            leaves.filter(leave_type__name=label, status='approved').count()
        )
        leave_type_rejected.append(
            leaves.filter(leave_type__name=label, status='rejected').count()
        )
        leave_type_pending.append(
            leaves.filter(leave_type__name=label, status='pending').count()
        )

    # 🔹 Monthly approved
    monthly_leaves = []
    for month in range(1, 13):
        monthly_leaves.append(
            leaves.filter(
                status='approved',
                start_date__month=month
            ).count()
        )

    # 🔹 ADDITION: Admin-only analytics tables
    if request.user.is_superuser:

        leave_by_dept = leaves.values(
            'faculty__department'
        ).annotate(
            total=Count('id'),
            total_days=Sum('number_of_days')
        )

        top_requesters = leaves.values(
            'faculty__user__first_name',
            'faculty__user__last_name',
            'faculty__department'
        ).annotate(
            total_requests=Count('id'),
            total_days=Sum('number_of_days')
        ).order_by('-total_requests')[:5]

    else:
        leave_by_dept = None
        top_requesters = None

    context = {
        "current_year": current_year,
        "leave_type_labels": json.dumps(leave_type_labels),
        "leave_type_totals": json.dumps(leave_type_totals),
        "leave_type_approved": json.dumps(leave_type_approved),
        "leave_type_rejected": json.dumps(leave_type_rejected),
        "leave_type_pending": json.dumps(leave_type_pending),
        "monthly_leaves": json.dumps(monthly_leaves),

        # 🔹 ADDED
        "leave_by_dept": leave_by_dept,
        "top_requesters": top_requesters,
    }

    return render(request, "leaves/reports.html", context)


@login_required
def export_leaves_csv(request):

    year = request.GET.get('year', now().year)

    # Role-based filtering
    if request.user.is_superuser:
        leaves = LeaveRequest.objects.filter(start_date__year=year)
    else:
        faculty_profile = FacultyProfile.objects.get(user=request.user)
        leaves = LeaveRequest.objects.filter(
            faculty=faculty_profile,
            start_date__year=year
        )

    response = HttpResponse(content_type='text/csv')
    response['Content-Disposition'] = f'attachment; filename="leave_report_{year}.csv"'

    writer = csv.writer(response)
    writer.writerow([
        "Faculty",
        "Leave Type",
        "Start Date",
        "End Date",
        "Days",
        "Status"
    ])

    for leave in leaves:
        writer.writerow([
            leave.faculty.user.get_full_name(),
            leave.leave_type.name,
            leave.start_date,
            leave.end_date,
            leave.number_of_days,
            leave.status
        ])

    return response


@login_required
def export_leaves_pdf(request):

    year = request.GET.get("year", now().year)

    if request.user.is_superuser:
        leaves = LeaveRequest.objects.filter(start_date__year=year)
    else:
        faculty_profile = FacultyProfile.objects.get(user=request.user)
        leaves = LeaveRequest.objects.filter(
            faculty=faculty_profile,
            start_date__year=year
        )

    buffer = io.BytesIO()
    p = canvas.Canvas(buffer)

    y = 800
    p.drawString(200, y, f"Leave Report - {year}")
    y -= 30

    for leave in leaves:
        text = f"{leave.faculty.user.get_full_name()} | {leave.leave_type.name} | {leave.start_date} | {leave.status}"
        p.drawString(50, y, text)
        y -= 20

    p.showPage()
    p.save()

    buffer.seek(0)

    response = HttpResponse(buffer, content_type="application/pdf")
    response["Content-Disposition"] = f'attachment; filename="leave_report_{year}.pdf"'

    return response
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
        <h2>Leave Reports & Analytics ({{ current_year }})</h2>
    </div>
    <div class="col-md-4 text-end">
        <div class="btn-group">
            <a href="{% url 'leaves:export_leaves_csv' %}?year={{ current_year }}" class="btn btn-success">
                Export CSV
            </a>
            <a href="{% url 'leaves:export_leaves_pdf' %}" class="btn btn-danger">
                Export PDF
            </a>
        </div>
    </div>
</div>

<!-- Leave Type Chart -->
<div class="card mb-4">
    <div class="card-header">
        Leave Requests by Type
    </div>
    <div class="card-body">
        <canvas id="leaveTypeChart"></canvas>
    </div>
</div>

<!-- Monthly Approved Leaves -->
<div class="col-md-6">
    <div class="card">
        <div class="card-header">
            Monthly Approved Leaves
        </div>
        <div class="card-body">
            <canvas id="monthlyChart"></canvas>
        </div>
    </div>
</div>

{% if request.user.is_superuser %}

<!-- Department-wise Leave Summary -->
<div class="row mb-4">
    <div class="col-md-6">
        <div class="card">
            <div class="card-header">
                Leave by Department
            </div>
            <div class="card-body">
                <table class="table table-bordered table-sm">
                    <thead>
                        <tr>
                            <th>Department</th>
                            <th>Total Requests</th>
                            <th>Total Days</th>
                        </tr>
                    </thead>
                    <tbody>
                        {% for dept in leave_by_dept %}
                        <tr>
                            <td>{{ dept.faculty__department }}</td>
                            <td>{{ dept.total }}</td>
                            <td>{{ dept.total_days|default:0 }}</td>
                        </tr>
                        {% empty %}
                        <tr>
                            <td colspan="3" class="text-center">No data available</td>
                        </tr>
                        {% endfor %}
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</div>

<!-- Top Requesters -->
<div class="card">
    <div class="card-header">
        Top Leave Requesters
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
                    <td>
                        {{ faculty.faculty__user__first_name }}
                        {{ faculty.faculty__user__last_name }}
                    </td>
                    <td>{{ faculty.faculty__department }}</td>
                    <td>{{ faculty.total_requests }}</td>
                    <td>{{ faculty.total_days|default:0 }}</td>
                </tr>
                {% empty %}
                <tr>
                    <td colspan="4" class="text-center">No data available</td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>
</div>

{% endif %}

{% endblock %}


{% block extra_js %}

<!-- Safe JSON Data -->
<script id="report-data" type="application/json">
{
    "labels": {{ leave_type_labels|default:"[]"|safe }},
    "totals": {{ leave_type_totals|default:"[]"|safe }},
    "approved": {{ leave_type_approved|default:"[]"|safe }},
    "rejected": {{ leave_type_rejected|default:"[]"|safe }},
    "pending": {{ leave_type_pending|default:"[]"|safe }},
    "monthly": {{ monthly_leaves|default:"[]"|safe }}
}
</script>

<script>
document.addEventListener("DOMContentLoaded", function () {

    const rawData = document.getElementById("report-data").textContent;
    const reportData = JSON.parse(rawData);

    new Chart(document.getElementById("leaveTypeChart"), {
        type: "bar",
        data: {
            labels: reportData.labels,
            datasets: [
                {
                    label: "Total",
                    data: reportData.totals
                },
                {
                    label: "Approved",
                    data: reportData.approved
                },
                {
                    label: "Rejected",
                    data: reportData.rejected
                },
                {
                    label: "Pending",
                    data: reportData.pending
                }
            ]
        },
        options: {
            responsive: true,
            scales: {
                y: { beginAtZero: true }
            }
        }
    });

    new Chart(document.getElementById("monthlyChart"), {
        type: "line",
        data: {
            labels: ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"],
            datasets: [{
                label: "Approved Leaves",
                data: reportData.monthly,
                tension: 0.3
            }]
        },
        options: {
            responsive: true,
            scales: {
                y: { beginAtZero: true }
            }
        }
    });

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
