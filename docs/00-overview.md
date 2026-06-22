# 00 — Project Overview

## RBAC App

Role-Based Access Control (RBAC) adalah sistem otorisasi di mana hak akses diberikan berdasarkan **peran (role)**, bukan per-user. User mendapat role, dan role memiliki permission.

### Arsitektur Aplikasi

```
┌─────────────────────────────────────────────┐
│              HTTP Request (Client)           │
├─────────────────────────────────────────────┤
│  Controller  ←  DTO Request/Response        │
├─────────────────────────────────────────────┤
│  Service (Business Logic)                   │
├─────────────────────────────────────────────┤
│  Repository (Data Access)                   │
├─────────────────────────────────────────────┤
│  Entity (Database Mapping)                  │
├─────────────────────────────────────────────┤
│  PostgreSQL (Database)                      │
└─────────────────────────────────────────────┘
```

### Stack

| Teknologi | Versi | Kegunaan |
|-----------|-------|----------|
| Java | 21 | Runtime |
| Spring Boot | 4.1.0 | Framework |
| Maven | - | Build tool |
| PostgreSQL | 16 | Database |
| Docker | - | Containerization |
| JPA / Hibernate | - | ORM (Object-Relational Mapping) |
| Spring Security | - | Auth & Authorization |
| JWT (jjwt) | 0.12.6 | Token-based auth |
| Lombok | - | Boilerplate reduction |
| Jakarta Validation | - | Input validation |

### Struktur Folder (Layer-based)

```
src/main/java/com/tiasa/rbac_app/
├── RbacAppApplication.java          # Entry point
├── common/
│   └── audit/
│       ├── BaseEntity.java          # Base class untuk semua entity
│       └── AuditAware.java          # Auto-fill createdBy/updatedBy
├── config/
│   ├── SecurityConfig.java          # Spring Security config
│   └── JpaConfig.java               # JPA Auditing config
├── entity/
│   ├── User.java                    # Entity User
│   ├── Role.java                    # Entity Role
│   └── Permission.java              # Entity Permission
├── repository/
│   ├── UserRepository.java          # Akses data User
│   ├── RoleRepository.java          # Akses data Role
│   └── PermissionRepository.java    # Akses data Permission
├── service/                         # Business logic (Fase 2)
├── controller/                      # REST API endpoints (Fase 2))
├── dto/                             # Request/Response objects (Fase 2)
└── exception/                       # Global exception handler (Fase 2)
```

### Penjelasan Setiap Layer

#### `entity/`
Tempat **JPA Entities** — class Java yang mewakili table di database. Setiap field di entity = column di table. Anotasi `@Entity` memberitahu JPA bahwa class ini harus di-mapping ke database.

```
User.java    → users table
Role.java    → roles table
Permission.java → permissions table
```

#### `repository/`
Tempat **Data Access Layer** — interface yang extends `JpaRepository`. JPA secara otomatis mengimplementasikan query CRUD (Create, Read, Update, Delete) tanpa perlu menulis SQL. Method seperti `findByEmail()` otomatis di-generate dari nama method.

#### `service/`
Tempat **Business Logic** — logika bisnis aplikasi. Misalnya: saat user register, service yang memvalidasi email belum terdaftar, hash password, lalu simpan user.

#### `controller/`
Tempat **REST API Endpoints** — menerima HTTP request dari client, memanggil service, lalu mengembalikan response.

#### `dto/`
Tempat **Data Transfer Objects** — class khusus untuk request/response API. Tidak sama dengan Entity. DTO hanya membawa data yang diperlukan client, tidak membawa field internal seperti `password` atau `createdAt`.

#### `common/audit/`
Tempat **shared/base class** yang dipakai bersama oleh banyak class lain. `BaseEntity` adalah parent class yang di-`extend` oleh semua entity. Berisi field `id`, `createdAt`, `updatedAt`, `createdBy`, `updatedBy` — sehingga tidak perlu diulang di setiap entity.

#### `config/`
Tempat konfigurasi Spring Boot via Java Config (bukan application.properties). Misalnya konfigurasi Security, JPA, CORS, dll.

#### `exception/`
Tempat custom exception class + `@RestControllerAdvice` untuk menangani error secara global.

### Docker Setup

```bash
# Build & jalankan aplikasi + database
docker compose up -d

# Akses aplikasi di:
http://localhost:8081

# Database PostgreSQL di host port 5433:
#   Host: localhost
#   Port: 5433
#   User: rbac_app
#   Pass: password
#   DB:   rbac_app
```

---

### Fase Development

| Fase | Scope |
|------|-------|
| **Fase 1** | UUID + BaseEntity + Audit + Repository + Docker setup |
| **Fase 2** | Spring Security + JWT + CRUD Service + Controller + DTO |
| **Fase 3** | Organization (multi-tenant) + Approval workflow |
