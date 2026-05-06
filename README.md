# fastapi-rabbitmq-traefik

[![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=flat-square&logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-009688?style=flat-square&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![RabbitMQ](https://img.shields.io/badge/RabbitMQ-3.12+-FF6600?style=flat-square&logo=rabbitmq&logoColor=white)](https://www.rabbitmq.com/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square&logo=docker&logoColor=white)](https://www.docker.com/)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)
[![Status](https://img.shields.io/badge/Status-Academic%20Project-blue?style=flat-square)]()
[![UPTC](https://img.shields.io/badge/UPTC-Ing.%20Sistemas-red?style=flat-square)](https://www.uptc.edu.co/)

A local microservices architecture built with FastAPI, RabbitMQ, and Traefik, orchestrated with Docker Compose. Developed as part of the Distributed Systems course at UPTC.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Usage](#usage)
- [Endpoints](#endpoints)
- [Monitoring](#monitoring)
- [Theoretical Background](#theoretical-background)
- [Known Issues and Academic Context](#known-issues-and-academic-context)

---

## Architecture Overview

This project implements a message-driven microservices architecture with four services communicating over a shared Docker network:

```
Client
  |
  v
Traefik (Reverse Proxy, port 80)
  |         |
  v         v
FastAPI   RabbitMQ Management UI (/monitor)
  |
  v
RabbitMQ Broker
  |
  v
Worker (message consumer)
```

- **Traefik** acts as the entry point, routing HTTP requests to the appropriate service based on path prefixes.
- **FastAPI** exposes a REST API that authenticates requests and publishes messages to RabbitMQ.
- **RabbitMQ** is the message broker, using a durable queue to ensure messages survive restarts.
- **Worker** is a Python consumer that reads messages from the queue and persists them to a log file.

---

## Project Structure

```
fastapi-rabbitmq-traefik/
├── api/
│   ├── app.py              # FastAPI application
│   ├── requirements.txt    # Python dependencies
│   └── Dockerfile
├── worker/
│   ├── worker.py           # RabbitMQ consumer
│   ├── requirements.txt
│   └── Dockerfile
├── traefik/
│   └── traefik.yml         # Traefik static configuration
├── docker-compose.yml
└── README.md
```

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) 20.10 or higher
- [Docker Compose](https://docs.docker.com/compose/install/) 1.29 or higher

No local Python installation is required. All services run inside containers.

---

## Getting Started

Clone the repository and start all services with a single command:

```bash
git clone https://github.com/julianReyes-dev/Parcial2_Corte2_Distri.git
cd Parcial2_Corte2_Distri
docker-compose up -d --build
```

To verify that all containers are running:

```bash
docker-compose ps
```

To stop all services:

```bash
docker-compose down
```

To stop and remove all persistent volumes (resets RabbitMQ data and message logs):

```bash
docker-compose down -v
```

---

## Configuration

Credentials and runtime parameters are set as environment variables in `docker-compose.yml`. The current defaults are intended for local development only.

| Variable              | Default   | Description                        |
|-----------------------|-----------|------------------------------------|
| `RABBITMQ_HOST`       | rabbitmq  | RabbitMQ service hostname          |
| `RABBITMQ_QUEUE`      | messages  | Queue name used by API and worker  |
| `BASIC_AUTH_USERNAME` | admin     | HTTP Basic Auth username for API   |
| `BASIC_AUTH_PASSWORD` | secret    | HTTP Basic Auth password for API   |
| `RABBITMQ_DEFAULT_USER` | admin   | RabbitMQ admin username            |
| `RABBITMQ_DEFAULT_PASS` | secret  | RabbitMQ admin password            |

> **Note:** See [Known Issues and Academic Context](#known-issues-and-academic-context) for an explanation of why credentials are hardcoded here and what the production-ready alternative looks like.

---

## Usage

### Publish a message

```bash
curl -X POST "http://localhost/api/message" \
  -H "Authorization: Basic YWRtaW46c2VjcmV0" \
  -H "Content-Type: application/json" \
  -d '{"content": "Hello from the API", "priority": 1}'
```

Expected response:

```json
{"status": "Message published to RabbitMQ"}
```

![Publish message](https://github.com/user-attachments/assets/2dd7d073-d079-4685-8804-bff3c4b4144d)

### View processed messages

Messages consumed by the worker are written to a log file inside the `messages_data` volume:

```bash
docker-compose exec worker cat /app/data/messages.log
```

Each line is a JSON object with this shape:

```json
{"timestamp": "2026-05-06T15:00:00.000000", "content": "Hello from the API", "priority": 1}
```

![Worker message log](https://github.com/user-attachments/assets/b4b79272-f3ee-4c18-9928-497630e1ea0e)

### View live worker logs

```bash
docker-compose logs -f worker
```

---

## Endpoints

### REST API (via Traefik at port 80)

| Method | Path           | Auth required | Description                        |
|--------|----------------|---------------|------------------------------------|
| POST   | /api/message   | Yes           | Publishes a message to RabbitMQ    |
| GET    | /api/health    | No            | Returns service health status      |
| GET    | /api/docs      | No            | Interactive Swagger UI             |
| GET    | /api/redoc     | No            | ReDoc API documentation            |

### Message schema

```json
{
  "content": "string (required)",
  "priority": "integer (optional, default: 1)"
}
```

### Web interfaces

| Service              | URL                          | Credentials       |
|----------------------|------------------------------|-------------------|
| API Documentation    | http://localhost/api/docs    | None              |
| RabbitMQ Management  | http://localhost/monitor     | admin / secret    |
| Traefik Dashboard    | http://localhost:8080/dashboard/ | None (insecure) |

**API Documentation (Swagger UI)**

![API docs](https://github.com/user-attachments/assets/59d59c3a-bcc9-4709-b4ee-a44de6182a74)

**RabbitMQ Management**

![RabbitMQ management](https://github.com/user-attachments/assets/eb9f68e9-7a9d-43b8-af32-ac5bc85763ef)

**Traefik Dashboard**

![Traefik dashboard](https://github.com/user-attachments/assets/39ba45f3-5bfb-41fb-8667-c191f3ffdf7a)

---

## Monitoring

### Health check

```bash
curl http://localhost/api/health
```

![Health check](https://github.com/user-attachments/assets/448de76b-e069-4f8d-bc61-074c7321d35c)

### Container logs

```bash
docker-compose logs -f api
docker-compose logs -f worker
docker-compose logs -f rabbitmq
```

### RabbitMQ metrics

The RabbitMQ Management UI at `http://localhost/monitor` provides:

- Queue depth and message rates
- Consumer count per queue
- Connection and channel status

### Traefik metrics

The Traefik dashboard at `http://localhost:8080/dashboard/` shows:

- Active routers and services
- Request routing rules
- Backend health status

---

## Theoretical Background

This section answers the theoretical questions that motivated the architecture decisions in this project.

### RabbitMQ

**What is RabbitMQ and when should you use a queue vs. a fanout exchange?**

RabbitMQ is an open-source message broker that implements the AMQP protocol. It decouples producers from consumers, enabling asynchronous communication between services.

- A **direct queue** (used in this project) is appropriate when each message must be processed exactly once, for example to handle an order or a task. If multiple workers are consuming the same queue, RabbitMQ load-balances across them automatically.
- A **fanout exchange** is appropriate when one event must be delivered to multiple independent consumers simultaneously, for example broadcasting a user registration event to a billing service and a notification service at the same time.

**What is a Dead Letter Queue (DLQ) and how is it configured in RabbitMQ?**

A Dead Letter Queue is a queue where messages are routed automatically when they cannot be processed. This happens when a message is rejected by the consumer, exceeds its TTL (time-to-live), or the target queue has reached its maximum length.

```python
# Declare the main queue with DLQ routing
args = {
    "x-dead-letter-exchange": "dlx_exchange",
    "x-dead-letter-routing-key": "dl_queue"
}
channel.queue_declare(queue='main_queue', arguments=args)

# Declare the dead letter exchange and queue
channel.exchange_declare(exchange='dlx_exchange', exchange_type='direct')
channel.queue_declare(queue='dl_queue')
channel.queue_bind(queue='dl_queue', exchange='dlx_exchange', routing_key='dl_queue')
```

This project does not implement a DLQ. Messages that fail to process are logged but not requeued, which means they are silently dropped. Adding a DLQ would be the next reliability improvement.

---

### Docker and Docker Compose

**What is the difference between a volume and a bind mount?**

A **Docker volume** is managed entirely by Docker. Its data lives under `/var/lib/docker/volumes` and persists independently of the container lifecycle. It is the recommended approach for production data.

```yaml
# docker-compose.yml
volumes:
  rabbitmq_data:
    driver: local

services:
  rabbitmq:
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
```

A **bind mount** maps a directory from the host machine directly into the container. Changes on the host are immediately reflected inside the container, which makes it useful during development.

```yaml
services:
  api:
    volumes:
      - ./api:/app   # host path : container path
```

This project uses named volumes for RabbitMQ data and message logs to ensure data persists across container restarts.

**What does `network_mode: host` imply?**

When `network_mode: host` is set, the container shares the network stack of the host machine. It uses the same network interfaces, IP addresses, and ports. This eliminates NAT overhead and can improve performance for network-intensive workloads, but it removes network isolation, can cause port conflicts with host services, and is not compatible with Docker Swarm. This project does not use host networking; all services communicate through the dedicated `microservices_net` bridge network.

---

### Traefik

**What is the role of Traefik in a microservices architecture?**

Traefik acts as a reverse proxy and API gateway. In this project it serves three main purposes:

1. It receives all incoming HTTP traffic on port 80 and routes requests to the correct service based on path prefix rules defined as Docker labels.
2. It enables service discovery by reading Docker metadata at runtime, so no manual configuration is needed when services are added or removed.
3. It provides a dashboard for observing active routes, services, and request flow.

**How can you secure an endpoint with automatic TLS certificates in Traefik?**

Traefik integrates with Let's Encrypt to provision and renew TLS certificates automatically. This requires a publicly accessible domain and the following configuration:

```yaml
# docker-compose.yml
services:
  traefik:
    image: traefik:v2.5
    command:
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.le.acme.email=your@email.com
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.le.acme.tlschallenge=true
    volumes:
      - ./letsencrypt:/letsencrypt
    ports:
      - "80:80"
      - "443:443"

  api:
    labels:
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls.certresolver=le"
      - "traefik.http.routers.api.tls.domains[0].main=yourdomain.com"
```

This is not configured in this project because it runs entirely on localhost.

---

## Known Issues and Academic Context

This section is intentionally transparent. The following decisions are not best practices for production systems. They exist because this is an academic exercise with a short delivery timeline, and understanding the tradeoff is part of the learning objective.

**Hardcoded credentials in docker-compose.yml and source code**

The username `admin` and password `secret` appear in multiple places: `docker-compose.yml`, `app.py`, and `worker.py`. In a real system, secrets should never be stored in source-controlled files. The correct approach is to use a secrets manager or Docker secrets, and reference them as environment variables that are injected at runtime:

```bash
# .env file (never committed to Git)
RABBITMQ_DEFAULT_USER=admin
RABBITMQ_DEFAULT_PASS=changeme
BASIC_AUTH_USERNAME=admin
BASIC_AUTH_PASSWORD=changeme
```

```yaml
# docker-compose.yml references the .env file automatically
environment:
  - RABBITMQ_DEFAULT_USER=${RABBITMQ_DEFAULT_USER}
  - RABBITMQ_DEFAULT_PASS=${RABBITMQ_DEFAULT_PASS}
```

A `.env.example` file with placeholder values should be committed instead of the real `.env` file, and `.env` should be listed in `.gitignore`.

**Traefik dashboard exposed without authentication (`--api.insecure=true`)**

The Traefik dashboard is accessible at port 8080 without any credentials. This is disabled in production environments. A proper setup would either disable the dashboard entirely or protect it with middleware authentication.

**New RabbitMQ connection on every API request**

`app.py` opens and closes a new connection to RabbitMQ for every POST request. This is functionally correct but inefficient. A production implementation would maintain a connection pool or a single long-lived connection managed at application startup.

**No message retry or dead letter queue**

If the worker fails to process a message due to an unexpected error, the message is logged but not requeued. Messages can be silently lost. Adding a DLQ and a retry policy would make this system reliable enough for real workloads.

**`depends_on` does not guarantee RabbitMQ is ready**

The `depends_on: rabbitmq` directive in `docker-compose.yml` only waits for the container to start, not for RabbitMQ to finish its internal initialization. The worker handles this with a retry loop, which is an acceptable workaround, but a `healthcheck` condition on the API service as well would be cleaner.

**`version` field in docker-compose.yml is deprecated**

The `version: '3.8'` key at the top of `docker-compose.yml` is ignored by modern versions of Docker Compose and will eventually cause a warning. It can be removed safely.

---

## Licencia

Este proyecto se distribuye bajo la **Licencia MIT**. Consulta el archivo [LICENSE](LICENSE) para más detalles.
