# PDF Template & Design Specification
## Personal Time OS - Goal Achievement MVP

**Version:** 1.0  
**Created:** February 14, 2026  
**Purpose:** Complete design specification for personalized PDF strategy documents  
**Target:** Claude Code implementation with HTML→PDF generation

---

## Executive Summary

This document defines the complete design specification for the personalized PDF strategy documents that users download after completing their analysis. The PDF combines comprehensive personalized content (AI-generated recommendations) with warm, actionable design that feels like a professional workbook, not a corporate report.

**Key Design Principles:**
- **Action-first content structure** - Users see keystone habits immediately after cover
- **Warm workbook aesthetic** - Cream, orange, gold palette with rounded corners and friendly typography
- **Visual hierarchy** - Clear distinction between high/medium/low priority goals
- **Printable and practical** - Designed for users to print, highlight, and post on walls
- **Adaptive layout** - Gracefully handles 1, 2, or 3 goals without wasted space

**Page Structure Overview (15-20 pages typical for 3 goals):**

```
┌─────────────────────────────────────┐
│ 1. COVER PAGE                       │ ← Personalized, warm branding
├─────────────────────────────────────┤
│ 2. QUICK START GUIDE                │ ← Keystone habits only, "start today"
├─────────────────────────────────────┤
│ 3. EXECUTIVE SUMMARY                │ ← Big picture, why this works
├─────────────────────────────────────┤
│ 4-6. HIGH-PRIORITY GOAL (2-3 pages) │ ← Orange accent, detailed strategy
├─────────────────────────────────────┤
│ 7-8. MEDIUM GOAL (1.5-2 pages)      │ ← If exists, standard styling
├─────────────────────────────────────┤
│ 9-10. LOW GOAL (1.5-2 pages)        │ ← If exists, standard styling
├─────────────────────────────────────┤
│ 11-12. CLIFTONSTRENGTHS PROFILE     │ ← Visual hierarchy, domain insights
├─────────────────────────────────────┤
│ 13-15. TRACKING TOOLS               │ ← 100-dot grid + checklists
├─────────────────────────────────────┤
│ 16-17. APPENDIX                     │ ← CD1-8 reference, next steps
└─────────────────────────────────────┘
```

**Technical Implementation:**
- **Generation method:** HTML→PDF via Puppeteer/Playwright
- **Templating:** Handlebars-style placeholders (`{{variable}}`)
- **Styling:** CSS with brand colors, responsive layout principles
- **Adaptivity:** Loop-based goal sections, conditional tracking tools
- **File size target:** 2-5 MB (includes graphics, readable text)

**Design Philosophy:**

This PDF should feel like **"a thoughtful workshop handout you'd want to keep and reference"** rather than a generic automated report. Every element serves actionability: users should be able to print it, highlight their keystone habit, and start today. The warm aesthetic (cream background, orange accents, rounded corners) matches the brand positioning of "supportive friend inviting you to change your life."

---

## Design System

### Color Palette

**Primary Colors:**
- **Cream Background:** `#FAF8F3` (soft, warm base for all pages)
- **Deep Orange:** `#E07A3E` (primary accent, high-priority elements)
- **Warm Gold:** `#D4A574` (secondary accent, badges, icons)
- **Charcoal Text:** `#2C2C2C` (primary text, high contrast on cream)

**Supporting Colors:**
- **Soft Gray:** `#6B6B6B` (secondary text, descriptions)
- **Light Orange:** `#F7E6D9` (subtle backgrounds for callouts)
- **Border Gray:** `#E0DDD6` (subtle dividers, box borders)
- **Success Green:** `#6A9E7F` (checkmarks, completed states - if needed)

**Usage Guidelines:**
- Cream background on ALL pages (consistency, warm feel)
- Deep Orange for: high-priority badges, keystone habit boxes, CTAs
- Warm Gold for: icons, section headers, accents
- Charcoal for: body text (never pure black - too harsh)
- Light Orange for: callout boxes ("Why this works" sections)

### Typography

**Font Stack:**
- **Headings:** Inter, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif
- **Body:** Inter, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif
- **Fallback:** System sans-serif (ensures readability even if web fonts fail)

**Type Scale:**

| Element | Size | Weight | Color | Line Height |
|---------|------|--------|-------|-------------|
| Page Title (Cover) | 36px | 700 (Bold) | Charcoal | 1.2 |
| Section Headers | 24px | 600 (Semibold) | Charcoal | 1.3 |
| Subsection Headers | 18px | 600 (Semibold) | Charcoal | 1.4 |
| Body Text | 14px | 400 (Regular) | Charcoal | 1.6 |
| Small Text/Labels | 12px | 400 (Regular) | Soft Gray | 1.5 |
| Action Titles | 16px | 600 (Semibold) | Charcoal | 1.4 |
| Emphasis | 14px | 500 (Medium) | Deep Orange | 1.6 |

