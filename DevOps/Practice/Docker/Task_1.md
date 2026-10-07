**Dockerfile:**
```dockerfile
FROM python:3.11-slim
LABEL maintainer="Miory" \
      description="Docker image for Task 1 of the project" \
      version="1.0"
      
WORKDIR /app

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    STUDENT_NAME="Miory"
    
RUN groupadd -r pyuser && useradd -r -g pyuser -u 1001 pyuser && \
    chown -R pyuser:pyuser /app
    
COPY requirements.txt .

RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt && \
    apt-get purge -y --auto-remove gcc && \
    rm -rf /var/lib/apt/lists/*
    
COPY . .

USER 1001
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')" || exit 1
    
CMD ["python", "app.py"]

```

**Проверка:**
```
docker build -t myapp .
docker run -d --name myap -p 8080:8080 myapp
docker ps
curl http://localhost:8080
```
