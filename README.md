# Court File Indexer Backend

FastAPI backend, PostgreSQL database, Redis broker, Qdrant vector store, and Celery workers for the Court File Indexer.

## Manual Ubuntu Run

These commands assume the backend and frontend repositories are cloned side-by-side under one project folder, with shared runtime storage at the project root:

```bash
mkdir -p ~/Court-file-indexer
cd ~/Court-file-indexer

git clone https://github.com/priyanshuiscoding/backend_Court_indexer.git
git clone https://github.com/priyanshuiscoding/Frontend_Court_indexer.git

mkdir -p storage/pdfs storage/rendered storage/ocr storage/exports storage/logs storage/config storage/library
```

Install system packages:

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip postgresql postgresql-contrib redis-server tesseract-ocr tesseract-ocr-eng tesseract-ocr-hin poppler-utils libgl1 libglib2.0-0 curl
sudo systemctl enable --now postgresql redis-server
```

Create the PostgreSQL database and user:

```bash
sudo -u postgres psql
```

Inside `psql`:

```sql
CREATE USER court_user WITH PASSWORD 'court_pass';
CREATE DATABASE court_indexer OWNER court_user;
GRANT ALL PRIVILEGES ON DATABASE court_indexer TO court_user;
\q
```

Install and start Qdrant manually:

```bash
cd ~/Court-file-indexer
curl -L https://github.com/qdrant/qdrant/releases/latest/download/qdrant-x86_64-unknown-linux-gnu.tar.gz -o qdrant.tar.gz
mkdir -p qdrant-bin qdrant-storage
tar -xzf qdrant.tar.gz -C qdrant-bin
./qdrant-bin/qdrant --storage-dir ./qdrant-storage
```

In a new terminal, configure and install the backend:

```bash
cd ~/Court-file-indexer/backend_Court_indexer
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
cp .env.ubuntu.example .env
```

Edit `.env` and set real values for `SECRET_KEY`, `CLIENT_API_KEY`, and any High Court MySQL fields if you use import from MySQL.

Start the API:

```bash
cd ~/Court-file-indexer/backend_Court_indexer
source .venv/bin/activate
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

Start Celery workers in separate terminals:

```bash
cd ~/Court-file-indexer/backend_Court_indexer
source .venv/bin/activate
celery -A app.tasks.celery_app.celery_app worker -Q index_queue --loglevel=INFO --concurrency=2 --hostname=index@%h
```

```bash
cd ~/Court-file-indexer/backend_Court_indexer
source .venv/bin/activate
celery -A app.tasks.celery_app.celery_app worker -Q vector_queue --loglevel=INFO --concurrency=1 --hostname=vector@%h
```

Start Celery beat if scheduled queue recovery or High Court scheduled imports are needed:

```bash
cd ~/Court-file-indexer/backend_Court_indexer
source .venv/bin/activate
celery -A app.tasks.celery_app.celery_app beat --loglevel=INFO
```

Health check:

```bash
curl http://localhost:8000/api/v1/health
```

## Storage

The manual env points storage to `../storage` because the backend runs from `backend_Court_indexer/` and the shared storage folder lives in the parent project root. Create it before starting the API:

```bash
cd ~/Court-file-indexer
mkdir -p storage/pdfs storage/rendered storage/ocr storage/exports storage/logs storage/config storage/library
```

If you use document mapping, place the Excel file here:

```bash
~/Court-file-indexer/storage/config/document_mapping.xlsx
```

The app also creates several storage subfolders automatically, but creating them up front avoids permission and missing-directory errors.
