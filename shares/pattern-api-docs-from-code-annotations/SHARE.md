---
title: "Auto-Generating API Documentation from Code Annotations"
slug: "pattern-api-docs-from-code-annotations"
solution_type: pattern
tags:
  - api-docs
  - codegen
  - jsdoc
  - typedoc
created: "2026-02-08"
---

## Problem

Hand-written API documentation drifts from the actual implementation because there is no automated link between code changes and the published reference docs.

## Solution

Generate API documentation directly from code annotations (JSDoc, TypeDoc, or OpenAPI decorators), integrate generation into CI, and publish alongside hand-written guides so reference docs are always in sync with the source code.

### 1. TypeDoc Setup for TypeScript Libraries

Install and configure TypeDoc to generate reference documentation from TypeScript source:

```bash
npm install -D typedoc typedoc-plugin-markdown
```

Create `typedoc.json`:

```json
{
  "entryPoints": ["src/index.ts"],
  "entryPointStrategy": "expand",
  "out": "docs/reference/api",
  "plugin": ["typedoc-plugin-markdown"],
  "outputFileStrategy": "members",
  "readme": "none",
  "excludePrivate": true,
  "excludeProtected": true,
  "excludeInternal": true,
  "categorizeByGroup": true,
  "sort": ["source-order"],
  "kindSortOrder": [
    "Function",
    "Class",
    "Interface",
    "TypeAlias",
    "Variable",
    "Enum"
  ],
  "hidePageHeader": false,
  "hidePageTitle": false,
  "useCodeBlocks": true
}
```

### 2. Well-Annotated Source Code

Write thorough JSDoc/TSDoc comments that TypeDoc will extract:

```typescript
/**
 * Client for interacting with the Acme authentication API.
 *
 * @example
 * ```typescript
 * const auth = new AuthClient({
 *   apiKey: process.env.ACME_API_KEY,
 *   baseUrl: 'https://api.acme.com',
 * });
 *
 * const session = await auth.createSession({
 *   email: 'user@example.com',
 *   password: 'securepassword',
 * });
 * ```
 *
 * @remarks
 * All methods throw {@link AuthError} on authentication failures
 * and {@link NetworkError} on connectivity issues.
 */
export class AuthClient {
  /**
   * Creates a new authenticated session.
   *
   * @param credentials - The user's login credentials.
   * @returns A session object containing the access token and expiry.
   * @throws {@link AuthError} if credentials are invalid.
   * @throws {@link RateLimitError} if too many attempts in the last 15 minutes.
   *
   * @example
   * ```typescript
   * const session = await auth.createSession({
   *   email: 'user@example.com',
   *   password: 'securepassword',
   * });
   * console.log(session.accessToken);
   * // => "eyJhbGciOiJIUzI1NiIs..."
   * ```
   */
  async createSession(credentials: LoginCredentials): Promise<Session> {
    // implementation
  }

  /**
   * Refreshes an existing session using a refresh token.
   *
   * @param refreshToken - The refresh token from the original session.
   * @param options - Optional configuration for the refresh.
   * @param options.extendExpiry - If true, extends the session beyond the
   *   default 24-hour window. Defaults to `false`.
   * @returns A new session with a fresh access token.
   *
   * @see {@link createSession} for initial authentication.
   */
  async refreshSession(
    refreshToken: string,
    options?: RefreshOptions,
  ): Promise<Session> {
    // implementation
  }
}

/**
 * Configuration options for the AuthClient.
 */
export interface AuthClientConfig {
  /** The API key for authentication. Obtain from the dashboard. */
  apiKey: string;

  /** Base URL of the Acme API. Defaults to `"https://api.acme.com"`. */
  baseUrl?: string;

  /** Request timeout in milliseconds. Defaults to `30000`. */
  timeout?: number;

  /** Maximum number of retry attempts for failed requests. Defaults to `3`. */
  maxRetries?: number;
}
```

### 3. OpenAPI Spec Generation from Express/Fastify Routes

For REST APIs, generate an OpenAPI spec from code annotations using `swagger-jsdoc`:

```bash
npm install -D swagger-jsdoc
```

Annotate route handlers:

```typescript
/**
 * @openapi
 * /api/v1/users:
 *   get:
 *     summary: List all users
 *     description: Returns a paginated list of users in the organization.
 *     tags:
 *       - Users
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           default: 1
 *         description: Page number for pagination
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           default: 20
 *           maximum: 100
 *         description: Number of results per page
 *       - in: query
 *         name: role
 *         schema:
 *           type: string
 *           enum: [admin, member, viewer]
 *         description: Filter users by role
 *     responses:
 *       200:
 *         description: A paginated list of users
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 data:
 *                   type: array
 *                   items:
 *                     $ref: '#/components/schemas/User'
 *                 pagination:
 *                   $ref: '#/components/schemas/Pagination'
 *       401:
 *         description: Unauthorized - invalid or missing API key
 */
app.get('/api/v1/users', authenticate, listUsersHandler);
```

