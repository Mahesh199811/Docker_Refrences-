FROM python:3.12-slim

LABEL key="value"

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5090

CMD ["python3", "app.py"]
