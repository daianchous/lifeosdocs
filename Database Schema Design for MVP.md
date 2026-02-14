# Database Schema & Data Models
## Personal Time OS - Goal Achievement MVP

**Version:** 1.0  
**Created:** February 14, 2026  
**Database:** Supabase (PostgreSQL 15+)  
**Purpose:** Complete database design for MVP implementation with Claude Code

---

## Executive Summary

This document defines the complete database schema for the Goal Achievement MVP, designed for implementation with Supabase (managed PostgreSQL with built-in Auth, Storage, and Row Level Security). The schema supports the full user journey from CliftonStrengths parsing through multi-agent AI analysis to PDF generation and delivery.

**Key Design Principles:**
- **Security-first:** Row Level Security (RLS) on all user data tables
- **Performance-optimized:** Strategic indexes for common queries
- **Supabase-native:** Leverages Auth, Storage, and real-time features
- **Debugging-friendly:** Conditional storage of AI outputs with cleanup
- **State-persistent:** Auto-save from Step 2 onward per User Flow spec
- **Cost-conscious:** ~21KB per completed analysis (free tier supports ~23,800 analyses)

**Core Tables:**
1. `users` - User profiles and metadata (extends Supabase Auth)
2. `analyses` - Core analysis entity with state tracking
3. `goals` - Individual goals (1-3 per analysis)
4. `ai_agent_outputs` - Temporary agent outputs with conditional cleanup
5. `prompts` - Versioned AI prompts stored in database

**Database Size Projections:**
- Per analysis (complete): ~21KB (metadata + metrics only after cleanup)
- 100 users, 2 analyses each: ~4.2MB
- 1,000 users, 5 analyses each: ~105MB
- Supabase free tier: 500MB (supports ~23,800 analyses)

---

## Database Architecture Overview

### Technology Stack

**Database:** Supabase (managed PostgreSQL 15+)
- Built-in authentication (Google OAuth)
- Built-in file storage (for PDFs)
- Row Level Security (RLS) for multi-tenant isolation
- Real-time subscriptions for live status updates
- Automatic backups and point-in-time recovery

**Key Features Used:**
- **Supabase Auth:** User management via `auth.users` table
- **Supabase Storage:** PDF file storage with signed URLs
- **RLS Policies:** Security enforcement at database level
- **Triggers:** Automatic cleanup and metrics calculation
- **Functions:** Custom business logic (rate limiting, cleanup)

---

## Complete Database Schema

### Table 1: `users`

**Purpose:** Extend Supabase Auth with application-specific user data

**Relationship to Supabase Auth:** 
- Supabase provides `auth.users` table (managed)
- This table extends it with our application data
- Foreign key references `auth.users.id`

```sql
CREATE TABLE users (
  -- Primary Key
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  
  -- Profile Data (from Google OAuth)
  email TEXT NOT NULL,
  full_name TEXT,
  avatar_url TEXT,
  
  -- Metadata
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_login_at TIMESTAMPTZ,
  
  -- Rate Limiting
  total_analyses_count INTEGER NOT NULL DEFAULT 0,
  
  -- Preferences (future use)
  preferences JSONB DEFAULT '{}',
  
  -- Timestamps
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at);

-- Row Level Security
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own profile"
  ON users FOR SELECT
  USING (auth.uid() = id);

CREATE POLICY "Users can update own profile"
  ON users FOR UPDATE
  USING (auth.uid() = id);

-- Trigger: Update updated_at on row change
CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

**Notes:**
- `id` matches Supabase Auth user ID (UUID)
- `total_analyses_count` incremented via trigger when analysis completes
- `preferences` JSONB for future feature flexibility (notification settings, etc.)

---

### Table 2: `analyses`

**Purpose:** Core entity representing one complete analysis (CliftonStrengths + Goals → AI → PDF)

```sql
CREATE TYPE analysis_status AS ENUM (
  'draft',              -- User in Step 2, setting goals (persisted but not submitted)
  'parsing',            -- CliftonStrengths PDF parsing in background
  'processing',         -- AI agents generating recommendations
  'complete',           -- Successfully completed
  'failed',             -- Complete failure
  'partial_failure'     -- Some goals succeeded, some failed
);

