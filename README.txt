# Microservices Containerization Project

## Overview

This project demonstrates the containerization and orchestration of a Node.js-based microservices application using Docker and Docker Compose.

### Services

| Service | Port |
|----------|------|
| User Service | 3000 |
| Product Service | 3001 |
| Gateway Service | 3003 |

---

## Project Structure

submission/
├── user-service/
│   └── Dockerfile
├── product-service/
│   └── Dockerfile
├── gateway-service/
│   └── Dockerfile
├── docker-compose.yml
└── README.md

---

## Prerequisites

- Docker
- Docker Compose

Verify installation:

docker --version
docker compose version

---

## Build Application

docker compose build

---

## Start Application

docker compose up -d

Verify running containers:

docker ps

---

## Service URLs

User Service:
http://localhost:3000

Product Service:
http://localhost:3001

Gateway Service:
http://localhost:3003

---

## Testing

### User Service

Open:
http://localhost:3000

Expected Response:

{
  "message": "User Service Running"
}

### Product Service

Open:
http://localhost:3001

Expected Response:

{
  "message": "Product Service Running"
}

### Gateway Service

Open:
http://localhost:3003

Expected Response:

Gateway Service Running

---

## View Logs

docker compose logs -f

---

## Stop Application

docker compose down

---

## Troubleshooting

### Port Already in Use

Check active processes and stop conflicting applications.

### Container Exits Immediately

Check logs:

docker logs user-service
docker logs product-service
docker logs gateway-service

### Service Communication Issues

Verify Docker network:

docker network ls

docker network inspect microservices-network

---

## Screenshots Required

1. Docker Compose running successfully.
2. Docker containers visible in Docker Desktop.
3. User Service response in browser.
4. Product Service response in browser.
5. Gateway Service response in browser.

---

## Submission Checklist

[x] Dockerfiles Created
[x] Docker Compose Configured
[x] Shared Network Configured
[x] Ports Mapped Correctly
[x] Documentation Included
[x] Testing Completed

---

## Conclusion

The Node.js microservices application has been containerized using Docker and orchestrated using Docker Compose. Services communicate through a shared bridge network and are accessible through their respective exposed ports.
