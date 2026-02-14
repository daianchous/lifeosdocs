# Frontend Architecture & State Management
## Personal Time OS - Goal Achievement MVP

**Version:** 1.0  
**Created:** February 15, 2026  
**Purpose:** Complete frontend architecture specification for React/Next.js implementation  
**Target:** Claude Code implementation with clear patterns and examples

---

## Executive Summary

This document defines the complete frontend architecture for the Goal Achievement MVP, a mobile-first web application built with Next.js 14+, React 18+, TypeScript, and React Query. The architecture prioritizes clarity over complexity, performance on mobile networks, and warm user experience aligned with the "supportive friend" brand tone.

**Key Architecture Decisions:**
- **Framework:** Next.js 14+ App Router for SEO, image optimization, and file-based routing
- **State Management:** React Context + Custom Hooks for global state, React Query for server state
- **Form Handling:** react-hook-form with progressive validation and warm error messages
- **File Upload:** Direct to Supabase Storage with React Query polling for parsing status
- **Error Tracking:** Sentry integration from day one for production monitoring
- **Mobile-First:** Single column layouts, 44px touch targets, <250KB initial bundle
- **TypeScript:** Strict mode enabled to prevent runtime errors and improve DX

**Core Principles:**
1. **Mobile-first, always** - Every component designed for small screens first
2. **Progressive enhancement** - Core functionality works, enhancements layer on
3. **Warm and forgiving** - Errors are helpful, not punishing; warnings are encouraging
4. **State persistence** - Users can leave and return without losing progress (Step 2+)
5. **Performance matters** - Fast initial load on slow networks, optimistic UI updates
6. **Type safety** - Strict TypeScript prevents entire classes of bugs

**Technology Stack:**
```
Frontend Framework:  Next.js 14+ (App Router)
UI Library:          React 18+
Language:            TypeScript 5+ (strict mode)
State Management:    React Context + React Query
Form Management:     react-hook-form + zod
Styling:             Tailwind CSS 3+
UI Components:       shadcn/ui (optional, for complex components)
API Client:          React Query + Fetch API
Authentication:      Supabase Auth SDK
File Upload:         Supabase Storage SDK
Error Tracking:      Sentry
Analytics:           Vercel Analytics (built-in)
```

---

## Project Structure

### Directory Organization

```
/goal-achievement-mvp
├── /app                          # Next.js App Router
│   ├── /api                      # API routes (proxies to backend)
│   │   └── /v1                   # Versioned API endpoints
│   ├── /(auth)                   # Auth route group
│   │   ├── /login                # Login page
│   │   └── /callback             # OAuth callback
│   ├── /(dashboard)              # Authenticated route group
│   │   ├── /assessment           # Assessment flow
│   │   │   ├── /step-1           # Step 1: Profile setup
│   │   │   ├── /step-2           # Step 2: Goal setting
│   │   │   └── /processing       # Processing status
│   │   ├── /preview              # PDF preview & download
│   │   └── /history              # Analysis history (future)
│   ├── layout.tsx                # Root layout
│   ├── page.tsx                  # Landing page
│   └── error.tsx                 # Global error boundary
├── /components
│   ├── /assessment               # Assessment-specific components
│   │   ├── StepOne.tsx           # Step 1 main component
│   │   ├── StepTwo.tsx           # Step 2 main component
│   │   ├── FileUpload.tsx        # PDF upload with parsing status
│   │   ├── GoalForm.tsx          # Individual goal form
│   │   ├── ProcessingStages.tsx  # ~130s processing UI
│   │   └── PreviewScreen.tsx     # PDF preview before download
│   ├── /common                   # Shared UI components
│   │   ├── Button.tsx            # Primary/secondary buttons
│   │   ├── Input.tsx             # Form inputs
│   │   ├── Select.tsx            # Dropdowns
│   │   ├── Checkbox.tsx          # Multi-select checkboxes
│   │   ├── RadioGroup.tsx        # Radio button groups
│   │   ├── DatePicker.tsx        # Mobile-friendly date picker
│   │   ├── Slider.tsx            # Hardness rating slider
│   │   ├── LoadingSpinner.tsx    # Loading states
│   │   ├── ErrorMessage.tsx      # Warm error messages
│   │   ├── ProgressBar.tsx       # Step progress indicator
│   │   └── Modal.tsx             # Modals (priority constraint)
│   ├── /layout                   # Layout components
│   │   ├── Header.tsx            # App header
│   │   ├── Footer.tsx            # App footer
│   │   └── Container.tsx         # Max-width container
│   └── /landing                  # Landing page components
│       ├── Hero.tsx              # Hero section
│       ├── HowItWorks.tsx        # 3-step process
│       ├── RateLimitBadge.tsx    # Rate limit info
│       └── SocialProof.tsx       # Testimonials
├── /lib
│   ├── /api                      # API client layer
│   │   ├── client.ts             # Base fetch wrapper
│   │   ├── analyses.ts           # Analysis endpoints
│   │   ├── goals.ts              # Goal endpoints
│   │   └── users.ts              # User endpoints
│   ├── /hooks                    # Custom React hooks
│   │   ├── useAuth.ts            # Authentication hook
│   │   ├── useFileUpload.ts      # File upload + parsing
│   │   ├── useAssessment.ts      # Assessment state
│   │   ├── useAnalysis.ts        # Analysis status polling
│   │   ├── useAutoSave.ts        # Debounced auto-save
│   │   └── useRateLimit.ts       # Rate limit checking
│   ├── /contexts                 # React contexts
│   │   ├── AuthContext.tsx       # Auth state provider
│   │   └── AssessmentContext.tsx # Assessment state provider
│   ├── /utils                    # Utility functions
│   │   ├── validation.ts         # Form validation rules
│   │   ├── formatting.ts         # Date/time formatting
│   │   └── constants.ts          # App constants
│   ├── supabase.ts               # Supabase client
│   └── sentry.ts                 # Sentry configuration
├── /types                        # TypeScript types
│   ├── api.ts                    # API response types
│   ├── assessment.ts             # Assessment types
│   ├── goal.ts                   # Goal types
│   └── user.ts                   # User types
├── /public                       # Static assets
│   ├── /images                   # Images
│   └── /fonts                    # Custom fonts (if any)
├── .env.local                    # Environment variables
├── next.config.js                # Next.js configuration
├── tailwind.config.js            # Tailwind configuration
├── tsconfig.json                 # TypeScript configuration
└── package.json                  # Dependencies
```

### File Naming Conventions

- **Components:** PascalCase (e.g., `FileUpload.tsx`)
- **Hooks:** camelCase with `use` prefix (e.g., `useFileUpload.ts`)
- **Utilities:** camelCase (e.g., `validation.ts`)
- **Types:** PascalCase for interfaces (e.g., `AssessmentData`)
- **Routes:** kebab-case (e.g., `/step-1`)

---

## Component Architecture

### Component Hierarchy

