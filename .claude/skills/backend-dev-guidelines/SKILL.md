---
name: backend-dev-guidelines
description: Comprehensive backend development guide for Node.js/Express/TypeScript microservices. Use when creating routes, controllers, services, repositories, middleware, or working with Express APIs, Prisma database access, Sentry error tracking, Zod validation, unifiedConfig, dependency injection, or async patterns. Covers layered architecture (routes → controllers → services → repositories), BaseController pattern, error handling, performance monitoring, testing strategies, and migration from legacy patterns.
---

# Backend Development Guidelines | 後端開發指南

## 繁體中文說明 | Traditional Chinese Documentation

**技能名稱：** backend-dev-guidelines（後端開發指南）

**用途：** 為 Node.js/Express/TypeScript 微服務提供全面的開發指南

**何時自動激活：**
- 創建或修改路由、端點、API
- 構建控制器、服務、存儲庫
- 實施中間件（認證、驗證、錯誤處理）
- 使用 Prisma 進行數據庫操作
- 使用 Sentry 進行錯誤跟踪
- 使用 Zod 進行輸入驗證
- 配置管理
- 後端測試和重構

**核心架構：分層架構（Layered Architecture）**
```
HTTP 請求
    ↓
路由層 (Routes) - 僅負責路由
    ↓
控制器層 (Controllers) - 處理請求
    ↓
服務層 (Services) - 業務邏輯
    ↓
存儲庫層 (Repositories) - 數據訪問
    ↓
數據庫 (Prisma)
```

**7 個核心原則：**
1. **路由只負責路由，控制器負責控制** - 不要在路由中寫業務邏輯
2. **所有控制器繼承 BaseController** - 統一的錯誤處理和響應格式
3. **所有錯誤發送到 Sentry** - 完整的錯誤跟踪
4. **使用 unifiedConfig，永不直接使用 process.env** - 集中配置管理
5. **使用 Zod 驗證所有輸入** - 類型安全的輸入驗證
6. **使用存儲庫模式進行數據訪問** - 服務 → 存儲庫 → 數據庫
7. **需要全面的測試** - 單元測試和集成測試

**目錄結構：**
```
service/src/
├── config/              # UnifiedConfig
├── controllers/         # 請求處理器
├── services/            # 業務邏輯
├── repositories/        # 數據訪問
├── routes/              # 路由定義
├── middleware/          # Express 中間件
├── types/               # TypeScript 類型
├── validators/          # Zod 模式
├── utils/               # 工具函數
├── tests/               # 測試
├── instrument.ts        # Sentry（第一個導入）
├── app.ts               # Express 設置
└── server.ts            # HTTP 服務器
```

**資源文件（11 個）：**
1. architecture-overview.md - 架構概述
2. routing-and-controllers.md - 路由和控制器
3. services-and-repositories.md - 服務和存儲庫
4. validation-patterns.md - 驗證模式
5. sentry-and-monitoring.md - Sentry 和監控
6. middleware-guide.md - 中間件指南
7. database-patterns.md - 數據庫模式
8. configuration.md - 配置管理
9. async-and-errors.md - 異步和錯誤處理
10. testing-guide.md - 測試指南
11. complete-examples.md - 完整示例

**要避免的反模式：**
❌ 在路由中寫業務邏輯
❌ 直接使用 process.env
❌ 缺少錯誤處理
❌ 沒有輸入驗證
❌ 到處直接使用 Prisma
❌ 使用 console.log 而不是 Sentry

---

## Purpose

Establish consistency and best practices across backend microservices (blog-api, auth-service, notifications-service) using modern Node.js/Express/TypeScript patterns.

## When to Use This Skill

Automatically activates when working on:
- Creating or modifying routes, endpoints, APIs
- Building controllers, services, repositories
- Implementing middleware (auth, validation, error handling)
- Database operations with Prisma
- Error tracking with Sentry
- Input validation with Zod
- Configuration management
- Backend testing and refactoring

---

## Quick Start

### New Backend Feature Checklist

- [ ] **Route**: Clean definition, delegate to controller
- [ ] **Controller**: Extend BaseController
- [ ] **Service**: Business logic with DI
- [ ] **Repository**: Database access (if complex)
- [ ] **Validation**: Zod schema
- [ ] **Sentry**: Error tracking
- [ ] **Tests**: Unit + integration tests
- [ ] **Config**: Use unifiedConfig

### New Microservice Checklist

- [ ] Directory structure (see [architecture-overview.md](architecture-overview.md))
- [ ] instrument.ts for Sentry
- [ ] unifiedConfig setup
- [ ] BaseController class
- [ ] Middleware stack
- [ ] Error boundary
- [ ] Testing framework

---

## Architecture Overview

### Layered Architecture

```
HTTP Request
    ↓
Routes (routing only)
    ↓
Controllers (request handling)
    ↓
Services (business logic)
    ↓
Repositories (data access)
    ↓
Database (Prisma)
```

**Key Principle:** Each layer has ONE responsibility.

See [architecture-overview.md](architecture-overview.md) for complete details.

---

## Directory Structure

```
service/src/
├── config/              # UnifiedConfig
├── controllers/         # Request handlers
├── services/            # Business logic
├── repositories/        # Data access
├── routes/              # Route definitions
├── middleware/          # Express middleware
├── types/               # TypeScript types
├── validators/          # Zod schemas
├── utils/               # Utilities
├── tests/               # Tests
├── instrument.ts        # Sentry (FIRST IMPORT)
├── app.ts               # Express setup
└── server.ts            # HTTP server
```

