FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .

# Install CPU-only torch from the PyTorch index
RUN pip install --no-cache-dir torch --index-url https://download.pytorch.org/whl/cpu

RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 9000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "9000"]
