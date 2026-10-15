# build
FROM python:3.11-slim-bookworm@sha256:0a310eeecf4e1f5a0743f9a6520c90c88d089c903ca5fd283f501e3a805f5f89 AS build
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/app/deps -r requirements.txt
# runtime
FROM gcr.io/distroless/python3-debian12:nonroot@sha256:7d1042ce588ab97019fe95c24ffca7bc5a82ccdac572511d5e09bda4435c89c5
WORKDIR /app
COPY --from=build /app/deps ./deps
COPY locustfile.py .
ENV PYTHONPATH=/app/deps
ENV LOCUST_HEADLESS=true
ENV LOCUST_USERS=10
ENV LOCUST_SPAWN_RATE=1
USER 65532
CMD ["-m", "locust"]