**Dockerfile:**
```dockerfile
FROM python:3.12-slim AS builder
LABEL maintainer="Miory" \
      version="1.0.0" \
      description="Py app"
WORKDIR /app
# Установка зависимостей из задачи, но по факту они не нужны,
# т.к. не попадают в финальный образ
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    curl \
    wget \
    git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
RUN python -m venv /venv
ENV PATH=/venv/bin:$PATH
COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt
COPY . .
# Оба на slim, т.к. alpine и slim могут быть не совместимы
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /venv /venv
COPY --from=builder /app .
ENV PATH=/venv/bin:$PATH
RUN adduser -D pyuser && chown -R pyuser /app
USER pyuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080')" || exit 1
CMD ["python3", "app.py"]
```
