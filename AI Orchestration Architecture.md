# AI Orchestration Architecture
## Personal Time OS - Goal Achievement MVP

**Version:** 1.0  
**Created:** February 15, 2026  
**Purpose:** Complete orchestration specification for coordinating 7 AI tasks within Vercel Fluid Compute constraints  
**Target:** Claude Code implementation with clear execution patterns

---

## Executive Summary

This document defines how all AI tasks coordinate within a single Vercel Fluid Compute function to transform CliftonStrengths assessments into personalized goal strategies. The architecture optimizes for the 300-second execution limit while maintaining resilience through intelligent retry logic, partial failure handling, and real-time progress tracking.

**Core Orchestration Principles:**
- **Pre-cached preparation:** Move slow tasks (PDF parsing, Octalysis mapping) to background before "Analyse" click
- **Intelligent parallelization:** Run 1-3 Goal Agents simultaneously with 2-second stagger to avoid API rate limits
- **Graceful degradation:** Per-agent retries + orchestration-level recovery for partial failures
- **Optimistic persistence:** Commit database updates as stages complete (not single transaction)
- **Lightweight observability:** Stage tracking for frontend, structured logging for debugging

**Performance Targets:**
- **Expected duration:** ~112 seconds (mapping pre-cached, 3 goals in parallel)
- **Worst case with retries:** ~150 seconds (well under 300s limit)
- **Safety buffer:** 150 seconds (50% margin for upgraded Vercel plans)
- **Cost per analysis:** ~$0.05 AI + ~$0.001 Vercel = ~$0.051 total

**Task Execution Summary:**
```
Background (Step 1, PDF upload):
  CliftonStrengths PDF parsing        ~5s   (Gemini Flash, async)
  Octalysis domain mapping           ~10s   (Gemini Flash, async)
  
Critical Path (Step 3, "Analyse" click):
  Load cached data                   ~0.1s  (Database read)
  Goal Agent 1 (parallel)            ~40s   (Claude Sonnet 4)
  Goal Agent 2 (parallel +2s)        ~40s   (Claude Sonnet 4)
  Goal Agent 3 (parallel +4s)        ~40s   (Claude Sonnet 4)
  Synthesizer                        ~30s   (Claude Sonnet 4, sequential)
  PDF Generation                     ~20s   (Pure code, sequential)
  Database storage                    ~2s   (Final updates)
  
Total: ~112s (mapping pre-cached) | ~122s (if mapping not cached)
```

**Key Architectural Decisions:**
1. Pre-cache Octalysis mapping in Step 1 to minimize critical path
2. Stagger parallel Goal Agent starts by 2 seconds to avoid Claude API rate limits
3. Implement per-agent retries (2 attempts) with orchestration-level recovery (1 additional attempt)
4. Use optimistic database updates (commit as stages complete, not single transaction)
5. Track processing stages in database for lightweight frontend polling (every 3 seconds)
6. Automatic cleanup of AI outputs via database trigger on success
7. Set orchestration timeout to 240 seconds (80% of 300s limit for safety buffer)
8. Log orchestration trace to Vercel console, store in database only on failures

---

## Orchestration Architecture Overview

### Technology Context

**Runtime Environment:**
- **Platform:** Vercel Fluid Compute (single function execution)
- **Execution limit:** 300 seconds maximum (Pro plan)
- **Memory:** 3GB available
- **Concurrency:** Managed by Vercel (auto-scaling)
- **Region:** Closest to Supabase database (minimize latency)

**External Dependencies:**
- **Supabase Database:** PostgreSQL 15+ for state management
- **Supabase Storage:** PDF file storage and retrieval
- **Claude API:** Sonnet 4 for goal analysis and synthesis
- **Gemini API:** Flash 2.0 for parsing and mapping (background tasks)

**Why Single Function Approach:**
- Vercel Fluid Compute optimized for long-running API routes
- No need for separate queuing/worker infrastructure
- Simpler deployment and debugging
- Cost-effective at MVP scale (100-1,000 users)
- Can migrate to dedicated workers later if needed

---

### Orchestration Layers

The orchestration system operates across three distinct layers:

**Layer 1: Background Processing (Step 1)**
- Triggered by: PDF upload event
- Tasks: CliftonStrengths parsing, Octalysis mapping
- Execution: Asynchronous, non-blocking to user
- Storage: Results cached in database for Step 3
- Timeout: 30 seconds per task (separate from main orchestration)

**Layer 2: Main Orchestration (Step 3)**
- Triggered by: User clicking "Analyse" button
- Tasks: Goal analysis (1-3 parallel), synthesis, PDF generation
- Execution: Synchronous within 300s limit
- Storage: Progressive updates as stages complete
- Timeout: 240 seconds total orchestration

**Layer 3: Post-Processing**
- Triggered by: Analysis completion or failure
- Tasks: Cleanup agent outputs, update metrics, send notifications (future)
- Execution: Database triggers and future edge functions
- Storage: Final state persistence
- Timeout: N/A (trigger-based)

---

## Task Execution Flow

### Background Processing Flow (Layer 1)

**Trigger:** User uploads CliftonStrengths PDF in Step 1

```typescript
// API endpoint: POST /api/v1/analyses/:id/upload-pdf
async function handlePDFUpload(
  pdfFile: File,
  analysisId: string,
  userId: string
) {
  // 1. Upload PDF to Supabase Storage
  const pdfPath = await uploadToStorage(pdfFile, userId, analysisId);
  
  // 2. Update analysis record
  await db.analyses.update({
    id: analysisId,
    status: 'parsing',
    clifton_pdf_path: pdfPath
  });
  
  // 3. Start background parsing (async, don't await)
  startBackgroundProcessing(analysisId, pdfPath)
    .catch(error => {
      // Log error but don't block user
      console.error('Background processing failed:', error);
      db.analyses.update({
        id: analysisId,
        status: 'draft',
        error_message: 'PDF parsing failed'
      });
    });
  
  // 4. Return immediately to user
  return { success: true, analysisId };
}
```

**Background processing task:**

```typescript
async function startBackgroundProcessing(
  analysisId: string,
  pdfPath: string
) {
  try {
    // Task 1: Parse CliftonStrengths PDF (Gemini Flash)
    const parseResult = await parseCliftonStrengthsPDF(pdfPath);
    
    if (!parseResult.success) {
      throw new Error(parseResult.error.message);
    }
    
    // Store parsed data
    await db.analyses.update({
      id: analysisId,
      clifton_top_10: parseResult.data.topTenThemes,
      clifton_bottom_5: parseResult.data.bottomFiveThemes,
      clifton_dominant_domain: parseResult.data.dominantDomain,
      parsing_completed_at: new Date()
    });
    
    // Task 2: Generate Octalysis mapping (Gemini Flash)
    const mappingResult = await mapCliftonToOctalysis(
      parseResult.data.topTenThemes,
      parseResult.data.dominantDomain
    );
    
    if (!mappingResult.success) {
      throw new Error(mappingResult.error.message);
    }
    
    // Store mapping in clifton_strengths_cache table
    await db.clifton_strengths_cache.upsert({
      analysis_id: analysisId,
      octalysis_mapping: mappingResult.data,
      cached_at: new Date()
    });
    
    // Mark background processing complete
    await db.analyses.update({
      id: analysisId,
      status: 'draft' // Ready for user to set goals
    });
    
  } catch (error) {
    // Background processing failed
    await db.analyses.update({
      id: analysisId,
      status: 'draft',
      error_message: `Background processing failed: ${error.message}`,
      error_details: { stage: 'background', error: error.stack }
    });
  }
}
```

