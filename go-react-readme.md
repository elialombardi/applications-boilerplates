# Web Application Scaffolding Prompt

## 1. Application Placeholders

> **Fill in the following before generating the application:**

| Placeholder | Value |
|---|---|
| `{{APP_NAME}}` | `MyApp` |
| `{{APP_FOCUS}}` | `A task management platform for small teams` |
| `{{APP_FEATURES}}` | `User registration/login with JWT, dashboard with items CRUD, per-user data isolation` |
| `{{MODULE_PATH}}` | `github.com/yourorg/myapp` |
| `{{DB_NAME}}` | `myapp_db` |
| `{{DB_USER}}` | `myapp_user` |
| `{{DB_PASSWORD}}` | `myapp_pass` |
| `{{JWT_SECRET}}` | `change-me-in-production` |

---

## 2. Folder Structure

```
{{APP_NAME}}/
├── backend/
│   ├── go.mod
│   ├── go.sum
│   ├── Makefile
│   ├── .env.example
│   ├── cmd/
│   │   ├── api/
│   │   │   └── main.go              # Fiber v3 HTTP entrypoint
│   │   └── worker/
│   │       └── main.go              # Console/worker entrypoint
│   ├── internal/
│   │   ├── config/
│   │   │   └── config.go            # Env loading
│   │   ├── httpserver/
│   │   │   ├── server.go            # Fiber app bootstrap, middleware
│   │   │   ├── routes.go            # Route registration
│   │   │   ├── auth_handler.go      # Controllers (HTTP → DTO → service)
│   │   │   ├── item_handler.go
│   │   │   └── middleware/
│   │   │       └── jwt.go           # JWT auth middleware
│   │   └── worker/
│   │       └── jobs.go              # Worker jobs (uses service layer)
│   └── pkg/                         # Shared library (imported by api & worker)
│       ├── db/
│       │   └── db.go                # GORM + Postgres connection
│       ├── models/
│       │   ├── user.go
│       │   └── item.go
│       ├── dto/
│       │   ├── auth.go
│       │   └── item.go
│       ├── service/
│       │   ├── auth_service.go      # Business logic
│       │   └── item_service.go
│       └── auth/
│           └── jwt.go               # Token generation/parsing
│
└── frontend/
    ├── package.json
    ├── vite.config.ts
    ├── tsconfig.json
    ├── index.html
    └── src/
        ├── main.tsx
        ├── App.tsx
        ├── theme.ts                 # MUI dark theme
        ├── store/
        │   ├── index.ts             # Redux store
        │   └── api.ts               # RTK Query base API + auth slice
        ├── features/
        │   ├── auth/
        │   │   ├── authSlice.ts
        │   │   ├── authApi.ts
        │   │   ├── LoginPage.tsx
        │   │   └── RegisterPage.tsx
        │   └── items/
        │       ├── itemsApi.ts
        │       └── DashboardPage.tsx
        ├── components/
        │   └── ProtectedRoute.tsx
        └── router.tsx
```

---

## 3. Backend

### 3.1 `backend/go.mod`

```go
module {{MODULE_PATH}}/backend

go 1.22

require (
    github.com/gofiber/fiber/v3 v3.0.0-beta.3
    github.com/golang-jwt/jwt/v5 v5.2.1
    golang.org/x/crypto v0.27.0
    gorm.io/driver/postgres v1.5.9
    gorm.io/gorm v1.25.12
    github.com/joho/godotenv v1.5.1
)
```

### 3.2 `backend/.env.example`

```env
APP_ENV=development
HTTP_ADDR=:8080
DATABASE_URL=postgres://{{DB_USER}}:{{DB_PASSWORD}}@localhost:5432/{{DB_NAME}}?sslmode=disable
JWT_SECRET={{JWT_SECRET}}
JWT_TTL_HOURS=24
```

### 3.3 `backend/internal/config/config.go`