**Typography Guidelines:**
- Use sentence case for headers (not ALL CAPS - too corporate)
- Generous line-height for readability (1.6 for body text)
- Never use pure black (#000000) - too harsh on cream background
- Bold sparingly (only for keystone habits and priority badges)

### Layout & Spacing

**Page Setup:**
- **Size:** US Letter (8.5" × 11" / 216mm × 279mm)
- **Orientation:** Portrait
- **Margins:** 1 inch (25.4mm) all sides (printable area)
- **Safe zone:** 0.75 inch (19mm) from edges (critical content)

**Grid System:**
- **Content width:** 6.5 inches (165mm) - centered
- **Gutter:** 0.5 inch (12.7mm) between columns
- **Baseline grid:** 8px (for vertical rhythm)

**Spacing Scale:**

| Token | Pixels | Usage |
|-------|--------|-------|
| xs | 4px | Tight spacing (icon to text) |
| sm | 8px | Element spacing within components |
| md | 16px | Default vertical spacing between elements |
| lg | 24px | Section spacing |
| xl | 32px | Major section breaks |
| 2xl | 48px | Page section dividers |
| 3xl | 64px | Between major content blocks |

**Layout Principles:**
- Generous white space (cream, not white) - avoid cramped feeling
- Consistent left alignment (easier to scan than centered)
- Rounded corners on all boxes (12px border-radius)
- Subtle shadows for depth (not flat, but not heavy drop shadows)

### Component Library

**Box Component:**
```css
.box {
  background: white;
  border: 1px solid #E0DDD6;
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 16px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}

.box-accent {
  border-left: 4px solid #E07A3E;
}
```

**Badge Component:**
```css
.badge {
  display: inline-block;
  padding: 4px 12px;
  border-radius: 16px;
  font-size: 12px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.badge-high {
  background: #E07A3E;
  color: white;
}

.badge-medium {
  background: #D4A574;
  color: white;
}

.badge-low {
  background: #E0DDD6;
  color: #6B6B6B;
}
```

**Icon System:**
- Use simple SVG icons (single color, 16-24px)
- Icons for: clock (time), target (difficulty), rocket (Core Drives)
- Warm Gold color (#D4A574) for all icons
- Position: left of text, 8px spacing

---

## Page-by-Page Specifications

### Page 1: Cover Page

**Layout:**

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│              [Small logo or branding]               │
│                                                     │
│                                                     │
│         YOUR PERSONALIZED GOAL                      │
│            ACHIEVEMENT STRATEGY                     │
│                                                     │
│         [User Name] • [Dominant Domain]             │
│              Generated: [Date]                      │
│                                                     │
│                                                     │
│              ┌──────────────────────┐               │
│              │  Inside this guide:  │               │
│              │  • [Goal 1 name]     │               │
│              │  • [Goal 2 name]     │  (if exists) │
│              │  • [Goal 3 name]     │  (if exists) │
│              └──────────────────────┘               │
│                                                     │
│                                                     │
│     [Subtle decorative element - warm colors]      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Content Placeholders:**
- `{{user_name}}` - From Google Auth or user input
- `{{dominant_domain}}` - From CliftonStrengths parser (e.g., "Strategic Thinking")
- `{{generated_date}}` - Format: "February 14, 2026"
- `{{goal_1_description}}` - Truncated to 60 chars max
- `{{goal_2_description}}` - Conditional, if exists
- `{{goal_3_description}}` - Conditional, if exists

**Styling Details:**
- Background: Full-page cream (#FAF8F3)
- Title: 36px, charcoal, centered
- Subtitle: 18px, soft gray, centered
- Preview box: White background, rounded corners, subtle border
- Decorative element: Abstract shapes in orange/gold gradient (optional, non-critical)

**Implementation Notes:**
- Keep simple and clean (avoid over-designing)
- Ensure all text is readable even if colors fail to render
- Logo/branding should be SVG (scalable, small file size)

---

### Page 2: Quick Start Guide

**Purpose:** One-page overview of ALL keystone habits so user can start TODAY.

**Layout:**

```
┌─────────────────────────────────────────────────────┐
│  QUICK START GUIDE                                  │
│  Start Today With These Keystone Habits             │
│  ────────────────────────────────────────────────   │
│                                                     │
│  [If High-Priority Goal exists:]                    │
│  ┌───────────────────────────────────────────────┐ │
│  │ 🎯 HIGH PRIORITY: [Goal Description]          │ │
│  │                                                │ │
│  │ ⭐ START HERE                                 │ │
│  │ [Keystone Habit Title]                        │ │
│  │ [Brief description - 1-2 sentences]           │ │
│  │                                                │ │
│  │ ⏱ Time: [X minutes] • 🔄 Frequency: [Daily]  │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  [If Medium-Priority Goal exists:]                  │
│  ┌───────────────────────────────────────────────┐ │
│  │ 🔸 MEDIUM PRIORITY: [Goal Description]        │ │
│  │                                                │ │
│  │ [Keystone Habit Title]                        │ │
│  │ [Brief description]                           │ │
│  │ ⏱ [X min] • 🔄 [Frequency]                   │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  [If Low-Priority Goal exists:]                     │
│  ┌───────────────────────────────────────────────┐ │
│  │ 🔹 LOW PRIORITY: [Goal Description]           │ │
│  │                                                │ │
│  │ [Keystone Habit Title]                        │ │
│  │ [Brief description]                           │ │
│  │ ⏱ [X min] • 🔄 [Frequency]                   │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  💡 TIP: Start with your high-priority keystone    │
│     habit. The others can wait until week 2.       │
└─────────────────────────────────────────────────────┘
```

**Content Placeholders:**
- `{{high_goal.description}}` - Conditional
- `{{high_goal.keystone_habit.title}}`
- `{{high_goal.keystone_habit.description}}`
- `{{high_goal.keystone_habit.estimated_time_minutes}}`
- `{{high_goal.keystone_habit.frequency}}`
- (Similar for medium and low goals)

**Styling Details:**
- High-priority box: Orange left border (4px), larger font
- Medium/low boxes: Standard white boxes
- Icons: Small emoji or SVG (16px)
- Tip box at bottom: Light orange background

**Implementation Notes:**
- This page ONLY shows keystone habits (core actions come later)
- Conditional rendering: Only show boxes for goals that exist
- Keep descriptions brief (max 2 sentences)

---

### Page 3: Executive Summary

**Purpose:** Big-picture overview explaining the strategy and why it works for this user.

**Layout:**

```
┌─────────────────────────────────────────────────────┐
│  EXECUTIVE SUMMARY                                  │
│  ────────────────────────────────────────────────   │
│                                                     │
│  [3-4 paragraphs from Synthesizer output]          │
│                                                     │
│  Your CliftonStrengths profile reveals a           │
│  [Dominant Domain] dominant domain, with           │
│  [Top Theme] as your #1 theme. This combination    │
│  makes you particularly well-suited for...         │
│                                                     │
│  The [X] goal[s] you've set - [list goals] -       │
│  all leverage your natural [strengths/patterns]... │
│                                                     │
│  With an estimated [X] hours per week required,    │
│  these goals are [ambitious/achievable/realistic]  │
│  given your current schedule.                      │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │ AT A GLANCE                                    │ │
│  │                                                │ │
│  │ • Number of Goals: [X]                        │ │
│  │ • Dominant Domain: [Strategic Thinking]       │ │
│  │ • Total Time/Week: [X] hours                  │ │
│  │ • Primary Core Drives: CD[X], CD[Y]           │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Content Placeholders:**
- `{{synthesizer.executive_summary}}` - Full text from AI
- `{{clifton_profile.dominant_domain}}`
- `{{clifton_profile.top_theme.name}}`
- `{{goal_count}}` - Number (1-3)
- `{{total_estimated_time_per_week}}` - From synthesizer
- `{{primary_core_drives}}` - Array of numbers (CD2, CD4, etc.)

**Styling Details:**
- Body text: 14px, line-height 1.6, charcoal color
- "At a Glance" box: White background, subtle border, rounded corners
- Bullet points: Use • character, not default bullets

**Implementation Notes:**
- Executive summary is direct copy from synthesizer output
- "At a Glance" box is generated from metadata
- Keep formatting simple (just paragraphs, no excessive styling)

---

### Pages 4-6: High-Priority Goal Strategy (2-3 pages)

**Page 4a: Goal Overview**

```
┌─────────────────────────────────────────────────────┐
│  ┌─────────────────────────────────────────────┐   │
│  │ 🎯 HIGH PRIORITY                             │   │ ← Orange badge
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  [Goal Description]                                 │
│  Deadline: [Date] • Difficulty: ⭐⭐⭐ (3/5)        │
│  Categories: [Work & Career, Health & Fitness]      │
│  ────────────────────────────────────────────────   │
│                                                     │
│  STRATEGY OVERVIEW                                  │
│  [2-3 sentence overview from Goal Agent]           │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │ 💡 WHY THIS WORKS FOR YOU                     │ │
│  │                                                │ │
│  │ [Explanation connecting CliftonStrengths to   │ │
│  │  this specific strategy - from Goal Agent]    │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  ESTIMATED TIME COMMITMENT                          │
│  [X] hours per week                                 │
│  [If capacity warning exists: display here]        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Content Placeholders:**
- `{{high_goal.description}}`
- `{{high_goal.deadline}}` - Format: "June 30, 2026"
- `{{high_goal.hardness_rating}}` - Display as stars (1-5)
- `{{high_goal.categories}}` - Array, comma-separated
- `{{high_goal.strategy_overview}}`
- `{{high_goal.why_this_works}}`
- `{{high_goal.estimated_time_per_week}}`
- `{{high_goal.capacity_warning}}` - Conditional

**Styling Details:**
- HIGH PRIORITY badge: Deep orange background, white text, uppercase
- "Why This Works" box: Light orange background, warm gold left border
- Difficulty stars: Orange filled stars for rating, gray outline for remaining
- Orange accent line under goal description

---

**Page 4b: Action Plan - Keystone Habit**

```
┌─────────────────────────────────────────────────────┐
│  YOUR ACTION PLAN                                   │
│  ────────────────────────────────────────────────   │
│                                                     │
│  ⭐ KEYSTONE HABIT: START HERE                     │
│  ┌───────────────────────────────────────────────┐ │
│  │                                                │ │
│  │  [Keystone Habit Title]                       │ │
│  │  [Full description from Goal Agent]           │ │
│  │                                                │ │
│  │  ⏱ Time: [X] minutes                         │ │
│  │  🔄 Frequency: [Daily]                        │ │
│  │  📊 Difficulty: ⚫⚫⚪ (2/3)                    │ │
│  │                                                │ │
│  │  🎮 Core Drives: CD[2], CD[4]                │ │
│  │  💪 Strengths Used: [Achiever, Discipline]   │ │
│  │                                                │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  CORE ACTIONS                                       │
│  These build on your keystone habit:                │
│                                                     │
│  1. [Core Action 1 Title]                          │
│     [Description]                                   │
│     ⏱ [X min] • 🔄 [Frequency] • 📊 [Diff]       │
│     🎮 CD[X] • 💪 [Strengths]                     │
│                                                     │
│  2. [Core Action 2 Title]                          │
│     [Description]                                   │
│     ⏱ [X min] • 🔄 [Frequency] • 📊 [Diff]       │
│                                                     │
│  3. [Core Action 3 Title]                          │
│     [Description]                                   │
│     ⏱ [X min] • 🔄 [Frequency] • 📊 [Diff]       │
│                                                     │
│  [Continue for all 3-5 core actions]               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Content Placeholders:**
- `{{high_goal.keystone_habit.title}}`
- `{{high_goal.keystone_habit.description}}`
- `{{high_goal.keystone_habit.estimated_time_minutes}}`
- `{{high_goal.keystone_habit.frequency}}`
- `{{high_goal.keystone_habit.difficulty_level}}` - Display as dots (1-3)
- `{{high_goal.keystone_habit.core_drives}}` - Array
- `{{high_goal.keystone_habit.strengths_used}}` - Array
- `{{high_goal.core_actions[]}}` - Loop through array

**Styling Details:**
- Keystone habit box: Orange border (4px left), larger text, white background
- Core actions: Numbered list, consistent spacing
- Icons: Small SVG or emoji, 16px, gold color
- Difficulty dots: Filled for level, empty for remaining (⚫⚪⚪)

---

**Page 4c: Bonus Accelerators & Milestones**

```
┌─────────────────────────────────────────────────────┐
│  BONUS ACCELERATORS (OPTIONAL)                      │
│  ────────────────────────────────────────────────   │
│                                                     │
│  Once you've mastered your core actions, try these: │
│                                                     │
│  ⚡ [Bonus Accelerator 1 Title]                    │
│     [Description]                                   │
│     ⏱ [X min] • 🔄 [Frequency] • 📊 [Diff]       │
│                                                     │
│  ⚡ [Bonus Accelerator 2 Title]                    │
│     [Description]                                   │
│     ⏱ [X min] • 🔄 [Frequency] • 📊 [Diff]       │
│                                                     │
│  [Loop for 2-3 bonus items]                        │
│                                                     │
│  ────────────────────────────────────────────────   │
│                                                     │
│  MILESTONES TO YOUR DEADLINE                        │
│                                                     │
│  ┌─────────────────────────────────────────┐       │
│  │ Week [2]: [Milestone Description]       │       │
│  │ ✓ Success Criteria: [Criteria]          │       │
│  └─────────────────────────────────────────┘       │
│                                                     │
│  ┌─────────────────────────────────────────┐       │
│  │ Week [6]: [Milestone Description]       │       │
│  │ ✓ Success Criteria: [Criteria]          │       │
│  └─────────────────────────────────────────┘       │
│                                                     │
│  [Continue for all 3-5 milestones]                 │
│                                                     │
│  ────────────────────────────────────────────────   │
│                                                     │
│  TRACKING METHOD                                    │
│  [Tracking method description from Goal Agent]     │
│  → See page [X] for your tracking tools            │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Content Placeholders:**
- `{{high_goal.bonus_accelerators[]}}` - Loop through array
- `{{high_goal.milestones[]}}` - Loop through array
- `{{high_goal.milestones[].week}}`
- `{{high_goal.milestones[].description}}`
- `{{high_goal.milestones[].success_criteria}}`
- `{{high_goal.tracking_method}}`
- `{{tracking_page_number}}` - Dynamic reference

**Styling Details:**
- Bonus section: Lighter emphasis (this is optional content)
- Milestones: White boxes, checkmark icon, numbered by week
- Tracking method: Light orange callout box

---

### Pages 7-8: Medium-Priority Goal Strategy (1.5-2 pages)

**Structure:** Same as high-priority goal but condensed:

**Page 7: Goal Overview + Action Plan (combined)**

```
┌─────────────────────────────────────────────────────┐
│  ┌─────────────────────────────────────────────┐   │
│  │ 🔸 MEDIUM PRIORITY                           │   │ ← Gold badge
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  [Goal Description]                                 │
│  Deadline: [Date] • Difficulty: ⭐⭐⭐            │
│  ────────────────────────────────────────────────   │
│                                                     │
│  STRATEGY OVERVIEW                                  │
│  [Strategy overview - from Goal Agent]             │
│                                                     │
│  💡 WHY THIS WORKS FOR YOU                         │
│  [Brief explanation]                                │
│                                                     │
│  ────────────────────────────────────────────────   │
│                                                     │
│  ⭐ KEYSTONE HABIT                                 │
│  [Title]                                           │
│  [Description]                                      │
│  ⏱ [X min] • 🔄 [Frequency]                       │
│                                                     │
│  CORE ACTIONS                                       │
│  1. [Action 1] - ⏱ [X min] • 🔄 [Freq]          │
│  2. [Action 2] - ⏱ [X min] • 🔄 [Freq]          │
│  3. [Action 3] - ⏱ [X min] • 🔄 [Freq]          │
│  [Continue for all actions]                        │
│                                                     │
│  BONUS ACCELERATORS (OPTIONAL)                     │
│  ⚡ [Bonus 1]                                      │
│  ⚡ [Bonus 2]                                      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Page 8: Milestones**

```
┌─────────────────────────────────────────────────────┐
│  MILESTONES                                         │
│  ────────────────────────────────────────────────   │
│                                                     │
│  Week [X]: [Milestone]                             │
│  ✓ [Success criteria]                              │
│                                                     │
│  [Loop for all milestones]                         │
│                                                     │
│  ────────────────────────────────────────────────   │
│                                                     │
│  TRACKING METHOD                                    │
│  [Method description]                               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Styling Differences from High-Priority:**
- Badge: Warm gold (#D4A574) instead of deep orange
- No orange accent border (just standard white boxes)
- Slightly smaller headings (22px vs 24px)
- More condensed spacing (actions in tighter list format)

---

### Pages 9-10: Low-Priority Goal Strategy (1.5-2 pages)

**Structure:** Identical to medium-priority goal structure

**Styling Differences:**
- Badge: Soft gray background (#E0DDD6) with charcoal text
- Fully condensed layout (minimal white space)
- Standard box styling throughout

---

### Pages 11-12: CliftonStrengths Profile Deep-Dive

**Page 11: Your Strengths Overview**

```
┌─────────────────────────────────────────────────────┐
│  YOUR CLIFTONSTRENGTHS PROFILE                      │
│  ────────────────────────────────────────────────   │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │ 💡 DOMINANT DOMAIN: [Strategic Thinking]      │ │
│  │                                                │ │
│  │ [2-3 paragraph explanation from Synthesizer   │ │
│  │  about what this domain means and how it      │ │
│  │  influences the overall strategy]             │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  YOUR TOP 10 THEMES                                 │
│  ────────────────────────────────────────────────   │
│                                                     │
│  1. [Theme Name] - [Domain]                        │
│  2. [Theme Name] - [Domain]                        │
│  3. [Theme Name] - [Domain]                        │
│  4. [Theme Name] - [Domain]                        │
│  5. [Theme Name] - [Domain]                        │
│  ────────────────                                   │
│  6. [Theme Name] - [Domain]                        │
│  7. [Theme Name] - [Domain]                        │
│  8. [Theme Name] - [Domain]                        │
│  9. [Theme Name] - [Domain]                        │
│  10. [Theme Name] - [Domain]                       │
│                                                     │
│  YOUR BOTTOM 5 THEMES                               │
│  These are areas that take more energy for you:     │
│                                                     │
│  30. [Theme Name] - [Domain]                       │
│  31. [Theme Name] - [Domain]                       │
│  32. [Theme Name] - [Domain]                       │
│  33. [Theme Name] - [Domain]                       │
│  34. [Theme Name] - [Domain]                       │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Content Placeholders:**
- `{{clifton_profile.dominant_domain}}`
- `{{synthesizer.clifton_strengths_overview}}` - Full explanation
- `{{clifton_profile.top_ten_themes[]}}` - Loop
- `{{clifton_profile.top_ten_themes[].name}}`
- `{{clifton_profile.top_ten_themes[].domain}}` - Map theme to domain
- `{{clifton_profile.bottom_five_themes[]}}` - Loop

**Styling Details:**
- Dominant domain box: Light orange background, larger text
- Top 5 themes: Larger font (16px), bold
- Themes 6-10: Regular font (14px)
- Bottom 5: Smaller, softer gray text
- Visual separator line between ranks 5 and 6

**Domain Icon Logic:**
- Strategic Thinking: 💡 (lightbulb)
- Executing: ⚙️ (gear)
- Influencing: 📢 (megaphone)
- Relationship Building: 🤝 (handshake)

---

**Page 12: How Your Strengths Shape Your Strategy**

```
┌─────────────────────────────────────────────────────┐
│  HOW YOUR STRENGTHS SHAPE YOUR STRATEGY             │
│  ────────────────────────────────────────────────   │
│                                                     │
│  [2-3 paragraphs from Synthesizer explaining       │
│   how the CliftonStrengths profile influenced      │
│   the specific action recommendations]             │
│                                                     │
│  Your top theme, [Theme #1], appears in [X] of     │
│  your action plans. Here's how it shows up:        │
│                                                     │
│  • [Example from high-priority goal]               │
│  • [Example from medium goal]                      │
│  • [Example from low goal]                         │
│                                                     │
│  ────────────────────────────────────────────────   │
│                                                     │
│  💡 UNDERSTANDING THE DOMAINS                       │
│                                                     │
│  [4 small boxes in 2x2 grid, each explaining a     │
│   CliftonStrengths domain briefly]                 │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐                │
│  │ 💡 STRATEGIC │  │ ⚙️ EXECUTING │                │
│  │ THINKING     │  │              │                │
│  │ [1-2 sent.]  │  │ [1-2 sent.]  │                │
│  └──────────────┘  └──────────────┘                │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐                │
│  │ 📢 INFLUENC. │  │ 🤝 RELATION. │                │
│  │              │  │ BUILDING     │                │
│  │ [1-2 sent.]  │  │ [1-2 sent.]  │                │
│  └──────────────┘  └──────────────┘                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Content Placeholders:**
- `{{synthesizer.clifton_strengths_overview}}` - Continued
- `{{top_theme_name}}` - From rank #1
- `{{top_theme_examples[]}}` - Generated list of how it appears in actions
- Domain descriptions: Static text (same for all users)

**Styling Details:**
- Domain boxes: White background, rounded corners, icon + text
- Keep domain descriptions brief (1-2 sentences each)
- 2x2 grid layout, equal sizing

---

### Pages 13-15: Tracking Tools

**Page 13: 100-Dot Grid (High-Priority Goal Only)**

```
┌─────────────────────────────────────────────────────┐
│  YOUR TRACKING TOOLS                                │
│  ────────────────────────────────────────────────   │
│                                                     │
│  100-DOT GRID: [High-Priority Goal Name]           │
│                                                     │
│  Track your progress by marking one dot each time  │
│  you complete your keystone habit. Seeing your     │
│  progress grow is powerful motivation.             │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │ ○ ○ ○ ○ ○ ○ ○ ○ ○ ○  (10 dots)              │ │
│  │ ○ ○ ○ ○ ○ ○ ○ ○ ○ ○                         │ │
│  │ ○ ○ ○ ○ ○ ○ ○ ○ ○ ○  ← Repeat 10 rows       │ │
│  │ ○ ○ ○ ○ ○ ○ ○ ○ ○ ○                         │ │
│  │ ○ ○ ○ ○ ○ ○ ○ ○ ○ ○                         │ │
│  │ ○ ○ ○ ○ ○ ○ ○ ○ ○ ○                         │ │
│  │ ○ ○ ○ ○ ○ ○ ○ ○ ○ ○                         │ │
│  │ ○ ○ ○ ○ ○ ○ ○ ○ ○ ○                         │ │
│  │ ○ ○ ○ ○ ○ ○ ○ ○ ○ ○                         │ │
│  │ ○ ○ ○ ○ ○ ○ ○ ○ ○ ○  (100 total dots)       │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  💡 TIP: Post this on your wall where you'll see   │
│     it every day. Each dot you fill is a win!      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Content Placeholders:**
- `{{high_goal.description}}` - Goal name
- 100-dot grid: Static SVG or CSS-generated grid

**Styling Details:**
- Dots: Circles, 20px diameter, 8px spacing between
- Empty dots: Stroke only (#E0DDD6), no fill
- Grid: 10x10 arrangement, centered on page
- Tearable: Add scissor icon and dashed line at top margin

**Implementation Notes:**
- ONLY generate if high-priority goal exists
- SVG for dots (scalable, clean print)
- Consider adding faint week markers (every 7 dots)

---

**Page 14: Checklists (Medium & Low Goals)**

```
┌─────────────────────────────────────────────────────┐
│  PROGRESS CHECKLISTS                                │
│  ────────────────────────────────────────────────   │
│                                                     │
│  [If Medium-Priority Goal exists:]                  │
│                                                     │
│  🔸 MEDIUM PRIORITY: [Goal Name]                   │
│  [Tracking method description]                     │
│                                                     │
│  Week 1:  ☐ ☐ ☐ ☐ ☐ ☐ ☐  (Daily checkboxes)      │
│  Week 2:  ☐ ☐ ☐ ☐ ☐ ☐ ☐                          │
│  Week 3:  ☐ ☐ ☐ ☐ ☐ ☐ ☐                          │
│  Week 4:  ☐ ☐ ☐ ☐ ☐ ☐ ☐                          │
│  [Continue for 12-16 weeks or until deadline]      │
│                                                     │
│  ────────────────────────────────────────────────   │
│                                                     │
│  [If Low-Priority Goal exists:]                     │
│                                                     │
│  🔹 LOW PRIORITY: [Goal Name]                      │
│  [Tracking method description]                     │
│                                                     │
│  Week 1:  ☐ ☐ ☐ ☐ ☐ ☐ ☐                          │
│  Week 2:  ☐ ☐ ☐ ☐ ☐ ☐ ☐                          │
│  [Continue for duration]                           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Content Placeholders:**
- `{{medium_goal.description}}` - Conditional
- `{{medium_goal.tracking_method}}`
- `{{low_goal.description}}` - Conditional
- `{{low_goal.tracking_method}}`
- Number of weeks: Calculate from deadline

**Styling Details:**
- Checkboxes: Empty squares (☐), 16px, gray border
- Weekly rows: Consistent spacing
- Goal headers: Use priority badge colors

**Implementation Notes:**
- Calculate weeks needed based on goal deadline
- If weekly tracking (not daily), adjust checkbox count
- Conditional rendering based on goal existence

---

**Page 15: Milestone Tracker (All Goals)**

```
┌─────────────────────────────────────────────────────┐
│  MILESTONE TRACKER                                  │
│  ────────────────────────────────────────────────   │
│                                                     │
│  Mark milestones as you achieve them. Review every  │
│  Sunday to stay on track.                          │
│                                                     │
│  🎯 HIGH PRIORITY: [Goal Name]                     │
│  ☐ Week [X]: [Milestone] - [Success Criteria]     │
│  ☐ Week [X]: [Milestone] - [Success Criteria]     │
│  ☐ Week [X]: [Milestone] - [Success Criteria]     │
│  [All milestones]                                  │
│                                                     │
│  ────────────────────────────────────────────────   │
│                                                     │
│  🔸 MEDIUM PRIORITY: [Goal Name]                   │
│  ☐ Week [X]: [Milestone]                          │
│  ☐ Week [X]: [Milestone]                          │
│  [All milestones]                                  │
│                                                     │
│  ────────────────────────────────────────────────   │
│                                                     │
│  🔹 LOW PRIORITY: [Goal Name]                      │
│  ☐ Week [X]: [Milestone]                          │
│  ☐ Week [X]: [Milestone]                          │
│  [All milestones]                                  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Content Placeholders:**
- Loop through all goals and their milestones
- `{{goal.milestones[].week}}`
- `{{goal.milestones[].description}}`
- `{{goal.milestones[].success_criteria}}`

**Styling Details:**
- Large checkboxes (easy to mark with pen)
- Week numbers in bold
- Milestone descriptions in regular text
- Success criteria in lighter gray (optional detail)

---

### Pages 16-17: Appendix

**Page 16: Octalysis Core Drives Reference**

```
┌─────────────────────────────────────────────────────┐
│  APPENDIX: OCTALYSIS CORE DRIVES                    │
│  ────────────────────────────────────────────────   │
│                                                     │
│  Your recommendations reference Core Drives (CD1-8).│
│  Here's what they mean:                            │
│                                                     │
│  CD1: Epic Meaning & Calling                       │
│  You're part of something bigger than yourself.    │
│                                                     │
│  CD2: Development & Accomplishment                  │
│  Making progress, leveling up, achieving mastery.  │
│                                                     │
│  CD3: Empowerment of Creativity & Feedback         │
│  Creative expression and seeing results of actions.│
│                                                     │
│  CD4: Ownership & Possession                        │
│  Building something you own, personalizing it.     │
│                                                     │
│  CD5: Social Influence & Relatedness               │
│  Connection, competition, collaboration with others.│
│                                                     │
│  CD6: Scarcity & Impatience                        │
│  Wanting what you can't have, fear of missing out. │
│                                                     │
│  CD7: Unpredictability & Curiosity                 │
│  Surprise, discovery, mystery that pulls you in.   │
│                                                     │
│  CD8: Loss & Avoidance                             │
│  Avoiding negative outcomes, not losing progress.  │
│                                                     │
│  ────────────────────────────────────────────────   │
│                                                     │
│  💡 Your strategy primarily uses: [List primary CDs]│
│     These match your CliftonStrengths profile.     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Content Placeholders:**
- Core Drive descriptions: Static text (same for all users)
- `{{primary_core_drives}}` - From synthesizer, list of CD numbers

**Styling Details:**
- Simple list format
- CD numbers in bold
- Descriptions in regular text
- Bottom callout box highlights user's primary drives

---

**Page 17: Next Steps & Community**

```
┌─────────────────────────────────────────────────────┐
│  YOUR NEXT STEPS                                    │
│  ────────────────────────────────────────────────   │
│                                                     │
│  ✓ [Next Step 1 from Synthesizer]                 │
│  ✓ [Next Step 2]                                   │
│  ✓ [Next Step 3]                                   │
│  ✓ [Next Step 4]                                   │
│  ✓ [Next Step 5]                                   │
│                                                     │
│  ────────────────────────────────────────────────   │
│                                                     │
│  JOIN OUR COMMUNITY                                 │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │ 💬 Telegram Community                         │ │
│  │                                                │ │
│  │ Connect with others using CliftonStrengths    │ │
│  │ for goal achievement. Share progress, get     │ │
│  │ support, stay accountable.                    │ │
│  │                                                │ │
│  │ → [Telegram Link/QR Code]                     │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │ 📧 Want More?                                 │ │
│  │                                                │ │
│  │ This MVP proves the value of personalized     │ │
│  │ strategies. Join our waitlist for the full    │ │
│  │ tracking app coming soon.                     │ │
│  │                                                │ │
│  │ → [Waitlist Link/QR Code]                     │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  ────────────────────────────────────────────────   │
│                                                     │
│  [Motivational Closing from Synthesizer]           │
│                                                     │
│  Good luck on your journey!                        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Content Placeholders:**
- `{{synthesizer.next_steps[]}}` - Array of 3-5 items
- `{{telegram_link}}` - Dynamic or static
- `{{waitlist_link}}` - Dynamic or static
- `{{synthesizer.motivational_closing}}` - Full paragraph

**Styling Details:**
- Next steps: Checkboxes (to encourage completion)
- Community boxes: Light orange background, CTA emphasis
- QR codes: Generate dynamically from links (optional but nice)
- Motivational closing: Larger text, centered, warm tone

---

## Technical Implementation Guide

### HTML Template Structure

**File Organization:**
```
/templates
  /pdf
    - base.html          (Master layout with header/footer)
    - cover.html         (Cover page partial)
    - quick-start.html   (Quick start partial)
    - executive.html     (Executive summary partial)
    - goal-strategy.html (Goal template - reusable)
    - profile.html       (CliftonStrengths profile partial)
    - tracking.html      (Tracking tools partial)
    - appendix.html      (Appendix partial)
```

**Base Template Pattern:**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{user_name}} - Goal Achievement Strategy</title>
  <style>
    /* Inline CSS for PDF generation */
    @page {
      size: letter;
      margin: 1in;
    }
    
    body {
      font-family: Inter, -apple-system, sans-serif;
      font-size: 14px;
      line-height: 1.6;
      color: #2C2C2C;
      background: #FAF8F3;
    }
    
    /* Include all component styles inline */
  </style>
</head>
<body>
  {{> cover}}
  <div class="page-break"></div>
  
  {{> quick-start}}
  <div class="page-break"></div>
  
  {{> executive}}
  <div class="page-break"></div>
  
  {{#each goals}}
    {{> goal-strategy goal=this}}
    <div class="page-break"></div>
  {{/each}}
  
  {{> profile}}
  <div class="page-break"></div>
  
  {{> tracking}}
  <div class="page-break"></div>
  
  {{> appendix}}
</body>
</html>
```

### CSS Styling Reference

**Complete Stylesheet (Inline in HTML):**

```css
/* Page Setup */
@page {
  size: letter;
  margin: 1in;
}

body {
  font-family: Inter, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  font-size: 14px;
  line-height: 1.6;
  color: #2C2C2C;
  background: #FAF8F3;
  margin: 0;
  padding: 0;
}

.page-break {
  page-break-after: always;
}

/* Typography */
h1 {
  font-size: 36px;
  font-weight: 700;
  color: #2C2C2C;
  line-height: 1.2;
  margin: 0 0 16px 0;
}

h2 {
  font-size: 24px;
  font-weight: 600;
  color: #2C2C2C;
  line-height: 1.3;
  margin: 24px 0 16px 0;
}

h3 {
  font-size: 18px;
  font-weight: 600;
  color: #2C2C2C;
  line-height: 1.4;
  margin: 16px 0 8px 0;
}

p {
  margin: 0 0 16px 0;
}

/* Badges */
.badge {
  display: inline-block;
  padding: 4px 12px;
  border-radius: 16px;
  font-size: 12px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 8px;
}

.badge-high {
  background: #E07A3E;
  color: white;
}

.badge-medium {
  background: #D4A574;
  color: white;
}

.badge-low {
  background: #E0DDD6;
  color: #6B6B6B;
}

/* Boxes */
.box {
  background: white;
  border: 1px solid #E0DDD6;
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 16px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}

.box-accent-orange {
  border-left: 4px solid #E07A3E;
}

.box-light-bg {
  background: #F7E6D9;
  border: none;
}

/* Keystone Habit - Special Styling */
.keystone-habit {
  background: white;
  border: 3px solid #E07A3E;
  border-radius: 12px;
  padding: 24px;
  margin: 16px 0;
  box-shadow: 0 4px 8px rgba(224, 122, 62, 0.1);
}

.keystone-habit h3 {
  color: #E07A3E;
  margin-top: 0;
}

/* Action Items */
.action-item {
  margin-bottom: 20px;
  padding-bottom: 16px;
  border-bottom: 1px solid #E0DDD6;
}

.action-item:last-child {
  border-bottom: none;
}

.action-title {
  font-size: 16px;
  font-weight: 600;
  color: #2C2C2C;
  margin-bottom: 8px;
}

.action-meta {
  font-size: 12px;
  color: #6B6B6B;
  margin-top: 8px;
}

/* Milestones */
.milestone {
  background: white;
  border: 1px solid #E0DDD6;
  border-radius: 8px;
  padding: 12px 16px;
  margin-bottom: 12px;
}

.milestone-week {
  font-weight: 600;
  color: #E07A3E;
}

/* Tracking Grid */
.dot-grid {
  display: grid;
  grid-template-columns: repeat(10, 1fr);
  gap: 8px;
  margin: 24px 0;
  padding: 20px;
  background: white;
  border-radius: 12px;
}

.dot {
  width: 20px;
  height: 20px;
  border: 2px solid #E0DDD6;
  border-radius: 50%;
}

/* Checkboxes */
.checkbox {
  display: inline-block;
  width: 16px;
  height: 16px;
  border: 2px solid #6B6B6B;
  border-radius: 3px;
  margin-right: 8px;
  vertical-align: middle;
}

/* Icons (using emoji) */
.icon {
  display: inline-block;
  margin-right: 6px;
  font-size: 16px;
}

/* Difficulty Indicators */
.difficulty {
  display: inline-flex;
  gap: 4px;
}

.difficulty-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #2C2C2C;
}

.difficulty-dot-empty {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #E0DDD6;
}

/* Cover Page Specific */
.cover {
  text-align: center;
  padding: 80px 20px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  min-height: 9in; /* Full page height minus margins */
}

.cover h1 {
  margin-bottom: 8px;
}

.cover .subtitle {
  font-size: 18px;
  color: #6B6B6B;
  margin-bottom: 48px;
}

.cover .preview-box {
  background: white;
  border: 1px solid #E0DDD6;
  border-radius: 12px;
  padding: 24px;
  margin: 32px 0;
  text-align: left;
  max-width: 400px;
}

/* Quick Start Special */
.quick-start-item {
  background: white;
  border: 2px solid #E0DDD6;
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 16px;
}

.quick-start-item.high-priority {
  border-color: #E07A3E;
  border-width: 3px;
}

/* Utility Classes */
.text-center { text-align: center; }
.text-small { font-size: 12px; color: #6B6B6B; }
.text-orange { color: #E07A3E; }
.text-gold { color: #D4A574; }
.mb-lg { margin-bottom: 24px; }
.mt-lg { margin-top: 24px; }
```

### Puppeteer Configuration

**PDF Generation Function:**

```typescript
async function generatePDF(htmlContent: string): Promise<Buffer> {
  const browser = await puppeteer.launch({
    args: ['--no-sandbox', '--disable-setuid-sandbox'],
    headless: true
  });
  
  const page = await browser.newPage();
  
  // Set content
  await page.setContent(htmlContent, {
    waitUntil: 'networkidle0'
  });
  
  // Generate PDF
  const pdfBuffer = await page.pdf({
    format: 'letter',
    printBackground: true,
    margin: {
      top: '1in',
      right: '1in',
      bottom: '1in',
      left: '1in'
    },
    preferCSSPageSize: true
  });
  
  await browser.close();
  
  return pdfBuffer;
}
```

**Key Configuration Options:**
- `format: 'letter'` - US Letter size (8.5" × 11")
- `printBackground: true` - CRITICAL for cream background and colors
- `margin: '1in'` - Matches @page CSS
- `preferCSSPageSize: true` - Respects CSS page breaks
- `waitUntil: 'networkidle0'` - Ensures fonts/images loaded

### Data Binding Implementation

**Template Filling Function:**

```typescript
function fillTemplate(
  template: string, 
  data: AnalysisData
): string {
  let filled = template;
  
  // Simple variable replacement
  const simpleVars = {
    user_name: data.userName,
    generated_date: formatDate(data.generatedDate),
    dominant_domain: data.cliftonProfile.dominantDomain,
    // ... all other simple variables
  };
  
  for (const [key, value] of Object.entries(simpleVars)) {
    const regex = new RegExp(`{{${key}}}`, 'g');
    filled = filled.replace(regex, value || '');
  }
  
  // Conditional sections
  filled = filled.replace(
    /{{#if high_goal}}([\s\S]*?){{\/if}}/g,
    data.goals.find(g => g.priority === 'high') ? '$1' : ''
  );
  
  // Loop sections
  filled = filled.replace(
    /{{#each goals}}([\s\S]*?){{\/each}}/g,
    (match, template) => {
      return data.goals.map(goal => 
        fillGoalTemplate(template, goal)
      ).join('\n');
    }
  );
  
  return filled;
}
```

**Goal Template Filling:**

```typescript
function fillGoalTemplate(
  template: string, 
  goal: Goal
): string {
  let filled = template;
  
  // Replace goal-specific variables
  const goalVars = {
    'goal.description': goal.description,
    'goal.deadline': formatDate(goal.deadline),
    'goal.priority': goal.priority,
    'goal.hardness_rating': '⭐'.repeat(goal.hardnessRating),
    // ... etc
  };
  
  for (const [key, value] of Object.entries(goalVars)) {
    const regex = new RegExp(`{{${key}}}`, 'g');
    filled = filled.replace(regex, value);
  }
  
  // Handle nested loops (core actions, milestones, etc.)
  filled = handleCoreActions(filled, goal.coreActions);
  filled = handleMilestones(filled, goal.milestones);
  
  return filled;
}
```

### File Generation Workflow

**Complete PDF Generation Pipeline:**

```typescript
async function generateStrategyPDF(analysisId: string): Promise<string> {
  try {
    // 1. Fetch all data from database
    const analysisData = await fetchAnalysisData(analysisId);
    
    // 2. Load HTML template
    const template = await loadTemplate('base.html');
    
    // 3. Fill template with data
    const filledHTML = fillTemplate(template, analysisData);
    
    // 4. Generate PDF from HTML
    const pdfBuffer = await generatePDF(filledHTML);
    
    // 5. Upload to storage (Supabase Storage)
    const fileName = `strategy_${analysisId}_${Date.now()}.pdf`;
    const fileUrl = await uploadToStorage(fileName, pdfBuffer);
    
    // 6. Update database with PDF URL
    await db.query(
      'UPDATE analyses SET pdf_url = $1, pdf_generated_at = NOW() WHERE id = $2',
      [fileUrl, analysisId]
    );
    
    return fileUrl;
    
  } catch (error) {
    console.error('PDF generation failed:', error);
    throw new Error('PDF_GENERATION_FAILED');
  }
}
```

**Performance Optimization:**
- Cache HTML template (don't reload from disk every time)
- Reuse Puppeteer browser instance for multiple PDFs (connection pooling)
- Generate PDF asynchronously (don't block user while processing)
- Store PDF in Supabase Storage, not database (avoid bloat)

---

## Content Placeholder Reference

### Complete Variable List

**User & Analysis Metadata:**
- `{{user_name}}` - String, from Google Auth
- `{{generated_date}}` - String, formatted date
- `{{analysis_id}}` - UUID
- `{{goal_count}}` - Number (1-3)

**CliftonStrengths Profile:**
- `{{clifton_profile.dominant_domain}}` - String (one of 4 domains)
- `{{clifton_profile.top_ten_themes[]}}` - Array of objects
  - `.name` - String (e.g., "Achiever")
  - `.rank` - Number (1-10)
  - `.domain` - String (calculated from theme mapping)
- `{{clifton_profile.bottom_five_themes[]}}` - Array of objects
  - `.name` - String
  - `.rank` - Number (30-34)

**Synthesizer Outputs:**
- `{{synthesizer.executive_summary}}` - String, 200-800 chars
- `{{synthesizer.clifton_strengths_overview}}` - String, 2-3 paragraphs
- `{{synthesizer.overarching_strategy}}` - String, 2-3 paragraphs
- `{{synthesizer.tracking_guidance}}` - String
- `{{synthesizer.motivational_closing}}` - String
- `{{synthesizer.next_steps[]}}` - Array of strings (3-5 items)
- `{{synthesizer.total_estimated_time_per_week}}` - Number

**Goals (Array - Loop):**
- `{{goals[]}}` - Array (1-3 items)
  - `.priority` - String ("high" | "medium" | "low")
  - `.description` - String
  - `.deadline` - Date
  - `.hardness_rating` - Number (1-5)
  - `.categories[]` - Array of strings
  - `.strategy_overview` - String
  - `.why_this_works` - String
  - `.primary_core_drives[]` - Array of numbers (1-8)
  - `.estimated_time_per_week` - Number
  - `.capacity_warning` - String (optional)
  - `.tracking_method` - String

**Keystone Habit (per goal):**
- `.keystone_habit.title` - String
- `.keystone_habit.description` - String
- `.keystone_habit.frequency` - String
- `.keystone_habit.estimated_time_minutes` - Number
- `.keystone_habit.difficulty_level` - Number (1-3)
- `.keystone_habit.core_drives[]` - Array of numbers
- `.keystone_habit.strengths_used[]` - Array of strings

**Core Actions (per goal, array):**
- `.core_actions[]` - Array (3-5 items)
  - `.title` - String
  - `.description` - String
  - `.frequency` - String
  - `.estimated_time_minutes` - Number
  - `.difficulty_level` - Number (1-3)
  - `.core_drives[]` - Array of numbers
  - `.strengths_used[]` - Array of strings

**Bonus Accelerators (per goal, array):**
- `.bonus_accelerators[]` - Array (2-3 items)
  - Same structure as core actions

**Milestones (per goal, array):**
- `.milestones[]` - Array (3-5 items)
  - `.week` - Number
  - `.description` - String
  - `.success_criteria` - String

**Static Content:**
- Telegram link/QR code
- Waitlist link/QR code
- Core Drive descriptions (CD1-8)
- Domain explanations

---

## Validation & Quality Checks

### Pre-Generation Validation

**Check before PDF generation:**
```typescript
function validatePDFData(data: AnalysisData): boolean {
  // Required fields
  if (!data.userName) throw new Error('Missing user_name');
  if (!data.cliftonProfile.dominantDomain) throw new Error('Missing dominant_domain');
  if (!data.goals || data.goals.length === 0) throw new Error('No goals found');
  
  // Goal validation
  data.goals.forEach(goal => {
    if (!goal.keystoneHabit) throw new Error('Missing keystone habit');
    if (goal.coreActions.length < 3) throw new Error('Insufficient core actions');
    if (goal.milestones.length < 3) throw new Error('Insufficient milestones');
  });
  
  // High-priority goal exists
  const hasHighPriority = data.goals.some(g => g.priority === 'high');
  if (!hasHighPriority) throw new Error('No high-priority goal');
  
  return true;
}
```

### Post-Generation Quality Checks

**Verify PDF quality:**
- File size: 2-5 MB (not too large, indicates proper compression)
- Page count: Approximately 5-7 pages per goal + fixed pages (15-20 total for 3 goals)
- Readable text: Test on different PDF viewers
- Colors rendered: Cream background, orange accents visible
- Print test: Actually print one copy to verify layout

### Error Handling

**Common PDF Generation Errors:**

| Error | Cause | Solution |
|-------|-------|----------|
| White background | `printBackground: false` | Set to `true` in Puppeteer |
| Missing fonts | Web fonts not loaded | Wait for `networkidle0` |
| Page breaks wrong | CSS page-break ignored | Add `.page-break` class |
| Content cut off | Margins too large | Adjust @page margins |
| File too large | High-res images | Compress/optimize images |
| Generation timeout | Complex HTML | Simplify template, optimize |

---

## Future Enhancements (Post-MVP)

**Not in v1, but consider for future versions:**

1. **Dynamic page numbering** - "Page X of Y" footer
2. **Table of contents** - Clickable links to sections (PDF bookmarks)
3. **QR codes** - Auto-generate for Telegram/waitlist links
4. **Custom branding** - User can customize colors (white-label)
5. **Multi-language** - Generate PDFs in Spanish, Portuguese, etc.
6. **Interactive PDF** - Clickable checkboxes (Adobe PDF forms)
7. **Print optimization** - Separate "screen" and "print" versions
8. **A/B test layouts** - Test different designs for conversion
9. **PDF analytics** - Track which sections users spend most time on
10. **Regeneration** - Allow users to regenerate with updated goals

---

## Appendix: Example Filled Content

### Sample Executive Summary (Filled)

```
Your CliftonStrengths profile reveals a Strategic Thinking dominant domain, 
with Achiever as your #1 theme. This combination makes you particularly 
well-suited for ambitious goals that require both careful planning and 
consistent execution. The three goals you've set - running a 5K, learning 
Spanish, and building a side project - all leverage your natural planning 
abilities while feeding your need for measurable progress. With an estimated 
12 hours per week required, these goals are ambitious but achievable given 
your current schedule.

Your top 10 themes cluster heavily in Strategic Thinking (5 themes) and 
Executing (3 themes), with Achiever leading the pack. This means you thrive 
on intellectual challenges paired with tangible completion. Your Achiever 
drive creates natural momentum through daily progress tracking, while your 
Strategic, Futuristic, and Ideation themes help you see the path forward 
clearly. We've designed your strategy to leverage these strengths: each goal 
starts with a micro-commitment (5-minute run, 10-minute Spanish session, 
30-minute coding block) that satisfies your Achiever's need for daily wins, 
then scales gradually using your Strategic mind's preference for measured 
progression.

The strategy across all three goals follows a consistent pattern that mirrors 
your natural working style: planning prevents decision fatigue, daily habits 
build momentum, and visible progress tracking keeps you motivated. This isn't 
generic advice - it's specifically designed for how YOUR brain works best.
```

### Sample Goal Strategy Page (Filled - High Priority)

```
┌─────────────────────────────────────────────────────┐
│  🎯 HIGH PRIORITY                                   │
│                                                     │
│  Run a 5K in under 30 minutes                      │
│  Deadline: June 30, 2026 • Difficulty: ⭐⭐⭐⭐   │
│  Categories: Health & Fitness, Rest & Recovery      │
│  ────────────────────────────────────────────────   │
│                                                     │
│  STRATEGY OVERVIEW                                  │
│  Build a consistent 5K running habit by starting   │
│  with micro-runs and leveraging your Achiever      │
│  drive for daily completion streaks.               │
│                                                     │
│  💡 WHY THIS WORKS FOR YOU                         │
│  Your Achiever theme thrives on daily progress and │
│  completion, making a streak-based approach highly │
│  motivating. Your Strategic thinking helps you     │
│  plan around schedule conflicts, while your        │
│  Discipline strength ensures consistency even on   │
│  tough days. Starting with just 5 minutes removes  │
│  the intimidation factor and lets you build        │
│  momentum before scaling up.                       │
│                                                     │
│  ESTIMATED TIME COMMITMENT                          │
│  3.5 hours per week                                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

**Document Status:** Complete - Ready for Implementation  
**Last Updated:** February 14, 2026  
**Next Document:** API Specification  
**Owner:** Igor (Founder, PM, Designer)

---

*This specification provides complete design and implementation guidance for generating personalized PDF strategy documents. Every section, style, placeholder, and technical detail is defined to enable Claude Code to build the PDF generation system without ambiguity.*
