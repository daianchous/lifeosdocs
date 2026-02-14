# User Flow & Experience Specification
## Personal Time OS - Goal Achievement MVP

**Version:** 1.0  
**Created:** February 14, 2026  
**Purpose:** Complete user experience specification from landing page to PDF download  
**Tone:** Warm, supportive, like a friend inviting someone to change their life

---

## Executive Summary

This document defines the complete user experience for the Goal Achievement MVP, a web-based platform that transforms CliftonStrengths assessments into personalized goal strategies. The experience is designed to be mobile-first, warm and supportive in tone, and optimized for clarity over complexity.

**Key UX Principles:**
- **Mobile-first design** - Every interaction optimized for small screens
- **Warm and supportive tone** - Never corporate, always encouraging
- **Honest about constraints** - Rate limits communicated upfront, not as surprises
- **Forgiving of mistakes** - State persistence, retry options, helpful error messages
- **Browser-close-safe** - Users can leave and return without losing progress
- **Inline guidance over modals** - Help where you need it, not blocking your flow

**Critical UX Decisions:**
- Background PDF parsing is subtly visible (not completely hidden)
- 2 analyses per day limit communicated on landing page and before Step 1
- State persists from Step 2 onward (earlier steps require restart)
- ~130 second wait shows stage descriptions with encouraging messages
- Preview screen before download encourages Telegram community signup
- All assessment fields optional with inline hints about recommendation quality
- Single clear CTA after success: Join Telegram community

**User Journey Overview:**

```
Landing Page → Google Auth → Assessment (Step 1 + 2) → Processing (~130s) → Preview Screen → Download PDF → Telegram CTA
     ↓              ↓              ↓                        ↓                   ↓              ↓              ↓
  Learn rate    Create        Fill profile +          AI generates        See summary,    Get PDF,     Join community,
   limits      account        set goals              recommendations      metadata        save copy    get support
```

---

## Complete User Journey

### Stage 1: Landing Page (Pre-Authentication)

**User arrives at:** `goalos.app` (or similar domain)

**Mobile-first hero section shows:**
- Clear headline: "Turn Your CliftonStrengths Into Achievable Goals"
- Subheadline: "Get a personalized strategy based on your unique personality—designed for how YOUR brain works"
- Primary CTA button: "Get Your Personalized Strategy" (prominent, mobile-friendly size)
- Visual: Warm color palette (cream, orange, gold), rounded corners, friendly illustration

**Scrolling down reveals:**

1. **How It Works (3 simple steps):**
   - Step 1: "Upload your CliftonStrengths PDF and share your current time investment"
   - Step 2: "Set 1-3 goals with priorities and deadlines"
   - Step 3: "Get a personalized PDF strategy you can print and execute"

2. **Rate Limit Information (clear, upfront):**
   - Small badge/pill: "Free MVP: 2 analyses per day"
   - Explanation: "While we're validating this MVP, you can create 2 personalized strategies daily. Want unlimited access? Join our waitlist for the full app!"
   - Waitlist CTA (secondary button)

3. **What You'll Get:**
   - Personality-matched action strategies
   - Tiered action plans (keystone habits → bonus accelerators)
   - Time estimates for realistic planning
   - Printable progress trackers
   - Access to our Telegram community

4. **Social Proof (if available):**
   - "Join 500+ people turning their strengths into strategies"
   - Sample PDF preview or testimonials

**Footer:**
- Link to Telegram community
- Privacy policy
- About

**Authentication:**
- Click "Get Your Personalized Strategy" → Google OAuth modal
- Single sign-on only (no email/password)
- Post-auth: redirect to Assessment Step 1

---

### Stage 2: Assessment - Step 1 (Profile Setup)

**URL:** `/assessment/step-1`

**Mobile-optimized screen shows:**

**Header:**
- Logo (top left)
- Progress indicator: "Step 1 of 2"
- User avatar (top right, shows Google profile pic)

**Main content:**

**Section A: CliftonStrengths PDF Upload**

Heading: "Upload Your CliftonStrengths Report"

- **File upload component:**
  - Large touch-friendly dropzone
  - "Tap to upload PDF" or drag-and-drop (desktop)
  - File type restriction: PDF only
  - Max size: 10MB
  - Inline hint: "Only PDF format accepted (no photos or screenshots)"

- **After file selected:**
  - Shows filename with remove option
  - Subtly visible parsing indicator appears:
    - Small animated icon (spinning or pulsing)
    - Text: "Processing your report in the background..."
    - Color: warm orange/gold, not alarm-red

- **Inline help (expandable):**
  - "Don't have your CliftonStrengths report? Purchase it from Gallup ($50-200)"
  - Link to Gallup site

**Section B: Weekly Time Investment**

Heading: "How do you currently spend your week?"

Subtext: "Optional but recommended—helps us validate your capacity and create realistic plans."

- **8 life categories with hour input fields:**
  1. Work & Career (__ hours/week)
  2. Health & Fitness (__ hours/week)
  3. Relationships & Family (__ hours/week)
  4. Learning & Growth (__ hours/week)
  5. Creativity & Hobbies (__ hours/week)
  6. Rest & Recovery (__ hours/week)
  7. Community & Service (__ hours/week)
  8. Admin & Maintenance (__ hours/week)

- **Mobile-optimized input:**
  - Number keyboard on tap
  - Touch-friendly increment/decrement buttons (+ / -)
  - Running total displayed: "Total: 87/168 hours"
  - No validation required (can be over or under 168)

- **Inline hint (below total):**
  - "Don't worry about being exact—rough estimates help us understand your current capacity."

**Section C: Optional Questionnaire**

Heading: "Tell us more (optional)"

Expandable accordion with 3-5 short questions:
1. "What's your biggest challenge right now?"
2. "What motivates you most?"
3. "When do you typically have the most energy?"
4. "What's worked well for you in the past?"
5. "What hasn't worked?"

- Text area inputs (mobile-friendly)
- Inline hint: "These answers help us personalize your recommendations even further."

**Bottom of screen:**

- **Primary CTA button:** "Next: Set Your Goals"
  - Always enabled (no required fields)
  - Mobile-friendly size, warm color

- **Secondary text (below button):**
  - "The more you fill out, the more precise your recommendations will be."

**State Management:**

- **If user leaves before clicking "Next":**
  - No data saved
  - Returning later shows empty Step 1
  - Uploaded PDF discarded, parsing stopped

- **If parsing completes before user clicks "Next":**
  - Green checkmark replaces spinner
  - Text: "✓ Report processed successfully"