**Timing expectations:**
- PDF parsing: ~5 seconds
- Octalysis mapping: ~10 seconds
- Total background: ~15 seconds
- User proceeds to Step 2 while this runs

**Failure handling:**
- If background processing fails, analysis stays in 'draft' state
- User can still set goals in Step 2
- Main orchestration will retry parsing if cache miss detected
- Error message shown to user: "We had trouble reading your PDF. Please re-upload."

---

### Main Orchestration Flow (Layer 2)

**Trigger:** User clicks "Analyse" button in Step 2

```typescript
// API endpoint: POST /api/v1/analyses/:id/submit
export async function POST(req: Request) {
  const { analysisId } = await req.json();
  const userId = await getUserIdFromSession(req);
  
  // Validate ownership
  const analysis = await db.analyses.findOne({
    id: analysisId,
    user_id: userId
  });
  
  if (!analysis) {
    return Response.json(
      { error: { code: 'NOT_FOUND', message: 'Analysis not found' } },
      { status: 404 }
    );
  }
  
  // Check rate limit
  const rateLimitCheck = await checkRateLimit(userId);
  if (!rateLimitCheck.allowed) {
    return Response.json(
      { error: { code: 'RATE_LIMIT', message: 'You have reached your daily limit of 2 analyses' } },
      { status: 429 }
    );
  }
  
  // Check for duplicate submission
  if (analysis.status === 'processing' || analysis.status === 'complete') {
    return Response.json(
      { error: { code: 'ALREADY_PROCESSING', message: 'This analysis is already being processed' } },
      { status: 409 }
    );
  }
  
  // Start orchestration (this is the main function)
  try {
    await orchestrateAnalysis(analysisId, userId);
    
    return Response.json({
      success: true,
      analysisId,
      message: 'Analysis started. Check status at /api/v1/analyses/:id/status'
    });
    
  } catch (error) {
    console.error('Orchestration failed:', error);
    
    return Response.json(
      { error: { code: 'ORCHESTRATION_FAILED', message: 'Failed to start analysis' } },
      { status: 500 }
    );
  }
}
```

**Main orchestration function:**

```typescript
async function orchestrateAnalysis(
  analysisId: string,
  userId: string
) {
  const startTime = Date.now();
  const logger = createLogger(analysisId);
  
  try {
    // Set overall timeout (240 seconds)
    const timeoutPromise = createTimeout(240_000, 'ORCHESTRATION_TIMEOUT');
    
    // Run orchestration with timeout protection
    await Promise.race([
      runOrchestration(analysisId, userId, logger),
      timeoutPromise
    ]);
    
  } catch (error) {
    logger.error('Orchestration failed', { error: error.message });
    
    // Handle timeout vs other errors
    if (error.message === 'ORCHESTRATION_TIMEOUT') {
      await handleTimeout(analysisId, logger);
    } else {
      await handleOrchestrationFailure(analysisId, error, logger);
    }
    
    throw error;
  }
}

async function runOrchestration(
  analysisId: string,
  userId: string,
  logger: Logger
) {
  // ========================================
  // STAGE 0: Load and validate data
  // ========================================
  logger.stage('data_loading_start');
  
  const analysis = await loadAnalysisData(analysisId);
  const goals = await db.goals.findMany({
    analysis_id: analysisId,
    order: [['priority', 'DESC']] // High priority first
  });
  
  // Validate we have goals
  if (goals.length === 0) {
    throw new Error('No goals found for analysis');
  }
  
  // Check if Octalysis mapping cached
  let octalysisMapping = await db.clifton_strengths_cache.findOne({
    analysis_id: analysisId
  });
  
  // If not cached (background processing failed), generate now
  if (!octalysisMapping) {
    logger.stage('octalysis_mapping_start', { reason: 'cache_miss' });
    
    const mappingResult = await mapCliftonToOctalysisWithRetry(
      analysis.clifton_top_10,
      analysis.clifton_dominant_domain
    );
    
    octalysisMapping = mappingResult.data;
    
    // Cache for future use
    await db.clifton_strengths_cache.insert({
      analysis_id: analysisId,
      octalysis_mapping: octalysisMapping,
      cached_at: new Date()
    });
    
    logger.stage('octalysis_mapping_complete', { 
      durationMs: mappingResult.metadata.durationMs 
    });
  } else {
    logger.stage('octalysis_mapping_cached');
  }
  
  // Update analysis status
  await db.analyses.update({
    id: analysisId,
    status: 'processing',
    processing_stage: 'goal_analysis',
    submitted_at: new Date()
  });
  
  // ========================================
  // STAGE 1: Parallel Goal Analysis
  // ========================================
  logger.stage('goal_analysis_start', { goalCount: goals.length });
  
  const goalAnalysisResults = await executeParallelGoalAnalysis(
    goals,
    analysis,
    octalysisMapping,
    logger
  );
  
  logger.stage('goal_analysis_complete', {
    succeeded: goalAnalysisResults.filter(r => r.success).length,
    failed: goalAnalysisResults.filter(r => !r.success).length
  });
  
  // Check if we have at least one successful goal
  const successfulGoals = goalAnalysisResults.filter(r => r.success);
  
  if (successfulGoals.length === 0) {
    throw new Error('All goal analyses failed');
  }
  
  // ========================================
  // STAGE 2: Synthesis
  // ========================================
  logger.stage('synthesis_start');
  
  await db.analyses.update({
    id: analysisId,
    processing_stage: 'synthesis'
  });
  
  const synthesisResult = await synthesizeRecommendationsWithRetry(
    successfulGoals,
    analysis,
    octalysisMapping
  );
  
  if (!synthesisResult.success) {
    throw new Error(`Synthesis failed: ${synthesisResult.error.message}`);
  }
  
  // Store synthesis output
  await db.ai_agent_outputs.insert({
    analysis_id: analysisId,
    agent_type: 'synthesizer',
    output_text: synthesisResult.data,
    tokens_used: synthesisResult.metadata.tokensUsed,
    cost_usd: synthesisResult.metadata.costUSD,
    duration_ms: synthesisResult.metadata.durationMs,
    model_name: synthesisResult.metadata.modelName,
    prompt_version: synthesisResult.metadata.promptVersion
  });
  
  logger.stage('synthesis_complete', {
    tokensUsed: synthesisResult.metadata.tokensUsed,
    durationMs: synthesisResult.metadata.durationMs
  });
  
  // ========================================
  // STAGE 3: PDF Generation
  // ========================================
  logger.stage('pdf_generation_start');
  
  await db.analyses.update({
    id: analysisId,
    processing_stage: 'pdf_generation'
  });
  
  const pdfResult = await generatePDF(
    synthesisResult.data,
    analysis,
    userId
  );
  
  if (!pdfResult.success) {
    throw new Error(`PDF generation failed: ${pdfResult.error}`);
  }
  
  logger.stage('pdf_generation_complete', {
    pdfSize: pdfResult.sizeBytes,
    durationMs: pdfResult.durationMs
  });
  
  // ========================================
  // STAGE 4: Final Updates
  // ========================================
  logger.stage('finalization_start');
  
  // Calculate total costs
  const totalCost = calculateTotalCost(goalAnalysisResults, synthesisResult);
  const totalTokens = calculateTotalTokens(goalAnalysisResults, synthesisResult);
  const totalDuration = Date.now() - startTime;
  
  // Determine final status
  const finalStatus = goalAnalysisResults.length === successfulGoals.length
    ? 'complete'
    : 'partial_failure';
  
  // Update analysis with final results
  await db.analyses.update({
    id: analysisId,
    status: finalStatus,
    processing_stage: null,
    output_pdf_path: pdfResult.storagePath,
    output_pdf_size: pdfResult.sizeBytes,
    output_pdf_generated_at: new Date(),
    total_tokens_used: totalTokens,
    total_cost_usd: totalCost,
    total_duration_ms: totalDuration,
    completed_at: new Date()
  });
  
  // Increment user's analysis count
  await db.users.increment('total_analyses_count', { id: userId });
  
  logger.stage('finalization_complete', {
    finalStatus,
    totalCost,
    totalDuration
  });
  
  // Note: Cleanup of ai_agent_outputs happens automatically via database trigger
  // when status changes to 'complete'
  
  logger.success('Orchestration complete', {
    durationMs: totalDuration,
    costUSD: totalCost,
    tokensUsed: totalTokens
  });
}
```