```
App
├── Layout (Root)
│   ├── Header
│   │   ├── Logo
│   │   └── UserAvatar (if authenticated)
│   ├── Main Content (Page)
│   │   └── [Page-specific components]
│   └── Footer
│
Pages:
├── Landing (/)
│   ├── Hero
│   ├── HowItWorks
│   ├── RateLimitBadge
│   └── SocialProof
│
├── Assessment Step 1 (/assessment/step-1)
│   ├── ProgressBar (1 of 2)
│   ├── FileUpload
│   │   ├── Dropzone
│   │   ├── ParsingStatus
│   │   └── ErrorFeedback
│   ├── TimeInvestment
│   │   ├── CategoryInput (x8)
│   │   └── TotalHours
│   └── OptionalQuestionnaire (accordion)
│
├── Assessment Step 2 (/assessment/step-2)
│   ├── ProgressBar (2 of 2)
│   ├── GoalForm (Goal 1, required)
│   │   ├── TextInput (description)
│   │   ├── RadioGroup (priority)
│   │   ├── CheckboxGroup (categories, 1-3)
│   │   ├── DatePicker (deadline)
│   │   └── Slider (hardness, 1-5)
│   ├── GoalForm (Goal 2, optional)
│   ├── GoalForm (Goal 3, optional)
│   ├── PriorityModal (if constraint violated)
│   └── AnalyseButton (with parsing status check)
│
├── Processing (/assessment/processing)
│   ├── ProcessingStages
│   │   ├── StageIndicator
│   │   ├── EncouragingMessage
│   │   └── TimeEstimate
│   └── BrowserCloseSafeNotice
│
└── Preview (/preview/:analysisId)
    ├── PDFMetadata
    ├── DownloadButton
    └── TelegramCTA
```

### Component Design Patterns

**1. Container/Presentation Pattern**

```typescript
// Container (handles logic)
export default function StepOnePage() {
  const { uploadFile, isUploading } = useFileUpload();
  const { saveAssessment } = useAssessment();
  
  return (
    <StepOne
      onFileUpload={uploadFile}
      isUploading={isUploading}
      onSave={saveAssessment}
    />
  );
}

// Presentation (pure UI)
interface StepOneProps {
  onFileUpload: (file: File) => void;
  isUploading: boolean;
  onSave: (data: AssessmentData) => void;
}

export function StepOne({ onFileUpload, isUploading, onSave }: StepOneProps) {
  // Only UI rendering, no business logic
  return <div>...</div>;
}
```

**2. Compound Components Pattern (for complex UI)**

```typescript
// FileUpload compound component
export function FileUpload({ children }: { children: React.ReactNode }) {
  return <div className="file-upload">{children}</div>;
}

FileUpload.Dropzone = function Dropzone({ onSelect }: { onSelect: (file: File) => void }) {
  return <div>...</div>;
};

FileUpload.ParsingStatus = function ParsingStatus({ status }: { status: ParsingStatus }) {
  return <div>...</div>;
};

// Usage
<FileUpload>
  <FileUpload.Dropzone onSelect={handleFileSelect} />
  <FileUpload.ParsingStatus status={parsingStatus} />
</FileUpload>
```

**3. Custom Hooks for Reusable Logic**

```typescript
// useDebounce.ts
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(handler);
  }, [value, delay]);
  
  return debouncedValue;
}

// Usage in component
const debouncedGoals = useDebounce(goals, 1000);
useEffect(() => {
  if (debouncedGoals) saveAssessmentDraft(debouncedGoals);
}, [debouncedGoals]);
```

---

## State Management Strategy

### Overview

We use a **layered state management approach**:

1. **Global State** (React Context) - Authentication, user profile
2. **Server State** (React Query) - API data, caching, polling
3. **Local State** (useState) - UI toggles, form inputs
4. **Form State** (react-hook-form) - Complex form management

### 1. Global State: React Context

**AuthContext** - Manages authentication state

```typescript
// /lib/contexts/AuthContext.tsx
import { createContext, useContext, useEffect, useState } from 'react';
import { User } from '@supabase/supabase-js';
import { supabase } from '@/lib/supabase';

interface AuthContextType {
  user: User | null;
  loading: boolean;
  signIn: () => Promise<void>;
  signOut: () => Promise<void>;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    // Get initial session
    supabase.auth.getSession().then(({ data: { session } }) => {
      setUser(session?.user ?? null);
      setLoading(false);
    });
    
    // Listen for auth changes
    const { data: { subscription } } = supabase.auth.onAuthStateChange(
      (_event, session) => {
        setUser(session?.user ?? null);
      }
    );
    
    return () => subscription.unsubscribe();
  }, []);
  
  const signIn = async () => {
    const { error } = await supabase.auth.signInWithOAuth({
      provider: 'google',
      options: {
        redirectTo: `${window.location.origin}/assessment/step-1`,
      },
    });
    if (error) throw error;
  };
  
  const signOut = async () => {
    const { error } = await supabase.auth.signOut();
    if (error) throw error;
  };
  
  return (
    <AuthContext.Provider value={{ user, loading, signIn, signOut }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
}
```

**AssessmentContext** - Manages assessment draft state

```typescript
// /lib/contexts/AssessmentContext.tsx
import { createContext, useContext, useState, useEffect } from 'react';
import { AssessmentDraft } from '@/types/assessment';

interface AssessmentContextType {
  draft: AssessmentDraft | null;
  updateDraft: (draft: Partial<AssessmentDraft>) => void;
  clearDraft: () => void;
}

const AssessmentContext = createContext<AssessmentContextType | undefined>(undefined);

export function AssessmentProvider({ children }: { children: React.ReactNode }) {
  const [draft, setDraft] = useState<AssessmentDraft | null>(null);
  
  // Load draft from server on mount
  useEffect(() => {
    // Load from API (handled by React Query hook)
  }, []);
  
  const updateDraft = (updates: Partial<AssessmentDraft>) => {
    setDraft(prev => prev ? { ...prev, ...updates } : null);
  };
  
  const clearDraft = () => {
    setDraft(null);
  };
  
  return (
    <AssessmentContext.Provider value={{ draft, updateDraft, clearDraft }}>
      {children}
    </AssessmentContext.Provider>
  );
}

export function useAssessmentDraft() {
  const context = useContext(AssessmentContext);
  if (!context) throw new Error('useAssessmentDraft must be used within AssessmentProvider');
  return context;
}
```

### 2. Server State: React Query

**React Query Setup**

```typescript
// /app/providers.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // 1 minute
        refetchOnWindowFocus: false,
        retry: 3,
      },
    },
  }));
  
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

**API Client Layer**

```typescript
// /lib/api/client.ts
const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || '/api/v1';

export class APIError extends Error {
  constructor(
    public statusCode: number,
    public code: string,
    message: string,
    public details?: any
  ) {
    super(message);
    this.name = 'APIError';
  }
}