Generate the spec with a script (`scripts/generate-openapi.ts`):

```typescript
import swaggerJsdoc from 'swagger-jsdoc';
import { writeFileSync } from 'fs';

const spec = swaggerJsdoc({
  definition: {
    openapi: '3.1.0',
    info: {
      title: 'Acme API',
      version: '1.0.0',
      description: 'REST API for the Acme platform',
    },
    servers: [
      { url: 'https://api.acme.com', description: 'Production' },
      { url: 'http://localhost:3000', description: 'Local development' },
    ],
    components: {
      securitySchemes: {
        apiKey: {
          type: 'apiKey',
          in: 'header',
          name: 'X-API-Key',
        },
      },
    },
    security: [{ apiKey: [] }],
  },
  apis: ['./src/routes/**/*.ts'],
});

writeFileSync('docs/reference/openapi.json', JSON.stringify(spec, null, 2));
console.log('OpenAPI spec written to docs/reference/openapi.json');
```

### 4. Zod Schema to OpenAPI (Modern Approach)

If you use Zod for runtime validation, generate OpenAPI directly from schemas with `zod-to-openapi`:

```typescript
import { z } from 'zod';
import { extendZodWithOpenApi, OpenApiGeneratorV31 } from '@asteasolutions/zod-to-openapi';

extendZodWithOpenApi(z);

// Define schemas once, use for validation AND documentation
export const UserSchema = z.object({
  id: z.string().uuid().openapi({ description: 'Unique user identifier' }),
  email: z.string().email().openapi({ description: 'User email address' }),
  name: z.string().min(1).max(100).openapi({ description: 'Display name' }),
  role: z.enum(['admin', 'member', 'viewer']).openapi({ description: 'User role' }),
  createdAt: z.string().datetime().openapi({ description: 'ISO 8601 creation timestamp' }),
}).openapi('User');

export const CreateUserSchema = UserSchema.omit({ id: true, createdAt: true }).openapi('CreateUser');

// Register routes
registry.registerPath({
  method: 'post',
  path: '/api/v1/users',
  summary: 'Create a new user',
  tags: ['Users'],
  request: { body: { content: { 'application/json': { schema: CreateUserSchema } } } },
  responses: {
    201: { description: 'User created', content: { 'application/json': { schema: UserSchema } } },
    400: { description: 'Validation error' },
  },
});

// Generate spec
const generator = new OpenApiGeneratorV31(registry.definitions);
const spec = generator.generateDocument({ /* info, servers, etc. */ });
```

### 5. CI Integration

Add doc generation to `.github/workflows/docs.yml`:

```yaml
name: Generate and Deploy API Docs

on:
  push:
    branches: [main]
    paths:
      - "src/**"
      - "typedoc.json"
      - "docs/**"

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci

      # Generate TypeDoc reference
      - name: Generate TypeDoc
        run: npx typedoc

      # Generate OpenAPI spec
      - name: Generate OpenAPI spec
        run: npx tsx scripts/generate-openapi.ts

      # Validate OpenAPI spec
      - name: Validate OpenAPI
        run: npx @redocly/cli lint docs/reference/openapi.json

      # Build the full docs site (includes generated reference)
      - name: Build docs
        run: npm run docs:build

      # Fail if generated docs differ from committed (optional safety check)
      - name: Check for uncommitted generated docs
        run: |
          git diff --exit-code docs/reference/ || {
            echo "ERROR: Generated API docs are out of date. Run 'npm run docs:generate' and commit."
            exit 1
          }
```

### 6. Package Scripts

```json
{
  "scripts": {
    "docs:typedoc": "typedoc",
    "docs:openapi": "tsx scripts/generate-openapi.ts",
    "docs:generate": "npm run docs:typedoc && npm run docs:openapi",
    "docs:build": "npm run docs:generate && vitepress build docs",
    "docs:dev": "npm run docs:generate && vitepress dev docs",
    "docs:validate": "redocly lint docs/reference/openapi.json"
  }
}
```

## Why It Works

When API documentation is generated from the same source code that runs in production, it is structurally impossible for the two to drift apart. TypeDoc extracts type information and JSDoc comments that already exist for IDE support, so the marginal effort of good reference docs is just writing better comments (which benefits all developers, not just doc readers). OpenAPI generation from route annotations or Zod schemas means the spec stays in sync with validation logic. The CI check acts as a guardrail -- if someone changes a function signature without updating the JSDoc, the build catches it.

## Context

This pattern covers TypeScript projects but the principle applies to any language with annotation-based doc generation: Javadoc for Java, Sphinx/docstrings for Python, rustdoc for Rust, GoDoc for Go. The TypeDoc markdown plugin is specifically chosen to output markdown files that integrate into static site generators like VitePress or Docusaurus, rather than TypeDoc's default HTML output. For OpenAPI, the Zod-based approach is recommended for new projects because it eliminates the duplication between validation schemas and API documentation.
