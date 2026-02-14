# API Specification
## Personal Time OS - Goal Achievement MVP

**Version:** 1.0  
**Created:** February 15, 2026  
**Purpose:** Complete REST API specification for MVP implementation with Claude Code  
**Base URL:** `https://goalos.app/api/v1`

---

## Executive Summary

This document defines the complete REST API for the Goal Achievement MVP, a web-based platform that transforms CliftonStrengths assessments into personalized goal strategies. The API follows RESTful principles with resource-oriented endpoints, standardized error responses, and async processing patterns optimized for the ~130 second AI generation workflow.

**Key API Characteristics:**
- **Versioned from day one:** `/api/v1/` prefix for future flexibility
- **Supabase-native authentication:** Session cookie validation, no custom JWT handling
- **Async-first processing:** Submit jobs, poll for status (no blocking operations)
- **Direct file uploads:** Signed URLs for client → Supabase Storage PDF uploads
- **Standardized errors:** Consistent error format with machine-readable codes
- **Rate limit enforcement:** 2 analyses per user per day, checked pre-flight and on submit
- **Auto-save drafts:** Debounced state persistence from Step 2 onward

**Technology Stack:**
- **Runtime:** Node.js 18+ with TypeScript
- **Hosting:** Vercel (Fluid Compute, 300s timeout)
- **Database:** Supabase (PostgreSQL + Auth + Storage)
- **Authentication:** Supabase Auth (Google OAuth)
- **File Storage:** Supabase Storage (signed URLs)

**Core Resources:**
1. **Users** (`/users`) - User profiles and rate limit status
2. **Analyses** (`/analyses`) - Core analysis lifecycle management
3. **Goals** (`/analyses/:id/goals`) - Goal creation and updates (nested under analyses)

**Request Flow Overview:**
```
1. Google OAuth → Supabase session → Cookie set
2. Upload PDF → Signed URL → Direct to Supabase Storage → Trigger parsing
3. Set goals → Auto-save drafts → Submit for processing
4. Poll status → Download PDF → Join community
```

---

## API Architecture Overview

### Versioning Strategy

**Pattern:** `/api/v1/` prefix from day one

All endpoints follow this structure:
```
https://goalos.app/api/v1/{resource}
```

**Future versioning:**
- Breaking changes trigger new version: `/api/v2/`
- Maintain old versions for 6 months minimum
- Deprecation warnings via `Deprecated` response header

**Why version now:**
- Zero overhead with code generation
- Prevents breaking changes for early adopters
- Professional API standard

---

### Authentication & Authorization

**Pattern:** Supabase session cookie validation (no manual JWT handling)

**Authentication Flow:**

1. **User login (frontend):**
```typescript
// Frontend uses Supabase client
const { data, error } = await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: {
    redirectTo: 'https://goalos.app/assessment/step-1'
  }
});
```

2. **Session management:**
- Supabase SDK automatically manages session cookies
- Cookie name: `sb-access-token` and `sb-refresh-token`
- HTTP-only, secure, SameSite=Lax

3. **API middleware validation:**
```typescript
// API endpoint middleware (conceptual)
async function authenticateRequest(req, res, next) {
  const supabase = createSupabaseClient(req);
  const { data: { user }, error } = await supabase.auth.getUser();
  
  if (error || !user) {
    return res.status(401).json({
      error: {
        code: 'UNAUTHORIZED',
        message: 'Authentication required',
        statusCode: 401
      }
    });
  }
  
  req.user = user; // Attach user to request
  next();
}
```

4. **User identification:**
- Extract `user.id` (UUID) from validated session
- Use for database queries (RLS policies handle row-level security)

**Authorization:**
- All endpoints require authentication (except health checks)
- Row Level Security (RLS) policies enforce data access at database level
- No additional authorization layer needed in MVP

**Session expiration:**
- Supabase default: 1 hour access token, 30 day refresh token
- Frontend auto-refreshes using SDK
- API returns 401 if session expired → frontend redirects to login

---

### Error Response Format

**Standardized Error Structure:**

```typescript
interface APIError {
  error: {
    code: string;           // Machine-readable error code
    message: string;        // Human-readable message
    statusCode: number;     // HTTP status code (mirrored in response status)
    details?: Record<string, any>;  // Optional contextual data
  }
}
```

**Example Error Responses:**

```json
// 401 Unauthorized
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Authentication required. Please log in.",
    "statusCode": 401
  }
}

// 429 Rate Limit Exceeded
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "You've used your 2 analyses today. Next analysis available in 4 hours.",
    "statusCode": 429,
    "details": {
      "resetAt": "2026-02-15T14:30:00Z",
      "current": 2,
      "limit": 2
    }
  }
}

// 400 Validation Error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid goal configuration",
    "statusCode": 400,
    "details": {
      "field": "priority",
      "constraint": "Only 1 goal can be high priority"
    }
  }
}

// 500 Internal Server Error
{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "An unexpected error occurred. Please try again.",
    "statusCode": 500,
    "details": {
      "requestId": "req_abc123"
    }
  }
}
```

**Complete Error Code Catalog:** See "Error Codes Reference" section below.

---

### Rate Limiting

**Limit:** 2 analyses per user per 24-hour rolling window

**Implementation Strategy:** Defense in depth (check on frontend AND backend)

**1. Pre-flight Check (Informational):**

Frontend calls `GET /api/v1/users/me/rate-limit-status` on Step 2 load to show/hide UI elements.

**2. Enforcement (Canonical):**

Backend checks rate limit on `POST /api/v1/analyses/:id/submit` before starting expensive processing.

**Rate Limit Calculation:**
```sql
-- Count analyses submitted in last 24 hours
SELECT COUNT(*) 
FROM analyses 
WHERE user_id = $1 
  AND submitted_at > NOW() - INTERVAL '24 hours'
  AND status IN ('processing', 'complete', 'partial_failure');
```

**Response Headers (on all authenticated requests):**
```
X-RateLimit-Limit: 2
X-RateLimit-Remaining: 1
X-RateLimit-Reset: 1708012800 (Unix timestamp)
```

