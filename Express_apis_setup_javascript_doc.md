# Production Express.js CRUD API Blueprint
> **JavaScript ES6 + Prisma 6 + MySQL + JWT + Zod**

*Flawless Step-by-Step Production Blueprint: Routes, Validations, Controllers, Services & Middlewares*

---

## 1. Architectural Overview (The Layered Pattern)

In professional software development, every layer in the backend has a single, strictly separated responsibility:

| Layer | Directory | Responsibility & Rules |
| :--- | :--- | :--- |
| **Routes** | `src/routes/` | Defines HTTP endpoints and binds validation & authentication middlewares. Contains zero business logic. |
| **Validation** | `src/schemas/` & `middlewares/validate.js` | Validates incoming request bodies with Zod (email format, password length, required fields) before the controller is ever called. |
| **Middlewares** | `src/middlewares/` | Intercepts requests: verifies JWT tokens, logs activity, and centralizes error handling. |
| **Controllers** | `src/controllers/` | Handles HTTP `req` and `res`. Reads params/body, calls the Service, and returns standardized JSON responses. |
| **Services** | `src/services/` | Pure business logic. Pure JavaScript classes. Talks to Prisma ORM. Completely HTTP-agnostic (no `req` or `res` objects). |
| **Database** | `src/config/` & `prisma/` | MySQL connection pool and Prisma ORM data modeling schema. |

---

## 2. Complete Project Directory Layout

This is the exact folder structure. When creating a new project, match this directory layout:

```text
my-express-api/
├── prisma/
│   ├── migrations/          # Auto-generated SQL migration history
│   └── schema.prisma        # MySQL connection and Prisma model definitions
├── src/
│   ├── config/
│   │   └── db.js            # PrismaClient singleton instance
│   ├── schemas/             # Zod validation schemas
│   │   ├── authSchema.js    # Validation rules for register & login
│   │   └── userSchema.js    # Validation rules for user update
│   ├── middlewares/
│   │   ├── validate.js      # Reusable Zod schema validation middleware
│   │   ├── authMiddleware.js# Verifies Bearer JWT tokens from headers
│   │   └── errorHandler.js  # Centralized global error handling
│   ├── services/
│   │   ├── authService.js   # User registration, bcrypt hashing, JWT signing
│   │   └── userService.js   # User database queries and business logic
│   ├── controllers/
│   │   ├── authController.js# HTTP handler for register, login, logout
│   │   └── userController.js# HTTP handler for user CRUD actions
│   ├── routes/
│   │   ├── authRoutes.js    # Routes for /api/auth with validation
│   │   ├── userRoutes.js    # Protected routes for /api/users
│   │   └── index.js         # Master router combining all feature routers
│   ├── utils/
│   │   └── apiError.js      # Custom error class with HTTP status codes
│   ├── app.js               # Express application configuration & middlewares
│   └── server.js            # Server entry point (app.listen on PORT)
├── .env                     # Database connection string and JWT secret keys
├── .gitignore               # Ignores node_modules, .env
└── package.json             # Dependencies and "type": "module"
```

---

## 3. Step-by-Step Implementation & Copy-Paste Code

### Step 1: Project Initialization & Package Dependencies

Create the project folder, generate `package.json`, and install dependencies. Notice that `@prisma/client@6` and `prisma@6` are explicitly pinned to version 6 to prevent version mismatches:

```bash
# 1. Create project folder & initialize npm
mkdir my-express-api
cd my-express-api
npm init -y

# 2. Create the source directories in one line (PowerShell / Windows)
mkdir src/config, src/schemas, src/middlewares, src/services, src/controllers, src/routes, src/utils

# (If using Git Bash / Linux / Mac, use this instead:)
# mkdir -p src/{config,schemas,middlewares,services,controllers,routes,utils}

# 3. Install production dependencies (Note: @prisma/client@6 matches prisma@6)
npm install express dotenv @prisma/client@6 bcryptjs jsonwebtoken zod

# 4. Install developer dependencies
npm install -D nodemon prisma@6
```

**File:** `package.json` *(Ensure `"type": "module"` is present for ES6 import/export syntax)*:

