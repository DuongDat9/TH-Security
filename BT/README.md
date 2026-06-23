# Bài tập thực hành SECURITY — NestJS

Project NestJS minh họa 4 yêu cầu của bài tập SECURITY: **Cookies**, **Session**,
**API đăng ký username/password có mã hóa (hashing)**, và **Authentication/Authorization bằng JWT**.

> Tham khảo chính thức: [docs.nestjs.com — Cookies](https://docs.nestjs.com/techniques/cookies),
> [Session](https://docs.nestjs.com/techniques/session),
> [Encryption and Hashing](https://docs.nestjs.com/security/encryption-and-hashing),
> [Authentication](https://docs.nestjs.com/security/authentication)

---

## 1. Công nghệ sử dụng

| Thành phần        | Package                                            |
|--------------------|-----------------------------------------------------|
| Framework          | NestJS 11                                          |
| Cookies            | `cookie-parser`                                    |
| Session            | `express-session`                                  |
| Hashing password   | `bcrypt`                                           |
| ORM / CSDL         | `@nestjs/typeorm` + `typeorm` (SQLite)             |
| JWT Auth           | `@nestjs/jwt`, `@nestjs/passport`, `passport-jwt`  |
| Validate DTO       | `class-validator`, `class-transformer`             |

> **Lưu ý về driver CSDL:** project này dùng driver TypeORM `sqljs` (SQLite chạy bằng
> WebAssembly, thuần JavaScript) để tránh phải build native module trong môi trường
> phát triển hạn chế mạng. Khi chạy ở máy cá nhân / GitHub Codespaces có internet đầy
> đủ, bạn nên đổi `type: 'sqljs'` thành `type: 'better-sqlite3'` (hoặc `postgres`,
> `mysql`,...) trong `src/app.module.ts` để có hiệu năng tốt hơn cho thực tế sản xuất.
> Cách dùng và API (Entity, Repository, hashing, JWT...) hoàn toàn không đổi.

---

## 2. Cấu trúc project

```
src/
├── app.module.ts            # Khai báo TypeOrmModule.forRoot(), import Users/Auth module
├── main.ts                  # Bootstrap app, apply cookie-parser & express-session middleware
├── users/
│   ├── entities/user.entity.ts   # Entity quản lý username & password (đã hash)
│   ├── dto/register.dto.ts       # DTO validate dữ liệu đăng ký
│   ├── users.service.ts          # Hash password bằng bcrypt, lưu CSDL
│   └── users.module.ts
└── auth/
    ├── constants.ts              # JWT secret (đọc từ biến môi trường)
    ├── dto/login.dto.ts
    ├── auth.service.ts           # Logic register/login, sinh JWT
    ├── auth.controller.ts        # Route: register, login, profile, cookie-demo, session-demo
    ├── auth.module.ts            # Khai báo JwtModule, PassportModule
    ├── strategies/jwt.strategy.ts# Passport JWT Strategy
    ├── guards/jwt-auth.guard.ts  # Guard bảo vệ route bằng JWT
    └── decorators/current-user.decorator.ts
```

---

## 3. Mô tả từng yêu cầu

### ✅ Yêu cầu 1 — Cookies
- Cài đặt: `npm i cookie-parser` + `npm i -D @types/cookie-parser`
- Áp dụng middleware `app.use(cookieParser())` trong `main.ts`
- Route demo:
  - `GET /auth/cookie-demo` — set cookie `demo_cookie` (httpOnly, maxAge 1 ngày)
  - `GET /auth/read-cookie` — đọc lại cookie từ `request.cookies`

### ✅ Yêu cầu 2 — Session
- Cài đặt: `npm i express-session` + `npm i -D @types/express-session`
- Áp dụng middleware `app.use(session({ secret, resave: false, saveUninitialized: false }))`
- Route demo:
  - `GET /auth/session-demo` — dùng `@Session()` decorator, đếm số lượt truy cập
    (`session.visits`) tăng dần qua mỗi request của **cùng một client** (cùng cookie `connect.sid`)

### ✅ Yêu cầu 3 — POST API đăng ký username/password (đã hash)
- **Entity** `User` (`src/users/entities/user.entity.ts`) quản lý `username`, `password`, `createdAt`
- `POST /auth/register` nhận `{ username, password }` từ body
- `UsersService.register()`:
  1. Kiểm tra username đã tồn tại chưa (`409 Conflict` nếu trùng)
  2. **Mã hóa password bằng `bcrypt.hash(password, 10)`** trước khi lưu
  3. Lưu username + password đã hash vào CSDL qua TypeORM Repository
  4. Trả về thông tin user **không bao gồm password**

### ✅ Yêu cầu 4 — Authentication/Authorization bằng JWT
- Cài đặt: `@nestjs/jwt`, `@nestjs/passport`, `passport`, `passport-jwt`
- `POST /auth/login` — nhận `{ username, password }`, so khớp bằng `bcrypt.compare()`,
  nếu đúng thì sinh `access_token` (JWT, HS256) chứa payload `{ sub, username }`
- `JwtStrategy` (Passport) verify token gửi qua header `Authorization: Bearer <token>`
- `JwtAuthGuard` bảo vệ route — áp dụng cho `GET /auth/profile` (route chỉ truy cập được
  khi có JWT hợp lệ)

---

## 4. Cài đặt & chạy project

```bash
# 1. Cài dependencies
npm install

# 2. Tạo file .env (tham khảo .env.example) - đổi JWT_SECRET, SESSION_SECRET thành giá trị riêng
cp .env.example .env

# 3. Chạy ở chế độ development (watch mode)
npm run start:dev

# Server chạy tại http://localhost:3000
```

> Khi khởi động lần đầu, TypeORM (`synchronize: true`) sẽ tự tạo bảng `users` trong
> file `database.sqlite` tại thư mục gốc project.

---

## 5. Danh sách API & cách test (cURL)

### Đăng ký (Yêu cầu 3)
```bash
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "sinhvien01", "password": "MatKhau@123"}'
```

### Đăng nhập, nhận JWT (Yêu cầu 4)
```bash
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "sinhvien01", "password": "MatKhau@123"}'
```

### Truy cập route được bảo vệ bằng JWT
```bash
curl http://localhost:3000/auth/profile \
  -H "Authorization: Bearer <access_token_nhận_được_ở_bước_login>"
```

### Cookies (Yêu cầu 1)
```bash
curl -c cookies.txt http://localhost:3000/auth/cookie-demo
curl -b cookies.txt http://localhost:3000/auth/read-cookie
```

### Session (Yêu cầu 2)
```bash
curl -c session.txt -b session.txt http://localhost:3000/auth/session-demo
curl -c session.txt -b session.txt http://localhost:3000/auth/session-demo
# visits sẽ tăng dần: 1, 2, 3, ...
```

---

## 6. Ảnh chụp màn hình kết quả

Toàn bộ ảnh kết quả test API và dữ liệu CSDL nằm trong thư mục `screenshots/`:

| File | Nội dung |
|------|----------|
| `01-register-success.png` | Đăng ký user mới thành công (201) |
| `02-register-duplicate.png` | Đăng ký trùng username (409) |
| `03-login-wrong-password.png` | Đăng nhập sai password (401) |
| `04-login-success-jwt.png` | Đăng nhập đúng, nhận JWT access_token (200) |
| `05-profile-no-token.png` | Truy cập `/profile` không có token (401) |
| `06-profile-with-token.png` | Truy cập `/profile` có token hợp lệ (200) |
| `07-cookie-set.png` | Set cookie thành công |
| `08-cookie-read.png` | Đọc lại cookie đã set |
| `09-session-demo-1/2/3.png` | Session `visits` tăng dần qua 3 lần gọi |
| `10-database-data.png` | Dữ liệu thật trong `database.sqlite` (password đã hash bcrypt) |

---

## 7. Lưu ý an toàn khi triển khai thực tế

- **Không hardcode JWT secret / session secret trong source code** — luôn đọc từ biến
  môi trường (`.env`) và không commit file `.env` lên Git (đã có trong `.gitignore`).
- Thời gian sống JWT (`expiresIn`) trong demo đặt 1 giờ; cần cân nhắc thêm refresh token
  cho ứng dụng thực tế.
- `synchronize: true` của TypeORM chỉ nên dùng cho môi trường dev/demo, **không dùng cho
  production** (nên dùng migration).
- Driver `sqljs` chỉ phù hợp cho mục đích demo/học tập; production nên dùng
  PostgreSQL/MySQL hoặc tối thiểu là `better-sqlite3`.
