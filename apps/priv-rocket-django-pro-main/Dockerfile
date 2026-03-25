# ---- Stage 1 : Build frontend ----
FROM node:18-slim AS frontend

WORKDIR /app
COPY package*.json ./
RUN npm i
COPY . .
RUN npm run build

# ---- Stage 2 : Build Python ----
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

COPY . .
COPY --from=frontend /app/static /app/static

RUN python manage.py collectstatic --no-input && \
    python manage.py makemigrations && \
    python manage.py migrate

EXPOSE 5005

CMD ["gunicorn", "--config", "gunicorn-cfg.py", "core.wsgi"]