```json
{
  "name": "my-express-api",
  "version": "1.0.0",
  "type": "module",
  "main": "src/server.js",
  "scripts": {
    "dev": "nodemon src/server.js",
    "start": "node src/server.js"
  },
  "dependencies": {
    "@prisma/client": "^6.19.3",
    "bcryptjs": "^3.0.3",
    "dotenv": "^17.4.2",
    "express": "^5.2.1",
    "jsonwebtoken": "^9.0.3",
    "zod": "^3.24.2"
  },
  "devDependencies": {
    "nodemon": "^3.1.14",
    "prisma": "^6.19.3"
  }
}
```

---

### Step 2: Generate Prisma Folder & .env Automatically

> [!TIP]
> **KEY COMMAND:** Run this command to automatically generate the `prisma/` directory, `schema.prisma`, and `.env` file configured for MySQL:

```bash
npx prisma init --datasource-provider mysql
```

This command automatically creates two essential files:
1. `prisma/schema.prisma` (pre-configured with MySQL datasource)
2. `.env` (pre-configured with a MySQL `DATABASE_URL`)

> [!WARNING]
> **IMPORTANT:** If a file named `prisma.config.ts` is ever created by an experimental CLI, delete it. In Prisma 6, configuration lives entirely inside `prisma/schema.prisma`.

Configure your actual MySQL credentials and JWT secrets in `.env`:

```env
PORT=3000
DATABASE_URL="mysql://root:YOUR_PASSWORD@localhost:3306/express_auth_db"
JWT_SECRET="your_super_secret_jwt_key_987654321"
JWT_EXPIRES_IN="1d"
```

**File:** `.gitignore`:

```gitignore
node_modules
.env
.DS_Store
```

---

### Step 3: Database Schema & Prisma Client Singleton

**File:** `prisma/schema.prisma` *(Define the MySQL User model)*:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  password  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("users")
}
```

Run the migration command in terminal to create the database and tables:

```bash
npx prisma migrate dev --name init
```

**File:** `src/config/db.js` *(Prisma Client Singleton Connection)*:

```javascript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  log: ['warn', 'error'],
});

export default prisma;
```

---

### Step 4: Error Handling & Utility Classes

**File:** `src/utils/apiError.js` *(Custom Error Class with HTTP Status)*:

```javascript
export class ApiError extends Error {
  constructor(statusCode, message) {
    super(message);
    this.statusCode = statusCode;
  }
}
```

**File:** `src/middlewares/errorHandler.js` *(Global Error Handling Middleware)*:

```javascript
export const errorHandler = (err, req, res, next) => {
  const statusCode = err.statusCode || 500;
  const message = err.message || 'Internal Server Error';

  console.error(`[Error] ${statusCode} - ${message}`);

  res.status(statusCode).json({
    success: false,
    status: statusCode,
    message: message,
    stack: process.env.NODE_ENV === 'development' ? err.stack : undefined,
  });
};
```

---

### Step 5: Request Validation Layer (Zod Schemas & Middleware)

**File:** `src/middlewares/validate.js` *(Reusable Middleware to Validate Any Zod Schema)*:

```javascript
export const validate = (schema) => (req, res, next) => {
  // safeParse validates req.body without throwing unhandled exceptions
  const result = schema.safeParse(req.body);

  if (!result.success) {
    const issues = result.error.issues || result.error.errors || [];
    const formattedErrors = issues.map((err) => ({
      field: err.path.join('.'),
      message: err.message,
    }));

    return res.status(400).json({
      success: false,
      message: 'Validation failed',
      errors: formattedErrors,
    });
  }

  req.validated = result.data;
  next(); // Data is valid, proceed to controller!
};
```

**File:** `src/schemas/authSchema.js` *(Validation Rules for Register and Login)*:

```javascript
import { z } from 'zod';

export const registerSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Please provide a valid email address'),
  password: z
    .string()
    .min(6, 'Password must be at least 6 characters')
    .max(50, 'Password is too long'),
});

export const loginSchema = z.object({
  email: z.string().email('Please provide a valid email address'),
  password: z.string().min(1, 'Password is required'),
});
```

**File:** `src/schemas/userSchema.js` *(Validation Rules for User Profile Updates)*:

```javascript
import { z } from 'zod';