---

## Multi-Agent Coordination

### Parallel Goal Analysis Execution

The core of the orchestration is the parallel execution of 1-3 Goal Agents. This section handles dynamic scaling, staggered starts, retry logic, and partial failure recovery.

```typescript
async function executeParallelGoalAnalysis(
  goals: Goal[],
  analysis: Analysis,
  octalysisMapping: OctalysisMapping,
  logger: Logger
): Promise<GoalAnalysisResult[]> {
  
  // Build context once (shared across all agents)
  const userContext = buildUserContext(analysis);
  const cliftonProfile = buildCliftonProfile(analysis);
  
  // Create staggered promises (0s, 2s, 4s delays)
  const goalPromises = goals.map((goal, index) => {
    return new Promise<GoalAnalysisResult>((resolve) => {
      setTimeout(async () => {
        try {
          // Update goal status
          await db.goals.update({
            id: goal.id,
            processing_status: 'processing',
            processing_started_at: new Date()
          });
          
          // Execute goal analysis with retry
          const result = await analyzeGoalWithRetry(
            goal,
            cliftonProfile,
            octalysisMapping,
            userContext,
            logger,
            { maxRetries: 2 }
          );
          
          // Update goal status on success
          await db.goals.update({
            id: goal.id,
            processing_status: 'complete',
            processing_completed_at: new Date()
          });
          
          resolve({
            success: true,
            goalId: goal.id,
            data: result.data,
            metadata: result.metadata
          });
          
        } catch (error) {
          logger.error(`Goal ${index + 1} failed after retries`, {
            goalId: goal.id,
            error: error.message
          });
          
          // Update goal status on failure
          await db.goals.update({
            id: goal.id,
            processing_status: 'failed',
            processing_completed_at: new Date()
          });
          
          resolve({
            success: false,
            goalId: goal.id,
            error: error.message
          });
        }
      }, index * 2000); // Stagger: 0s, 2s, 4s
    });
  });
  
  // Wait for all goals to complete (success or failure)
  const results = await Promise.all(goalPromises);
  
  // Check for partial failures
  const failed = results.filter(r => !r.success);
  const succeeded = results.filter(r => r.success);
  
  // If some failed but some succeeded, attempt orchestration-level retry
  if (failed.length > 0 && succeeded.length > 0) {
    logger.stage('partial_failure_retry', {
      failedCount: failed.length,
      succeededCount: succeeded.length
    });
    
    const retryResults = await retryFailedGoals(
      failed,
      goals,
      cliftonProfile,
      octalysisMapping,
      userContext,
      logger
    );
    
    // Merge original successes with retry results
    return [...succeeded, ...retryResults];
  }
  
  return results;
}
```

**Staggering strategy:**
- First goal: Starts immediately (0s delay)
- Second goal: Starts after 2 seconds
- Third goal: Starts after 4 seconds
- Total overhead: 6 seconds (minimal compared to 60s execution time)

**Why stagger:**
- Prevents simultaneous Claude API requests that might trigger rate limits
- Distributes load across time window
- Still much faster than sequential execution (6s overhead vs 120s sequential)

---

### Per-Agent Retry Logic

Each Goal Agent gets 2 attempts to succeed before being marked as failed.

```typescript
async function analyzeGoalWithRetry(
  goal: Goal,
  cliftonProfile: CliftonProfile,
  octalysisMapping: OctalysisMapping,
  userContext: UserContext,
  logger: Logger,
  options: { maxRetries: number }
): Promise<GoalAgentOutput> {
  
  let lastError: Error | null = null;
  
  for (let attempt = 1; attempt <= options.maxRetries; attempt++) {
    try {
      logger.debug(`Goal ${goal.id} attempt ${attempt}/${options.maxRetries}`);
      
      // Call Goal Agent AI task
      const result = await analyzeGoal(
        goal,
        cliftonProfile,
        octalysisMapping,
        userContext
      );
      
      // Validate response
      if (!result.success) {
        throw new Error(result.error.message);
      }
      
      // Store agent output in database
      await db.ai_agent_outputs.insert({
        analysis_id: goal.analysis_id,
        goal_id: goal.id,
        agent_type: `goal_agent_${goal.priority}` as AgentType,
        output_text: result.data,
        tokens_used: result.metadata.tokensUsed,
        cost_usd: result.metadata.costUSD,
        duration_ms: result.metadata.durationMs,
        model_name: result.metadata.modelName,
        prompt_version: result.metadata.promptVersion
      });
      
      // Success - return result
      return result;
      
    } catch (error) {
      lastError = error;
      
      // Classify error
      const classification = classifyError(error);
      
      logger.warn(`Goal ${goal.id} attempt ${attempt} failed`, {
        error: error.message,
        retriable: classification.shouldRetry,
        errorType: classification.type
      });
      
      // If non-retriable or last attempt, throw immediately
      if (!classification.shouldRetry || attempt === options.maxRetries) {
        throw error;
      }
      
      // Wait before retry (exponential backoff)
      const backoffMs = 2000 * attempt; // 2s, 4s
      await sleep(backoffMs);
    }
  }
  
  // Should never reach here, but TypeScript needs it
  throw lastError || new Error('Unknown error during goal analysis');
}
```

**Error classification:**

