# 🏥 MediCare AMS — Hospital Appointment Management System

> A pure Java (no Maven / no Spring) backend with a MySQL database and a single-page HTML/CSS/JS frontend.

---

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Architecture](#architecture)
- [Design Patterns](#design-patterns)
- [Database Schema](#database-schema)
- [API Reference](#api-reference)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Setup & Run](#setup--run)
- [Default Credentials](#default-credentials)
- [Troubleshooting](#troubleshooting)

---

## Overview

MediCare AMS is a lightweight Hospital Appointment Management System built entirely with the Java standard library (`com.sun.net.httpserver`). It exposes a RESTful JSON API consumed by a single-page frontend (`index.html`). No build tools (Maven, Gradle) or external frameworks (Spring, Jakarta EE) are required — just a JDK and a MySQL server.

**Key highlights:**

- Role-based access control: **Admin**, **Doctor**, **Patient**
- Full appointment lifecycle: Book → Complete / Cancel
- Automatic slot management and overbooking protection
- Report export in CSV and plain-text formats
- Four Gang-of-Four design patterns applied throughout
- MVC layering: Server (Controller) → DAO (Model) → MySQL

---

## Project Structure

```
HospitalAMS/
├── src/
│   ├── HospitalServer.java          ← Entry point & HTTP route wiring
│   ├── util/
│   │   ├── DBConnection.java        ← Singleton DB connection pool
│   │   └── JsonUtil.java            ← Lightweight JSON builder (no external lib)
│   ├── model/
│   │   ├── User.java
│   │   ├── Doctor.java
│   │   ├── Patient.java
│   │   ├── Appointment.java
│   │   └── Schedule.java
│   ├── dao/
│   │   ├── UserDAO.java
│   │   ├── DoctorDAO.java
│   │   ├── PatientDAO.java
│   │   ├── ScheduleDAO.java
│   │   └── AppointmentDAO.java
│   ├── factory/
│   │   └── UserFactory.java         ← Creational: Factory pattern
│   ├── observer/
│   │   ├── AppointmentObserver.java ← Behavioral: Observer interface
│   │   ├── AppointmentEventManager.java
│   │   └── AuditLogObserver.java
│   ├── adapter/
│   │   ├── ReportFormatter.java     ← Structural: Adapter pattern (interface)
│   │   ├── RawCsvGenerator.java
│   │   ├── CsvReportAdapter.java
│   │   └── TextReportFormatter.java
│   └── strategy/
│       └── FilterStrategy.java      ← Behavioral: Strategy pattern
├── frontend/
│   └── index.html                   ← Full SPA (HTML + CSS + JS, single file)
├── lib/
│   └── mysql-connector-j-*.jar      ← ⚠ You must place this file here
├── sql/
│   └── schema.sql                   ← Database creation + seed data
├── run.sh                           ← Linux / macOS startup script
├── run.bat                          ← Windows startup script
└── README.md
```

---

## Architecture

```
┌─────────────────────────────────────────┐
│        Browser — index.html (SPA)       │
│           Fetch API (JSON)              │
└───────────────────┬─────────────────────┘
                    │ HTTP / JSON
┌───────────────────▼─────────────────────┐
│         HospitalServer.java             │
│  (Controller layer — route handlers)    │
│  Built on com.sun.net.httpserver        │
│  Thread pool: 10 threads                │
└───────────────────┬─────────────────────┘
                    │
┌───────────────────▼─────────────────────┐
│           DAO Layer (Model)             │
│  UserDAO · DoctorDAO · PatientDAO       │
│  ScheduleDAO · AppointmentDAO           │
└───────────────────┬─────────────────────┘
                    │ JDBC
┌───────────────────▼─────────────────────┐
│          MySQL 8.0+ Database            │
│          (hospital_ams schema)          │
└─────────────────────────────────────────┘
```

The server uses Java's built-in HTTP server — **no external frameworks needed**. CORS headers are included in every response, so the frontend can be served from any origin during development.

---

## Design Patterns

| Pattern | Category | Location | Purpose |
|---------|----------|----------|---------|
| **Singleton** | Creational | `util/DBConnection.java` | Single shared JDBC connection instance across the app |
| **Factory** | Creational | `factory/UserFactory.java` | Creates `User` objects pre-configured for ADMIN / DOCTOR / PATIENT roles |
| **Adapter** | Structural | `adapter/CsvReportAdapter.java` | Adapts the raw `RawCsvGenerator` to the unified `ReportFormatter` interface, letting the report endpoint switch formats transparently |
| **Observer** | Behavioral | `observer/` package | `AppointmentEventManager` notifies all registered observers (e.g., `AuditLogObserver`) whenever an appointment is booked, completed, or cancelled |
| **Strategy** | Behavioral | `strategy/FilterStrategy.java` | Swappable filtering algorithms for appointment queries (by date, patient, doctor, status) |

---

## Database Schema

Five tables in the `hospital_ams` database:

| Table | Description |
|-------|-------------|
| `users` | Base table for all roles — stores credentials, name, email, phone |
| `doctors` | Extends `users` with specialization, qualification, and total slot count |
| `patients` | Extends `users` with age, gender, and address |
| `schedules` | Available time slots per doctor per date; `is_booked` flag prevents double-booking |
| `appointments` | Links a patient, doctor, and schedule slot; tracks status (BOOKED / COMPLETED / CANCELLED) |

Seed data (4 doctors, 1 patient, 7 days of schedule slots) is included in `sql/schema.sql`.

---

## API Reference

All endpoints are prefixed with `/api`. Request and response bodies are JSON.

### Authentication

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| `POST` | `/api/login` | `{ username, password }` | Returns user object with role on success |

### Doctors

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/doctors` | List all doctors |
| `POST` | `/api/doctors` | Add a new doctor (Admin only) |
| `DELETE` | `/api/doctors/{id}` | Remove a doctor (Admin only) |

### Patients

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/patients` | List all patients (Admin / Doctor) |
| `POST` | `/api/patients` | Register a new patient |
| `DELETE` | `/api/patients/{id}` | Remove a patient (Admin only) |

### Schedules

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/schedules?doctorId=&date=` | Get available slots for a doctor on a specific date |
| `POST` | `/api/schedules` | Add a new time slot for a doctor |

### Appointments

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/appointments` | Retrieve all appointments |
| `POST` | `/api/appointments` | Book an appointment |
| `PUT` | `/api/appointments/{id}/status` | Update status (COMPLETED / CANCELLED) |
| `GET` | `/api/appointments/date?date=` | Filter appointments by date |
| `GET` | `/api/appointments/patient/{id}` | All appointments for a patient |
| `GET` | `/api/appointments/doctor/{id}` | All appointments for a doctor |

### Reports & Stats

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/summary?date=` | Daily appointment summary (count by status) |
| `GET` | `/api/doctor-stats` | Appointment count per doctor (used for bar chart) |
| `GET` | `/api/report?format=csv` | Download full appointment report as CSV |
| `GET` | `/api/report?format=txt` | Download full appointment report as plain text |

---

## Features

- [x] Role-based access control (Admin, Doctor, Patient)
- [x] Doctor & Patient CRUD management
- [x] Appointment booking with slot selection
- [x] Overbooking protection — slots marked `is_booked` after confirmed booking
- [x] Automatic slot release when an appointment is cancelled
- [x] Appointment status lifecycle: `BOOKED → COMPLETED / CANCELLED`
- [x] Daily appointment summary dashboard strip
- [x] Doctor-wise appointment count bar chart
- [x] Date-wise appointment history view
- [x] Search and filter by patient name, doctor name, appointment ID, or status
- [x] Report download in CSV and TXT formats
- [x] Observer-based audit logging on every appointment event
- [x] MVC architecture — clean separation of concerns
- [x] SOLID principles applied throughout
- [x] Five design patterns (Singleton, Factory, Adapter, Observer, Strategy)
- [x] No external libraries — only JDK + MySQL JDBC connector

---

## Prerequisites

| Requirement | Version | Download |
|-------------|---------|----------|
| Java JDK | 11 or higher | https://adoptium.net/ |
| MySQL | 8.0 or higher | https://dev.mysql.com/downloads/mysql/ |
| MySQL JDBC Connector JAR | Latest | https://dev.mysql.com/downloads/connector/j/ |

---

## Setup & Run

### Step 1 — Clone or extract the project

```bash
unzip HospitalAMS.zip
cd HospitalAMS
```

### Step 2 — Set up the database

Open MySQL CLI or MySQL Workbench and run the schema file:

```sql
source /path/to/HospitalAMS/sql/schema.sql;
```

Or paste the contents of `sql/schema.sql` directly into your SQL client. This creates the `hospital_ams` database, all tables, and seeds sample data.

### Step 3 — Add the MySQL JDBC driver

1. Download `mysql-connector-j-*.jar` from the MySQL website.
2. Place the JAR inside the `lib/` folder:

```
HospitalAMS/lib/mysql-connector-j-8.x.x.jar
```

### Step 4 — Configure database credentials

Edit `src/util/DBConnection.java` and update these three constants:

```java
private static final String URL      = "jdbc:mysql://localhost:3306/hospital_ams?useSSL=false&serverTimezone=UTC";
private static final String USER     = "root";       // ← your MySQL username
private static final String PASSWORD = "root";       // ← your MySQL password
```

### Step 5 — Compile and run

**Linux / macOS:**
```bash
chmod +x run.sh
./run.sh
```

**Windows:**
```
Double-click run.bat
```

**Manual compile (any OS):**
```bash
mkdir out

# Compile all Java sources
find src -name "*.java" | xargs javac -cp "lib/mysql-connector-j-*.jar" -d out -sourcepath src

# Start the server
java -cp "out:lib/mysql-connector-j-*.jar" HospitalServer
```

> On Windows, replace `:` with `;` in the classpath: `out;lib\mysql-connector-j-*.jar`

### Step 6 — Open in the browser

```
http://localhost:8080
```

The SPA frontend is served directly from `frontend/index.html` by the Java server's static file handler.

---

## Default Credentials

| Role | Username | Password |
|------|----------|----------|
| Admin | `admin` | `admin123` |
| Doctor | `dr.sharma` | `doc123` |
| Doctor | `dr.priya` | `doc123` |
| Doctor | `dr.mehta` | `doc123` |
| Doctor | `dr.kavya` | `doc123` |
| Patient | `patient1` | `pat123` |

> ⚠️ These are development seed credentials. Change them before any production or shared deployment.

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| `ERROR: MySQL JDBC connector JAR not found` | JAR missing from `lib/` | Download and place `mysql-connector-j-*.jar` in `lib/` |
| `Communications link failure` | MySQL not running or wrong credentials | Start MySQL, check `DBConnection.java` settings |
| `Compilation FAILED` | JDK not installed or wrong version | Verify with `java -version` (needs 11+) |
| Port 8080 already in use | Another process is using the port | Change `PORT` constant in `HospitalServer.java` or free the port |
| Blank page at `localhost:8080` | `frontend/index.html` not found | Ensure you run from the project root directory |
| Slots not appearing | Schedule not seeded | Re-run `sql/schema.sql` to re-seed schedule data |
