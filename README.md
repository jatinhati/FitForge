# FitForge

<p align="center">
  <!-- Simple inline SVG logo (no external assets required) -->
  <svg width="120" height="120" viewBox="0 0 120 120" fill="none" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="FitForge logo">
    <defs>
      <linearGradient id="ff_g" x1="20" y1="20" x2="100" y2="100" gradientUnits="userSpaceOnUse">
        <stop stop-color="#6D28D9"/>
        <stop offset="1" stop-color="#22C55E"/>
      </linearGradient>
    </defs>
    <rect x="10" y="10" width="100" height="100" rx="24" fill="url(#ff_g)"/>
    <!-- Dumbbell -->
    <rect x="34" y="54" width="52" height="12" rx="6" fill="white" opacity="0.95"/>
    <rect x="24" y="48" width="12" height="24" rx="6" fill="white" opacity="0.95"/>
    <rect x="84" y="48" width="12" height="24" rx="6" fill="white" opacity="0.95"/>
    <!-- Spark/AI star -->
    <path d="M62 28l3 8 8 3-8 3-3 8-3-8-8-3 8-3 3-8z" fill="white" opacity="0.95"/>
  </svg>
</p>

<h1 align="center">FitForge</h1>
<p align="center">
  <b>AI-powered fitness tracking platform</b> built with <b>Spring Boot microservices</b> + <b>Kafka</b> + <b>Keycloak</b> + a <b>React</b> frontend.
</p>

<p align="center">
  <a href="https://github.com/jatinhati/FitForge"><img alt="Repo" src="https://img.shields.io/badge/GitHub-jatinhati%2FFitForge-black"></a>
  <img alt="Backend" src="https://img.shields.io/badge/Backend-Spring%20Boot%203.x-6DB33F">
  <img alt="Gateway" src="https://img.shields.io/badge/Gateway-Spring%20Cloud%20Gateway-0EA5E9">
  <img alt="Auth" src="https://img.shields.io/badge/Auth-Keycloak-1D4ED8">
  <img alt="Messaging" src="https://img.shields.io/badge/Messaging-Apache%20Kafka-111827">
  <img alt="Frontend" src="https://img.shields.io/badge/Frontend-React%20%2B%20Vite-61DAFB">
</p>
<img width="1920" height="1080" alt="Screenshot (104)" src="https://github.com/user-attachments/assets/c8379194-6f61-465f-9316-72570ca16dd2" />
<img width="1920" height="1080" alt="Screenshot (107)" src="https://github.com/user-attachments/assets/7d1bf350-11bd-42c9-986d-3cca8743682c" />
<img width="1920" height="1080" alt="Screenshot (106)" src="https://github.com/user-attachments/assets/fcf9603d-2874-49dd-a495-ff553b7019ff" />

---

## What is FitForge?

FitForge is a full-stack **AI-powered fitness tracking platform**.