```typescript
interface ErrorClassification {
  type: 'retriable' | 'permanent' | 'unknown';
  shouldRetry: boolean;
  maxRetries?: number;
}

function classifyError(error: any): ErrorClassification {
  // Tier 1: Retriable (transient network/API issues)
  if (
    error.code === 'ECONNRESET' ||
    error.code === 'ETIMEDOUT' ||
    error.status === 429 || // Rate limit
    error.status === 503 || // Service unavailable
    error.message?.includes('timeout') ||
    error.message?.includes('socket hang up')
  ) {
    return { type: 'retriable', shouldRetry: true };
  }
  
  // Tier 2: Permanent (bad input, validation failures)
  if (
    error.code?.startsWith('4') || // 4xx errors
    error.message?.includes('validation') ||
    error.message?.includes('invalid') ||
    error.message?.includes('malformed')
  ) {
    return { type: 'permanent', shouldRetry: false };
  }
  
  // Tier 3: Unknown (err on side of retry once)
  return { 
    type: 'unknown', 
    shouldRetry: true, 
    maxRetries: 1 
  };
}
```

---

### Orchestration-Level Retry (Partial Failure Recovery)

If some goals succeed but others fail, the orchestration attempts one more recovery pass for the failed goals.

```typescript
async function retryFailedGoals(
  failedResults: GoalAnalysisResult[],
  allGoals: Goal[],
  cliftonProfile: CliftonProfile,
  octalysisMapping: OctalysisMapping,
  userContext: UserContext,
  logger: Logger
): Promise<GoalAnalysisResult[]> {
  
  logger.stage('orchestration_retry_start', {
    failedGoalIds: failedResults.map(r => r.goalId)
  });
  
  const retryPromises = failedResults.map(async (failedResult) => {
    try {
      // Find the original goal
      const goal = allGoals.find(g => g.id === failedResult.goalId);
      if (!goal) {
        throw new Error('Goal not found for retry');
      }
      
      // Reset goal status for retry
      await db.goals.update({
        id: goal.id,
        processing_status: 'processing'
      });
      
      // Single retry attempt (no nested retries)
      const result = await analyzeGoal(
        goal,
        cliftonProfile,
        octalysisMapping,
        userContext
      );
      
      if (!result.success) {
        throw new Error(result.error.message);
      }
      
      // Store successful retry output
      await db.ai_agent_outputs.insert({
        analysis_id: goal.analysis_id,
        goal_id: goal.id,
        agent_type: `goal_agent_${goal.priority}` as AgentType,
        output_text: result.data,
        tokens_used: result.metadata.tokensUsed,
        cost_usd: result.metadata.costUSD,
        duration_ms: result.metadata.durationMs,
        model_name: result.metadata.modelName,
        prompt_version: result.metadata.promptVersion
      });
      
      // Update goal status
      await db.goals.update({
        id: goal.id,
        processing_status: 'complete',
        processing_completed_at: new Date()
      });
      
      logger.info(`Retry succeeded for goal ${goal.id}`);
      
      return {
        success: true,
        goalId: goal.id,
        data: result.data,
        metadata: result.metadata
      };
      
    } catch (error) {
      logger.error(`Retry failed for goal ${failedResult.goalId}`, {
        error: error.message
      });
      
      // Keep goal status as 'failed'
      await db.goals.update({
        id: failedResult.goalId,
        processing_status: 'failed',
        processing_completed_at: new Date()
      });
      
      return {
        success: false,
        goalId: failedResult.goalId,
        error: error.message
      };
    }
  });
  
  const retryResults = await Promise.all(retryPromises);
  
  logger.stage('orchestration_retry_complete', {
    recovered: retryResults.filter(r => r.success).length,
    stillFailed: retryResults.filter(r => !r.success).length
  });
  
  return retryResults;
}
```

**Retry strategy summary:**
1. **Per-agent retries:** Each goal gets 2 attempts (immediate retry with backoff)
2. **Orchestration retry:** Failed goals get 1 additional attempt after all goals complete
3. **Maximum attempts:** 3 total (2 per-agent + 1 orchestration)
4. **Partial success:** If high-priority goal succeeds, analysis proceeds even if low-priority goals fail

---

## Database Transaction Strategy

### Optimistic Updates Pattern

The orchestration uses multiple small transactions instead of one large transaction to ensure resilience and visibility.

```typescript
// ANTI-PATTERN (don't do this):
async function orchestrateWithSingleTransaction(analysisId: string) {
  await db.transaction(async (tx) => {
    // This holds a database lock for 130+ seconds
    await tx.analyses.update({ status: 'processing' });
    await runGoalAnalysis();
    await runSynthesis();
    await generatePDF();
    await tx.analyses.update({ status: 'complete' });
  });
}

// CORRECT PATTERN (optimistic updates):
async function orchestrateWithOptimisticUpdates(analysisId: string) {
  // Update 1: Start processing (commit immediately)
  await db.analyses.update({
    id: analysisId,
    status: 'processing',
    processing_stage: 'goal_analysis',
    submitted_at: new Date()
  });
  
  // Process goals (commit as each completes)
  for (const goalResult of goalResults) {
    await db.ai_agent_outputs.insert(goalResult);
    await db.goals.update({
      id: goalResult.goalId,
      processing_status: 'complete'
    });
  }
  
  // Update 2: Synthesis stage (commit immediately)
  await db.analyses.update({
    id: analysisId,
    processing_stage: 'synthesis'
  });
  
  // Store synthesis output (commit immediately)
  await db.ai_agent_outputs.insert(synthesisOutput);
  
  // Update 3: PDF generation stage (commit immediately)
  await db.analyses.update({
    id: analysisId,
    processing_stage: 'pdf_generation'
  });
  
  // Update 4: Complete (commit immediately)
  await db.analyses.update({
    id: analysisId,
    status: 'complete',
    total_cost_usd: calculateTotalCost(),
    completed_at: new Date()
  });
}
```

**Benefits of optimistic updates:**
- No long-held database locks (each update commits in <100ms)
- If orchestration crashes, database shows partial progress
- Easy to resume or clean up failed attempts
- Frontend can poll intermediate states
- Minimal risk of deadlocks or transaction timeouts

**Trade-offs:**
- If orchestration fails mid-flight, database may have partial data
- Requires cleanup logic for abandoned analyses
- Not truly atomic (but acceptable for MVP)

**Cleanup strategy for abandoned analyses:**

```sql
-- Cron job runs every hour
SELECT id FROM analyses
WHERE status = 'processing'
  AND updated_at < NOW() - INTERVAL '1 hour';

-- For each abandoned analysis:
UPDATE analyses
SET status = 'failed',
    error_message = 'Analysis timed out or was abandoned',
    processing_stage = NULL
WHERE id = :analysis_id;
```

---

## Status Updates & Progress Visibility

### Database Schema for Stage Tracking

The `analyses` table includes a `processing_stage` field to track orchestration progress:

```sql
ALTER TABLE analyses 
ADD COLUMN processing_stage TEXT;

-- Valid values:
-- 'mapping' - CliftonStrengths → Octalysis mapping
-- 'goal_analysis' - Processing goals
-- 'synthesis' - Combining outputs
-- 'pdf_generation' - Rendering PDF
-- NULL - Not processing or complete
```

**Stage update pattern:**

```typescript
// Stage updates happen at each major transition
await db.analyses.update({
  id: analysisId,
  processing_stage: 'goal_analysis'
});

// ... goal analysis happens ...

await db.analyses.update({
  id: analysisId,
  processing_stage: 'synthesis'
});

// ... synthesis happens ...

await db.analyses.update({
  id: analysisId,
  processing_stage: 'pdf_generation'
});

// ... PDF generation happens ...

await db.analyses.update({
  id: analysisId,
  processing_stage: null,
  status: 'complete'
});
```