**429 Response Details:**
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "You've used your 2 analyses today. Next slot available in 3 hours 45 minutes.",
    "statusCode": 429,
    "details": {
      "resetAt": "2026-02-15T14:30:00Z",
      "current": 2,
      "limit": 2,
      "resetInSeconds": 13500
    }
  }
}
```

---

### CORS Configuration

**Allowed Origins (Environment-based):**

```typescript
const allowedOrigins = [
  process.env.VERCEL_URL,           // Production: goalos.app
  process.env.VERCEL_BRANCH_URL,    // Preview deployments
  'http://localhost:3000',           // Local dev (Next.js)
  'http://localhost:5173',           // Local dev (Vite)
];

// CORS middleware config
{
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,  // Allow cookies
  methods: ['GET', 'POST', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  exposedHeaders: ['X-RateLimit-Limit', 'X-RateLimit-Remaining', 'X-RateLimit-Reset'],
  maxAge: 86400  // 24 hours
}
```

**Why CORS in Vercel monorepo:**
- Even same-domain deployments benefit from explicit CORS for security
- Supports local development workflow
- Ready for future mobile apps or external integrations

---

## Complete Endpoint Reference

### Overview: All Endpoints

| Method | Endpoint | Purpose | Auth Required |
|--------|----------|---------|---------------|
| GET | `/api/v1/health` | Health check | No |
| GET | `/api/v1/users/me` | Get current user profile | Yes |
| GET | `/api/v1/users/me/rate-limit-status` | Check rate limit status | Yes |
| GET | `/api/v1/analyses` | List user's analyses | Yes |
| GET | `/api/v1/analyses/:id` | Get single analysis | Yes |
| POST | `/api/v1/analyses` | Create new analysis (draft) | Yes |
| PATCH | `/api/v1/analyses/:id` | Update analysis (auto-save) | Yes |
| POST | `/api/v1/analyses/:id/upload-url` | Get signed URL for PDF upload | Yes |
| POST | `/api/v1/analyses/:id/trigger-parsing` | Start CliftonStrengths parsing | Yes |
| POST | `/api/v1/analyses/:id/submit` | Submit analysis for AI processing | Yes |
| POST | `/api/v1/analyses/:id/goals` | Add goal to analysis | Yes |
| PATCH | `/api/v1/analyses/:id/goals/:goalId` | Update goal | Yes |
| DELETE | `/api/v1/analyses/:id/goals/:goalId` | Remove goal | Yes |

---

### Endpoint 1: Health Check

**Purpose:** Verify API availability (used by monitoring tools)

**Endpoint:** `GET /api/v1/health`

**Authentication:** None required

**Request:**
```http
GET /api/v1/health HTTP/1.1
Host: goalos.app
```

**Response (200 OK):**
```json
{
  "status": "ok",
  "timestamp": "2026-02-15T10:30:00Z",
  "version": "1.0.0"
}
```

**Error Response:** None (returns 503 if API down)

---

### Endpoint 2: Get Current User Profile

**Purpose:** Fetch authenticated user's profile and metadata

**Endpoint:** `GET /api/v1/users/me`

**Authentication:** Required

**Request:**
```http
GET /api/v1/users/me HTTP/1.1
Host: goalos.app
Cookie: sb-access-token=...; sb-refresh-token=...
```

**Response (200 OK):**
```json
{
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "fullName": "Jane Doe",
    "avatarUrl": "https://lh3.googleusercontent.com/...",
    "createdAt": "2026-02-10T08:00:00Z",
    "lastLoginAt": "2026-02-15T10:00:00Z",
    "totalAnalysesCount": 3
  }
}
```

**Response Schema:**
```typescript
interface UserProfileResponse {
  data: {
    id: string;                  // UUID
    email: string;
    fullName: string | null;
    avatarUrl: string | null;
    createdAt: string;           // ISO 8601
    lastLoginAt: string | null;  // ISO 8601
    totalAnalysesCount: number;
  }
}
```

**Error Responses:**
- `401 UNAUTHORIZED` - No valid session

---

### Endpoint 3: Get Rate Limit Status

**Purpose:** Check how many analyses remaining for current user

**Endpoint:** `GET /api/v1/users/me/rate-limit-status`

**Authentication:** Required

**Request:**
```http
GET /api/v1/users/me/rate-limit-status HTTP/1.1
Host: goalos.app
Cookie: sb-access-token=...; sb-refresh-token=...
```

**Response (200 OK):**
```json
{
  "data": {
    "limit": 2,
    "remaining": 1,
    "resetAt": "2026-02-15T14:30:00Z",
    "resetInSeconds": 13500
  }
}
```

**Response Schema:**
```typescript
interface RateLimitStatusResponse {
  data: {
    limit: number;         // Max analyses per 24h (always 2 in MVP)
    remaining: number;     // Analyses available now (0-2)
    resetAt: string;       // ISO 8601 - when oldest analysis exits window
    resetInSeconds: number; // Seconds until reset (for countdown timers)
  }
}
```

**Logic:**
```typescript
// Pseudo-code for rate limit calculation
const submittedInLast24h = await db.query(`
  SELECT id, submitted_at
  FROM analyses
  WHERE user_id = $1
    AND submitted_at > NOW() - INTERVAL '24 hours'
    AND status IN ('processing', 'complete', 'partial_failure')
  ORDER BY submitted_at ASC
`);

const remaining = Math.max(0, 2 - submittedInLast24h.length);
const oldestSubmission = submittedInLast24h[0];
const resetAt = oldestSubmission 
  ? new Date(oldestSubmission.submitted_at.getTime() + 24 * 60 * 60 * 1000)
  : new Date();