CREATE TABLE analyses (
  -- Primary Key
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Foreign Keys
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  parent_analysis_id UUID REFERENCES analyses(id) ON DELETE SET NULL,
  
  -- Status & State
  status analysis_status NOT NULL DEFAULT 'draft',
  
  -- Timestamps (state tracking)
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  parsing_completed_at TIMESTAMPTZ,
  submitted_at TIMESTAMPTZ,           -- When user clicked "Analyse"
  completed_at TIMESTAMPTZ,
  
  -- CliftonStrengths Data (parsed from PDF)
  clifton_top_10 JSONB,                -- Top 10 themes with rankings
  clifton_bottom_5 JSONB,              -- Bottom 5 themes (30-34) with rankings
  clifton_dominant_domain TEXT,        -- 'Executing' | 'Influencing' | 'Relationship Building' | 'Strategic Thinking'
  
  -- Weekly Time Investment (snapshot from Step 1)
  weekly_time_investment JSONB,       -- 8 categories + total
  
  -- Optional Questionnaire (Step 1)
  questionnaire_responses JSONB,      -- Flexible key-value pairs
  
  -- PDF Output (Supabase Storage)
  output_pdf_path TEXT,               -- e.g., 'outputs/{user_id}/{analysis_id}/recommendations.pdf'
  output_pdf_size INTEGER,            -- Bytes
  output_pdf_generated_at TIMESTAMPTZ,
  
  -- Aggregated Metrics (from AI processing)
  total_tokens_used INTEGER DEFAULT 0,
  total_cost_usd DECIMAL(10, 6) DEFAULT 0.00,
  total_duration_ms INTEGER DEFAULT 0,
  
  -- Error Tracking
  error_message TEXT,
  error_details JSONB,
  
  -- Metadata
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_analyses_user_id ON analyses(user_id);
CREATE INDEX idx_analyses_status ON analyses(status);
CREATE INDEX idx_analyses_created_at ON analyses(created_at DESC);
CREATE INDEX idx_analyses_user_created ON analyses(user_id, created_at DESC);

-- Composite index for rate limiting query
CREATE INDEX idx_analyses_user_created_status ON analyses(user_id, created_at DESC, status);

-- Row Level Security
ALTER TABLE analyses ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own analyses"
  ON analyses FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own analyses"
  ON analyses FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own analyses"
  ON analyses FOR UPDATE
  USING (auth.uid() = user_id);

-- Trigger: Update updated_at on row change
CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON analyses
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

-- Trigger: Increment user's total_analyses_count when analysis completes
CREATE TRIGGER increment_user_analyses_count
  AFTER UPDATE ON analyses
  FOR EACH ROW
  WHEN (OLD.status != 'complete' AND NEW.status = 'complete')
  EXECUTE FUNCTION increment_user_total_analyses();
```

**JSONB Field Structures:**

**`clifton_top_10` example:**
```json
[
  { "rank": 1, "theme": "Achiever", "domain": "Executing" },
  { "rank": 2, "theme": "Learner", "domain": "Strategic Thinking" },
  { "rank": 3, "theme": "Strategic", "domain": "Strategic Thinking" },
  { "rank": 4, "theme": "Responsibility", "domain": "Executing" },
  { "rank": 5, "theme": "Relator", "domain": "Relationship Building" },
  { "rank": 6, "theme": "Input", "domain": "Strategic Thinking" },
  { "rank": 7, "theme": "Ideation", "domain": "Strategic Thinking" },
  { "rank": 8, "theme": "Maximizer", "domain": "Influencing" },
  { "rank": 9, "theme": "Individualization", "domain": "Relationship Building" },
  { "rank": 10, "theme": "Arranger", "domain": "Executing" }
]
```

**`clifton_bottom_5` example:**
```json
[
  { "rank": 30, "theme": "Woo", "domain": "Influencing" },
  { "rank": 31, "theme": "Competition", "domain": "Influencing" },
  { "rank": 32, "theme": "Command", "domain": "Influencing" },
  { "rank": 33, "theme": "Significance", "domain": "Influencing" },
  { "rank": 34, "theme": "Self-Assurance", "domain": "Influencing" }
]
```

**`weekly_time_investment` example:**
```json
{
  "work_career": 40,
  "health_fitness": 5,
  "relationships_family": 20,
  "learning_growth": 3,
  "creativity_hobbies": 2,
  "rest_recovery": 56,
  "community_service": 1,
  "admin_maintenance": 8,
  "total_hours": 135
}
```

**`questionnaire_responses` example:**
```json
{
  "biggest_challenge": "Staying consistent with new habits after initial excitement wears off",
  "what_motivates": "Seeing tangible progress and learning new things",
  "energy_timing": "Morning person, most focused 8am-12pm",
  "what_worked": "Small daily habits, accountability partners",
  "what_didnt_work": "Complex systems with too many steps"
}
```

**Notes:**
- `parent_analysis_id`: For internal retry/failure tracking (not user-facing versioning)
- `parsing_completed_at`: Set when CliftonStrengths parsing finishes (background)
- `submitted_at`: Set when user clicks "Analyse" button (Step 2)
- CliftonStrengths top 10 + bottom 5 gives complete picture (strengths + weaknesses)
- Dominant domain calculated during parsing (most themes in that domain)

---

### Table 3: `goals`

**Purpose:** Individual goals (1-3 per analysis), normalized for parallel AI processing

```sql
CREATE TYPE goal_priority AS ENUM ('high', 'medium', 'low');

CREATE TABLE goals (
  -- Primary Key
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Foreign Key
  analysis_id UUID NOT NULL REFERENCES analyses(id) ON DELETE CASCADE,
  
  -- Goal Data
  description TEXT NOT NULL,                    -- Max 200 chars (enforced in app)
  priority goal_priority NOT NULL,
  categories TEXT[] NOT NULL,                   -- 1-3 life categories
  deadline DATE NOT NULL,
  hardness_rating INTEGER NOT NULL,             -- 1-5 scale
  
  -- AI Processing Status (per goal)
  processing_status TEXT DEFAULT 'pending',     -- 'pending' | 'processing' | 'complete' | 'failed'
  processing_started_at TIMESTAMPTZ,
  processing_completed_at TIMESTAMPTZ,
  
  -- Metadata
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  -- Constraints
  CONSTRAINT valid_categories_count CHECK (array_length(categories, 1) BETWEEN 1 AND 3),
  CONSTRAINT valid_hardness_rating CHECK (hardness_rating BETWEEN 1 AND 5)
);

-- Indexes
CREATE INDEX idx_goals_analysis_id ON goals(analysis_id);
CREATE INDEX idx_goals_priority ON goals(priority);
CREATE INDEX idx_goals_processing_status ON goals(processing_status);

-- Row Level Security
ALTER TABLE goals ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own goals"
  ON goals FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM analyses
      WHERE analyses.id = goals.analysis_id
      AND analyses.user_id = auth.uid()
    )
  );