- **If parsing fails before user clicks "Next":**
  - Warning icon (warm orange, not red)
  - Text: "Hmm, we couldn't read that PDF. Is it a CliftonStrengths report from Gallup?"
  - "Try another file" button

---

### Stage 3: Assessment - Step 2 (Goal Setting)

**URL:** `/assessment/step-2`

**User transitions here after clicking "Next" on Step 1**

**Header:**
- Logo (top left)
- Progress indicator: "Step 2 of 2"
- User avatar (top right)
- Back button: "← Back to Step 1" (if they need to edit)

**Main content:**

Heading: "What goals do you want to achieve?"

Subtext: "Set 1-3 goals. We'll create a personalized strategy for each based on your CliftonStrengths profile."

**Goal Entry Interface (Mobile-First):**

Each goal is a card/section with:

**Goal 1 (always visible, required):**

1. **Goal Description:**
   - Text input: "What do you want to achieve?"
   - Placeholder: "Example: Run a 5K in under 30 minutes"
   - Inline help: "Be specific—clear goals get better strategies"
   - Character limit: 200

2. **Priority:**
   - Radio buttons (large, touch-friendly):
     - ○ High Priority (only 1 allowed across all goals)
     - ○ Medium Priority
     - ○ Low Priority
   - Inline help: "High priority = this goal gets the most detailed strategy"

3. **Life Categories:**
   - Multi-select checkboxes (1-3 required):
     - ☐ Work & Career
     - ☐ Health & Fitness
     - ☐ Relationships & Family
     - ☐ Learning & Growth
     - ☐ Creativity & Hobbies
     - ☐ Rest & Recovery
     - ☐ Community & Service
     - ☐ Admin & Maintenance
   - Validation: Must select 1-3 categories
   - Inline help: "Which areas of life does this goal impact? (Pick 1-3)"

4. **Deadline:**
   - Date picker (mobile-friendly)
   - Validation: 
     - Warns if <2 weeks: "That's soon! Make sure it's achievable."
     - Warns if >1 year: "Long-term goals are great, but we recommend breaking them into shorter milestones."
   - Inline help: "When do you want to achieve this by?"

5. **Difficulty (Hardness Rating):**
   - Visual slider or radio buttons: 1-5
     - 1 = "I've done this before"
     - 2 = "Familiar territory, new challenge"
     - 3 = "Moderate stretch"
     - 4 = "Big stretch, excited and nervous"
     - 5 = "Completely new, feels daunting"
   - Inline help visible below slider with emoji indicators
   - Example: "1 😊 Easy → 5 😰 Very Hard"

**Goal 2 (optional, appears after "Add Another Goal" button):**
- Same structure as Goal 1
- "Remove Goal" button (small, text-only, not prominent)

**Goal 3 (optional, appears after 2nd goal added):**
- Same structure as Goal 1 & 2
- After Goal 3, "Add Another Goal" button disappears

**Priority Constraint Enforcement:**

If user tries to set a second goal to "High Priority":
- Modal appears:
  - Heading: "Only 1 High Priority Goal Allowed"
  - Body: "You can only have one high-priority goal at a time. Which goal should be your top priority?"
  - Options:
    - Radio: ○ Make Goal 1 high priority (downgrade Goal 2)
    - Radio: ○ Make Goal 2 high priority (downgrade Goal 1)
  - Buttons: "Confirm" | "Cancel"
- After selection, modal closes, UI updates

**Bottom of screen:**

**Analyse Button State Logic:**

1. **If background parsing still running:**
   - Button disabled (greyed out)
   - Tooltip on hover/tap: "Still processing your CliftonStrengths PDF... (~30 seconds remaining)"
   - Subtle spinner icon next to button

2. **If parsing failed:**
   - Button disabled
   - Message above button: "We need your CliftonStrengths data to create recommendations. Please go back to Step 1 and upload a valid PDF."
   - Link: "← Back to Step 1"

3. **If parsing complete + at least Goal 1 filled:**
   - Button enabled (warm, prominent color)
   - Text: "Analyse My Goals"
   - Below button: "This will take about 2 minutes. You can close your browser—we'll save your results."

**Rate Limit Check (happens on button click):**

If user has used 2/2 analyses today:
- Modal appears before processing starts:
  - Heading: "You've Used Your 2 Analyses Today"
  - Body: "Your next analysis slot opens in 4 hours 23 minutes. Want unlimited analyses?"
  - Primary CTA: "Join the Waitlist" (links to waitlist form)
  - Secondary CTA: "Join Our Telegram Community" (links to Telegram)
  - Tertiary: "Got it, I'll come back later" (closes modal)

**State Management:**

- **From Step 2 onward, all data persists:**
  - User can close browser, return later
  - Data linked to their Google account
  - Refreshing page shows current progress

- **If user navigates back to Step 1:**
  - Step 2 data preserved
  - Can edit Step 1 and return
  - Re-parsing triggered if new PDF uploaded

---

### Stage 4: Processing & Waiting (~130 seconds)

**URL:** `/analysis/processing` or `/analysis/[analysis_id]`

**User lands here after clicking "Analyse My Goals"**

**Full-screen waiting experience (mobile-optimized):**

**Header:**
- Logo (centered)
- No close button (but browser close is safe)

**Main content (centered):**

**Visual:**
- Animated illustration (warm colors, gentle motion)
- Could be: progress circles, flowing particles, growing tree, etc.
- NOT a traditional progress bar (creates pressure)

**Stage Descriptions (rotate every ~30 seconds):**

Cycle through these messages with smooth transitions:

1. **Stage 1: "Understanding your goals..."**
   - Body: "We're analyzing what you want to achieve and matching it to your unique CliftonStrengths profile."
   - Icon: magnifying glass or lightbulb

2. **Stage 2: "Finding your strengths..."**
   - Body: "Your top themes are guiding us toward strategies that work with your natural wiring, not against it."
   - Icon: puzzle pieces or strengths badge

3. **Stage 3: "Creating your strategy..."**
   - Body: "We're building personalized action plans with realistic time estimates and progress trackers."
   - Icon: map or blueprint

4. **Stage 4: "Almost there..."**
   - Body: "Just putting the finishing touches on your recommendations. This is going to be good."
   - Icon: sparkles or checkmark

