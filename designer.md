---
alwaysApply: true
---

# 🎨 DESIGN SYSTEM INSTRUCTIONS (Mono-Lightweight Architecture)

This document defines the **design philosophy**, **visual system**, and **Tailwind conventions** used across the project.  
It serves as a single source of truth for all visual, layout, and UI design decisions.  
Every component, widget, and page **must** follow these rules for consistency and scalability.

---

## 🧭 1. DESIGN CONTEXT SECTION (Fill before starting a project)

| Category | Description |
|-----------|-------------|
| **Project Name** |  |
| **Target Users** | (Who are they? Students? Soldiers? Researchers? Early adopters?) |
| **User Needs / Problems to Solve** | (Be specific: “Quickly visualize knowledge relationships,” “Reduce onboarding friction,” etc.) |
| **Intended Mood / Tone** | (Examples: Calm, minimal, techy, trustworthy, energetic…) |
| **Core Design Goals** | (Examples: fast perception, minimal clicks, clarity, warmth, elegance…) |
| **Brand Keywords** | (Examples: glassmorphic, iOS-like, oceanic, geometric, etc.) |
| **Primary Device Focus** | (Desktop / Mobile / Both) |
| **Accessibility Goals** | (AA or AAA? Screen-reader optimized? Colorblind-safe palette?) |

> 💡 This section is not just documentation — it’s the foundation of all spacing, typography, and color decisions.

---

## ⚙️ 2. DESIGN SYSTEM PRINCIPLES

1. **Tailwind-First**  
   - Prefer Tailwind utility classes over custom CSS.  
   - If a design pattern recurs, create a `@layer components` class or React wrapper, **not** inline repetition.
   - Shadcn/UI serves as the base layer for accessibility and interaction.

2. **Systematic Logic over Intuition**  
   Every visual parameter (spacing, font, radius, color) must follow a *defined logic system* instead of ad-hoc values.

3. **Consistent Hierarchy**  
   - Typography, color, and spacing should form predictable scales.  
   - Avoid arbitrary pixel values — derive from the scale table below.

4. **Deliberate Visual Tone**  
   - Always match the intended mood and brand keywords defined above.  
   - Use fewer effects (blur, gradients, shadows) — when used, they must have *semantic purpose* (e.g., depth hierarchy, focus, affordance).

---

## 🧩 3. LOGICAL DESIGN SYSTEM

### 🧱 Spacing Scale

| Step | Tailwind Token | Example Usage |
|------|----------------|----------------|
| 4px  | `p-1`, `m-1` | Tight inner padding |
| 8px  | `p-2`, `gap-2` | Component inner gap |
| 12px | `p-3` | Small block spacing |
| 16px | `p-4`, `gap-4` | Standard block spacing |
| 24px | `p-6` | Section padding |
| 32px | `p-8` | Page margin / layout boundary |
| 48px | `p-12` | Desktop section or card grouping |

> 🔁 Always prefer relative spacing (`gap-*`, `px-*`, `py-*`) over fixed height unless layout requires it.

---

### 🔤 Typography System

| Element | Class | Weight | Notes |
|----------|--------|--------|------|
| Display / Hero | `text-5xl font-bold` | 700 | Page titles, hero areas |
| Heading 1 | `text-3xl font-semibold` | 600 | Section titles |
| Heading 2 | `text-xl font-semibold` | 600 | Subsections, card headers |
| Body | `text-base font-normal` | 400 | Paragraph text |
| Small / Label | `text-sm text-muted-foreground` | 400 | Secondary info, helper text |

- Use the **Pretendard** or **Inter** font (depending on locale).  
- Font sizes should always be defined in Tailwind tokens (`text-sm`, `text-base`, etc.), not in raw `px`.

---

### 🧭 Radius & Depth System

| Context | Class | Description |
|----------|--------|-------------|
| Small elements | `rounded-md` | Buttons, inputs |
| Medium components | `rounded-xl` | Cards, modals |
| Major containers | `rounded-2xl` | Panels, boards |
| Floating / glassmorphic layers | `rounded-3xl shadow-lg backdrop-blur-md` | Use sparingly for premium feel |

> Never invent new radius values; pick from the system above.

---

### 🌈 Color System

Follow Tailwind’s palette with project-specific tokens in `tailwind.config.ts` under `theme.extend.colors`.

| Role | Token | Example |
|------|--------|---------|
| Primary | `text-primary`, `bg-primary`, `border-primary` | Core brand accent |
| Secondary | `text-secondary`, `bg-secondary` | Subtle highlights |
| Muted | `text-muted-foreground`, `bg-muted` | Backgrounds, neutral areas |
| Destructive | `text-destructive`, `bg-destructive` | Errors, warnings |
| Success | `text-success`, `bg-success` | Positive actions |
| Background | `bg-background` | Page base |
| Card | `bg-card`, `border-card` | Surfaces |

Always maintain **WCAG contrast ≥ 4.5** for text on backgrounds.

---

## 🧠 4. DESIGN TOKEN RULES

All design parameters should map to a token, even if stored inline via Tailwind config.

```ts
// tailwind.config.ts excerpt
theme: {
  extend: {
    colors: {
      brand: {
        DEFAULT: "#2563eb",
        foreground: "#ffffff",
      },
      success: "#10b981",
      destructive: "#ef4444",
    },
    borderRadius: {
      md: "0.375rem",
      xl: "0.75rem",
      "2xl": "1rem",
      "3xl": "1.5rem",
    },
  },
},