```

**Error Responses:**
- `401 UNAUTHORIZED` - No valid session

---

### Endpoint 4: List User's Analyses

**Purpose:** Retrieve all analyses for authenticated user (with optional filtering)

**Endpoint:** `GET /api/v1/analyses`

**Authentication:** Required

**Query Parameters:**
- `limit` (optional): Max results to return (default: 50, max: 100)
- `status` (optional): Filter by status (`draft`, `processing`, `complete`, `failed`, `partial_failure`)

**Request:**
```http
GET /api/v1/analyses?limit=20&status=complete HTTP/1.1
Host: goalos.app
Cookie: sb-access-token=...; sb-refresh-token=...
```

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "userId": "550e8400-e29b-41d4-a716-446655440000",
      "status": "complete",
      "createdAt": "2026-02-14T09:00:00Z",
      "submittedAt": "2026-02-14T09:15:00Z",
      "completedAt": "2026-02-14T09:17:23Z",
      "outputPdfPath": "outputs/550e8400.../a1b2c3d4.../recommendations.pdf",
      "goalCount": 3,
      "totalCostUsd": 0.048,
      "totalDurationMs": 128450
    }
  ],
  "meta": {
    "total": 3,
    "returned": 3,
    "limit": 50
  }
}
```

**Response Schema:**
```typescript
interface AnalysesListResponse {
  data: AnalysisSummary[];
  meta: {
    total: number;      // Total analyses for user
    returned: number;   // Count in current response
    limit: number;      // Applied limit
  }
}

interface AnalysisSummary {
  id: string;
  userId: string;
  status: 'draft' | 'parsing' | 'processing' | 'complete' | 'failed' | 'partial_failure';
  createdAt: string;
  submittedAt: string | null;
  completedAt: string | null;
  outputPdfPath: string | null;
  goalCount: number;
  totalCostUsd: number | null;
  totalDurationMs: number | null;
}
```

**Sorting:** Always returns newest first (`ORDER BY created_at DESC`)

**Error Responses:**
- `401 UNAUTHORIZED` - No valid session

---

### Endpoint 5: Get Single Analysis

**Purpose:** Retrieve complete details for one analysis (including goals, CliftonStrengths data, status)

**Endpoint:** `GET /api/v1/analyses/:id`

**Authentication:** Required

**Path Parameters:**
- `id` (required): Analysis UUID

**Query Parameters:**
- `fields` (optional): Comma-separated list for lightweight queries
  - Example: `?fields=status,updatedAt` (returns only those fields)
  - Useful for status polling to reduce bandwidth

**Request:**
```http
GET /api/v1/analyses/a1b2c3d4-e5f6-7890-abcd-ef1234567890 HTTP/1.1
Host: goalos.app
Cookie: sb-access-token=...; sb-refresh-token=...
```

**Response (200 OK - Full):**
```json
{
  "data": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "userId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "complete",
    "createdAt": "2026-02-14T09:00:00Z",
    "submittedAt": "2026-02-14T09:15:00Z",
    "completedAt": "2026-02-14T09:17:23Z",
    "parsingCompletedAt": "2026-02-14T09:03:45Z",
    "cliftonTop10": [
      { "name": "Achiever", "rank": 1 },
      { "name": "Learner", "rank": 2 },
      { "name": "Strategic", "rank": 3 }
    ],
    "cliftonBottom5": [
      { "name": "Command", "rank": 30 },
      { "name": "Woo", "rank": 31 }
    ],
    "cliftonDominantDomain": "Strategic Thinking",
    "weeklyTimeInvestment": {
      "work_career": 40,
      "health_fitness": 5,
      "relationships_family": 15,
      "learning_growth": 8,
      "creativity_hobbies": 3,
      "rest_recovery": 56,
      "community_service": 2,
      "admin_maintenance": 8,
      "total": 137
    },
    "questionnaireResponses": {
      "biggest_challenge": "Finding time for learning while working full-time",
      "past_successes": "Completed online certification while managing team"
    },
    "goals": [
      {
        "id": "goal-uuid-1",
        "analysisId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "description": "Run a 5K in under 30 minutes",
        "priority": "high",
        "categories": ["health_fitness"],
        "deadline": "2026-05-01T00:00:00Z",
        "hardness": 3,
        "createdAt": "2026-02-14T09:10:00Z",
        "updatedAt": "2026-02-14T09:12:00Z"
      }
    ],
    "outputPdfPath": "outputs/550e8400.../a1b2c3d4.../recommendations.pdf",
    "outputPdfUrl": "https://supabase.co/storage/v1/object/sign/...",
    "outputPdfSize": 2456789,
    "outputPdfGeneratedAt": "2026-02-14T09:17:20Z",
    "totalTokensUsed": 45230,
    "totalCostUsd": 0.048,
    "totalDurationMs": 128450,
    "errorMessage": null,
    "errorDetails": null,
    "updatedAt": "2026-02-14T09:17:23Z"
  }
}
```

**Response (200 OK - Lightweight with `?fields=status,updatedAt`):**
```json
{
  "data": {
    "status": "processing",
    "updatedAt": "2026-02-15T10:30:15Z"
  }
}
```

**Response Schema:**
```typescript
interface AnalysisDetailResponse {
  data: AnalysisDetail;
}

interface AnalysisDetail {
  id: string;
  userId: string;
  status: 'draft' | 'parsing' | 'processing' | 'complete' | 'failed' | 'partial_failure';
  createdAt: string;
  submittedAt: string | null;
  completedAt: string | null;
  parsingCompletedAt: string | null;
  
  // CliftonStrengths data (null until parsing complete)
  cliftonTop10: CliftonTheme[] | null;
  cliftonBottom5: CliftonTheme[] | null;
  cliftonDominantDomain: string | null;
  
  // User input data
  weeklyTimeInvestment: Record<string, number> | null;
  questionnaireResponses: Record<string, string> | null;
  
  // Goals (nested)
  goals: Goal[];
  
  // Output
  outputPdfPath: string | null;
  outputPdfUrl: string | null;  // Signed URL for download (expires in 1 hour)
  outputPdfSize: number | null;
  outputPdfGeneratedAt: string | null;
  
  // Metrics
  totalTokensUsed: number | null;
  totalCostUsd: number | null;
  totalDurationMs: number | null;
  
  // Errors
  errorMessage: string | null;
  errorDetails: any | null;
  
  updatedAt: string;
}

interface CliftonTheme {
  name: string;
  rank: number;
  description?: string;
}

interface Goal {
  id: string;
  analysisId: string;
  description: string;
  priority: 'high' | 'medium' | 'low';
  categories: string[];  // 1-3 categories
  deadline: string;      // ISO 8601
  hardness: 1 | 2 | 3 | 4 | 5;
  createdAt: string;
  updatedAt: string;
}
```

