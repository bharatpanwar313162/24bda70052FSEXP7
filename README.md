# Experiment 7 — Secure Backend with JWT Auth, RBAC & Encrypted Storage

## Project Structure

```
exp7/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/exp7/
    │   │   ├── SecureBackendApplication.java        ← Entry point + DB seeder
    │   │   ├── entity/
    │   │   │   ├── Role.java                        ← ROLE_USER, ROLE_ADMIN, ROLE_MODERATOR
    │   │   │   ├── User.java                        ← JPA entity + UserDetails
    │   │   │   ├── RefreshToken.java                ← Server-side token store
    │   │   │   └── ThirdPartyCredential.java        ← Encrypted OAuth credentials
    │   │   ├── repository/
    │   │   │   ├── UserRepository.java
    │   │   │   ├── RefreshTokenRepository.java
    │   │   │   └── ThirdPartyCredentialRepository.java
    │   │   ├── security/
    │   │   │   ├── JwtAuthFilter.java               ← OncePerRequestFilter (the "guard")
    │   │   │   └── SecurityConfig.java              ← Filter chain + RBAC rules
    │   │   ├── service/
    │   │   │   ├── JwtService.java                  ← Token generation & validation
    │   │   │   ├── RefreshTokenService.java         ← Token lifecycle management
    │   │   │   ├── EncryptionService.java           ← AES-256-CBC + PBKDF2
    │   │   │   ├── AuthService.java                 ← Register / Login / Refresh / Logout
    │   │   │   ├── CredentialService.java           ← Encrypted credential CRUD
    │   │   │   └── UserDetailsServiceImpl.java
    │   │   ├── controller/
    │   │   │   ├── AuthController.java              ← /api/auth/**
    │   │   │   ├── UserController.java              ← /api/user/**, /api/admin/**
    │   │   │   └── CredentialController.java        ← /api/credentials/**
    │   │   ├── dto/     (7 DTOs)
    │   │   └── exception/
    │   │       ├── TokenRefreshException.java
    │   │       └── GlobalExceptionHandler.java
    │   └── resources/
    │       └── application.properties
    └── test/
        └── SecureBackendApplicationTests.java       ← 7 integration tests
```

---

## Security Architecture

```
Incoming Request
      │
      ▼
 JwtAuthFilter          ← Extracts Bearer token, validates signature + expiry
      │
      ▼
 SecurityFilterChain    ← URL-level RBAC rules
      │
      ▼
 @PreAuthorize          ← Method-level RBAC (double guard)
      │
      ▼
  Controller → Service → Repository
```

---

## API Endpoints

### Auth (Public — no token required)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/register` | Create account |
| POST | `/api/auth/login` | Get access + refresh tokens |
| POST | `/api/auth/refresh` | Rotate tokens |
| POST | `/api/auth/logout` | Revoke refresh token |

### User (JWT required)

| Method | Endpoint | Role |
|--------|----------|------|
| GET | `/api/user/profile` | Any authenticated |
| GET | `/api/user/dashboard` | USER, ADMIN |

### Admin (JWT required)

| Method | Endpoint | Role |
|--------|----------|------|
| GET | `/api/admin/panel` | ADMIN only |
| GET | `/api/admin/users` | ADMIN only |
| GET | `/api/moderator/dashboard` | ADMIN, MODERATOR |

### Credentials (JWT required)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/credentials` | Save encrypted credential |
| GET | `/api/credentials` | List all (decrypted) |
| GET | `/api/credentials/{provider}` | Get by provider |
| DELETE | `/api/credentials/{id}` | Delete |

---

## How to Test

### Step 1 — Build & Run
```bash
./mvnw spring-boot:run
```

### Step 2 — Register a user
```bash
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","email":"alice@test.com","password":"pass123"}'
```

### Step 3 — Login (get tokens)
```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'
```

### Step 4 — Call protected endpoint
```bash
curl http://localhost:8080/api/user/profile \
  -H "Authorization: Bearer <access-token>"
```

### Step 5 — Store encrypted credential
```bash
curl -X POST http://localhost:8080/api/credentials \
  -H "Authorization: Bearer <access-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "google",
    "clientId": "my-client-id",
    "clientSecret": "very-secret-value",
    "accessToken": "ya29.access...",
    "oauthRefreshToken": "1//refresh..."
  }'
```

### Step 6 — View H2 Console
Open: http://localhost:8080/h2-console  
JDBC URL: `jdbc:h2:mem:securedb`

---

## Key Concepts Implemented

| Concept | Implementation |
|---------|---------------|
| JWT Generation | `JwtService.generateToken()` — HS256 signed, roles in payload |
| JWT Validation | `JwtService.isTokenValid()` — signature + expiry check |
| JWT Filter | `JwtAuthFilter extends OncePerRequestFilter` |
| RBAC (URL level) | `SecurityConfig.authorizeHttpRequests()` |
| RBAC (method level) | `@PreAuthorize("hasRole('ADMIN')")` |
| Password Hashing | `BCryptPasswordEncoder(strength=12)` |
| Refresh Token | UUID stored in DB, verifiable and revocable |
| Token Rotation | Old refresh token deleted on each refresh call |
| Logout | Server-side revocation via DB delete |
| AES-256 Encryption | `EncryptionService` — PBKDF2 key derivation + CBC mode |
| Stateless Sessions | `SessionCreationPolicy.STATELESS` |

---

> **Default demo accounts** seeded on startup:
> - `admin` / `admin123` — ROLE_ADMIN + ROLE_USER  
> - `user` / `user123` — ROLE_USER
