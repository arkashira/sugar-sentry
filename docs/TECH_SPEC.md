# TECH_SPEC.md – sugar‑sentry

> **Author:** Senior Product / Engineering Lead  
> **Repository:** https://github.com/axentx/sugar-sentry  
> **Last Updated:** 2026‑06‑12  

---

## 1. Executive Summary

`sugar‑sentry` is a **real‑time monitoring & alerting platform** designed for modern developer workflows. It ingests telemetry from distributed services, aggregates metrics, detects anomalies, and delivers actionable insights via customizable alerts. The system is built to scale horizontally, integrate with popular notification channels, and expose a clean REST/GraphQL API for internal tooling.

---

## 2. Architecture Overview

```
+-------------------+          +-------------------+          +-------------------+
|  Frontend (React) | <------> |  API Gateway      | <------> |  Backend Services |
|  (Dashboard)      |          |  (Express + JWT)  |          |  (Node.js)        |
+-------------------+          +-------------------+          +-------------------+
          |                               |                           |
          |  WebSocket / SSE              |  REST / GraphQL          |
          |                               |                           |
          |                               v                           v
          |                        +-------------------+   +-------------------+
          |                        |  Metrics Store    |   |  Alert Store      |
          |                        |  (MongoDB)        |   |  (MongoDB)        |
          |                        +-------------------+   +-------------------+
          |                               |                           |
          |                               |  Pub/Sub (RabbitMQ)      |
          |                               |                           |
          |                               v                           v
          |                        +-------------------+   +-------------------+
          |                        |  Ingestion Agent  |   |  Notification Hub |
          |                        |  (Node/Go)        |   |  (Slack, Email,   |
          |                        |  (Collects logs, |   |   Webhook)        |
          |                        |   metrics)       |   +-------------------+
          |                        +-------------------+
          |
          v
+-------------------+
|  External Systems |
|  (Databases,      |
|   Cloud APIs,     |
|   CI/CD, etc.)    |
+-------------------+
```

* **Monolith‑ish API Gateway** – Handles authentication, rate limiting, and request routing.  
* **Micro‑service‑style Backend** – Each domain (metrics, alerts, users, notifications) is a separate module, but shares the same Node.js runtime for simplicity.  
* **Event‑Driven Ingestion** – Lightweight agents ship logs/metrics to the ingestion service via gRPC or HTTP/2.  
* **Message Queue** – RabbitMQ decouples ingestion from downstream processing (alert evaluation, metric aggregation).  
* **MongoDB** – Primary data store for time‑series metrics (via capped collections) and alert metadata.  
* **WebSocket / Server‑Sent Events** – Push real‑time updates to the dashboard.  
* **CI/CD** – GitHub Actions + Docker Hub + Kubernetes (Helm charts).  

---

## 3. Core Components

| Component | Responsibility | Key Tech |
|-----------|----------------|----------|
| **Frontend** | UI for dashboards, alert rules, settings | React, TypeScript, Vite, TailwindCSS |
| **API Gateway** | Auth, routing, rate‑limit | Express, express‑jwt, express‑rate-limit |
| **Auth Service** | User & role management | Passport.js, JWT, OAuth2 (optional) |
| **Metrics Service** | Ingest, store, aggregate metrics | Node.js, Mongoose, RabbitMQ consumer |
| **Alert Service** | Rule evaluation, anomaly detection | Node.js, Mongoose, cron, machine‑learning model (optional) |
| **Notification Service** | Dispatch alerts to Slack, email, webhook | Node.js, Nodemailer, Slack SDK |
| **Ingestion Agent** | Collects logs/metrics from target services | Go (for low‑overhead), gRPC |
| **Database** | Persistent storage | MongoDB 7.x (capped collections for metrics) |
| **Message Queue** | Decouples ingestion from processing | RabbitMQ 3.12 |
| **Monitoring** | Self‑monitoring of sugar‑sentry | Prometheus, Grafana |
| **CI/CD** | Build, test, deploy | GitHub Actions, Docker, Helm |

---

## 4. Data Model

### 4.1 Collections

| Collection | Schema (partial) | Purpose |
|------------|------------------|---------|
| **users** | `{ _id, email, passwordHash, role, createdAt }` | Auth & RBAC |
| **projects** | `{ _id, name, description, ownerId, createdAt }` | Logical grouping of monitored services |
| **services** | `{ _id, projectId, name, endpoint, tags, createdAt }` | Individual monitored endpoints |
|