**ETag Support (for efficient polling):**

Response includes `ETag` header:
```http
HTTP/1.1 200 OK
ETag: "a1b2c3d4-2026-02-15T10:30:15Z"
Content-Type: application/json
```

Client can use `If-None-Match` on subsequent requests:
```http
GET /api/v1/analyses/a1b2c3d4?fields=status,updatedAt HTTP/1.1
If-None-Match: "a1b2c3d4-2026-02-15T10:30:15Z"
```

If unchanged, server returns:
```http
HTTP/1.1 304 Not Modified
ETag: "a1b2c3d4-2026-02-15T10:30:15Z"
```

**Error Responses:**
- `401 UNAUTHORIZED` - No valid session
- `404 NOT_FOUND` - Analysis doesn't exist or doesn't belong to user

---

### Endpoint 6: Create New Analysis (Draft)

**Purpose:** Initialize a new analysis in draft state (happens automatically when user lands on Step 1)

**Endpoint:** `POST /api/v1/analyses`

**Authentication:** Required

**Request:**
```http
POST /api/v1/analyses HTTP/1.1
Host: goalos.app
Cookie: sb-access-token=...; sb-refresh-token=...
Content-Type: application/json

{}
```

**Request Body:** Empty object (all fields optional initially)

**Response (201 Created):**
```json
{
  "data": {
    "id": "new-analysis-uuid",
    "userId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "draft",
    "createdAt": "2026-02-15T10:30:00Z",
    "goals": [],
    "updatedAt": "2026-02-15T10:30:00Z"
  }
}
```

**Response Schema:**
```typescript
interface CreateAnalysisResponse {
  data: {
    id: string;
    userId: string;
    status: 'draft';
    createdAt: string;
    goals: Goal[];
    updatedAt: string;
  }
}
```

**Database Action:**
```sql
INSERT INTO analyses (user_id, status)
VALUES ($1, 'draft')
RETURNING *;
```

**Error Responses:**
- `401 UNAUTHORIZED` - No valid session

---

### Endpoint 7: Update Analysis (Auto-Save)

**Purpose:** Update draft analysis fields (weekly time, questionnaire, goals metadata)

**Endpoint:** `PATCH /api/v1/analyses/:id`

**Authentication:** Required

**Path Parameters:**
- `id` (required): Analysis UUID

**Request:**
```http
PATCH /api/v1/analyses/a1b2c3d4-e5f6-7890-abcd-ef1234567890 HTTP/1.1
Host: goalos.app
Cookie: sb-access-token=...; sb-refresh-token=...
Content-Type: application/json

{
  "weeklyTimeInvestment": {
    "work_career": 40,
    "health_fitness": 5,
    "relationships_family": 15,
    "learning_growth": 8,
    "creativity_hobbies": 3,
    "rest_recovery": 56,
    "community_service": 2,
    "admin_maintenance": 8,
    "total": 137
  },
  "questionnaireResponses": {
    "biggest_challenge": "Finding time for learning",
    "past_successes": "Completed certification"
  }
}
```

**Request Schema:**
```typescript
interface UpdateAnalysisRequest {
  weeklyTimeInvestment?: {
    work_career?: number;
    health_fitness?: number;
    relationships_family?: number;
    learning_growth?: number;
    creativity_hobbies?: number;
    rest_recovery?: number;
    community_service?: number;
    admin_maintenance?: number;
    total?: number;
  };
  questionnaireResponses?: Record<string, string>;
}
```

**Response (200 OK):**
```json
{
  "data": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "status": "draft",
    "weeklyTimeInvestment": { /* updated data */ },
    "questionnaireResponses": { /* updated data */ },
    "updatedAt": "2026-02-15T10:31:00Z"
  }
}
```

**Validation:**
- Can only update analyses in `draft` status
- Cannot update `cliftonTop10`, `cliftonBottom5`, `cliftonDominantDomain` (those come from parsing)

**Error Responses:**
- `401 UNAUTHORIZED` - No valid session
- `404 NOT_FOUND` - Analysis doesn't exist or doesn't belong to user
- `409 CONFLICT` - Analysis is not in draft status (already submitted)

---

### Endpoint 8: Get Signed Upload URL

**Purpose:** Generate temporary signed URL for client to upload CliftonStrengths PDF directly to Supabase Storage

**Endpoint:** `POST /api/v1/analyses/:id/upload-url`

**Authentication:** Required

**Path Parameters:**
- `id` (required): Analysis UUID

**Request:**
```http
POST /api/v1/analyses/a1b2c3d4/upload-url HTTP/1.1
Host: goalos.app
Cookie: sb-access-token=...; sb-refresh-token=...
Content-Type: application/json

{
  "fileName": "cliftonstrengths_report.pdf",
  "fileSize": 2456789,
  "contentType": "application/pdf"
}
```

**Request Schema:**
```typescript
interface GetUploadUrlRequest {
  fileName: string;      // Original filename
  fileSize: number;      // Bytes (validated: max 10MB = 10485760)
  contentType: string;   // Must be "application/pdf"
}
```

**Validation:**
- `fileSize` must be ≤ 10,485,760 bytes (10MB)
- `contentType` must be exactly "application/pdf"
- Analysis must be in `draft` or `parsing` status

**Response (200 OK):**
```json
{
  "data": {
    "uploadUrl": "https://supabase.co/storage/v1/object/upload/sign/uploads/550e8400.../a1b2c3d4.../cliftonstrengths.pdf?token=...",
    "filePath": "uploads/550e8400.../a1b2c3d4.../cliftonstrengths.pdf",
    "expiresAt": "2026-02-15T10:45:00Z"
  }
}
```

**Response Schema:**
```typescript
interface GetUploadUrlResponse {
  data: {
    uploadUrl: string;   // Signed URL for PUT request
    filePath: string;    // Storage path (use for trigger-parsing)
    expiresAt: string;   // URL expires in 15 minutes
  }
}
```

**File Path Pattern:**
```
uploads/{userId}/{analysisId}/{sanitizedFileName}
```

