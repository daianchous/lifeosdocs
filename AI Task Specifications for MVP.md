# AI Task Specifications
## Personal Time OS - Goal Achievement MVP

**Version:** 1.0  
**Created:** February 14, 2026  
**Purpose:** Complete specifications for all AI tasks in the MVP recommendation engine  
**Target:** Claude Code implementation with clear input/output contracts

---

## Executive Summary

This document defines the specifications for all AI tasks in the Goal Achievement MVP. The system uses a multi-agent architecture with strategic model selection to balance cost and quality: Gemini Flash 2.0 (free tier) handles parsing and validation tasks, while Claude Sonnet 4 powers the core recommendation engine.

**Core AI Tasks:**
1. **CliftonStrengths PDF Parser** (Gemini Flash) - Extracts structured personality data
2. **Goal Agent** (Claude Sonnet 4) - Generates personalized recommendations (1-3 parallel instances)
3. **Synthesizer Agent** (Claude Sonnet 4) - Combines agent outputs into cohesive PDF content

**Supporting Tasks:**
4. **Input Validator** - Pre-processing validation (mix of code + Gemini Flash)
5. **PDF Generator** - Renders final document (pure code, no AI)
6. **Error Handler** - Universal error classification and messaging (pure code)

**Key Architectural Decisions:**
- **Dynamic scaling:** Run 1-3 Goal Agents based on user's goal count
- **Context management:** Summarize long questionnaire responses before passing to agents
- **Retry strategy:** Simple 2-attempt pattern for all AI tasks
- **Cost target:** ~$0.05 per analysis (within free tier limits)
- **Performance target:** ~130 seconds total processing time
- **Prompts:** Stored in database `prompts` table, versioned and retrievable

**Processing Flow:**
```
Step 1: PDF uploaded → Parser extracts CliftonStrengths data
        ↓
Step 2: User sets 1-3 goals → Input Validator checks quality
        ↓
Step 3: Click "Analyse" → Spawn 1-3 parallel Goal Agents
        ↓
Step 4: All agents complete → Synthesizer combines outputs
        ↓
Step 5: PDF Generator renders document → User downloads
```

---

## Core AI Task 1: CliftonStrengths PDF Parser

### Overview

**Purpose:** Extract structured personality data from Gallup CliftonStrengths PDF reports

**Model:** Gemini Flash 2.0 (free tier)

**Execution:** Background task triggered on PDF upload (Step 1), asynchronous

**Target Performance:**
- Latency: <5 seconds
- Cost: ~$0.001 per parse
- Success rate: >95%

### Input Specification

**Function Signature:**
```typescript
async function parseCliftonStrengthsPDF(
  pdfBuffer: Buffer,
  userId: string,
  analysisId: string
): Promise<ParserOutput>
```

**Input Details:**
- `pdfBuffer`: Binary PDF file (max 10MB)
- `userId`: UUID for database updates
- `analysisId`: UUID for tracking this specific analysis

**PDF Source:** Gallup CliftonStrengths report (expected formats: 2020-2026 versions)

### Output Specification

**TypeScript Interface:**
```typescript
interface ParserOutput {
  success: boolean;
  data?: {
    topTenThemes: CliftonTheme[];
    bottomFiveThemes: CliftonTheme[];
    dominantDomain: DominantDomain;
    rawExtraction: string; // Full text extracted for debugging
  };
  error?: {
    code: string;
    message: string;
    details?: any;
  };
  metadata: {
    tokensUsed: number;
    costUSD: number;
    durationMs: number;
    modelName: string;
    promptVersion: string;
  };
}

interface CliftonTheme {
  name: string;          // e.g., "Achiever"
  rank: number;          // 1-34
  description?: string;  // Optional: Short description from PDF
}

type DominantDomain = 
  | "Executing"
  | "Influencing" 
  | "Relationship Building"
  | "Strategic Thinking";
```

**Data Extraction Requirements:**

1. **Top 10 Themes** (Ranked 1-10):
   - Extract theme names exactly as written in PDF
   - Capture numerical rankings
   - Preserve order
   - All 10 must be present (error if <10 found)

2. **Bottom 5 Themes** (Ranked 30-34):
   - Extract theme names for lowest 5 themes
   - Capture rankings 30, 31, 32, 33, 34
   - All 5 must be present (error if <5 found)

3. **Dominant Domain Calculation:**
   - Count themes in each domain from Top 10
   - Domain with most themes = dominant domain
   - **Domain mappings** (reference for AI prompt):
     - **Executing:** Achiever, Arranger, Belief, Consistency, Deliberative, Discipline, Focus, Responsibility, Restorative
     - **Influencing:** Activator, Command, Communication, Competition, Maximizer, Self-Assurance, Significance, Woo
     - **Relationship Building:** Adaptability, Connectedness, Developer, Empathy, Harmony, Includer, Individualization, Positivity, Relator
     - **Strategic Thinking:** Analytical, Context, Futuristic, Ideation, Input, Intellection, Learner, Strategic
   - Tie-breaking: If 2 domains tied, use earliest theme's rank

4. **Raw Extraction:**
   - Store full extracted text for debugging
   - Helps identify parsing failures

### Processing Logic

**Step 1: PDF Text Extraction**
```typescript
// Use PDF parsing library (pdf-parse or similar)
const pdfText = await extractTextFromPDF(pdfBuffer);
```

**Step 2: AI Extraction**
- Send `pdfText` to Gemini Flash with structured extraction prompt
- Prompt retrieved from database: `prompts` table, category = `clifton_parser`
- Request JSON output with schema matching `ParserOutput.data`

**Step 3: Validation**
```typescript
// Validation rules
if (data.topTenThemes.length !== 10) {
  throw new Error("Expected exactly 10 top themes");
}
if (data.bottomFiveThemes.length !== 5) {
  throw new Error("Expected exactly 5 bottom themes");
}
if (!["Executing", "Influencing", "Relationship Building", "Strategic Thinking"].includes(data.dominantDomain)) {
  throw new Error("Invalid dominant domain");
}
// Verify all theme names are valid (against master list of 34 themes)
// Verify rankings are 1-10 for top, 30-34 for bottom
```

**Step 4: Database Update**
```sql
UPDATE analyses
SET 
  clifton_top_10 = $1,
  clifton_bottom_5 = $2,
  clifton_dominant_domain = $3,
  parsing_completed_at = NOW(),
  status = CASE 
    WHEN status = 'parsing' THEN 'draft'
    ELSE status
  END
WHERE id = $4;
```

### Error Handling

**Retry Strategy:**
- On failure: Retry once with same inputs (total: 2 attempts)
- After 2nd failure: Mark analysis status = 'failed', store error

**Error Classifications:**