**Time indicator (bottom of main content):**
- "This usually takes about 2 minutes"
- No countdown timer (reduces anxiety)
- No percentage (can't be accurate with parallel processing)

**Browser Close Safety Message (below time indicator):**
- Small text, warm color
- "Feel free to close this tab—we'll save your results and you can come back anytime."

**Bottom of screen:**
- Link: "What happens next?" 
  - Expands to show: "You'll get a preview of your personalized strategy, then you can download a PDF to print or save."

**Technical Details (not visible to user):**

- Processing continues even if browser closed
- Analysis record linked to user account
- When user returns, check status:
  - If complete: redirect to preview
  - If still processing: show this waiting screen
  - If failed: redirect to error page

**Failure Mid-Processing:**

If any goal analysis fails (e.g., timeout, AI error):

- Redirect to: `/analysis/[analysis_id]/partial-failure`

**Partial Failure Screen:**

Heading: "We Hit a Snag"

Body: "We successfully analyzed [X] of your [Y] goals, but ran into an issue with [Goal Name]. Here's what happened:"

- Shows which step failed (for debugging):
  - Example: "Goal 3 analysis timed out during strategy generation"

**Options:**

1. **Primary CTA:** "Retry Failed Goal"
   - Attempts to re-run just the failed portion
   - Does NOT count against daily limit (since original attempt failed)

2. **Secondary CTA:** "See Partial Results"
   - Shows preview with completed goals only
   - Note: "You can retry the failed goal later from your history"

3. **Tertiary link:** "Contact Support"
   - Opens Telegram community or email

**State saved:**
- Partial analysis stored with status: "partial_failure"
- User can retry later from history

---

### Stage 5: Preview Screen (Pre-Download)

**URL:** `/analysis/[analysis_id]/preview`

**User lands here after successful processing**

**Header:**
- Logo (top left)
- "Download PDF" button (top right, always visible on mobile)

**Main content:**

Heading: "Your Personalized Strategy Is Ready!"

Subtext: "Here's what we created for you based on your CliftonStrengths profile and goals."

**Summary Cards (mobile-stacked):**

**Card 1: Goals Analyzed**
- Shows each goal with icon:
  - Goal 1: [Goal description] - High Priority ⭐
  - Goal 2: [Goal description] - Medium Priority
  - Goal 3: [Goal description] - Low Priority

**Card 2: Your Dominant Domain**
- "Your CliftonStrengths profile shows you're strongest in: [Executing/Influencing/Relationship Building/Strategic Thinking]"
- "We've matched your strategies to leverage this strength."

**Card 3: What's Inside Your PDF**
- ✓ Personalized strategy for each goal
- ✓ Tiered action plans (keystone → accelerators)
- ✓ Time estimates for realistic planning
- ✓ Octalysis Core Drive labels
- ✓ Printable progress trackers (100-dot grids)
- ✓ [X] pages of actionable recommendations

**Card 4: Next Steps**
- "Print your PDF and start with your keystone habits"
- "Join our Telegram community for support and accountability"
- "Track progress with the dot grids"
- "Come back to create strategies for new goals (you have [X] analyses left today)"

**Primary CTA (large, prominent):**
- Button: "Download Your PDF"
- Icon: download arrow
- Warm color, mobile-friendly size

**Secondary CTA (below download button):**
- Button: "Join Our Telegram Community" (with Telegram icon)
- Text: "Get support, share progress, and connect with others using this system"
- Link opens in new tab

**Tertiary options (smaller, below):**
- "View Analysis History" (link to user's past analyses)
- "Create Another Strategy" (if they have analyses left today)
- If they've used 2/2: "You've used your daily analyses. Next slot opens in [X] hours."

**Social Sharing (optional, subtle):**
- Small text: "Share the app:" [Twitter icon] [Facebook icon] [Link copy]

**State Management:**
- User can return to this preview anytime via history
- Download can be triggered multiple times
- Analysis marked as "viewed" (for analytics)

---

### Stage 6: PDF Download & Post-Download

**Download Behavior:**

When user clicks "Download Your PDF":

1. **Browser initiates download**
   - Filename: `PersonalTimeOS_Strategy_[Date]_[UserFirstName].pdf`
   - Example: `PersonalTimeOS_Strategy_Feb14_2026_Igor.pdf`

2. **Download confirmation appears** (browser-native, non-intrusive)

3. **Preview screen updates:**
   - Download button changes to: "Download Again" (in case they need it)
   - Success message appears: "✓ PDF downloaded! Check your Downloads folder."

4. **Telegram CTA intensifies:**
   - Animation or highlight on Telegram button
   - Text updates: "Great! Now join our community to stay accountable →"

**Post-Download Screen State:**

Everything remains the same except:
- Download button → "Download Again"
- Telegram CTA more prominent
- Optional: "What's next?" expandable section
  - "Print your PDF and put it somewhere visible"
  - "Start with just the keystone habits (don't try to do everything)"
  - "Join Telegram to connect with others on similar journeys"
  - "Set a reminder to review progress in 2 weeks"

**Telegram Community CTA (always visible):**

Clicking "Join Our Telegram Community":
- Opens Telegram link in new tab
- Link format: `https://t.me/personaltimeos` (or whatever)
- User joins group, sees welcome message
- Analytics: track Telegram conversion

**Analysis History:**

User can access past analyses from:
- Top nav: "My Analyses" or user dropdown
- URL: `/history`

**History Page shows:**
- List of past analyses (newest first)
- Each card:
  - Date created
  - Goals analyzed (titles)
  - Status: "Completed" | "Processing" | "Failed"
  - Actions: "View Preview" | "Download PDF" | "Retry" (if failed)

---

## State Diagrams

### Overall User Journey State Machine

```
                    START
                      │
                      ▼
              ┌───────────────┐
              │ Landing Page  │
              └───────┬───────┘
                      │
            [Click CTA: Get Strategy]
                      │
                      ▼
              ┌───────────────┐
              │  Google Auth  │
              └───────┬───────┘
                      │
              [Auth Successful]
                      │
                      ▼
         ┌────────────────────────┐
         │ Assessment - Step 1    │
         │ - Upload PDF           │
         │ - Weekly Hours         │
         │ - Questionnaire        │
         └────────┬───────────────┘
                  │
         [Click "Next: Set Goals"]
                  │
                  ▼
    ┌─────────────────────────────────┐
    │ Background PDF Parsing Starts   │◄──── Subtle indicator visible
    └─────────────────────────────────┘
                  │
                  ▼
         ┌────────────────────────┐
         │ Assessment - Step 2    │
         │ - Set 1-3 Goals        │
         │ - Priority/Categories  │
         │ - Deadline/Difficulty  │
         └────────┬───────────────┘
                  │
      [Click "Analyse My Goals"]
                  │
                  ├────────── [Rate limit hit] ──────┐
                  │                                   │
                  ▼                                   ▼
         ┌────────────────┐              ┌────────────────────┐
         │ Rate Limit OK  │              │  Rate Limit Modal  │
         └────────┬───────┘              │  - Wait X hours    │
                  │                      │  - Join waitlist   │
                  ▼                      │  - Join Telegram   │
    ┌──────────────────────────┐        └────────────────────┘
    │ Processing Screen        │                   │
    │ - 4 stage messages       │                   │
    │ - ~130 seconds           │                   └──► [User exits]
    │ - Browser-close-safe     │
    └──────────┬───────────────┘
               │
               ├─── [Success] ──────────┐
               │                        │
               └─── [Partial Fail] ───┐ │
                                      │ │
                                      ▼ ▼
                            ┌──────────────────┐
                            │ Partial Failure  │
                            │ - Show what failed│
                            │ - Retry option   │
                            │ - See partial    │
                            └────┬─────────────┘
                                 │
                    [Retry] ◄────┴───► [See Partial]
                       │                    │
                       └──────┬─────────────┘
                              │
                              ▼
                   ┌──────────────────────┐
                   │   Preview Screen     │
                   │ - Summary cards      │
                   │ - Download CTA       │
                   │ - Telegram CTA       │
                   └──────┬───────────────┘
                          │
              [Click "Download PDF"]
                          │
                          ▼
                   ┌──────────────────┐
                   │  PDF Downloads   │
                   │  - Success msg   │
                   │  - Telegram CTA  │
                   └──────┬───────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
    [Join Telegram]  [History]    [New Analysis]
          │               │               │
          └───────────────┴───────────────┘
                          │
                          ▼
                        END
```

### PDF Parsing State Machine

```
          [User uploads PDF]
                 │
                 ▼
         ┌───────────────┐
         │  Validating   │──── Check file type, size
         └───────┬───────┘
                 │
    ┌────────────┼────────────┐
    │                         │
    ▼                         ▼
[Invalid]              [Valid PDF]
    │                         │
    ▼                         ▼
┌─────────────┐      ┌────────────────┐
│ Show Error  │      │ Start Parsing  │──── Subtle spinner visible
│ Try Again   │      │ (Background)   │
└─────────────┘      └───────┬────────┘
                             │
                ┌────────────┼────────────┐
                │                         │
                ▼                         ▼
         [Parse Success]          [Parse Failure]
                │                         │
                ▼                         ▼
        ┌───────────────┐         ┌──────────────┐
        │ Extract Data  │         │ Show Warning │
        │ - Themes      │         │ "Not a valid │
        │ - Rankings    │         │ CS report"   │
        │ - Domain      │         └──────────────┘
        └───────┬───────┘
                │
                ▼
        ┌───────────────┐
        │ Parsing Done  │──── Green checkmark
        │ (Stored)      │
        └───────────────┘
```

### Analysis Processing State Machine

```
    [User clicks "Analyse"]
            │
            ▼
    ┌───────────────┐
    │ Check Limits  │
    └───────┬───────┘
            │
     ┌──────┴──────┐
     │             │
     ▼             ▼
[2/2 used]    [OK to proceed]
     │             │
     ▼             ▼
┌─────────┐   ┌──────────────────┐
│ Show    │   │ Start Processing │
│ Limit   │   │ - Create job     │
│ Modal   │   │ - Show wait UI   │
└─────────┘   └────────┬─────────┘
                       │
           ┌───────────┼───────────┐
           │           │           │
           ▼           ▼           ▼
      [Goal 1]    [Goal 2]    [Goal 3]
      Agent       Agent       Agent
           │           │           │
           └───────────┼───────────┘
                       │
                       ▼
               ┌───────────────┐
               │  Synthesizer  │
               │     Agent     │
               └───────┬───────┘
                       │
           ┌───────────┼───────────┐
           │                       │
           ▼                       ▼
      [All Success]          [Any Failure]
           │                       │
           ▼                       ▼
    ┌──────────────┐      ┌────────────────┐
    │ Generate PDF │      │ Partial Result │
    │ Store Result │      │ - Save partial │
    └──────┬───────┘      │ - Mark failed  │
           │              └────────┬───────┘
           │                       │
           └───────────┬───────────┘
                       │
                       ▼
               ┌───────────────┐
               │ Show Preview  │
               │    Screen     │
               └───────────────┘
```

---

## Screen Descriptions

### Screen 1: Landing Page (Mobile View)

**Layout:**
- Full-width hero section with gradient background (cream to light orange)
- Logo top-left (warm colors, friendly typeface)
- Hamburger menu top-right (About, Community, Sign In)

**Hero Content:**
- H1: "Turn Your CliftonStrengths Into Achievable Goals"
- H2: "Get a personalized strategy based on your unique personality—designed for how YOUR brain works"
- CTA Button (large, rounded): "Get Your Personalized Strategy →"
- Small badge below: "✨ Free MVP • 2 analyses per day"

**Section 2: How It Works**
- 3 numbered cards (stacked vertically on mobile):
  1. Icon: Upload | "Upload your CliftonStrengths PDF and share your current time investment"
  2. Icon: Target | "Set 1-3 goals with priorities and deadlines"
  3. Icon: Document | "Get a personalized PDF strategy you can print and execute"

**Section 3: Rate Limit Info**
- Heading: "About the Free MVP"
- Body: "While we're validating this concept, you can create 2 personalized strategies per day. Want unlimited access? Join our waitlist for the full tracking app coming soon!"
- CTA: "Join the Waitlist" (secondary button style)

**Section 4: What You'll Get**
- Checkmark list:
  - ✓ Personality-matched strategies
  - ✓ Tiered action plans
  - ✓ Realistic time estimates
  - ✓ Printable progress trackers
  - ✓ Telegram community access

**Section 5: Social Proof**
- Statistic: "Join 500+ people transforming their strengths into strategies"
- Sample PDF preview (thumbnail or screenshot)

**Footer:**
- Links: Community | Privacy | About
- Telegram icon link

---

### Screen 2: Assessment Step 1 (Mobile View)

**Header:**
- Logo (left)
- Progress: "Step 1 of 2" (center)
- Avatar (right, Google profile pic)

**Content (scrollable):**

**Upload Section:**
- Large upload box with dashed border
- Icon: PDF document
- Text: "Tap to upload your CliftonStrengths PDF"
- Hint: "Only PDF format (max 10MB)"

**After upload:**
- Filename: "CliftonStrengths_Report.pdf" with X to remove
- Parsing indicator: 🔄 "Processing your report in the background..."
- OR success: ✓ "Report processed successfully"
- OR error: ⚠️ "Hmm, we couldn't read that PDF. Is it from Gallup?"

**Weekly Hours Section:**
- Heading: "How do you currently spend your week?"
- Hint: "Optional but recommended—helps us validate your capacity"
- 8 input fields (stacked, each with +/- buttons):
  - Work & Career: [__] hours
  - Health & Fitness: [__] hours
  - [etc for all 8]
- Running total: "Total: 87/168 hours"
- Bottom hint: "Rough estimates are fine!"

**Questionnaire Section:**
- Expandable accordion: "Tell us more (optional) ▼"
- When expanded, shows 5 text areas
- Hint: "These answers help personalize your recommendations"

**Bottom:**
- CTA: "Next: Set Your Goals" (full-width button)
- Helper text: "The more you fill out, the more precise your recommendations will be"

---

### Screen 3: Assessment Step 2 (Mobile View)

**Header:**
- Back arrow (left): "← Step 1"
- Progress: "Step 2 of 2" (center)
- Avatar (right)

**Content:**
- Heading: "What goals do you want to achieve?"
- Hint: "Set 1-3 goals. We'll create a personalized strategy for each."

**Goal 1 Card:**
- Text input: "What do you want to achieve?"
- Placeholder: "Example: Run a 5K in under 30 minutes"

- Priority (radio buttons, large touch targets):
  - ○ High Priority
  - ○ Medium Priority
  - ○ Low Priority
- Hint: "High priority = most detailed strategy"

- Categories (checkboxes):
  - ☐ Work & Career
  - ☐ Health & Fitness
  - [etc, pick 1-3]

- Deadline (date picker):
  - Mobile-friendly calendar popup
  - Shows warnings if <2 weeks or >1 year

- Difficulty slider:
  - 1 😊 → 5 😰
  - Labels show on selection

**Add Goal Button:**
- "+ Add Another Goal" (text button, subtle)

**Bottom:**
- If parsing incomplete:
  - Disabled button: "Analyse My Goals" (greyed)
  - Loading text: "Still processing your PDF... (~30s remaining)"

- If parsing complete:
  - Enabled button: "Analyse My Goals" (warm, prominent)
  - Hint: "This takes ~2 minutes. You can close your browser—we'll save results."

---

### Screen 4: Processing Wait Screen (Mobile View)

**Layout:**
- Centered content, minimal UI
- No header, just logo at top center
- No close button (browser close is safe)

**Main Content:**
- Animated illustration (gentle motion, warm colors)
- Could be: growing tree, flowing particles, puzzle pieces assembling

**Stage Message (rotates every ~30s):**
- Icon (changes per stage)
- Heading: "Understanding your goals..."
- Body text: "We're analyzing what you want to achieve and matching it to your unique CliftonStrengths profile."

**Time Indicator:**
- Small text below: "This usually takes about 2 minutes"

**Browser-Close Message:**
- Even smaller text: "Feel free to close this tab—we'll save your results and you can come back anytime."

**Expandable Info:**
- Link: "What happens next? ▼"
- Expands to: "You'll get a preview of your strategy, then download a PDF to print or save."

**No progress bar, no percentage, no countdown.**
Just encouraging messages and gentle animation.

---

### Screen 5: Preview Screen (Mobile View)

**Header:**
- Logo (left)
- "Download PDF" button (right, always visible)

**Content (scrollable):**

**Success Message:**
- Heading: "Your Personalized Strategy Is Ready! 🎉"
- Subtext: "Here's what we created based on your CliftonStrengths profile and goals."

**Summary Cards (stacked):**

**Card 1: Goals Analyzed**
- Goal 1: "Run a 5K in under 30 minutes" - High Priority ⭐
- Goal 2: "Read 12 books this year" - Medium Priority
- Goal 3: "Learn basic Spanish" - Low Priority

**Card 2: Your Dominant Domain**
- "Your CliftonStrengths profile shows you're strongest in: Executing"
- "We've matched your strategies to leverage this strength."

**Card 3: What's Inside**
- ✓ Personalized strategy for each goal
- ✓ Tiered action plans (keystone → accelerators)
- ✓ Time estimates for realistic planning
- ✓ Octalysis Core Drive labels
- ✓ Printable 100-dot progress trackers
- ✓ 12 pages of actionable recommendations

**Card 4: Next Steps**
- Print your PDF and start with keystone habits
- Join our Telegram community for support
- Track progress with dot grids
- Come back for new strategies (1 analysis left today)

**CTAs:**
- Primary (large): "Download Your PDF" (download icon)
- Secondary: "Join Our Telegram Community" (Telegram icon)
- Tertiary links:
  - "View Analysis History"
  - "Create Another Strategy"

---

### Screen 6: Post-Download State

Same as Screen 5, except:

- Download button → "Download Again"
- Success message: "✓ PDF downloaded! Check your Downloads folder."
- Telegram CTA highlighted/animated: "Great! Now join our community to stay accountable →"

---

### Screen 7: Analysis History (Mobile View)

**Header:**
- Back arrow: "← Home"
- Title: "My Analyses"

**Content:**
- List of analysis cards (newest first):

**Card Example:**
- Date: "February 14, 2026 at 2:30 PM"
- Status badge: "Completed" (green) | "Processing" (orange) | "Failed" (red)
- Goals preview:
  - 1. Run a 5K
  - 2. Read 12 books
  - 3. Learn Spanish
- Actions:
  - "View Preview" (if completed)
  - "Download PDF" (if completed)
  - "Retry" (if failed)
  - "Continue" (if processing)

**Empty state** (no analyses yet):
- Illustration
- Text: "You haven't created any strategies yet"
- CTA: "Create Your First Strategy"

---

## Loading States

### 1. PDF Upload Processing
**State:** Uploading file to server

**Visual:**
- Upload progress bar (0-100%)
- Text: "Uploading... 45%"

**Duration:** 1-5 seconds (depending on file size)

---

### 2. Background PDF Parsing
**State:** AI extracting CliftonStrengths data

**Visual:**
- Small spinning icon (warm orange)
- Text: "Processing your report in the background..."

**Duration:** 15-45 seconds

**Success:**
- Icon: ✓ (green)
- Text: "Report processed successfully"

**Failure:**
- Icon: ⚠️ (warm orange, not red)
- Text: "Hmm, we couldn't read that PDF. Is it a CliftonStrengths report from Gallup?"
- Action: "Try another file" button

---

### 3. Analyse Button Disabled (Parsing Incomplete)
**State:** Waiting for background parsing

**Visual:**
- Button greyed out (not clickable)
- Spinner icon next to button
- Tooltip on hover/tap: "Still processing your CliftonStrengths PDF... (~30 seconds remaining)"

---

### 4. Main Processing Screen (~130 seconds)
**State:** AI generating personalized recommendations

**Visual:**
- Animated illustration (continuous gentle motion)
- Stage messages (rotate every ~30s):
  1. "Understanding your goals..."
  2. "Finding your strengths..."
  3. "Creating your strategy..."
  4. "Almost there..."

**No progress bar or countdown timer**

**Duration:** ~130 seconds average

---

### 5. PDF Generation
**State:** Converting recommendations to PDF (final step)

**Visual:** 
- Last stage message: "Almost there..."
- No separate loading state (happens within Stage 4)

**Duration:** ~10-20 seconds

---

### 6. Retry Processing
**State:** Re-attempting failed goal analysis

**Visual:**
- Same as main processing screen
- Message: "Retrying [Goal Name]..."

**Duration:** ~30-60 seconds (single goal)

---

## Error Scenarios

### Error 1: Invalid PDF Format
**Trigger:** User uploads non-PDF or corrupted file

**Message:**
- Icon: ⚠️ (warm orange)
- Heading: "We couldn't read that file"
- Body: "Please make sure you're uploading a PDF file from Gallup CliftonStrengths. Photos or screenshots won't work."
- Action: "Try another file" button

**Recovery:** User can upload different file, no limit on attempts

---

### Error 2: PDF Too Large
**Trigger:** File size > 10MB

**Message:**
- Icon: ⚠️
- Heading: "File too large"
- Body: "Please upload a PDF smaller than 10MB. CliftonStrengths reports are usually 2-5MB."
- Action: "Try another file" button

**Recovery:** User must compress or use different file

---

### Error 3: Not a CliftonStrengths PDF
**Trigger:** Parsing succeeds but doesn't find expected CliftonStrengths data

**Message:**
- Icon: ⚠️
- Heading: "This doesn't look like a CliftonStrengths report"
- Body: "We couldn't find the expected themes and rankings. Make sure you're uploading the full PDF report from Gallup, not a summary or screenshot."
- Action: "Try another file" button

**Recovery:** User uploads correct file

---

### Error 4: Rate Limit Exceeded
**Trigger:** User tries 3rd analysis in same day

**Modal:**
- Heading: "You've Used Your 2 Analyses Today"
- Body: "Your next analysis slot opens in 4 hours 23 minutes."
- Options:
  - "Want unlimited analyses? Join the waitlist for the full app!" (primary CTA)
  - "Join Our Telegram Community" (secondary CTA)
  - "Got it, I'll come back later" (closes modal)

**Recovery:** User must wait or join waitlist

---

### Error 5: Parsing Timeout
**Trigger:** Background parsing takes >2 minutes

**Message:**
- Icon: ⚠️
- Heading: "Processing is taking longer than expected"
- Body: "This usually means the PDF is complex or our servers are busy. Want to try again?"
- Actions:
  - "Retry" (primary)
  - "Upload different file" (secondary)

**Recovery:** User retries or uploads different file

---

### Error 6: Partial Analysis Failure
**Trigger:** 1+ goals fail during processing, others succeed

**Screen:** `/analysis/[id]/partial-failure`

**Content:**
- Heading: "We Hit a Snag"
- Body: "We successfully analyzed 2 of your 3 goals, but ran into an issue with [Goal 3: Learn Spanish]."
- Technical detail (for debugging): "Goal 3 analysis timed out during strategy generation"

**Options:**
- Primary: "Retry Failed Goal" (doesn't count against limit)
- Secondary: "See Partial Results" (view completed goals)
- Link: "Contact Support" (Telegram/email)

**Recovery:** Retry or accept partial results

---

### Error 7: Complete Analysis Failure
**Trigger:** All goals fail during processing

**Screen:** `/analysis/[id]/failure`

**Content:**
- Heading: "Something Went Wrong"
- Body: "We ran into an issue generating your strategy. This is rare and we're sorry for the trouble!"
- Technical detail: "Analysis failed: [error code]"

**Options:**
- Primary: "Try Again" (doesn't count against limit)
- Secondary: "Contact Support"

**Recovery:** User retries or contacts support

---

### Error 8: Network Error During Processing
**Trigger:** User's internet drops mid-processing

**Behavior:**
- Processing continues on server (user doesn't lose progress)
- When user reconnects and returns:
  - If complete: show preview screen
  - If still processing: show wait screen
  - If failed: show error screen

**No special error message** - recovery is automatic

---

### Error 9: Google Auth Failure
**Trigger:** OAuth fails or user cancels

**Message:**
- Heading: "Sign-in was canceled"
- Body: "You need to sign in with Google to use this app. We only use your email and name—no other data."
- Action: "Try signing in again" button

**Recovery:** User reattempts auth

---

### Error 10: Browser Doesn't Support PDF Download
**Trigger:** Old browser or restrictive settings

**Fallback:**
- If download fails, show inline PDF viewer
- Message: "Your PDF is ready! If the download didn't start, you can view it here:"
- Embedded PDF viewer with download button

**Recovery:** User views inline or copies link

---

## Edge Cases

### Edge Case 1: User Closes Browser During Step 1
**Behavior:**
- No data saved
- PDF parsing stopped (if started)
- Return visit shows fresh Step 1

**Reasoning:** Early exit before commitment point

---

### Edge Case 2: User Closes Browser During Step 2 or Later
**Behavior:**
- All data persists (linked to account)
- Background parsing continues if active
- Return visit shows current progress state

**Reasoning:** User has committed to analysis, should be able to resume

---

### Edge Case 3: User Closes Browser During ~130s Wait
**Behavior:**
- Processing continues on server
- Analysis completes normally
- User can return anytime to see results

**Return scenarios:**
- If complete: redirect to preview
- If still processing: show wait screen
- If failed: show error screen

---

### Edge Case 4: User Tries to Set 0 Goals
**Behavior:**
- "Analyse" button stays disabled
- Hint appears: "Please add at least one goal to continue"

**Reasoning:** Need minimum 1 goal to create strategy

---

### Edge Case 5: User Sets Only 1 Category for Goal
**Behavior:**
- Allowed (1-3 categories means 1 is valid)
- Strategy focuses on that single category

---

### Edge Case 6: User Sets 4+ Categories for Goal
**Behavior:**
- After 3rd checkbox selected, others become disabled
- Hint: "You can select up to 3 categories. Uncheck one to change."

**Reasoning:** Force focus, prevent overly generic strategies

---

### Edge Case 7: User Sets All Goals to Low Priority
**Behavior:**
- Allowed
- No high-priority goal means no single focus
- All strategies get equal treatment

---

### Edge Case 8: User Edits Step 1 After Completing Step 2
**Behavior:**
- Can navigate back via back button
- If new PDF uploaded:
  - Re-triggers parsing
  - Step 2 data preserved
  - "Analyse" button disabled until re-parsing complete
- If weekly hours changed:
  - Saved immediately
  - No re-parsing needed

---

### Edge Case 9: Analysis Completes While User is Offline
**Behavior:**
- Results saved to account
- When user reconnects and returns:
  - Redirect to preview screen
  - No notification sent (MVP has no email notifications)

**Future:** Could add email/Telegram notification when complete

---

### Edge Case 10: User Tries Multiple Browsers/Devices
**Behavior:**
- Data synced via account (Supabase)
- Can start on mobile, finish on desktop
- Latest state always shown

---

### Edge Case 11: User Refreshes Page During Processing
**Behavior:**
- Check processing status on server
- If complete: redirect to preview
- If processing: show wait screen (picks up where it was)
- If failed: show error screen

---

### Edge Case 12: User Sets Deadline in Past
**Behavior:**
- Date picker prevents past dates
- If somehow submitted: validation error
- Message: "Deadline must be in the future"

---

### Edge Case 13: User Leaves All Weekly Hours at 0
**Behavior:**
- Allowed (optional field)
- Strategy generated without capacity validation
- Recommendation quality may be lower (but still generated)

---

### Edge Case 14: User Enters 500 Hours/Week
**Behavior:**
- No hard validation (allow any number)
- Reasoning: User might be tracking multiple people, business hours, etc.
- Strategy adapts to stated capacity

**Future consideration:** Could add warning if >168 hours

---

### Edge Case 15: User Has 1 Analysis Left, Starts 2nd, Hits Midnight
**Behavior:**
- Analysis started before midnight = completes normally
- Counter resets at midnight (new day)
- User gets 2 new analyses next day

---

## Success States

### Success 1: PDF Upload Complete
**Visual:**
- ✓ Green checkmark icon
- Text: "Report processed successfully"
- Filename shown with remove option

**User can proceed to Step 2**

---

### Success 2: Background Parsing Complete
**Visual:**
- ✓ Green checkmark replaces spinner
- Text: "✓ Report processed successfully"
- "Analyse" button becomes enabled

---

### Success 3: Analysis Processing Complete
**Transition:**
- Wait screen → Preview screen (smooth transition)
- Success message: "Your Personalized Strategy Is Ready! 🎉"

**No modal, just direct redirect to preview**

---

### Success 4: PDF Downloaded
**Visual:**
- Browser native download confirmation
- Preview screen updates:
  - Button: "Download Your PDF" → "Download Again"
  - Message: "✓ PDF downloaded! Check your Downloads folder."
  - Telegram CTA highlighted

---

### Success 5: Joined Telegram Community
**Behavior:**
- Telegram link opens in new tab
- User joins group
- Analytics event fired
- Original tab stays on preview screen

**Message on preview:**
- "Great! Check Telegram for your welcome message."

---

### Success 6: Waitlist Signup Complete
**Behavior:**
- Modal or new page for email capture
- Submit email → confirmation
- Return to preview screen

**Message:**
- "✓ You're on the waitlist! We'll email you when the full app launches."

---

## Mobile Considerations

### Mobile-Specific Optimizations

**1. Touch Targets**
- All buttons minimum 44x44px (Apple HIG standard)
- Radio buttons, checkboxes larger than desktop
- Spacing between clickable elements ≥8px

**2. Text Input**
- Auto-focus on first field
- Appropriate keyboard types:
  - Number pad for hours input
  - Default keyboard for text
  - Date picker for deadlines
- "Next" button on keyboard advances fields
- "Done" on last field

**3. File Upload (PDF)**
- Mobile shows native file picker
- Options: Files app, iCloud, Google Drive
- Clear instructions: "Tap to select PDF from your files"
- No drag-and-drop (not supported on mobile)

**4. Scrolling Behavior**
- Step 1 & 2 are scrollable pages
- Fixed header (logo, progress, avatar)
- Fixed CTA button at bottom (always visible)
- Body content scrolls between

**5. Modal Behavior**
- Full-screen modals on mobile (not popups)
- Close button top-left
- Action buttons bottom (always visible)

**6. Processing Wait Screen**
- Fills entire screen
- No header/footer distractions
- Centered content
- Optimized for portrait orientation

**7. Preview Screen**
- Cards stack vertically
- Download button sticky at bottom
- Telegram button prominent
- All content scrollable

**8. Form Validation**
- Inline errors below fields (not tooltips)
- Clear error icons (⚠️)
- Red borders on invalid fields
- Error text in warm orange (not harsh red)

**9. Loading States**
- Skeletons instead of blank screens
- Progress indicators for uploads
- Friendly animations (not just spinners)

---

### Mobile vs. Desktop Differences

**Desktop Enhancements (not in mobile):**
- Hover states on buttons
- Tooltips on icons
- Drag-and-drop for PDF upload
- Side-by-side layout (e.g., 2 goals per row)
- Floating labels
- Keyboard shortcuts

**Mobile-Only Features:**
- Native file picker integration
- Swipe gestures (optional)
- Pull-to-refresh (if on history page)
- Bottom sheet modals
- Safe area insets (iOS notch support)

---

### Responsive Breakpoints

**Mobile:** 0-767px
- Single column layout
- Stacked cards
- Full-width buttons
- Larger touch targets

**Tablet:** 768-1023px
- 2-column layout where appropriate
- Larger text
- More whitespace

**Desktop:** 1024px+
- Max content width: 1200px (centered)
- Multi-column layouts
- Hover effects
- Desktop-optimized spacing

---

## Open Questions

### 1. Email Notifications
**Question:** Should we send email when analysis completes (for users who closed browser)?

**Current state:** No email notifications in MVP

**Considerations:**
- Pro: Better UX, users don't have to remember to check back
- Con: Requires email service integration (SendGrid, etc.)
- Con: Adds complexity

**Decision needed:** MVP or post-MVP feature?

---

### 2. Telegram Notification Alternative
**Question:** Could we send Telegram message instead of email when analysis completes?

**Considerations:**
- Pro: Already integrating Telegram for community
- Pro: More immediate, higher open rates
- Con: Requires users to opt-in to Telegram first
- Con: Adds complexity to onboarding

**Decision needed:** Worth exploring or save for post-MVP?

---

### 3. Progress Auto-Save Frequency
**Question:** How often should we auto-save Step 2 data?

**Options:**
- On every field change (real-time)
- Every 30 seconds (periodic)
- Only on "Next" button click (manual)

**Current spec:** Real-time auto-save (every field change)

**Decision:** Is real-time necessary or overkill?

---

### 4. PDF Preview Before Download
**Question:** Should preview screen show PDF thumbnail or just metadata?

**Current spec:** Metadata only (summary cards)

**Alternative:** Embed PDF viewer inline

**Considerations:**
- Pro: Users can see what they're getting
- Con: Slows page load, large file
- Con: May reduce download rate (if they can read it inline)

**Decision:** Stick with metadata or add inline viewer?

---

### 5. History Search/Filter
**Question:** If user has 20+ analyses, do they need search?

**Current spec:** Just chronological list

**Future consideration:** 
- Search by goal text
- Filter by date range
- Sort options

**Decision:** MVP or post-MVP?

---

### 6. Analysis Naming
**Question:** Can users name their analyses for easier history navigation?

**Current spec:** No naming, just date + goals preview

**Alternative:** Optional "Name this analysis" field

**Decision:** Add or skip for MVP?

---

### 7. Duplicate Goal Detection
**Question:** Should we warn if user sets very similar goals across multiple analyses?

**Example:** 
- Analysis 1: "Run a 5K"
- Analysis 2: "Run a 5K in under 30 minutes"

**Current spec:** No detection

**Decision:** Would this be helpful or annoying?

---

### 8. Social Sharing
**Question:** Should users be able to share their PDF or preview on social media?

**Considerations:**
- Pro: Organic growth, virality
- Con: Privacy concerns (personal goals)
- Con: May reduce perceived value (if everyone shares, not special)

**Current spec:** Share app link only, not personal results

**Decision:** Revisit post-launch based on user requests?

---

### 9. Offline Mode
**Question:** Should assessment work offline (with sync later)?

**Considerations:**
- Pro: Works on unstable connections
- Con: Complex state management
- Con: Can't do AI processing offline anyway

**Current spec:** Requires internet connection

**Decision:** MVP or future enhancement?

---

### 10. Multi-Language Support
**Question:** Is this MVP English-only?

**Current spec:** English only

**Future consideration:** Spanish, Portuguese, others?

**Decision:** When to add i18n?

---

## Validation & Testing Plan

### User Testing Scenarios

**Scenario 1: First-Time User (Happy Path)**
- Start on landing page (mobile)
- Read rate limit info
- Click "Get Strategy"
- Auth with Google
- Upload valid PDF
- Fill weekly hours (partial)
- Skip questionnaire
- Set 1 goal (high priority)
- Wait through processing
- View preview
- Download PDF
- Join Telegram

**Expected time:** 8-10 minutes

---

**Scenario 2: Power User (Multiple Goals)**
- Return user (already authed)
- Upload PDF quickly
- Fill all weekly hours
- Complete questionnaire
- Set 3 goals (varied priorities)
- Wait through processing
- Download PDF
- Check history

**Expected time:** 12-15 minutes

---

**Scenario 3: Error Recovery**
- Upload wrong PDF (not CliftonStrengths)
- See error message
- Upload correct PDF
- Set goals
- Partial analysis failure
- Retry failed goal
- Success

**Expected time:** 10-12 minutes

---

**Scenario 4: Rate Limit Hit**
- Complete 2 analyses
- Try 3rd
- See rate limit modal
- Join waitlist
- Return next day
- Complete 3rd analysis

**Expected time:** Across 2 days

---

### Key Metrics to Track

**Conversion Funnel:**
1. Landing page views
2. CTA clicks ("Get Strategy")
3. Google auth completions
4. Step 1 completions
5. Step 2 completions
6. "Analyse" button clicks
7. Processing completions
8. PDF downloads
9. Telegram joins
10. Waitlist signups

**Engagement Metrics:**
- Time on landing page
- Time to complete Step 1
- Time to complete Step 2
- Drop-off points
- Return visits
- Multiple analyses per user

**Error Metrics:**
- Invalid PDF uploads
- Parsing failures
- Processing timeouts
- Partial failures
- Rate limit hits

**Success Metrics:**
- Analysis completion rate (target: >85%)
- PDF download rate (target: >95%)
- Telegram conversion (target: >30%)
- Waitlist conversion (target: >40%)
- Return user rate (target: >50% create 2nd analysis)

---

## Appendix: User-Facing Messages

### Landing Page
- H1: "Turn Your CliftonStrengths Into Achievable Goals"
- H2: "Get a personalized strategy based on your unique personality—designed for how YOUR brain works"
- CTA: "Get Your Personalized Strategy"
- Badge: "✨ Free MVP • 2 analyses per day"

### Step 1
- Header: "Step 1 of 2"
- Upload: "Upload Your CliftonStrengths Report"
- Hours: "How do you currently spend your week?"
- Hint: "The more you fill out, the more precise your recommendations will be."
- CTA: "Next: Set Your Goals"

### Step 2
- Header: "Step 2 of 2"
- Heading: "What goals do you want to achieve?"
- Subtext: "Set 1-3 goals. We'll create a personalized strategy for each."
- CTA: "Analyse My Goals"
- Hint: "This takes ~2 minutes. You can close your browser—we'll save results."

### Processing Screen
- Stage 1: "Understanding your goals..."
- Stage 2: "Finding your strengths..."
- Stage 3: "Creating your strategy..."
- Stage 4: "Almost there..."
- Time: "This usually takes about 2 minutes"
- Browser hint: "Feel free to close this tab—we'll save your results."

### Preview Screen
- Heading: "Your Personalized Strategy Is Ready! 🎉"
- Subtext: "Here's what we created based on your CliftonStrengths profile and goals."
- CTA: "Download Your PDF"
- Secondary: "Join Our Telegram Community"

### Success Messages
- Upload: "✓ Report processed successfully"
- Download: "✓ PDF downloaded! Check your Downloads folder."
- Telegram: "Great! Now join our community to stay accountable →"

### Error Messages
- Invalid PDF: "We couldn't read that file. Please upload a PDF from Gallup CliftonStrengths."
- Too large: "File too large. Please upload a PDF smaller than 10MB."
- Wrong PDF: "This doesn't look like a CliftonStrengths report. Make sure it's the full PDF from Gallup."
- Rate limit: "You've used your 2 analyses today. Next slot opens in 4 hours 23 minutes."
- Processing fail: "We hit a snag. Want to try again?"

---

**Document Status:** Complete - Ready for Technical Implementation  
**Last Updated:** February 14, 2026  
**Next Document:** Database Schema & Data Models  
**Owner:** Igor (Founder, PM, Designer)

---

*This specification defines every screen, state, error, and interaction in the user journey. It's designed to guide frontend development with crystal clarity while maintaining the warm, supportive tone that makes this app feel like a friend, not a tool.*