**Implementation Notes:**
```typescript
// Generate signed upload URL using Supabase Storage
const { data, error } = await supabase.storage
  .from('uploads')
  .createSignedUploadUrl(`${userId}/${analysisId}/${sanitizedFileName}`);
```

**Error Responses:**
- `401 UNAUTHORIZED` - No valid session
- `400 VALIDATION_ERROR` - File size exceeds 10MB
- `400 VALIDATION_ERROR` - Content type is not PDF
- `404 NOT_FOUND` - Analysis doesn't exist
- `409 CONFLICT` - Analysis already has PDF uploaded

---

### Endpoint 9: Trigger CliftonStrengths Parsing

**Purpose:** Start background parsing of uploaded CliftonStrengths PDF

**Endpoint:** `POST /api/v1/analyses/:id/trigger-parsing`

**Authentication:** Required

**Path Parameters:**
- `id` (required): Analysis UUID

**Request:**
```http
POST /api/v1/analyses/a1b2c3d4/trigger-parsing HTTP/1.1
Host: goalos.app
Cookie: sb-access-token=...; sb-refresh-token=...
Content-Type: application/json

{
  "filePath": "uploads/550e8400.../a1b2c3d4.../cliftonstrengths.pdf"
}
```

**Request Schema:**
```typescript
interface TriggerParsingRequest {
  filePath: string;  // Must match path returned from upload-url endpoint
}
```

**Response (202 Accepted):**
```json
{
  "data": {
    "analysisId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "status": "parsing",
    "message": "CliftonStrengths parsing started. Check status at GET /api/v1/analyses/:id"
  }
}
```

**Response Schema:**
```typescript
interface TriggerParsingResponse {
  data: {
    analysisId: string;
    status: 'parsing';
    message: string;
  }
}
```

**Implementation Flow:**

1. Validate file exists in Supabase Storage at provided `filePath`
2. Update analysis status to `parsing` in database
3. Trigger background job (Vercel serverless function or async task)
4. Background job:
   - Downloads PDF from Supabase Storage
   - Calls Gemini Flash 2.0 with parsing prompt
   - Updates database with extracted CliftonStrengths data
   - Sets status to `draft` and `parsing_completed_at` timestamp

**Database Update:**
```sql
UPDATE analyses
SET status = 'parsing'
WHERE id = $1 AND user_id = $2
RETURNING *;
```

**Error Responses:**
- `401 UNAUTHORIZED` - No valid session
- `404 NOT_FOUND` - Analysis doesn't exist
- `404 NOT_FOUND` - File not found at specified path
- `409 CONFLICT` - Parsing already in progress or completed

**Polling for Completion:**

Frontend should poll `GET /api/v1/analyses/:id?fields=status,parsingCompletedAt` every 2-3 seconds until:
- `parsingCompletedAt` is not null, OR
- `status` changes to `failed`

---

### Endpoint 10: Submit Analysis for Processing

**Purpose:** Submit analysis for AI recommendation generation (multi-agent processing)

**Endpoint:** `POST /api/v1/analyses/:id/submit`

**Authentication:** Required

**Path Parameters:**
- `id` (required): Analysis UUID

**Request:**
```http
POST /api/v1/analyses/a1b2c3d4/submit HTTP/1.1
Host: goalos.app
Cookie: sb-access-token=...; sb-refresh-token=...
Content-Type: application/json

{}
```

**Request Body:** Empty object (all data already in database from previous steps)

**Pre-Submission Validation:**

1. **Analysis must be in `draft` status**
2. **CliftonStrengths parsing must be complete** (`parsing_completed_at` not null)
3. **At least 1 goal must exist**
4. **Rate limit check:** User must have remaining analyses in 24h window
5. **Only 1 goal can be high priority** (validated at goal creation, but double-check)

**Response (202 Accepted):**
```json
{
  "data": {
    "analysisId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "status": "processing",
    "submittedAt": "2026-02-15T10:35:00Z",
    "estimatedCompletionAt": "2026-02-15T10:37:10Z",
    "message": "Analysis submitted. Processing will take approximately 2 minutes."
  }
}
```

**Response Schema:**
```typescript
interface SubmitAnalysisResponse {
  data: {
    analysisId: string;
    status: 'processing';
    submittedAt: string;
    estimatedCompletionAt: string;  // submittedAt + 130 seconds
    message: string;
  }
}
```

**Database Updates:**
```sql
-- Mark as processing
UPDATE analyses
SET 
  status = 'processing',
  submitted_at = NOW()
WHERE id = $1 AND user_id = $2
RETURNING *;

-- Increment user's total count (for historical tracking)
UPDATE users
SET total_analyses_count = total_analyses_count + 1
WHERE id = $2;
```

**Background Processing Flow:**

After returning 202 response, trigger background job that:

1. **Retrieve data** from database (analysis, goals, CliftonStrengths)
2. **Spawn Goal Agents** (1-3 parallel Claude Sonnet 4 calls based on goal count)
3. **Run Synthesizer Agent** (1 Claude Sonnet 4 call to combine outputs)
4. **Generate PDF** (using PDF template specification)
5. **Upload PDF** to Supabase Storage
6. **Update database** with results and metrics
7. **Set status** to `complete` (or `partial_failure` if some goals failed)

**Polling for Completion:**

Frontend polls `GET /api/v1/analyses/:id?fields=status,completedAt,outputPdfUrl` every 5 seconds until:
- `status` becomes `complete`, `partial_failure`, or `failed`

**Error Responses:**
- `401 UNAUTHORIZED` - No valid session
- `404 NOT_FOUND` - Analysis doesn't exist
- `400 VALIDATION_ERROR` - CliftonStrengths parsing not complete
- `400 VALIDATION_ERROR` - No goals found
- `409 CONFLICT` - Analysis already submitted
- `429 RATE_LIMIT_EXCEEDED` - User exceeded 2 analyses per 24h

---

### Endpoint 11: Create Goal

**Purpose:** Add a new goal to analysis (nested resource)

**Endpoint:** `POST /api/v1/analyses/:id/goals`

**Authentication:** Required

**Path Parameters:**
- `id` (required): Analysis UUID (parent)

