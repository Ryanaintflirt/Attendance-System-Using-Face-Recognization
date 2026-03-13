# Face Recognition Attendance System

A Flask application for browser-based attendance capture using face recognition and blink detection, backed by SQLite, SQLAlchemy, and role-based access control.

## Overview

This project now uses:

- SQLite through Flask-SQLAlchemy for persistent storage
- browser webcam capture through HTML5 and JavaScript
- server-side face matching and blink detection
- Flask-Login for authentication
- role-based access control for Admin and Teacher users

The application is intended for local or internal-network use where a webcam-enabled browser can access the Flask server.

## Features

- Face recognition with blink-based attendance confirmation
- Browser-side webcam capture with `getUserMedia`
- SQLAlchemy models for students, batches, subjects, sessions, records, and users
- Attendance reports generated from database queries
- Analytics charts generated from database aggregates
- Admin-only analytics access
- Teacher and Admin access for attendance capture and reporting
- In-app password change screen
- CLI commands for creating Admin and Teacher accounts

## Tech Stack

- Backend: Flask
- Authentication: Flask-Login
- ORM / Database: Flask-SQLAlchemy, SQLite
- Computer vision: OpenCV, face_recognition, dlib
- Charts: matplotlib
- Frontend: HTML, Tailwind CSS, Font Awesome, vanilla JavaScript

## Project Structure

```text
Face-Recognition-Attendance-System/
├── app.py
├── models.py
├── init_db.py
├── register_students.py
├── requirements.txt
├── Photos/
├── static/
│   └── generated_analytics/
└── templates/
	├── base.html
	├── login.html
	├── change_password.html
	├── index.html
	├── select_batch_subject.html
	├── attendance_session.html
	├── attendance_report.html
	├── view_analytics.html
	└── analytics_result.html
```

## Database Models

The application stores data in SQLite through SQLAlchemy models:

- `User`: application login accounts with roles (`Admin`, `Teacher`)
- `Batch`: class or group identifier
- `Subject`: subject name
- `Student`: student profile with face encoding blob
- `AttendanceSession`: one attendance run for a batch and subject on a date/time
- `AttendanceRecord`: one student’s status for a session

## Roles and Access Control

### Admin

- Can log in
- Can take attendance
- Can view attendance reports
- Can view analytics
- Can change password

### Teacher

- Can log in
- Can take attendance
- Can view attendance reports
- Can change password
- Cannot access analytics

## How Attendance Works Now

1. A logged-in user selects a batch and subject.
2. The browser requests webcam access with `navigator.mediaDevices.getUserMedia(...)`.
3. JavaScript captures frames from the video stream every few hundred milliseconds.
4. Each frame is sent to `/api/process_frame` as a Base64 JPEG string.
5. The backend decodes the frame, matches faces against student encodings in the database, and performs blink detection.
6. When the user finishes the session, `/api/finalize_attendance` writes the final `AttendanceRecord` rows into the database.
7. Reports and analytics are generated from database queries, not CSV files.

## Requirements

- Python 3.10 or 3.11 is recommended on Windows
- A webcam-enabled browser
- A working webcam
- Build tools may be required for `dlib` and `face_recognition`

Important:

`face_recognition` depends on `dlib`, which may fail on unsupported Python versions or on machines missing build tools. If you hit installation problems, use Python 3.10 or 3.11 and recreate the virtual environment.

## Setup

### 1. Create and activate a virtual environment

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

### 2. Install dependencies

```powershell
pip install -r requirements.txt
```

If `face_recognition` fails, install a supported Python version first and recreate the environment.

### 3. Initialize the database

```powershell
flask --app app.py init-db
```

This creates the database tables and seeds the default batches and subjects.

### 4. Create the first Admin account

```powershell
flask --app app.py create-admin
```

You will be prompted for:

- username
- password
- password confirmation

### 5. Create Teacher accounts if needed

```powershell
flask --app app.py create-teacher
```