```go
package config

import (
    "os"
    "strconv"
    "time"

    "github.com/joho/godotenv"
)

type Config struct {
    Env         string
    HTTPAddr    string
    DatabaseURL string
    JWTSecret   string
    JWTTTL      time.Duration
}

func Load() *Config {
    _ = godotenv.Load()
    ttl, _ := strconv.Atoi(getEnv("JWT_TTL_HOURS", "24"))
    return &Config{
        Env:         getEnv("APP_ENV", "development"),
        HTTPAddr:    getEnv("HTTP_ADDR", ":8080"),
        DatabaseURL: getEnv("DATABASE_URL", ""),
        JWTSecret:   getEnv("JWT_SECRET", "dev-secret"),
        JWTTTL:      time.Duration(ttl) * time.Hour,
    }
}

func getEnv(k, def string) string {
    if v := os.Getenv(k); v != "" {
        return v
    }
    return def
}
```

### 3.4 `backend/pkg/models/user.go`

```go
package models

import "time"

type User struct {
    ID           uint      `gorm:"primaryKey" json:"id"`
    Email        string    `gorm:"uniqueIndex;size:255;not null" json:"email"`
    PasswordHash string    `gorm:"not null" json:"-"`
    CreatedAt    time.Time `json:"createdAt"`
    UpdatedAt    time.Time `json:"updatedAt"`
}
```

### 3.5 `backend/pkg/models/item.go`

```go
package models

import "time"

type Item struct {
    ID        uint      `gorm:"primaryKey" json:"id"`
    UserID    uint      `gorm:"index;not null" json:"userId"`
    Title     string    `gorm:"size:255;not null" json:"title"`
    Body      string    `gorm:"type:text" json:"body"`
    CreatedAt time.Time `json:"createdAt"`
    UpdatedAt time.Time `json:"updatedAt"`
}
```

### 3.6 `backend/pkg/db/db.go`

```go
package db

import (
    "log"

    "{{MODULE_PATH}}/backend/pkg/models"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func Connect(dsn string) *gorm.DB {
    g, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatalf("db connect: %v", err)
    }
    if err := g.AutoMigrate(&models.User{}, &models.Item{}); err != nil {
        log.Fatalf("db migrate: %v", err)
    }
    return g
}
```

### 3.7 `backend/pkg/dto/auth.go`

```go
package dto

type RegisterRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=6"`
}

type LoginRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required"`
}

type AuthResponse struct {
    Token string `json:"token"`
    User  UserDTO `json:"user"`
}

type UserDTO struct {
    ID    uint   `json:"id"`
    Email string `json:"email"`
}
```

### 3.8 `backend/pkg/dto/item.go`

```go
package dto

type CreateItemRequest struct {
    Title string `json:"title" validate:"required,max=255"`
    Body  string `json:"body"`
}

type UpdateItemRequest struct {
    Title string `json:"title" validate:"required,max=255"`
    Body  string `json:"body"`
}

type ItemDTO struct {
    ID    uint   `json:"id"`
    Title string `json:"title"`
    Body  string `json:"body"`
}
```

### 3.9 `backend/pkg/auth/jwt.go`

```go
package auth

import (
    "errors"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
    UserID uint `json:"uid"`
    jwt.RegisteredClaims
}

