# 🚆 IRCTC Backend - Microservices Architecture

> A production-grade microservices-based railway booking system backend, built for learning and demonstrating enterprise architecture patterns.

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Port Reference](#-port-reference)
- [Tech Stack](#-tech-stack)
- [Getting Started](#-getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
  - [Running the Application](#running-the-application)
- [Services](#-services)
- [Kafka Topics](#-kafka-topics)
- [API Documentation](#-api-documentation)
- [Environment Variables](#-environment-variables)
- [Project Structure](#-project-structure)

---

## 🎯 Overview

This project demonstrates a complete microservices architecture for a railway booking system (IRCTC clone), covering:

- **Microservices Design Patterns** — Database-per-service, API Gateway, Saga
- **Inter-Service Communication** — REST (sync) and Kafka (async/event-driven)
- **Authentication & Authorization** — JWT (access + refresh), OTP via email, Google OAuth
- **Database Management** — PostgreSQL via Prisma ORM (5 services), Elasticsearch (search)
- **Caching & Locking** — Redis for sessions, OTP storage, distributed seat locks
- **Containerization** — Docker Compose for the full infra stack
- **Resilience** — Rate limiting, circuit breakers, Dead-Letter Queues (DLQ) on every Kafka consumer

**🎓 Learning Objectives:**
- Build scalable microservices with database-per-service ownership
- Implement real-world authentication flows (OTP + JWT + Google OAuth)
- Coordinate distributed transactions with Saga + Kafka
- Handle concurrent seat reservations with Redis distributed locks
- Build a production-style API Gateway with proxying, JWT enforcement, rate limiting, and circuit breakers

---

## 🏗️ Architecture

```
                            ┌───────────────────────┐
                            │  Frontend (React)     │
                            │  Port 3000            │
                            └───────────┬───────────┘
                                        │
                            ┌───────────▼───────────┐
                            │   API Gateway         │
                            │   Port 4000           │
                            │ (JWT, rate limit,     │
                            │  circuit breaker)     │
                            └───┬───────┬───────┬───┘
              ┌─────────────────┼───────┼───────┼───────────────┐
              │                 │       │       │               │
   ┌──────────▼──────┐ ┌────────▼─┐ ┌───▼────┐ ┌▼──────────┐ ┌──▼──────────┐
   │ User Service    │ │ Search   │ │ Admin  │ │ Booking   │ │ Payment     │
   │ Port 4001       │ │ 4002     │ │ 4003   │ │ 4005      │ │ 4006        │
   └────────┬────────┘ └────┬─────┘ └───┬────┘ └─────┬─────┘ └─────┬───────┘
            │               │           │            │             │
            │          ┌────▼──────┐    │     ┌──────▼─────┐       │
            │          │ Inventory │    │     │ Notification│      │
            │          │ 4007      │    │     │ 4004 (Kafka)│      │
            │          └────┬──────┘    │     └──────┬──────┘      │
            │               │           │            │             │
            └───────────────┴───────────┴────────────┴─────────────┘
                                      │
                ┌─────────────────────┼─────────────────────┐
                │                     │                     │
        ┌───────▼──────┐    ┌─────────▼────────┐   ┌────────▼────────┐
        │  PostgreSQL  │    │  Redis Stack     │   │   Kafka         │
        │  Port 5432   │    │  Port 6379/8001  │   │   Port 9092/9093│
        └──────────────┘    └──────────────────┘   └─────────────────┘
                                                            │
                                                  ┌─────────▼─────────┐
                                                  │  Elasticsearch    │
                                                  │  Port 9200        │
                                                  └───────────────────┘
```

**Highlights:**
- **API Gateway** is the single entrypoint for the frontend; it proxies to the right service and enforces JWT, rate limits, and circuit breakers.
- **Database-per-service** — each service owns its own Postgres database, search-service uses Elasticsearch.
- **Event-driven** — Kafka decouples booking, payment, inventory, search and notification flows. Topics are centralized in [shared/constants/kafka-topics.js](shared/constants/kafka-topics.js).
- **Notification service has no HTTP API** — it is purely a Kafka consumer that sends emails via SendGrid.

---

## 🔌 Port Reference

A single quick-glance table of every port the project uses.

### Application Services

| Service | Port | Direct URL |
|---|---|---|
| Frontend (Vite + React) | 3000 | http://localhost:3000 |
| API Gateway | 4000 | http://localhost:4000 |
| User Service | 4001 | http://localhost:4001 |
| Search Service | 4002 | http://localhost:4002 |
| Admin Service | 4003 | http://localhost:4003 |
| Notification Service | 4004 | (Kafka-only, no HTTP) |
| Booking Service | 4005 | http://localhost:4005 |
| Payment Service | 4006 | http://localhost:4006 |
| Inventory Service | 4007 | http://localhost:4007 |

### Infrastructure (from [docker-compose.yml](docker-compose.yml))

| Component | Port(s) | Access |
|---|---|---|
| PostgreSQL 15 | 5432 | `admin` / `irctcpass` |
| pgAdmin | 8081 | http://localhost:8081 — `admin@admin.com` / `admin` |
| Redis Stack | 6379 (Redis), 8001 (RedisInsight) | password `irctcpass` — RedisInsight at http://localhost:8001 |
| Zookeeper | 2181 | — |
| Kafka | 9092 (internal), 9093 (host) | host clients connect to `localhost:9093` |
| Kafka UI | 8080 | http://localhost:8080 |
| Elasticsearch 8.12 | 9200 | http://localhost:9200 (single-node, security disabled) |
| Kibana 8.12 | 5601 | http://localhost:5601 |

---