**Request:**
```http
POST /api/v1/analyses/a1b2c3d4/goals HTTP/1.1
Host: goalos.app
Cookie: sb-access-token=...; sb-refresh-token=...
Content-Type: application/json

{
  "description": "Run a 5K in under 30 minutes",
  "priority": "high",
  "categories": ["health_fitness"],
  "deadline": "2026-05-01T00:00:00Z",
  "hardness": 3
}
```

**Request Schema:**
```typescript
interface CreateGoalRequest {
  description: string;              // 1-200 characters
  priority: 'high' | 'medium' | 'low';
  categories: string[];             // 1-3 categories from predefined list
  deadline: string;                 // ISO 8601 date
  hardness: 1 | 2 | 3 | 4 | 5;
}
```

**Validation:**
- `description`: 1-200 characters, required
- `priority`: Only 1 goal per analysis can be "high"
- `categories`: Must be 1-3 items from valid list
- `deadline`: Must be future date
- `hardness`: Must be 1-5
- Analysis must be in `draft` status
- Analysis cannot have more than 3 goals

**Valid Categories:**
```typescript
const VALID_CATEGORIES = [
  'work_career',
  'health_fitness',
  'relationships_family',
  'learning_growth',
  'creativity_hobbies',
  'rest_recovery',
  'community_service',
  'admin_maintenance'
];
```

**Response (201 Created):**
```json
{
  "data": {
    "id": "goal-uuid-1",
    "analysisId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "description": "Run a 5K in under 30 minutes",
    "priority": "high",
    "categories": ["health_fitness"],
    "deadline": "2026-05-01T00:00:00Z",
    "hardness": 3,
    "createdAt": "2026-02-15T10:36:00Z",
    "updatedAt": "2026-02-15T10:36:00Z"
  }
}
```

**Response Schema:**
```typescript
interface CreateGoalResponse {
  data: Goal;
}
```

**Error Responses:**
- `401 UNAUTHORIZED` - No valid session
- `404 NOT_FOUND` - Analysis doesn't exist
- `400 VALIDATION_ERROR` - Invalid field values
- `409 CONFLICT` - Analysis already has 3 goals
- `409 CONFLICT` - Analysis already has a high priority goal
- `409 CONFLICT` - Analysis not in draft status

---

### Endpoint 12: Update Goal

**Purpose:** Modify existing goal fields

**Endpoint:** `PATCH /api/v1/analyses/:analysisId/goals/:goalId`

**Authentication:** Required

**Path Parameters:**
- `analysisId` (required): Analysis UUID
- `goalId` (required): Goal UUID

**Request:**
```http
PATCH /api/v1/analyses/a1b2c3d4/goals/goal-uuid-1 HTTP/1.1
Host: goalos.app
Cookie: sb-access-token=...; sb-refresh-token=...
Content-Type: application/json

{
  "priority": "medium",
  "deadline": "2026-06-01T00:00:00Z"
}
```

**Request Schema:**
```typescript
interface UpdateGoalRequest {
  description?: string;
  priority?: 'high' | 'medium' | 'low';
  categories?: string[];
  deadline?: string;
  hardness?: 1 | 2 | 3 | 4 | 5;
}
```

**Validation:**
- Same rules as create goal
- If changing priority to "high", check no other goal is already high
- Analysis must be in `draft` status

**Response (200 OK):**
```json
{
  "data": {
    "id": "goal-uuid-1",
    "analysisId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "description": "Run a 5K in under 30 minutes",
    "priority": "medium",
    "categories": ["health_fitness"],
    "deadline": "2026-06-01T00:00:00Z",
    "hardness": 3,
    "createdAt": "2026-02-15T10:36:00Z",
    "updatedAt": "2026-02-15T10:37:00Z"
  }
}
```

**Error Responses:**
- `401 UNAUTHORIZED` - No valid session
- `404 NOT_FOUND` - Analysis or goal doesn't exist
- `400 VALIDATION_ERROR` - Invalid field values
- `409 CONFLICT` - Another goal already has high priority
- `409 CONFLICT` - Analysis not in draft status

---

### Endpoint 13: Delete Goal

**Purpose:** Remove goal from analysis

**Endpoint:** `DELETE /api/v1/analyses/:analysisId/goals/:goalId`

**Authentication:** Required

**Path Parameters:**
- `analysisId` (required): Analysis UUID
- `goalId` (required): Goal UUID

**Request:**
```http
DELETE /api/v1/analyses/a1b2c3d4/goals/goal-uuid-1 HTTP/1.1
Host: goalos.app
Cookie: sb-access-token=...; sb-refresh-token=...
```

**Validation:**
- Analysis must be in `draft` status
- Analysis must have at least 2 goals (cannot delete the last goal)

**Response (204 No Content):**
```http
HTTP/1.1 204 No Content
```

**Error Responses:**
- `401 UNAUTHORIZED` - No valid session
- `404 NOT_FOUND` - Analysis or goal doesn't exist
- `409 CONFLICT` - Cannot delete last goal (minimum 1 required)
- `409 CONFLICT` - Analysis not in draft status

---

## Error Codes Reference

Complete catalog of error codes used across all endpoints:

### 4xx Client Errors

| Code | HTTP Status | Description | Example Use Case |
|------|-------------|-------------|------------------|
| `UNAUTHORIZED` | 401 | No valid session or session expired | Missing or invalid auth cookie |
| `FORBIDDEN` | 403 | Valid session but access denied | Attempting to access another user's analysis (RLS blocks this at DB level) |
| `NOT_FOUND` | 404 | Resource doesn't exist | Analysis ID doesn't exist or doesn't belong to user |
| `VALIDATION_ERROR` | 400 | Invalid request data | File size exceeds 10MB, invalid goal priority |
| `CONFLICT` | 409 | Request conflicts with current state | Trying to submit analysis that's already processing |
| `RATE_LIMIT_EXCEEDED` | 429 | User exceeded analysis quota | Submitted 2 analyses in last 24h |
| `FILE_TOO_LARGE` | 400 | PDF exceeds 10MB limit | CliftonStrengths PDF validation |
| `INVALID_FILE_TYPE` | 400 | File is not a PDF | Uploaded image instead of PDF |
| `PARSING_FAILED` | 400 | CliftonStrengths extraction failed | AI couldn't extract themes from PDF |
| `INVALID_GOAL_CONFIG` | 400 | Goal constraints violated | More than 3 goals, invalid category |
| `HIGH_PRIORITY_CONFLICT` | 409 | Multiple high priority goals | Trying to set 2nd goal to high priority |