export const updateUserSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters').optional(),
  email: z.string().email('Please provide a valid email address').optional(),
});
```

---

### Step 6: Authentication Service Layer

**File:** `src/services/authService.js` *(Password Hashing, Verification & JWT Creation)*:

```javascript
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import prisma from '../config/db.js';
import { ApiError } from '../utils/apiError.js';

class AuthService {
  generateToken(userId) {
    return jwt.sign(
      { id: userId },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRES_IN || '1d' }
    );
  }

  async register({ name, email, password }) {
    const existingUser = await prisma.user.findUnique({ where: { email } });
    if (existingUser) {
      throw new ApiError(409, 'A user with this email already exists');
    }

    const hashedPassword = await bcrypt.hash(password, 10);

    const user = await prisma.user.create({
      data: { name, email, password: hashedPassword },
      select: { id: true, name: true, email: true, createdAt: true },
    });

    const token = this.generateToken(user.id);
    return { user, token };
  }

  async login({ email, password }) {
    const user = await prisma.user.findUnique({ where: { email } });
    if (!user) {
      throw new ApiError(401, 'Invalid email or password');
    }

    const isPasswordValid = await bcrypt.compare(password, user.password);
    if (!isPasswordValid) {
      throw new ApiError(401, 'Invalid email or password');
    }

    const token = this.generateToken(user.id);
    return {
      user: { id: user.id, name: user.name, email: user.email },
      token,
    };
  }

  async logout() {
    return { message: 'Logged out successfully' };
  }
}

export default new AuthService();
```

---

### Step 7: Auth Controller & Routes (With Validation Attached)

**File:** `src/controllers/authController.js`:

```javascript
import authService from '../services/authService.js';

class AuthController {
  async register(req, res, next) {
    try {
      const result = await authService.register(req.validated);
      res.status(201).json({ success: true, message: 'User registered successfully', data: result });
    } catch (error) {
      next(error);
    }
  }

  async login(req, res, next) {
    try {
      const result = await authService.login(req.validated);
      res.status(200).json({ success: true, message: 'Login successful', data: result });
    } catch (error) {
      next(error);
    }
  }

  async logout(req, res, next) {
    try {
      const result = await authService.logout();
      res.status(200).json({ success: true, message: result.message });
    } catch (error) {
      next(error);
    }
  }
}

export default new AuthController();
```

**File:** `src/routes/authRoutes.js` *(Validates incoming input with Zod middleware)*:

```javascript
import { Router } from 'express';
import authController from '../controllers/authController.js';
import { validate } from '../middlewares/validate.js';
import { registerSchema, loginSchema } from '../schemas/authSchema.js';

const router = Router();

router.post('/register', validate(registerSchema), authController.register);
router.post('/login', validate(loginSchema), authController.login);
router.post('/logout', authController.logout);

export default router;
```

---

### Step 8: Protecting Routes with Auth Middleware (JWT Guard)

**File:** `src/middlewares/authMiddleware.js`:

```javascript
import jwt from 'jsonwebtoken';
import prisma from '../config/db.js';
import { ApiError } from '../utils/apiError.js';

export const authenticate = async (req, res, next) => {
  try {
    const authHeader = req.headers.authorization;
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      throw new ApiError(401, 'Unauthorized: Access token is missing or malformed');
    }

    const token = authHeader.split(' ')[1];
    let decoded;
    try {
      decoded = jwt.verify(token, process.env.JWT_SECRET);
    } catch (err) {
      if (err.name === 'TokenExpiredError') {
        throw new ApiError(401, 'Unauthorized: Token expired. Please login again');
      }
      throw new ApiError(401, 'Unauthorized: Invalid token');
    }

    const user = await prisma.user.findUnique({
      where: { id: decoded.id },
      select: { id: true, name: true, email: true },
    });

    if (!user) {
      throw new ApiError(401, 'Unauthorized: User no longer exists');
    }

    req.user = user; // Attach user to request
    next();
  } catch (error) {
    next(error);
  }
};
```

---

### Step 9: Full User CRUD (Service, Controller & Protected Routes)

**File:** `src/services/userService.js`:

```javascript
import prisma from '../config/db.js';
import { ApiError } from '../utils/apiError.js';

class UserService {
  async getAllUsers() {
    return await prisma.user.findMany({
      select: { id: true, name: true, email: true, createdAt: true, updatedAt: true },
      orderBy: { createdAt: 'desc' },
    });
  }