**Total database writes during processing:** 5-6 stage updates (acceptable overhead)

---

### Frontend Polling Pattern

The frontend polls the analysis status every 3 seconds to show progress to users.

```typescript
// Frontend polling logic (React example)
useEffect(() => {
  if (status !== 'processing') return;
  
  const pollInterval = setInterval(async () => {
    const response = await fetch(`/api/v1/analyses/${analysisId}/status`);
    const data = await response.json();
    
    setAnalysisStatus(data);
    
    // Stop polling when complete or failed
    if (data.status === 'complete' || data.status === 'failed') {
      clearInterval(pollInterval);
    }
  }, 3000); // Poll every 3 seconds
  
  return () => clearInterval(pollInterval);
}, [analysisId, status]);
```

**Status endpoint response:**

```json
{
  "id": "analysis-uuid",
  "status": "processing",
  "processingStage": "synthesis",
  "goals": [
    {
      "id": "goal-1-uuid",
      "priority": "high",
      "description": "Run a 5K",
      "processingStatus": "complete"
    },
    {
      "id": "goal-2-uuid",
      "priority": "medium",
      "description": "Learn Spanish",
      "processingStatus": "complete"
    },
    {
      "id": "goal-3-uuid",
      "priority": "low",
      "description": "Read 12 books",
      "processingStatus": "processing"
    }
  ],
  "createdAt": "2026-02-15T10:30:00Z",
  "submittedAt": "2026-02-15T10:35:00Z",
  "estimatedCompletionAt": "2026-02-15T10:37:00Z"
}
```

**Why 3-second polling:**
- Balances responsiveness vs database load
- ~40 requests per 2-minute analysis (acceptable)
- Could upgrade to Supabase Realtime later without schema changes

---

### User-Facing Progress Messages

Frontend maps processing stages to user-friendly messages:

```typescript
const STAGE_MESSAGES = {
  mapping: 'Analyzing your CliftonStrengths profile...',
  goal_analysis: 'Creating personalized strategies for your goals...',
  synthesis: 'Combining everything into a cohesive plan...',
  pdf_generation: 'Generating your PDF guide...',
};

// Per-goal status messages
const GOAL_STATUS_MESSAGES = {
  pending: 'Waiting to start...',
  processing: 'Analyzing...',
  complete: '✓ Complete',
  failed: '✗ Failed',
};
```

**Example UI state progression:**

```
[10:35:00] Analysis started
[10:35:02] Analyzing your CliftonStrengths profile...
[10:35:15] Creating personalized strategies for your goals...
           → Goal 1: Analyzing... (high priority)
           → Goal 2: Analyzing... (medium priority)
           → Goal 3: Waiting to start... (low priority)
[10:35:45] Creating personalized strategies for your goals...
           → Goal 1: ✓ Complete (high priority)
           → Goal 2: ✓ Complete (medium priority)
           → Goal 3: Analyzing... (low priority)
[10:36:15] Combining everything into a cohesive plan...
[10:36:45] Generating your PDF guide...
[10:37:05] ✓ Your personalized strategy is ready!
```

---

## Timeout Management

### Timeout Hierarchy

The orchestration implements timeouts at multiple levels to prevent runaway executions:

```typescript
const TIMEOUTS = {
  // Per-task timeouts (with 10-25% buffer over expected time)
  cliftonParsing: 10_000,        // 10s (expected: 5s)
  octalysisMapping: 15_000,      // 15s (expected: 10s)
  goalAgent: 45_000,             // 45s (expected: 30-40s)
  synthesizer: 35_000,           // 35s (expected: 30s)
  pdfGeneration: 25_000,         // 25s (expected: 20s)
  
  // Orchestration timeout (80% of Vercel limit for safety)
  totalOrchestration: 240_000,   // 240s = 4 minutes (Vercel Pro: 300s limit)
};
```

**Individual task timeout implementation:**

```typescript
async function analyzeGoalWithTimeout(
  goal: Goal,
  profile: CliftonProfile,
  mapping: OctalysisMapping,
  context: UserContext
): Promise<GoalAgentOutput> {
  
  const timeoutPromise = new Promise<never>((_, reject) => {
    setTimeout(() => {
      reject(new Error('GOAL_AGENT_TIMEOUT'));
    }, TIMEOUTS.goalAgent);
  });
  
  try {
    return await Promise.race([
      analyzeGoal(goal, profile, mapping, context),
      timeoutPromise
    ]);
  } catch (error) {
    if (error.message === 'GOAL_AGENT_TIMEOUT') {
      throw new Error(`Goal analysis timed out after ${TIMEOUTS.goalAgent}ms`);
    }
    throw error;
  }
}
```

**Overall orchestration timeout:**

```typescript
async function orchestrateAnalysisWithTimeout(
  analysisId: string,
  userId: string
) {
  const timeoutPromise = createTimeout(
    TIMEOUTS.totalOrchestration,
    'ORCHESTRATION_TIMEOUT'
  );
  
  try {
    await Promise.race([
      runOrchestration(analysisId, userId, logger),
      timeoutPromise
    ]);
  } catch (error) {
    if (error.message === 'ORCHESTRATION_TIMEOUT') {
      await handleOrchestrationTimeout(analysisId);
    }
    throw error;
  }
}

function createTimeout(ms: number, message: string): Promise<never> {
  return new Promise((_, reject) => {
    setTimeout(() => reject(new Error(message)), ms);
  });
}
```

---

### Timeout Handling Strategies

**Goal Agent timeout:**
- **Action:** Treat as retriable error, attempt retry
- **After retries:** Mark goal as failed, continue with other goals
- **User message:** "We had trouble creating a strategy for [Goal Name]. Try again?"

**Synthesizer timeout:**
- **Action:** Retry once with reduced `max_tokens` (4000 instead of 6000)
- **After retry:** Fail entire analysis (synthesis is critical)
- **User message:** "We had trouble combining your strategies. Please try again."

**Overall orchestration timeout (240s):**
- **Action:** Save partial results (what was completed successfully)
- **Mark as:** `status='failed'`, `error_message='Analysis timed out'`
- **Store:** Whatever outputs exist in `ai_agent_outputs` table
- **User message:** "Your analysis took too long. Please try again with fewer goals."

**Timeout recovery scenario:**

```typescript
async function handleOrchestrationTimeout(analysisId: string) {
  // Check what was completed before timeout
  const completedGoals = await db.goals.findMany({
    analysis_id: analysisId,
    processing_status: 'complete'
  });
  
  if (completedGoals.length > 0) {
    // We have some completed goals, save partial results
    await db.analyses.update({
      id: analysisId,
      status: 'partial_failure',
      error_message: 'Analysis timed out, but some goals completed successfully',
      error_details: {
        completedGoalIds: completedGoals.map(g => g.id),
        timeoutAt: 'orchestration',
        durationMs: 240_000
      }
    });
  } else {
    // Nothing completed, complete failure
    await db.analyses.update({
      id: analysisId,
      status: 'failed',
      error_message: 'Analysis timed out before any goals completed',
      error_details: {
        timeoutAt: 'orchestration',
        durationMs: 240_000
      }
    });
  }
}
```

---

### Scaling for Upgraded Plans

