# Full-Stack TypeScript Project Setup Guide

This document defines the canonical structure, tooling, and coding patterns for full-stack TypeScript projects. Coding agents must follow this guide precisely when scaffolding or extending any project.

---

## 1. Repository Layout

**Monorepo with pnpm workspaces.**

```
<project-name>/
├── apps/
│   ├── server/          # NestJS backend
│   └── web/             # React frontend (Vite)
├── packages/
│   └── shared/          # Shared Zod schemas, DTOs, types
├── deploy/              # Deployment scripts (blue-green)
├── nginx/               # Nginx config
├── docker-compose.dev.yml
├── docker-compose.prod.yml
├── pnpm-workspace.yaml
└── package.json         # Root — scripts only, no deps
```

**`pnpm-workspace.yaml`**:

```yaml
packages:
    - "apps/*"
    - "packages/*"
```

**Root `package.json`**:

```json
{
    "name": "<project-name>",
    "private": true,
    "packageManager": "pnpm@10.33.0",
    "engines": { "node": ">=20" },
    "scripts": {
        "dev:web": "pnpm --filter web dev",
        "dev:server": "pnpm --filter server start:dev",
        "build": "pnpm --filter @<project>/shared build && pnpm --filter web build && pnpm --filter server build"
    }
}
```

---

## 2. Tech Stack

| Layer              | Technology                                | Version    |
| ------------------ | ----------------------------------------- | ---------- |
| Backend framework  | NestJS                                    | 11.x       |
| Backend ORM        | Kysely                                    | 0.28.x     |
| Database           | PostgreSQL                                | 16         |
| Cache / pub-sub    | Redis (ioredis)                           | 7 / 5.x    |
| Real-time          | Socket.io                                 | 4.x        |
| Auth               | Google OAuth 2.0 + JWT (httpOnly cookies) | —          |
| Frontend framework | React + Vite                              | 19.x / 8.x |
| Frontend state     | Jotai                                     | 2.x        |
| Frontend routing   | React Router                              | 7.x        |
| UI library         | Mantine + Tailwind CSS                    | 8.x / 3.x  |
| Validation         | Zod (shared)                              | 4.x        |
| HTTP client        | Axios                                     | 1.x        |
| Package manager    | pnpm                                      | 10.33.0    |
| Runtime            | Node.js                                   | ≥20        |
| TypeScript         | —                                         | 5.7.x      |
| Linting            | ESLint 9 (flat config)                    | 9.x        |
| Formatting         | Prettier                                  | 3.x        |
| Containers         | Docker Compose                            | —          |
| Reverse proxy      | Nginx (blue-green)                        | —          |

---

## 3. Shared Package (`packages/shared`)

Everything shared between frontend and backend lives here. **Never duplicate types across apps.**

### Structure

```
packages/shared/
├── src/
│   ├── types/           # Domain enums and helpers (access levels, roles)
│   ├── http/            # HTTP request/response Zod schemas + TS types
│   ├── socket/          # Socket.io event names + payload schemas
│   └── index.ts         # Re-exports everything
├── package.json
└── tsconfig.json
```

### `package.json`

```json
{
    "name": "@<project>/shared",
    "version": "0.0.1",
    "main": "./dist/cjs/index.js",
    "module": "./dist/esm/index.js",
    "types": "./dist/types/index.d.ts",
    "exports": {
        ".": {
            "require": "./dist/cjs/index.js",
            "import": "./dist/esm/index.js",
            "types": "./dist/types/index.d.ts"
        }
    },
    "scripts": {
        "build": "tsc -p tsconfig.cjs.json && tsc -p tsconfig.esm.json && tsc -p tsconfig.types.json",
        "dev": "tsc -p tsconfig.cjs.json --watch"
    },
    "dependencies": { "zod": "^4.3.6" }
}
```

### Naming conventions

- Schemas: `<Name>Schema` (Zod object)
- Types: `<Name>Dto` (inferred via `z.infer<typeof <Name>Schema>`)
- Socket events: `SOCKET_EVENTS` const object with `SERVER` (client→server) and `CLIENT` (server→client) variants

### Pattern — HTTP DTO