  async getUserById(id) {
    const numericId = parseInt(id, 10);
    if (isNaN(numericId)) throw new ApiError(400, 'Invalid user ID format');

    const user = await prisma.user.findUnique({
      where: { id: numericId },
      select: { id: true, name: true, email: true, createdAt: true, updatedAt: true },
    });

    if (!user) throw new ApiError(404, `User with ID ${numericId} not found`);
    return user;
  }

  async updateUser(id, updateData) {
    const numericId = parseInt(id, 10);
    if (isNaN(numericId)) throw new ApiError(400, 'Invalid user ID format');

    const existingUser = await prisma.user.findUnique({ where: { id: numericId } });
    if (!existingUser) throw new ApiError(404, `User with ID ${numericId} not found`);

    if (updateData.email && updateData.email !== existingUser.email) {
      const emailTaken = await prisma.user.findUnique({ where: { email: updateData.email } });
      if (emailTaken) throw new ApiError(409, 'Email is already in use by another account');
    }

    return await prisma.user.update({
      where: { id: numericId },
      data: {
        name: updateData.name ?? existingUser.name,
        email: updateData.email ?? existingUser.email,
      },
      select: { id: true, name: true, email: true, updatedAt: true },
    });
  }

  async deleteUser(id) {
    const numericId = parseInt(id, 10);
    if (isNaN(numericId)) throw new ApiError(400, 'Invalid user ID format');

    const existingUser = await prisma.user.findUnique({ where: { id: numericId } });
    if (!existingUser) throw new ApiError(404, `User with ID ${numericId} not found`);

    await prisma.user.delete({ where: { id: numericId } });
    return { message: `User with ID ${numericId} deleted successfully` };
  }

  async getProfile(userId) {
    return await this.getUserById(userId);
  }
}

export default new UserService();
```

**File:** `src/controllers/userController.js`:

```javascript
import userService from '../services/userService.js';

class UserController {
  async getProfile(req, res, next) {
    res.status(200).json({ success: true, data: req.user });
  }

  async list(req, res, next) {
    try {
      const users = await userService.getAllUsers();
      res.status(200).json({ success: true, data: users });
    } catch (error) { next(error); }
  }

  async getById(req, res, next) {
    try {
      const user = await userService.getUserById(req.params.id);
      res.status(200).json({ success: true, data: user });
    } catch (error) { next(error); }
  }

  async update(req, res, next) {
    try {
      const updated = await userService.updateUser(req.params.id, req.validated);
      res.status(200).json({ success: true, message: 'Updated successfully', data: updated });
    } catch (error) { next(error); }
  }

  async remove(req, res, next) {
    try {
      const result = await userService.deleteUser(req.params.id);
      res.status(200).json({ success: true, message: result.message });
    } catch (error) { next(error); }
  }
}

export default new UserController();
```

**File:** `src/routes/userRoutes.js` *(Notice: `/me` is placed BEFORE `/:id` so Express does not treat `'me'` as an ID)*:

```javascript
import { Router } from 'express';
import userController from '../controllers/userController.js';
import { authenticate } from '../middlewares/authMiddleware.js';
import { validate } from '../middlewares/validate.js';
import { updateUserSchema } from '../schemas/userSchema.js';

const router = Router();

// All routes below require valid JWT token in Authorization: Bearer <token>
router.use(authenticate);

// Important: Define /me BEFORE /:id to prevent route collision
router.get('/me', userController.getProfile);
router.get('/', userController.list);
router.get('/:id', userController.getById);
router.put('/:id', validate(updateUserSchema), userController.update);
router.delete('/:id', userController.remove);

export default router;
```

---

### Step 10: Master Router, Express App & Server Listener

**File:** `src/routes/index.js` *(Central Master Router)*:

```javascript
import { Router } from 'express';
import authRoutes from './authRoutes.js';
import userRoutes from './userRoutes.js';

const router = Router();

router.use('/auth', authRoutes);
router.use('/users', userRoutes);

export default router;
```

**File:** `src/app.js` *(Express Config, Central Router, 404 & Error Handler - Compatible with Express 4 & 5)*:

```javascript
import express from 'express';
import apiRouter from './routes/index.js';
import { errorHandler } from './middlewares/errorHandler.js';
import { ApiError } from './utils/apiError.js';

