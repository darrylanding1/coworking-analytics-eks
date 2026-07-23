# Use AWS ECR Public mirror of python:3.10-slim — avoids Docker Hub rate limits
FROM public.ecr.aws/docker/library/python:3.10-slim

# Set working directory inside the container
WORKDIR /app

# Install OS-level build deps for psycopg2 compilation, then clean up apt cache to keep image small
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy dependency file first for better Docker layer caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application source
COPY . .

# App listens on this port (matches APP_PORT default in app.py)
EXPOSE 5153

# Run the Flask app
CMD ["python", "app.py"]