```typescript
// packages/shared/src/http/document.ts
import { z } from "zod";

export const CreateDocumentRequestSchema = z.object({
    workspaceId: z.number(),
});
export type CreateDocumentRequestDto = z.infer<
    typeof CreateDocumentRequestSchema
>;

export const CreateDocumentResponseSchema = z.object({
    ok: z.literal(true),
    documentId: z.number(),
});
export type CreateDocumentResponseDto = z.infer<
    typeof CreateDocumentResponseSchema
>;
```

### Pattern — Socket events

```typescript
// packages/shared/src/socket/events.ts
export const SOCKET_EVENTS = {
    // client → server
    SYNC_DOC_SERVER: "sync-doc-server",
    // server → client
    SYNC_DOC_CLIENT: "sync-doc-client",
    DOC_READY: "doc-ready",
    DISCONNECT: "disconnect",
} as const;

export const SyncDocServerSchema = z.object({
    update: z.instanceof(Uint8Array),
});
```

### Pattern — Domain types

```typescript
// packages/shared/src/types/access.ts
export const DocumentAccessLevel = z.enum([
    "owner",
    "admin",
    "editor",
    "viewer",
    "noAccess",
]);
export type DocumentAccessLevel = z.infer<typeof DocumentAccessLevel>;

export const ACCESS_RANK: Record<DocumentAccessLevel, number> = {
    owner: 4,
    admin: 3,
    editor: 2,
    viewer: 1,
    noAccess: 0,
};

export const hasAccess = (
    userLevel: DocumentAccessLevel,
    required: DocumentAccessLevel,
): boolean => ACCESS_RANK[userLevel] >= ACCESS_RANK[required];
```

---

## 4. Backend (`apps/server`)

### Framework: NestJS with standard module pattern

### Directory layout

```
apps/server/src/
├── <feature>/
│   ├── <feature>.controller.ts
│   ├── <feature>.service.ts
│   ├── <feature>.module.ts
│   └── <feature>.gateway.ts    # Only if real-time needed
├── auth/
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── auth.guard.ts
│   └── auth.module.ts
├── db/
│   ├── database.schema.ts      # Kysely table type map
│   ├── database.service.ts     # Pool + migrations runner
│   └── database.module.ts
├── redis/
│   ├── redis.service.ts
│   └── redis.module.ts
├── guards/
│   └── throttler.guard.ts
├── pipes/
│   └── zod-validation.pipe.ts
├── utils/
│   ├── env.loader.ts
│   ├── error.filter.ts
│   └── http.helper.ts          # httpOK() wrapper
├── migrations/                 # Kysely migration files (0001_..., 0002_...)
├── app.module.ts
└── main.ts
```

### `package.json` key deps

```json
{
    "dependencies": {
        "@nestjs/common": "^11.0.1",
        "@nestjs/core": "^11.0.1",
        "@nestjs/config": "^4.0.2",
        "@nestjs/platform-express": "^11.0.1",
        "@nestjs/platform-socket.io": "^11.0.5",
        "@nestjs/throttler": "^6.4.0",
        "@nestjs/websockets": "^11.0.5",
        "kysely": "^0.28.15",
        "pg": "^8.20.0",
        "ioredis": "^5.10.1",
        "socket.io": "^4.8.3",
        "jsonwebtoken": "^9.0.3",
        "zod": "^4.3.6",
        "axios": "^1.14.0",
        "@<project>/shared": "workspace:*"
    },
    "devDependencies": {
        "@nestjs/cli": "^11.0.0",
        "@nestjs/schematics": "^11.0.0",
        "typescript": "^5.7.3",
        "eslint": "^9.18.0",
        "prettier": "^3.0.0",
        "eslint-plugin-prettier": "^5.0.0",
        "typescript-eslint": "^8.0.0"
    }
}
```

### TypeScript config (`tsconfig.json`)

```json
{
    "compilerOptions": {
        "target": "ES2023",
        "module": "nodenext",
        "moduleResolution": "nodenext",
        "strict": true,
        "experimentalDecorators": true,
        "emitDecoratorMetadata": true,
        "sourceMap": true,
        "outDir": "./dist",
        "rootDir": "./src"
    }
}
```

### Patterns

#### Database service