const app = express();

app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Health check
app.get('/api/health', (req, res) => {
  res.status(200).json({ status: 'ok', message: 'API is healthy' });
});

// Mount all API endpoints under /api
app.use('/api', apiRouter);

// Catch-all for undefined routes (404) - Compatible with Express 4 & 5
app.use((req, res, next) => {
  next(new ApiError(404, `Cannot find ${req.method} ${req.originalUrl} on this server`));
});

// Central Error Handler (must be registered last)
app.use(errorHandler);

export default app;
```

**File:** `src/server.js` *(Starts the HTTP Server Listener)*:

```javascript
import dotenv from 'dotenv';
import app from './app.js';

dotenv.config();

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`🚀 Server running on http://localhost:${PORT}`);
});
```

---

## 4. API Testing Guide (Postman / Thunder Client / cURL)

### 1. Health Check (Public)
```http
GET http://localhost:3000/api/health
```

---

### 2. Validation Failure Test (Verify Zod Rejects Bad Input)

**Request:**
```http
POST http://localhost:3000/api/auth/register
Content-Type: application/json

{
  "name": "D",
  "email": "not-an-email",
  "password": "123"
}
```

**Expected Response (400 Bad Request):**
```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    { "field": "name", "message": "Name must be at least 2 characters" },
    { "field": "email", "message": "Please provide a valid email address" },
    { "field": "password", "message": "Password must be at least 6 characters" }
  ]
}
```

---

### 3. Register User (Valid Input)

**Request:**
```http
POST http://localhost:3000/api/auth/register
Content-Type: application/json

{
  "name": "Devang Patel",
  "email": "devang@example.com",
  "password": "Password123!"
}
```

**Response (201 Created):**
> Returns the created user object and a JWT access token.

---

### 4. Login User (Public)

**Request:**
```http
POST http://localhost:3000/api/auth/login
Content-Type: application/json

{
  "email": "devang@example.com",
  "password": "Password123!"
}
```

**Response (200 OK):**
> Returns user object and JWT token &rarr; Copy the `token` string for protected requests!

---

### 5. Logout User (Public)

**Request:**
```http
POST http://localhost:3000/api/auth/logout
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

---

### 6. Get Current User Profile (Protected)

**Request:**
```http
GET http://localhost:3000/api/users/me
Authorization: Bearer <YOUR_COPIED_TOKEN>
```

---

### 7. List All Users (Protected)

**Request:**
```http
GET http://localhost:3000/api/users
Authorization: Bearer <YOUR_COPIED_TOKEN>
```

---

### 8. Get User by ID (Protected)

**Request:**
```http
GET http://localhost:3000/api/users/1
Authorization: Bearer <YOUR_COPIED_TOKEN>
```

---

### 9. Update User by ID (Protected & Validated)

**Request:**
```http
PUT http://localhost:3000/api/users/1
Authorization: Bearer <YOUR_COPIED_TOKEN>
Content-Type: application/json

{
  "name": "Devang P.",
  "email": "devang_new@example.com"
}
```

---

### 10. Delete User by ID (Protected)

**Request:**
```http
DELETE http://localhost:3000/api/users/1
Authorization: Bearer <YOUR_COPIED_TOKEN>
```

---

## 5. Production Best Practices & Rules

1. **Validate at the Door:** Always validate user input with Zod middleware before the controller runs. Never let unvalidated data reach your service or database layers.
2. **ES Module Imports:** In relative imports, Node.js ES modules require the full file extension `.js` (e.g. `import db from './config/db.js'`).
3. **Prisma Version Alignment:** Ensure that `@prisma/client` and `prisma` CLI share the exact same major version (e.g. both on version 6). Mismatched versions cause WebAssembly engine loading errors.
4. **Environment Security:** Never hardcode secrets. Always store `DATABASE_URL` and `JWT_SECRET` in `.env` and verify that `.env` is in your `.gitignore`.
5. **Service Purity:** Services must stay HTTP-agnostic. Never pass `req` or `res` into service functions. Only pass clean JavaScript objects/primitives.
6. **Prisma Migrations:** Whenever you change models in `prisma/schema.prisma`, always run `npx prisma migrate dev` to update MySQL and regenerate the Prisma Client.