### 5xx Server Errors

| Code | HTTP Status | Description | Example Use Case |
|------|-------------|-------------|------------------|
| `INTERNAL_ERROR` | 500 | Unexpected server error | Unhandled exception |
| `DATABASE_ERROR` | 500 | Database operation failed | Supabase connection timeout |
| `STORAGE_ERROR` | 500 | File storage operation failed | Supabase Storage upload error |
| `AI_TIMEOUT` | 500 | AI model timed out | Claude Sonnet 4 exceeded 60s timeout |
| `AI_ERROR` | 500 | AI model returned invalid response | Malformed JSON from AI |
| `PDF_GENERATION_ERROR` | 500 | PDF rendering failed | PDF library error |

---

## Status Polling Pattern

**Recommended Frontend Pattern:**

```typescript
// Poll for analysis completion
async function pollAnalysisStatus(analysisId: string): Promise<Analysis> {
  const maxAttempts = 60;  // 60 attempts × 3s = 3 minutes max
  const pollInterval = 3000; // 3 seconds
  
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    const response = await fetch(
      `/api/v1/analyses/${analysisId}?fields=status,completedAt,outputPdfUrl`,
      {
        headers: {
          'If-None-Match': lastEtag  // Use ETag to reduce bandwidth
        }
      }
    );
    
    if (response.status === 304) {
      // Not modified, wait and retry
      await sleep(pollInterval);
      continue;
    }
    
    const data = await response.json();
    
    if (['complete', 'partial_failure', 'failed'].includes(data.data.status)) {
      return data.data;  // Processing finished
    }
    
    // Still processing, wait and retry
    await sleep(pollInterval);
  }
  
  throw new Error('Analysis timed out');
}
```

**Optimization: Exponential Backoff (Future Enhancement)**

For production, consider exponential backoff:
- Attempts 1-10: 2 seconds
- Attempts 11-30: 5 seconds
- Attempts 31+: 10 seconds

---

## File Upload Flow (Complete Sequence)

**Step-by-step flow from user clicking "Upload PDF" to parsing complete:**

### 1. User selects PDF file in browser

Frontend validates:
- File type is `application/pdf`
- File size ≤ 10MB

### 2. Frontend requests signed upload URL

```typescript
POST /api/v1/analyses/{analysisId}/upload-url
{
  "fileName": "cliftonstrengths_report.pdf",
  "fileSize": 2456789,
  "contentType": "application/pdf"
}
```

### 3. Backend returns signed URL

```typescript
{
  "uploadUrl": "https://supabase.co/storage/v1/object/upload/...",
  "filePath": "uploads/{userId}/{analysisId}/cliftonstrengths.pdf",
  "expiresAt": "2026-02-15T10:45:00Z"
}
```

### 4. Frontend uploads directly to Supabase Storage

```typescript
const response = await fetch(signedUploadUrl, {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/pdf'
  },
  body: pdfFile
});
```

### 5. Frontend triggers parsing

```typescript
POST /api/v1/analyses/{analysisId}/trigger-parsing
{
  "filePath": "uploads/{userId}/{analysisId}/cliftonstrengths.pdf"
}
```

### 6. Backend validates file exists and starts parsing

- Update analysis status to `parsing`
- Trigger background job
- Return 202 Accepted immediately

### 7. Background job processes PDF

- Download from Supabase Storage
- Extract text with pdf-parse library
- Send to Gemini Flash 2.0 with parsing prompt
- Validate extracted data
- Update database with CliftonStrengths data
- Set `parsing_completed_at` timestamp
- Change status back to `draft`

### 8. Frontend polls for completion

```typescript
GET /api/v1/analyses/{analysisId}?fields=status,parsingCompletedAt
// Poll every 2-3 seconds until parsingCompletedAt is not null
```

**Total Expected Duration:** 3-8 seconds for typical PDF

---

## Request/Response Headers

### Standard Request Headers

All authenticated requests should include:

```http
Cookie: sb-access-token=...; sb-refresh-token=...
Content-Type: application/json (for POST/PATCH requests)
Accept: application/json
```

Optional:
```http
If-None-Match: "etag-value" (for efficient polling)
```

### Standard Response Headers

All responses include:

```http
Content-Type: application/json
X-Request-ID: req_abc123xyz (for debugging)
X-RateLimit-Limit: 2
X-RateLimit-Remaining: 1
X-RateLimit-Reset: 1708012800
```

For cacheable resources (analysis details):
```http
ETag: "analysis-id-updated-at"
Cache-Control: no-cache (must revalidate)
```

---

## Implementation Checklist for Claude Code

When implementing this API specification, ensure:

### Authentication & Middleware

- [ ] Supabase client initialization with environment variables
- [ ] Authentication middleware validates session cookies
- [ ] Extract `user.id` from validated session and attach to request
- [ ] Return 401 for missing/invalid sessions
- [ ] CORS middleware with environment-based origin allowlist

### Database Integration

- [ ] Supabase client configured for database queries
- [ ] All queries use parameterized statements (prevent SQL injection)
- [ ] Row Level Security (RLS) policies handle data access
- [ ] Queries use `user_id` from authenticated session

### Rate Limiting

- [ ] Implement rate limit check function (queries last 24h submissions)
- [ ] Add rate limit headers to all authenticated responses
- [ ] Enforce limit on submit endpoint with 429 response
- [ ] Calculate `resetAt` timestamp correctly

### Error Handling

- [ ] Global error handler catches all exceptions
- [ ] Map exceptions to standardized error format
- [ ] Include `requestId` in error responses for debugging
- [ ] Log errors with context (user ID, analysis ID, stack trace)

### File Upload

- [ ] Generate signed upload URLs using Supabase Storage SDK
- [ ] Validate file size and content type before generating URL
- [ ] Use consistent path pattern: `uploads/{userId}/{analysisId}/{fileName}`
- [ ] Set appropriate expiration time (15 minutes)

