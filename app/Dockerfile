FROM python:3.12-alpine

WORKDIR /app
COPY app.py .

EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:8080/health || exit 1

CMD ["python", "app.py"]