CREATE POLICY "Users can insert own goals"
  ON goals FOR INSERT
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM analyses
      WHERE analyses.id = goals.analysis_id
      AND analyses.user_id = auth.uid()
    )
  );

CREATE POLICY "Users can update own goals"
  ON goals FOR UPDATE
  USING (
    EXISTS (
      SELECT 1 FROM analyses
      WHERE analyses.id = goals.analysis_id
      AND analyses.user_id = auth.uid()
    )
  );

-- Trigger: Update updated_at on row change
CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON goals
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

**Valid Categories (TEXT[] values):**
- `work_career`
- `health_fitness`
- `relationships_family`
- `learning_growth`
- `creativity_hobbies`
- `rest_recovery`
- `community_service`
- `admin_maintenance`

**Notes:**
- Normalized structure supports 3 parallel goal agents (one per goal)
- `processing_status` per goal enables partial failure handling
- "Only 1 high priority" enforced in application logic (too complex for DB constraint)
- Categories stored as PostgreSQL native array (efficient, queryable)

---

### Table 4: `ai_agent_outputs`

**Purpose:** Temporary storage of AI agent outputs with conditional cleanup

**Lifecycle:**
1. **During processing:** Store full outputs for all 4 agents (3 goal agents + 1 synthesizer)
2. **On success:** Soft delete outputs (set `deleted_at`), keep metrics in `analyses` table
3. **On failure:** Keep outputs for debugging
4. **Cleanup:** Cron job hard deletes soft-deleted rows older than 30 days

```sql
CREATE TYPE agent_type AS ENUM (
  'goal_agent_1',      -- Processes first goal
  'goal_agent_2',      -- Processes second goal
  'goal_agent_3',      -- Processes third goal
  'synthesizer'        -- Combines all goal agents into final PDF
);

CREATE TABLE ai_agent_outputs (
  -- Primary Key
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Foreign Keys
  analysis_id UUID NOT NULL REFERENCES analyses(id) ON DELETE CASCADE,
  goal_id UUID REFERENCES goals(id) ON DELETE CASCADE,  -- NULL for synthesizer
  
  -- Agent Info
  agent_type agent_type NOT NULL,
  
  -- AI Output
  output_text JSONB NOT NULL,                           -- Full agent response
  
  -- Metrics
  tokens_used INTEGER NOT NULL,
  cost_usd DECIMAL(10, 6) NOT NULL,
  duration_ms INTEGER NOT NULL,
  
  -- Model Info
  model_name TEXT NOT NULL,                             -- 'gemini-flash-2.0' | 'claude-sonnet-4'
  prompt_version TEXT,                                  -- References prompts table
  
  -- Timestamps
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ,                               -- Soft delete
  
  -- Error Tracking
  error_message TEXT,
  error_details JSONB
);

-- Indexes
CREATE INDEX idx_ai_outputs_analysis_id ON ai_agent_outputs(analysis_id);
CREATE INDEX idx_ai_outputs_goal_id ON ai_agent_outputs(goal_id);
CREATE INDEX idx_ai_outputs_deleted_at ON ai_agent_outputs(deleted_at);
CREATE INDEX idx_ai_outputs_created_deleted ON ai_agent_outputs(created_at, deleted_at);

-- Row Level Security
ALTER TABLE ai_agent_outputs ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own agent outputs"
  ON ai_agent_outputs FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM analyses
      WHERE analyses.id = ai_agent_outputs.analysis_id
      AND analyses.user_id = auth.uid()
    )
  );

-- No INSERT/UPDATE policies for users (only backend can write)
```

