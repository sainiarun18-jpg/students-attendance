
"""
Student Attendance & WhatsApp Alert System - Single-File Flask App
--------------------------------------------------------------------
Everything (DB models, routes, HTML, CSS) lives in this one file so you
only need to upload app.py to PythonAnywhere.

Setup on PythonAnywhere:
    1. Upload this file to your PythonAnywhere project folder.
    2. Open a Bash console there and run:
         pip3 install --user flask flask_sqlalchemy twilio
    3. Set env vars (Web tab -> your app -> "Environment variables", or
       just hardcode them below in the CONFIG section for a quick test):
         TWILIO_ACCOUNT_SID
         TWILIO_AUTH_TOKEN
         TWILIO_WHATSAPP_FROM   (e.g. whatsapp:+14155238886)
    4. On the "Web" tab, point the WSGI file to import `app` from this file
       (PythonAnywhere's auto-generated WSGI file just needs:
          from app import app as application
       — adjust the path to wherever you uploaded app.py).
    5. Reload the web app.

Run locally instead:
    pip install flask flask_sqlalchemy twilio
    python app.py
    -> http://localhost:5000
"""

import os
from datetime import date, datetime

from flask import Flask, request, redirect, url_for, flash, render_template_string
from flask_sqlalchemy import SQLAlchemy

# ---------------------------------------------------------------------------
# CONFIG
# ---------------------------------------------------------------------------

app = Flask(__name__)
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///attendance.db"
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False
app.secret_key = "dev-secret-change-me"

db = SQLAlchemy(app)

# "twilio" or "pywhatkit"
MESSAGING_BACKEND = os.environ.get("MESSAGING_BACKEND", "twilio")


# ---------------------------------------------------------------------------
# MODELS
# ---------------------------------------------------------------------------

class Student(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(120), nullable=False)
    roll_no = db.Column(db.String(20), nullable=False, unique=True)
    class_name = db.Column(db.String(50), nullable=False)
    parent_name = db.Column(db.String(120))
    parent_whatsapp = db.Column(db.String(20), nullable=False)  # e.g. +919812345678

    attendance_records = db.relationship("Attendance", backref="student", lazy=True)