func Generate(secret string, userID uint, ttl time.Duration) (string, error) {
    c := Claims{
        UserID: userID,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(ttl)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    return jwt.NewWithClaims(jwt.SigningMethodHS256, c).SignedString([]byte(secret))
}

func Parse(secret, token string) (*Claims, error) {
    t, err := jwt.ParseWithClaims(token, &Claims{}, func(t *jwt.Token) (any, error) {
        return []byte(secret), nil
    })
    if err != nil {
        return nil, err
    }
    if c, ok := t.Claims.(*Claims); ok && t.Valid {
        return c, nil
    }
    return nil, errors.New("invalid token")
}
```

### 3.10 `backend/pkg/service/auth_service.go`

```go
package service

import (
    "errors"
    "time"

    "{{MODULE_PATH}}/backend/pkg/auth"
    "{{MODULE_PATH}}/backend/pkg/dto"
    "{{MODULE_PATH}}/backend/pkg/models"
    "golang.org/x/crypto/bcrypt"
    "gorm.io/gorm"
)

type AuthService struct {
    db        *gorm.DB
    jwtSecret string
    jwtTTL    time.Duration
}

func NewAuthService(db *gorm.DB, secret string, ttl time.Duration) *AuthService {
    return &AuthService{db: db, jwtSecret: secret, jwtTTL: ttl}
}

func (s *AuthService) Register(req dto.RegisterRequest) (*dto.AuthResponse, error) {
    var existing models.User
    if err := s.db.Where("email = ?", req.Email).First(&existing).Error; err == nil {
        return nil, errors.New("email already registered")
    }
    hash, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
    if err != nil {
        return nil, err
    }
    u := models.User{Email: req.Email, PasswordHash: string(hash)}
    if err := s.db.Create(&u).Error; err != nil {
        return nil, err
    }
    return s.issue(u)
}

func (s *AuthService) Login(req dto.LoginRequest) (*dto.AuthResponse, error) {
    var u models.User
    if err := s.db.Where("email = ?", req.Email).First(&u).Error; err != nil {
        return nil, errors.New("invalid credentials")
    }
    if err := bcrypt.CompareHashAndPassword([]byte(u.PasswordHash), []byte(req.Password)); err != nil {
        return nil, errors.New("invalid credentials")
    }
    return s.issue(u)
}

func (s *AuthService) issue(u models.User) (*dto.AuthResponse, error) {
    token, err := auth.Generate(s.jwtSecret, u.ID, s.jwtTTL)
    if err != nil {
        return nil, err
    }
    return &dto.AuthResponse{
        Token: token,
        User:  dto.UserDTO{ID: u.ID, Email: u.Email},
    }, nil
}
```

### 3.11 `backend/pkg/service/item_service.go`

```go
package service

import (
    "errors"

    "{{MODULE_PATH}}/backend/pkg/dto"
    "{{MODULE_PATH}}/backend/pkg/models"
    "gorm.io/gorm"
)

type ItemService struct{ db *gorm.DB }

func NewItemService(db *gorm.DB) *ItemService { return &ItemService{db: db} }

func (s *ItemService) List(userID uint) ([]dto.ItemDTO, error) {
    var items []models.Item
    if err := s.db.Where("user_id = ?", userID).Order("id desc").Find(&items).Error; err != nil {
        return nil, err
    }
    out := make([]dto.ItemDTO, 0, len(items))
    for _, it := range items {
        out = append(out, dto.ItemDTO{ID: it.ID, Title: it.Title, Body: it.Body})
    }
    return out, nil
}

func (s *ItemService) Create(userID uint, req dto.CreateItemRequest) (*dto.ItemDTO, error) {
    it := models.Item{UserID: userID, Title: req.Title, Body: req.Body}
    if err := s.db.Create(&it).Error; err != nil {
        return nil, err
    }
    return &dto.ItemDTO{ID: it.ID, Title: it.Title, Body: it.Body}, nil
}

func (s *ItemService) Update(userID, id uint, req dto.UpdateItemRequest) (*dto.ItemDTO, error) {
    var it models.Item
    if err := s.db.Where("id = ? AND user_id = ?", id, userID).First(&it).Error; err != nil {
        return nil, errors.New("item not found")
    }
    it.Title, it.Body = req.Title, req.Body
    if err := s.db.Save(&it).Error; err != nil {
        return nil, err
    }
    return &dto.ItemDTO{ID: it.ID, Title: it.Title, Body: it.Body}, nil
}

func (s *ItemService) Delete(userID, id uint) error {
    res := s.db.Where("id = ? AND user_id = ?", id, userID).Delete(&models.Item{})
    if res.Error != nil {
        return res.Error
    }
    if res.RowsAffected == 0 {
        return errors.New("item not found")
    }
    return nil
}
```

### 3.12 `backend/internal/httpserver/middleware/jwt.go`

```go
package middleware

import (
    "strings"

    "{{MODULE_PATH}}/backend/pkg/auth"
    "github.com/gofiber/fiber/v3"
)

func JWT(secret string) fiber.Handler {
    return func(c fiber.Ctx) error {
        h := c.Get("Authorization")
        if !strings.HasPrefix(h, "Bearer ") {
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{"error": "missing token"})
        }
        claims, err := auth.Parse(secret, strings.TrimPrefix(h, "Bearer "))
        if err != nil {
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{"error": "invalid token"})
        }
        c.Locals("userID", claims.UserID)
        return c.Next()
    }
}
```

### 3.13 `backend/internal/httpserver/auth_handler.go`

```go
package httpserver

