# Lab 2 buổi chiều: Dockerfile
FROM python:3.13-alpine
RUN pip install flask prometheus-flask-exporter gunicorn
COPY app.py /app/app.py
WORKDIR /app
ENV FLASK_APP=app.py
EXPOSE 8080
CMD ["gunicorn", "--workers=3", "--bind=0.0.0.0:8080", "app:app"]