If the Vercel plan is upgraded (e.g., Enterprise with 900s limit), adjust timeouts proportionally:

```typescript
// Environment-based timeout configuration
const VERCEL_TIMEOUT_LIMIT = process.env.VERCEL_TIMEOUT_LIMIT 
  ? parseInt(process.env.VERCEL_TIMEOUT_LIMIT) 
  : 300_000; // Default: 300s (Pro plan)

const SAFETY_BUFFER_PERCENTAGE = 0.2; // 20% buffer

const TIMEOUTS = {
  // Orchestration timeout: 80% of Vercel limit
  totalOrchestration: VERCEL_TIMEOUT_LIMIT * (1 - SAFETY_BUFFER_PERCENTAGE),
  
  // Per-task timeouts remain fixed (based on AI model performance)
  goalAgent: 45_000,
  synthesizer: 35_000,
  pdfGeneration: 25_000,
};

// Example: Enterprise plan (900s limit)
// totalOrchestration = 900,000 * 0.8 = 720,000ms (12 minutes)
```

**Best practice:** Always leave 20% safety buffer for cleanup and shutdown.

---

## Cost Tracking & Metrics

### Real-Time Cost Aggregation

Costs are tracked and aggregated as each AI task completes:

```typescript
async function storeAgentOutput(
  analysisId: string,
  goalId: string | null,
  agentType: AgentType,
  output: any,
  metadata: AIMetadata
) {
  // Store agent output with metrics
  await db.ai_agent_outputs.insert({
    analysis_id: analysisId,
    goal_id: goalId,
    agent_type: agentType,
    output_text: output,
    tokens_used: metadata.tokensUsed,
    cost_usd: metadata.costUSD,
    duration_ms: metadata.durationMs,
    model_name: metadata.modelName,
    prompt_version: metadata.promptVersion
  });
  
  // Update running total in analyses table (for quick access)
  await db.raw(`
    UPDATE analyses
    SET 
      total_tokens_used = total_tokens_used + :tokens,
      total_cost_usd = total_cost_usd + :cost,
      total_duration_ms = total_duration_ms + :duration
    WHERE id = :analysisId
  `, {
    tokens: metadata.tokensUsed,
    cost: metadata.costUSD,
    duration: metadata.durationMs,
    analysisId
  });
}
```

**Cost tracking per agent type:**

| Agent Type | Model | Expected Tokens | Expected Cost |
|------------|-------|-----------------|---------------|
| CliftonStrengths Parser | Gemini Flash 2.0 | ~500 | $0.001 (free tier) |
| Octalysis Mapping | Gemini Flash 2.0 | ~800 | $0.001 (free tier) |
| Goal Agent (x3) | Claude Sonnet 4 | ~3,500 each | $0.0157 each |
| Synthesizer | Claude Sonnet 4 | ~4,000 | $0.018 |
| **Total per analysis** | | **~15,300** | **~$0.065** |

**Cost tracking on failure:**

```typescript
// If analysis fails midway, still track costs incurred
async function handleOrchestrationFailure(
  analysisId: string,
  error: Error,
  logger: Logger
) {
  // Costs are already tracked incrementally in total_cost_usd
  // Just mark as failed
  await db.analyses.update({
    id: analysisId,
    status: 'failed',
    error_message: error.message,
    error_details: {
      stack: error.stack,
      failedAt: logger.getCurrentStage()
    }
  });
  
  // Don't trigger cleanup (keep outputs for debugging)
  logger.error('Analysis failed', {
    error: error.message,
    totalCostUSD: await getTotalCost(analysisId)
  });
}

async function getTotalCost(analysisId: string): Promise<number> {
  const analysis = await db.analyses.findOne({ id: analysisId });
  return analysis?.total_cost_usd || 0;
}
```

---

### Conditional Cleanup Trigger

Agent outputs are automatically cleaned up on success via database trigger:

```sql
CREATE FUNCTION cleanup_agent_outputs()
RETURNS TRIGGER AS $$
BEGIN
  -- When analysis completes successfully, soft-delete agent outputs
  -- (keep metrics in analyses table)
  IF NEW.status = 'complete' AND OLD.status != 'complete' THEN
    UPDATE ai_agent_outputs
    SET deleted_at = NOW()
    WHERE analysis_id = NEW.id
      AND deleted_at IS NULL;
  END IF;
  
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_cleanup_agent_outputs
  AFTER UPDATE ON analyses
  FOR EACH ROW
  EXECUTE FUNCTION cleanup_agent_outputs();
```

**Cleanup behavior:**

| Analysis Status | Agent Outputs | Metrics Retained |
|----------------|---------------|------------------|
| `processing` | Full outputs stored | Running totals in `analyses` |
| `complete` | Soft-deleted (`deleted_at` set) | ✓ In `analyses.total_*` fields |
| `failed` | Kept for debugging | ✓ In `analyses.total_*` fields |
| `partial_failure` | Kept for debugging | ✓ In `analyses.total_*` fields |

**Hard delete old soft-deleted outputs (future cron job):**

```sql
-- Run monthly to free up database space
DELETE FROM ai_agent_outputs
WHERE deleted_at IS NOT NULL
  AND deleted_at < NOW() - INTERVAL '30 days';
```

---

## Observability & Debugging

### Structured Logging

Orchestration logs to Vercel console using structured JSON format:

```typescript
interface Logger {
  stage(stage: string, metadata?: object): void;
  info(message: string, metadata?: object): void;
  warn(message: string, metadata?: object): void;
  error(message: string, metadata?: object): void;
  success(message: string, metadata?: object): void;
  debug(message: string, metadata?: object): void;
}

function createLogger(analysisId: string): Logger {
  const logBase = {
    analysisId,
    orchestrationVersion: '1.0',
  };
  
  let currentStage: string | null = null;
  
  return {
    stage: (stage: string, metadata?: object) => {
      currentStage = stage;
      console.log(JSON.stringify({
        ...logBase,
        level: 'stage',
        stage,
        timestamp: new Date().toISOString(),
        ...metadata
      }));
    },
    
    info: (message: string, metadata?: object) => {
      console.log(JSON.stringify({
        ...logBase,
        level: 'info',
        stage: currentStage,
        message,
        timestamp: new Date().toISOString(),
        ...metadata
      }));
    },
    
    warn: (message: string, metadata?: object) => {
      console.warn(JSON.stringify({
        ...logBase,
        level: 'warn',
        stage: currentStage,
        message,
        timestamp: new Date().toISOString(),
        ...metadata
      }));
    },
    
    error: (message: string, metadata?: object) => {
      console.error(JSON.stringify({
        ...logBase,
        level: 'error',
        stage: currentStage,
        message,
        timestamp: new Date().toISOString(),
        ...metadata
      }));
    },
    
    success: (message: string, metadata?: object) => {
      console.log(JSON.stringify({
        ...logBase,
        level: 'success',
        stage: currentStage,
        message,
        timestamp: new Date().toISOString(),
        ...metadata
      }));
    },
    
    debug: (message: string, metadata?: object) => {
      if (process.env.NODE_ENV === 'development') {
        console.debug(JSON.stringify({
          ...logBase,
          level: 'debug',
          stage: currentStage,
          message,
          timestamp: new Date().toISOString(),
          ...metadata
        }));
      }
    },
    
    getCurrentStage: () => currentStage
  };
}
```

