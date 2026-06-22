# 01 — Fase 1: UUID, BaseEntity, Audit & Repository

## Daftar Isi

1. [UUID vs Long Auto-increment](#1-uuid-vs-long-auto-increment)
2. [BaseEntity & Inheritance (Pewarisan)](#2-baseentity--inheritance-pewarisan)
3. [JPA Auditing (Audit Trail)](#3-jpa-auditing-audit-trail)
4. [Entity Validation Best Practice](#4-entity-validation-best-practice)
5. [Repository Pattern](#5-repository-pattern)

---

## 1. UUID vs Long Auto-increment

### Konsep

`@GeneratedValue(strategy = GenerationType.IDENTITY)` menginstruksikan database untuk meng-generate ID secara otomatis (auto-increment 1, 2, 3...). Ini adalah default yang paling umum di tutorial.

**Masalah `Long` auto-increment untuk enterprise:**

1. **Keamanan (Enumeration Attack)** — ID 1, 2, 3 bisa ditebak. Attacker bisa mengakses resource dengan mengubah ID di URL: `/users/1`, `/users/2`, `/users/3`.
2. **Distributed System** — Jika nanti aplikasi dipecah jadi microservice, auto-increment bisa menyebabkan konflik ID antar service.
3. **Data Exposure** — ID sequential memberikan informasi tentang jumlah data (misal: ID user 1000 = sudah ada 1000 user).

**Solusi: UUID (Universally Unique Identifier)**

UUID adalah string 128-bit yang di-generate secara random. Contoh: `a1b2c3d4-e5f6-7890-abcd-ef1234567890`.

| Aspek | Long (auto-increment) | UUID |
|-------|----------------------|------|
| Contoh | 1, 2, 3, ... | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` |
| Bisa ditebak | Ya | Tidak |
| Ukuran database | 8 bytes | 16 bytes |
| Performa index | Lebih cepat | Sedikit lebih lambat |
| Cocok distributed | Tidak | Ya |

### Implementasi di Java

```java
// ❌ SEBELUM (Long auto-increment)
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// ✅ SESUDAH (UUID — otomatis di BaseEntity)
@Id
private UUID id;
```

UUID di-generate otomatis di `@PrePersist`:

```java
@PrePersist
public void prePersist() {
    if (id == null) {
        id = UUID.randomUUID();  // generate UUID sebelum insert ke DB
    }
}
```

**Penjelasan:**
- `@PrePersist` — method yang dijalankan OTOMATIS oleh JPA **sebelum** data disimpan ke database.
- `UUID.randomUUID()` — static method Java yang menghasilkan UUID unik secara acak.
- Kenapa di `BaseEntity`? — Karena semua entity butuh ID, jadi logic generate UUID cukup ditulis sekali di parent class.

---

## 2. BaseEntity & Inheritance (Pewarisan)

### Konsep Inheritance di Java

Inheritance (pewarisan) adalah konsep OOP di mana sebuah class mewarisi field dan method dari class lain. Keyword `extends` digunakan untuk inheritance.

```java
// Parent class
public class Animal {
    private String name;
    public void eat() { ... }
}

// Child class → mewarisi name + eat()
public class Dog extends Animal {
    public void bark() { ... }
}

// Dog bisa: dog.name, dog.eat(), dog.bark()
```

### Inheritance di JPA

Di JPA, ada beberapa strategi inheritance. Yang paling sederhana adalah **`@MappedSuperclass`**:

| Anotasi | Fungsi |
|---------|--------|
| `@MappedSuperclass` | Parent class TIDAK punya table sendiri. Field-nya di-inherit ke child class yang punya table masing-masing |
| `@Entity` (inherit) | Child class punya table sendiri dengan field dari parent + field sendiri |

```
@MappedSuperclass (BaseEntity)
    ├── @Entity (User) → users table (id, createdAt, ..., fullName, email, ...)
    ├── @Entity (Role) → roles table (id, createdAt, ..., name, description, ...)
    └── @Entity (Permission) → permissions table (id, createdAt, ..., name, ...)
```

### Implementasi BaseEntity

```java
@Getter
@Setter
@SuperBuilder                      // Builder pattern dengan inheritance support
@MappedSuperclass                   // Field di sini akan di-inherit ke child
@EntityListeners(AuditingEntityListener.class)  // Aktifkan audit otomatis
public abstract class BaseEntity {  // abstract = tidak bisa di-instantiate langsung

    @Id
    private UUID id;                // Primary key — dipakai semua entity

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;  // Diisi otomatis saat INSERT

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;  // Diisi otomatis saat UPDATE

    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private String createdBy;        // Diisi otomatis siapa yang INSERT

    @LastModifiedBy
    @Column(name = "updated_by")
    private String updatedBy;        // Diisi otomatis siapa yang UPDATE

    protected BaseEntity() {        // JPA butuh no-arg constructor
    }

    @PrePersist
    public void prePersist() {
        if (id == null) {
            id = UUID.randomUUID();  // Generate UUID sebelum INSERT
        }
    }
}
```

### Implementasi Entity (Child)

```java
@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@SuperBuilder                       // WAJIB @SuperBuilder (bukan @Builder)!
public class User extends BaseEntity {  // ← extends BaseEntity

    @NotBlank
    @Column(nullable = false)
    private String fullName;

    @Email
    @NotBlank
    @Column(nullable = false, unique = true)
    private String email;

    // ... field lain
}
```

**Poin penting:**
- `extends BaseEntity` — User mewarisi `id`, `createdAt`, `updatedAt`, `createdBy`, `updatedBy`
- `@SuperBuilder` — Lombok annotation khusus untuk builder pattern dengan class inheritance. Berbeda dengan `@Builder` biasa.
- `@NoArgsConstructor` — JPA butuh constructor tanpa argumen untuk membuat instance entity
- `@AllArgsConstructor` — Lombok generate constructor dengan semua field (termasuk field dari BaseEntity)

### Kenapa @SuperBuilder bukan @Builder?

```java
// ❌ @Builder — hanya untuk class tanpa parent
@Builder
public class User {
    private String name;
}
// User.builder().name("Budi").build() ✓

// ✅ @SuperBuilder — untuk class yang extends parent class
@SuperBuilder
public class User extends BaseEntity {
    private String name;
}
// User.builder().id(uuid).name("Budi").createdAt(now).build() ✓
//           ↑                ↑                    ↑
//      field dari BaseEntity   |             field dari BaseEntity
//                        field dari User
```

---

## 3. JPA Auditing (Audit Trail)

### Konsep

Audit trail adalah catatan **siapa membuat** dan **kapan** sebuah data dibuat/diubah. Di enterprise, audit adalah keharusan untuk compliance dan debugging.

### Komponen Audit

| Annotation | Field | Terisi saat |
|-----------|-------|-------------|
| `@CreatedDate` | `createdAt` | INSERT (pertama kali simpan) |
| `@LastModifiedDate` | `updatedAt` | INSERT + UPDATE |
| `@CreatedBy` | `createdBy` | INSERT |
| `@LastModifiedBy` | `updatedBy` | INSERT + UPDATE |

### Cara Kerja

```
1. @EntityListeners(AuditingEntityListener.class)
   → Spring secara otomatis "mendengarkan" event INSERT/UPDATE

2. @EnableJpaAuditing (di JpaConfig.java)
   → Mengaktifkan fitur auditing di Spring

3. AuditAware.java
   → Memberitahu Spring siapa "user saat ini" yang melakukan operasi
```

### Implementasi AuditAware

```java
@Component
public class AuditAware implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        // Fase 1: sementara return "SYSTEM"
        // Fase 2: akan diubah untuk mengambil username dari JWT token
        // SecurityContextHolder.getContext().getAuthentication().getName()
        return Optional.of("SYSTEM");
    }
}
```

### Implementasi JpaConfig

```java
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditAware")
//                    ↑ nama bean dari AuditAware (nama method atau class dengan huruf kecil pertama)
public class JpaConfig {
}
```

**Kenapa perlu `auditorAwareRef = "auditAware"`?**
- Spring perlu tahu bean mana yang menyediakan informasi "current user"
- `auditAwareRef` merujuk ke nama bean `AuditAware` (Spring otomatis membuat bean dari class yang punya `@Component`, dengan nama camelCase: `auditAware`)

---

## 4. Entity Validation Best Practice

### Konsep

Validasi memastikan data yang masuk ke database sesuai aturan bisnis. Di Java, validasi dilakukan dengan **Jakarta Bean Validation** (`jakarta.validation.constraints`).

### Anotasi Validasi yang Dipakai

| Anotasi | Fungsi | Contoh |
|---------|--------|--------|
| `@NotBlank` | String tidak boleh null dan tidak boleh kosong | `@NotBlank String fullName` |
| `@Email` | String harus format email valid | `@Email String email` |
| `@NotNull` | Field tidak boleh null | `@NotNull Long id` |
| `@Size(min, max)` | Panjang string dibatasi | `@Size(min = 6, max = 20) String password` |
| `@Positive` | Angka harus positif | `@Positive BigDecimal amount` |

### Implementasi di Entity

```java
@Entity
public class User {

    @NotBlank(message = "Nama lengkap wajib diisi")
    @Column(nullable = false)
    private String fullName;

    @Email(message = "Format email tidak valid")
    @NotBlank(message = "Email wajib diisi")
    @Column(nullable = false, unique = true)
    private String email;

    @NotBlank(message = "Password wajib diisi")
    @Column(nullable = false)
    private String password;
}
```

### Tingkatan Validasi

```
Layer 1: Database (via @Column constraints)
  → @Column(nullable = false)     → NOT NULL di SQL
  → @Column(unique = true)        → UNIQUE constraint di SQL
  → @Column(length = 100)         → VARCHAR(100) di SQL

Layer 2: JPA Entity (via Jakarta Validation)
  → @NotBlank, @Email, @Size      → Validasi Java-side sebelum query SQL

Layer 3: Controller (via @Valid di DTO)
  → @Valid pada Request body       → Validasi sebelum masuk Service (Fase 2)
```

**Best Practice:**
1. Selalu pasang validasi di **entity** sebagai safety net
2. Tambahkan validasi di **DTO** (Fase 2) untuk error message yang lebih user-friendly
3. Gunakan kombinasi `@Column` + annotation validation — database constraint sebagai lapisan terakhir

---

## 5. Repository Pattern

### Konsep

**Repository** adalah pattern yang memisahkan logika akses data dari business logic. Di Spring Data JPA, cukup membuat **interface** yang extends `JpaRepository`, maka semua implementasi CRUD otomatis tersedia.

### JpaRepository

```java
public interface UserRepository extends JpaRepository<User, UUID> {
    //                         Entity ↑     ↑ ID type
    // Method CRUD otomatis:
    //   save(), findById(), findAll(), deleteById(), count(), ...
}
```

### Method Query dari Nama Method

Spring Data JPA bisa meng-generate query hanya dari **nama method**:

| Method Name | SQL yang di-generate |
|-------------|---------------------|
| `findByEmail(String email)` | `SELECT * FROM users WHERE email = ?` |
| `findByName(String name)` | `SELECT * FROM roles WHERE name = ?` |
| `existsByEmail(String email)` | `SELECT COUNT(*) FROM users WHERE email = ?` |
| `findByFullNameContaining(String name)` | `SELECT * FROM users WHERE full_name LIKE %?%` |
| `deleteByEmail(String email)` | `DELETE FROM users WHERE email = ?` |

### Implementasi

```java
// UserRepository.java
public interface UserRepository extends JpaRepository<User, UUID> {

    // SELECT * FROM users WHERE email = ?
    Optional<User> findByEmail(String email);

    // SELECT COUNT(*) FROM users WHERE email = ?
    boolean existsByEmail(String email);
}
```

**Kenapa return type `Optional<User>`?**
- `Optional` adalah container yang bisa berisi value (`Optional.of(user)`) atau kosong (`Optional.empty()`)
- Mencegah `NullPointerException` — caller WAJIB mengecek apakah data ada atau tidak
- Best practice modern Java, menghindari return `null`

### Cara Pakai Repository

```java
// Akan dipakai di Service layer (Fase 2)
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User findByEmail(String email) {
        return userRepository.findByEmail(email)
            .orElseThrow(() -> new RuntimeException("User tidak ditemukan"));
        //      ↑ Jika Optional kosong, throw exception
    }
}
```

### Relasi Antar Entity

```
User ──ManyToMany──► Role ──ManyToMany──► Permission
  │                     │
  │  @JoinTable:        │  @JoinTable:
  │  user_roles         │  role_permissions
  │  user_id → users    │  role_id → roles
  │  role_id → roles    │  permission_id → permissions
```

Implementasi di Java:

```java
// User.java — relasi ke Role
@ManyToMany(fetch = FetchType.EAGER)
@JoinTable(
    name = "user_roles",                          // Nama tabel perantara
    joinColumns = @JoinColumn(name = "user_id"),         // FK ke table users
    inverseJoinColumns = @JoinColumn(name = "role_id")   // FK ke table roles
)
private Set<Role> roles = new HashSet<>();

// Role.java — relasi ke Permission
@ManyToMany(fetch = FetchType.EAGER)
@JoinTable(
    name = "role_permissions",                     // Nama tabel perantara
    joinColumns = @JoinColumn(name = "role_id"),          // FK ke table roles
    inverseJoinColumns = @JoinColumn(name = "permission_id")  // FK ke table permissions
)
private Set<Permission> permissions = new HashSet<>();
```

**Penjelasan `@ManyToMany`:**
- Relasi many-to-many: 1 user bisa punya banyak role, 1 role bisa dipakai banyak user
- JPA membutuhkan **tabel perantara** (join table) di database
- `user_id` dan `role_id` adalah foreign key (FK) yang mereferensi ke primary key table masing-masing
- `new HashSet<>()` — inisialisasi default agar tidak null saat menambah data

---

## Diagram Database (setelah Fase 1)

```
┌─────────────────────────────────────────────────────┐
│                    users                             │
├─────────────────────────────────────────────────────┤
│ id (UUID)          ← PRIMARY KEY                     │
│ full_name (VARCHAR) NOT NULL                         │
│ email (VARCHAR)    NOT NULL UNIQUE                   │
│ password (VARCHAR) NOT NULL                          │
│ enabled (BOOLEAN)  DEFAULT true                      │
│ created_at (TIMESTAMP)                               │
│ updated_at (TIMESTAMP)                               │
│ created_by (VARCHAR)                                 │
│ updated_by (VARCHAR)                                 │
└───────────────────────┬─────────────────────────────┘
                        │
         ┌──────────────┴──────────────┐
         │       user_roles            │
         │  user_id (FK → users.id)    │
         │  role_id (FK → roles.id)    │
         └──────────────┬──────────────┘
                        │
         ┌──────────────┴──────────────┐
         │         roles               │
         ├─────────────────────────────┤
         │ id (UUID)  ← PRIMARY KEY    │
         │ name (VARCHAR) NOT NULL UNIQUE│
         │ description (VARCHAR)       │
         │ ... audit fields            │
         └──────────────┬──────────────┘
                        │
         ┌──────────────┴──────────────┐
         │     role_permissions        │
         │  role_id (FK → roles.id)    │
         │  permission_id (FK → ...)   │
         └──────────────┬──────────────┘
                        │
         ┌──────────────┴──────────────┐
         │      permissions            │
         ├─────────────────────────────┤
         │ id (UUID)  ← PRIMARY KEY    │
         │ name (VARCHAR) NOT NULL UNIQUE│
         │ description (VARCHAR)       │
         │ ... audit fields            │
         └─────────────────────────────┘
```

## Cara Verifikasi Fase 1

```bash
# 1. Compile project (pastikan tidak ada error)
./mvnw compile -q

# 2. Jalankan Docker
docker compose up -d

# 3. Cek log aplikasi
docker compose logs -f app

# 4. Cek tabel di database via DBeaver
#    Host: localhost:5433
#    User: rbac_app
#    DB:   rbac_app
#    Seharusnya tabel: users, roles, permissions, user_roles, role_permissions
#    Sudah auto-created oleh Hibernate ddl-auto=update
```