```typescript
// db/database.service.ts
@Injectable()
export class DatabaseService implements OnModuleInit {
    kysely!: Kysely<Database>;

    async onModuleInit() {
        this.kysely = new Kysely<Database>({
            dialect: new PostgresDialect({
                pool: new Pool({ connectionString: process.env.DATABASE_URL }),
            }),
        });
        await this.runMigrations();
    }

    private async runMigrations() {
        const migrator = new Migrator({
            db: this.kysely,
            provider: new FileMigrationProvider({
                fs,
                path,
                migrationFolder: join(__dirname, "migrations"),
            }),
        });
        await migrator.migrateToLatest();
    }
}
```

#### Kysely schema

```typescript
// db/database.schema.ts
import { Generated, ColumnType } from "kysely";

export interface UsersTable {
    id: Generated<number>;
    email: string;
    name: string;
    avatar_url: string | null;
    created_at: ColumnType<Date, never, never>;
}

export interface Database {
    users: UsersTable;
    // ...other tables
}
```

#### Service

```typescript
@Injectable()
export class DocumentService {
    constructor(
        private readonly db: DatabaseService,
        private readonly accessService: DocumentAccessService,
    ) {}

    async getDocument(
        documentId: number,
        userId: number,
    ): Promise<GetDocumentResponseDto> {
        const access = await this.accessService.resolveAccess(
            documentId,
            userId,
        );
        if (!hasAccess(access, "viewer"))
            throw new ForbiddenException("No access");

        const row = await this.db.kysely
            .selectFrom("documents as d")
            .innerJoin("workspaces as w", "w.id", "d.workspace_id")
            .select(["d.id", "d.title", "w.name as workspaceName"])
            .where("d.id", "=", documentId)
            .executeTakeFirstOrThrow();

        return httpOK(row);
    }
}
```

#### Controller

```typescript
@Controller("/document")
@UseGuards(AuthGuard)
export class DocumentController {
    constructor(private readonly documentService: DocumentService) {}

    @Get("/id/:documentId")
    async handleGetDocument(
        @Req() req: Request,
        @Param("documentId", ParseIntPipe) documentId: number,
    ): Promise<GetDocumentResponseDto> {
        const userId = (req as any).userId as number;
        return this.documentService.getDocument(documentId, userId);
    }

    @Post("/")
    async handleCreateDocument(
        @Req() req: Request,
        @Body(new ZodHttpValidationPipe(CreateDocumentRequestSchema))
        body: CreateDocumentRequestDto,
    ): Promise<CreateDocumentResponseDto> {
        const userId = (req as any).userId as number;
        return this.documentService.createDocument(userId, body.workspaceId);
    }
}
```

#### Auth guard

```typescript
// auth/auth.guard.ts
@Injectable()
export class AuthGuard implements CanActivate {
    canActivate(context: ExecutionContext): boolean {
        const req = context.switchToHttp().getRequest();
        const token = req.cookies?.token;
        if (!token) throw new UnauthorizedException();
        try {
            const payload = jwt.verify(token, process.env.JWT_SECRET!) as {
                userId: number;
            };
            req.userId = payload.userId;
            return true;
        } catch {
            throw new UnauthorizedException();
        }
    }
}
```

#### Zod validation pipe

```typescript
// pipes/zod-validation.pipe.ts
@Injectable()
export class ZodHttpValidationPipe implements PipeTransform {
    constructor(private schema: ZodSchema) {}

    transform(value: unknown) {
        const result = this.schema.safeParse(value);
        if (!result.success)
            throw new BadRequestException(result.error.flatten());
        return result.data;
    }
}
```

#### HTTP helper

```typescript
// utils/http.helper.ts
export const httpOK = <T>(data: T): { ok: true } & T => ({ ok: true, ...data });
```

#### Socket.io gateway

```typescript
@WebSocketGateway({
    cors: { origin: process.env.CLIENT_URL, credentials: true },
})
export class DocumentGateway implements OnGatewayConnection {
    @WebSocketServer() server!: Server;

    async handleConnection(client: Socket) {
        const documentId = client.handshake.query.documentId as string;
        // validate, join room
        await client.join(documentId);
        client.emit(SOCKET_EVENTS.DOC_READY);
    }

    @SubscribeMessage(SOCKET_EVENTS.SYNC_DOC_SERVER)
    async handleSync(
        @ConnectedSocket() client: Socket,
        @MessageBody(new ZodWsValidationPipe(SyncDocServerSchema))
        data: SyncDocServerDto,
    ) {
        // process and broadcast
        client.to(data.documentId).emit(SOCKET_EVENTS.SYNC_DOC_CLIENT, data);
    }
}
```

