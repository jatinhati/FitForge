# FitForge 🏋️

FitForge is a full-stack **AI-powered fitness tracking platform** built with a microservices architecture. Users can log workouts, track calories, and receive personalised AI-generated recommendations powered by Google Gemini — all secured by Keycloak OAuth 2.0 / OIDC.

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
- [Getting Started](#getting-started)
  - [1. Infrastructure (Keycloak, Kafka, Databases)](#1-infrastructure-keycloak-kafka-databases)
  - [2. Backend Microservices](#2-backend-microservices)
  - [3. Frontend](#3-frontend)
- [Configuration Reference](#configuration-reference)
- [API Reference](#api-reference)
- [Project Structure](#project-structure)
- [Contributing](#contributing)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Browser                                  │
│              React + Vite + MUI Frontend (:5173)                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │  HTTP (Bearer JWT)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              API Gateway (Spring Cloud Gateway) :8080            │
│         OAuth2 Resource Server (validates JWT via Keycloak)      │
└────────┬─────────────────┬──────────────────┬───────────────────┘
         │ /api/users/**   │ /api/activities/**│ /api/recommendations/**
         ▼                 ▼                  ▼
  ┌────────────┐  ┌──────────────────┐  ┌──────────────────┐
  │ User Svc   │  │  Activity Svc    │  │    AI Svc        │
  │  :8081     │  │     :8082        │  │    :8083         │
  │ PostgreSQL │  │    MongoDB       │  │   MongoDB        │
  └────────────┘  └────────┬─────────┘  └───────┬──────────┘
                           │  Kafka              │
                           │  (activity-events)  │ Google Gemini API
                           └─────────────────────┘

      ┌──────────────────────┐   ┌──────────────────────┐
      │  Eureka Server :8761 │   │ Config Server :8888  │
      └──────────────────────┘   └──────────────────────┘

      ┌──────────────────────┐
      │  Keycloak :8181      │
      │  realm: fitness-app  │
      └──────────────────────┘
```

All backend services register with Eureka and pull their configuration from the Config Server. The API Gateway performs JWT validation and routes requests by path prefix. Activity events are published to Apache Kafka and consumed by the AI Service to generate Gemini-powered recommendations.

---

## Services

### Config Server

| Detail | Value |
|--------|-------|
| Port | `8888` |
| Module | `configserver/` |

Centralised configuration server (Spring Cloud Config, native profile). Stores per-service YAML files under `configserver/src/main/resources/config/`.

### Eureka – Service Discovery

| Detail | Value |
|--------|-------|
| Port | `8761` |
| Module | `eureka/` |
| Dashboard | `http://localhost:8761` |

Netflix Eureka server. All backend microservices register here; the gateway uses Eureka to resolve load-balanced URIs (`lb://SERVICE-NAME`).

### API Gateway

| Detail | Value |
|--------|-------|
| Port | `8080` |
| Module | `gateway/` |

Spring Cloud Gateway (reactive, WebFlux) that:
- Validates JWT tokens issued by Keycloak via the JWK Set endpoint.
- Routes requests to downstream services via Eureka load-balancing.

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

**Supported activity types:** `RUNNING`, `WALKING`, `CYCLING`, `SWIMMING`, `WEIGHT_TRAINING`, `YOGA`, `HIIT`, `CARDIO`, `STRETCHING`, `OTHER`

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

Listens on the `activity-events` Kafka topic and calls the Google Gemini API to generate personalised recommendations including improvement tips, workout suggestions, and safety notes. Results are persisted in MongoDB.

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

React SPA that authenticates with Keycloak via the **OAuth 2.0 PKCE** flow (`react-oauth2-code-pkce`). State is managed with Redux Toolkit. UI components use Material UI (MUI v7). Routing is handled by React Router v7.

**Key pages / routes:**

| Route | Component | Description |
|-------|-----------|-------------|
| `/` | Redirects | Redirects to `/activities` when logged in |
| `/activities` | `ActivityForm` + `ActivityList` | Log and view activities |
| `/activities/:id` | `ActivityDetail` | View AI recommendation for an activity |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
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

- **Java 24+** (services use `java.version=24`; userservice uses 25)
- **Maven 3.9+** (or use the included `mvnw` wrappers)
- **Node.js 20+** and **npm**
- **Docker** (recommended for running infrastructure)
- **Keycloak** instance
- **Apache Kafka** + Zookeeper
- **PostgreSQL 15+**
- **MongoDB 7+**
- **Google Gemini API key**

---

## Getting Started

### 1. Infrastructure (Keycloak, Kafka, Databases)

Start the required infrastructure. Using Docker is the easiest approach:

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

#### Keycloak Setup

1. Open `http://localhost:8181` and sign in with `admin / admin`.
2. Create a realm named **`fitness-app`**.
3. Inside the realm, create a client:
   - **Client ID:** `oauth2-pkce-client`
   - **Client authentication:** OFF (public client)
   - **Standard flow / Direct access grants:** enabled
   - **Valid redirect URIs:** `http://localhost:5173/*`
   - **Web origins:** `http://localhost:5173`
4. Create a test user and set a password.

### 2. Backend Microservices

Start the services **in this order** (each subsequent service depends on the previous):

#### Config Server
```bash
cd configserver
./mvnw spring-boot:run
# Listening on http://localhost:8888
```

#### Eureka Server
```bash
cd eureka
./mvnw spring-boot:run
# Dashboard: http://localhost:8761
```

#### User Service
```bash
cd userservice
./mvnw spring-boot:run
# Listening on http://localhost:8081
```

#### Activity Service
```bash
cd activityservice
./mvnw spring-boot:run
# Listening on http://localhost:8082
```

#### AI Service

Export your Gemini credentials first:
```bash
export GEMINI_URL=https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent
export GEMINI_KEY=<your-google-gemini-api-key>
```

```bash
cd aiservice
./mvnw spring-boot:run
# Listening on http://localhost:8083
```

#### API Gateway
```bash
cd gateway
./mvnw spring-boot:run
# Listening on http://localhost:8080
```

### 3. Frontend

```bash
cd FitForge-frontend
npm install
npm run dev
# Open http://localhost:5173
```

Click **LOGIN** to authenticate via Keycloak. Once logged in you are redirected to `/activities` where you can log workouts and view AI-generated recommendations.

---

## Configuration Reference

All per-service configuration lives in `configserver/src/main/resources/config/`.

| File | Service | Key settings |
|------|---------|-------------|
| `user-service.yml` | User Service | PostgreSQL URL/credentials, Hibernate DDL, Eureka URL, port `8081` |
| `activity-service.yml` | Activity Service | MongoDB URI (`aiactivityfitness`), Kafka bootstrap servers, Kafka topic `activity-events`, port `8082` |
| `ai-service.yml` | AI Service | MongoDB URI (`airecommendationfitness`), Kafka consumer settings, `${GEMINI_URL}`, `${GEMINI_KEY}`, port `8083` |
| `gateway-service.yml` | API Gateway | Eureka URL, Keycloak JWK-set URI, route predicates, port `8080` |

> **Security note:** The default `user-service.yml` contains a plain-text database password (`admin@123`). Replace it with a strong password and consider using environment variable substitution (`${DB_PASSWORD}`) before deploying to any non-local environment.

### Environment Variables

| Variable | Used by | Description |
|----------|---------|-------------|
| `GEMINI_URL` | AI Service | Google Gemini API endpoint URL |
| `GEMINI_KEY` | AI Service | Google Gemini API key |

---

## API Reference

All endpoints are accessed through the gateway at `http://localhost:8080`. Every request must include an `Authorization: Bearer <JWT>` header (obtained from Keycloak). The gateway extracts the user ID from the JWT and forwards it as the `X-User-ID` header to downstream services.

### User Service

```
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

```
POST   /api/activities          (X-User-ID required)
GET    /api/activities          (X-User-ID required)
```

### AI / Recommendations Service

```
GET    /api/recommendations/user/{userId}
GET    /api/recommendations/activity/{activityId}
```

---

## Project Structure

```
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
3. Run each service's tests before submitting a pull request:
   ```bash
   # Java service
   ./mvnw test

   # Frontend
   npm run lint
   ```
4. Open a pull request with a clear description of the changes.