**Naming Conventions:**
- Controllers: `PascalCase` - `UserController.ts`
- Services: `camelCase` - `userService.ts`
- Routes: `camelCase + Routes` - `userRoutes.ts`
- Repositories: `PascalCase + Repository` - `UserRepository.ts`

---

## Core Principles (7 Key Rules)

### 1. Routes Only Route, Controllers Control

```typescript
// ❌ NEVER: Business logic in routes
router.post('/submit', async (req, res) => {
    // 200 lines of logic
});

// ✅ ALWAYS: Delegate to controller
router.post('/submit', (req, res) => controller.submit(req, res));
```

### 2. All Controllers Extend BaseController

```typescript
export class UserController extends BaseController {
    async getUser(req: Request, res: Response): Promise<void> {
        try {
            const user = await this.userService.findById(req.params.id);
            this.handleSuccess(res, user);
        } catch (error) {
            this.handleError(error, res, 'getUser');
        }
    }
}
```

### 3. All Errors to Sentry

```typescript
try {
    await operation();
} catch (error) {
    Sentry.captureException(error);
    throw error;
}
```

### 4. Use unifiedConfig, NEVER process.env

```typescript
// ❌ NEVER
const timeout = process.env.TIMEOUT_MS;

// ✅ ALWAYS
import { config } from './config/unifiedConfig';
const timeout = config.timeouts.default;
```

### 5. Validate All Input with Zod

```typescript
const schema = z.object({ email: z.string().email() });
const validated = schema.parse(req.body);
```

### 6. Use Repository Pattern for Data Access

```typescript
// Service → Repository → Database
const users = await userRepository.findActive();
```

### 7. Comprehensive Testing Required

```typescript
describe('UserService', () => {
    it('should create user', async () => {
        expect(user).toBeDefined();
    });
});
```

---

## Common Imports

```typescript
// Express
import express, { Request, Response, NextFunction, Router } from 'express';

// Validation
import { z } from 'zod';

// Database
import { PrismaClient } from '@prisma/client';
import type { Prisma } from '@prisma/client';

// Sentry
import * as Sentry from '@sentry/node';

// Config
import { config } from './config/unifiedConfig';

// Middleware
import { SSOMiddlewareClient } from './middleware/SSOMiddleware';
import { asyncErrorWrapper } from './middleware/errorBoundary';
```

---

## Quick Reference

### HTTP Status Codes

| Code | Use Case |
|------|----------|
| 200 | Success |
| 201 | Created |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 500 | Server Error |

### Service Templates

**Blog API** (✅ Mature) - Use as template for REST APIs
**Auth Service** (✅ Mature) - Use as template for authentication patterns

---

## Anti-Patterns to Avoid

❌ Business logic in routes
❌ Direct process.env usage
❌ Missing error handling
❌ No input validation
❌ Direct Prisma everywhere
❌ console.log instead of Sentry

---

## Navigation Guide

| Need to... | Read this |
|------------|-----------|
| Understand architecture | [architecture-overview.md](architecture-overview.md) |
| Create routes/controllers | [routing-and-controllers.md](routing-and-controllers.md) |
| Organize business logic | [services-and-repositories.md](services-and-repositories.md) |
| Validate input | [validation-patterns.md](validation-patterns.md) |
| Add error tracking | [sentry-and-monitoring.md](sentry-and-monitoring.md) |
| Create middleware | [middleware-guide.md](middleware-guide.md) |
| Database access | [database-patterns.md](database-patterns.md) |
| Manage config | [configuration.md](configuration.md) |
| Handle async/errors | [async-and-errors.md](async-and-errors.md) |
| Write tests | [testing-guide.md](testing-guide.md) |
| See examples | [complete-examples.md](complete-examples.md) |

---

## Resource Files

### [architecture-overview.md](architecture-overview.md)
Layered architecture, request lifecycle, separation of concerns

### [routing-and-controllers.md](routing-and-controllers.md)
Route definitions, BaseController, error handling, examples

### [services-and-repositories.md](services-and-repositories.md)
Service patterns, DI, repository pattern, caching

### [validation-patterns.md](validation-patterns.md)
Zod schemas, validation, DTO pattern

### [sentry-and-monitoring.md](sentry-and-monitoring.md)
Sentry init, error capture, performance monitoring

### [middleware-guide.md](middleware-guide.md)
Auth, audit, error boundaries, AsyncLocalStorage

### [database-patterns.md](database-patterns.md)
PrismaService, repositories, transactions, optimization

### [configuration.md](configuration.md)
UnifiedConfig, environment configs, secrets

### [async-and-errors.md](async-and-errors.md)
Async patterns, custom errors, asyncErrorWrapper

### [testing-guide.md](testing-guide.md)
Unit/integration tests, mocking, coverage

### [complete-examples.md](complete-examples.md)
Full examples, refactoring guide

---

## Related Skills

- **database-verification** - Verify column names and schema consistency
- **error-tracking** - Sentry integration patterns
- **skill-developer** - Meta-skill for creating and managing skills

---

**Skill Status**: COMPLETE ✅
**Line Count**: < 500 ✅
**Progressive Disclosure**: 11 resource files ✅