**Example log output:**

```json
{"analysisId":"abc-123","orchestrationVersion":"1.0","level":"stage","stage":"goal_analysis_start","timestamp":"2026-02-15T10:35:15.234Z","goalCount":3}
{"analysisId":"abc-123","orchestrationVersion":"1.0","level":"info","stage":"goal_analysis_start","message":"Goal 1 attempt 1/2","timestamp":"2026-02-15T10:35:15.456Z"}
{"analysisId":"abc-123","orchestrationVersion":"1.0","level":"warn","stage":"goal_analysis_start","message":"Goal 2 attempt 1 failed","timestamp":"2026-02-15T10:35:45.123Z","error":"Claude API timeout","retriable":true}
{"analysisId":"abc-123","orchestrationVersion":"1.0","level":"stage","stage":"goal_analysis_complete","timestamp":"2026-02-15T10:36:15.789Z","succeeded":3,"failed":0}
```

---

### Database Storage for Failed Orchestrations

On analysis failure, store orchestration trace in `error_details` JSONB field:

```typescript
async function handleOrchestrationFailure(
  analysisId: string,
  error: Error,
  logger: Logger
) {
  // Capture full orchestration trace
  const orchestrationTrace = logger.getFullTrace(); // Implement as needed
  
  await db.analyses.update({
    id: analysisId,
    status: 'failed',
    error_message: error.message,
    error_details: {
      errorType: error.name,
      stack: error.stack,
      failedAt: logger.getCurrentStage(),
      orchestrationTrace: orchestrationTrace.slice(-20), // Last 20 log entries
      partialResults: {
        completedGoals: await getCompletedGoalIds(analysisId),
        totalCostUSD: await getTotalCost(analysisId),
        durationMs: Date.now() - logger.getStartTime()
      }
    }
  });
}
```

**Example `error_details` JSONB:**

```json
{
  "errorType": "Error",
  "stack": "Error: Synthesis failed: Claude API timeout\n    at synthesizeRecommendations...",
  "failedAt": "synthesis",
  "orchestrationTrace": [
    {"stage": "goal_analysis_complete", "timestamp": "2026-02-15T10:36:15.789Z"},
    {"stage": "synthesis_start", "timestamp": "2026-02-15T10:36:16.123Z"},
    {"level": "error", "message": "Synthesis timed out", "timestamp": "2026-02-15T10:36:51.456Z"}
  ],
  "partialResults": {
    "completedGoals": ["goal-1-uuid", "goal-2-uuid", "goal-3-uuid"],
    "totalCostUSD": 0.047,
    "durationMs": 95234
  }
}
```

---

### Vercel Analytics Integration

Track key orchestration metrics for aggregate analysis:

```typescript
// Install: @vercel/analytics
import { track } from '@vercel/analytics/server';

// Track analysis lifecycle events
await track('analysis_started', {
  analysisId,
  goalCount: goals.length,
  hasQuestionnaire: !!analysis.questionnaire_responses
});

await track('analysis_completed', {
  analysisId,
  goalCount: goals.length,
  durationMs: totalDuration,
  totalCost: totalCost,
  status: finalStatus // 'complete' | 'partial_failure'
});

await track('analysis_failed', {
  analysisId,
  goalCount: goals.length,
  failedAt: currentStage,
  errorType: error.name
});
```

**Metrics dashboard queries:**
- Success rate: `COUNT(analysis_completed) / COUNT(analysis_started)`
- Average duration: `AVG(analysis_completed.durationMs)`
- Cost per analysis: `AVG(analysis_completed.totalCost)`
- Failure points: `GROUP BY(analysis_failed.failedAt)`

---

## Implementation Patterns

### Helper Functions

**Sleep utility for retry backoff:**

```typescript
function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

**User context builder:**

```typescript
function buildUserContext(analysis: Analysis): UserContext {
  return {
    weeklyTimeInvestment: analysis.weekly_time_investment,
    questionnaireResponses: analysis.questionnaire_responses || {},
    totalAvailableHours: calculateTotalHours(analysis.weekly_time_investment)
  };
}

function calculateTotalHours(investment: any): number {
  const categories = [
    'work_career',
    'health_fitness',
    'relationships_family',
    'learning_growth',
    'creativity_hobbies',
    'rest_recovery',
    'community_service',
    'admin_maintenance'
  ];
  
  return categories.reduce((total, category) => {
    return total + (investment[category] || 0);
  }, 0);
}
```

**CliftonStrengths profile builder:**

```typescript
function buildCliftonProfile(analysis: Analysis): CliftonProfile {
  return {
    topTenThemes: analysis.clifton_top_10,
    bottomFiveThemes: analysis.clifton_bottom_5,
    dominantDomain: analysis.clifton_dominant_domain
  };
}
```

**Cost calculator:**

```typescript
function calculateTotalCost(
  goalResults: GoalAnalysisResult[],
  synthesisResult: SynthesizerOutput
): number {
  const goalCosts = goalResults
    .filter(r => r.success)
    .reduce((sum, r) => sum + r.metadata.costUSD, 0);
  
  const synthesisCost = synthesisResult.metadata.costUSD;
  
  return goalCosts + synthesisCost;
}