import (
    "{{MODULE_PATH}}/backend/pkg/dto"
    "{{MODULE_PATH}}/backend/pkg/service"
    "github.com/gofiber/fiber/v3"
)

type AuthHandler struct{ svc *service.AuthService }

func NewAuthHandler(s *service.AuthService) *AuthHandler { return &AuthHandler{svc: s} }

func (h *AuthHandler) Register(c fiber.Ctx) error {
    var req dto.RegisterRequest
    if err := c.Bind().Body(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": "invalid body"})
    }
    res, err := h.svc.Register(req)
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": err.Error()})
    }
    return c.Status(fiber.StatusCreated).JSON(res)
}

func (h *AuthHandler) Login(c fiber.Ctx) error {
    var req dto.LoginRequest
    if err := c.Bind().Body(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": "invalid body"})
    }
    res, err := h.svc.Login(req)
    if err != nil {
        return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{"error": err.Error()})
    }
    return c.JSON(res)
}
```

### 3.14 `backend/internal/httpserver/item_handler.go`

```go
package httpserver

import (
    "strconv"

    "{{MODULE_PATH}}/backend/pkg/dto"
    "{{MODULE_PATH}}/backend/pkg/service"
    "github.com/gofiber/fiber/v3"
)

type ItemHandler struct{ svc *service.ItemService }

func NewItemHandler(s *service.ItemService) *ItemHandler { return &ItemHandler{svc: s} }

func (h *ItemHandler) List(c fiber.Ctx) error {
    uid := c.Locals("userID").(uint)
    items, err := h.svc.List(uid)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": err.Error()})
    }
    return c.JSON(items)
}

func (h *ItemHandler) Create(c fiber.Ctx) error {
    uid := c.Locals("userID").(uint)
    var req dto.CreateItemRequest
    if err := c.Bind().Body(&req); err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "invalid body"})
    }
    it, err := h.svc.Create(uid, req)
    if err != nil {
        return c.Status(400).JSON(fiber.Map{"error": err.Error()})
    }
    return c.Status(201).JSON(it)
}

func (h *ItemHandler) Update(c fiber.Ctx) error {
    uid := c.Locals("userID").(uint)
    id, _ := strconv.Atoi(c.Params("id"))
    var req dto.UpdateItemRequest
    if err := c.Bind().Body(&req); err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "invalid body"})
    }
    it, err := h.svc.Update(uid, uint(id), req)
    if err != nil {
        return c.Status(404).JSON(fiber.Map{"error": err.Error()})
    }
    return c.JSON(it)
}

func (h *ItemHandler) Delete(c fiber.Ctx) error {
    uid := c.Locals("userID").(uint)
    id, _ := strconv.Atoi(c.Params("id"))
    if err := h.svc.Delete(uid, uint(id)); err != nil {
        return c.Status(404).JSON(fiber.Map{"error": err.Error()})
    }
    return c.SendStatus(204)
}
```

### 3.15 `backend/internal/httpserver/routes.go`

```go
package httpserver

import (
    "{{MODULE_PATH}}/backend/internal/httpserver/middleware"
    "github.com/gofiber/fiber/v3"
)

func RegisterRoutes(app *fiber.App, cfg *AppDeps) {
    api := app.Group("/api")

    auth := api.Group("/auth")
    auth.Post("/register", cfg.AuthHandler.Register)
    auth.Post("/login", cfg.AuthHandler.Login)

    items := api.Group("/items", middleware.JWT(cfg.JWTSecret))
    items.Get("/", cfg.ItemHandler.List)
    items.Post("/", cfg.ItemHandler.Create)
    items.Put("/:id", cfg.ItemHandler.Update)
    items.Delete("/:id", cfg.ItemHandler.Delete)
}
```

### 3.16 `backend/internal/httpserver/server.go`

```go
package httpserver

import (
    "log"

    "{{MODULE_PATH}}/backend/internal/config"
    "{{MODULE_PATH}}/backend/pkg/service"
    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/fiber/v3/middleware/cors"
    "github.com/gofiber/fiber/v3/middleware/logger"
)

type AppDeps struct {
    AuthHandler *AuthHandler
    ItemHandler *ItemHandler
    JWTSecret   string
}