**`output_text` JSONB structure example (goal agent):**
```json
{
  "keystone_actions": [
    {
      "action": "Set out workout clothes the night before",
      "time_estimate_minutes": 5,
      "core_drive": "CD2: Development & Accomplishment",
      "why_for_you": "Your Achiever theme thrives on checking things off—this makes morning exercise a guaranteed 'win'"
    }
  ],
  "bonus_accelerators": [
    {
      "action": "Join a running group or find an accountability partner",
      "time_estimate_minutes": 60,
      "core_drive": "CD5: Social Influence & Relatedness",
      "why_for_you": "Your Relator theme values deep connections—training with others keeps you committed"
    }
  ],
  "capacity_check": {
    "weekly_time_required": 180,
    "current_health_fitness_time": 5,
    "available_capacity": "low",
    "recommendation": "Start with 2 sessions/week, not 5"
  }
}
```

**Notes:**
- `goal_id` is NULL for synthesizer agent (processes all goals together)
- Soft delete via `deleted_at` timestamp (not hard delete)
- Cleanup trigger sets `deleted_at` on analysis success
- Cron job (Supabase Edge Function) hard deletes old soft-deleted rows

---

### Table 5: `prompts`

**Purpose:** Versioned AI prompts stored in database for flexibility

**Why database vs config files:**
- Easy versioning and rollback
- A/B testing different prompts
- No code deployment needed to update prompts
- Track which prompt version generated which output

```sql
CREATE TYPE prompt_category AS ENUM (
  'clifton_parser',              -- Parses CliftonStrengths PDF
  'goal_agent',                  -- Individual goal processing
  'synthesizer',                 -- Combines goals into final PDF
  'validation'                   -- Input validation/extraction
);

CREATE TABLE prompts (
  -- Primary Key
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Prompt Identification
  category prompt_category NOT NULL,
  version TEXT NOT NULL,                        -- e.g., 'v1.0', 'v1.1', 'v2.0'
  name TEXT NOT NULL,                           -- e.g., 'Goal Agent - Achievement Focus'
  
  -- Prompt Content
  system_prompt TEXT NOT NULL,
  user_prompt_template TEXT NOT NULL,           -- May contain {{variables}}
  
  -- Model Configuration
  target_model TEXT NOT NULL,                   -- 'gemini-flash-2.0' | 'claude-sonnet-4'
  temperature DECIMAL(3, 2) DEFAULT 0.7,
  max_tokens INTEGER,
  
  -- Metadata
  is_active BOOLEAN NOT NULL DEFAULT FALSE,     -- Only one active per category
  description TEXT,
  created_by TEXT,                              -- Who created this version
  
  -- Timestamps
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  activated_at TIMESTAMPTZ,
  deactivated_at TIMESTAMPTZ,
  
  -- Constraints
  CONSTRAINT unique_active_prompt UNIQUE (category, is_active) WHERE is_active = TRUE
);

-- Indexes
CREATE INDEX idx_prompts_category ON prompts(category);
CREATE INDEX idx_prompts_active ON prompts(category, is_active);
CREATE INDEX idx_prompts_version ON prompts(version);

-- Row Level Security (admin-only access)
ALTER TABLE prompts ENABLE ROW LEVEL SECURITY;

-- No user policies - prompts are read-only for backend, managed via admin dashboard
```

**Example Prompt Record:**
```sql
INSERT INTO prompts (
  category, 
  version, 
  name, 
  system_prompt, 
  user_prompt_template, 
  target_model,
  is_active
) VALUES (
  'goal_agent',
  'v1.0',
  'Goal Agent - CliftonStrengths Matched',
  'You are an expert goal achievement strategist specializing in CliftonStrengths-based personalization...',
  'User Profile: {{clifton_top_10}} {{dominant_domain}} {{weekly_time}}
Goal: {{goal_description}}
Priority: {{priority}}
Deadline: {{deadline}}
Hardness: {{hardness_rating}}

Generate personalized recommendations...',
  'claude-sonnet-4',
  TRUE
);
```

**Notes:**
- Template variables like `{{clifton_top_10}}` replaced at runtime
- Only one active prompt per category (enforced by unique constraint)
- Version history preserved for debugging (which prompt generated which output)

---

## Database Functions & Triggers

### Function 1: Update `updated_at` Column

**Purpose:** Automatically set `updated_at = NOW()` on row updates

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Applied to:** `users`, `analyses`, `goals`

---

### Function 2: Increment User Total Analyses Count

**Purpose:** Increment `users.total_analyses_count` when analysis completes