#### Environment loading

```typescript
// utils/env.loader.ts
import { config } from "dotenv";
import { join } from "path";

export const loadEnv = () => {
    const env = process.env.NODE_ENV ?? "dev";
    config({ path: join(process.cwd(), `.env.${env}`), override: false });
    config({ path: join(process.cwd(), ".env"), override: false });
};
```

Call `loadEnv()` as the very first line in `main.ts` before the NestJS bootstrap.

#### `main.ts`

```typescript
import { loadEnv } from "./utils/env.loader";
loadEnv();

import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import cookieParser from "cookie-parser";

async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    app.use(cookieParser());
    app.enableCors({ origin: process.env.CLIENT_URL, credentials: true });
    await app.listen(process.env.PORT ?? 5000);
}
bootstrap();
```

### ESLint config (`eslint.config.mjs`)

```js
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";
import prettierPlugin from "eslint-plugin-prettier/recommended";

export default tseslint.config(
    eslint.configs.recommended,
    ...tseslint.configs.recommended,
    prettierPlugin,
    {
        rules: {
            "@typescript-eslint/no-explicit-any": "off",
            "@typescript-eslint/no-floating-promises": "warn",
        },
    },
);
```

### Prettier config (`.prettierrc`)

```json
{
    "singleQuote": true,
    "trailingComma": "all"
}
```

---

## 5. Frontend (`apps/web`)

### `package.json` key deps

```json
{
    "dependencies": {
        "react": "^19.2.4",
        "react-dom": "^19.2.4",
        "react-router-dom": "^7.13.2",
        "jotai": "^2.19.0",
        "@mantine/core": "^8.3.18",
        "@mantine/hooks": "^8.3.18",
        "axios": "^1.14.0",
        "socket.io-client": "^4.8.3",
        "zod": "^4.3.6",
        "@<project>/shared": "workspace:*"
    },
    "devDependencies": {
        "vite": "^8.0.1",
        "@vitejs/plugin-react": "^4.0.0",
        "tailwindcss": "^3.4.19",
        "postcss": "^8.4.0",
        "autoprefixer": "^10.4.0",
        "typescript": "^5.7.3",
        "eslint": "^9.18.0",
        "typescript-eslint": "^8.0.0",
        "eslint-plugin-react-hooks": "^5.0.0",
        "eslint-plugin-react-refresh": "^0.4.0"
    }
}
```

### Directory layout

```
apps/web/src/
├── pages/
│   ├── <feature>/
│   │   ├── <Feature>Page.tsx    # Page root component
│   │   └── <subComponent>/     # Sub-components scoped to page
│   └── not-found/
├── components/                  # Reusable cross-page components
│   └── ui/                      # Primitive UI components (no business logic)
├── atoms/                       # Jotai atoms
│   ├── auth.ts
│   ├── socket.ts
│   └── <feature>.ts
├── hooks/                       # Custom hooks (one concern per hook)
│   ├── useAuth.ts
│   ├── useSocket.ts
│   └── use<Feature>.ts
├── lib/
│   ├── http.ts                  # Axios instance
│   └── socket.ts                # Socket.io singleton
├── utils/
│   └── utils.ts
├── theme/
├── constants/
│   └── constants.ts
├── App.tsx                      # Route tree only
└── main.tsx
```

### Patterns

#### Jotai atoms

```typescript
// atoms/auth.ts
import { atom } from "jotai";

type AuthState =
    | { status: "loading" }
    | { status: "authenticated"; user: UserDto }
    | { status: "unauthenticated"; user: null };

export const authAtom = atom<AuthState>({ status: "loading" });
```

Keep atoms small and primitive. Derive computed state with `atom(get => ...)` rather than storing it.

#### HTTP client

```typescript
// lib/http.ts
import axios from "axios";

export const apiClient = axios.create({
    baseURL: import.meta.env.VITE_SERVER_URL,
    withCredentials: true,
});
```