1. **User Error (4xxx):**
   - `4001`: Not a PDF file
   - `4002`: PDF too large (>10MB)
   - `4003`: Not a CliftonStrengths report (wrong content)
   - `4004`: Corrupted PDF (can't extract text)

2. **AI Error (6xxx):**
   - `6001`: AI model timeout (>5s)
   - `6002`: Invalid AI response (missing fields)
   - `6003`: AI quota exceeded

3. **System Error (5xxx):**
   - `5001`: PDF parsing library failed
   - `5002`: Database update failed

**User-Facing Messages:**
- `4001-4004`: "We couldn't read that PDF. Please upload your CliftonStrengths report from Gallup."
- `6xxx`: "We hit a temporary issue. Please try uploading again."
- `5xxx`: "Something went wrong on our end. Please try again in a moment."

### Validation Rules

**Pre-AI Validation:**
- File is PDF format (MIME type check)
- File size ≤10MB
- PDF is not password-protected
- PDF has extractable text (not just images)

**Post-AI Validation:**
- All required fields present in output
- Exactly 10 top themes + 5 bottom themes
- All theme names match master list
- Rankings in correct ranges (1-10, 30-34)
- No duplicate themes
- Dominant domain is valid enum value

### Example Output

```json
{
  "success": true,
  "data": {
    "topTenThemes": [
      { "name": "Achiever", "rank": 1 },
      { "name": "Learner", "rank": 2 },
      { "name": "Strategic", "rank": 3 },
      { "name": "Futuristic", "rank": 4 },
      { "name": "Ideation", "rank": 5 },
      { "name": "Input", "rank": 6 },
      { "name": "Intellection", "rank": 7 },
      { "name": "Maximizer", "rank": 8 },
      { "name": "Focus", "rank": 9 },
      { "name": "Responsibility", "rank": 10 }
    ],
    "bottomFiveThemes": [
      { "name": "Harmony", "rank": 30 },
      { "name": "Includer", "rank": 31 },
      { "name": "Adaptability", "rank": 32 },
      { "name": "Developer", "rank": 33 },
      { "name": "Positivity", "rank": 34 }
    ],
    "dominantDomain": "Strategic Thinking",
    "rawExtraction": "CliftonStrengths Assessment Results..."
  },
  "metadata": {
    "tokensUsed": 1243,
    "costUSD": 0.00087,
    "durationMs": 3421,
    "modelName": "gemini-2.0-flash-exp",
    "promptVersion": "v1.0"
  }
}
```

---

## Core AI Task 2: Goal Agent

### Overview

**Purpose:** Generate personalized goal achievement strategy based on CliftonStrengths profile

**Model:** Claude Sonnet 4

**Execution:** 1-3 parallel instances (one per user goal), triggered when user clicks "Analyse"

**Target Performance:**
- Latency: <40 seconds per agent
- Cost: ~$0.015 per agent
- Success rate: >90%

**Scaling Logic:**
- User sets 1 goal → Run 1 agent
- User sets 2 goals → Run 2 agents (parallel)
- User sets 3 goals → Run 3 agents (parallel)

### Input Specification

**Function Signature:**
```typescript
async function generateGoalRecommendations(
  goalData: Goal,
  cliftonProfile: CliftonProfile,
  userContext: UserContext,
  analysisId: string
): Promise<GoalAgentOutput>
```

**Input Interfaces:**

```typescript
interface Goal {
  id: string;                    // UUID
  description: string;           // User's goal text
  priority: "high" | "medium" | "low";
  categories: string[];          // 1-3 from: Work, Health, Relationships, etc.
  deadline: Date;
  hardnessRating: 1 | 2 | 3 | 4 | 5;
}

interface CliftonProfile {
  topTenThemes: CliftonTheme[];
  bottomFiveThemes: CliftonTheme[];
  dominantDomain: DominantDomain;
}

interface UserContext {
  weeklyTimeInvestment?: {
    work: number;
    health: number;
    relationships: number;
    learning: number;
    creativity: number;
    rest: number;
    community: number;
    admin: number;
    total: number;
  };
  questionnaireResponses?: {
    biggestChallenge?: string;
    motivations?: string;
    energyPeaks?: string;
    pastSuccesses?: string;
    pastFailures?: string;
  };
}
```

**Context Window Management:**

If `questionnaireResponses` fields exceed 500 characters combined:
1. Summarize using Gemini Flash before passing to Goal Agent
2. Target: Reduce to <200 characters total
3. Preserve key insights, remove redundancy
4. Store original in database, pass summary to agent

### Output Specification

**TypeScript Interface:**
```typescript
interface GoalAgentOutput {
  success: boolean;
  goalId: string;
  data?: {
    // Core Strategy
    strategyOverview: string;           // 2-3 sentences explaining the approach
    whyThisWorks: string;               // Why this strategy fits this person's CliftonStrengths
    
    // Octalysis Mapping
    primaryCoreDrives: number[];        // Array of 1-3 CD numbers (1-8)
    secondaryCoreDrives: number[];      // Array of 0-2 CD numbers
    
    // Tiered Action Plan
    keystoneHabit: ActionItem;          // THE minimal viable starting action
    coreActions: ActionItem[];          // 3-5 essential actions
    bonusAccelerators: ActionItem[];    // 2-3 optional power-ups
    
    // Capacity Validation
    estimatedTimePerWeek: number;       // Hours needed
    capacityWarning?: string;           // Only if overcommitted
    
    // Progress Tracking
    milestones: Milestone[];            // 3-5 checkpoints to deadline
    trackingMethod: string;             // How to track (dot grid, checklist, etc.)
  };
  error?: {
    code: string;
    message: string;
  };
  metadata: {
    tokensUsed: number;
    costUSD: number;
    durationMs: number;
    modelName: string;
    promptVersion: string;
  };
}

interface ActionItem {
  title: string;                   // Short action name
  description: string;             // How to do it
  frequency: string;               // "Daily", "3x/week", "Weekly", etc.
  estimatedTimeMinutes: number;    // Per instance
  coreDrives: number[];            // Which CD1-8 this activates
  strengthsUsed: string[];         // Which CliftonStrengths themes this leverages
  difficultyLevel: 1 | 2 | 3;     // 1=Easy, 2=Moderate, 3=Challenging
}

interface Milestone {
  week: number;                    // Week number from start (1-based)
  description: string;             // What should be achieved
  successCriteria: string;         // How to know you hit it
}
```

### Processing Logic

**Step 1: Load Prompt from Database**
```sql
SELECT system_prompt, user_prompt_template, temperature, max_tokens
FROM prompts
WHERE category = 'goal_agent' 
  AND is_active = true
ORDER BY version DESC
LIMIT 1;
```

**Step 2: Build Prompt Context**
- Inject `CliftonProfile` data (top 10 themes, dominant domain)
- Inject `Goal` details (description, priority, deadline, hardness)
- Inject `UserContext` (time investment, questionnaire - summarized if needed)
- Fill template variables in `user_prompt_template`

**Step 3: Call Claude Sonnet 4 API**
```typescript
const response = await anthropic.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 4000,
  temperature: 0.7,
  system: systemPrompt,
  messages: [
    { role: "user", content: filledUserPrompt }
  ]
});
```

**Step 4: Parse and Validate Response**
- Expect structured JSON matching `GoalAgentOutput.data` schema
- Validate all required fields present
- Validate Core Drives are numbers 1-8
- Validate time estimates are reasonable (not negative, not >168hr/week)
- Validate milestone count (3-5 expected)

**Step 5: Store in Database**
```sql
INSERT INTO ai_agent_outputs (
  analysis_id, goal_id, agent_type, 
  output_text, tokens_used, cost_usd, duration_ms,
  model_name, prompt_version
) VALUES ($1, $2, 'goal_agent', $3, $4, $5, $6, $7, $8);

UPDATE goals
SET 
  processing_status = 'complete',
  processing_completed_at = NOW()
WHERE id = $goalId;
```

### Parallel Execution Pattern

**For 3 goals:**
```typescript
const goalAgentPromises = goals.map(goal => 
  generateGoalRecommendations(goal, cliftonProfile, userContext, analysisId)
);

const results = await Promise.allSettled(goalAgentPromises);

// Handle partial failures
const successfulResults = results.filter(r => r.status === 'fulfilled');
const failedResults = results.filter(r => r.status === 'rejected');

if (successfulResults.length === 0) {
  // Complete failure - mark analysis as failed
  await markAnalysisFailed(analysisId);
} else if (failedResults.length > 0) {
  // Partial failure - mark as partial_failure
  await markAnalysisPartialFailure(analysisId, failedResults);
} else {
  // All succeeded - proceed to Synthesizer
  await invokeSynthesizer(analysisId, successfulResults);
}
```

### Error Handling

**Retry Strategy:**
- On failure: Retry once per goal (total: 2 attempts per agent)
- Failed goals tracked individually
- User sees which goals failed in UI

**Error Classifications:**

1. **AI Error (6xxx):**
   - `6001`: Claude API timeout (>40s)
   - `6002`: Invalid response format (unparseable JSON)
   - `6003`: Missing required fields in response
   - `6004`: Claude API quota exceeded

2. **System Error (5xxx):**
   - `5001`: Database read failure
   - `5002`: Database write failure
   - `5003`: Prompt template not found

**User-Facing Messages:**
- `6001-6003`: "We had trouble creating recommendations for [Goal Name]. Try again?"
- `6004`: "We've hit our API limit. Please try again in a few minutes."
- `5xxx`: "Something went wrong on our end. Please try again."

### Validation Rules

**Post-AI Validation:**
- `strategyOverview` is 50-500 characters
- `primaryCoreDrives` has 1-3 items, all in range 1-8
- `keystoneHabit` is present
- `coreActions` has 3-5 items
- `bonusAccelerators` has 2-3 items
- Each `ActionItem` has valid frequency string
- Each `ActionItem.estimatedTimeMinutes` is 5-180
- `estimatedTimePerWeek` is 0.5-40 hours
- `milestones` has 3-5 items
- No duplicate action titles

### Example Output

```json
{
  "success": true,
  "goalId": "goal-uuid-123",
  "data": {
    "strategyOverview": "Build a consistent 5K running habit by starting with micro-runs and leveraging your Achiever drive for daily completion streaks.",
    "whyThisWorks": "Your Achiever theme thrives on daily progress and completion, making a streak-based approach highly motivating. Your Strategic thinking helps you plan around schedule conflicts.",
    "primaryCoreDrives": [2, 4],
    "secondaryCoreDrives": [3],
    "keystoneHabit": {
      "title": "5-Minute Morning Run",
      "description": "Run for just 5 minutes every morning, no distance goal. Focus on showing up daily.",
      "frequency": "Daily",
      "estimatedTimeMinutes": 15,
      "coreDrives": [2, 4],
      "strengthsUsed": ["Achiever", "Discipline"],
      "difficultyLevel": 1
    },
    "coreActions": [
      {
        "title": "Gradual Distance Increase",
        "description": "Add 1 minute to your run every week until you reach 30 minutes.",
        "frequency": "Weekly progression",
        "estimatedTimeMinutes": 30,
        "coreDrives": [2, 3],
        "strengthsUsed": ["Achiever", "Strategic"],
        "difficultyLevel": 2
      },
      {
        "title": "Track Completion Streak",
        "description": "Mark each day you run on your dot grid. Aim for 66 consecutive days.",
        "frequency": "Daily",
        "estimatedTimeMinutes": 2,
        "coreDrives": [2, 4],
        "strengthsUsed": ["Achiever"],
        "difficultyLevel": 1
      },
      {
        "title": "Pre-Plan Running Routes",
        "description": "Every Sunday, plan your 7 running routes for the week to eliminate decision fatigue.",
        "frequency": "Weekly",
        "estimatedTimeMinutes": 20,
        "coreDrives": [3],
        "strengthsUsed": ["Strategic", "Discipline"],
        "difficultyLevel": 2
      }
    ],
    "bonusAccelerators": [
      {
        "title": "Join Running Community",
        "description": "Find a local running group or virtual community for social accountability.",
        "frequency": "Weekly meetup",
        "estimatedTimeMinutes": 60,
        "coreDrives": [5],
        "strengthsUsed": ["Achiever", "Strategic"],
        "difficultyLevel": 2
      }
    ],
    "estimatedTimePerWeek": 3.5,
    "milestones": [
      {
        "week": 2,
        "description": "14-day completion streak",
        "successCriteria": "14 consecutive days marked on dot grid"
      },
      {
        "week": 6,
        "description": "Comfortable 15-minute runs",
        "successCriteria": "Can run 15 minutes without stopping"
      },
      {
        "week": 10,
        "description": "Habit formation complete (66 days)",
        "successCriteria": "66 consecutive days + running feels automatic"
      },
      {
        "week": 16,
        "description": "First 5K completion",
        "successCriteria": "Complete 5K distance in under 35 minutes"
      }
    ],
    "trackingMethod": "100-dot grid for high-priority goal (mark 1 dot per completed run)"
  },
  "metadata": {
    "tokensUsed": 3421,
    "costUSD": 0.0154,
    "durationMs": 34219,
    "modelName": "claude-sonnet-4-20250514",
    "promptVersion": "v1.2"
  }
}
```

---

## Core AI Task 3: Synthesizer Agent

### Overview

**Purpose:** Combine 1-3 Goal Agent outputs into cohesive PDF-ready content

**Model:** Claude Sonnet 4

**Execution:** Single instance, runs after all Goal Agents complete

**Target Performance:**
- Latency: <30 seconds
- Cost: ~$0.02
- Success rate: >95%

### Input Specification

**Function Signature:**
```typescript
async function synthesizeRecommendations(
  goalAgentOutputs: GoalAgentOutput[],
  cliftonProfile: CliftonProfile,
  userContext: UserContext,
  analysisId: string
): Promise<SynthesizerOutput>
```

**Input Details:**
- `goalAgentOutputs`: Array of 1-3 completed Goal Agent outputs
- `cliftonProfile`: Same CliftonStrengths data passed to Goal Agents
- `userContext`: Same user context (for overarching insights)
- `analysisId`: For database tracking

### Output Specification

**TypeScript Interface:**
```typescript
interface SynthesizerOutput {
  success: boolean;
  data?: {
    // PDF Sections
    executiveSummary: string;              // 3-4 paragraphs: Big picture overview
    cliftonStrengthsOverview: string;      // 2-3 paragraphs: Profile explanation
    overarchingStrategy: string;           // 2-3 paragraphs: How goals connect
    
    // Per-Goal Sections (preserves Goal Agent outputs)
    goals: SynthesizedGoal[];
    
    // Supporting Content
    trackingGuidance: string;              // How to use dot grids, checkpoints
    motivationalClosing: string;           // Encouraging final message
    nextSteps: string[];                   // 3-5 immediate action items
    
    // Metadata for PDF
    generatedDate: string;
    userName: string;
    dominantDomain: string;
    totalEstimatedTimePerWeek: number;
  };
  error?: {
    code: string;
    message: string;
  };
  metadata: {
    tokensUsed: number;
    costUSD: number;
    durationMs: number;
    modelName: string;
    promptVersion: string;
  };
}

interface SynthesizedGoal {
  priority: "high" | "medium" | "low";
  goalDescription: string;
  
  // From Goal Agent (mostly preserved)
  strategyOverview: string;
  whyThisWorks: string;
  primaryCoreDrives: number[];
  
  // Actions (potentially refined for consistency)
  keystoneHabit: ActionItem;
  coreActions: ActionItem[];
  bonusAccelerators: ActionItem[];
  
  milestones: Milestone[];
  estimatedTimePerWeek: number;
  trackingMethod: string;
}
```

### Processing Logic

**Step 1: Load Prompt from Database**
```sql
SELECT system_prompt, user_prompt_template, temperature, max_tokens
FROM prompts
WHERE category = 'synthesizer' 
  AND is_active = true
ORDER BY version DESC
LIMIT 1;
```

**Step 2: Build Synthesis Context**
- Inject all Goal Agent outputs (full JSON)
- Inject CliftonStrengths profile
- Inject user context (weekly time, questionnaire)
- Count goals, identify high-priority goal
- Calculate total estimated time across all goals

**Step 3: Call Claude Sonnet 4 API**
```typescript
const response = await anthropic.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 6000,
  temperature: 0.7,
  system: systemPrompt,
  messages: [
    { role: "user", content: synthesisPrompt }
  ]
});
```

**Step 4: Parse and Validate Response**
- Expect structured JSON matching `SynthesizerOutput.data`
- Validate all sections present
- Validate goal count matches input (1-3)
- Validate action items preserved from Goal Agents
- Validate `nextSteps` has 3-5 items

**Step 5: Store in Database**
```sql
INSERT INTO ai_agent_outputs (
  analysis_id, agent_type, 
  output_text, tokens_used, cost_usd, duration_ms,
  model_name, prompt_version
) VALUES ($1, 'synthesizer', $2, $3, $4, $5, $6, $7);

UPDATE analyses
SET 
  status = 'processing',  -- Will be 'complete' after PDF generation
  total_tokens_used = (SELECT SUM(tokens_used) FROM ai_agent_outputs WHERE analysis_id = $1),
  total_cost_usd = (SELECT SUM(cost_usd) FROM ai_agent_outputs WHERE analysis_id = $1),
  total_duration_ms = $durationMs
WHERE id = $1;
```

### Synthesis Guidelines

**What the Synthesizer Does:**

1. **Creates connective tissue:**
   - Writes executive summary tying all goals together
   - Explains how CliftonStrengths profile influences overall strategy
   - Identifies common themes across goals (e.g., "All three goals leverage your Achiever drive")

2. **Ensures consistency:**
   - Removes contradictory advice across goals
   - Harmonizes tone and voice
   - Makes sure similar concepts use same terminology

3. **Adds meta-insights:**
   - Points out overarching patterns ("You're ambitious - all 3 goals require 10hr/week")
   - Provides capacity reality check
   - Suggests sequencing if goals overlap

4. **Preserves Goal Agent outputs:**
   - Does NOT rewrite action items
   - Does NOT change Core Drive mappings
   - Does NOT alter milestones
   - Only organizes and adds context

**What the Synthesizer Does NOT Do:**

- Generate new recommendations from scratch
- Contradict Goal Agent strategies
- Remove actions (only add connective content)
- Change time estimates or difficulty levels

### Error Handling

**Retry Strategy:**
- On failure: Retry once with same inputs (total: 2 attempts)
- After 2nd failure: Mark analysis as failed

**Error Classifications:**

1. **AI Error (6xxx):**
   - `6001`: Claude API timeout (>30s)
   - `6002`: Invalid response format
   - `6003`: Missing required sections

2. **System Error (5xxx):**
   - `5001`: Database read failure
   - `5002`: Database write failure

**User-Facing Messages:**
- `6xxx`: "We had trouble finalizing your recommendations. Trying again..."
- `5xxx`: "Something went wrong on our end. Please try again."

### Validation Rules

**Post-AI Validation:**
- `executiveSummary` is 200-800 characters
- `goals` array length matches input (1-3)
- Each goal has all required fields
- `nextSteps` has 3-5 items
- `totalEstimatedTimePerWeek` ≤ sum of individual goal estimates

### Example Output

```json
{
  "success": true,
  "data": {
    "executiveSummary": "Your CliftonStrengths profile reveals a Strategic Thinking dominant domain, with Achiever as your #1 theme. This combination makes you particularly well-suited for ambitious goals that require both careful planning and consistent execution. The three goals you've set - running a 5K, learning Spanish, and building a side project - all leverage your natural planning abilities while feeding your need for measurable progress. With an estimated 12 hours per week required, these goals are ambitious but achievable given your current schedule.",
    
    "cliftonStrengthsOverview": "Your top 10 themes cluster heavily in Strategic Thinking (5 themes) and Executing (3 themes), with Achiever leading the pack. This means you thrive on intellectual challenges paired with tangible completion. Your Achiever drive creates natural momentum through daily progress tracking, while your Strategic, Futuristic, and Ideation themes help you see the path forward clearly...",
    
    "overarchingStrategy": "The strategy across all three goals follows a consistent pattern: start with micro-commitments (5-minute run, 10-minute Spanish session, 30-minute coding block) to build immediate momentum, then gradually scale. This mirrors your Achiever's need for daily wins while respecting your Strategic mind's preference for measured progression...",
    
    "goals": [
      {
        "priority": "high",
        "goalDescription": "Run a 5K in under 30 minutes",
        "strategyOverview": "Build a consistent 5K running habit by starting with micro-runs and leveraging your Achiever drive for daily completion streaks.",
        "whyThisWorks": "Your Achiever theme thrives on daily progress and completion, making a streak-based approach highly motivating...",
        "primaryCoreDrives": [2, 4],
        "keystoneHabit": { /* ... */ },
        "coreActions": [ /* ... */ ],
        "bonusAccelerators": [ /* ... */ ],
        "milestones": [ /* ... */ ],
        "estimatedTimePerWeek": 3.5,
        "trackingMethod": "100-dot grid (mark 1 dot per run)"
      },
      {
        "priority": "medium",
        "goalDescription": "Achieve conversational Spanish fluency",
        "strategyOverview": "Use spaced repetition and daily micro-practice to build vocabulary while your Learner theme keeps engagement high.",
        /* ... similar structure ... */
        "estimatedTimePerWeek": 4.0,
        "trackingMethod": "Simple checklist (daily practice yes/no)"
      },
      {
        "priority": "low",
        "goalDescription": "Launch a profitable side project",
        "strategyOverview": "Apply your Strategic and Ideation strengths to build systematically, with weekly shipping milestones to satisfy Achiever.",
        /* ... similar structure ... */
        "estimatedTimePerWeek": 4.5,
        "trackingMethod": "Weekly milestone checklist"
      }
    ],
    
    "trackingGuidance": "For your high-priority 5K goal, use the 100-dot grid included on page 8. Mark one dot each day you complete your run. The visual progress will trigger your Achiever's satisfaction with completion. For medium and low-priority goals, simple daily or weekly checklists are sufficient...",
    
    "motivationalClosing": "Your Strategic Thinking dominant domain gives you a powerful advantage: you can see the path before you walk it. Combined with your Achiever drive, you have both the vision and the tenacity to make these goals reality. Start with your keystone habits this week - they're designed to be so small that skipping them feels harder than doing them. Trust the process, track your progress, and watch your momentum build.",
    
    "nextSteps": [
      "Print this PDF and highlight your keystone habits",
      "Set up your tracking systems (dot grid for running, checklists for others)",
      "Schedule your first week of micro-commitments in your calendar",
      "Join our Telegram community to share your first week's progress",
      "Review milestones every Sunday to stay on track"
    ],
    
    "generatedDate": "2026-02-14",
    "userName": "Alex Johnson",
    "dominantDomain": "Strategic Thinking",
    "totalEstimatedTimePerWeek": 12.0
  },
  "metadata": {
    "tokensUsed": 5234,
    "costUSD": 0.0198,
    "durationMs": 27456,
    "modelName": "claude-sonnet-4-20250514",
    "promptVersion": "v1.1"
  }
}
```

---

## Supporting Task 4: Input Validator

### Overview

**Purpose:** Validate user inputs before triggering expensive AI processing

**Approach:** Mix of code-based validation + Gemini Flash for semantic checks

**Execution:** Synchronous, runs before Goal Agents are spawned

### Validation Categories

**1. Code-Based Validation (No AI)**

Checked client-side AND server-side:

```typescript
interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];
}

interface ValidationError {
  field: string;
  code: string;
  message: string;
}

function validateGoalInputs(goals: Goal[]): ValidationResult {
  const errors: ValidationError[] = [];
  
  // Goal count
  if (goals.length < 1 || goals.length > 3) {
    errors.push({
      field: "goals",
      code: "4010",
      message: "You must set between 1 and 3 goals"
    });
  }
  
  // High priority constraint
  const highPriorityCount = goals.filter(g => g.priority === "high").length;
  if (highPriorityCount > 1) {
    errors.push({
      field: "priority",
      code: "4011",
      message: "Only 1 goal can be high priority"
    });
  }
  
  // Goal descriptions
  goals.forEach((goal, idx) => {
    if (!goal.description || goal.description.trim().length < 10) {
      errors.push({
        field: `goals[${idx}].description`,
        code: "4012",
        message: "Goal description must be at least 10 characters"
      });
    }
    if (goal.description.length > 200) {
      errors.push({
        field: `goals[${idx}].description`,
        code: "4013",
        message: "Goal description must be under 200 characters"
      });
    }
    
    // Categories
    if (goal.categories.length < 1 || goal.categories.length > 3) {
      errors.push({
        field: `goals[${idx}].categories`,
        code: "4014",
        message: "Each goal must have 1-3 life categories"
      });
    }
    
    // Deadline
    const today = new Date();
    const twoWeeks = new Date(today.getTime() + 14 * 24 * 60 * 60 * 1000);
    const oneYear = new Date(today.getTime() + 365 * 24 * 60 * 60 * 1000);
    
    if (goal.deadline < twoWeeks) {
      // Warning, not error
      errors.push({
        field: `goals[${idx}].deadline`,
        code: "4015",
        message: "Warning: Deadline is less than 2 weeks away. Make sure this is achievable."
      });
    }
    if (goal.deadline > oneYear) {
      errors.push({
        field: `goals[${idx}].deadline`,
        code: "4016",
        message: "Warning: Consider breaking long-term goals into shorter milestones."
      });
    }
    
    // Hardness rating
    if (![1, 2, 3, 4, 5].includes(goal.hardnessRating)) {
      errors.push({
        field: `goals[${idx}].hardnessRating`,
        code: "4017",
        message: "Hardness rating must be between 1 and 5"
      });
    }
  });
  
  return {
    valid: errors.length === 0,
    errors
  };
}
```

**2. AI-Powered Semantic Validation (Gemini Flash)**

Optional deeper validation before processing (can be skipped if user has already been warned):

```typescript
async function validateGoalQuality(
  goals: Goal[]
): Promise<SemanticValidationResult> {
  // Use Gemini Flash to check:
  // - Are goals actually goals? (not vague wishes like "be happy")
  // - Are goals specific enough? (not "get fit" but "run 5K")
  // - Are goals realistic given deadline + hardness rating?
  
  const prompt = `
  Analyze these goals for quality:
  ${JSON.stringify(goals)}
  
  For each goal, identify:
  1. Is this a clear, actionable goal? (yes/no)
  2. Is it specific enough to create a strategy? (yes/no)
  3. Any red flags? (contradictions, impossibility)
  
  Return JSON: { goalQualityChecks: [...] }
  `;
  
  // Call Gemini Flash
  // Return warnings (not errors - don't block submission)
}
```

**Decision:** This semantic validation is **optional for MVP**. Can be added post-launch if we see lots of vague goals.

### Rate Limiting Check

Before processing, verify user hasn't exceeded 2 analyses per day:

```sql
SELECT COUNT(*) as count
FROM analyses
WHERE user_id = $1
  AND created_at >= NOW() - INTERVAL '24 hours'
  AND status IN ('processing', 'complete', 'partial_failure');
```

If count ≥ 2, return error:
```json
{
  "code": "4020",
  "message": "You've used your 2 analyses today. Next slot opens at [timestamp]."
}
```

---

## Supporting Task 5: PDF Generator

### Overview

**Purpose:** Render Synthesizer output into final PDF document

**Approach:** Pure code (no AI), uses PDF generation library

**Execution:** Runs after Synthesizer completes, synchronous

### Library Choice

**Recommended:** Puppeteer (HTML → PDF)

**Reasoning:**
- Synthesizer outputs rich text (paragraphs, lists, formatting)
- HTML/CSS easier to style than programmatic PDF layout
- Can reuse web design patterns
- Supports custom fonts, colors, images

**Alternative:** PDFKit (programmatic)
- More control, but harder to maintain
- Use if Puppeteer adds too much deployment complexity

### Input Specification

**Function Signature:**
```typescript
async function generatePDF(
  synthesizerOutput: SynthesizerOutput,
  analysisId: string,
  userId: string
): Promise<PDFGenerationResult>
```

**Input:** Full `SynthesizerOutput.data` object

### Output Specification

```typescript
interface PDFGenerationResult {
  success: boolean;
  data?: {
    filePath: string;       // Supabase Storage path
    fileSize: number;       // Bytes
    publicUrl: string;      // Signed URL for download
  };
  error?: {
    code: string;
    message: string;
  };
  metadata: {
    durationMs: number;
  };
}
```

### Processing Logic

**Step 1: Render HTML Template**
```typescript
const htmlContent = renderPDFTemplate(synthesizerOutput);
// Uses template engine (Handlebars, EJS, etc.)
// Injects all data from synthesizerOutput
```

**Step 2: Generate PDF with Puppeteer**
```typescript
const browser = await puppeteer.launch();
const page = await browser.newPage();
await page.setContent(htmlContent);
const pdfBuffer = await page.pdf({
  format: 'A4',
  printBackground: true,
  margin: { top: '20mm', bottom: '20mm', left: '15mm', right: '15mm' }
});
await browser.close();
```

**Step 3: Upload to Supabase Storage**
```typescript
const filePath = `outputs/${userId}/${analysisId}/recommendations.pdf`;
const { data, error } = await supabase.storage
  .from('pdfs')
  .upload(filePath, pdfBuffer, {
    contentType: 'application/pdf',
    upsert: false
  });

// Generate signed URL (expires in 7 days)
const { data: urlData } = await supabase.storage
  .from('pdfs')
  .createSignedUrl(filePath, 604800); // 7 days in seconds
```

**Step 4: Update Database**
```sql
UPDATE analyses
SET 
  output_pdf_path = $1,
  output_pdf_size = $2,
  output_pdf_generated_at = NOW(),
  status = 'complete',
  completed_at = NOW()
WHERE id = $3;
```

### Error Handling

**Error Classifications:**

1. **System Error (5xxx):**
   - `5010`: HTML rendering failed
   - `5011`: Puppeteer PDF generation failed
   - `5012`: Supabase Storage upload failed
   - `5013`: Signed URL generation failed

**Retry Strategy:**
- Retry once on failure (total: 2 attempts)
- If both fail, mark analysis status = 'failed'

**User-Facing Messages:**
- "We had trouble creating your PDF. Trying again..."

### Template Design

**Structure covered in Document 5: PDF Template & Design Specification**

For this task, we just need to know:
- Template receives `SynthesizerOutput.data` as input
- Template is HTML/CSS with placeholders
- Output must be print-friendly (A4 format)

---

## Supporting Task 6: Error Handler

### Overview

**Purpose:** Universal error classification, logging, and user messaging

**Approach:** Pure code, no AI

**Scope:** Applies to all AI tasks and system tasks

### Error Classification System

**Error Code Ranges:**

- **4xxx: User Errors** - Fixable by user, not our fault
- **5xxx: System Errors** - Our infrastructure failed
- **6xxx: AI Errors** - AI provider issues or invalid AI responses

**Complete Error Taxonomy:**

```typescript
enum ErrorCode {
  // User Errors (4xxx)
  INVALID_PDF = "4001",
  PDF_TOO_LARGE = "4002",
  NOT_CLIFTON_PDF = "4003",
  CORRUPTED_PDF = "4004",
  INVALID_GOAL_COUNT = "4010",
  MULTIPLE_HIGH_PRIORITY = "4011",
  GOAL_TOO_SHORT = "4012",
  GOAL_TOO_LONG = "4013",
  INVALID_CATEGORIES = "4014",
  DEADLINE_TOO_SOON = "4015",
  DEADLINE_TOO_FAR = "4016",
  INVALID_HARDNESS = "4017",
  RATE_LIMIT_EXCEEDED = "4020",
  
  // System Errors (5xxx)
  DATABASE_READ_FAILED = "5001",
  DATABASE_WRITE_FAILED = "5002",
  STORAGE_UPLOAD_FAILED = "5003",
  PDF_GENERATION_FAILED = "5010",
  PROMPT_NOT_FOUND = "5011",
  
  // AI Errors (6xxx)
  AI_TIMEOUT = "6001",
  AI_INVALID_RESPONSE = "6002",
  AI_MISSING_FIELDS = "6003",
  AI_QUOTA_EXCEEDED = "6004",
}
```

### Error Response Format

**Standard Error Response:**
```typescript
interface ErrorResponse {
  success: false;
  error: {
    code: string;           // From ErrorCode enum
    message: string;        // User-facing message
    userAction?: string;    // What user should do next
    retryable: boolean;     // Can user retry?
    technicalDetails?: any; // For logging only (not shown to user)
  };
  metadata?: {
    timestamp: string;
    analysisId?: string;
    userId?: string;
  };
}
```

### User-Facing Messages

**Mapping error codes to friendly messages:**

```typescript
const errorMessages: Record<string, { message: string; userAction: string }> = {
  "4001": {
    message: "That file doesn't appear to be a PDF.",
    userAction: "Please upload a PDF file from Gallup CliftonStrengths."
  },
  "4002": {
    message: "That PDF is too large (over 10MB).",
    userAction: "Please upload a smaller file or compress your PDF."
  },
  "4003": {
    message: "We couldn't find CliftonStrengths data in that PDF.",
    userAction: "Make sure you're uploading the full report from Gallup, not a summary."
  },
  "4020": {
    message: "You've used your 2 analyses for today.",
    userAction: "Your next slot opens in [X hours]. Join our waitlist for unlimited access!"
  },
  "6001": {
    message: "We're experiencing high demand right now.",
    userAction: "Please try again in a moment."
  },
  "6004": {
    message: "We've hit our AI usage limit.",
    userAction: "This should resolve in a few minutes. Please try again shortly."
  },
  "5xxx": {
    message: "Something went wrong on our end.",
    userAction: "We've been notified. Please try again in a moment."
  }
};
```

### Logging Strategy

**What to log:**

```typescript
interface ErrorLog {
  timestamp: string;
  errorCode: string;
  errorMessage: string;
  userId?: string;
  analysisId?: string;
  taskName: string;         // Which AI task failed
  stackTrace?: string;
  context: {
    attemptNumber: number;  // 1 or 2 (for retries)
    inputSummary?: any;     // Sanitized inputs (no PII)
    aiResponse?: string;    // Full AI response if available
  };
}
```

**Where to log:**
- Console (captured by Vercel logs)
- Database: `error_logs` table (optional, for MVP can skip and use Vercel logs)
- Sentry (optional, post-MVP)

**Log Levels:**
- `ERROR`: All 5xxx and 6xxx errors
- `WARN`: 4xxx errors (user errors)
- `INFO`: Successful retries

### Retry Decision Logic

```typescript
function shouldRetry(errorCode: string, attemptNumber: number): boolean {
  // Never retry user errors
  if (errorCode.startsWith("4")) return false;
  
  // Always retry once for AI and system errors
  if (attemptNumber < 2) return true;
  
  // After 2nd attempt, give up
  return false;
}
```

---

## Universal Patterns

### Retry Strategy

**All AI tasks follow this pattern:**

```typescript
async function executeWithRetry<T>(
  taskFn: () => Promise<T>,
  taskName: string,
  maxAttempts: number = 2
): Promise<T> {
  let lastError: Error;
  
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      const result = await taskFn();
      
      if (attempt > 1) {
        // Log successful retry
        console.info(`${taskName} succeeded on attempt ${attempt}`);
      }
      
      return result;
      
    } catch (error) {
      lastError = error;
      
      // Log failure
      console.error(`${taskName} failed on attempt ${attempt}`, error);
      
      // Check if we should retry
      const shouldRetry = attempt < maxAttempts && 
                         !error.code?.startsWith("4");
      
      if (!shouldRetry) {
        break;
      }
      
      // Wait before retry (simple backoff)
      await sleep(1000 * attempt); // 1s, 2s
    }
  }
  
  throw lastError;
}

// Usage
const parserOutput = await executeWithRetry(
  () => parseCliftonStrengthsPDF(pdfBuffer, userId, analysisId),
  "CliftonStrengths Parser"
);
```

### Database Transaction Pattern

**For multi-step operations:**

```typescript
async function processAnalysis(analysisId: string) {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Step 1: Update status
    await client.query(
      'UPDATE analyses SET status = $1 WHERE id = $2',
      ['processing', analysisId]
    );
    
    // Step 2: Run Goal Agents
    const results = await runGoalAgents(analysisId);
    
    // Step 3: Run Synthesizer
    const synthesis = await runSynthesizer(results);
    
    // Step 4: Generate PDF
    const pdf = await generatePDF(synthesis);
    
    // Step 5: Update to complete
    await client.query(
      'UPDATE analyses SET status = $1, completed_at = NOW() WHERE id = $2',
      ['complete', analysisId]
    );
    
    await client.query('COMMIT');
    
  } catch (error) {
    await client.query('ROLLBACK');
    
    // Mark as failed
    await client.query(
      'UPDATE analyses SET status = $1, error_message = $2 WHERE id = $3',
      ['failed', error.message, analysisId]
    );
    
    throw error;
    
  } finally {
    client.release();
  }
}
```

### Prompt Retrieval Pattern

**All AI tasks load prompts from database:**

```typescript
async function getActivePrompt(category: PromptCategory): Promise<Prompt> {
  const { data, error } = await supabase
    .from('prompts')
    .select('*')
    .eq('category', category)
    .eq('is_active', true)
    .order('version', { ascending: false })
    .limit(1)
    .single();
    
  if (error || !data) {
    throw new Error(`Prompt not found for category: ${category}`);
  }
  
  return data;
}

// Usage
const prompt = await getActivePrompt('goal_agent');
const filledPrompt = fillTemplate(prompt.user_prompt_template, variables);
```

### Cost & Token Tracking Pattern

**Every AI call records metrics:**

```typescript
async function trackAIUsage(
  analysisId: string,
  taskType: string,
  tokensUsed: number,
  costUSD: number,
  durationMs: number
): Promise<void> {
  // Update analysis totals
  await supabase.rpc('increment_analysis_metrics', {
    p_analysis_id: analysisId,
    p_tokens: tokensUsed,
    p_cost: costUSD,
    p_duration: durationMs
  });
  
  // Log for monitoring
  console.info(`AI Usage: ${taskType}`, {
    tokensUsed,
    costUSD,
    durationMs
  });
}
```

---

## Performance & Cost Targets

### Per-Task SLAs

| Task | Target Latency | Max Latency | Target Cost | Max Cost |
|------|----------------|-------------|-------------|----------|
| CliftonStrengths Parser | 3s | 5s | $0.001 | $0.002 |
| Goal Agent (each) | 30s | 40s | $0.015 | $0.020 |
| Synthesizer | 20s | 30s | $0.018 | $0.025 |
| PDF Generator | 5s | 10s | $0 (no AI) | $0 |
| **Total (3 goals)** | **~100s** | **~130s** | **~$0.05** | **~$0.07** |

### Cost Breakdown (3-goal analysis)

```
Parser:      $0.001
Goal Agent 1: $0.015
Goal Agent 2: $0.015
Goal Agent 3: $0.015
Synthesizer: $0.018
PDF Gen:     $0.000
────────────────────
Total:       $0.064  (within $0.07 budget)
```

### Performance Monitoring

**Metrics to track in production:**

```typescript
interface PerformanceMetrics {
  // Per Task
  avgLatency: Record<TaskName, number>;
  p95Latency: Record<TaskName, number>;
  p99Latency: Record<TaskName, number>;
  failureRate: Record<TaskName, number>;
  retryRate: Record<TaskName, number>;
  
  // Per Analysis
  avgTotalDuration: number;
  avgTotalCost: number;
  
  // Overall
  dailyAnalysesCount: number;
  dailyAICost: number;
  successRate: number;
}
```

**Alerts:**
- If any task exceeds max latency >10% of the time
- If total cost per analysis exceeds $0.10
- If failure rate >5% for any task
- If daily AI cost exceeds $50

---

## Appendix: TypeScript Interfaces

### Complete Type Definitions

```typescript
// ============================================================================
// CORE TYPES
// ============================================================================

type DominantDomain = 
  | "Executing"
  | "Influencing" 
  | "Relationship Building"
  | "Strategic Thinking";

type GoalPriority = "high" | "medium" | "low";

type ProcessingStatus = 
  | "pending"
  | "processing"
  | "complete"
  | "failed";

type AnalysisStatus =
  | "draft"
  | "parsing"
  | "processing"
  | "complete"
  | "failed"
  | "partial_failure";

// ============================================================================
// CLIFTONSTRENGTHS DATA
// ============================================================================

interface CliftonTheme {
  name: string;
  rank: number;
  description?: string;
}

interface CliftonProfile {
  topTenThemes: CliftonTheme[];
  bottomFiveThemes: CliftonTheme[];
  dominantDomain: DominantDomain;
}

// ============================================================================
// USER CONTEXT
// ============================================================================

interface WeeklyTimeInvestment {
  work: number;
  health: number;
  relationships: number;
  learning: number;
  creativity: number;
  rest: number;
  community: number;
  admin: number;
  total: number;
}

interface QuestionnaireResponses {
  biggestChallenge?: string;
  motivations?: string;
  energyPeaks?: string;
  pastSuccesses?: string;
  pastFailures?: string;
}

interface UserContext {
  weeklyTimeInvestment?: WeeklyTimeInvestment;
  questionnaireResponses?: QuestionnaireResponses;
}

// ============================================================================
// GOALS
// ============================================================================

interface Goal {
  id: string;
  description: string;
  priority: GoalPriority;
  categories: string[];
  deadline: Date;
  hardnessRating: 1 | 2 | 3 | 4 | 5;
  processingStatus?: ProcessingStatus;
}

// ============================================================================
// ACTIONS & MILESTONES
// ============================================================================

interface ActionItem {
  title: string;
  description: string;
  frequency: string;
  estimatedTimeMinutes: number;
  coreDrives: number[];
  strengthsUsed: string[];
  difficultyLevel: 1 | 2 | 3;
}

interface Milestone {
  week: number;
  description: string;
  successCriteria: string;
}

// ============================================================================
// AI TASK OUTPUTS
// ============================================================================

interface AITaskMetadata {
  tokensUsed: number;
  costUSD: number;
  durationMs: number;
  modelName: string;
  promptVersion: string;
}

interface ParserOutput {
  success: boolean;
  data?: {
    topTenThemes: CliftonTheme[];
    bottomFiveThemes: CliftonTheme[];
    dominantDomain: DominantDomain;
    rawExtraction: string;
  };
  error?: {
    code: string;
    message: string;
    details?: any;
  };
  metadata: AITaskMetadata;
}

interface GoalAgentOutput {
  success: boolean;
  goalId: string;
  data?: {
    strategyOverview: string;
    whyThisWorks: string;
    primaryCoreDrives: number[];
    secondaryCoreDrives: number[];
    keystoneHabit: ActionItem;
    coreActions: ActionItem[];
    bonusAccelerators: ActionItem[];
    estimatedTimePerWeek: number;
    capacityWarning?: string;
    milestones: Milestone[];
    trackingMethod: string;
  };
  error?: {
    code: string;
    message: string;
  };
  metadata: AITaskMetadata;
}

interface SynthesizedGoal {
  priority: GoalPriority;
  goalDescription: string;
  strategyOverview: string;
  whyThisWorks: string;
  primaryCoreDrives: number[];
  keystoneHabit: ActionItem;
  coreActions: ActionItem[];
  bonusAccelerators: ActionItem[];
  milestones: Milestone[];
  estimatedTimePerWeek: number;
  trackingMethod: string;
}

interface SynthesizerOutput {
  success: boolean;
  data?: {
    executiveSummary: string;
    cliftonStrengthsOverview: string;
    overarchingStrategy: string;
    goals: SynthesizedGoal[];
    trackingGuidance: string;
    motivationalClosing: string;
    nextSteps: string[];
    generatedDate: string;
    userName: string;
    dominantDomain: string;
    totalEstimatedTimePerWeek: number;
  };
  error?: {
    code: string;
    message: string;
  };
  metadata: AITaskMetadata;
}

interface PDFGenerationResult {
  success: boolean;
  data?: {
    filePath: string;
    fileSize: number;
    publicUrl: string;
  };
  error?: {
    code: string;
    message: string;
  };
  metadata: {
    durationMs: number;
  };
}

// ============================================================================
// ERROR HANDLING
// ============================================================================

interface ValidationError {
  field: string;
  code: string;
  message: string;
}

interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];
}

interface ErrorResponse {
  success: false;
  error: {
    code: string;
    message: string;
    userAction?: string;
    retryable: boolean;
    technicalDetails?: any;
  };
  metadata?: {
    timestamp: string;
    analysisId?: string;
    userId?: string;
  };
}

// ============================================================================
// PROMPTS
// ============================================================================

type PromptCategory = 
  | "clifton_parser"
  | "goal_agent"
  | "synthesizer";

interface Prompt {
  id: string;
  category: PromptCategory;
  version: string;
  systemPrompt: string;
  userPromptTemplate: string;
  targetModel: string;
  temperature: number;
  maxTokens: number;
  isActive: boolean;
  createdAt: Date;
}
```

---

## Open Questions & Future Considerations

### Question 1: Gemini Flash Reliability
**Status:** Unvalidated

We haven't tested if Gemini Flash can reliably parse CliftonStrengths PDFs. Before implementing, we should:
1. Collect 5-10 sample PDFs (different formats, years)
2. Test Gemini Flash extraction accuracy
3. Decide if we need fallback to manual entry

**Decision point:** After collecting sample PDFs

---

### Question 2: Prompt Storage vs. Code
**Status:** Decided (database storage)

Prompts stored in database for versioning, but we could also:
- Store in code (easier for Claude Code to implement)
- Store in config files (middle ground)

**Tradeoff:** Database = more flexible, code = simpler deployment

**Current decision:** Database (enables A/B testing post-MVP)

---

### Question 3: Semantic Goal Validation
**Status:** Optional for MVP

Should we use Gemini Flash to validate goal quality before processing?

**Pros:** Catch vague goals early, save cost
**Cons:** Adds latency, complexity, might annoy users

**Current decision:** Skip for MVP, add if we see lots of bad goals

---

### Question 4: Synthesizer Creativity Level
**Status:** Needs prompt engineering

How much should Synthesizer rewrite vs. preserve Goal Agent outputs?

**Options:**
A. Minimal rewrite (just add connective tissue)
B. Moderate rewrite (harmonize language, improve flow)
C. Heavy rewrite (complete narrative restructure)

**Current decision:** Option A for MVP (safest, preserves agent quality)

---

### Question 5: PDF Template Complexity
**Status:** Deferred to Document 5

Do we want:
- Simple text-based PDF (easy)
- Branded PDF with graphics (medium)
- Interactive PDF with fillable forms (complex)

**Current decision:** Deferred to PDF Template & Design Specification

---

## Document Metadata

**Document Status:** Complete - Ready for Prompt Engineering  
**Last Updated:** February 14, 2026  
**Next Document:** Prompt Engineering Specification  
**Owner:** Igor (Founder, PM, Designer)

---

## Summary for Claude Code Implementation

**What Claude Code needs to build:**

1. **3 Core AI Functions:**
   - `parseCliftonStrengthsPDF()` - Gemini Flash extraction
   - `generateGoalRecommendations()` - Claude Sonnet 4 strategy (1-3 parallel)
   - `synthesizeRecommendations()` - Claude Sonnet 4 synthesis

2. **3 Supporting Functions:**
   - `validateGoalInputs()` - Pre-processing validation
   - `generatePDF()` - Puppeteer rendering
   - `handleError()` - Universal error management

3. **Universal Patterns:**
   - Retry logic (2 attempts max)
   - Database prompt retrieval
   - Cost/token tracking
   - Error classification

4. **Database Operations:**
   - Read/write to `analyses`, `goals`, `ai_agent_outputs`, `prompts` tables
   - Upload PDFs to Supabase Storage
   - Transaction management for multi-step operations

5. **Performance Requirements:**
   - Total processing: ~130s for 3 goals
   - Total cost: ~$0.05 per analysis
   - Success rate: >90%

**All input/output contracts defined with TypeScript interfaces in Appendix.**

---

*This specification provides complete implementation details for all AI tasks in the Goal Achievement MVP. Each task has clear input/output contracts, error handling patterns, validation rules, and performance targets suitable for Claude Code implementation.*