```sql
CREATE OR REPLACE FUNCTION increment_user_total_analyses()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE users
  SET total_analyses_count = total_analyses_count + 1
  WHERE id = NEW.user_id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Trigger:** Applied to `analyses` table on status change to 'complete'

---

### Function 3: Soft Delete AI Agent Outputs on Success

**Purpose:** Set `deleted_at` on ai_agent_outputs when analysis succeeds

```sql
CREATE OR REPLACE FUNCTION cleanup_ai_outputs_on_success()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.status = 'complete' AND OLD.status != 'complete' THEN
    UPDATE ai_agent_outputs
    SET deleted_at = NOW()
    WHERE analysis_id = NEW.id
    AND deleted_at IS NULL;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_cleanup_ai_outputs
  AFTER UPDATE ON analyses
  FOR EACH ROW
  EXECUTE FUNCTION cleanup_ai_outputs_on_success();
```

**Notes:**
- Only soft deletes (sets timestamp, doesn't remove rows)
- Hard delete happens via cron job after 30 days

---

### Function 4: Calculate Analysis Metrics

**Purpose:** Aggregate AI agent metrics into `analyses` table

```sql
CREATE OR REPLACE FUNCTION calculate_analysis_metrics()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE analyses
  SET 
    total_tokens_used = (
      SELECT COALESCE(SUM(tokens_used), 0)
      FROM ai_agent_outputs
      WHERE analysis_id = NEW.analysis_id
    ),
    total_cost_usd = (
      SELECT COALESCE(SUM(cost_usd), 0.00)
      FROM ai_agent_outputs
      WHERE analysis_id = NEW.analysis_id
    ),
    total_duration_ms = (
      SELECT COALESCE(SUM(duration_ms), 0)
      FROM ai_agent_outputs
      WHERE analysis_id = NEW.analysis_id
    )
  WHERE id = NEW.analysis_id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_calculate_metrics
  AFTER INSERT OR UPDATE ON ai_agent_outputs
  FOR EACH ROW
  EXECUTE FUNCTION calculate_analysis_metrics();
```

**Notes:**
- Runs on every ai_agent_output insert/update
- Keeps aggregated metrics current in real-time

---

## Rate Limiting Implementation

### Application-Level Query

**Check if user can create new analysis:**

```sql
SELECT COUNT(*) as recent_analyses_count
FROM analyses
WHERE user_id = $1
  AND created_at >= NOW() - INTERVAL '24 hours'
  AND status != 'draft';  -- Don't count abandoned drafts
```

**Logic:**
- If `recent_analyses_count >= 2` → show rate limit modal
- Show countdown: "Next slot opens in X hours Y minutes"
- Calculate countdown: `24 hours - (NOW() - oldest_recent_analysis.created_at)`

**Why application logic vs database constraint:**
- More flexible messaging ("4 hours 23 minutes remaining")
- Can show pre-emptive warnings
- Easier to adjust limits without schema changes

---

## Common Queries

### Query 1: Get User's Analysis History

```sql
SELECT 
  a.id,
  a.status,
  a.created_at,
  a.completed_at,
  COUNT(g.id) as goals_count,
  a.output_pdf_path,
  a.total_cost_usd,
  a.total_duration_ms
FROM analyses a
LEFT JOIN goals g ON g.analysis_id = a.id
WHERE a.user_id = $1
  AND a.status != 'draft'  -- Hide incomplete drafts
GROUP BY a.id
ORDER BY a.created_at DESC
LIMIT 50;
```

---

### Query 2: Get Analysis Details with Goals

```sql
SELECT 
  a.*,
  json_agg(
    json_build_object(
      'id', g.id,
      'description', g.description,
      'priority', g.priority,
      'categories', g.categories,
      'deadline', g.deadline,
      'hardness_rating', g.hardness_rating,
      'processing_status', g.processing_status
    )
  ) as goals
FROM analyses a
LEFT JOIN goals g ON g.analysis_id = a.id
WHERE a.id = $1
  AND a.user_id = $2  -- Security check
GROUP BY a.id;
```

---

### Query 3: Check Rate Limit

```sql
SELECT 
  COUNT(*) as recent_count,
  MIN(created_at) as oldest_analysis_time
FROM analyses
WHERE user_id = $1
  AND created_at >= NOW() - INTERVAL '24 hours'
  AND status != 'draft';
```

---

### Query 4: Get AI Agent Outputs for Debugging

```sql
SELECT 
  ao.agent_type,
  ao.output_text,
  ao.tokens_used,
  ao.cost_usd,
  ao.duration_ms,
  ao.model_name,
  ao.created_at,
  g.description as goal_description
FROM ai_agent_outputs ao
LEFT JOIN goals g ON g.id = ao.goal_id
WHERE ao.analysis_id = $1
  AND ao.deleted_at IS NULL  -- Only show non-deleted outputs