export async function fetchAPI<T>(
  endpoint: string,
  options?: RequestInit
): Promise<T> {
  const response = await fetch(`${API_BASE_URL}${endpoint}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
    credentials: 'include', // Send cookies (Supabase session)
  });
  
  const data = await response.json();
  
  if (!response.ok) {
    throw new APIError(
      response.status,
      data.error.code,
      data.error.message,
      data.error.details
    );
  }
  
  return data;
}
```

**Analysis API Endpoints**

```typescript
// /lib/api/analyses.ts
import { fetchAPI } from './client';
import { Analysis, AnalysisStatus, CreateAnalysisRequest } from '@/types/api';

export const analysesAPI = {
  // Get rate limit status
  getRateLimit: () => 
    fetchAPI<{ remaining: number; resetAt: string }>('/users/me/rate-limit'),
  
  // Create new analysis
  create: (data: CreateAnalysisRequest) =>
    fetchAPI<{ analysisId: string }>('/analyses', {
      method: 'POST',
      body: JSON.stringify(data),
    }),
  
  // Get analysis status (for polling)
  getStatus: (analysisId: string) =>
    fetchAPI<AnalysisStatus>(`/analyses/${analysisId}/status`),
  
  // Get completed analysis
  get: (analysisId: string) =>
    fetchAPI<Analysis>(`/analyses/${analysisId}`),
  
  // Get user's analysis history
  list: () =>
    fetchAPI<{ analyses: Analysis[] }>('/analyses'),
};
```

**Custom Hooks with React Query**

```typescript
// /lib/hooks/useFileUpload.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { supabase } from '@/lib/supabase';

export function useFileUpload() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: async (file: File) => {
      // 1. Get signed upload URL from API
      const { data: urlData } = await fetchAPI<{ uploadUrl: string; fileId: string }>(
        '/files/upload-url',
        { method: 'POST' }
      );
      
      // 2. Upload directly to Supabase Storage
      const { data, error } = await supabase.storage
        .from('cliftonstrengths-pdfs')
        .upload(urlData.fileId, file);
      
      if (error) throw error;
      
      // 3. Trigger parsing
      await fetchAPI('/files/parse', {
        method: 'POST',
        body: JSON.stringify({ fileId: urlData.fileId }),
      });
      
      return { fileId: urlData.fileId };
    },
    onSuccess: (data) => {
      // Invalidate and start polling parsing status
      queryClient.invalidateQueries({ queryKey: ['parsing-status', data.fileId] });
    },
  });
}

// /lib/hooks/useParsingStatus.ts
import { useQuery } from '@tanstack/react-query';

export function useParsingStatus(fileId?: string) {
  return useQuery({
    queryKey: ['parsing-status', fileId],
    queryFn: () => fetchAPI<ParsingStatus>(`/files/${fileId}/parsing-status`),
    enabled: !!fileId,
    refetchInterval: (data) => {
      // Poll every 2s while processing
      if (data?.status === 'processing') return 2000;
      // Poll every 3s if queued
      if (data?.status === 'queued') return 3000;
      // Stop polling if complete or failed
      return false;
    },
    retry: 3,
  });
}

// /lib/hooks/useAnalysis.ts
import { useMutation, useQuery } from '@tanstack/react-query';
import { analysesAPI } from '@/lib/api/analyses';

export function useCreateAnalysis() {
  return useMutation({
    mutationFn: analysesAPI.create,
    onSuccess: (data) => {
      // Redirect to processing page
      window.location.href = `/assessment/processing?id=${data.analysisId}`;
    },
  });
}

export function useAnalysisStatus(analysisId?: string) {
  return useQuery({
    queryKey: ['analysis-status', analysisId],
    queryFn: () => analysesAPI.getStatus(analysisId!),
    enabled: !!analysisId,
    refetchInterval: (data) => {
      // Poll every 3s while processing
      if (data?.status === 'processing' || data?.status === 'queued') {
        return 3000;
      }
      // Stop polling if complete or failed
      return false;
    },
    retry: false, // Don't retry on error
  });
}
```

**Auto-Save Hook**

```typescript
// /lib/hooks/useAutoSave.ts
import { useEffect } from 'react';
import { useMutation } from '@tanstack/react-query';
import { useDebounce } from './useDebounce';

export function useAutoSave<T>(
  data: T,
  saveFn: (data: T) => Promise<void>,
  delay = 1000
) {
  const debouncedData = useDebounce(data, delay);
  
  const { mutate: save } = useMutation({
    mutationFn: saveFn,
    onError: (error) => {
      console.error('Auto-save failed:', error);
      // Show toast notification (optional)
    },
  });
  
  useEffect(() => {
    if (debouncedData) {
      save(debouncedData);
    }
  }, [debouncedData, save]);
}
```

### 3. Local State: useState

Used for simple UI state:
- Modal open/close
- Accordion expand/collapse
- Input focus state
- Temporary UI flags

```typescript
// Example: Modal state
const [isPriorityModalOpen, setIsPriorityModalOpen] = useState(false);

// Example: Accordion state
const [isQuestionnaireOpen, setIsQuestionnaireOpen] = useState(false);
```

### 4. Form State: react-hook-form

Used for complex forms with validation:

```typescript
// /components/assessment/GoalForm.tsx
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const goalSchema = z.object({
  description: z.string()
    .min(10, 'Please provide more detail (at least 10 characters)')
    .max(200, 'Description too long (max 200 characters)'),
  priority: z.enum(['high', 'medium', 'low']),
  categories: z.array(z.string())
    .min(1, 'Select at least 1 category')
    .max(3, 'Select at most 3 categories'),
  deadline: z.date()
    .min(new Date(Date.now() + 14 * 24 * 60 * 60 * 1000), 'Deadline should be at least 2 weeks away'),
  hardness: z.number().min(1).max(5),
});

type GoalFormData = z.infer<typeof goalSchema>;

export function GoalForm({ goalNumber, onSubmit }: GoalFormProps) {
  const {
    register,
    control,
    handleSubmit,
    watch,
    formState: { errors },
  } = useForm<GoalFormData>({
    resolver: zodResolver(goalSchema),
    mode: 'onBlur', // Validate on blur
  });
  
  const deadline = watch('deadline');
  const categories = watch('categories');
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Input
        {...register('description')}
        error={errors.description?.message}
        placeholder="Example: Run a 5K in under 30 minutes"
      />
      
      <Controller
        name="categories"
        control={control}
        render={({ field }) => (
          <CheckboxGroup
            {...field}
            options={LIFE_CATEGORIES}
            max={3}
            error={errors.categories?.message}
          />
        )}
      />
      
      {/* Other fields... */}
    </form>
  );
}
```

---

## Routing & Navigation

### Next.js App Router Structure

```
/app
├── layout.tsx              # Root layout (AuthProvider, QueryProvider)
├── page.tsx                # Landing page (/)
├── /(auth)
│   └── callback/page.tsx   # OAuth callback (/callback)
├── /(dashboard)
│   ├── layout.tsx          # Dashboard layout (requires auth)
│   ├── assessment
│   │   ├── step-1/page.tsx
│   │   ├── step-2/page.tsx
│   │   └── processing/page.tsx
│   └── preview
│       └── [id]/page.tsx   # Dynamic route for analysis preview
└── error.tsx               # Global error boundary
```

### Route Groups

**Auth Routes:** `(auth)` group for login/callback
**Dashboard Routes:** `(dashboard)` group with authentication middleware

```typescript
// /app/(dashboard)/layout.tsx
import { redirect } from 'next/navigation';
import { useAuth } from '@/lib/contexts/AuthContext';

export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const { user, loading } = useAuth();
  
  if (loading) return <LoadingSpinner />;
  if (!user) redirect('/');
  
  return (
    <div>
      <Header />
      {children}
      <Footer />
    </div>
  );
}
```

### Navigation Patterns

**Programmatic Navigation:**

```typescript
import { useRouter } from 'next/navigation';

const router = useRouter();

// Navigate to next step
router.push('/assessment/step-2');

// Navigate with state (use query params)
router.push(`/preview/${analysisId}`);

// Replace (don't add to history)
router.replace('/assessment/step-1');
```

**Link Component:**

```typescript
import Link from 'next/link';

<Link 
  href="/assessment/step-2"
  className="button-primary"
>
  Next: Set Your Goals
</Link>
```

### Protected Routes Middleware

```typescript
// /middleware.ts
import { createMiddlewareClient } from '@supabase/auth-helpers-nextjs';
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export async function middleware(req: NextRequest) {
  const res = NextResponse.next();
  const supabase = createMiddlewareClient({ req, res });
  
  const { data: { session } } = await supabase.auth.getSession();
  
  // Protect /assessment and /preview routes
  if (req.nextUrl.pathname.startsWith('/assessment') || 
      req.nextUrl.pathname.startsWith('/preview')) {
    if (!session) {
      return NextResponse.redirect(new URL('/', req.url));
    }
  }
  
  return res;
}

export const config = {
  matcher: ['/assessment/:path*', '/preview/:path*'],
};
```

---

## Form Handling & Validation

### Progressive Validation Strategy

**Validation Timing:**
1. **Real-time (as user types):** Character counts, category selection limits
2. **On blur:** Field-level validation with helpful hints
3. **On submit:** Final validation before API call

### Validation Rules

```typescript
// /lib/utils/validation.ts
import { z } from 'zod';

export const LIFE_CATEGORIES = [
  'work_career',
  'health_fitness',
  'relationships_family',
  'learning_growth',
  'creativity_hobbies',
  'rest_recovery',
  'community_service',
  'admin_maintenance',
] as const;

// Goal validation schema
export const goalSchema = z.object({
  description: z.string()
    .min(10, 'Please provide more detail (at least 10 characters)')
    .max(200, 'Keep it concise (max 200 characters)'),
  
  priority: z.enum(['high', 'medium', 'low']),
  
  categories: z.array(z.enum(LIFE_CATEGORIES))
    .min(1, 'Select at least 1 category')
    .max(3, 'Select at most 3 categories'),
  
  deadline: z.date()
    .refine(
      (date) => date > new Date(Date.now() + 14 * 24 * 60 * 60 * 1000),
      'That's soon! Make sure it's achievable.'
    )
    .refine(
      (date) => date < new Date(Date.now() + 365 * 24 * 60 * 60 * 1000),
      'Long-term goals are great, but we recommend breaking them into shorter milestones.'
    ),
  
  hardness: z.number().min(1).max(5),
});

// Assessment Step 1 schema
export const assessmentStep1Schema = z.object({
  fileId: z.string().uuid('Please upload a valid CliftonStrengths PDF'),
  
  weeklyHours: z.object({
    work_career: z.number().min(0).max(168),
    health_fitness: z.number().min(0).max(168),
    relationships_family: z.number().min(0).max(168),
    learning_growth: z.number().min(0).max(168),
    creativity_hobbies: z.number().min(0).max(168),
    rest_recovery: z.number().min(0).max(168),
    community_service: z.number().min(0).max(168),
    admin_maintenance: z.number().min(0).max(168),
  }),
  
  questionnaire: z.object({
    biggest_challenge: z.string().max(500).optional(),
    motivation: z.string().max(500).optional(),
    energy_time: z.string().max(500).optional(),
    what_worked: z.string().max(500).optional(),
    what_didnt_work: z.string().max(500).optional(),
  }).optional(),
});

// Assessment Step 2 schema (multiple goals)
export const assessmentStep2Schema = z.object({
  goals: z.array(goalSchema).min(1).max(3),
}).refine(
  (data) => {
    const highPriorityCount = data.goals.filter(g => g.priority === 'high').length;
    return highPriorityCount <= 1;
  },
  'Only 1 goal can be high priority'
);
```

### Form Component Pattern

```typescript
// /components/assessment/StepTwo.tsx
import { useForm, useFieldArray } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { assessmentStep2Schema } from '@/lib/utils/validation';
import { useAutoSave } from '@/lib/hooks/useAutoSave';

export function StepTwo() {
  const { control, handleSubmit, watch, formState: { errors } } = useForm({
    resolver: zodResolver(assessmentStep2Schema),
    defaultValues: {
      goals: [
        { description: '', priority: 'medium', categories: [], deadline: null, hardness: 3 }
      ],
    },
    mode: 'onBlur',
  });
  
  const { fields, append, remove } = useFieldArray({
    control,
    name: 'goals',
  });
  
  const formData = watch();
  
  // Auto-save with 1 second debounce
  useAutoSave(
    formData,
    async (data) => {
      await fetchAPI('/assessments/draft', {
        method: 'PUT',
        body: JSON.stringify(data),
      });
    },
    1000
  );
  
  const onSubmit = async (data: z.infer<typeof assessmentStep2Schema>) => {
    // Check rate limit
    const { remaining } = await analysesAPI.getRateLimit();
    
    if (remaining === 0) {
      // Show rate limit modal
      setShowRateLimitModal(true);
      return;
    }
    
    // Submit for analysis
    const { analysisId } = await analysesAPI.create(data);
    router.push(`/assessment/processing?id=${analysisId}`);
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {fields.map((field, index) => (
        <GoalForm
          key={field.id}
          goalNumber={index + 1}
          control={control}
          errors={errors.goals?.[index]}
          onRemove={index > 0 ? () => remove(index) : undefined}
        />
      ))}
      
      {fields.length < 3 && (
        <Button
          type="button"
          variant="secondary"
          onClick={() => append({ description: '', priority: 'medium', categories: [], deadline: null, hardness: 3 })}
        >
          Add Another Goal
        </Button>
      )}
      
      <Button type="submit" disabled={parsingStatus !== 'complete'}>
        Analyse My Goals
      </Button>
    </form>
  );
}
```

### Error Message Components

```typescript
// /components/common/ErrorMessage.tsx
interface ErrorMessageProps {
  error?: string;
  variant?: 'error' | 'warning' | 'info';
}

export function ErrorMessage({ error, variant = 'error' }: ErrorMessageProps) {
  if (!error) return null;
  
  const styles = {
    error: 'text-red-600 bg-red-50 border-red-200',
    warning: 'text-orange-600 bg-orange-50 border-orange-200',
    info: 'text-blue-600 bg-blue-50 border-blue-200',
  };
  
  return (
    <div className={`text-sm p-3 rounded-lg border ${styles[variant]}`}>
      {error}
    </div>
  );
}

// Usage
<ErrorMessage error={errors.description?.message} variant="error" />
<ErrorMessage error="That's soon! Make sure it's achievable." variant="warning" />
```

---

## File Upload Architecture

### Upload Flow

```
User selects PDF
  ↓
Request signed upload URL from API
  ↓
Upload directly to Supabase Storage (client → Supabase)
  ↓
Trigger parsing via API endpoint
  ↓
Poll parsing status every 2s
  ↓
Show result (success/failure)
```

### FileUpload Component

```typescript
// /components/assessment/FileUpload.tsx
import { useCallback, useState } from 'react';
import { useDropzone } from 'react-dropzone';
import { useFileUpload } from '@/lib/hooks/useFileUpload';
import { useParsingStatus } from '@/lib/hooks/useParsingStatus';

export function FileUpload({ onSuccess }: { onSuccess: (fileId: string) => void }) {
  const [fileId, setFileId] = useState<string>();
  
  const { mutate: uploadFile, isLoading: isUploading } = useFileUpload();
  const { data: parsingStatus } = useParsingStatus(fileId);
  
  const onDrop = useCallback((acceptedFiles: File[]) => {
    const file = acceptedFiles[0];
    
    uploadFile(file, {
      onSuccess: (data) => {
        setFileId(data.fileId);
      },
      onError: (error) => {
        // Show error message
        console.error('Upload failed:', error);
      },
    });
  }, [uploadFile]);
  
  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    onDrop,
    accept: {
      'application/pdf': ['.pdf'],
    },
    maxSize: 10 * 1024 * 1024, // 10MB
    multiple: false,
  });
  
  useEffect(() => {
    if (parsingStatus?.status === 'complete') {
      onSuccess(fileId!);
    }
  }, [parsingStatus, fileId, onSuccess]);
  
  return (
    <div className="space-y-4">
      {/* Dropzone */}
      <div
        {...getRootProps()}
        className={`
          border-2 border-dashed rounded-lg p-8 text-center cursor-pointer
          transition-colors
          ${isDragActive ? 'border-orange-500 bg-orange-50' : 'border-gray-300'}
          ${isUploading ? 'opacity-50 cursor-not-allowed' : 'hover:border-orange-400'}
        `}
      >
        <input {...getInputProps()} />
        
        <div className="space-y-2">
          <div className="text-4xl">📄</div>
          <p className="text-lg font-medium">
            {isDragActive ? 'Drop your PDF here' : 'Tap to upload PDF'}
          </p>
          <p className="text-sm text-gray-500">
            Only PDF format accepted (max 10MB)
          </p>
        </div>
      </div>
      
      {/* Parsing Status */}
      {fileId && (
        <ParsingStatus status={parsingStatus?.status} />
      )}
    </div>
  );
}

// Parsing Status Component
function ParsingStatus({ status }: { status?: 'queued' | 'processing' | 'complete' | 'failed' }) {
  if (!status) return null;
  
  const states = {
    queued: {
      icon: '⏳',
      text: 'Waiting in queue...',
      color: 'text-gray-600',
    },
    processing: {
      icon: '🔄',
      text: 'Processing your report in the background...',
      color: 'text-orange-600',
    },
    complete: {
      icon: '✓',
      text: 'Report processed successfully',
      color: 'text-green-600',
    },
    failed: {
      icon: '⚠️',
      text: 'Hmm, we couldn't read that PDF. Is it a CliftonStrengths report from Gallup?',
      color: 'text-orange-600',
    },
  };
  
  const state = states[status];
  
  return (
    <div className={`flex items-center gap-2 text-sm ${state.color}`}>
      <span className={status === 'processing' ? 'animate-spin' : ''}>
        {state.icon}
      </span>
      <span>{state.text}</span>
    </div>
  );
}
```

---

## Error Handling & Boundaries

### Error Boundary Hierarchy

**1. Global Error Boundary** (catches app-level errors)

```typescript
// /app/error.tsx
'use client';

import { useEffect } from 'react';
import * as Sentry from '@sentry/nextjs';

export default function GlobalError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    Sentry.captureException(error);
  }, [error]);
  
  return (
    <html>
      <body>
        <div className="min-h-screen flex items-center justify-center p-4">
          <div className="max-w-md text-center space-y-4">
            <h1 className="text-2xl font-semibold text-gray-900">
              Oops, something went wrong
            </h1>
            <p className="text-gray-600">
              We've been notified and are looking into it. Please try again.
            </p>
            <button
              onClick={reset}
              className="px-6 py-3 bg-orange-500 text-white rounded-lg hover:bg-orange-600"
            >
              Try Again
            </button>
          </div>
        </div>
      </body>
    </html>
  );
}
```

**2. Page-Level Error Boundary** (catches route-specific errors)

```typescript
// /app/(dashboard)/assessment/step-1/error.tsx
'use client';

import { useEffect } from 'react';
import * as Sentry from '@sentry/nextjs';

export default function StepOneError({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  useEffect(() => {
    Sentry.captureException(error);
  }, [error]);
  
  return (
    <div className="max-w-2xl mx-auto p-6 space-y-4">
      <h2 className="text-xl font-semibold">Something went wrong</h2>
      <p className="text-gray-600">
        We had trouble loading this page. Your progress is saved, so you can try again.
      </p>
      <button
        onClick={reset}
        className="px-4 py-2 bg-orange-500 text-white rounded-lg"
      >
        Try Again
      </button>
    </div>
  );
}
```

**3. Component-Level Error Boundary** (catches feature crashes)

```typescript
// /components/common/ErrorBoundary.tsx
import React, { Component, ReactNode } from 'react';
import * as Sentry from '@sentry/nextjs';

interface Props {
  children: ReactNode;
  fallback: ReactNode;
}

interface State {
  hasError: boolean;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false };
  }
  
  static getDerivedStateFromError(_: Error): State {
    return { hasError: true };
  }
  
  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    Sentry.captureException(error, { extra: errorInfo });
  }
  
  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    
    return this.props.children;
  }
}

// Usage
<ErrorBoundary fallback={<FileUploadError />}>
  <FileUpload />
</ErrorBoundary>
```

### Sentry Configuration

```typescript
// /lib/sentry.ts
import * as Sentry from '@sentry/nextjs';

export function initSentry() {
  Sentry.init({
    dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
    environment: process.env.NODE_ENV,
    
    // Capture 100% of errors in production
    tracesSampleRate: 1.0,
    
    // Set context
    beforeSend(event, hint) {
      // Filter out known user errors (e.g., network issues)
      if (hint.originalException instanceof TypeError && 
          hint.originalException.message.includes('Failed to fetch')) {
        return null; // Don't send to Sentry
      }
      
      return event;
    },
    
    // Ignore certain errors
    ignoreErrors: [
      'ResizeObserver loop limit exceeded',
      'Non-Error promise rejection captured',
    ],
  });
}
```

```typescript
// /app/layout.tsx
'use client';

import { useEffect } from 'react';
import { initSentry } from '@/lib/sentry';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    if (process.env.NODE_ENV === 'production') {
      initSentry();
    }
  }, []);
  
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

### API Error Handling

```typescript
// /lib/hooks/useAPIError.ts
import { useEffect } from 'react';
import { APIError } from '@/lib/api/client';
import * as Sentry from '@sentry/nextjs';

export function useAPIError(error: unknown) {
  useEffect(() => {
    if (!error) return;
    
    if (error instanceof APIError) {
      // Log to Sentry with context
      Sentry.captureException(error, {
        extra: {
          statusCode: error.statusCode,
          errorCode: error.code,
          details: error.details,
        },
      });
      
      // Show user-friendly message
      showToast(getUserFriendlyMessage(error));
    }
  }, [error]);
}

function getUserFriendlyMessage(error: APIError): string {
  const messages: Record<string, string> = {
    'RATE_LIMIT_EXCEEDED': 'You've used your 2 analyses for today. Come back tomorrow!',
    'INVALID_PDF': 'We couldn't read that PDF. Is it a CliftonStrengths report from Gallup?',
    'PARSING_FAILED': 'Something went wrong processing your file. Want to try another one?',
    'UNAUTHORIZED': 'Please sign in to continue',
  };
  
  return messages[error.code] || 'Something went wrong. Please try again.';
}
```

---

## Loading & Processing States

### Loading States Strategy

**1. Skeleton Loaders** (for initial page load)

```typescript
// /components/common/SkeletonLoader.tsx
export function SkeletonLoader() {
  return (
    <div className="animate-pulse space-y-4">
      <div className="h-4 bg-gray-200 rounded w-3/4"></div>
      <div className="h-4 bg-gray-200 rounded w-1/2"></div>
      <div className="h-32 bg-gray-200 rounded"></div>
    </div>
  );
}
```

**2. Inline Spinners** (for mutations/actions)

```typescript
// /components/common/LoadingSpinner.tsx
export function LoadingSpinner({ size = 'md' }: { size?: 'sm' | 'md' | 'lg' }) {
  const sizes = {
    sm: 'w-4 h-4',
    md: 'w-8 h-8',
    lg: 'w-12 h-12',
  };
  
  return (
    <div className={`${sizes[size]} animate-spin`}>
      <div className="w-full h-full border-4 border-orange-500 border-t-transparent rounded-full" />
    </div>
  );
}
```

**3. Processing Stages** (~130 second wait)

```typescript
// /components/assessment/ProcessingStages.tsx
import { useEffect, useState } from 'react';
import { useAnalysisStatus } from '@/lib/hooks/useAnalysis';

const STAGES = [
  { 
    name: 'Analyzing your strengths',
    duration: 10,
    message: 'Mapping your CliftonStrengths to behavioral science frameworks...',
  },
  {
    name: 'Creating personalized strategies',
    duration: 60,
    message: 'Generating action plans tailored to your unique personality...',
  },
  {
    name: 'Synthesizing recommendations',
    duration: 30,
    message: 'Combining insights across all your goals...',
  },
  {
    name: 'Generating your PDF',
    duration: 20,
    message: 'Almost there! Creating your downloadable strategy...',
  },
];

export function ProcessingStages({ analysisId }: { analysisId: string }) {
  const { data: status } = useAnalysisStatus(analysisId);
  const [currentStage, setCurrentStage] = useState(0);
  const [elapsedTime, setElapsedTime] = useState(0);
  
  useEffect(() => {
    const timer = setInterval(() => {
      setElapsedTime(prev => prev + 1);
      
      // Update current stage based on elapsed time
      let totalDuration = 0;
      for (let i = 0; i < STAGES.length; i++) {
        totalDuration += STAGES[i].duration;
        if (elapsedTime < totalDuration) {
          setCurrentStage(i);
          break;
        }
      }
    }, 1000);
    
    return () => clearInterval(timer);
  }, [elapsedTime]);
  
  // Handle completion from API
  useEffect(() => {
    if (status?.status === 'complete') {
      router.push(`/preview/${analysisId}`);
    }
  }, [status, analysisId]);
  
  return (
    <div className="max-w-2xl mx-auto p-6 space-y-8">
      {/* Header */}
      <div className="text-center space-y-2">
        <h1 className="text-2xl font-semibold">Creating Your Strategy</h1>
        <p className="text-gray-600">
          This takes about 2 minutes. You can close your browser—we'll save your results.
        </p>
      </div>
      
      {/* Stages */}
      <div className="space-y-4">
        {STAGES.map((stage, index) => (
          <div
            key={index}
            className={`p-4 rounded-lg border-2 transition-all ${
              index === currentStage
                ? 'border-orange-500 bg-orange-50'
                : index < currentStage
                ? 'border-green-500 bg-green-50'
                : 'border-gray-200 bg-white'
            }`}
          >
            <div className="flex items-center gap-3">
              {/* Icon */}
              <div className={`flex-shrink-0 w-8 h-8 rounded-full flex items-center justify-center ${
                index === currentStage
                  ? 'bg-orange-500 text-white'
                  : index < currentStage
                  ? 'bg-green-500 text-white'
                  : 'bg-gray-200 text-gray-400'
              }`}>
                {index < currentStage ? '✓' : index === currentStage ? '⋯' : index + 1}
              </div>
              
              {/* Content */}
              <div className="flex-1">
                <div className="font-medium">{stage.name}</div>
                {index === currentStage && (
                  <div className="text-sm text-gray-600 mt-1">
                    {stage.message}
                  </div>
                )}
              </div>
              
              {/* Spinner */}
              {index === currentStage && (
                <LoadingSpinner size="sm" />
              )}
            </div>
          </div>
        ))}
      </div>
      
      {/* Estimated time */}
      <div className="text-center text-sm text-gray-500">
        Estimated time remaining: {Math.max(0, 130 - elapsedTime)}s
      </div>
    </div>
  );
}
```

---

## Mobile-First Responsive Design

### Breakpoint System

```typescript
// tailwind.config.js
module.exports = {
  theme: {
    screens: {
      'sm': '640px',   // Tablet
      'md': '768px',   // Desktop (rarely used)
      'lg': '1024px',  // Large desktop (rarely used)
    },
  },
};
```

### Mobile-First Design Patterns

**1. Touch Target Sizing**

```typescript
// /components/common/Button.tsx
export function Button({ children, ...props }: ButtonProps) {
  return (
    <button
      className="
        min-h-[44px] px-6 py-3 rounded-lg
        text-base font-medium
        active:scale-95 transition-transform
      "
      {...props}
    >
      {children}
    </button>
  );
}
```

**2. Single Column Layouts**

```typescript
// Always single column, max-width for desktop
<div className="max-w-2xl mx-auto px-4 sm:px-6">
  {/* Content */}
</div>
```

**3. Mobile-Optimized Forms**

```typescript
// /components/common/Input.tsx
export function Input({ type, ...props }: InputProps) {
  return (
    <input
      type={type}
      className="
        w-full min-h-[44px] px-4 py-3
        text-base rounded-lg border-2
        focus:border-orange-500 focus:outline-none
      "
      // Prevent iOS zoom on input focus
      style={{ fontSize: '16px' }}
      {...props}
    />
  );
}
```

**4. Number Input with +/- Buttons**

```typescript
// /components/assessment/HourInput.tsx
export function HourInput({ value, onChange }: HourInputProps) {
  return (
    <div className="flex items-center gap-2">
      <button
        type="button"
        onClick={() => onChange(Math.max(0, value - 1))}
        className="w-11 h-11 rounded-lg border-2 flex items-center justify-center"
      >
        −
      </button>
      
      <input
        type="number"
        value={value}
        onChange={(e) => onChange(Number(e.target.value))}
        className="w-20 h-11 text-center rounded-lg border-2"
        inputMode="numeric"
      />
      
      <button
        type="button"
        onClick={() => onChange(Math.min(168, value + 1))}
        className="w-11 h-11 rounded-lg border-2 flex items-center justify-center"
      >
        +
      </button>
    </div>
  );
}
```

**5. Mobile-Friendly Date Picker**

```typescript
// Use native date picker on mobile
<input
  type="date"
  className="w-full min-h-[44px] px-4 py-3 rounded-lg border-2"
  min={new Date().toISOString().split('T')[0]}
/>
```

### Responsive Utilities

```typescript
// /lib/utils/responsive.ts
export function useIsMobile() {
  const [isMobile, setIsMobile] = useState(false);
  
  useEffect(() => {
    const checkMobile = () => {
      setIsMobile(window.innerWidth < 640);
    };
    
    checkMobile();
    window.addEventListener('resize', checkMobile);
    return () => window.removeEventListener('resize', checkMobile);
  }, []);
  
  return isMobile;
}
```

---

## Performance Optimization

### Code Splitting Strategy

**1. Route-Based Splitting** (automatic with Next.js)

```typescript
// Each page is automatically code-split
// /app/assessment/step-1/page.tsx → separate chunk
// /app/assessment/step-2/page.tsx → separate chunk
```

**2. Component-Level Lazy Loading**

```typescript
// Lazy load heavy components
import dynamic from 'next/dynamic';

const ProcessingStages = dynamic(
  () => import('@/components/assessment/ProcessingStages'),
  { loading: () => <SkeletonLoader /> }
);

const PDFPreview = dynamic(
  () => import('@/components/assessment/PDFPreview'),
  { ssr: false } // Don't render on server
);
```

**3. Prefetching Critical Routes**

```typescript
import Link from 'next/link';

// Prefetch Step 2 when user is on Step 1
<Link href="/assessment/step-2" prefetch={true}>
  Next: Set Your Goals
</Link>
```

### Bundle Size Optimization

**Target:** <250KB gzipped for initial load

**Tools:**
```bash
# Analyze bundle
npm run build
npx @next/bundle-analyzer
```

**Optimizations:**
- Tree-shake unused code
- Use barrel exports sparingly
- Lazy load heavy libraries (e.g., date-fns, PDF viewer)

### Image Optimization

```typescript
import Image from 'next/image';

// Automatic optimization with next/image
<Image
  src="/hero-illustration.png"
  alt="Goal Achievement"
  width={600}
  height={400}
  priority // Load immediately for above-fold images
/>

// Lazy load below-fold images
<Image
  src="/social-proof.png"
  alt="Testimonial"
  width={300}
  height={200}
  loading="lazy"
/>
```

### Caching Strategy

```typescript
// React Query cache config
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000, // 1 minute
      cacheTime: 5 * 60 * 1000, // 5 minutes
      
      // Cache these indefinitely (until invalidation)
      staleTime: {
        userProfile: Infinity,
        cliftonStrengthsData: Infinity,
      },
    },
  },
});
```

### API Call Optimization

**Batch Related Requests:**

```typescript
// Instead of 3 separate calls
const user = await fetchAPI('/users/me');
const rateLimit = await fetchAPI('/users/me/rate-limit');
const draft = await fetchAPI('/assessments/draft');

