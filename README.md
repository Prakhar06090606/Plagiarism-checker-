Plagiarism Detection API

Project Overview

This project is a high-performance, dual-server plagiarism detection system. It allows users to upload batches of documents (PDFs, Word documents, and raw handwritten images) and cross-references them against three distinct sources:

1. Internal Batch: Other documents in the same uploaded batch.
2. Historical Database: Previously uploaded documents stored as dense vector embeddings in Pinecone.
3. The Open Internet: Live web searches using SerpApi to catch copied content from online sources.

The system utilizes a Two-Tier Filtering Architecture to ensure computational efficiency, reserving heavy AI processing (Transformer models) only for documents that pass a lightning-fast mathematical gatekeeper.

System Architecture

The project is split into two entirely decoupled servers:

Backend (Django REST Framework): Acts as the heavy-lifting NLP and OCR engine.
Frontend (Flask): A lightweight server that renders the Tailwind CSS interface and routes user uploads to the Django API.

Core Technologies

Language: Python 3.x
Frameworks: Django, Django REST Framework, Flask
Machine Learning/NLP: `sentence-transformers` (all-MiniLM-L6-v2) for dense vector embeddings, `scikit-learn` for TF-IDF.
OCR (Optical Character Recognition): Tesseract, `pytesseract`, `pdfplumber`, `python-docx`
Vector Database: Pinecone
External APIs: SerpApi (Google Search Engine Results)

Complete Setup Guide

Follow these steps to deploy the system locally.

Step 1: System Prerequisites

Before writing or running Python, you must install the underlying OCR software.

Windows: Download and install the Tesseract OCR executable. Note the installation path (usually `C:\Program Files\Tesseract-OCR\tesseract.exe`).
Mac: Run `brew install tesseract`
Linux: Run `sudo apt-get install tesseract-ocr`

Step 2: Environment Variables

Create an account on [Pinecone](https://www.pinecone.io/) and [SerpApi](https://serpapi.com/) to retrieve your free API keys. You will need to create a Pinecone index named `plagiarism-index` with a dimension of `384` and metric `cosine`.

Step 3: Backend Setup (Django)

Open a terminal and configure the heavy engine:

1. Navigate to the backend folder
cd backend

2. Create and activate a virtual environment
python -m venv venv
.\venv\Scripts\activate  # (On Mac/Linux: source venv/bin/activate)

3. Install dependencies
pip install django djangorestframework pdfplumber python-docx pytesseract pillow scikit-learn requests pinecone sentence-transformers urllib3

4. Apply database migrations
python manage.py makemigrations
python manage.py migrate

5. Start the engine
python manage.py runserver

Step 4: Frontend Setup (Flask)

Open a second, separate terminal and configure the UI:

1. Navigate to the frontend folder
cd frontend

2. Create and activate a virtual environment
python -m venv venv
.\venv\Scripts\activate

3. Install dependencies
pip install flask requests

# 4. Start the UI server
python app.py

Open your browser and navigate to `http://127.0.0.1:5000/`.


Code Explanation & Engineering Decisions

1. The Two-Tier Filtering Pipeline (`backend/api/services.py`)

Processing every document through a neural network is computationally expensive. This system uses a Gatekeeper Pattern.

Tier 1 (N-gram Gatekeeper): The system breaks text down into 5-word chunks (5-grams) and uses high-speed Python `sets` to calculate the Overlap Coefficient. If the overlap between two documents is less than 10%, the system mathematically proves they are not plagiarized and safely skips the heavy scan.
Tier 2 (Deep Scan): If a document fails the gatekeeper (>10% overlap), the system generates exact matched strings and highlights to track the specific copied sentences.

2. Dense Vector Embeddings & Pinecone

Instead of storing raw text strings in the database, the system uses Hugging Face's `all-MiniLM-L6-v2` transformer model to convert 50-word chunks into 384-dimensional floating-point vectors.

These vectors are batch-uploaded to Pinecone.
When a new document is uploaded, it is converted into vectors and queried against Pinecone using Cosine Similarity to find historical overlaps, even if the user slightly reworded the sentence.

3. Rate-Limit Protection (Exponential Backoff)

When checking the live internet, the system utilizes `urllib3.util.retry`. If SerpApi or Google returns a `429 Too Many Requests` or `500 Server Error`, the system will automatically pause and retry the request, doubling the wait time after each failure (0s, 2s, 4s). This prevents the pipeline from crashing during large batch uploads.

4. API Request Lifecycle (`backend/api/views.py`)

The `PlagiarismCheckViewSet` controls the exact order of operations to prevent false positives:

1. Extract: OCR processes the PDFs/Images.
2. External Check: The text is chunked and sent to SerpApi.
3. Historical Check: The text vectors query Pinecone for past matches.
4. Internal Check: The batch is cross-referenced against itself using the N-gram gatekeeper.
5. Index (Critical Step): Only after all checks are complete are the new document's vectors uploaded to Pinecone. If this happened earlier, the document would match against itself and return a 100% score.

5. Chunk-Level Highlighting (`frontend/templates/index.html`)

The API doesn't just return a score; it returns a JSON array of specific plagiarized text blocks. The Flask frontend utilizes JavaScript Regular Expressions (`RegExp`) to dynamically search the original text block, escape special characters, ignore layout-breaking white spaces, and inject HTML `<mark>` tags and source badges seamlessly into the modal.

Logging and Monitoring

The Django backend is equipped with a custom logging dictionary in `settings.py`. Every OCR attempt, Pinecone batch-encoding, API rate-limit warning, and processing time metric is permanently recorded with microsecond precision in `backend/plagiarism_engine.log`.

Administrators can monitor the system in real-time by running: `Get-Content plagiarism_engine.log -Wait -Tail 20`.