func New(cfg *config.Config, authSvc *service.AuthService, itemSvc *service.ItemService) *fiber.App {
    app := fiber.New(fiber.Config{AppName: "{{APP_NAME}}"})
    app.Use(logger.New())
    app.Use(cors.New(cors.Config{
        AllowOrigins: []string{"http://localhost:5173"},
        AllowHeaders: []string{"Origin", "Content-Type", "Authorization"},
    }))

    app.Get("/health", func(c fiber.Ctx) error {
        return c.JSON(fiber.Map{"status": "ok"})
    })

    RegisterRoutes(app, &AppDeps{
        AuthHandler: NewAuthHandler(authSvc),
        ItemHandler: NewItemHandler(itemSvc),
        JWTSecret:   cfg.JWTSecret,
    })

    log.Printf("listening on %s", cfg.HTTPAddr)
    return app
}
```

### 3.17 `backend/cmd/api/main.go`

```go
package main

import (
    "log"

    "{{MODULE_PATH}}/backend/internal/config"
    "{{MODULE_PATH}}/backend/internal/httpserver"
    "{{MODULE_PATH}}/backend/pkg/db"
    "{{MODULE_PATH}}/backend/pkg/service"
)

func main() {
    cfg := config.Load()
    g := db.Connect(cfg.DatabaseURL)

    authSvc := service.NewAuthService(g, cfg.JWTSecret, cfg.JWTTTL)
    itemSvc := service.NewItemService(g)

    app := httpserver.New(cfg, authSvc, itemSvc)
    if err := app.Listen(cfg.HTTPAddr); err != nil {
        log.Fatal(err)
    }
}
```

### 3.18 `backend/cmd/worker/main.go`

```go
package main

import (
    "log"
    "time"

    "{{MODULE_PATH}}/backend/internal/config"
    "{{MODULE_PATH}}/backend/internal/worker"
    "{{MODULE_PATH}}/backend/pkg/db"
    "{{MODULE_PATH}}/backend/pkg/service"
)

func main() {
    cfg := config.Load()
    g := db.Connect(cfg.DatabaseURL)

    itemSvc := service.NewItemService(g)
    w := worker.New(itemSvc)

    log.Println("worker started")
    ticker := time.NewTicker(60 * time.Second)
    defer ticker.Stop()
    for range ticker.C {
        w.RunOnce()
    }
}
```

### 3.19 `backend/internal/worker/jobs.go`

```go
package worker

import (
    "log"

    "{{MODULE_PATH}}/backend/pkg/service"
)

type Worker struct{ items *service.ItemService }

func New(items *service.ItemService) *Worker { return &Worker{items: items} }

func (w *Worker) RunOnce() {
    // Placeholder scheduled job. Uses the same service layer as the API.
    log.Println("worker tick")
}
```

### 3.20 `backend/Makefile`

```makefile
.PHONY: api worker tidy

api:
	go run ./cmd/api

worker:
	go run ./cmd/worker

tidy:
	go mod tidy
```

---

## 4. Frontend

### 4.1 `frontend/package.json`

```json
{
  "name": "{{APP_NAME}}-frontend",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@emotion/react": "^11.13.0",
    "@emotion/styled": "^11.13.0",
    "@mui/material": "^6.1.0",
    "@reduxjs/toolkit": "^2.2.7",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-redux": "^9.1.2",
    "react-router-dom": "^6.26.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.5",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.1",
    "typescript": "^5.5.4",
    "vite": "^5.4.0"
  }
}
```

### 4.2 `frontend/vite.config.ts`

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': 'http://localhost:8080',
    },
  },
});
```

### 4.3 `frontend/src/theme.ts`

```ts
import { createTheme } from '@mui/material/styles';

export const theme = createTheme({
  palette: {
    mode: 'dark',
    primary: { main: '#7c4dff' },
    background: { default: '#0e1116', paper: '#161b22' },
  },
  shape: { borderRadius: 10 },
});
```

### 4.4 `frontend/src/store/api.ts`

```ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const api = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({
    baseUrl: '/api',
    prepareHeaders: (headers, { getState }) => {
      const token = (getState() as any).auth.token as string | null;
      if (token) headers.set('Authorization', `Bearer ${token}`);
      return headers;
    },
  }),
  tagTypes: ['Items'],
  endpoints: () => ({}),
});
```

