# Security & Authentication Specification
## Personal Time OS - Goal Achievement MVP

**Version:** 1.0  
**Created:** February 15, 2026  
**Purpose:** Complete security architecture for MVP implementation with Claude Code  
**Database:** Supabase (PostgreSQL 15+ with Auth & Storage)  
**Hosting:** Vercel (Node.js 18+ serverless)

---

## Executive Summary

This document defines the complete security architecture for the Goal Achievement MVP, implementing defense-in-depth with multiple security layers: Supabase Auth for authentication, Row Level Security for database isolation, API middleware for request validation, signed URLs for file access control, and comprehensive security headers for infrastructure hardening.

**Key Security Principles:**
- **Defense in Depth:** Multiple independent security layers
- **Least Privilege:** Users access only their own data
- **Secure by Default:** All endpoints require authentication, all data encrypted
- **Fail Securely:** Errors deny access rather than grant it
- **Audit Trail:** All security events logged for debugging and monitoring

**Security Architecture Overview:**

```
┌─────────────────────────────────────────────────────────────┐
│                        User Browser                          │
│  • HTTPS only (TLS 1.2+)                                    │
│  • HTTP-only session cookies                                │
│  • CSP headers prevent XSS                                  │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                   Vercel Edge Network                        │
│  • Rate limiting (100 req/15min per IP)                     │
│  • HSTS, X-Frame-Options, security headers                  │
│  • DDoS protection (Vercel managed)                         │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                API Middleware (Vercel Functions)             │
│  • Session validation (Supabase Auth)                       │
│  • Rate limit check (2 analyses/24hr)                       │
│  • Input validation & sanitization                          │
│  • Request logging                                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                  Supabase (Database + Auth)                  │
│  • Row Level Security (RLS) policies                        │
│  • Encryption at rest (AES-256)                             │
│  • Automatic backups (encrypted)                            │
│  • Audit logs (auth events)                                 │
└─────────────────────────────────────────────────────────────┘
```

**Technology Stack Security:**
- **Authentication:** Supabase Auth with Google OAuth
- **Session Management:** HTTP-only cookies, 1hr access + 30d refresh tokens
- **Database Security:** PostgreSQL RLS policies, parameterized queries
- **File Storage:** Supabase Storage with signed URLs (15min expiry)
- **API Security:** Vercel serverless with environment variable secrets
- **Transport Security:** TLS 1.2+ enforced via HSTS headers

**OWASP Top 10 Coverage:**
This specification explicitly addresses all relevant OWASP vulnerabilities for the MVP:
- ✓ A01 (Broken Access Control) → RLS policies + session validation
- ✓ A02 (Cryptographic Failures) → TLS + encryption at rest
- ✓ A03 (Injection) → Parameterized queries + input validation
- ✓ A04 (Insecure Design) → Threat modeling + security testing
- ✓ A05 (Security Misconfiguration) → Security headers + hardening
- ✓ A07 (Auth Failures) → Supabase Auth + secure session management
- ✓ A08 (Software Integrity) → Dependency scanning + Supabase backups
- ✓ A09 (Logging Failures) → Comprehensive security event logging
- ✓ A10 (SSRF) → No outbound requests from user content

---

## Table of Contents

