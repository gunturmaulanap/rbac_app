# RBAC App

Role-Based Access Control (RBAC) application built with **Spring Boot 4.1**, **Java 21**, and **PostgreSQL**. Runs entirely in Docker.

## Tech Stack

| Teknologi | Versi |
|-----------|-------|
| Java | 21 |
| Spring Boot | 4.1.0 |
| Maven | - |
| PostgreSQL | 16 |
| Docker | - |
| JPA / Hibernate | - |
| Spring Security | - |
| JWT (jjwt) | 0.12.6 |
| Lombok | - |

## Quick Start

```bash
# Build & jalankan aplikasi + database
docker compose up -d

# Akses aplikasi
http://localhost:8081
```

## Database

PostgreSQL berjalan di Docker, dengan port mapping `5433:5432` (host:container).

| Parameter | Value |
|-----------|-------|
| Host | `localhost` |
| Port | `5433` |
| Database | `rbac_app` |
| Username | `rbac_app` |
| Password | `password` |

## Project Structure

```
src/main/java/com/tiasa/rbac_app/
├── RbacAppApplication.java          # Entry point
├── common/audit/                    # Shared base class
│   ├── BaseEntity.java              # UUID id + audit fields
│   └── AuditAware.java              # Auto-fill createdBy/updatedBy
├── config/                          # Application configs
│   ├── SecurityConfig.java          # Spring Security
│   └── JpaConfig.java               # JPA Auditing
├── entity/                          # JPA Entities
│   ├── User.java
│   ├── Role.java
│   └── Permission.java
├── repository/                      # Data access layer
│   ├── UserRepository.java
│   ├── RoleRepository.java
│   └── PermissionRepository.java
├── service/                         # Business logic (Fase 2)
├── controller/                      # REST API (Fase 2)
├── dto/                             # Request/Response (Fase 2)
└── exception/                       # Global handler (Fase 2)
```

## Features

- **UUID** primary keys (not sequential Long) for security
- **JPA Auditing** — auto-track `createdAt`, `updatedAt`, `createdBy`, `updatedBy`
- **Entity Validation** — `@NotBlank`, `@Email`, `@Column` constraints
- **Dockerized** — PostgreSQL + app in containers
- **Many-to-Many** relationships: User ↔ Role ↔ Permission

## Development Roadmap

| Fase | Scope | Status |
|------|-------|--------|
| **Fase 1** | UUID + BaseEntity + Audit + Repository + Docker | ✅ Selesai |
| **Fase 2** | Spring Security + JWT + CRUD Service/Controller/DTO | ⏳ |
| **Fase 3** | Organization + Approval workflow | 📅 |

## Docker Commands

```bash
# Start semua service
docker compose up -d

# Lihat log aplikasi
docker compose logs -f app

# Stop (hanya container project ini)
docker compose down

# Stop + hapus volume database
docker compose down -v

# Rebuild image
docker compose up -d --build
```

## Documentation

Detailed documentation available in [`docs/`](docs/):

- [00 — Project Overview](docs/00-overview.md)
- [01 — Fase 1: UUID, BaseEntity, Audit & Repository](docs/01-fase-1.md)
