# Bulk Certificate Generator

A backend API for bulk generating certificates, built with Django and Django REST Framework.

## Objective
The system handles certificate generation requests for a list of recipients asynchronously and provides endpoints to track status and retrieve the generated certificates as PDFs.

## Technology Stack
- **Framework:** Python, Django, Django REST Framework
- **Database:** SQLite
- **Certificate Generation:** ReportLab (for generating simple and clean PDF certificates)
- **Background Processing:** Python `threading` (For this specific assignment, a simple threading implementation was chosen over Celery to minimize dependencies such as Redis/RabbitMQ, making setup much simpler while still achieving asynchronous processing).

## Prerequisites
- Python 3.8+

## Setup Instructions

1. **Create and activate a virtual environment:**
   ```bash
   python -m venv venv
   # On Windows:
   .\venv\Scripts\activate
   # On macOS/Linux:
   source venv/bin/activate
   ```

2. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

3. **Run database migrations:**
   ```bash
   python manage.py migrate
   ```

## Running the Application

Start the development server:
```bash
python manage.py runserver
```

The API will be available at `http://127.0.0.1:8000/`.

## Running Tests

Tests cover API validation, model creation, certificate generation, status tracking, failure handling, and PDF retrieval. 

Run tests with `pytest`:
```bash
pytest
```

## API Endpoints

### 1. Submit a Certificate Generation Request (Bulk)
**POST** `/api/jobs/`

**Payload:**
```json
{
    "recipients": [
        {
            "name": "John Doe",
            "email": "john@example.com",
            "course": "Python Basics"
        },
        {
            "name": "Jane Smith",
            "email": "jane@example.com",
            "course": "Django Advanced"
        }
    ]
}
```

**Response:** `201 Created`
```json
{
    "job_id": "123e4567-e89b-12d3-a456-426614174000",
    "status": "PENDING"
}
```

### 2. Check Job Status
**GET** `/api/jobs/{job_id}/`

**Response:** `200 OK`
```json
{
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "status": "COMPLETED",
    "created_at": "2023-10-09T10:00:00Z",
    "updated_at": "2023-10-09T10:00:05Z",
    "certificates": [
        {
            "id": "abcd-efgh-ijkl-mnop",
            "recipient_name": "John Doe",
            "recipient_email": "john@example.com",
            "course_name": "Python Basics",
            "status": "SUCCESS",
            "error_message": null
        },
        {
            "id": "wxyz-1234-5678-90ab",
            "recipient_name": "Jane Smith",
            "recipient_email": "jane@example.com",
            "course_name": "Django Advanced",
            "status": "FAILED",
            "error_message": "Generation error text..."
        }
    ]
}
```

### 3. Retrieve Generated Certificate
**GET** `/api/certificates/{certificate_id}/download/`

Downloads the generated PDF file if the status is `SUCCESS`.

## Important Implementation Decisions
1. **Background Processing:** I utilized native Python `threading` (`threading.Thread`) for processing jobs. When a client submits a bulk request, the main thread instantly returns a `201 Created` along with a tracking Job ID. The actual PDF generation happens in the spawned background thread. While `Celery` + `Redis` is typically used for robust enterprise asynchronous workers, standard threading provides a dependency-free, zero-configuration background runner that fulfills the requirements effectively for this assignment scope.
2. **Failure Handling at Item-Level:** When bulk processing, if one certificate fails to generate (e.g., template error), the system intercepts the exception, marks *that specific certificate's* status as `FAILED`, records the error message, and seamlessly continues processing the rest of the valid certificates in the job.
3. **Template Engine:** Used `reportlab` to natively render a vector PDF from scratch to ensure text is sharp and layout is easily adjusted without relying on external system utilities like `wkhtmltopdf`.
