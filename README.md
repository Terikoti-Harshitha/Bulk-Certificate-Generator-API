# CertiFlow

**Bulk Certificate Generator API**

## Project Overview

CertiFlow is a small backend API. A client sends **one request containing many recipients**, and CertiFlow generates a PDF certificate for each of them from a single predefined template. Generation runs in the background, so the API replies immediately. The client can then check progress, list the results, and download each PDF.

There is no frontend. Use the built-in Swagger UI at <http://localhost:8000/docs> to try everything.

## Features

- **Bulk certificate generation** - many recipients in a single `POST` request
- **Validation** - Pydantic checks every field and returns a clear HTTP 422 on bad input
- **Predefined PDF template** - one clean landscape design drawn with ReportLab
- **Background processing** - the API responds right away; PDFs are made afterwards
- **Progress tracking** - total / successful / failed / completed counts, updated per recipient
- **Individual failure handling** - one failed recipient never stops the rest; the reason is stored
- **Certificate retrieval** - list all results of a job and download each real PDF

## Technology Stack

| Tool | Used for |
|------|----------|
| Python 3.12+ | Language |
| FastAPI + Uvicorn | REST API and server |
| SQLite | Database (a single file, no setup) |
| SQLAlchemy | Talking to the database with Python classes |
| Pydantic | Request validation and response models |
| ReportLab | Drawing the PDF certificate |
| Pytest + TestClient | Automated tests |

## Project Structure

```text
certiflow/
├── app/
│   ├── main.py                    # creates the FastAPI app, folder and tables on startup
│   ├── config.py                  # reads DATABASE_URL and CERTIFICATE_STORAGE_PATH
│   ├── database.py                # SQLAlchemy engine, session, get_db()
│   ├── models.py                  # tables: GenerationJob, Certificate
│   ├── schemas.py                 # Pydantic request / response models
│   ├── routers/
│   │   ├── jobs.py                # create job, job status, list certificates
│   │   └── certificates.py        # download a PDF
│   └── services/
│       ├── certificate_generator.py   # the PDF template (ReportLab)
│       └── job_processor.py           # background loop over all recipients
├── generated_certificates/        # PDFs are saved here (created automatically)
├── tests/                         # pytest tests
├── requirements.txt
├── pytest.ini                     # lets pytest find the `app` package
├── .env.example
├── .gitignore
└── run.py                         # optional launcher: python run.py
```

## Setup

```bash
python -m venv venv
```

Activate it:

```bash
# Windows
venv\Scripts\activate

# macOS / Linux
source venv/bin/activate
```

Install and run:

```bash
pip install -r requirements.txt
uvicorn app.main:app --reload
```

Open <http://localhost:8000/docs> for Swagger UI.

**Configuration (optional):** copy `.env.example` to `.env` to change the database file or PDF folder. The defaults are the same values, so this step can be skipped.

**Run the tests:**

```bash
pytest
```

The tests use a temporary database and folder, so they never touch your real data.

## API Documentation

### 1. Create a generation job

| | |
|---|---|
| **Method / URL** | `POST /api/jobs/` |
| **Purpose** | Submit all recipients in one request and start background generation |
| **Success status** | `202 Accepted` (the work is accepted and continues in the background) |

**Request body**

| Field | Rules |
|-------|-------|
| `course_name` | required, not blank |
| `organization_name` | required, not blank |
| `certificate_date` | valid date, `YYYY-MM-DD` |
| `recipients` | at least 1 item |
| `recipients[].name` | required, not blank |
| `recipients[].email` | valid email address |

**Example request**

```json
{
  "course_name": "Python Programming Workshop",
  "organization_name": "ABC Institute",
  "certificate_date": "2026-10-08",
  "recipients": [
    { "name": "Harshitha Terikoti", "email": "harshitha@example.com" },
    { "name": "Akhil Kumar", "email": "akhil@example.com" }
  ]
}
```

**Example response**

```json
{
  "job_id": "8f52c7a1",
  "status": "PROCESSING",
  "total_recipients": 2
}
```

**Possible errors**

- `422 Unprocessable Entity` - invalid input. Example body for an empty course name:

```json
{
  "detail": [
    {
      "type": "value_error",
      "loc": ["body", "course_name"],
      "msg": "Value error, must not be empty"
    }
  ]
}
```

### 2. Get job status

| | |
|---|---|
| **Method / URL** | `GET /api/jobs/{job_id}/` |
| **Purpose** | Check status and progress of a job |
| **Request body** | none |

**Example response**

```json
{
  "job_id": "8f52c7a1",
  "status": "COMPLETED_WITH_ERRORS",
  "total": 100,
  "successful": 98,
  "failed": 2,
  "completed": 100
}
```

**Job statuses**

| Status | Meaning |
|--------|---------|
| `PENDING` | Default value of a new job record (jobs are set to `PROCESSING` as soon as they are queued) |
| `PROCESSING` | Certificates are being generated |
| `COMPLETED` | Every certificate succeeded |
| `COMPLETED_WITH_ERRORS` | Some succeeded, some failed |
| `FAILED` | Every certificate failed, or the job itself crashed |

**Possible errors**

- `404 Not Found` - `{"detail": "Generation job not found"}`

### 3. List the certificates of a job