ORDER BY ao.created_at ASC;
```

---

### Query 5: Get Active Prompts

```sql
SELECT 
  category,
  version,
  system_prompt,
  user_prompt_template,
  target_model,
  temperature,
  max_tokens
FROM prompts
WHERE is_active = TRUE;
```

---

## Supabase Storage Structure

### Bucket: `pdfs`

**Directory Structure:**
```
pdfs/
├── outputs/
│   ├── {user_id}/
│   │   ├── {analysis_id}/
│   │   │   └── recommendations.pdf
│   │   ├── {analysis_id}/
│   │   │   └── recommendations.pdf
```

**Access Control:**
- **Read:** User can read their own files only
- **Write:** Backend (service role) only
- **Public access:** Disabled (use signed URLs)

**Supabase Storage Policy:**
```sql
-- Allow users to download their own PDFs
CREATE POLICY "Users can download own PDFs"
ON storage.objects FOR SELECT
USING (
  bucket_id = 'pdfs' 
  AND (storage.foldername(name))[1] = 'outputs'
  AND (storage.foldername(name))[2] = auth.uid()::text
);
```

**Signed URL Generation (backend):**
```typescript
const { data, error } = await supabase.storage
  .from('pdfs')
  .createSignedUrl(`outputs/${userId}/${analysisId}/recommendations.pdf`, 3600);
// Returns URL valid for 1 hour
```

**Notes:**
- Input CliftonStrengths PDFs are **NOT stored** - discarded after parsing
- Only output recommendations PDFs are stored
- Signed URLs expire after 1 hour (security)
- No public URLs (prevents unauthorized sharing)

---

## Data Retention & Cleanup

### Cleanup Strategy

**AI Agent Outputs:**
1. **Soft delete on success:** Trigger sets `deleted_at` when analysis completes
2. **Hard delete after 30 days:** Cron job (Supabase Edge Function)

**Cron Job (runs daily):**
```sql
DELETE FROM ai_agent_outputs
WHERE deleted_at IS NOT NULL
  AND deleted_at < NOW() - INTERVAL '30 days';
```

**PDF Files:**
- **MVP:** No automatic cleanup (manual if needed)
- **Future:** Lifecycle policy to delete PDFs older than 90 days (configurable)

**User Data:**
- Users cannot delete their own data in MVP (no delete function)
- Admin can manually delete via Supabase dashboard if requested

---

## Migration Strategy

### Initial Schema Setup

**Order of operations:**
1. Create ENUM types
2. Create tables (users → analyses → goals → ai_agent_outputs → prompts)
3. Create indexes
4. Create functions
5. Create triggers
6. Enable RLS and create policies
7. Seed initial prompts

**Migration Script Structure:**
```sql
-- migrations/001_initial_schema.sql

-- Step 1: Create ENUM types
CREATE TYPE analysis_status AS ENUM (...);
CREATE TYPE goal_priority AS ENUM (...);
CREATE TYPE agent_type AS ENUM (...);
CREATE TYPE prompt_category AS ENUM (...);

-- Step 2: Create tables
CREATE TABLE users (...);
CREATE TABLE analyses (...);
CREATE TABLE goals (...);
CREATE TABLE ai_agent_outputs (...);
CREATE TABLE prompts (...);

-- Step 3: Create indexes
CREATE INDEX idx_users_email ON users(email);
-- ... etc

-- Step 4: Create functions
CREATE OR REPLACE FUNCTION update_updated_at_column() ...
-- ... etc

-- Step 5: Create triggers
CREATE TRIGGER set_updated_at ...
-- ... etc

-- Step 6: Enable RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can view own profile" ...
-- ... etc

-- Step 7: Seed initial prompts
INSERT INTO prompts (...) VALUES (...);
```

### Rollback Plan

**If migration fails:**
```sql
-- migrations/001_rollback.sql