### 4.5 `frontend/src/features/auth/authSlice.ts`

```ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface AuthState {
  token: string | null;
  user: { id: number; email: string } | null;
}

const persisted = localStorage.getItem('auth');
const initial: AuthState = persisted ? JSON.parse(persisted) : { token: null, user: null };

const slice = createSlice({
  name: 'auth',
  initialState: initial,
  reducers: {
    setCredentials(state, action: PayloadAction<{ token: string; user: { id: number; email: string } }>) {
      state.token = action.payload.token;
      state.user = action.payload.user;
      localStorage.setItem('auth', JSON.stringify(state));
    },
    logout(state) {
      state.token = null;
      state.user = null;
      localStorage.removeItem('auth');
    },
  },
});

export const { setCredentials, logout } = slice.actions;
export default slice.reducer;
```

### 4.6 `frontend/src/features/auth/authApi.ts`

```ts
import { api } from '../../store/api';

interface AuthResponse {
  token: string;
  user: { id: number; email: string };
}

export const authApi = api.injectEndpoints({
  endpoints: (build) => ({
    register: build.mutation<AuthResponse, { email: string; password: string }>({
      query: (body) => ({ url: '/auth/register', method: 'POST', body }),
    }),
    login: build.mutation<AuthResponse, { email: string; password: string }>({
      query: (body) => ({ url: '/auth/login', method: 'POST', body }),
    }),
  }),
});

export const { useRegisterMutation, useLoginMutation } = authApi;
```

### 4.7 `frontend/src/features/auth/LoginPage.tsx`

```tsx
import { useState } from 'react';
import { Box, Button, Paper, TextField, Typography, Link } from '@mui/material';
import { Link as RouterLink, useNavigate } from 'react-router-dom';
import { useDispatch } from 'react-redux';
import { useLoginMutation } from './authApi';
import { setCredentials } from './authSlice';

export default function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [login, { isLoading, error }] = useLoginMutation();
  const dispatch = useDispatch();
  const navigate = useNavigate();

  const onSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      const res = await login({ email, password }).unwrap();
      dispatch(setCredentials(res));
      navigate('/');
    } catch {}
  };

  return (
    <Box display="flex" justifyContent="center" alignItems="center" minHeight="100vh">
      <Paper sx={{ p: 4, width: 360 }}>
        <Typography variant="h5" gutterBottom>Sign in</Typography>
        <form onSubmit={onSubmit}>
          <TextField fullWidth label="Email" margin="normal" value={email} onChange={(e) => setEmail(e.target.value)} />
          <TextField fullWidth label="Password" type="password" margin="normal" value={password} onChange={(e) => setPassword(e.target.value)} />
          {error && <Typography color="error" variant="body2">Login failed</Typography>}
          <Button type="submit" fullWidth variant="contained" sx={{ mt: 2 }} disabled={isLoading}>Login</Button>
        </form>
        <Typography variant="body2" sx={{ mt: 2 }}>
          No account? <Link component={RouterLink} to="/register">Register</Link>
        </Typography>
      </Paper>
    </Box>
  );
}
```

### 4.8 `frontend/src/features/auth/RegisterPage.tsx`

```tsx
import { useState } from 'react';
import { Box, Button, Paper, TextField, Typography, Link } from '@mui/material';
import { Link as RouterLink, useNavigate } from 'react-router-dom';
import { useDispatch } from 'react-redux';
import { useRegisterMutation } from './authApi';
import { setCredentials } from './authSlice';

export default function RegisterPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [register, { isLoading, error }] = useRegisterMutation();
  const dispatch = useDispatch();
  const navigate = useNavigate();

  const onSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      const res = await register({ email, password }).unwrap();
      dispatch(setCredentials(res));
      navigate('/');
    } catch {}
  };

  return (
    <Box display="flex" justifyContent="center" alignItems="center" minHeight="100vh">
      <Paper sx={{ p: 4, width: 360 }}>
        <Typography variant="h5" gutterBottom>Create account</Typography>
        <form onSubmit={onSubmit}>
          <TextField fullWidth label="Email" margin="normal" value={email} onChange={(e) => setEmail(e.target.value)} />
          <TextField fullWidth label="Password" type="password" margin="normal" value={password} onChange={(e) => setPassword(e.target.value)} />
          {error && <Typography color="error" variant="body2">Registration failed</Typography>}
          <Button type="submit" fullWidth variant="contained" sx={{ mt: 2 }} disabled={isLoading}>Register</Button>
        </form>
        <Typography variant="body2" sx={{ mt: 2 }}>
          Already have an account? <Link component={RouterLink} to="/login">Login</Link>
        </Typography>
      </Paper>
    </Box>
  );
}
```