### Background Jobs

- [ ] Trigger parsing job asynchronously (don't block API response)
- [ ] Trigger processing job asynchronously
- [ ] Jobs update database with results and errors
- [ ] Jobs handle retries for transient failures

### Status Polling

- [ ] Implement `fields` query parameter for lightweight responses
- [ ] Return ETag header based on analysis ID + updatedAt
- [ ] Support If-None-Match header for 304 responses
- [ ] Include all status transitions in response

### Validation

- [ ] Validate request schemas before database operations
- [ ] Check analysis status before allowing state changes
- [ ] Enforce goal constraints (max 3, only 1 high priority)
- [ ] Validate category names against allowed list

### Testing Endpoints

- [ ] Health check returns 200 with version info
- [ ] All endpoints require authentication (except health check)
- [ ] Endpoints return correct status codes (200, 201, 202, 204, 400, 401, 404, 409, 429, 500)
- [ ] Error responses follow standardized format

---

## API Versioning Strategy

### Current Version: v1

**Breaking Change Policy:**

A change is considered "breaking" if it:
- Removes an endpoint
- Removes a required field from response
- Adds a required field to request
- Changes field data types
- Changes error code semantics

**Non-Breaking Changes (Safe to Add):**

- New optional request fields
- New response fields
- New endpoints
- New error codes (for new scenarios)

### Future Versioning (When Needed)

**Option A: URL Path Versioning (Recommended)**
```
/api/v1/analyses
/api/v2/analyses
```

**Option B: Header Versioning**
```
Accept: application/vnd.goalos.v2+json
```

For MVP, stick with **Option A** (path versioning) for simplicity.

**Deprecation Process:**

1. Announce deprecation 3 months in advance
2. Add `Deprecated: true` response header to old version
3. Maintain old version for 6 months minimum
4. Sunset old version with 410 Gone responses

---

## Security Considerations

### Input Validation

- **Never trust client input** - validate all request bodies server-side
- **Sanitize file names** - prevent path traversal attacks
- **Limit request sizes** - max 10MB for file uploads, 100KB for JSON payloads

### Authentication

- **Use Supabase session validation** - don't implement custom JWT parsing
- **Check session on every request** - don't cache authentication state
- **Handle expired sessions gracefully** - return 401 with clear message

### Authorization

- **Rely on Row Level Security (RLS)** - database enforces data access
- **Always filter by user_id** - never trust client-provided user IDs
- **Validate resource ownership** - ensure analysis belongs to authenticated user

### Rate Limiting

- **Enforce at API layer** - don't rely solely on client-side checks
- **Use database queries** - most accurate rate limit calculation
- **Consider abuse patterns** - monitor for suspicious activity

### File Upload

- **Validate file types** - check magic bytes, not just extension
- **Scan for malware** - consider integrating virus scanning (future)
- **Limit file sizes** - prevent DoS via large uploads
- **Use signed URLs** - prevent unauthorized access to storage

### Error Messages

- **Don't leak sensitive info** - avoid exposing database errors to users
- **Use generic messages** - "Something went wrong" for 500 errors
- **Log detailed errors** - but only on server side
- **Include request IDs** - for debugging without exposing internals

---

## Performance Optimization

### Database Queries

- **Use indexes** - especially on user_id, created_at, status fields
- **Limit result sets** - default to 50 analyses max per request
- **Use field filtering** - `?fields=status,updatedAt` for lightweight queries
- **Implement ETag caching** - reduce bandwidth for polling

### File Operations

- **Direct client → Supabase uploads** - avoid double network hop through API
- **Use signed URLs** - offload storage operations to Supabase
- **Set appropriate expirations** - 15 minutes for upload URLs, 1 hour for download URLs

### API Responses

- **Gzip compression** - enable for all JSON responses
- **Minimal payloads** - only return requested fields
- **Cache headers** - use ETags for conditional requests
- **Async processing** - never block API responses waiting for AI

---

## Monitoring & Observability

### Request Logging

Log every API request with:
- Request ID (generated)
- User ID (from session)
- Endpoint and method
- Response status code
- Response time (ms)
- Error details (if applicable)

**Example Log Entry:**
```json
{
  "requestId": "req_abc123xyz",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "method": "POST",
  "endpoint": "/api/v1/analyses/a1b2c3d4/submit",
  "statusCode": 202,
  "responseTimeMs": 345,
  "timestamp": "2026-02-15T10:35:00Z"
}
```

### Metrics to Track

- **Request rate** - requests per minute/hour
- **Success rate** - 2xx responses / total requests
- **Error rate** - 4xx and 5xx responses / total requests
- **Response times** - p50, p95, p99 latencies
- **Rate limit hits** - 429 responses / total requests
- **Parsing success rate** - successful CliftonStrengths extractions
- **Processing success rate** - complete analyses / submitted analyses

### Alerts

Set up alerts for:
- **Error rate > 5%** (over 5 minute window)
- **Response time p95 > 2 seconds**
- **Rate limit hit rate > 50%** (may need to adjust limits)
- **Parsing failure rate > 10%**
- **Processing failure rate > 5%**

**Monitoring Tools:**
- Vercel Analytics (built-in)
- Sentry (error tracking)
- Supabase Dashboard (database metrics)

---

## Next Steps: Document 7

After completing this API Specification, the next document in the sequence is:

**Document 7: Security & Authentication Specification**

This will detail:
- Supabase Auth configuration (Google OAuth setup)
- Row Level Security (RLS) policies for all tables
- API key management (if needed for integrations)
- Security headers and CSRF protection
- Data encryption at rest and in transit
- Compliance considerations (GDPR, data retention)

---

**Document Status:** Complete  
**Dependencies:** User Flow (#1), Database Schema (#2), AI Task Specifications (#3)  
**Next Document:** Security & Authentication Specification (#7)  
**Implementation Ready:** Yes - can be used directly by Claude Code

---

*This API specification provides complete implementation details for building the Goal Achievement MVP REST API with Claude Code. All endpoints, schemas, error codes, and patterns are defined with sufficient detail for autonomous code generation.*