#### Socket.io singleton

```typescript
// lib/socket.ts
import { io } from "socket.io-client";

export const socket = io(import.meta.env.VITE_SERVER_URL, {
    autoConnect: false,
    withCredentials: true,
});
```

#### Custom hook (data fetching)

```typescript
// hooks/useLibrary.ts
export const useLibrary = () => {
    const [documents, setDocuments] = useState<LibraryDocumentDto[]>([]);
    const [status, setStatus] = useState<"loading" | "ready" | "error">(
        "loading",
    );

    useEffect(() => {
        const fetch = async () => {
            try {
                const { data } =
                    await apiClient.get<GetLibraryDocumentsResponseDto>(
                        "/document/library",
                    );
                setDocuments(data.documents);
                setStatus("ready");
            } catch {
                setStatus("error");
            }
        };
        fetch();
    }, []);

    return { documents, status };
};
```

#### Custom hook (socket lifecycle)

```typescript
// hooks/useSocket.ts
export const useSocket = (canConnect: boolean, documentId?: string) => {
    const setIsSocketReady = useSetAtom(isSocketReadyAtom);

    useEffect(() => {
        const onReady = () => setIsSocketReady(true);
        const onDisconnect = () => setIsSocketReady(false);

        socket.on(SOCKET_EVENTS.DOC_READY, onReady);
        socket.on(SOCKET_EVENTS.DISCONNECT, onDisconnect);

        if (canConnect && documentId) {
            socket.io.opts.query = { documentId };
            socket.connect();
        }

        return () => {
            socket.off(SOCKET_EVENTS.DOC_READY, onReady);
            socket.off(SOCKET_EVENTS.DISCONNECT, onDisconnect);
            socket.disconnect();
        };
    }, [canConnect, documentId]);
};
```

#### Page component

```typescript
// pages/editor/EditorPage.tsx
const EditorPage = () => {
  const { documentId } = useParams();
  const { editor, status, access } = useEditor(documentId);
  const isSocketReady = useAtomValue(isSocketReadyAtom);

  if (status === 'forbidden') return <div>No access</div>;

  return (
    <Page authRequired haveSidebar>
      {status === 'ready' && <EditorPageHeader ... />}
      <div className="flex-1 overflow-y-auto">
        {/* content */}
      </div>
    </Page>
  );
};
```

#### `Page` auth wrapper

```typescript
// components/Page.tsx
interface PageProps {
  children: ReactNode;
  authRequired?: boolean;
  haveSidebar?: boolean;
}

const Page = ({ children, authRequired, haveSidebar }: PageProps) => {
  const auth = useAtomValue(authAtom);

  if (authRequired && auth.status === 'loading') return <LoadingScreen />;
  if (authRequired && auth.status === 'unauthenticated') {
    return <Navigate to="/auth" replace />;
  }

  return (
    <div className="flex h-screen">
      {haveSidebar && <Sidebar />}
      <main className="flex-1 flex flex-col">{children}</main>
    </div>
  );
};
```

#### `App.tsx` (route tree only — no logic)

```typescript
const App = () => (
  <BrowserRouter>
    <Routes>
      <Route path="/" element={<Navigate to="/library" replace />} />
      <Route path="/library" element={<LibraryPage />} />
      <Route path="/document/:documentId" element={<EditorPage />} />
      <Route path="/workspaces" element={<WorkspacesPage />} />
      <Route path="/auth" element={<AuthPage />} />
      <Route path="/auth/callback" element={<AuthCallbackPage />} />
      <Route path="*" element={<NotFoundPage />} />
    </Routes>
  </BrowserRouter>
);
```

### ESLint config (`eslint.config.js`)

```js
import js from "@eslint/js";
import tseslint from "typescript-eslint";
import reactHooks from "eslint-plugin-react-hooks";
import reactRefresh from "eslint-plugin-react-refresh";

export default tseslint.config(
    js.configs.recommended,
    ...tseslint.configs.recommended,
    {
        plugins: { "react-hooks": reactHooks, "react-refresh": reactRefresh },
        rules: {
            ...reactHooks.configs.recommended.rules,
            "react-refresh/only-export-components": "warn",
        },
    },
);
```

---