### 4.9 `frontend/src/features/items/itemsApi.ts`

```ts
import { api } from '../../store/api';

export interface Item {
  id: number;
  title: string;
  body: string;
}

export const itemsApi = api.injectEndpoints({
  endpoints: (build) => ({
    listItems: build.query<Item[], void>({
      query: () => '/items',
      providesTags: ['Items'],
    }),
    createItem: build.mutation<Item, { title: string; body: string }>({
      query: (body) => ({ url: '/items', method: 'POST', body }),
      invalidatesTags: ['Items'],
    }),
    deleteItem: build.mutation<void, number>({
      query: (id) => ({ url: `/items/${id}`, method: 'DELETE' }),
      invalidatesTags: ['Items'],
    }),
  }),
});

export const { useListItemsQuery, useCreateItemMutation, useDeleteItemMutation } = itemsApi;
```

### 4.10 `frontend/src/features/items/DashboardPage.tsx`

```tsx
import { useState } from 'react';
import {
  AppBar, Box, Button, Container, IconButton, List, ListItem,
  ListItemText, TextField, Toolbar, Typography,
} from '@mui/material';
import DeleteIcon from '@mui/icons-material/Delete';
import LogoutIcon from '@mui/icons-material/Logout';
import { useDispatch } from 'react-redux';
import { useNavigate } from 'react-router-dom';
import { useCreateItemMutation, useDeleteItemMutation, useListItemsQuery } from './itemsApi';
import { logout } from '../auth/authSlice';

export default function DashboardPage() {
  const { data: items = [] } = useListItemsQuery();
  const [createItem] = useCreateItemMutation();
  const [deleteItem] = useDeleteItemMutation();
  const [title, setTitle] = useState('');
  const dispatch = useDispatch();
  const navigate = useNavigate();

  const add = async () => {
    if (!title.trim()) return;
    await createItem({ title, body: '' });
    setTitle('');
  };

  const signOut = () => {
    dispatch(logout());
    navigate('/login');
  };

  return (
    <Box>
      <AppBar position="static" color="transparent" elevation={0}>
        <Toolbar>
          <Typography variant="h6" sx={{ flexGrow: 1 }}>{{APP_NAME}}</Typography>
          <IconButton onClick={signOut}><LogoutIcon /></IconButton>
        </Toolbar>
      </AppBar>
      <Container sx={{ mt: 4 }}>
        <Typography variant="h4" gutterBottom>Dashboard</Typography>
        <Box display="flex" gap={2} mb={3}>
          <TextField fullWidth label="New item title" value={title} onChange={(e) => setTitle(e.target.value)} />
          <Button variant="contained" onClick={add}>Add</Button>
        </Box>
        <List>
          {items.map((it) => (
            <ListItem
              key={it.id}
              secondaryAction={
                <IconButton edge="end" onClick={() => deleteItem(it.id)}><DeleteIcon /></IconButton>
              }
            >
              <ListItemText primary={it.title} secondary={it.body} />
            </ListItem>
          ))}
        </List>
      </Container>
    </Box>
  );
}
```

### 4.11 `frontend/src/components/ProtectedRoute.tsx`

```tsx
import { Navigate } from 'react-router-dom';
import { useSelector } from 'react-redux';
import { ReactNode } from 'react';

export default function ProtectedRoute({ children }: { children: ReactNode }) {
  const token = useSelector((s: any) => s.auth.token);
  return token ? <>{children}</> : <Navigate to="/login" replace />;
}
```

### 4.12 `frontend/src/store/index.ts`

```ts
import { configureStore } from '@reduxjs/toolkit';
import { api } from './api';
import authReducer from '../features/auth/authSlice';

export const store = configureStore({
  reducer: {
    [api.reducerPath]: api.reducer,
    auth: authReducer,
  },
  middleware: (g) => g.concat(api.middleware),
});

export type RootState = ReturnType<typeof store.getState>;
```