- Log workouts (type, duration, calories, and additional metrics)
- Persist activity history
- Stream activity events via **Kafka**
- Generate **personalized AI recommendations** (tips, suggestions, safety notes) via the **Google Gemini API**
- Secure everything with **Keycloak** (OIDC/OAuth 2.0) and JWT validation at the **API Gateway**

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Services](#services)
  - [Config Server](#config-server)
  - [Eureka – Service Discovery](#eureka--service-discovery)
  - [API Gateway](#api-gateway)
  - [User Service](#user-service)
  - [Activity Service](#activity-service)
  - [AI Service](#ai-service)
  - [Frontend](#frontend)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Quick Start (Docker)](#quick-start-docker)
- [Getting Started (Step-by-step)](#getting-started-step-by-step)
  - [1. Infrastructure (Keycloak, Kafka, Databases)](#1-infrastructure-keycloak-kafka-databases)
  - [2. Backend Microservices](#2-backend-microservices)
  - [3. Frontend](#3-frontend)
- [Environment Variables](#environment-variables)
- [Configuration Reference](#configuration-reference)
- [API Reference](#api-reference)
- [Project Structure](#project-structure)
- [Contributing](#contributing)

---

## Architecture Overview

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                                 Browser                                  │
│                   React + Vite + MUI Frontend (:5173)                    │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │  HTTP (Bearer JWT)
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                 API Gateway (Spring Cloud Gateway) :8080                 │
│        OAuth2 Resource Server (validates JWT via Keycloak JWKs)          │
└──────────────┬──────────────────────┬───────────────────────┬────────────┘
               │ /api/users/**        │ /api/activities/**    │ /api/recommendations/**
               ▼                      ▼                       ▼
      ┌─────────────────┐   ┌───────────────────┐    ┌───────────────────┐
      │   User Service   │   │  Activity Service  │    │     AI Service     │
      │      :8081       │   │       :8082        │    │       :8083        │
      │   PostgreSQL     │   │      MongoDB       │    │      MongoDB       │
      └─────────────────┘   └──────────┬─────────┘    └─────────┬─────────┘
                                       │  Kafka (activity-events)          │ Google Gemini API
                                       └───────────────────────────────────┘

            ┌──────────────────────┐         ┌──────────────────────┐
            │  Eureka Server :8761 │         │ Config Server :8888  │
            └──────────────────────┘         └──────────────────────┘

            ┌──────────────────────┐
            │  Keycloak :8181      │
            │  realm: fitness-app  │
            └──────────────────────┘
```

**How requests flow:**

1. The **Frontend** authenticates with **Keycloak** using **OAuth 2.0 PKCE**.
2. The **API Gateway** validates JWTs (via Keycloak JWKs) and routes traffic by path prefix.
3. The **Activity Service** publishes events to **Kafka** (`activity-events`).
4. The **AI Service** consumes those events, calls **Google Gemini**, and stores generated recommendations.

---

## Services

### Config Server

| Detail | Value |
|--------|-------|
| Port | `8888` |
| Module | `configserver/` |

Centralized configuration server (Spring Cloud Config, native profile). Stores per-service YAML under:

- `configserver/src/main/resources/config/`

### Eureka – Service Discovery

| Detail | Value |
|--------|-------|
| Port | `8761` |
| Module | `eureka/` |
| Dashboard | `http://localhost:8761` |

Netflix Eureka server. All backend services register here; the gateway uses Eureka to resolve `lb://SERVICE-NAME` URIs.

### API Gateway

| Detail | Value |
|--------|-------|
| Port | `8080` |
| Module | `gateway/` |

Spring Cloud Gateway (reactive / WebFlux):

- Validates JWT tokens issued by Keycloak (JWK Set endpoint)
- Routes requests to downstream services via Eureka load-balancing

**Route table:**

| Path Prefix | Target Service |
|-------------|---------------|
| `/api/users/**` | `USER-SERVICE` |
| `/api/activities/**` | `ACTIVITY-SERVICE` |
| `/api/recommendations/**` | `AI-SERVICE` |

### User Service

| Detail | Value |
|--------|-------|
| Port | `8081` |
| Module | `userservice/` |
| Database | PostgreSQL — `fitness-micro-user` |

Manages user profiles stored in PostgreSQL via Spring Data JPA.

**Endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/users/register` | Register a new user |
| `GET` | `/api/users/{userId}` | Fetch a user profile |
| `GET` | `/api/users/{userId}/validate` | Check whether a user exists |

**User roles:** `USER`, `ADMIN`

### Activity Service

| Detail | Value |
|--------|-------|
| Port | `8082` |
| Module | `activityservice/` |
| Database | MongoDB — `aiactivityfitness` |
| Kafka topic | `activity-events` (producer) |

Records fitness activities in MongoDB, validates the user via the User Service (WebClient), and publishes activity events to Kafka so the AI Service can generate recommendations.

**Supported activity types:**

`RUNNING`, `WALKING`, `CYCLING`, `SWIMMING`, `WEIGHT_TRAINING`, `YOGA`, `HIIT`, `CARDIO`, `STRETCHING`, `OTHER`

**Endpoints:**

| Method | Path | Headers | Description |
|--------|------|---------|-------------|
| `POST` | `/api/activities` | `X-User-ID`, `Authorization` | Log a new activity |
| `GET` | `/api/activities` | `X-User-ID`, `Authorization` | List user's activities |

**Activity payload example:**

```json
{
  "type": "RUNNING",
  "duration": 30,
  "caloriesBurned": 300,
  "startTime": "2025-05-01T07:00:00",
  "additionalMetrics": {
    "distanceKm": 5.2,
    "avgHeartRate": 145
  }
}
```

### AI Service

| Detail | Value |
|--------|-------|
| Port | `8083` |
| Module | `aiservice/` |
| Database | MongoDB — `airecommendationfitness` |
| Kafka topic | `activity-events` (consumer, group: `activity-processor-group`) |
| AI Provider | Google Gemini API |

Consumes `activity-events` and calls Gemini to generate personalized recommendations: overall analysis + improvements + suggestions + safety notes.

**Endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/recommendations/user/{userId}` | All recommendations for a user |
| `GET` | `/api/recommendations/activity/{activityId}` | Recommendation for a specific activity |

**Recommendation structure:**

```json
{
  "id": "...",
  "activityId": "...",
  "userId": "...",
  "type": "RUNNING",
  "recommendation": "Overall analysis text...",
  "improvements": ["Keep a consistent pace", "..."],
  "suggestions": ["Try interval training", "..."],
  "safety": ["Stay hydrated", "..."],
  "createdAt": "2025-05-01T07:05:00"
}
```

### Frontend

| Detail | Value |
|--------|-------|
| Port | `5173` (dev) |
| Module | `FitForge-frontend/` |
| Framework | React 19 + Vite 7 |

React SPA that authenticates with Keycloak via **OAuth 2.0 PKCE** (`react-oauth2-code-pkce`). Uses Redux Toolkit for state and Material UI (MUI v7).

**Key routes:**

| Route | Description |
|------|-------------|
| `/` | Redirects to `/activities` when logged in |
| `/activities` | Log and list activities |
| `/activities/:id` | View AI recommendation for an activity |

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | React 19, Vite 7, MUI 7, Redux Toolkit, Axios, React Router 7 |
| API Gateway | Spring Cloud Gateway (WebFlux), Spring Boot 3.5, Spring Security OAuth2 |
| User Service | Spring Boot 3.5, Spring Data JPA, PostgreSQL, Lombok |
| Activity Service | Spring Boot 3.5, Spring Data MongoDB, Apache Kafka, WebClient, Lombok |
| AI Service | Spring Boot 3.5, Spring Data MongoDB, Apache Kafka, WebClient, Google Gemini API, Lombok |
| Service Discovery | Netflix Eureka (Spring Cloud 2025.0.0) |
| Config Server | Spring Cloud Config (native) |
| Auth | Keycloak (OIDC / OAuth 2.0, PKCE) |
| Messaging | Apache Kafka |
| Databases | PostgreSQL (users), MongoDB (activities + recommendations) |
| Build | Maven (Java 24+), npm |

---

## Prerequisites

- **Java 24+** (services use `java.version=24`; `userservice` uses 25)
- **Maven 3.9+** (or use `mvnw` wrappers)
- **Node.js 20+** + **npm**
- **Docker** (recommended for local infrastructure)
- **Keycloak** instance
- **Apache Kafka** + Zookeeper
- **PostgreSQL 15+**
- **MongoDB 7+**
- **Google Gemini API key**

---

## Quick Start (Docker)

If you just want to run the dependencies quickly, start infra with Docker:

```bash
# PostgreSQL
docker run -d --name postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=admin@123 \
  -e POSTGRES_DB=fitness-micro-user \
  -p 5432:5432 postgres:15

# MongoDB
docker run -d --name mongo \
  -p 27017:27017 mongo:7

# Zookeeper + Kafka
docker run -d --name zookeeper -p 2181:2181 zookeeper:3.9
docker run -d --name kafka \
  -p 9092:9092 \
  -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  --link zookeeper apache/kafka:latest

# Keycloak (development mode)
docker run -d --name keycloak \
  -p 8181:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:latest start-dev
```

---

## Getting Started (Step-by-step)

### 1. Infrastructure (Keycloak, Kafka, Databases)

Follow the **Quick Start** commands above.

#### Keycloak Setup

1. Open `http://localhost:8181` and sign in with `admin / admin`.
2. Create a realm named **`fitness-app`**.
3. Create a client:
   - **Client ID:** `oauth2-pkce-client`
   - **Client authentication:** OFF (public client)
   - **Standard flow / Direct access grants:** enabled
   - **Valid redirect URIs:** `http://localhost:5173/*`
   - **Web origins:** `http://localhost:5173`
4. Create a test user and set a password.

### 2. Backend Microservices

Start services **in this order**:

#### Config Server

```bash
cd configserver
./mvnw spring-boot:run
# http://localhost:8888
```

#### Eureka Server

```bash
cd eureka
./mvnw spring-boot:run
# http://localhost:8761
```

#### User Service

```bash
cd userservice
./mvnw spring-boot:run
# http://localhost:8081
```

#### Activity Service

```bash
cd activityservice
./mvnw spring-boot:run
# http://localhost:8082
```

#### AI Service

Export Gemini credentials first:

```bash
export GEMINI_URL=https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent
export GEMINI_KEY=<your-google-gemini-api-key>
```

```bash
cd aiservice
./mvnw spring-boot:run
# http://localhost:8083
```

#### API Gateway

```bash
cd gateway
./mvnw spring-boot:run
# http://localhost:8080
```

### 3. Frontend

```bash
cd FitForge-frontend
npm install
npm run dev
# http://localhost:5173
```

Click **LOGIN** to authenticate via Keycloak. After login, you will land on `/activities`.

---

## Environment Variables

| Variable | Used by | Description |
|----------|---------|-------------|
| `GEMINI_URL` | AI Service | Google Gemini API endpoint URL |
| `GEMINI_KEY` | AI Service | Google Gemini API key |

---

## Configuration Reference

All per-service configuration lives in `configserver/src/main/resources/config/`.

| File | Service | Key settings |
|------|---------|-------------|
| `user-service.yml` | User Service | PostgreSQL URL/credentials, Hibernate DDL, Eureka URL, port `8081` |
| `activity-service.yml` | Activity Service | MongoDB URI (`aiactivityfitness`), Kafka bootstrap servers, topic `activity-events`, port `8082` |
| `ai-service.yml` | AI Service | MongoDB URI (`airecommendationfitness`), Kafka consumer settings, `${GEMINI_URL}`, `${GEMINI_KEY}`, port `8083` |
| `gateway-service.yml` | API Gateway | Eureka URL, Keycloak JWK-set URI, route predicates, port `8080` |

> **Security note:** Avoid committing real secrets. Prefer environment variables (`${VAR}`) or secret managers.

---

## API Reference

All endpoints are accessed through the gateway at `http://localhost:8080`.

### User Service

```text
POST   /api/users/register
GET    /api/users/{userId}
GET    /api/users/{userId}/validate
```

**Register request body:**

```json
{
  "email": "user@example.com",
  "password": "secret",
  "firstName": "Jane",
  "lastName": "Doe"
}
```

### Activity Service

```text
POST   /api/activities          (X-User-ID required)
GET    /api/activities          (X-User-ID required)
```

### AI / Recommendations Service

```text
GET    /api/recommendations/user/{userId}
GET    /api/recommendations/activity/{activityId}
```

---

## Project Structure

```text
FitForge/
├── configserver/               # Spring Cloud Config Server
│   └── src/main/resources/
│       ├── application.yml
│       └── config/             # Per-service configuration files
│           ├── user-service.yml
│           ├── activity-service.yml
│           ├── ai-service.yml
│           └── gateway-service.yml
├── eureka/                     # Netflix Eureka Server
├── gateway/                    # Spring Cloud API Gateway + OAuth2
├── userservice/                # User management (PostgreSQL + JPA)
├── activityservice/            # Activity tracking (MongoDB + Kafka producer)
├── aiservice/                  # AI recommendations (MongoDB + Kafka consumer + Gemini)
└── FitForge-frontend/          # React + Vite SPA
    └── src/
        ├── components/         # ActivityForm, ActivityList, ActivityDetail
        ├── services/           # Axios API client
        ├── store/              # Redux store + authSlice
        ├── authConfig.js       # Keycloak PKCE configuration
        └── App.jsx             # Root component + routing
```

---

## Contributing

1. Fork the repository and create a feature branch.
2. Follow the existing code style (Lombok for Java, functional components for React).
3. Run tests before submitting a PR:

```bash
# Java services
./mvnw test

# Frontend
npm run lint
```

4. Open a pull request with a clear description of the changes.