## 6. Environment Variables

### Backend env files

- `.env` — shared defaults (committed, no secrets)
- `.env.dev` — dev overrides (committed or gitignored per team preference)
- `.env.prod` — production secrets (gitignored, injected at deploy time)

### Standard backend vars

```
NODE_ENV=dev
PORT=5000
CLIENT_URL=http://localhost:5173
DATABASE_URL=postgres://user:pass@localhost:5432/dbname
REDIS_URL=redis://localhost:6379
JWT_SECRET=change-me
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=http://localhost:5000/auth/google/callback
```

### Frontend env (Vite)

```
VITE_SERVER_URL=http://localhost:5000
```

---

## 7. Docker Setup

### `docker-compose.dev.yml` structure

```yaml
services:
    postgres:
        image: postgres:16
        environment:
            { POSTGRES_DB: appdb, POSTGRES_USER: user, POSTGRES_PASSWORD: pass }
        ports: ["5432:5432"]

    redis:
        image: redis:7-alpine
        ports: ["6379:6379"]

    server:
        build: { context: ., dockerfile: apps/server/Dockerfile }
        ports: ["5000:5000"]
        environment:
            NODE_ENV: dev
            DATABASE_URL: postgres://user:pass@postgres:5432/appdb
            REDIS_URL: redis://redis:6379
        volumes:
            - ./apps/server/src:/app/apps/server/src
            - ./packages/shared/src:/app/packages/shared/src
        depends_on: [postgres, redis]

    web:
        build: { context: ., dockerfile: apps/web/Dockerfile }
        ports: ["5173:5173"]
        environment:
            VITE_SERVER_URL: http://localhost:5000
        volumes:
            - ./apps/web/src:/app/apps/web/src
        depends_on: [server]
```

### Backend `Dockerfile` (dev, hot-reload)

```dockerfile
FROM node:20-alpine
RUN corepack enable && corepack prepare pnpm@10.33.0 --activate
WORKDIR /app
COPY pnpm-workspace.yaml package.json pnpm-lock.yaml ./
COPY packages/shared/package.json ./packages/shared/
COPY apps/server/package.json ./apps/server/
RUN pnpm install --frozen-lockfile
COPY . .
CMD ["sh", "-c", "pnpm --filter @<project>/shared dev & pnpm --filter server start:dev"]
```

### Backend `Dockerfile.prod` (multi-stage)

```dockerfile
FROM node:20-alpine AS builder
RUN corepack enable && corepack prepare pnpm@10.33.0 --activate
WORKDIR /app
COPY pnpm-workspace.yaml package.json pnpm-lock.yaml ./
COPY packages/shared/package.json ./packages/shared/
COPY apps/server/package.json ./apps/server/
RUN pnpm install --frozen-lockfile
COPY . .
RUN pnpm --filter @<project>/shared build && pnpm --filter server build

FROM node:20-alpine AS runner
RUN corepack enable && corepack prepare pnpm@10.33.0 --activate
WORKDIR /app
COPY --from=builder /app .
RUN pnpm install --frozen-lockfile --prod
CMD ["node", "apps/server/dist/main"]
```

---

## 8. Database Migrations (Kysely)

- Files in `apps/server/src/migrations/`
- Named `0001_init.ts`, `0002_add_workspaces.ts`, … (zero-padded, sequential)
- Each file exports `up` and `down` functions

```typescript
// migrations/0001_init.ts
import { Kysely, sql } from "kysely";

export async function up(db: Kysely<any>): Promise<void> {
    await db.schema
        .createTable("users")
        .addColumn("id", "serial", (col) => col.primaryKey())
        .addColumn("email", "varchar(255)", (col) => col.notNull().unique())
        .addColumn("name", "varchar(255)", (col) => col.notNull())
        .addColumn("created_at", "timestamptz", (col) =>
            col.notNull().defaultTo(sql`now()`),
        )
        .execute();
}

export async function down(db: Kysely<any>): Promise<void> {
    await db.schema.dropTable("users").execute();
}
```

---

## 9. Auth Flow