| | |
|---|---|
| **Method / URL** | `GET /api/jobs/{job_id}/certificates/` |
| **Purpose** | See the result for every recipient |
| **Request body** | none |

**Example response**

```json
{
  "job_id": "8f52c7a1",
  "certificates": [
    {
      "id": 1,
      "recipient_name": "Harshitha Terikoti",
      "email": "harshitha@example.com",
      "status": "SUCCESS",
      "certificate_id": "CERT-000001",
      "download_url": "/api/certificates/1/download",
      "error": null
    },
    {
      "id": 2,
      "recipient_name": "John Doe",
      "email": "john@example.com",
      "status": "FAILED",
      "certificate_id": null,
      "download_url": null,
      "error": "Certificate generation failed: <reason>"
    }
  ]
}
```

Certificate `status` is `PENDING` (not processed yet), `SUCCESS` or `FAILED`.

**Possible errors**

- `404 Not Found` - `{"detail": "Generation job not found"}`

### 4. Download a certificate

| | |
|---|---|
| **Method / URL** | `GET /api/certificates/{certificate_id}/download` |
| **Purpose** | Download the real generated PDF |
| **Request body** | none |

`{certificate_id}` is the numeric `id` from the certificate list (the same number used in `download_url`), not the `CERT-000001` text.

**Example response:** the PDF file (`Content-Type: application/pdf`), downloaded as e.g. `Harshitha_Terikoti.pdf`.

**Possible errors**

- `404 Not Found` - `{"detail": "Certificate not found"}`
- `404 Not Found` - `{"detail": "Certificate file not available"}` (the certificate failed, is still pending, or the file is missing)

## Example API Workflow

```text
Client
  |
  | POST /api/jobs/
  | sends 100 recipients
  ↓
FastAPI
  |
  | validates request
  ↓
Create Generation Job (+ one Certificate row per recipient)
  |
  | background processing
  ↓
Generate Certificate 1
Generate Certificate 2
Generate Certificate 3
...
Generate Certificate 100
  |
  ↓
Update database (after every recipient)
  |
  ↓
Client checks:
GET /api/jobs/{job_id}/
  |
  ↓
Client retrieves:
GET /api/jobs/{job_id}/certificates/
  |
  ↓
Client downloads:
GET /api/certificates/{id}/download
```

Try it with curl:

```bash
curl -X POST http://localhost:8000/api/jobs/ \
  -H "Content-Type: application/json" \
  -d '{"course_name":"Python Programming Workshop","organization_name":"ABC Institute","certificate_date":"2026-10-08","recipients":[{"name":"Harshitha Terikoti","email":"harshitha@example.com"}]}'

curl http://localhost:8000/api/jobs/<job_id>/
curl http://localhost:8000/api/jobs/<job_id>/certificates/
curl -o certificate.pdf http://localhost:8000/api/certificates/1/download
```

## How It Works (for interview explanations)

- **Database:** one `GenerationJob` has many `Certificate` rows (`job.certificates` / `certificate.job` in SQLAlchemy). The job stores the shared details (course, organization, date) and the counters. Each certificate row stores one recipient, its status, its PDF path and any error.
- **Background flow:** the `POST` handler saves the job and one `PENDING` certificate row per recipient, queues `process_generation_job(job_id)` with `BackgroundTasks`, and returns. Afterwards the worker loops over the recipients, generates each PDF, and commits after each one, so the status endpoint shows live progress.
- **Failure isolation:** inside the loop, every recipient has its own `try/except`. On error that row becomes `FAILED` with an error message and the loop continues.
- **Certificate IDs:** `CERT-` plus the certificate row's database id padded to 6 digits. A database primary key is always unique, so the IDs are unique. Failed rows use up a number too, so gaps are possible.
- **File storage:** PDFs are saved as `generated_certificates/CERT-000001.pdf`. The database stores only the path, never the PDF bytes.

## Design Decisions

**Why FastAPI?** It makes REST APIs simple to write, uses Pydantic for validation out of the box, and generates OpenAPI documentation and Swagger UI automatically, so the API can be tested without a frontend.

**Why SQLite?** This is a small assignment. SQLite needs no server or setup (it is a single file) but is still a real relational database, so the job-to-certificates relationship works properly.

**Why SQLAlchemy?** It lets us work with tables as Python classes, gives clean sessions and queries, and handles the one-to-many relationship between jobs and certificates.

**Why ReportLab?** It draws PDFs directly from Python code, so the certificate template is just a function that fills in the recipient's details. No external design tool or browser is needed.

**Why BackgroundTasks?** Generating many PDFs takes time. With `BackgroundTasks` the client gets its job ID immediately and generation continues after the response, without needing Celery or Redis.

**Why process each recipient independently?** One bad record (for example a name that cannot be drawn) should not stop everyone else from getting a certificate. Each recipient is handled in its own `try/except` and the result is recorded per recipient.

## Limitations

- Background tasks run inside the API process. If the server restarts mid-job, that job stays `PROCESSING`. A task queue such as Celery would be the next step, and was left out on purpose to keep the project simple.
- The template uses ReportLab's built-in fonts, which cover Latin characters only. A name in another script (for example Telugu) will not render correctly on the certificate.
- There is no authentication, by design (out of scope for this assignment).
