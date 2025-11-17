# 實戰教學：Todo List Web App 開發指南

> 使用 Claude Code Infrastructure Showcase 的最佳實踐構建一個完整的 Todo List 應用

## 目錄

1. [項目概述](#項目概述)
2. [準備工作](#準備工作)
3. [後端開發](#後端開發)
4. [前端開發](#前端開發)
5. [集成和測試](#集成和測試)
6. [使用 Claude Code 開發](#使用-claude-code-開發)
7. [完整代碼示例](#完整代碼示例)

---

## 項目概述

### 功能需求

我們將構建一個具有以下功能的 Todo List 應用：

- ✅ 創建待辦事項
- ✅ 查看待辦事項列表
- ✅ 標記完成/未完成
- ✅ 編輯待辦事項
- ✅ 刪除待辦事項
- ✅ 按狀態過濾（全部/完成/未完成）
- ✅ 用戶認證（每個用戶有自己的待辦事項）

### 技術棧

**後端：**
- Node.js + Express + TypeScript
- Prisma（PostgreSQL）
- Zod（驗證）
- Sentry（錯誤跟踪）

**前端：**
- React + TypeScript
- MUI v7（Material-UI）
- TanStack Query（數據獲取）
- TanStack Router（路由）

---

## 準備工作

### 1. 設置 Claude Code Skills

首先，確保您已經設置了必要的 skills 和 hooks：

```bash
# 複製 skills 到您的項目
cp -r .claude/skills/backend-dev-guidelines your-project/.claude/skills/
cp -r .claude/skills/frontend-dev-guidelines your-project/.claude/skills/
cp -r .claude/skills/error-tracking your-project/.claude/skills/

# 複製 hooks
cp -r .claude/hooks/* your-project/.claude/hooks/
chmod +x your-project/.claude/hooks/*.sh

# 複製 agents（可選，用於代碼審查）
cp .claude/agents/code-architecture-reviewer.md your-project/.claude/agents/
cp .claude/agents/frontend-error-fixer.md your-project/.claude/agents/
```

### 2. 創建項目結構

```bash
mkdir -p todo-app/{backend,frontend}
cd todo-app
```

### 3. 更新 skill-rules.json

在 `.claude/skills/skill-rules.json` 中配置路徑模式：

```json
{
  "skills": {
    "backend-dev-guidelines": {
      "type": "domain",
      "enforcement": "suggest",
      "priority": "high",
      "fileTriggers": {
        "pathPatterns": [
          "backend/**/*.ts"
        ]
      }
    },
    "frontend-dev-guidelines": {
      "type": "guardrail",
      "enforcement": "block",
      "priority": "high",
      "fileTriggers": {
        "pathPatterns": [
          "frontend/src/**/*.tsx",
          "frontend/src/**/*.ts"
        ]
      }
    }
  }
}
```

---

## 後端開發

### 階段 1: 項目初始化

#### 1.1 初始化後端項目

```bash
cd backend
npm init -y
npm install express prisma @prisma/client zod
npm install -D typescript @types/node @types/express ts-node nodemon
npm install @sentry/node
```

#### 1.2 配置 TypeScript

創建 `tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

#### 1.3 設置 Prisma Schema

創建 `prisma/schema.prisma`：

```prisma
// This is your Prisma schema file

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  todos     Todo[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Todo {
  id          Int      @id @default(autoincrement())
  title       String
  description String?
  completed   Boolean  @default(false)
  userId      Int
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@index([userId])
  @@index([completed])
}
```

生成 Prisma Client：

```bash
npx prisma generate
npx prisma migrate dev --name init
```

### 階段 2: 後端架構實現

遵循 **backend-dev-guidelines** 技能中的分層架構。

#### 2.1 創建目錄結構

```bash
mkdir -p src/{config,controllers,services,repositories,routes,middleware,types,validators,utils,tests}
```

#### 2.2 配置管理 (config/unifiedConfig.ts)

```typescript
import { z } from 'zod';

const configSchema = z.object({
  port: z.number().default(3000),
  nodeEnv: z.enum(['development', 'production', 'test']).default('development'),
  database: z.object({
    url: z.string(),
  }),
  sentry: z.object({
    dsn: z.string().optional(),
    environment: z.string(),
  }),
});

export type Config = z.infer<typeof configSchema>;

function loadConfig(): Config {
  return configSchema.parse({
    port: parseInt(process.env.PORT || '3000', 10),
    nodeEnv: process.env.NODE_ENV || 'development',
    database: {
      url: process.env.DATABASE_URL || 'postgresql://localhost:5432/todoapp',
    },
    sentry: {
      dsn: process.env.SENTRY_DSN,
      environment: process.env.NODE_ENV || 'development',
    },
  });
}

export const config = loadConfig();
```

#### 2.3 Sentry 初始化 (instrument.ts)

⚠️ **重要：** 這必須是第一個導入！

```typescript
import * as Sentry from '@sentry/node';
import { config } from './config/unifiedConfig';

if (config.sentry.dsn) {
  Sentry.init({
    dsn: config.sentry.dsn,
    environment: config.sentry.environment,
    tracesSampleRate: 1.0,
  });
}

export { Sentry };
```

#### 2.4 Prisma Service (services/prismaService.ts)

```typescript
import { PrismaClient } from '@prisma/client';
import { Sentry } from '../instrument';

class PrismaService {
  private static instance: PrismaService;
  public client: PrismaClient;

  private constructor() {
    this.client = new PrismaClient({
      log: ['query', 'error', 'warn'],
    });

    // Sentry 集成
    this.client.$use(async (params, next) => {
      const start = Date.now();
      try {
        return await next(params);
      } catch (error) {
        Sentry.captureException(error, {
          contexts: {
            prisma: {
              model: params.model,
              action: params.action,
            },
          },
        });
        throw error;
      } finally {
        const duration = Date.now() - start;
        console.log(`Query ${params.model}.${params.action} took ${duration}ms`);
      }
    });
  }

  static getInstance(): PrismaService {
    if (!PrismaService.instance) {
      PrismaService.instance = new PrismaService();
    }
    return PrismaService.instance;
  }

  async disconnect(): Promise<void> {
    await this.client.$disconnect();
  }
}

export const prismaService = PrismaService.getInstance();
export const prisma = prismaService.client;
```

#### 2.5 Repository 層 (repositories/TodoRepository.ts)

```typescript
import { Prisma, Todo } from '@prisma/client';
import { prisma } from '../services/prismaService';
import { Sentry } from '../instrument';

export class TodoRepository {
  async findAll(userId: number, completed?: boolean): Promise<Todo[]> {
    try {
      const where: Prisma.TodoWhereInput = { userId };
      if (completed !== undefined) {
        where.completed = completed;
      }

      return await prisma.todo.findMany({
        where,
        orderBy: { createdAt: 'desc' },
      });
    } catch (error) {
      Sentry.captureException(error);
      throw error;
    }
  }

  async findById(id: number, userId: number): Promise<Todo | null> {
    try {
      return await prisma.todo.findFirst({
        where: { id, userId },
      });
    } catch (error) {
      Sentry.captureException(error);
      throw error;
    }
  }

  async create(data: Prisma.TodoCreateInput): Promise<Todo> {
    try {
      return await prisma.todo.create({ data });
    } catch (error) {
      Sentry.captureException(error);
      throw error;
    }
  }

  async update(id: number, userId: number, data: Prisma.TodoUpdateInput): Promise<Todo> {
    try {
      return await prisma.todo.update({
        where: { id },
        data: { ...data, userId },
      });
    } catch (error) {
      Sentry.captureException(error);
      throw error;
    }
  }

  async delete(id: number, userId: number): Promise<void> {
    try {
      await prisma.todo.delete({
        where: { id, userId },
      });
    } catch (error) {
      Sentry.captureException(error);
      throw error;
    }
  }
}
```

#### 2.6 Zod 驗證器 (validators/todoValidators.ts)

```typescript
import { z } from 'zod';

export const createTodoSchema = z.object({
  title: z.string().min(1, 'Title is required').max(200, 'Title too long'),
  description: z.string().max(1000, 'Description too long').optional(),
});

export const updateTodoSchema = z.object({
  title: z.string().min(1).max(200).optional(),
  description: z.string().max(1000).optional(),
  completed: z.boolean().optional(),
});

export const queryTodoSchema = z.object({
  completed: z
    .string()
    .transform((val) => val === 'true')
    .optional(),
});

export type CreateTodoDto = z.infer<typeof createTodoSchema>;
export type UpdateTodoDto = z.infer<typeof updateTodoSchema>;
export type QueryTodoDto = z.infer<typeof queryTodoSchema>;
```

#### 2.7 Service 層 (services/todoService.ts)

```typescript
import { Todo } from '@prisma/client';
import { TodoRepository } from '../repositories/TodoRepository';
import { CreateTodoDto, UpdateTodoDto } from '../validators/todoValidators';
import { Sentry } from '../instrument';

export class TodoService {
  constructor(private todoRepository: TodoRepository) {}

  async getAllTodos(userId: number, completed?: boolean): Promise<Todo[]> {
    try {
      return await this.todoRepository.findAll(userId, completed);
    } catch (error) {
      Sentry.captureException(error, {
        tags: { service: 'TodoService', method: 'getAllTodos' },
      });
      throw new Error('Failed to fetch todos');
    }
  }

  async getTodoById(id: number, userId: number): Promise<Todo> {
    try {
      const todo = await this.todoRepository.findById(id, userId);
      if (!todo) {
        throw new Error('Todo not found');
      }
      return todo;
    } catch (error) {
      Sentry.captureException(error);
      throw error;
    }
  }

  async createTodo(userId: number, data: CreateTodoDto): Promise<Todo> {
    try {
      return await this.todoRepository.create({
        ...data,
        user: { connect: { id: userId } },
      });
    } catch (error) {
      Sentry.captureException(error);
      throw new Error('Failed to create todo');
    }
  }

  async updateTodo(id: number, userId: number, data: UpdateTodoDto): Promise<Todo> {
    try {
      // 首先驗證 todo 存在且屬於用戶
      await this.getTodoById(id, userId);
      return await this.todoRepository.update(id, userId, data);
    } catch (error) {
      Sentry.captureException(error);
      throw error;
    }
  }

  async deleteTodo(id: number, userId: number): Promise<void> {
    try {
      await this.getTodoById(id, userId);
      await this.todoRepository.delete(id, userId);
    } catch (error) {
      Sentry.captureException(error);
      throw error;
    }
  }
}
```

#### 2.8 BaseController (controllers/BaseController.ts)

```typescript
import { Response } from 'express';
import { Sentry } from '../instrument';

export class BaseController {
  protected handleSuccess<T>(res: Response, data: T, statusCode: number = 200): void {
    res.status(statusCode).json({
      success: true,
      data,
    });
  }

  protected handleError(error: unknown, res: Response, context: string): void {
    console.error(`Error in ${context}:`, error);

    Sentry.captureException(error, {
      tags: { context },
    });

    if (error instanceof Error) {
      if (error.message === 'Todo not found') {
        res.status(404).json({
          success: false,
          error: error.message,
        });
        return;
      }
    }

    res.status(500).json({
      success: false,
      error: 'Internal server error',
    });
  }
}
```

#### 2.9 Controller 層 (controllers/TodoController.ts)

```typescript
import { Request, Response } from 'express';
import { BaseController } from './BaseController';
import { TodoService } from '../services/todoService';
import {
  createTodoSchema,
  updateTodoSchema,
  queryTodoSchema,
} from '../validators/todoValidators';

export class TodoController extends BaseController {
  constructor(private todoService: TodoService) {
    super();
  }

  async getAllTodos(req: Request, res: Response): Promise<void> {
    try {
      const userId = req.user!.id; // 假設從認證中間件獲取
      const query = queryTodoSchema.parse(req.query);

      const todos = await this.todoService.getAllTodos(userId, query.completed);
      this.handleSuccess(res, todos);
    } catch (error) {
      this.handleError(error, res, 'TodoController.getAllTodos');
    }
  }

  async getTodoById(req: Request, res: Response): Promise<void> {
    try {
      const userId = req.user!.id;
      const id = parseInt(req.params.id, 10);

      const todo = await this.todoService.getTodoById(id, userId);
      this.handleSuccess(res, todo);
    } catch (error) {
      this.handleError(error, res, 'TodoController.getTodoById');
    }
  }

  async createTodo(req: Request, res: Response): Promise<void> {
    try {
      const userId = req.user!.id;
      const data = createTodoSchema.parse(req.body);

      const todo = await this.todoService.createTodo(userId, data);
      this.handleSuccess(res, todo, 201);
    } catch (error) {
      this.handleError(error, res, 'TodoController.createTodo');
    }
  }

  async updateTodo(req: Request, res: Response): Promise<void> {
    try {
      const userId = req.user!.id;
      const id = parseInt(req.params.id, 10);
      const data = updateTodoSchema.parse(req.body);

      const todo = await this.todoService.updateTodo(id, userId, data);
      this.handleSuccess(res, todo);
    } catch (error) {
      this.handleError(error, res, 'TodoController.updateTodo');
    }
  }

  async deleteTodo(req: Request, res: Response): Promise<void> {
    try {
      const userId = req.user!.id;
      const id = parseInt(req.params.id, 10);

      await this.todoService.deleteTodo(id, userId);
      this.handleSuccess(res, { message: 'Todo deleted successfully' });
    } catch (error) {
      this.handleError(error, res, 'TodoController.deleteTodo');
    }
  }
}
```

#### 2.10 Routes 層 (routes/todoRoutes.ts)

```typescript
import { Router } from 'express';
import { TodoController } from '../controllers/TodoController';
import { TodoService } from '../services/todoService';
import { TodoRepository } from '../repositories/TodoRepository';
// import { authMiddleware } from '../middleware/auth'; // 假設您有認證中間件

const router = Router();

// 依賴注入
const todoRepository = new TodoRepository();
const todoService = new TodoService(todoRepository);
const todoController = new TodoController(todoService);

// 路由定義 - 只負責路由，不包含邏輯
router.get('/', (req, res) => todoController.getAllTodos(req, res));
router.get('/:id', (req, res) => todoController.getTodoById(req, res));
router.post('/', (req, res) => todoController.createTodo(req, res));
router.put('/:id', (req, res) => todoController.updateTodo(req, res));
router.delete('/:id', (req, res) => todoController.deleteTodo(req, res));

export default router;
```

#### 2.11 Express App 設置 (app.ts)

```typescript
import './instrument'; // 必須是第一個導入！
import express, { Application } from 'express';
import cors from 'cors';
import * as Sentry from '@sentry/node';
import todoRoutes from './routes/todoRoutes';

const app: Application = express();

// Sentry 請求處理器必須是第一個中間件
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// 基本中間件
app.use(cors());
app.use(express.json());

// 路由
app.use('/api/todos', todoRoutes);

// 健康檢查
app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

// Sentry 錯誤處理器必須在其他錯誤處理器之前
app.use(Sentry.Handlers.errorHandler());

// 錯誤處理中間件
app.use((err: Error, req: express.Request, res: express.Response, next: express.NextFunction) => {
  console.error(err.stack);
  res.status(500).json({
    success: false,
    error: 'Something went wrong!',
  });
});

export default app;
```

#### 2.12 服務器入口 (server.ts)

```typescript
import app from './app';
import { config } from './config/unifiedConfig';
import { prismaService } from './services/prismaService';

const PORT = config.port;

const server = app.listen(PORT, () => {
  console.log(`🚀 Server is running on port ${PORT}`);
  console.log(`Environment: ${config.nodeEnv}`);
});

// 優雅關閉
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down gracefully');
  server.close(async () => {
    await prismaService.disconnect();
    process.exit(0);
  });
});
```

---

## 前端開發

### 階段 3: 前端架構實現

遵循 **frontend-dev-guidelines** 技能中的現代 React 模式。

#### 3.1 初始化前端項目

```bash
cd ../frontend
npm create vite@latest . -- --template react-ts
npm install @mui/material @emotion/react @emotion/styled
npm install @tanstack/react-query @tanstack/react-router
npm install axios
```

#### 3.2 配置導入別名 (vite.config.ts)

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '~types': path.resolve(__dirname, './src/types'),
      '~components': path.resolve(__dirname, './src/components'),
      '~features': path.resolve(__dirname, './src/features'),
    },
  },
});
```

#### 3.3 創建目錄結構

```bash
mkdir -p src/{features/todos/{api,components,hooks,types},components,lib,hooks,types}
```

#### 3.4 API Client (lib/apiClient.ts)

```typescript
import axios from 'axios';

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:3000/api',
  headers: {
    'Content-Type': 'application/json',
  },
});

// 請求攔截器（例如添加認證 token）
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// 響應攔截器
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // 處理未授權
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

#### 3.5 TypeScript 類型 (features/todos/types/index.ts)

```typescript
export interface Todo {
  id: number;
  title: string;
  description?: string;
  completed: boolean;
  userId: number;
  createdAt: string;
  updatedAt: string;
}

export interface CreateTodoDto {
  title: string;
  description?: string;
}

export interface UpdateTodoDto {
  title?: string;
  description?: string;
  completed?: boolean;
}

export type TodoFilter = 'all' | 'active' | 'completed';
```

#### 3.6 API Service (features/todos/api/todoApi.ts)

```typescript
import { apiClient } from '@/lib/apiClient';
import type { Todo, CreateTodoDto, UpdateTodoDto } from '../types';

export const todoApi = {
  async getTodos(completed?: boolean): Promise<Todo[]> {
    const params = completed !== undefined ? { completed } : {};
    const response = await apiClient.get<{ success: boolean; data: Todo[] }>('/todos', {
      params,
    });
    return response.data.data;
  },

  async getTodoById(id: number): Promise<Todo> {
    const response = await apiClient.get<{ success: boolean; data: Todo }>(`/todos/${id}`);
    return response.data.data;
  },

  async createTodo(data: CreateTodoDto): Promise<Todo> {
    const response = await apiClient.post<{ success: boolean; data: Todo }>('/todos', data);
    return response.data.data;
  },

  async updateTodo(id: number, data: UpdateTodoDto): Promise<Todo> {
    const response = await apiClient.put<{ success: boolean; data: Todo }>(`/todos/${id}`, data);
    return response.data.data;
  },

  async deleteTodo(id: number): Promise<void> {
    await apiClient.delete(`/todos/${id}`);
  },
};
```

#### 3.7 SuspenseLoader 組件 (components/SuspenseLoader/SuspenseLoader.tsx)

```typescript
import React from 'react';
import { Box, CircularProgress, Fade } from '@mui/material';
import type { SxProps, Theme } from '@mui/material';

interface SuspenseLoaderProps {
  children: React.ReactNode;
  minHeight?: string;
}

const styles: Record<string, SxProps<Theme>> = {
  container: {
    display: 'flex',
    justifyContent: 'center',
    alignItems: 'center',
    minHeight: '200px',
  },
};

export const SuspenseLoader: React.FC<SuspenseLoaderProps> = ({ children, minHeight }) => {
  return (
    <React.Suspense
      fallback={
        <Fade in timeout={300}>
          <Box sx={{ ...styles.container, minHeight: minHeight || '200px' }}>
            <CircularProgress />
          </Box>
        </Fade>
      }
    >
      {children}
    </React.Suspense>
  );
};

export default SuspenseLoader;
```

#### 3.8 自定義 Hook (features/todos/hooks/useTodos.ts)

```typescript
import { useSuspenseQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { todoApi } from '../api/todoApi';
import type { CreateTodoDto, UpdateTodoDto, TodoFilter } from '../types';

export const useTodos = (filter: TodoFilter = 'all') => {
  const queryClient = useQueryClient();

  const completed = filter === 'all' ? undefined : filter === 'completed';

  const { data: todos } = useSuspenseQuery({
    queryKey: ['todos', filter],
    queryFn: () => todoApi.getTodos(completed),
  });

  const createMutation = useMutation({
    mutationFn: (data: CreateTodoDto) => todoApi.createTodo(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  const updateMutation = useMutation({
    mutationFn: ({ id, data }: { id: number; data: UpdateTodoDto }) =>
      todoApi.updateTodo(id, data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  const deleteMutation = useMutation({
    mutationFn: (id: number) => todoApi.deleteTodo(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  return {
    todos,
    createTodo: createMutation.mutate,
    updateTodo: updateMutation.mutate,
    deleteTodo: deleteMutation.mutate,
    isCreating: createMutation.isPending,
    isUpdating: updateMutation.isPending,
    isDeleting: deleteMutation.isPending,
  };
};
```

#### 3.9 TodoItem 組件 (features/todos/components/TodoItem.tsx)

```typescript
import React, { useState, useCallback } from 'react';
import {
  Card,
  CardContent,
  Typography,
  Checkbox,
  IconButton,
  Box,
  TextField,
  Button,
} from '@mui/material';
import { Delete as DeleteIcon, Edit as EditIcon, Save as SaveIcon } from '@mui/icons-material';
import type { SxProps, Theme } from '@mui/material';
import type { Todo, UpdateTodoDto } from '../types';

interface TodoItemProps {
  todo: Todo;
  onUpdate: (id: number, data: UpdateTodoDto) => void;
  onDelete: (id: number) => void;
}

const styles: Record<string, SxProps<Theme>> = {
  card: {
    mb: 2,
    '&:hover': {
      boxShadow: 3,
    },
  },
  content: {
    display: 'flex',
    alignItems: 'center',
    gap: 2,
  },
  textContainer: {
    flex: 1,
  },
  completedText: {
    textDecoration: 'line-through',
    color: 'text.secondary',
  },
  actions: {
    display: 'flex',
    gap: 1,
  },
};

export const TodoItem: React.FC<TodoItemProps> = ({ todo, onUpdate, onDelete }) => {
  const [isEditing, setIsEditing] = useState(false);
  const [editTitle, setEditTitle] = useState(todo.title);
  const [editDescription, setEditDescription] = useState(todo.description || '');

  const handleToggleComplete = useCallback(() => {
    onUpdate(todo.id, { completed: !todo.completed });
  }, [todo.id, todo.completed, onUpdate]);

  const handleSave = useCallback(() => {
    onUpdate(todo.id, {
      title: editTitle,
      description: editDescription,
    });
    setIsEditing(false);
  }, [todo.id, editTitle, editDescription, onUpdate]);

  const handleDelete = useCallback(() => {
    if (window.confirm('確定要刪除這個待辦事項嗎？')) {
      onDelete(todo.id);
    }
  }, [todo.id, onDelete]);

  return (
    <Card sx={styles.card}>
      <CardContent sx={styles.content}>
        <Checkbox checked={todo.completed} onChange={handleToggleComplete} />

        <Box sx={styles.textContainer}>
          {isEditing ? (
            <>
              <TextField
                fullWidth
                size="small"
                value={editTitle}
                onChange={(e) => setEditTitle(e.target.value)}
                sx={{ mb: 1 }}
              />
              <TextField
                fullWidth
                size="small"
                multiline
                rows={2}
                value={editDescription}
                onChange={(e) => setEditDescription(e.target.value)}
              />
            </>
          ) : (
            <>
              <Typography
                variant="h6"
                sx={todo.completed ? styles.completedText : undefined}
              >
                {todo.title}
              </Typography>
              {todo.description && (
                <Typography
                  variant="body2"
                  color="text.secondary"
                  sx={todo.completed ? styles.completedText : undefined}
                >
                  {todo.description}
                </Typography>
              )}
            </>
          )}
        </Box>

        <Box sx={styles.actions}>
          {isEditing ? (
            <>
              <IconButton color="primary" onClick={handleSave}>
                <SaveIcon />
              </IconButton>
              <Button onClick={() => setIsEditing(false)}>取消</Button>
            </>
          ) : (
            <>
              <IconButton color="primary" onClick={() => setIsEditing(true)}>
                <EditIcon />
              </IconButton>
              <IconButton color="error" onClick={handleDelete}>
                <DeleteIcon />
              </IconButton>
            </>
          )}
        </Box>
      </CardContent>
    </Card>
  );
};

export default TodoItem;
```

#### 3.10 TodoList 主組件 (features/todos/components/TodoList.tsx)

```typescript
import React, { useState, useCallback } from 'react';
import {
  Container,
  Paper,
  Typography,
  Box,
  TextField,
  Button,
  ToggleButtonGroup,
  ToggleButton,
  Grid,
} from '@mui/material';
import type { SxProps, Theme } from '@mui/material';
import { useTodos } from '../hooks/useTodos';
import type { TodoFilter } from '../types';
import TodoItem from './TodoItem';

const styles: Record<string, SxProps<Theme>> = {
  container: {
    py: 4,
  },
  paper: {
    p: 3,
    mb: 3,
  },
  header: {
    mb: 3,
    textAlign: 'center',
  },
  addForm: {
    mb: 3,
  },
  filterBar: {
    mb: 3,
    display: 'flex',
    justifyContent: 'center',
  },
};

export const TodoList: React.FC = () => {
  const [filter, setFilter] = useState<TodoFilter>('all');
  const [newTitle, setNewTitle] = useState('');
  const [newDescription, setNewDescription] = useState('');

  const { todos, createTodo, updateTodo, deleteTodo, isCreating } = useTodos(filter);

  const handleAddTodo = useCallback(() => {
    if (!newTitle.trim()) return;

    createTodo(
      {
        title: newTitle,
        description: newDescription || undefined,
      },
      {
        onSuccess: () => {
          setNewTitle('');
          setNewDescription('');
        },
      }
    );
  }, [newTitle, newDescription, createTodo]);

  const handleFilterChange = useCallback(
    (_: React.MouseEvent<HTMLElement>, newFilter: TodoFilter | null) => {
      if (newFilter !== null) {
        setFilter(newFilter);
      }
    },
    []
  );

  return (
    <Container maxWidth="md" sx={styles.container}>
      <Typography variant="h3" sx={styles.header}>
        📝 我的待辦事項
      </Typography>

      {/* 新增表單 */}
      <Paper sx={{ ...styles.paper, ...styles.addForm }}>
        <Grid container spacing={2}>
          <Grid size={{ xs: 12 }}>
            <TextField
              fullWidth
              label="標題"
              value={newTitle}
              onChange={(e) => setNewTitle(e.target.value)}
              onKeyPress={(e) => e.key === 'Enter' && handleAddTodo()}
            />
          </Grid>
          <Grid size={{ xs: 12 }}>
            <TextField
              fullWidth
              label="描述（可選）"
              multiline
              rows={2}
              value={newDescription}
              onChange={(e) => setNewDescription(e.target.value)}
            />
          </Grid>
          <Grid size={{ xs: 12 }}>
            <Button
              variant="contained"
              fullWidth
              onClick={handleAddTodo}
              disabled={isCreating || !newTitle.trim()}
            >
              新增待辦事項
            </Button>
          </Grid>
        </Grid>
      </Paper>

      {/* 過濾器 */}
      <Box sx={styles.filterBar}>
        <ToggleButtonGroup value={filter} exclusive onChange={handleFilterChange}>
          <ToggleButton value="all">全部</ToggleButton>
          <ToggleButton value="active">未完成</ToggleButton>
          <ToggleButton value="completed">已完成</ToggleButton>
        </ToggleButtonGroup>
      </Box>

      {/* 待辦事項列表 */}
      <Box>
        {todos.length === 0 ? (
          <Paper sx={styles.paper}>
            <Typography variant="body1" color="text.secondary" textAlign="center">
              {filter === 'all' && '還沒有待辦事項，開始添加一些吧！'}
              {filter === 'active' && '沒有未完成的待辦事項'}
              {filter === 'completed' && '還沒有完成的待辦事項'}
            </Typography>
          </Paper>
        ) : (
          todos.map((todo) => (
            <TodoItem key={todo.id} todo={todo} onUpdate={updateTodo} onDelete={deleteTodo} />
          ))
        )}
      </Box>

      {/* 統計 */}
      <Paper sx={styles.paper}>
        <Typography variant="body2" color="text.secondary" textAlign="center">
          {filter === 'all' && `共 ${todos.length} 個待辦事項`}
          {filter === 'active' && `${todos.length} 個未完成`}
          {filter === 'completed' && `${todos.length} 個已完成`}
        </Typography>
      </Paper>
    </Container>
  );
};

export default TodoList;
```

#### 3.11 路由設置 (routes/todos/index.tsx)

```typescript
import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';

const TodoList = lazy(() => import('~features/todos/components/TodoList'));

export const Route = createFileRoute('/todos/')({
  component: TodoList,
  loader: () => ({ crumb: '待辦事項' }),
});
```

#### 3.12 App 入口 (App.tsx)

```typescript
import React from 'react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ThemeProvider, createTheme, CssBaseline } from '@mui/material';
import { SuspenseLoader } from '~components/SuspenseLoader';

const TodoList = React.lazy(() => import('~features/todos/components/TodoList'));

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
});

const theme = createTheme({
  palette: {
    mode: 'light',
    primary: {
      main: '#1976d2',
    },
  },
});

const App: React.FC = () => {
  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider theme={theme}>
        <CssBaseline />
        <SuspenseLoader>
          <TodoList />
        </SuspenseLoader>
      </ThemeProvider>
    </QueryClientProvider>
  );
};

export default App;
```

---

## 使用 Claude Code 開發

### 使用 Skills 進行開發

當您在 `.claude/skills/` 中設置了 skills 後，Claude 會在相關文件中自動激活：

#### 1. 後端開發時

編輯 `backend/src/controllers/TodoController.ts` 時：

```
您：我想添加一個批量更新待辦事項狀態的功能

Claude：[backend-dev-guidelines skill 自動激活]
好的，我會遵循後端開發指南來實現這個功能。讓我：

1. 在 TodoRepository 添加批量更新方法
2. 在 TodoService 添加業務邏輯
3. 在 TodoController 添加新的端點
4. 創建 Zod 驗證器
5. 添加到路由
6. 確保所有錯誤都發送到 Sentry

[Claude 會按照分層架構實現功能]
```

#### 2. 前端開發時

編輯 `frontend/src/features/todos/components/TodoList.tsx` 時：

```
您：我想添加拖放功能來重新排序待辦事項

Claude：[frontend-dev-guidelines skill 自動激活]
好的，我會使用現代 React 模式來實現：

1. 創建一個自定義 hook useTodoDragDrop
2. 使用 useCallback 優化拖放處理器
3. 確保組件使用 React.FC 模式
4. 使用 SuspenseLoader 處理加載狀態
5. 使用 useMuiSnackbar 顯示成功消息

[Claude 會按照前端開發指南實現]
```

### 使用 Agents 進行審查

完成功能後，使用 agents 進行審查：

```bash
您：使用 code-architecture-reviewer agent 審查我的 Todo 功能實現

Claude：[啟動 code-architecture-reviewer agent]
我將審查您的 Todo 功能實現...

審查報告：
✅ 分層架構正確實施
✅ 所有控制器繼承 BaseController
✅ 使用了 Repository 模式
✅ Zod 驗證已實施
⚠️  建議：在 TodoService 中添加更多單元測試
✅ Sentry 錯誤跟踪已配置

總體評分：9/10
```

### 使用 Commands 創建計劃

對於複雜的功能，使用 `/dev-docs` 命令：

```bash
您：/dev-docs 為 Todo List 添加標籤和優先級功能

Claude：[創建綜合計劃]
已創建計劃文檔在 dev/active/todo-tags-priority/

文件：
- todo-tags-priority-plan.md - 完整的實施計劃
- todo-tags-priority-context.md - 關鍵文件和決策
- todo-tags-priority-tasks.md - 任務檢查清單

執行摘要：
1. 數據庫 Schema 更新（添加 Tag 和 Priority 模型）
2. 後端 API 實現（CRUD 操作）
3. 前端組件開發（標籤選擇器、優先級選擇器）
4. 集成和測試

估計時間：2-3 天
```

---

## 測試和部署

### 後端測試

創建 `backend/src/tests/todoService.test.ts`：

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { TodoService } from '../services/todoService';
import { TodoRepository } from '../repositories/TodoRepository';

describe('TodoService', () => {
  let todoService: TodoService;
  let mockRepository: jest.Mocked<TodoRepository>;

  beforeEach(() => {
    mockRepository = {
      findAll: jest.fn(),
      findById: jest.fn(),
      create: jest.fn(),
      update: jest.fn(),
      delete: jest.fn(),
    } as any;

    todoService = new TodoService(mockRepository);
  });

  it('should create a todo', async () => {
    const mockTodo = {
      id: 1,
      title: 'Test Todo',
      completed: false,
      userId: 1,
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    mockRepository.create.mockResolvedValue(mockTodo);

    const result = await todoService.createTodo(1, { title: 'Test Todo' });

    expect(result).toEqual(mockTodo);
    expect(mockRepository.create).toHaveBeenCalledWith({
      title: 'Test Todo',
      user: { connect: { id: 1 } },
    });
  });
});
```

### 運行測試

```bash
# 後端
cd backend
npm test

# 前端
cd frontend
npm test
```

---

## 完整代碼示例

所有完整的代碼示例都已在上面的各個部分中提供。您可以直接複製使用。

### 快速開始腳本

創建 `package.json` scripts：

**後端 (backend/package.json):**
```json
{
  "scripts": {
    "dev": "nodemon --exec ts-node src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "test": "vitest",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev"
  }
}
```

**前端 (frontend/package.json):**
```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "test": "vitest"
  }
}
```

### 環境變量

**後端 (.env):**
```env
DATABASE_URL="postgresql://user:password@localhost:5432/todoapp"
PORT=3000
NODE_ENV=development
SENTRY_DSN=your_sentry_dsn_here
```

**前端 (.env):**
```env
VITE_API_URL=http://localhost:3000/api
```

---

## 總結

這個實戰教學展示了如何使用 Claude Code Infrastructure Showcase 中的最佳實踐構建一個完整的 Todo List 應用：

### 後端遵循的原則：
✅ 分層架構（Routes → Controllers → Services → Repositories）
✅ BaseController 模式
✅ Sentry 錯誤跟踪
✅ Zod 驗證
✅ UnifiedConfig
✅ 依賴注入
✅ TypeScript 嚴格模式

### 前端遵循的原則：
✅ React.lazy() 和 Suspense
✅ useSuspenseQuery 數據獲取
✅ Features 目錄結構
✅ MUI v7 樣式
✅ 導入別名
✅ useCallback 優化
✅ 無 early return

### Claude Code 集成：
✅ Skills 自動激活提供指導
✅ Agents 進行代碼審查
✅ Commands 創建實施計劃
✅ Hooks 跟踪文件更改

這個教學提供了一個完整的、生產就緒的架構模板，您可以將其擴展為更複雜的應用。

---

**作者：** Claude Code Infrastructure Showcase
**最後更新：** 2025-11-17
**版本：** 1.0.0