### 4.13 `frontend/src/router.tsx`

```tsx
import { createBrowserRouter } from 'react-router-dom';
import LoginPage from './features/auth/LoginPage';
import RegisterPage from './features/auth/RegisterPage';
import DashboardPage from './features/items/DashboardPage';
import ProtectedRoute from './components/ProtectedRoute';

export const router = createBrowserRouter([
  { path: '/login', element: <LoginPage /> },
  { path: '/register', element: <RegisterPage /> },
  { path: '/', element: <ProtectedRoute><DashboardPage /></ProtectedRoute> },
]);
```

### 4.14 `frontend/src/App.tsx`

```tsx
import { RouterProvider } from 'react-router-dom';
import { router } from './router';

export default function App() {
  return <RouterProvider router={router} />;
}
```

### 4.15 `frontend/src/main.tsx`

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { Provider } from 'react-redux';
import { ThemeProvider, CssBaseline } from '@mui/material';
import { store } from './store';
import { theme } from './theme';
import App from './App';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <Provider store={store}>
      <ThemeProvider theme={theme}>
        <CssBaseline />
        <App />
      </ThemeProvider>
    </Provider>
  </React.StrictMode>
);
```

### 4.16 `frontend/index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>{{APP_NAME}}</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

---

## 5. Initial CLI Commands

### 5.1 Folder + backend init

```bash
mkdir -p {{APP_NAME}} && cd {{APP_NAME}}
mkdir -p backend frontend

# Backend
cd backend
go mod init {{MODULE_PATH}}/backend
mkdir -p cmd/api cmd/worker internal/config internal/httpserver/middleware internal/worker pkg/db pkg/models pkg/dto pkg/service pkg/auth

go get github.com/gofiber/fiber/v3@latest
go get github.com/golang-jwt/jwt/v5
go get golang.org/x/crypto/bcrypt
go get gorm.io/gorm
go get gorm.io/driver/postgres
go get github.com/joho/godotenv

# create files as per spec above, then:
go mod tidy
```

### 5.2 Frontend init

```bash
cd ../frontend
npm create vite@latest . -- --template react-ts
npm install
npm install @mui/material @emotion/react @emotion/styled @mui/icons-material
npm install @reduxjs/toolkit react-redux react-router-dom

# create files as per spec above
```

### 5.3 Postgres (Docker)

```bash
docker run --name {{APP_NAME}}-pg \
  -e POSTGRES_USER={{DB_USER}} \
  -e POSTGRES_PASSWORD={{DB_PASSWORD}} \
  -e POSTGRES_DB={{DB_NAME}} \
  -p 5432:5432 -d postgres:16
```

### 5.4 Run debug

```bash
# Terminal 1 - backend API (http://localhost:8080)
cd backend
cp .env.example .env
make api

# Terminal 2 - worker
cd backend
make worker

# Terminal 3 - frontend (http://localhost:5173)
cd frontend
npm run dev
```

### 5.5 Production build

```bash
# Backend
cd backend
go build -o bin/api ./cmd/api
go build -o bin/worker ./cmd/worker

# Frontend
cd ../frontend
npm run build   # outputs to frontend/dist
```

---

## 6. Architecture Notes (Rationale)

- **Backend layering:** HTTP handlers (`internal/httpserver`) and workers (`internal/worker`) both depend on the shared `pkg/service` package. Services receive DTOs from `pkg/dto`, apply business rules, and talk to Postgres via GORM. This keeps the API and the worker consistent without duplicating logic.
- **`pkg/` vs `internal/`:** `pkg/` is intentionally the reusable library (models, DTOs, services, db, auth) so both `cmd/api` and `cmd/worker` import it. `internal/` holds entrypoint-specific wiring.
- **Fiber v3:** Uses the beta v3 module path `github.com/gofiber/fiber/v3` and its `Bind().Body(...)` API.
- **Frontend:** RTK Query's base API injects `Authorization: Bearer <token>` from the auth slice. `ProtectedRoute` guards the dashboard. MUI dark theme is applied at the root with `CssBaseline`.