1. Frontend redirects to `/auth/google` → server returns Google OAuth URL
2. Google redirects to `GOOGLE_REDIRECT_URI` with `code`
3. Server exchanges `code` for Google ID token, decodes it
4. Server upserts user in DB, signs JWT (`{ userId }`, HS256, 7d TTL)
5. JWT set as `httpOnly; SameSite=Strict` cookie
6. All protected endpoints use `AuthGuard` which reads and verifies the cookie

---

## 10. Access Control Pattern

When a resource has per-user access levels:

1. Define levels as ordered enum in shared: `owner > admin > editor > viewer > noAccess`
2. Store `ACCESS_RANK` map and `hasAccess(userLevel, required)` helper in shared
3. Service resolves access before any data operation
4. Throw `ForbiddenException` if access insufficient

```typescript
const access = await this.accessService.resolveAccess(resourceId, userId);
if (!hasAccess(access, "viewer")) throw new ForbiddenException();
```

---

## 11. Rate Limiting

Use `@nestjs/throttler` with Redis storage for distributed rate limiting.

```typescript
// app.module.ts
ThrottlerModule.forRoot({
  throttlers: [{ ttl: 60000, limit: 10 }],
  storage: new ThrottlerStorageRedisService(redisClient),
}),
```

Apply `@UseGuards(ThrottlerGuard)` to controllers or specific routes as needed.

---

## 12. Conventions & Rules

### General

- All data crossing a service boundary (HTTP, WebSocket, user input) must be validated with a Zod schema from `@<project>/shared`.
- Never use `any` unless unavoidable (e.g., NestJS `req` object userId injection); if you do, add a type assertion immediately.
- No business logic in controllers — delegate entirely to services.
- No DB queries outside of service files.

### Backend

- One NestJS module per feature domain.
- Services are `@Injectable()`. Never instantiate services manually.
- Migrations run automatically on server start via `DatabaseService.onModuleInit`.
- All async route handlers must be `async` and return the DTO type.

### Frontend

- One hook per concern (`useAuth`, `useSocket`, `useEditor`, `useLibrary` — not one god hook).
- Atoms hold only primitive state; derived state uses `atom(get => ...)`.
- `App.tsx` is route tree only — no hooks, no logic.
- Page components delegate all logic to hooks; render only based on returned state.
- All API calls go through the `apiClient` Axios instance (never raw `fetch`).
- All socket emits/receives use event name constants from `SOCKET_EVENTS` (never bare strings).

### Shared

- Every HTTP request and response shape has a Zod schema AND an inferred TypeScript type in `packages/shared`.
- Every socket event payload has a Zod schema in `packages/shared`.
- Domain enums with ordering have a `RANK` map and a `has<X>` helper function.

---

## 13. Nginx (Production)

- Blue-green deployment: two upstream groups (`_blue`, `_green`) on different port ranges
- `ip_hash` for sticky sessions (required for Socket.io)
- WebSocket upgrade headers on location blocks handling socket connections
- Proxy buffer sizes increased for large binary payloads
- SPA frontend: `try_files $uri /index.html`
- HTTPS via Let's Encrypt / Certbot

---

## 14. Scaffolding Checklist (New Project)

When starting a new project:

- [ ] Init monorepo with `pnpm-workspace.yaml`
- [ ] Create `packages/shared` with Zod, dual CJS+ESM build, `index.ts` re-exports
- [ ] Create `apps/server` with NestJS CLI, install all listed deps
- [ ] Create `apps/web` with Vite React TS template, install all listed deps
- [ ] Add workspace reference `"@<project>/shared": "workspace:*"` to both apps
- [ ] Set up `DatabaseService` with Kysely + migrations runner
- [ ] Create first migration `0001_init.ts` with users table
- [ ] Set up `RedisService` with ioredis
- [ ] Set up `AuthModule` (Google OAuth + JWT + AuthGuard)
- [ ] Set up `ZodHttpValidationPipe`
- [ ] Set up `httpOK` helper and global exception filter
- [ ] Set up Jotai `authAtom` and `useAuth` hydration hook
- [ ] Set up Axios `apiClient` and Socket.io singleton in `lib/`
- [ ] Set up `Page` auth wrapper component
- [ ] Configure ESLint (flat config) and Prettier for both apps
- [ ] Create `docker-compose.dev.yml` with postgres, redis, server, web
- [ ] Create `.env.example` documenting all required vars