function calculateTotalTokens(
  goalResults: GoalAnalysisResult[],
  synthesisResult: SynthesizerOutput
): number {
  const goalTokens = goalResults
    .filter(r => r.success)
    .reduce((sum, r) => sum + r.metadata.tokensUsed, 0);
  
  const synthesisTokens = synthesisResult.metadata.tokensUsed;
  
  return goalTokens + synthesisTokens;
}
```

---

### Synthesis Wrapper with Retry

```typescript
async function synthesizeRecommendationsWithRetry(
  goalOutputs: GoalAnalysisResult[],
  analysis: Analysis,
  octalysisMapping: OctalysisMapping
): Promise<SynthesizerOutput> {
  
  const maxRetries = 2;
  let lastError: Error | null = null;
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      // Build synthesis context
      const synthesisContext = {
        goals: goalOutputs.map(r => ({
          goalId: r.goalId,
          output: r.data
        })),
        cliftonProfile: buildCliftonProfile(analysis),
        userContext: buildUserContext(analysis),
        octalysisMapping
      };
      
      // Call synthesizer
      const result = await synthesizeRecommendations(
        synthesisContext.goals,
        synthesisContext.cliftonProfile,
        synthesisContext.userContext,
        analysis.id
      );
      
      if (!result.success) {
        throw new Error(result.error.message);
      }
      
      return result;
      
    } catch (error) {
      lastError = error;
      
      // If timeout on first attempt, retry with reduced max_tokens
      if (error.message.includes('timeout') && attempt === 1) {
        console.warn('Synthesis timed out, retrying with reduced max_tokens');
        // Next attempt will use reduced tokens (implement in synthesizer)
        await sleep(2000);
        continue;
      }
      
      // If non-retriable or last attempt, throw
      const classification = classifyError(error);
      if (!classification.shouldRetry || attempt === maxRetries) {
        throw error;
      }
      
      await sleep(2000 * attempt);
    }
  }
  
  throw lastError || new Error('Synthesis failed for unknown reason');
}
```

---

## Testing Strategy

### Unit Tests

Test individual orchestration components:

```typescript
// tests/orchestration/retry-logic.test.ts
describe('analyzeGoalWithRetry', () => {
  it('should succeed on first attempt', async () => {
    const mockGoal = createMockGoal();
    const result = await analyzeGoalWithRetry(mockGoal, profile, mapping, context, logger, { maxRetries: 2 });
    expect(result.success).toBe(true);
  });
  
  it('should retry on retriable error', async () => {
    const mockError = new Error('ECONNRESET');
    jest.spyOn(aiModule, 'analyzeGoal')
      .mockRejectedValueOnce(mockError)
      .mockResolvedValueOnce({ success: true, data: {} });
    
    const result = await analyzeGoalWithRetry(mockGoal, profile, mapping, context, logger, { maxRetries: 2 });
    expect(result.success).toBe(true);
    expect(aiModule.analyzeGoal).toHaveBeenCalledTimes(2);
  });
  
  it('should fail immediately on non-retriable error', async () => {
    const mockError = new Error('Validation failed');
    mockError.code = '400';
    jest.spyOn(aiModule, 'analyzeGoal').mockRejectedValue(mockError);
    
    await expect(
      analyzeGoalWithRetry(mockGoal, profile, mapping, context, logger, { maxRetries: 2 })
    ).rejects.toThrow('Validation failed');
    expect(aiModule.analyzeGoal).toHaveBeenCalledTimes(1);
  });
});
```

### Integration Tests

Test full orchestration flow with mocked AI APIs:

```typescript
// tests/orchestration/full-flow.test.ts
describe('orchestrateAnalysis', () => {
  beforeEach(() => {
    // Mock AI API responses
    mockClaudeAPI();
    mockGeminiAPI();
  });
  
  it('should complete successfully with 3 goals', async () => {
    const analysisId = await createTestAnalysis(3);
    
    await orchestrateAnalysis(analysisId, userId);
    
    const analysis = await db.analyses.findOne({ id: analysisId });
    expect(analysis.status).toBe('complete');
    expect(analysis.output_pdf_path).toBeTruthy();
    expect(analysis.total_cost_usd).toBeGreaterThan(0);
  });
  
  it('should handle partial failure gracefully', async () => {
    const analysisId = await createTestAnalysis(3);
    
    // Mock Goal 2 to fail
    jest.spyOn(aiModule, 'analyzeGoal')
      .mockResolvedValueOnce({ success: true }) // Goal 1
      .mockRejectedValue(new Error('API timeout')) // Goal 2 (all attempts)
      .mockResolvedValueOnce({ success: true }); // Goal 3
    
    await orchestrateAnalysis(analysisId, userId);
    
    const analysis = await db.analyses.findOne({ id: analysisId });
    expect(analysis.status).toBe('partial_failure');
    
    const goals = await db.goals.findMany({ analysis_id: analysisId });
    expect(goals.filter(g => g.processing_status === 'complete').length).toBe(2);
    expect(goals.filter(g => g.processing_status === 'failed').length).toBe(1);
  });
  
  it('should handle orchestration timeout', async () => {
    const analysisId = await createTestAnalysis(3);
    
    // Mock slow AI response
    jest.spyOn(aiModule, 'analyzeGoal').mockImplementation(() => 
      new Promise(resolve => setTimeout(() => resolve({ success: true }), 300_000))
    );
    
    await expect(
      orchestrateAnalysis(analysisId, userId)
    ).rejects.toThrow('ORCHESTRATION_TIMEOUT');
    
    const analysis = await db.analyses.findOne({ id: analysisId });
    expect(analysis.status).toBe('failed');
    expect(analysis.error_message).toContain('timed out');
  });
});
```

### Load Testing

Test orchestration under concurrent load:

```typescript
// tests/orchestration/load.test.ts
describe('Concurrent orchestration', () => {
  it('should handle 10 concurrent analyses', async () => {
    const analysisIds = await Promise.all(
      Array(10).fill(null).map(() => createTestAnalysis(2))
    );
    
    const results = await Promise.allSettled(
      analysisIds.map(id => orchestrateAnalysis(id, userId))
    );
    
    const succeeded = results.filter(r => r.status === 'fulfilled').length;
    expect(succeeded).toBeGreaterThan(7); // At least 70% success rate
  });
});
```

---

## Connections to Other Documents

This orchestration architecture integrates with:

**AI Task Specifications (Document 3):**
- Implements execution patterns for all 7 AI tasks
- Uses exact input/output formats defined in Task Specs
- Follows model selection strategy (Gemini Flash vs Claude Sonnet 4)
- Reference: Section on Goal Agent, Synthesizer, Parser specifications

**Database Schema & Data Models (Document 2):**
- Uses `analyses`, `goals`, and `ai_agent_outputs` tables as defined
- Implements conditional cleanup via database trigger
- Follows RLS policies for security
- Updates `processing_stage` field for status tracking
- Reference: Table definitions, conditional storage strategy

**API Specification (Document 6):**
- Powers `/api/v1/analyses/:id/submit` endpoint
- Provides status updates via `/api/v1/analyses/:id/status` polling
- Follows error response format standards
- Implements rate limit checks before orchestration
- Reference: Analysis submission and status endpoints

**Frontend Architecture (Document 8):**
- Frontend polls status every 3 seconds during processing
- Displays progress using `processing_stage` field
- Shows per-goal status using `goals.processing_status`
- Reference: Status polling patterns, progress indicators

**Security & Authentication (Document 7):**
- Uses Supabase session validation before orchestration
- Verifies analysis ownership before processing
- Implements RLS policies at database level
- Reference: Authentication middleware patterns

**Prompt Engineering Specification (Document 4):**
- Loads prompts from database `prompts` table
- Uses versioned prompts for each AI task
- Tracks `prompt_version` in agent outputs
- Reference: Prompt storage and retrieval patterns

**PDF Template & Design (Document 5):**
- Synthesizer output feeds directly into PDF generation
- Follows PDF structure defined in template spec
- Reference: PDF content sections and data format

---

## Summary

This AI Orchestration Architecture provides a resilient, efficient system for coordinating 7 AI tasks within Vercel's 300-second execution limit. Key design decisions prioritize:

1. **Performance:** Pre-caching slow tasks, parallel execution with staggering
2. **Resilience:** Multi-level retry logic, partial failure handling, graceful degradation
3. **Observability:** Structured logging, stage tracking, cost monitoring
4. **Maintainability:** Optimistic database updates, clear error classification, testable components

**Performance Expectations:**
- Target: ~112 seconds (3 goals, mapping pre-cached)
- Worst case: ~150 seconds (with retries)
- Safety buffer: 150 seconds under 300s limit

**Cost Expectations:**
- Per analysis: ~$0.051 ($0.05 AI + $0.001 Vercel)
- Per user per month (2 analyses): ~$0.10
- At 100 users: ~$10/month
- At 1,000 users: ~$100/month

**Next Steps:**
- Implement orchestration function using these patterns
- Write unit and integration tests
- Deploy to Vercel with monitoring
- Validate performance under real load
- Iterate based on failure patterns

---

**Document Status:** Complete and ready for Claude Code implementation  
**Version:** 1.0  
**Last Updated:** February 15, 2026