// Single combined endpoint
const { user, rateLimit, draft } = await fetchAPI('/users/me/dashboard');
```

**Debounce Auto-Save:**

```typescript
// Already implemented with 1s debounce
useAutoSave(formData, saveDraft, 1000);
```

---

## TypeScript Configuration

### tsconfig.json

```json
{
  "compilerOptions": {
    // Strict mode enabled
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    
    // Module resolution
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "allowJs": true,
    "esModuleInterop": true,
    "isolatedModules": true,
    "jsx": "preserve",
    
    // Type checking
    "skipLibCheck": true,
    "allowSyntheticDefaultImports": true,
    
    // Path aliases
    "baseUrl": ".",
    "paths": {
      "@/*": ["./*"]
    },
    
    // Next.js specific
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ]
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### Shared Type Definitions

```typescript
// /types/api.ts
export interface APIResponse<T> {
  data: T;
  error?: never;
}

export interface APIError {
  error: {
    code: string;
    message: string;
    statusCode: number;
    details?: Record<string, any>;
  };
  data?: never;
}

export type APIResult<T> = APIResponse<T> | APIError;

// /types/assessment.ts
export interface AssessmentDraft {
  fileId: string;
  weeklyHours: Record<LifeCategory, number>;
  questionnaire?: {
    biggest_challenge?: string;
    motivation?: string;
    energy_time?: string;
    what_worked?: string;
    what_didnt_work?: string;
  };
  goals: Goal[];
  updatedAt: string;
}

export type LifeCategory =
  | 'work_career'
  | 'health_fitness'
  | 'relationships_family'
  | 'learning_growth'
  | 'creativity_hobbies'
  | 'rest_recovery'
  | 'community_service'
  | 'admin_maintenance';

export type Priority = 'high' | 'medium' | 'low';

export interface Goal {
  description: string;
  priority: Priority;
  categories: LifeCategory[];
  deadline: Date;
  hardness: 1 | 2 | 3 | 4 | 5;
}

// /types/user.ts
export interface User {
  id: string;
  email: string;
  name: string;
  avatarUrl?: string;
  createdAt: string;
}

export interface RateLimitStatus {
  remaining: number;
  resetAt: string;
}
```

### Runtime Validation with Zod

```typescript
// Validate API responses at runtime
import { z } from 'zod';

const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string(),
  avatarUrl: z.string().url().optional(),
  createdAt: z.string().datetime(),
});

// In API client
export async function getUser(): Promise<User> {
  const response = await fetchAPI('/users/me');
  
  // Validate response matches expected type
  const validated = UserSchema.parse(response);
  
  return validated;
}
```

---

## Development Workflow

### Environment Variables

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
NEXT_PUBLIC_API_URL=https://goalos.app/api/v1
NEXT_PUBLIC_SENTRY_DSN=your-sentry-dsn

# Server-only (not exposed to browser)
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
```

### Local Development Setup

```bash
# Install dependencies
npm install

# Run development server
npm run dev

# Type check
npm run type-check

# Lint
npm run lint

# Build for production
npm run build

# Start production server
npm start
```

### Git Workflow

```
feature/assessment-step-1
feature/goal-form-validation
fix/file-upload-parsing
```

---

## Testing Strategy

### Unit Tests (Jest + React Testing Library)

```typescript
// /components/assessment/__tests__/GoalForm.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { GoalForm } from '../GoalForm';

describe('GoalForm', () => {
  it('shows error when description too short', async () => {
    render(<GoalForm goalNumber={1} />);
    
    const input = screen.getByPlaceholderText(/Example: Run a 5K/);
    fireEvent.change(input, { target: { value: 'Run' } });
    fireEvent.blur(input);
    
    expect(await screen.findByText(/at least 10 characters/)).toBeInTheDocument();
  });
  
  it('enforces 1-3 category selection', async () => {
    render(<GoalForm goalNumber={1} />);
    
    // Select 4 categories
    fireEvent.click(screen.getByLabelText('Work & Career'));
    fireEvent.click(screen.getByLabelText('Health & Fitness'));
    fireEvent.click(screen.getByLabelText('Learning & Growth'));
    fireEvent.click(screen.getByLabelText('Creativity & Hobbies'));
    
    expect(await screen.findByText(/at most 3 categories/)).toBeInTheDocument();
  });
});
```

### Integration Tests (Playwright)

```typescript
// /tests/assessment-flow.spec.ts
import { test, expect } from '@playwright/test';

test('complete assessment flow', async ({ page }) => {
  // Land on homepage
  await page.goto('/');
  await page.click('text=Get Your Personalized Strategy');
  
  // Mock Google OAuth (test environment)
  // ... authenticate
  
  // Step 1: Upload PDF
  await page.setInputFiles('input[type=file]', 'fixtures/cliftonstrengths.pdf');
  await expect(page.locator('text=Processing your report')).toBeVisible();
  await expect(page.locator('text=Report processed successfully')).toBeVisible({ timeout: 10000 });
  
  // Fill weekly hours
  await page.fill('input[name="weeklyHours.work_career"]', '40');
  // ... other fields
  
  await page.click('text=Next: Set Your Goals');
  
  // Step 2: Set goals
  await page.fill('textarea[name="goals.0.description"]', 'Run a 5K in under 30 minutes');
  await page.click('input[value="high"]');
  await page.click('text=Health & Fitness');
  // ... deadline, hardness
  
  await page.click('text=Analyse My Goals');
  
  // Processing page
  await expect(page.locator('text=Creating Your Strategy')).toBeVisible();
  
  // ... wait for completion and verify PDF download
});
```

---

## Connections to Other Documents

This Frontend Architecture document connects to:

**1. User Flow & Experience Specification**
- Implements all UX patterns defined in user flow
- Mobile-first design principles
- Warm error messaging tone
- Auto-save and state persistence rules

**2. API Specification**
- API client layer mirrors API endpoint structure
- Type definitions match API response schemas
- Authentication flow uses Supabase session cookies
- Rate limiting integrated into UI

**3. Database Schema & Data Models**
- TypeScript types generated from database schema
- Assessment draft structure matches DB model
- Analysis status polling matches DB state machine

**4. Security & Authentication Specification**
- Supabase Auth SDK integration
- Protected routes middleware
- Session management patterns

**5. AI Task Specifications**
- Processing stages UI reflects 7 AI task workflow
- Time estimates based on task durations
- Error handling for AI failures

**6. PDF Template & Design Specification**
- Preview screen shows PDF metadata
- Download flow integrated into user journey

---

## Implementation Checklist

### Phase 1: Foundation (Week 1)
- [ ] Next.js project setup with TypeScript
- [ ] Tailwind CSS configuration
- [ ] Supabase client initialization
- [ ] React Query setup
- [ ] Sentry integration
- [ ] Environment variables configuration
- [ ] Shared component library (Button, Input, etc.)

### Phase 2: Authentication (Week 1)
- [ ] AuthContext implementation
- [ ] Google OAuth flow
- [ ] Protected route middleware
- [ ] User session management
- [ ] Sign out functionality

### Phase 3: Landing Page (Week 1)
- [ ] Hero section
- [ ] How It Works section
- [ ] Rate limit badge
- [ ] Social proof section
- [ ] Mobile-responsive design

### Phase 4: Assessment Step 1 (Week 2)
- [ ] File upload component with dropzone
- [ ] Supabase Storage integration
- [ ] Parsing status polling
- [ ] Weekly hours input (8 categories)
- [ ] Optional questionnaire (accordion)
- [ ] Form validation
- [ ] Navigation to Step 2

### Phase 5: Assessment Step 2 (Week 2)
- [ ] Goal form component (reusable)
- [ ] Dynamic goal addition (1-3 goals)
- [ ] Priority constraint enforcement (modal)
- [ ] Category multi-select (1-3)
- [ ] Date picker integration
- [ ] Hardness slider
- [ ] Auto-save with debouncing
- [ ] Form validation
- [ ] Rate limit check before submit

### Phase 6: Processing & Preview (Week 3)
- [ ] Processing stages component
- [ ] Analysis status polling
- [ ] Stage-based progress UI
- [ ] Time estimation display
- [ ] Preview screen
- [ ] PDF download button
- [ ] Telegram CTA

### Phase 7: Error Handling (Week 3)
- [ ] Global error boundary
- [ ] Page-level error boundaries
- [ ] Component error boundaries
- [ ] API error handling
- [ ] User-friendly error messages
- [ ] Sentry error reporting

### Phase 8: Testing & Optimization (Week 4)
- [ ] Unit tests for components
- [ ] Integration tests for flows
- [ ] Performance optimization
- [ ] Bundle size analysis
- [ ] Mobile testing (real devices)
- [ ] Cross-browser testing
- [ ] Accessibility audit

---

**Document Status:** COMPLETE - Ready for Claude Code Implementation  
**Next Document:** AI Orchestration Architecture  
**Owner:** Igor (Founder, PM, Designer)

---

*This frontend architecture provides Claude Code with complete patterns, examples, and specifications to build a production-ready, mobile-first React application with excellent UX, type safety, and error handling.*