This creates or updates a non-admin teacher account without manual database edits.

### 6. Add student photos

Put student images inside `Photos/`.

You can either:

- place photos in batch subfolders, for example `Photos/BatchA/Meet.jpg`
- or use the root folder and pass `--batch` when registering students

Rules:

- one clear face per image
- filename becomes the student name
- examples: `Meet.jpg`, `Divy.png`, `Elon.jpeg`

### 7. Register students into the database

With batch folders:

```powershell
python register_students.py
```

With a root-level `Photos/` folder:

```powershell
python register_students.py --batch BatchA
```

This stores student records and face encodings directly in the database.

### 8. Run the application

```powershell
python app.py
```

Open:

```text
http://127.0.0.1:5000
```

## Authentication Flow

1. Open the application in the browser.
2. Log in with an Admin or Teacher account.
3. Use the navigation bar to access attendance, reports, analytics, or password change.
4. Use `Logout` when finished.

## Password Management

Logged-in users can change their password from the navigation bar.

The password change screen requires:

- current password
- new password
- confirmation of the new password

## Attendance Workflow

1. Log in.
2. Go to `Attendance`.
3. Select a batch and subject.
4. Allow browser webcam access.
5. Look at the camera and blink when prompted.
6. Click `Finish and Save Attendance`.
7. Review the attendance report.

## Reporting and Analytics

### Attendance Report

The report page queries `AttendanceSession`, `AttendanceRecord`, and `Student` from the database and supports filters by:

- batch
- subject
- exact date
- month

### Analytics

Analytics are generated from SQLAlchemy aggregate queries and saved to `static/generated_analytics/`.

Admin users can generate:

- single-subject present vs absent charts for a selected month
- all-subject comparison charts for a selected batch and month

## CLI Commands

### Initialize the database

```powershell
flask --app app.py init-db
```

### Create or update an Admin user

```powershell
flask --app app.py create-admin
```

### Create or update a Teacher user

```powershell
flask --app app.py create-teacher
```

## Configuration

Optional environment variables:

- `FLASK_SECRET_KEY`: overrides the default development secret key
- `FLASK_DEBUG=1`: enables Flask debug mode
- `DATABASE_URL`: overrides the default SQLite database path

Example:

```powershell
$env:FLASK_SECRET_KEY="change-this"
$env:FLASK_DEBUG="1"
python app.py
```

## Common Problems

### `ModuleNotFoundError: No module named 'face_recognition'`

Cause:

- the package is not installed in the active environment
- `dlib` failed to build

Fix:

```powershell
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

If it still fails, use Python 3.10 or 3.11 and recreate the environment.

### The app redirects to login repeatedly

Check:

- the database has a `users` table
- you created an admin or teacher account
- browser cookies are enabled

### Camera permission is denied in the browser

Check:

- the browser has permission to use the webcam
- the page is opened from the same browser session where you logged in
- no other app is locking the webcam

### No students are recognized

Check:

- students were registered with `register_students.py`
- the selected batch matches the students’ batch
- the photos contain one clear face each

## Limitations

- Active attendance-session state is kept in server memory until the session is finalized
- Batch and subject seed values are still initialized in application code
- This project is intended for local or controlled internal use, not public deployment
- Recognition quality depends on lighting, camera quality, and the quality of source student photos

## Future Improvements

- Add database migrations with Alembic or Flask-Migrate
- Add user management screens in the Admin UI
- Add automated tests for authentication and attendance APIs
- Add cancel-session cleanup for abandoned attendance sessions

## Credits

- [face_recognition](https://github.com/ageitgey/face_recognition)
- [OpenCV](https://opencv.org/)
- [Matplotlib](https://matplotlib.org/)
- [Flask](https://flask.palletsprojects.com/)
- [Flask-Login](https://flask-login.readthedocs.io/)
#   A t t e n d a n c e - S y s t e m - U s i n g - F a c e - R e c o g n i z a t i o n  
 