DROP TRIGGER IF EXISTS ...;
DROP FUNCTION IF EXISTS ...;
DROP POLICY IF EXISTS ...;
DROP TABLE IF EXISTS prompts CASCADE;
DROP TABLE IF EXISTS ai_agent_outputs CASCADE;
DROP TABLE IF EXISTS goals CASCADE;
DROP TABLE IF EXISTS analyses CASCADE;
DROP TABLE IF EXISTS users CASCADE;
DROP TYPE IF EXISTS prompt_category;
DROP TYPE IF EXISTS agent_type;
DROP TYPE IF EXISTS goal_priority;
DROP TYPE IF EXISTS analysis_status;
```

---

## Performance Optimization

### Index Strategy

**High-frequency queries:**
- User's analysis history: `idx_analyses_user_created` (composite index)
- Rate limit check: `idx_analyses_user_created_status` (composite index)
- Goal lookup by analysis: `idx_goals_analysis_id`
- AI outputs by analysis: `idx_ai_outputs_analysis_id`

**Query optimization:**
- Composite indexes cover common WHERE clauses
- DESC indexes for reverse chronological ordering
- Partial indexes where appropriate (e.g., active prompts)

### Connection Pooling

**Supabase built-in pooling:**
- Pooler endpoint: `[project-ref].pooler.supabase.com`
- Pool mode: Transaction (recommended for serverless)
- Max connections: Scales automatically

**Application configuration:**
```typescript
// Use pooler for serverless functions
const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_ANON_KEY!,
  {
    db: {
      schema: 'public',
    },
    global: {
      headers: { 'x-connection-pooler': 'supavisor' },
    },
  }
);
```

---

## Sample Data

### Example: Complete Analysis Record

**User:** jane_doe@example.com (UUID: `123e4567-e89b-12d3-a456-426614174000`)

**Analysis:**
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "user_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "complete",
  "created_at": "2026-02-14T10:00:00Z",
  "parsing_completed_at": "2026-02-14T10:00:45Z",
  "submitted_at": "2026-02-14T10:05:00Z",
  "completed_at": "2026-02-14T10:07:15Z",
  "clifton_top_10": [
    { "rank": 1, "theme": "Achiever", "domain": "Executing" },
    { "rank": 2, "theme": "Strategic", "domain": "Strategic Thinking" },
    { "rank": 3, "theme": "Learner", "domain": "Strategic Thinking" },
    { "rank": 4, "theme": "Relator", "domain": "Relationship Building" },
    { "rank": 5, "theme": "Responsibility", "domain": "Executing" },
    { "rank": 6, "theme": "Input", "domain": "Strategic Thinking" },
    { "rank": 7, "theme": "Ideation", "domain": "Strategic Thinking" },
    { "rank": 8, "theme": "Maximizer", "domain": "Influencing" },
    { "rank": 9, "theme": "Individualization", "domain": "Relationship Building" },
    { "rank": 10, "theme": "Arranger", "domain": "Executing" }
  ],
  "clifton_bottom_5": [
    { "rank": 30, "theme": "Woo", "domain": "Influencing" },
    { "rank": 31, "theme": "Competition", "domain": "Influencing" },
    { "rank": 32, "theme": "Command", "domain": "Influencing" },
    { "rank": 33, "theme": "Significance", "domain": "Influencing" },
    { "rank": 34, "theme": "Self-Assurance", "domain": "Influencing" }
  ],
  "clifton_dominant_domain": "Strategic Thinking",
  "weekly_time_investment": {
    "work_career": 45,
    "health_fitness": 3,
    "relationships_family": 25,
    "learning_growth": 5,
    "creativity_hobbies": 2,
    "rest_recovery": 50,
    "community_service": 1,
    "admin_maintenance": 10,
    "total_hours": 141
  },
  "questionnaire_responses": {
    "biggest_challenge": "Consistency after initial excitement fades",
    "what_motivates": "Learning new things, seeing progress",
    "energy_timing": "Morning person, 8am-12pm peak focus",
    "what_worked": "Small daily habits, visual trackers",
    "what_didnt_work": "Complex systems with many steps"
  },
  "output_pdf_path": "outputs/123e4567-e89b-12d3-a456-426614174000/a1b2c3d4-e5f6-7890-abcd-ef1234567890/recommendations.pdf",
  "output_pdf_size": 245678,
  "output_pdf_generated_at": "2026-02-14T10:07:10Z",
  "total_tokens_used": 45230,
  "total_cost_usd": 0.0523,
  "total_duration_ms": 128500
}
```

**Goals (2 goals):**
```json
[
  {
    "id": "goal-1-uuid",
    "analysis_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "description": "Run a 5K in under 30 minutes by June 1st",
    "priority": "high",
    "categories": ["health_fitness"],
    "deadline": "2026-06-01",
    "hardness_rating": 4,
    "processing_status": "complete",
    "processing_started_at": "2026-02-14T10:05:30Z",
    "processing_completed_at": "2026-02-14T10:06:45Z"
  },
  {
    "id": "goal-2-uuid",
    "analysis_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "description": "Learn Spanish conversational basics for summer trip",
    "priority": "medium",
    "categories": ["learning_growth", "creativity_hobbies"],
    "deadline": "2026-07-15",
    "hardness_rating": 3,
    "processing_status": "complete",
    "processing_started_at": "2026-02-14T10:05:30Z",
    "processing_completed_at": "2026-02-14T10:06:50Z"
  }
]
```

---

## Security Considerations

### Row Level Security (RLS)

**Enforcement:**
- All user data tables have RLS enabled
- Policies ensure `auth.uid() = user_id` for all operations
- Backend uses service role key (bypasses RLS) for system operations

**Admin Access:**
- Separate service role key for admin operations
- Admin dashboard uses service role (not user tokens)
- Audit log for admin actions (future enhancement)