1. [Authentication & Session Management](#authentication--session-management)
2. [Row Level Security (RLS) Policies](#row-level-security-rls-policies)
3. [API Security](#api-security)
4. [File Upload Security](#file-upload-security)
5. [Rate Limiting & Abuse Prevention](#rate-limiting--abuse-prevention)
6. [API Key Management](#api-key-management)
7. [Data Encryption](#data-encryption)
8. [Security Headers](#security-headers)
9. [Error Handling & Information Disclosure](#error-handling--information-disclosure)
10. [Privacy & GDPR Compliance](#privacy--gdpr-compliance)
11. [OWASP Top 10 Compliance](#owasp-top-10-compliance)
12. [Security Testing Checklist](#security-testing-checklist)
13. [Incident Response](#incident-response)

---

## Authentication & Session Management

### Supabase Auth Configuration

**Provider:** Google OAuth (single sign-on only, no email/password)

**Configuration Steps:**

1. **Enable Google OAuth in Supabase Dashboard:**
   - Navigate to Authentication → Providers → Google
   - Enable provider
   - Configure OAuth credentials from Google Cloud Console

2. **Google Cloud Console Setup:**
   - Create OAuth 2.0 Client ID (Web application)
   - **Authorized JavaScript origins:**
     - Production: `https://goalos.app`
     - Development: `http://localhost:3000`
   - **Authorized redirect URIs:**
     - Production: `https://{project-ref}.supabase.co/auth/v1/callback`
     - Development: `https://{project-ref}.supabase.co/auth/v1/callback`
   - Copy Client ID and Client Secret to Supabase settings

3. **Supabase Auth Settings:**
   - **Email confirmation:** DISABLED (Google already verifies emails)
   - **Site URL:** `https://goalos.app` (production), `http://localhost:3000` (development)
   - **Redirect URLs (allowed):**
     - `https://goalos.app/assessment/step-1`
     - `http://localhost:3000/assessment/step-1`
   - **Session duration:** Keep Supabase defaults:
     - Access token: 1 hour
     - Refresh token: 30 days
   - **Auto-refresh:** Enabled (SDK handles automatically)

4. **User Metadata Storage:**

From Google OAuth profile, store in `auth.users` table (Supabase managed):
- `email` (required)
- `user_metadata.full_name`
- `user_metadata.avatar_url`

Extend in `public.users` table (our table):
- `id` (FK to auth.users.id)
- `email` (denormalized for querying)
- `full_name`
- `avatar_url`
- `total_analyses_count`
- `created_at`, `updated_at`

### Session Management

**Cookie Configuration:**

Supabase automatically manages session cookies with secure defaults:

```
Cookie Name: sb-access-token
Properties:
  - HttpOnly: true (prevents JavaScript access)
  - Secure: true (HTTPS only in production)
  - SameSite: Lax (CSRF protection)
  - Path: /
  - Max-Age: 3600 (1 hour)

Cookie Name: sb-refresh-token
Properties:
  - HttpOnly: true
  - Secure: true
  - SameSite: Lax
  - Path: /
  - Max-Age: 2592000 (30 days)
```

**Session Validation (API Middleware):**

Every API request validates the session using Supabase SDK:

```typescript
// middleware/auth.ts
import { createServerSupabaseClient } from '@supabase/auth-helpers-nextjs';
import type { NextRequest } from 'next/server';

export async function validateSession(req: NextRequest) {
  const supabase = createServerSupabaseClient({ req });
  
  const {
    data: { session },
    error
  } = await supabase.auth.getSession();
  
  if (error || !session) {
    return {
      authenticated: false,
      error: {
        code: 'UNAUTHORIZED',
        message: 'Authentication required. Please log in.',
        statusCode: 401
      }
    };
  }
  
  return {
    authenticated: true,
    userId: session.user.id,
    user: session.user
  };
}
```

**Session Refresh:**

Frontend automatically refreshes tokens using Supabase SDK:

```typescript
// Frontend: app/layout.tsx or _app.tsx
import { createClientComponentClient } from '@supabase/auth-helpers-nextjs';

const supabase = createClientComponentClient();

// SDK automatically refreshes access token when expired
// No manual refresh logic needed
```

**Session Expiration Handling:**

When access token expires:
1. SDK attempts auto-refresh using refresh token
2. If refresh succeeds → User continues seamlessly
3. If refresh fails (refresh token expired) → 401 error
4. Frontend intercepts 401 → Redirects to login page

**Logout Implementation:**

```typescript
// Frontend logout
const { error } = await supabase.auth.signOut();
if (!error) {
  // Redirect to landing page
  window.location.href = '/';
}
```

### Authentication Flow Diagram

```
┌──────────────┐
│ Landing Page │
└──────┬───────┘
       │
       │ [Click "Get Your Strategy"]
       │
       ↓
┌─────────────────────────────────────────┐
│  Frontend: supabase.auth.signInWithOAuth │
│  Provider: Google                        │
└──────┬──────────────────────────────────┘
       │
       ↓
┌──────────────────────────┐
│  Google OAuth Consent    │
│  - Email, Name, Avatar   │
└──────┬───────────────────┘
       │
       │ [User approves]
       │
       ↓
┌────────────────────────────────────────┐
│  Supabase Auth Callback                │
│  - Creates auth.users record           │
│  - Issues access + refresh tokens      │
│  - Sets HTTP-only cookies              │
└──────┬─────────────────────────────────┘
       │
       ↓
┌────────────────────────────────────────┐
│  Database Trigger: upsert_user_profile │
│  - Creates/updates public.users record │
│  - Stores email, name, avatar          │
└──────┬─────────────────────────────────┘
       │
       ↓
┌────────────────────────────────────────┐
│  Redirect to /assessment/step-1        │
│  - User authenticated                  │
│  - Session cookies set                 │
└────────────────────────────────────────┘
```

---

## Row Level Security (RLS) Policies

All user data tables use PostgreSQL Row Level Security to enforce data isolation at the database layer. This ensures even compromised application code cannot access unauthorized data.

**Global Principle:** Users can only SELECT, INSERT, UPDATE, DELETE their own records (where `user_id = auth.uid()`).

### Table: `public.users`

**Purpose:** User profile data (extends auth.users)

```sql
-- Enable RLS
ALTER TABLE public.users ENABLE ROW LEVEL SECURITY;

-- Policy: Users can view their own profile
CREATE POLICY "users_select_own"
  ON public.users
  FOR SELECT
  USING (auth.uid() = id);

-- Policy: Users can update their own profile
CREATE POLICY "users_update_own"
  ON public.users
  FOR UPDATE
  USING (auth.uid() = id);

-- Policy: System can insert new users (triggered by auth.users creation)
CREATE POLICY "users_insert_system"
  ON public.users
  FOR INSERT
  WITH CHECK (auth.uid() = id);

-- Note: No DELETE policy - users are soft-deleted via status flag
```

**Reasoning:**
- SELECT: Users read only their own profile
- UPDATE: Users modify only their own profile (preferences, etc.)
- INSERT: Handled by database trigger on auth.users creation
- DELETE: Not allowed (prevent accidental data loss)

### Table: `public.analyses`

**Purpose:** Core analysis records with state tracking

```sql
-- Enable RLS
ALTER TABLE public.analyses ENABLE ROW LEVEL SECURITY;

-- Policy: Users can view their own analyses
CREATE POLICY "analyses_select_own"
  ON public.analyses
  FOR SELECT
  USING (auth.uid() = user_id);

-- Policy: Users can create analyses for themselves
CREATE POLICY "analyses_insert_own"
  ON public.analyses
  FOR INSERT
  WITH CHECK (auth.uid() = user_id);

-- Policy: Users can update their own analyses
CREATE POLICY "analyses_update_own"
  ON public.analyses
  FOR UPDATE
  USING (auth.uid() = user_id);

-- Policy: Users can delete their own analyses
CREATE POLICY "analyses_delete_own"
  ON public.analyses
  FOR DELETE
  USING (auth.uid() = user_id);
```

**Reasoning:**
- SELECT: Users see only their own analyses
- INSERT: Users create analyses only for themselves (user_id must match session)
- UPDATE: Users modify only their own analyses (auto-save, status changes)
- DELETE: Users can permanently delete their own analyses

### Table: `public.goals`

**Purpose:** Individual goals (1-3 per analysis)

```sql
-- Enable RLS
ALTER TABLE public.goals ENABLE ROW LEVEL SECURITY;

-- Policy: Users can view goals for their own analyses
CREATE POLICY "goals_select_own"
  ON public.goals
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM public.analyses
      WHERE analyses.id = goals.analysis_id
        AND analyses.user_id = auth.uid()
    )
  );

-- Policy: Users can create goals for their own analyses
CREATE POLICY "goals_insert_own"
  ON public.goals
  FOR INSERT
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM public.analyses
      WHERE analyses.id = goals.analysis_id
        AND analyses.user_id = auth.uid()
    )
  );

-- Policy: Users can update goals for their own analyses
CREATE POLICY "goals_update_own"
  ON public.goals
  FOR UPDATE
  USING (
    EXISTS (
      SELECT 1 FROM public.analyses
      WHERE analyses.id = goals.analysis_id
        AND analyses.user_id = auth.uid()
    )
  );

-- Policy: Users can delete goals for their own analyses
CREATE POLICY "goals_delete_own"
  ON public.goals
  FOR DELETE
  USING (
    EXISTS (
      SELECT 1 FROM public.analyses
      WHERE analyses.id = goals.analysis_id
        AND analyses.user_id = auth.uid()
    )
  );
```

**Reasoning:**
- All policies check ownership via JOIN to analyses table
- Users can only access goals belonging to their analyses
- Cascading security: analysis ownership → goal ownership

### Table: `public.ai_agent_outputs`

**Purpose:** Temporary storage of agent outputs (cleaned up after processing)

```sql
-- Enable RLS
ALTER TABLE public.ai_agent_outputs ENABLE ROW LEVEL SECURITY;

-- Policy: Users can view agent outputs for their own analyses
CREATE POLICY "agent_outputs_select_own"
  ON public.ai_agent_outputs
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM public.analyses
      WHERE analyses.id = ai_agent_outputs.analysis_id
        AND analyses.user_id = auth.uid()
    )
  );

-- Policy: System can insert agent outputs (background jobs)
-- Note: Background jobs use service role, bypass RLS
-- No user-facing INSERT policy needed

-- Policy: System can delete agent outputs (cleanup after completion)
-- Note: Cleanup uses service role, bypass RLS
-- No user-facing DELETE policy needed
```

**Reasoning:**
- Users can view agent outputs for debugging (while analysis is processing)
- INSERT/DELETE handled by background jobs using service role
- Service role bypasses RLS (trusted system operations)

### Table: `public.prompts`

**Purpose:** Versioned AI prompt templates (read-only for users)

```sql
-- Enable RLS
ALTER TABLE public.prompts ENABLE ROW LEVEL SECURITY;

-- Policy: All authenticated users can read prompts
CREATE POLICY "prompts_select_all"
  ON public.prompts
  FOR SELECT
  USING (auth.uid() IS NOT NULL);

-- No INSERT/UPDATE/DELETE policies for users
-- Prompts managed by admin/migrations only
```

**Reasoning:**
- Prompts are shared across all users (not user-specific)
- Read-only for regular users
- INSERT/UPDATE/DELETE via service role (admin operations)

### Service Role Access

**Background Jobs & Admin Operations:**

Background jobs and admin operations use the Supabase **service role key**, which **bypasses all RLS policies**. This is necessary for:
- Creating analyses via API (user_id comes from validated session)
- Inserting agent outputs during processing
- Cleanup jobs deleting old agent outputs
- Admin dashboard viewing all analyses (future)

**Security:**
- Service role key stored in Vercel environment variables (never exposed to frontend)
- Only server-side code can use service role
- All service role operations logged

---

## API Security

### Input Validation

**Principle:** Never trust client input. Validate all request bodies server-side.

**Validation Libraries:**
- **Zod** for schema validation (TypeScript-first)
- **validator.js** for string sanitization

**Example: Goal Creation Validation**

```typescript
// schemas/goal.ts
import { z } from 'zod';

export const GoalSchema = z.object({
  title: z.string()
    .min(3, 'Goal title must be at least 3 characters')
    .max(200, 'Goal title must be less than 200 characters')
    .trim(),
  
  description: z.string()
    .max(1000, 'Description must be less than 1000 characters')
    .trim()
    .optional(),
  
  priority: z.enum(['high', 'medium', 'low']),
  
  deadline: z.string()
    .datetime({ message: 'Deadline must be valid ISO 8601 datetime' })
    .refine(
      (date) => new Date(date) > new Date(),
      { message: 'Deadline must be in the future' }
    ),
  
  category: z.enum([
    'work_career',
    'health_fitness',
    'relationships_family',
    'learning_growth',
    'creativity_hobbies',
    'rest_recovery',
    'community_service',
    'finance_security'
  ]),
  
  difficulty: z.enum(['easy', 'moderate', 'challenging', 'ambitious'])
});

// API endpoint validation
import { GoalSchema } from '@/schemas/goal';

export async function POST(req: Request) {
  const body = await req.json();
  
  // Validate input
  const validation = GoalSchema.safeParse(body);
  if (!validation.success) {
    return Response.json(
      {
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Invalid goal data',
          statusCode: 400,
          details: validation.error.flatten()
        }
      },
      { status: 400 }
    );
  }
  
  const goal = validation.data;
  // ... proceed with validated data
}
```

**Validation Rules:**

1. **String fields:**
   - Trim whitespace
   - Enforce min/max lengths
   - Sanitize HTML (strip tags if not expected)

2. **Numeric fields:**
   - Check ranges (e.g., hours/week 0-168)
   - Validate integer vs float

3. **Enum fields:**
   - Whitelist allowed values
   - Reject unknown values

4. **Date fields:**
   - Validate ISO 8601 format
   - Check future/past constraints

5. **File uploads:**
   - Validate file size (max 10MB)
   - Validate MIME type (application/pdf)
   - Check magic bytes (PDF signature: `25 50 44 46`)

### CSRF Protection

**Mechanism:** SameSite cookie attribute + origin validation

**SameSite Cookies:**

Supabase session cookies use `SameSite=Lax`, which provides CSRF protection for:
- POST requests (cookies not sent from cross-origin)
- State-changing operations

**Additional Protection:**

For high-security operations (delete analysis, submit for processing), verify Origin header:

```typescript
// middleware/csrf.ts
export function validateOrigin(req: Request): boolean {
  const origin = req.headers.get('origin');
  const allowedOrigins = [
    'https://goalos.app',
    'http://localhost:3000' // Development only
  ];
  
  return origin ? allowedOrigins.includes(origin) : false;
}

// Usage in API endpoint
export async function POST(req: Request) {
  if (!validateOrigin(req)) {
    return Response.json(
      { error: { code: 'INVALID_ORIGIN', message: 'Request origin not allowed', statusCode: 403 } },
      { status: 403 }
    );
  }
  // ... proceed
}
```

**Note:** For MVP, SameSite=Lax is sufficient. Explicit CSRF tokens not needed.

### SQL Injection Prevention

**Mechanism:** Parameterized queries only (never string concatenation)

**Supabase Client (Safe):**

```typescript
// ✓ SAFE - Parameterized query
const { data, error } = await supabase
  .from('analyses')
  .select('*')
  .eq('user_id', userId)
  .eq('id', analysisId);

// ✗ NEVER DO THIS - String concatenation
const { data } = await supabase.rpc('raw_query', {
  query: `SELECT * FROM analyses WHERE id = '${analysisId}'`
});
```

**Raw SQL (When Necessary):**

Use `$1`, `$2` placeholders:

```typescript
// ✓ SAFE - Parameterized raw SQL
const { data } = await supabase.rpc('custom_function', {
  p_user_id: userId,
  p_analysis_id: analysisId
});

// Database function definition
CREATE FUNCTION custom_function(p_user_id UUID, p_analysis_id UUID)
RETURNS TABLE(...) AS $$
  SELECT * FROM analyses
  WHERE user_id = p_user_id AND id = p_analysis_id;
$$ LANGUAGE sql SECURITY DEFINER;
```

### API Endpoint Security Checklist

Every API endpoint must:

- [ ] Validate session via `validateSession(req)` middleware
- [ ] Extract `userId` from validated session (never trust client-provided user IDs)
- [ ] Validate request body with Zod schema
- [ ] Use parameterized queries only (Supabase client methods)
- [ ] Check rate limits before processing (for analysis submission)
- [ ] Log security events (failed auth, validation errors)
- [ ] Return standardized error responses
- [ ] Include `X-Request-ID` header for debugging

---

## File Upload Security

### Threat Model

**Risks:**
- Malicious PDF files (exploiting parser vulnerabilities)
- Oversized files (DoS via storage exhaustion)
- Wrong file types (user mistake or malicious)
- Path traversal attacks (manipulating file paths)
- Metadata leakage (EXIF data, authorship info)

**Mitigations:**
- Magic byte validation (verify actual file type)
- File size limits (10MB hard cap)
- UUID-based filenames (prevent path traversal)
- Signed URLs with expiry (time-limited access)
- Isolated parsing environment (Vercel function sandboxing)

### Upload Flow Security

**Step 1: Request Signed Upload URL**

```typescript
// POST /api/v1/analyses/{analysisId}/upload-url
export async function POST(req: Request, { params }: { params: { analysisId: string } }) {
  // 1. Validate session
  const auth = await validateSession(req);
  if (!auth.authenticated) {
    return Response.json({ error: auth.error }, { status: 401 });
  }
  
  // 2. Validate analysis ownership (RLS handles this, but double-check)
  const { data: analysis } = await supabase
    .from('analyses')
    .select('user_id, status')
    .eq('id', params.analysisId)
    .single();
  
  if (!analysis || analysis.user_id !== auth.userId) {
    return Response.json(
      { error: { code: 'NOT_FOUND', message: 'Analysis not found', statusCode: 404 } },
      { status: 404 }
    );
  }
  
  if (analysis.status !== 'draft') {
    return Response.json(
      { error: { code: 'INVALID_STATE', message: 'Cannot upload to non-draft analysis', statusCode: 400 } },
      { status: 400 }
    );
  }
  
  // 3. Generate secure file path (UUID-based, no user input)
  const fileName = `cliftonstrengths-${Date.now()}.pdf`;
  const filePath = `uploads/${auth.userId}/${params.analysisId}/${fileName}`;
  
  // 4. Generate signed upload URL (15 minute expiry)
  const { data: signedUrl, error } = await supabase.storage
    .from('analysis-uploads')
    .createSignedUploadUrl(filePath, {
      upsert: false // Don't overwrite existing files
    });
  
  if (error) {
    return Response.json(
      { error: { code: 'STORAGE_ERROR', message: 'Failed to generate upload URL', statusCode: 500 } },
      { status: 500 }
    );
  }
  
  return Response.json({
    data: {
      uploadUrl: signedUrl.signedUrl,
      filePath: filePath,
      expiresAt: new Date(Date.now() + 15 * 60 * 1000).toISOString()
    }
  });
}
```

**Step 2: Client Uploads Directly to Supabase Storage**

```typescript
// Frontend: Direct upload (bypasses API)
const response = await fetch(signedUploadUrl, {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/pdf',
    'Content-Length': file.size.toString()
  },
  body: file
});

if (!response.ok) {
  throw new Error('Upload failed');
}
```

**Supabase Storage Bucket Configuration:**

```sql
-- Create bucket with security settings
INSERT INTO storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
VALUES (
  'analysis-uploads',
  'analysis-uploads',
  false, -- Not public (require signed URLs)
  10485760, -- 10MB limit (10 * 1024 * 1024 bytes)
  ARRAY['application/pdf'] -- PDF only
);

-- RLS policy: Users can upload to their own folder
CREATE POLICY "users_upload_own_folder"
  ON storage.objects
  FOR INSERT
  WITH CHECK (
    bucket_id = 'analysis-uploads' AND
    (storage.foldername(name))[1] = auth.uid()::text
  );

-- RLS policy: Users can read their own uploads
CREATE POLICY "users_read_own_uploads"
  ON storage.objects
  FOR SELECT
  USING (
    bucket_id = 'analysis-uploads' AND
    (storage.foldername(name))[1] = auth.uid()::text
  );
```

### File Validation

**Step 3: Backend Validates File Before Parsing**

```typescript
// POST /api/v1/analyses/{analysisId}/trigger-parsing
export async function POST(req: Request, { params }: { params: { analysisId: string } }) {
  const { filePath } = await req.json();
  
  // 1. Validate session (omitted for brevity)
  
  // 2. Download file from storage (server-side only)
  const { data: fileData, error } = await supabase.storage
    .from('analysis-uploads')
    .download(filePath);
  
  if (error || !fileData) {
    return Response.json(
      { error: { code: 'FILE_NOT_FOUND', message: 'File not found', statusCode: 404 } },
      { status: 404 }
    );
  }
  
  // 3. Validate file size (double-check, even though bucket has limit)
  const MAX_SIZE = 10 * 1024 * 1024; // 10MB
  if (fileData.size > MAX_SIZE) {
    return Response.json(
      { error: { code: 'FILE_TOO_LARGE', message: 'File exceeds 10MB limit', statusCode: 400 } },
      { status: 400 }
    );
  }
  
  // 4. Validate magic bytes (PDF signature: %PDF)
  const buffer = await fileData.arrayBuffer();
  const bytes = new Uint8Array(buffer);
  const isPDF = 
    bytes[0] === 0x25 && // %
    bytes[1] === 0x50 && // P
    bytes[2] === 0x44 && // D
    bytes[3] === 0x46;   // F
  
  if (!isPDF) {
    return Response.json(
      { error: { code: 'INVALID_FILE_TYPE', message: 'File is not a valid PDF', statusCode: 400 } },
      { status: 400 }
    );
  }
  
  // 5. Trigger background parsing job
  // ... (enqueue job with validated filePath)
}
```

**Magic Byte Validation:**

Checks actual file content, not just extension or MIME type:

| File Type | Magic Bytes (Hex) | Description |
|-----------|-------------------|-------------|
| PDF       | `25 50 44 46`     | %PDF        |

**For MVP:** Only PDF validation. Future: expand to images, spreadsheets if needed.

### PDF Parsing Security

**Isolation:**
- PDF parsing runs in isolated Vercel function (separate from API endpoints)
- Function timeout: 60 seconds max (prevents infinite loops in malicious PDFs)
- No outbound network access from PDF content (SSRF prevention)

**Library Security:**
- Use `pdf-parse` library (actively maintained, no known CVEs)
- Update dependencies weekly via `npm audit`
- Pin exact versions in package-lock.json

**Error Handling:**
- Catch all parsing errors (don't crash function)
- Log errors with file hash (not full content)
- Return generic "parsing failed" message to user (don't leak internal errors)

**Content Sanitization:**
- Extract text only (no embedded files, scripts, or links)
- Strip JavaScript from PDF (pdf-parse doesn't execute it, but be explicit)
- Validate extracted text length (max 50KB text content)

---

## Rate Limiting & Abuse Prevention

### Analysis Submission Rate Limit

**Limit:** 2 analyses per user per 24 hours (rolling window)

**Enforcement Points:**

1. **Pre-flight Check (API Endpoint):**

```typescript
// GET /api/v1/users/me/quota
export async function GET(req: Request) {
  const auth = await validateSession(req);
  if (!auth.authenticated) {
    return Response.json({ error: auth.error }, { status: 401 });
  }
  
  // Count analyses submitted in last 24 hours
  const twentyFourHoursAgo = new Date(Date.now() - 24 * 60 * 60 * 1000);
  
  const { data: recentAnalyses, error } = await supabase
    .from('analyses')
    .select('id, submitted_at')
    .eq('user_id', auth.userId)
    .gte('submitted_at', twentyFourHoursAgo.toISOString())
    .not('submitted_at', 'is', null); // Only count submitted analyses
  
  const used = recentAnalyses?.length || 0;
  const limit = 2;
  const remaining = Math.max(0, limit - used);
  
  // Calculate reset time (24 hours after earliest submission)
  let resetAt: string | null = null;
  if (used >= limit && recentAnalyses && recentAnalyses.length > 0) {
    const earliestSubmission = new Date(
      Math.min(...recentAnalyses.map(a => new Date(a.submitted_at!).getTime()))
    );
    resetAt = new Date(earliestSubmission.getTime() + 24 * 60 * 60 * 1000).toISOString();
  }
  
  return Response.json({
    data: {
      limit,
      used,
      remaining,
      resetAt
    }
  });
}
```

2. **Submission Enforcement (Database Constraint):**

```typescript
// POST /api/v1/analyses/{analysisId}/submit
export async function POST(req: Request, { params }: { params: { analysisId: string } }) {
  const auth = await validateSession(req);
  
  // Check rate limit before submission
  const twentyFourHoursAgo = new Date(Date.now() - 24 * 60 * 60 * 1000);
  
  const { data: recentCount } = await supabase
    .from('analyses')
    .select('id', { count: 'exact', head: true })
    .eq('user_id', auth.userId)
    .gte('submitted_at', twentyFourHoursAgo.toISOString())
    .not('submitted_at', 'is', null);
  
  if ((recentCount?.count || 0) >= 2) {
    // Calculate reset time
    const { data: earliest } = await supabase
      .from('analyses')
      .select('submitted_at')
      .eq('user_id', auth.userId)
      .gte('submitted_at', twentyFourHoursAgo.toISOString())
      .not('submitted_at', 'is', null)
      .order('submitted_at', { ascending: true })
      .limit(1)
      .single();
    
    const resetAt = earliest 
      ? new Date(new Date(earliest.submitted_at).getTime() + 24 * 60 * 60 * 1000)
      : new Date(Date.now() + 24 * 60 * 60 * 1000);
    
    return Response.json(
      {
        error: {
          code: 'RATE_LIMIT_EXCEEDED',
          message: `You've used your 2 analyses today. Next analysis available in ${Math.ceil((resetAt.getTime() - Date.now()) / (60 * 60 * 1000))} hours.`,
          statusCode: 429,
          details: {
            limit: 2,
            used: 2,
            remaining: 0,
            resetAt: resetAt.toISOString()
          }
        }
      },
      { 
        status: 429,
        headers: {
          'X-RateLimit-Limit': '2',
          'X-RateLimit-Remaining': '0',
          'X-RateLimit-Reset': Math.floor(resetAt.getTime() / 1000).toString()
        }
      }
    );
  }
  
  // Proceed with submission
  // ...
}
```

### IP-Based Rate Limiting (Vercel)

**Vercel Edge Middleware:**

Prevent brute force and API abuse from single IPs:

```typescript
// middleware.ts (runs on Vercel Edge)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

// Configure rate limiter (requires Upstash Redis or similar)
const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(100, '15 m'), // 100 requests per 15 minutes
  analytics: true,
});

export async function middleware(request: NextRequest) {
  // Only rate limit API routes
  if (!request.nextUrl.pathname.startsWith('/api/')) {
    return NextResponse.next();
  }
  
  // Get client IP
  const ip = request.ip ?? request.headers.get('x-forwarded-for') ?? 'unknown';
  
  // Check rate limit
  const { success, limit, reset, remaining } = await ratelimit.limit(ip);
  
  const response = success 
    ? NextResponse.next() 
    : NextResponse.json(
        {
          error: {
            code: 'TOO_MANY_REQUESTS',
            message: 'Too many requests from this IP. Please try again later.',
            statusCode: 429
          }
        },
        { status: 429 }
      );
  
  // Add rate limit headers
  response.headers.set('X-RateLimit-Limit', limit.toString());
  response.headers.set('X-RateLimit-Remaining', remaining.toString());
  response.headers.set('X-RateLimit-Reset', reset.toString());
  
  return response;
}

export const config = {
  matcher: '/api/:path*',
};
```

**For MVP (Simpler Approach):**

If Upstash Redis is overkill, use Vercel's built-in rate limiting (when available) or skip IP-based limits initially. Focus on per-user analysis limit (more important).

---

## API Key Management

### Environment Variables

**Storage:** Vercel project settings (plain environment variables)

Vercel environment variables are:
- Encrypted at rest in Vercel's infrastructure
- Only accessible to deployment functions (not exposed to frontend)
- Separate per environment (development, preview, production)

**Required API Keys:**

```bash
# .env.example (committed to git)
SUPABASE_URL=https://[project-ref].supabase.co
SUPABASE_ANON_KEY=[public-anon-key]
SUPABASE_SERVICE_ROLE_KEY=[private-service-key]
ANTHROPIC_API_KEY=[claude-api-key]
GOOGLE_AI_API_KEY=[gemini-api-key]
NEXT_PUBLIC_SUPABASE_URL=https://[project-ref].supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=[public-anon-key]
```

**Configuration in Vercel:**

1. Navigate to Project Settings → Environment Variables
2. Add each key with appropriate scope:
   - `SUPABASE_URL` → All environments
   - `SUPABASE_ANON_KEY` → All environments (also needs NEXT_PUBLIC_ variant)
   - `SUPABASE_SERVICE_ROLE_KEY` → Production + Preview (never in Development)
   - `ANTHROPIC_API_KEY` → Production only (use separate dev key for Development)
   - `GOOGLE_AI_API_KEY` → Production only

3. Mark sensitive keys as "Encrypted" (Vercel default)

**Access Control:**

Each API key used by specific function groups:

| Key | Used By | Access Level |
|-----|---------|--------------|
| SUPABASE_ANON_KEY | Frontend + API | Public (read-only via RLS) |
| SUPABASE_SERVICE_ROLE_KEY | Background jobs only | Full access (bypasses RLS) |
| ANTHROPIC_API_KEY | AI orchestration functions | External API |
| GOOGLE_AI_API_KEY | PDF parsing function | External API |

**Security Best Practices:**

```typescript
// ✓ GOOD: Load from environment variables
const apiKey = process.env.ANTHROPIC_API_KEY;
if (!apiKey) {
  throw new Error('ANTHROPIC_API_KEY not configured');
}

// ✗ BAD: Hardcode keys in code
const apiKey = 'sk-ant-api03-...'; // NEVER DO THIS

// ✗ BAD: Expose to frontend
export const config = {
  anthropicKey: process.env.ANTHROPIC_API_KEY // Will be bundled in client JS!
};
```

### Key Rotation Strategy

**Frequency:** Quarterly (every 3 months)

**Process:**

1. **Generate new key** in provider dashboard (Anthropic, Google AI, Supabase)
2. **Add new key** to Vercel environment variables (don't replace old one yet)
3. **Deploy with both keys** configured (old + new)
4. **Update code** to use new key
5. **Deploy again** with new key active
6. **Wait 48 hours** for all in-flight requests to complete
7. **Revoke old key** in provider dashboard
8. **Remove old key** from Vercel environment variables

**For MVP:**
- Manual rotation (set calendar reminder every 3 months)
- No automated rotation needed yet
- Document rotation process in internal wiki

### API Usage Monitoring

**Track usage via cost metrics** (already in database schema):

```sql
-- Query total AI costs per day
SELECT
  DATE(created_at) as date,
  SUM(total_cost_usd) as total_cost,
  COUNT(*) as total_analyses
FROM analyses
WHERE status = 'complete'
  AND created_at >= NOW() - INTERVAL '30 days'
GROUP BY DATE(created_at)
ORDER BY date DESC;
```

**Set up alerts** (future enhancement):
- Daily cost exceeds $50 → Email alert
- Single analysis cost exceeds $5 → Investigate (possible bug)
- Spike in API errors → Check key validity

**For MVP:**
- Manual cost review weekly
- No automated alerts needed initially

---

## Data Encryption

### Encryption at Rest

**Supabase Automatic Encryption:**

All data stored in Supabase is encrypted at rest using AES-256:
- Database tables (PostgreSQL)
- File storage (S3-compatible backend)
- Backups (automated daily backups)

**Configuration:**
- No additional configuration needed
- Encryption handled by Supabase infrastructure
- Keys managed by Supabase (not accessible to users)

**Verification:**
- Check Supabase dashboard → Database → Settings → Encryption status
- Should show "Enabled" with AES-256

### Encryption in Transit

**TLS Configuration:**

All connections use TLS 1.2 or higher:

**Frontend ↔ API:**
- Enforced via HSTS header (see Security Headers section)
- Vercel automatically provisions SSL certificates (Let's Encrypt)
- Auto-renews certificates before expiry

**API ↔ Supabase:**
- Supabase SDK uses HTTPS for all requests
- Connection string: `https://[project-ref].supabase.co` (not http)

**API ↔ External Services (Anthropic, Google AI):**
- All provider SDKs use HTTPS by default
- Verify in code:

```typescript
// Anthropic SDK
const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
  // baseURL defaults to https://api.anthropic.com (secure)
});

// Google AI SDK
const model = genai.getGenerativeModel({
  // SDK enforces HTTPS for all requests
});
```

**TLS Version Enforcement:**

Vercel enforces TLS 1.2+ by default (no configuration needed).

To verify:
```bash
curl -I https://goalos.app
# Should show HTTP/2 (which requires TLS 1.2+)
```

### Field-Level Encryption

**For MVP: NOT IMPLEMENTED**

**Reasoning:**
- Supabase encryption at rest is sufficient for MVP
- CliftonStrengths data is not highly sensitive (users voluntarily upload)
- Adds significant complexity (key management, query limitations)

**Future Consideration:**

If user demand or compliance requires field-level encryption:

```typescript
// Hypothetical implementation (not MVP)
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

const ENCRYPTION_KEY = Buffer.from(process.env.FIELD_ENCRYPTION_KEY, 'hex'); // 32 bytes

function encrypt(text: string): { encrypted: string; iv: string } {
  const iv = randomBytes(16);
  const cipher = createCipheriv('aes-256-cbc', ENCRYPTION_KEY, iv);
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  return { encrypted, iv: iv.toString('hex') };
}

function decrypt(encrypted: string, ivHex: string): string {
  const iv = Buffer.from(ivHex, 'hex');
  const decipher = createDecipheriv('aes-256-cbc', ENCRYPTION_KEY, iv);
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}
```

**Not needed for MVP.**

---

## Security Headers

### HTTP Security Headers

All responses include security headers via Vercel configuration.

**Configuration:** `vercel.json`

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "Strict-Transport-Security",
          "value": "max-age=31536000; includeSubDomains"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        },
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "Referrer-Policy",
          "value": "strict-origin-when-cross-origin"
        },
        {
          "key": "Permissions-Policy",
          "value": "geolocation=(), microphone=(), camera=()"
        }
      ]
    },
    {
      "source": "/api/(.*)",
      "headers": [
        {
          "key": "X-Request-ID",
          "value": "req_{{requestId}}"
        }
      ]
    }
  ]
}
```

**Header Explanations:**

| Header | Value | Purpose |
|--------|-------|---------|
| **Strict-Transport-Security** | max-age=31536000; includeSubDomains | Force HTTPS for 1 year, including all subdomains |
| **X-Frame-Options** | DENY | Prevent clickjacking by disallowing iframe embedding |
| **X-Content-Type-Options** | nosniff | Prevent MIME type sniffing (force browsers to respect Content-Type) |
| **Referrer-Policy** | strict-origin-when-cross-origin | Only send origin (not full URL) on cross-origin requests |
| **Permissions-Policy** | geolocation=(), microphone=(), camera=() | Disable unnecessary browser features |

### Content Security Policy (CSP)

**For MVP: Moderate Policy**

Allows Google Fonts, Supabase CDN, and inline scripts (Next.js requires some inline):

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "Content-Security-Policy",
          "value": "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; img-src 'self' data: https://*.supabase.co; connect-src 'self' https://*.supabase.co https://api.anthropic.com; frame-ancestors 'none';"
        }
      ]
    }
  ]
}
```

**Directive Breakdown:**

- `default-src 'self'` → Only load resources from same origin by default
- `script-src 'self' 'unsafe-inline' 'unsafe-eval'` → Allow inline scripts (Next.js requirement), self-hosted scripts
- `style-src 'self' 'unsafe-inline' https://fonts.googleapis.com` → Allow inline styles, Google Fonts CSS
- `font-src 'self' https://fonts.gstatic.com` → Allow Google Fonts files
- `img-src 'self' data: https://*.supabase.co` → Allow images from self, data URIs, Supabase Storage
- `connect-src 'self' https://*.supabase.co https://api.anthropic.com` → Allow API calls to Supabase, Anthropic
- `frame-ancestors 'none'` → Prevent embedding in iframes (redundant with X-Frame-Options, but more specific)

**Note:** CSP in report-only mode initially to test:

```
Content-Security-Policy-Report-Only: ... (same directives)
```

Monitor violations via browser console, then enforce after testing.

---

## Error Handling & Information Disclosure

### Error Response Strategy

**For MVP: Detailed Errors (Debuggability Priority)**

User explicitly requested full error details for easier debugging during MVP phase.

**Post-MVP:** Hide internal errors in production, show detailed errors only in development.

**Current Implementation:**

```typescript
// API error handler
function handleError(error: unknown, requestId: string): Response {
  console.error(`[${requestId}]`, error);
  
  // Determine error type and construct response
  if (error instanceof ZodError) {
    // Validation error - show field details
    return Response.json(
      {
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Invalid input data',
          statusCode: 400,
          details: error.flatten(),
          requestId
        }
      },
      { status: 400 }
    );
  }
  
  if (error instanceof AuthError) {
    // Authentication error
    return Response.json(
      {
        error: {
          code: 'UNAUTHORIZED',
          message: error.message, // Show actual auth error
          statusCode: 401,
          requestId
        }
      },
      { status: 401 }
    );
  }
  
  if (error instanceof DatabaseError) {
    // Database error - show full details for MVP
    return Response.json(
      {
        error: {
          code: 'DATABASE_ERROR',
          message: 'Database operation failed',
          statusCode: 500,
          details: {
            dbMessage: error.message, // Full DB error
            dbCode: error.code,
            table: error.table
          },
          requestId
        }
      },
      { status: 500 }
    );
  }
  
  // Generic error
  return Response.json(
    {
      error: {
        code: 'INTERNAL_ERROR',
        message: 'An unexpected error occurred',
        statusCode: 500,
        details: {
          errorType: error?.constructor?.name,
          errorMessage: error?.message,
          stack: error?.stack // Full stack trace for MVP
        },
        requestId
      }
    },
    { status: 500 }
  );
}
```

**Logging:**

All errors logged to Vercel Functions logs:

```typescript
console.error('[Security Event]', {
  type: 'authentication_failure',
  userId: userId || 'unknown',
  ip: request.ip,
  timestamp: new Date().toISOString(),
  error: error.message
});
```

**Post-MVP Security Hardening:**

Replace detailed errors with generic messages in production:

```typescript
// Future production error handler
if (process.env.NODE_ENV === 'production') {
  // Hide internal details from users
  return Response.json(
    {
      error: {
        code: 'INTERNAL_ERROR',
        message: 'An unexpected error occurred. Please try again.',
        statusCode: 500,
        requestId // Keep request ID for user support
      }
    },
    { status: 500 }
  );
} else {
  // Show full details in development
  return Response.json({ error: { ...fullDetails } });
}
```

### Security Event Logging

**Events to Log:**

1. **Authentication Events:**
   - Successful login (user ID, timestamp)
   - Failed login (email, reason, IP)
   - Logout (user ID, timestamp)
   - Session expiry (user ID, timestamp)

2. **Authorization Events:**
   - Access denied (user ID, resource, reason)
   - RLS policy violation (user ID, table, operation)

3. **Rate Limit Events:**
   - Rate limit exceeded (user ID, limit type, IP)
   - Suspicious activity (multiple failures from same IP)

4. **Data Events:**
   - Analysis submission (user ID, analysis ID, cost)
   - Analysis deletion (user ID, analysis ID)
   - File upload (user ID, file size, file type)

5. **Error Events:**
   - 500 errors (request ID, error type, stack trace)
   - Validation failures (request ID, fields)
   - External API failures (provider, error code)

**Log Format:**

```typescript
interface SecurityLog {
  timestamp: string; // ISO 8601
  level: 'info' | 'warn' | 'error';
  event: string; // 'auth_success', 'rate_limit_exceeded', etc.
  userId?: string;
  ip?: string;
  requestId?: string;
  details: Record<string, any>;
}

// Example
console.log(JSON.stringify({
  timestamp: new Date().toISOString(),
  level: 'warn',
  event: 'rate_limit_exceeded',
  userId: 'user-123',
  ip: '203.0.113.45',
  requestId: 'req_abc123',
  details: {
    limit: 2,
    used: 3,
    resetAt: '2026-02-16T10:30:00Z'
  }
}));
```

**Retention:**
- Vercel Functions logs: 7 days (free tier)
- Upgrade to Vercel Pro: 30 days
- For longer retention: Stream logs to external service (future)

---

## Privacy & GDPR Compliance

### Data Collection & Storage

**Data Collected:**

| Data Type | Source | Purpose | Retention |
|-----------|--------|---------|-----------|
| Email, Name, Avatar | Google OAuth | User identification | Until user deletion |
| CliftonStrengths PDF | User upload | Generate recommendations | Until user deletion |
| Weekly time investment | User input (optional) | Personalize recommendations | Until user deletion |
| Questionnaire responses | User input (optional) | Personalize recommendations | Until user deletion |
| Goals (title, priority, deadline) | User input | Generate strategies | Until user deletion |
| Generated PDF recommendations | AI output | Deliver to user | Until user deletion |
| AI agent outputs | AI processing | Debugging (temporary) | 7 days auto-delete |
| Session cookies | Supabase Auth | Authentication | 30 days (auto-expire) |
| Usage analytics (future) | User actions | Product improvement | Anonymized, 90 days |

### User Rights (GDPR)

**Right to Access:**

User can view all their data via:
- Account settings page (future)
- API endpoint: `GET /api/v1/users/me/data` (future)

Returns JSON export of:
- Profile data
- All analyses (including goals, CliftonStrengths data)
- Generated PDFs

**Right to Erasure:**

User can delete account via:
- Account settings → "Delete Account" button (future)
- API endpoint: `DELETE /api/v1/users/me`

**Deletion Process:**

```typescript
// DELETE /api/v1/users/me
export async function DELETE(req: Request) {
  const auth = await validateSession(req);
  
  // 1. Delete all user's analyses (cascade to goals, agent outputs)
  await supabase
    .from('analyses')
    .delete()
    .eq('user_id', auth.userId);
  
  // 2. Delete all uploaded PDFs
  const { data: files } = await supabase.storage
    .from('analysis-uploads')
    .list(`${auth.userId}/`);
  
  if (files) {
    await supabase.storage
      .from('analysis-uploads')
      .remove(files.map(f => `${auth.userId}/${f.name}`));
  }
  
  // 3. Delete generated PDFs
  const { data: generatedFiles } = await supabase.storage
    .from('generated-pdfs')
    .list(`${auth.userId}/`);
  
  if (generatedFiles) {
    await supabase.storage
      .from('generated-pdfs')
      .remove(generatedFiles.map(f => `${auth.userId}/${f.name}`));
  }
  
  // 4. Delete user profile
  await supabase
    .from('users')
    .delete()
    .eq('id', auth.userId);
  
  // 5. Delete auth account (Supabase Auth)
  await supabase.auth.admin.deleteUser(auth.userId);
  
  // 6. Log deletion event
  console.log(JSON.stringify({
    event: 'user_deleted',
    userId: auth.userId,
    timestamp: new Date().toISOString()
  }));
  
  return Response.json({ data: { message: 'Account deleted successfully' } });
}
```

**Cascading Deletes:**

Database foreign keys with `ON DELETE CASCADE` ensure all related data is removed:
- User deleted → All analyses deleted → All goals deleted → All agent outputs deleted

**Right to Data Portability:**

**For MVP: NOT IMPLEMENTED**

User can download PDFs manually (fulfills primary use case).

**Future:** Export all data as JSON + ZIP of PDFs.

### Privacy Policy Requirements

**Minimum Disclosures:**

1. **Data Collection:**
   - What: Email, name, avatar (Google OAuth), CliftonStrengths PDF, time investment, questionnaire responses, goals
   - Why: Generate personalized goal strategies
   - How: User uploads PDF, fills forms

2. **Data Usage:**
   - Generate AI recommendations via Anthropic Claude API
   - Parse PDFs via Google Gemini API
   - No third-party marketing or selling

3. **Data Storage:**
   - Supabase (USA/EU regions, specify)
   - Encrypted at rest (AES-256)
   - Encrypted in transit (TLS 1.2+)

4. **Data Retention:**
   - Keep until user requests deletion
   - Auto-delete AI processing outputs after 7 days

5. **User Rights:**
   - Access your data
   - Delete your account
   - Download PDFs

6. **Cookies:**
   - Session cookies (necessary for authentication)
   - No tracking cookies

**Cookie Consent:**

**For MVP: Implied Consent**

Session cookies are necessary for authentication (not optional). Implied consent via Google OAuth flow is sufficient.

**Display:**
- Small banner on landing page: "We use necessary cookies for authentication. By signing in, you agree to our Privacy Policy."
- No blocking cookie consent modal needed

**Post-MVP:** If adding analytics cookies, implement proper consent management.

---

## OWASP Top 10 Compliance

### A01: Broken Access Control

**Mitigations:**
- ✓ Row Level Security (RLS) on all user data tables
- ✓ Session validation on every API request
- ✓ User ID extracted from validated session (never trusted from client)
- ✓ Foreign key constraints enforce ownership cascades
- ✓ Service role key isolated to background jobs (not exposed to users)

**Testing:**
- [ ] Verify users cannot access other users' analyses (RLS test)
- [ ] Verify users cannot bypass authentication (direct DB query)
- [ ] Verify admin endpoints require service role key

### A02: Cryptographic Failures

**Mitigations:**
- ✓ TLS 1.2+ enforced via HSTS header
- ✓ Supabase encryption at rest (AES-256)
- ✓ HTTP-only cookies (prevent JavaScript access)
- ✓ Secure cookie attribute (HTTPS only)
- ✓ No sensitive data in URLs (use POST bodies)

**Testing:**
- [ ] Verify HTTPS redirect works (http → https)
- [ ] Verify session cookies have Secure, HttpOnly attributes
- [ ] Verify database encryption status in Supabase dashboard

### A03: Injection

**Mitigations:**
- ✓ Parameterized queries only (Supabase SDK)
- ✓ Input validation via Zod schemas
- ✓ No raw SQL string concatenation
- ✓ PDF text extraction (no code execution)

**Testing:**
- [ ] SQL injection test: Try `'; DROP TABLE users; --` in inputs
- [ ] Command injection test: Try shell metacharacters in filenames
- [ ] XSS test: Try `<script>alert(1)</script>` in text fields

### A04: Insecure Design

**Threat Model:**

| Threat | Mitigation | Status |
|--------|------------|--------|
| Malicious PDF exploits parser | Isolated parsing, timeout, no outbound requests | ✓ |
| User uploads oversized files | 10MB hard limit, magic byte validation | ✓ |
| Brute force rate limit bypass | Rolling 24h window, DB-enforced | ✓ |
| CSRF via cross-origin requests | SameSite cookies, Origin header check | ✓ |

**Security Testing Checklist:**
- [ ] Upload malformed PDF → Should fail gracefully
- [ ] Upload 11MB PDF → Should reject with clear error
- [ ] Submit 3 analyses in 24h → Should hit rate limit
- [ ] Cross-origin POST request → Should be blocked

### A05: Security Misconfiguration

**Mitigations:**
- ✓ Security headers via Vercel config
- ✓ Environment variables for secrets (not hardcoded)
- ✓ Database default deny (RLS enabled on all tables)
- ✓ Supabase service role key restricted to backend only
- ✓ npm audit run weekly (dependency scanning)

**Testing:**
- [ ] Verify security headers present: `curl -I https://goalos.app`
- [ ] Verify no secrets in git history: `git log -p | grep -i "api_key"`
- [ ] Verify RLS enabled: `SELECT tablename FROM pg_tables WHERE rowsecurity = false;`

### A06: Vulnerable Components

**Mitigations:**
- ✓ npm audit run weekly
- ✓ Dependabot enabled on GitHub (auto PR for updates)
- ✓ Pin exact versions in package-lock.json
- ✓ Review changelogs before major version upgrades

**Testing:**
- [ ] Run `npm audit` → Should show 0 high/critical vulnerabilities
- [ ] Check GitHub Security tab → Should show no open alerts

### A07: Identification & Authentication Failures

**Mitigations:**
- ✓ Supabase Auth (managed, battle-tested)
- ✓ Google OAuth only (no password vulnerabilities)
- ✓ Session timeout (1h access, 30d refresh)
- ✓ Auto-refresh via SDK (transparent to user)
- ✓ Logout invalidates session

**Testing:**
- [ ] Verify session expires after 1 hour (check token)
- [ ] Verify logout clears cookies and invalidates session
- [ ] Verify expired session returns 401 (not 500)

### A08: Software & Data Integrity Failures

**Mitigations:**
- ✓ Supabase automatic backups (daily, encrypted)
- ✓ Point-in-time recovery available (Supabase feature)
- ✓ npm integrity checks (package-lock.json)
- ✓ Vercel build logs (tamper-evident)

**Testing:**
- [ ] Verify Supabase backups enabled: Dashboard → Database → Backups
- [ ] Verify package-lock.json committed to git
- [ ] Test restore from backup (manual test)

### A09: Security Logging & Monitoring Failures

**Mitigations:**
- ✓ All security events logged (auth, rate limits, errors)
- ✓ Request IDs for traceability
- ✓ Vercel Functions logs (7 days retention)
- ✓ Supabase Auth logs (login attempts, failures)

**Testing:**
- [ ] Trigger failed login → Verify logged in Vercel dashboard
- [ ] Trigger rate limit → Verify logged with user ID
- [ ] Trigger 500 error → Verify stack trace logged

### A10: Server-Side Request Forgery (SSRF)

**Not Applicable for MVP**

Our app does not:
- Fetch URLs from user input
- Make outbound requests based on PDF content
- Proxy external resources

**If future features require:**
- Whitelist allowed domains
- Validate URLs before fetching
- Disable outbound requests from PDF parsing context

---

## Security Testing Checklist

### Pre-Launch Security Review

**Authentication & Session Management:**
- [ ] Google OAuth flow works end-to-end
- [ ] Session cookies are HttpOnly, Secure, SameSite=Lax
- [ ] Session expires after 1 hour (access token)
- [ ] Refresh token auto-renews seamlessly
- [ ] Logout clears all cookies and invalidates session
- [ ] Expired session redirects to login (no 500 errors)

**Authorization & Access Control:**
- [ ] Users can only view their own analyses (RLS test)
- [ ] Users cannot access other users' analyses via direct API calls
- [ ] Background jobs can access all data (service role key)
- [ ] RLS policies block unauthorized SELECT, INSERT, UPDATE, DELETE

**Input Validation:**
- [ ] All API endpoints validate request bodies (Zod)
- [ ] Invalid JSON returns 400 with clear error
- [ ] Missing required fields returns 400 with field details
- [ ] SQL injection attempts are blocked (parameterized queries)
- [ ] XSS attempts are escaped (React auto-escaping)

**File Upload Security:**
- [ ] PDF upload requires valid session
- [ ] Oversized files (>10MB) are rejected
- [ ] Non-PDF files are rejected (magic byte check)
- [ ] Malformed PDFs fail gracefully (no crashes)
- [ ] Uploaded files are isolated per user (RLS on storage)

**Rate Limiting:**
- [ ] 2 analyses per 24 hours enforced at API level
- [ ] Rate limit check happens before processing (pre-flight)
- [ ] Rate limit exceeded returns 429 with resetAt timestamp
- [ ] IP-based rate limiting works (100 req/15min)

**Security Headers:**
- [ ] HSTS header present on all responses
- [ ] X-Frame-Options: DENY present
- [ ] X-Content-Type-Options: nosniff present
- [ ] CSP header blocks unauthorized resources
- [ ] Referrer-Policy prevents URL leakage

**Data Encryption:**
- [ ] All connections use HTTPS (verify with browser)
- [ ] Database encryption at rest enabled (Supabase dashboard)
- [ ] Uploaded PDFs encrypted in storage
- [ ] Generated PDFs encrypted in storage

**Error Handling:**
- [ ] 500 errors include request ID for debugging
- [ ] Database errors are logged (not exposed to users post-MVP)
- [ ] Validation errors show field-level details
- [ ] Rate limit errors show reset time

**Privacy & GDPR:**
- [ ] User can view all their data (account settings)
- [ ] User can delete account (cascades to all data)
- [ ] Privacy policy linked on landing page
- [ ] Cookie notice visible before login

### Penetration Testing (Manual)

**Basic Tests (MVP):**

1. **Authentication Bypass:**
   - Try accessing `/api/v1/analyses` without session cookie
   - Expected: 401 Unauthorized

2. **CSRF:**
   - Create malicious site with form targeting `/api/v1/analyses/{id}/submit`
   - Expected: Blocked by SameSite cookie or Origin check

3. **SQL Injection:**
   - Send `analysis_id = "'; DROP TABLE users; --"` in API request
   - Expected: Parameterized query prevents execution

4. **Path Traversal:**
   - Upload PDF with filename `../../../etc/passwd.pdf`
   - Expected: UUID-based naming prevents directory escape

5. **Rate Limit Bypass:**
   - Submit 3 analyses within 1 hour
   - Expected: Third submission returns 429

**Advanced Tests (Post-MVP):**
- Automated security scanning (OWASP ZAP, Burp Suite)
- Third-party penetration testing
- Bug bounty program

---

## Incident Response

### Security Incident Classification

**Severity Levels:**

| Level | Definition | Examples | Response Time |
|-------|------------|----------|---------------|
| **Critical** | Active exploit, data breach | User data exposed, API keys leaked | Immediate (15min) |
| **High** | Vulnerability discovered, no active exploit | Unpatched dependency with CVE | 4 hours |
| **Medium** | Security misconfiguration, no immediate risk | Missing security header | 24 hours |
| **Low** | Minor security improvement | Update documentation | 1 week |

### Incident Response Plan

**Step 1: Detection**

Sources:
- User reports (email, Telegram community)
- Automated alerts (Vercel, Supabase)
- Security scanning (npm audit, GitHub Dependabot)
- Log analysis (unusual activity patterns)

**Step 2: Containment (Critical/High Only)**

Actions:
1. **Stop the bleeding:**
   - Disable affected API endpoints (via Vercel environment variables)
   - Revoke compromised API keys
   - Force logout all users (if auth compromise)

2. **Preserve evidence:**
   - Export Vercel logs
   - Export Supabase audit logs
   - Screenshot affected systems

3. **Communicate:**
   - Email all affected users (if data breach)
   - Post to Telegram community (if service disruption)

**Step 3: Investigation**

Questions:
- What was compromised? (user data, API keys, code)
- How was the attack vector? (known vulnerability, config error, social engineering)
- How many users affected?
- What data was accessed/modified?

**Step 4: Remediation**

Actions:
1. Patch vulnerability (code fix, dependency update, config change)
2. Rotate all API keys (even if not compromised)
3. Force password reset (if auth system compromised)
4. Restore from backup (if data corruption)

**Step 5: Recovery**

Actions:
1. Deploy patched code
2. Re-enable disabled endpoints
3. Monitor for further issues (24-48h intensive monitoring)

**Step 6: Post-Mortem**

Document:
- Timeline of events
- Root cause analysis
- What went well / What went wrong
- Action items to prevent recurrence

**Communication Template (Data Breach):**

```
Subject: Important Security Update - Action Required

Dear [User],

We are writing to inform you of a security incident affecting your Goal Achievement account.

**What Happened:**
[Brief description of incident]

**What Data Was Affected:**
[Specific data types: email, CliftonStrengths data, etc.]

**What We're Doing:**
- [Immediate actions taken]
- [Long-term preventions]

**What You Should Do:**
- Review your account activity at [link]
- Change your password (if applicable)
- Monitor for suspicious activity

**Questions:**
Contact us at security@goalos.app or our Telegram community.

We sincerely apologize for this incident and are committed to protecting your data.

The Personal Time OS Team
```

---

## Appendix A: Security Checklist for Claude Code

When implementing with Claude Code, ensure:

### Authentication & Authorization
- [ ] Supabase Auth configured with Google OAuth
- [ ] Session validation middleware on all protected routes
- [ ] User ID extracted from session (never client input)
- [ ] RLS policies enabled on all tables
- [ ] Service role key restricted to backend only

### Input Validation
- [ ] Zod schemas for all request bodies
- [ ] Magic byte validation for PDF uploads
- [ ] File size limits enforced (10MB)
- [ ] SQL injection prevention (parameterized queries)

### Rate Limiting
- [ ] 2 analyses per 24h enforced at API level
- [ ] Pre-flight rate limit check before processing
- [ ] IP-based rate limiting (100 req/15min)

### Security Headers
- [ ] HSTS header (max-age=31536000)
- [ ] X-Frame-Options: DENY
- [ ] X-Content-Type-Options: nosniff
- [ ] CSP header configured
- [ ] Referrer-Policy header

### Error Handling
- [ ] All errors include request ID
- [ ] Detailed errors in MVP (per user request)
- [ ] Security events logged to console

### API Key Management
- [ ] All secrets in Vercel environment variables
- [ ] No hardcoded API keys in code
- [ ] Service role key never exposed to frontend

### File Upload Security
- [ ] Signed URLs for uploads (15min expiry)
- [ ] UUID-based filenames (no user input)
- [ ] Storage RLS policies (users own folder)

### Testing
- [ ] Authentication bypass test
- [ ] CSRF test
- [ ] SQL injection test
- [ ] Rate limit test
- [ ] RLS policy test

---

**Document Status:** Complete  
**Dependencies:** Database Schema (#2), API Specification (#6)  
**Next Document:** Frontend Architecture & State Management (#8)  
**Implementation Ready:** Yes - comprehensive security specification for Claude Code

---

*This Security & Authentication Specification provides complete implementation details for building a secure Goal Achievement MVP with defense-in-depth security architecture. All policies, headers, validation rules, and testing procedures are defined with sufficient detail for autonomous code generation with Claude Code.*