class Attendance(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    student_id = db.Column(db.Integer, db.ForeignKey("student.id"), nullable=False)
    date = db.Column(db.Date, nullable=False, default=date.today)
    status = db.Column(db.String(10), nullable=False)  # "present" / "absent"
    message_sent = db.Column(db.Boolean, default=False)
    message_sent_at = db.Column(db.DateTime, nullable=True)

    __table_args__ = (db.UniqueConstraint("student_id", "date", name="uq_student_date"),)


# ---------------------------------------------------------------------------
# MESSAGING GATEWAY
# ---------------------------------------------------------------------------

def send_whatsapp_message(to_number: str, message: str):
    if MESSAGING_BACKEND == "twilio":
        return _send_via_twilio(to_number, message)
    elif MESSAGING_BACKEND == "pywhatkit":
        return _send_via_pywhatkit(to_number, message)
    return False, f"Unknown MESSAGING_BACKEND: {MESSAGING_BACKEND}"


def _send_via_twilio(to_number: str, message: str):
    try:
        from twilio.rest import Client
    except ImportError:
        return False, "twilio not installed. Run: pip install twilio"

    account_sid = os.environ.get("TWILIO_ACCOUNT_SID")
    auth_token = os.environ.get("TWILIO_AUTH_TOKEN")
    from_whatsapp = os.environ.get("TWILIO_WHATSAPP_FROM")

    if not all([account_sid, auth_token, from_whatsapp]):
        return False, "Missing TWILIO_ACCOUNT_SID / TWILIO_AUTH_TOKEN / TWILIO_WHATSAPP_FROM env vars"

    try:
        client = Client(account_sid, auth_token)
        msg = client.messages.create(
            from_=from_whatsapp,
            to=f"whatsapp:{to_number}",
            body=message,
        )
        return True, msg.sid
    except Exception as e:
        return False, str(e)


def _send_via_pywhatkit(to_number: str, message: str):
    try:
        import pywhatkit as kit
    except ImportError:
        return False, "pywhatkit not installed. Run: pip install pywhatkit"
    try:
        kit.sendwhatmsg_instantly(to_number, message, wait_time=15, tab_close=True)
        return True, "sent via pywhatkit"
    except Exception as e:
        return False, str(e)


def build_absence_message(student: Student) -> str:
    return (
        f"Dear {student.parent_name or 'Parent'}, this is to inform you that "
        f"{student.name} (Roll No: {student.roll_no}, Class: {student.class_name}) "
        f"was marked ABSENT today ({date.today().strftime('%d-%m-%Y')}). "
        f"Please contact the school office if this is unexpected. - School Attendance System"
    )


# ---------------------------------------------------------------------------
# TEMPLATES (all inline, rendered with render_template_string)
# ---------------------------------------------------------------------------

BASE_HTML = """
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>{% block title %}Attendance System{% endblock %}</title>
<style>
* { box-sizing: border-box; }
body { margin: 0; font-family: -apple-system, Segoe UI, Roboto, Arial, sans-serif; background: #f4f6f9; color: #1f2937; }
.layout { display: flex; min-height: 100vh; }
.sidebar { width: 220px; background: #111827; color: #fff; padding: 24px 16px; flex-shrink: 0; }
.sidebar .brand { font-size: 18px; margin-bottom: 24px; }
.sidebar nav { display: flex; flex-direction: column; gap: 4px; }
.sidebar nav a { color: #cbd5e1; text-decoration: none; padding: 10px 12px; border-radius: 8px; font-size: 14px; }
.sidebar nav a:hover { background: #1f2937; color: #fff; }
.sidebar nav a.active { background: #2563eb; color: #fff; }
.content { flex: 1; padding: 32px 40px; }
h1 { margin: 0 0 4px; }
.subtitle { color: #6b7280; margin: 0 0 24px; }
.cards { display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); gap: 16px; margin-bottom: 28px; }
.card { background: #fff; border-radius: 12px; padding: 20px; box-shadow: 0 1px 3px rgba(0,0,0,0.08); display: flex; flex-direction: column; gap: 8px; }
.card-label { font-size: 13px; color: #6b7280; text-transform: uppercase; letter-spacing: .03em; }
.card-value { font-size: 32px; font-weight: 700; }
.card-green .card-value { color: #16a34a; }
.card-red .card-value { color: #dc2626; }
.card-blue .card-value { color: #2563eb; }
.quick-actions { display: flex; gap: 12px; margin-bottom: 24px; }
.btn { display: inline-block; border: none; border-radius: 8px; padding: 10px 18px; font-size: 14px; cursor: pointer; text-decoration: none; font-weight: 600; }
.btn-primary { background: #2563eb; color: #fff; }
.btn-outline { background: #fff; border: 1px solid #d1d5db; color: #1f2937; }
.btn-whatsapp { background: #25D366; color: #fff; }
.btn-whatsapp:disabled { background: #a7e3bd; cursor: not-allowed; }
.btn-lg { padding: 14px 24px; font-size: 16px; }
.btn-sm { padding: 6px 12px; font-size: 13px; }
.table { width: 100%; border-collapse: collapse; background: #fff; border-radius: 12px; overflow: hidden; box-shadow: 0 1px 3px rgba(0,0,0,0.08); }
.table th, .table td { padding: 12px 16px; text-align: left; border-bottom: 1px solid #f1f5f9; font-size: 14px; }
.table th { background: #f9fafb; font-weight: 600; color: #374151; }
.badge { padding: 4px 10px; border-radius: 999px; font-size: 12px; font-weight: 600; }
.badge-green { background: #dcfce7; color: #15803d; }
.badge-gray { background: #f1f5f9; color: #6b7280; }
.form { display: flex; flex-direction: column; gap: 14px; max-width: 420px; }
.form label { display: flex; flex-direction: column; gap: 6px; font-size: 13px; color: #374151; font-weight: 600; }
.form input { padding: 10px 12px; border: 1px solid #d1d5db; border-radius: 8px; font-size: 14px; }
.flash { padding: 10px 16px; border-radius: 8px; margin-bottom: 16px; font-size: 14px; }
.flash-success { background: #dcfce7; color: #15803d; }
.flash-error { background: #fee2e2; color: #b91c1c; }
</style>
</head>
<body>
<div class="layout">
  <aside class="sidebar">
    <h2 class="brand">📋 Attendance</h2>
    <nav>
      <a href="{{ url_for('dashboard') }}" class="{{ 'active' if request.endpoint == 'dashboard' }}">Dashboard</a>
      <a href="{{ url_for('mark_attendance') }}" class="{{ 'active' if request.endpoint == 'mark_attendance' }}">Mark Attendance</a>
      <a href="{{ url_for('absentees') }}" class="{{ 'active' if request.endpoint == 'absentees' }}">Absentee List</a>
      <a href="{{ url_for('students') }}" class="{{ 'active' if request.endpoint in ['students', 'add_student'] }}">Students</a>
    </nav>
  </aside>
  <main class="content">
    {% with messages = get_flashed_messages(with_categories=true) %}
      {% if messages %}
        {% for category, msg in messages %}
          <div class="flash flash-{{ category }}">{{ msg }}</div>
        {% endfor %}
      {% endif %}
    {% endwith %}
    {% block content %}{% endblock %}
  </main>
</div>
</body>
</html>
"""

DASHBOARD_HTML = """
{% extends base %}
{% block title %}Dashboard{% endblock %}
{% block content %}
<h1>Dashboard</h1>
<p class="subtitle">{{ today.strftime('%A, %d %B %Y') }}</p>
<div class="cards">
  <div class="card"><span class="card-label">Total Enrolled</span><span class="card-value">{{ total_students }}</span></div>
  <div class="card card-green"><span class="card-label">Present Today</span><span class="card-value">{{ present_today }}</span></div>
  <div class="card card-red"><span class="card-label">Absent Today</span><span class="card-value">{{ absent_today }}</span></div>
  <div class="card card-blue"><span class="card-label">Messages Sent</span><span class="card-value">{{ messages_sent_today }}</span></div>
</div>
<div class="quick-actions">
  <a href="{{ url_for('mark_attendance') }}" class="btn btn-primary">Mark Today's Attendance</a>
  <a href="{{ url_for('absentees') }}" class="btn btn-outline">View Absentee List</a>
</div>
{% endblock %}
"""

STUDENTS_HTML = """
{% extends base %}
{% block title %}Students{% endblock %}
{% block content %}
<h1>Students</h1>
<a href="{{ url_for('add_student') }}" class="btn btn-primary" style="margin-bottom:16px;display:inline-block;">+ Add Student</a>
<table class="table">
  <thead><tr><th>Roll No</th><th>Name</th><th>Class</th><th>Parent</th><th>WhatsApp No.</th></tr></thead>
  <tbody>
    {% for s in students %}
    <tr>
      <td>{{ s.roll_no }}</td><td>{{ s.name }}</td><td>{{ s.class_name }}</td>
      <td>{{ s.parent_name or '-' }}</td><td>{{ s.parent_whatsapp }}</td>
    </tr>
    {% endfor %}
  </tbody>
</table>
{% endblock %}
"""

ADD_STUDENT_HTML = """
{% extends base %}
{% block title %}Add Student{% endblock %}
{% block content %}
<h1>Add Student</h1>
<form method="POST" class="form">
  <label>Name<input type="text" name="name" required></label>
  <label>Roll No<input type="text" name="roll_no" required></label>
  <label>Class<input type="text" name="class_name" required placeholder="e.g. 8-A"></label>
  <label>Parent Name<input type="text" name="parent_name"></label>
  <label>Parent WhatsApp Number<input type="text" name="parent_whatsapp" required placeholder="+919812345678"></label>
  <button type="submit" class="btn btn-primary">Add Student</button>
</form>
{% endblock %}
"""

MARK_ATTENDANCE_HTML = """
{% extends base %}
{% block title %}Mark Attendance{% endblock %}
{% block content %}
<h1>Mark Attendance</h1>
<p class="subtitle">{{ today.strftime('%A, %d %B %Y') }}</p>
<form method="POST">
  <table class="table">
    <thead><tr><th>Roll No</th><th>Name</th><th>Class</th><th>Present</th><th>Absent</th></tr></thead>
    <tbody>
      {% for s in students %}
      <tr>
        <td>{{ s.roll_no }}</td><td>{{ s.name }}</td><td>{{ s.class_name }}</td>
        <td><input type="radio" name="status_{{ s.id }}" value="present" {{ 'checked' if existing.get(s.id, 'present') == 'present' }}></td>
        <td><input type="radio" name="status_{{ s.id }}" value="absent" {{ 'checked' if existing.get(s.id) == 'absent' }}></td>
      </tr>
      {% endfor %}
    </tbody>
  </table>
  <button type="submit" class="btn btn-primary">Save Attendance</button>
</form>
{% endblock %}
"""

ABSENTEES_HTML = """
{% extends base %}
{% block title %}Absentee List{% endblock %}
{% block content %}
<h1>Absentee List</h1>
<p class="subtitle">{{ today.strftime('%A, %d %B %Y') }}</p>
{% if records %}
<form method="POST" action="{{ url_for('send_all_alerts') }}" style="margin-bottom:16px;">
  <button type="submit" class="btn btn-whatsapp btn-lg">📲 Send WhatsApp Alerts to All Absentees</button>
</form>
<table class="table">
  <thead>
    <tr><th>Roll No</th><th>Name</th><th>Class</th><th>Parent</th><th>WhatsApp No.</th><th>Alert Status</th><th>Action</th></tr>
  </thead>
  <tbody>
    {% for record, student in records %}
    <tr>
      <td>{{ student.roll_no }}</td>
      <td>{{ student.name }}</td>
      <td>{{ student.class_name }}</td>
      <td>{{ student.parent_name or '-' }}</td>
      <td>{{ student.parent_whatsapp }}</td>
      <td>
        {% if record.message_sent %}
          <span class="badge badge-green">Sent {{ record.message_sent_at.strftime('%H:%M') }}</span>
        {% else %}
          <span class="badge badge-gray">Not sent</span>
        {% endif %}
      </td>
      <td>
        <form method="POST" action="{{ url_for('send_alert', attendance_id=record.id) }}">
          <button type="submit" class="btn btn-whatsapp btn-sm" {{ 'disabled' if record.message_sent }}>
            {{ 'Sent ✓' if record.message_sent else 'Send Alert' }}
          </button>
        </form>
      </td>
    </tr>
    {% endfor %}
  </tbody>
</table>
{% else %}
<p>No absentees recorded for today yet. 🎉</p>
{% endif %}
{% endblock %}
"""


def render(template_str, **context):
    """Renders a template string, always passing `base` for {% extends base %}."""
    return render_template_string(template_str, base=BASE_HTML, **context)


# ---------------------------------------------------------------------------
# ROUTES
# ---------------------------------------------------------------------------

@app.route("/")
def dashboard():
    today = date.today()
    total_students = Student.query.count()
    present_today = Attendance.query.filter_by(date=today, status="present").count()
    absent_today = Attendance.query.filter_by(date=today, status="absent").count()
    messages_sent_today = Attendance.query.filter_by(date=today, message_sent=True).count()
    return render(
        DASHBOARD_HTML,
        total_students=total_students,
        present_today=present_today,
        absent_today=absent_today,
        messages_sent_today=messages_sent_today,
        today=today,
    )


@app.route("/students")
def students():
    all_students = Student.query.order_by(Student.class_name, Student.roll_no).all()
    return render(STUDENTS_HTML, students=all_students)


@app.route("/students/add", methods=["GET", "POST"])
def add_student():
    if request.method == "POST":
        s = Student(
            name=request.form["name"],
            roll_no=request.form["roll_no"],
            class_name=request.form["class_name"],
            parent_name=request.form.get("parent_name"),
            parent_whatsapp=request.form["parent_whatsapp"],
        )
        db.session.add(s)
        db.session.commit()
        flash(f"Added {s.name}", "success")
        return redirect(url_for("students"))
    return render(ADD_STUDENT_HTML)


@app.route("/attendance", methods=["GET", "POST"])
def mark_attendance():
    today = date.today()
    all_students = Student.query.order_by(Student.class_name, Student.roll_no).all()

    if request.method == "POST":
        for s in all_students:
            status = request.form.get(f"status_{s.id}", "absent")
            record = Attendance.query.filter_by(student_id=s.id, date=today).first()
            if record:
                record.status = status
            else:
                record = Attendance(student_id=s.id, date=today, status=status)
                db.session.add(record)
        db.session.commit()
        flash("Attendance saved for today.", "success")
        return redirect(url_for("dashboard"))

    existing = {a.student_id: a.status for a in Attendance.query.filter_by(date=today).all()}
    return render(MARK_ATTENDANCE_HTML, students=all_students, existing=existing, today=today)


@app.route("/absentees")
def absentees():
    today = date.today()
    records = (
        db.session.query(Attendance, Student)
        .join(Student, Attendance.student_id == Student.id)
        .filter(Attendance.date == today, Attendance.status == "absent")
        .all()
    )
    return render(ABSENTEES_HTML, records=records, today=today)


@app.route("/absentees/send-alert/<int:attendance_id>", methods=["POST"])
def send_alert(attendance_id):
    record = Attendance.query.get_or_404(attendance_id)
    student = Student.query.get_or_404(record.student_id)

    message = build_absence_message(student)
    success, info = send_whatsapp_message(student.parent_whatsapp, message)

    if success:
        record.message_sent = True
        record.message_sent_at = datetime.now()
        db.session.commit()
        flash(f"WhatsApp alert sent to {student.parent_name or student.name}'s parent.", "success")
    else:
        flash(f"Failed to send alert: {info}", "error")

    return redirect(url_for("absentees"))


@app.route("/absentees/send-all", methods=["POST"])
def send_all_alerts():
    today = date.today()
    records = Attendance.query.filter_by(date=today, status="absent", message_sent=False).all()

    sent, failed = 0, 0
    for record in records:
        student = Student.query.get(record.student_id)
        message = build_absence_message(student)
        success, info = send_whatsapp_message(student.parent_whatsapp, message)
        if success:
            record.message_sent = True
            record.message_sent_at = datetime.now()
            sent += 1
        else:
            failed += 1
    db.session.commit()

    flash(f"Sent {sent} alerts. {failed} failed.", "success" if failed == 0 else "error")
    return redirect(url_for("absentees"))


# ---------------------------------------------------------------------------
# INIT
# ---------------------------------------------------------------------------

with app.app_context():
    db.create_all()

if __name__ == "__main__":
    app.run(debug=True)