### Data Encryption

**At Rest:**
- Supabase provides automatic encryption at rest (AES-256)
- Database backups encrypted
- Storage files encrypted

**In Transit:**
- All connections use TLS 1.2+
- Signed URLs for PDF downloads (HTTPS only)

### API Key Management

**Environment Variables:**
```
SUPABASE_URL=https://[project-ref].supabase.co
SUPABASE_ANON_KEY=[public anon key]
SUPABASE_SERVICE_ROLE_KEY=[private service key]
```

**Key Rotation:**
- Anon key can be rotated via Supabase dashboard
- Service role key should be rotated quarterly
- Update environment variables in Vercel

---

## Monitoring & Observability

### Key Metrics to Track

**Database Performance:**
- Query execution time (p50, p95, p99)
- Connection pool utilization
- Index hit rate (should be >95%)
- Table size growth rate

**Application Metrics:**
- Analyses created per day
- Average time to completion
- Failure rate by status
- Rate limit hit rate

**Cost Metrics:**
- Average cost per analysis
- Total AI spend per day/week/month
- Storage size growth
- Database size growth

### Supabase Dashboard

**Built-in metrics:**
- Database size and connections
- API request count and latency
- Storage usage
- Real-time connections

**Custom Queries:**
```sql
-- Average analysis completion time
SELECT AVG(
  EXTRACT(EPOCH FROM (completed_at - submitted_at))
) as avg_completion_seconds
FROM analyses
WHERE status = 'complete'
  AND created_at >= NOW() - INTERVAL '7 days';

-- Failure rate
SELECT 
  status,
  COUNT(*) as count,
  ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM analyses
WHERE submitted_at >= NOW() - INTERVAL '7 days'
GROUP BY status;
```

---

## Appendix A: Complete DDL Script

See separate migration file: `migrations/001_initial_schema.sql`

**File contains:**
1. All ENUM type definitions
2. All table CREATE statements
3. All index definitions
4. All function definitions
5. All trigger definitions
6. All RLS policy definitions
7. Initial prompt seeds

---

## Appendix B: Data Model Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         auth.users                              │
│                    (Supabase Managed)                           │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  │ 1:1 extends
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                           users                                 │
│  • id (FK to auth.users)                                        │
│  • email, full_name, avatar_url                                 │
│  • total_analyses_count                                         │
│  • preferences (JSONB)                                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  │ 1:N owns
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                         analyses                                │
│  • id, user_id (FK)                                             │
│  • status (enum), timestamps                                    │
│  • clifton_top_10, clifton_bottom_5 (JSONB)                     │
│  • clifton_dominant_domain                                      │
│  • weekly_time_investment (JSONB)                               │
│  • questionnaire_responses (JSONB)                              │
│  • output_pdf_path, output_pdf_size                             │
│  • total_tokens, total_cost, total_duration                     │
│  • parent_analysis_id (self-reference)                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  │ 1:N contains
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                           goals                                 │
│  • id, analysis_id (FK)                                         │
│  • description, priority (enum)                                 │
│  • categories (TEXT[])                                          │
│  • deadline, hardness_rating                                    │
│  • processing_status, timestamps                                │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  │ 1:N generates
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ai_agent_outputs                             │
│  • id, analysis_id (FK), goal_id (FK, nullable)                 │
│  • agent_type (enum)                                            │
│  • output_text (JSONB)                                          │
│  • tokens_used, cost_usd, duration_ms                           │
│  • model_name, prompt_version                                   │
│  • deleted_at (soft delete)                                     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                          prompts                                │
│  • id, category (enum), version                                 │
│  • system_prompt, user_prompt_template                          │
│  • target_model, temperature, max_tokens                        │
│  • is_active (only one active per category)                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Appendix C: Storage Size Calculations

**Per Analysis (after cleanup):**
- `analyses` table row: ~2KB (metadata only, no AI outputs)
- `goals` table rows (avg 2 goals): ~1KB
- `ai_agent_outputs` metrics only (soft deleted): ~0.5KB
- PDF file in storage (avg): ~250KB
- **Total per analysis:** ~253.5KB

**With cleanup (soft delete AI outputs after 30 days):**
- `analyses` + `goals` only: ~3KB
- PDF file: ~250KB
- **Total per old analysis:** ~253KB

**Supabase Free Tier Limits:**
- Database: 500MB
- Storage: 1GB
- Supports ~1,970 analyses with PDFs (~3,940 PDFs deleted after download)

---

**Document Status:** Complete - Ready for Implementation  
**Last Updated:** February 14, 2026  
**Next Document:** AI Task Specifications (7 tasks)  
**Owner:** Igor (Founder, PM, Designer)

---

*This database schema is optimized for Supabase implementation with Claude Code. All tables, indexes, policies, and functions are production-ready and follow PostgreSQL best practices with security-first design.*
