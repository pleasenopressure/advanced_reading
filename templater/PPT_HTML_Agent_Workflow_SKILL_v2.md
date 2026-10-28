---
description: |
  Advanced AI presentation generation workflow. Use this skill for
  creating, redesigning, or improving PowerPoint presentations. It uses
  HTML/CSS as the presentation intermediate representation, a
  component-based design system, multi-stage generation agents, visual
  evaluation loops, and editable PPTX compilation.
license: Proprietary. LICENSE.txt has complete terms
name: ppt-html-agent-workflow
---

# PPT HTML Agent Workflow

## Overview

This skill treats presentation generation as a compilation pipeline.

The source of truth is not PPTX. The source of truth is a structured
presentation model rendered through HTML/CSS.

    User Requirement
            ↓
    Presentation Planner Agent
            ↓
    Content Architecture
            ↓
    Design System Extraction
            ↓
    HTML Slide Generation
            ↓
    Browser Rendering
            ↓
    Visual Critic Agent
            ↓
    Layout Repair Agent
            ↓
    PPTX Compiler
            ↓
    PPTX Validation
            ↓
    Final Editable Deck

The goal is not only generating slides, but producing professional,
editable, visually consistent presentations.

------------------------------------------------------------------------

# Agent Architecture

## 1. Requirement Analyzer Agent

Responsibilities:

-   Understand presentation purpose
-   Identify audience
-   Determine tone:
    -   executive
    -   academic
    -   technical
    -   marketing
    -   consulting
-   Determine slide count
-   Identify required assets
-   Detect whether a reference template exists

Output:

``` json
{
  "audience": "executives",
  "style": "professional consulting",
  "slides": 12,
  "editable_required": true,
  "template_available": true
}
```

------------------------------------------------------------------------

# 2. Template Intelligence Agent

When a PPT/POTX template exists:

Do not directly copy slides.

Extract:

## Visual Tokens

-   Primary colors
-   Secondary colors
-   Background styles
-   Typography hierarchy
-   Corner radius
-   Shadow style
-   Icon style

## Layout Patterns

Convert existing slides into reusable layouts:

    Cover
    Section Divider
    Problem Statement
    Two Column Explanation
    Three Card Framework
    Timeline
    Architecture Diagram
    Comparison Table
    Dashboard
    Conclusion

Output:

    design-system/
     ├── colors.css
     ├── typography.css
     ├── components.css
     └── layouts/

The template becomes a design system.

------------------------------------------------------------------------

# 3. Story Architecture Agent

Before generating slides:

Create a narrative structure.

Avoid:

-   random information dumping
-   repeating layouts
-   text-heavy pages

Every slide must define:

``` json
{
 "purpose": "explain AI adoption roadmap",
 "message": "three implementation phases",
 "visual": "timeline",
 "layout": "timeline-v1"
}
```

A slide should answer:

"What is the single message this page communicates?"

------------------------------------------------------------------------

# 4. Component-Based HTML Generation

Generate slides using reusable HTML components.

Example:

    components/

    TitleBlock
    MetricCard
    TimelineNode
    ComparisonColumn
    ArchitectureLayer
    ImagePanel
    ChartContainer
    QuoteBlock

Example:

``` html
<section class="slide timeline">
  <TitleBlock />
  <TimelineNode />
  <TimelineNode />
</section>
```

Never allow uncontrolled free-form HTML.

------------------------------------------------------------------------

# 5. Design Constraint Engine

The AI must follow:

## Typography

Recommended:

    Title:
    36-44px

    Section:
    24-30px

    Body:
    16-20px

    Caption:
    12-14px

## Spacing

Use a spacing scale:

    8
    16
    24
    32
    48
    64

Do not create random margins.

## Color

Rules:

-   One dominant color
-   One supporting color
-   One accent color
-   Maintain contrast

Never generate:

-   white text on light background
-   gray text with insufficient contrast
-   random gradients

------------------------------------------------------------------------

# 6. Rendering Pipeline

Each slide is rendered independently.

Structure:

    slides/
     |
     ├── slide-01.html
     ├── slide-02.html
     |
     ├── style.css
     |
     └── assets/

Rendering:

    HTML
     ↓
    Chromium Screenshot
     ↓
    Visual Evaluation

The screenshot is a QA reference, not the final PPT.

------------------------------------------------------------------------

# 7. Visual Critic Agent

The critic checks:

## Layout

-   Alignment
-   Balance
-   Margin
-   Element spacing
-   Grid consistency

## Typography

-   Overflow
-   Font size
-   Line breaks
-   Hierarchy

## Design

-   Contrast
-   Visual identity
-   Repetition
-   Empty space

## AI Generation Problems

Detect:

-   generic card layouts
-   excessive rounded boxes
-   decorative lines
-   meaningless icons
-   text-heavy slides

Output:

``` json
{
 "slide": 5,
 "issues": [
   {
    "type":"contrast",
    "severity":"high",
    "fix":"change background or text color"
   }
 ]
}
```

------------------------------------------------------------------------

# 8. Repair Agent

Never manually patch PPTX.

Repair the source:

    HTML/CSS
       ↓
    Regenerate
       ↓
    Render
       ↓
    Compare

Priority:

1.  Broken layout
2.  Text overflow
3.  Contrast problems
4.  Alignment
5.  Visual polish

------------------------------------------------------------------------

# 9. PPTX Compiler Agent

Convert HTML components into editable PowerPoint objects.

Mapping:

  HTML    PPTX
  ------- --------------
  text    textbox
  div     shape
  svg     vector image
  img     image
  table   table
  chart   native chart

Avoid screenshot-based PPT generation.

------------------------------------------------------------------------

# 10. Final Validation Agent

Perform three checks.

## Content Validation

Check:

-   missing information
-   wrong order
-   incorrect text
-   placeholders

## Structural Validation

Check:

-   PPTX opens
-   XML integrity
-   assets exist

## Visual Validation

Compare:

    HTML Render
          VS
    PPTX Render

Measure:

-   position deviation
-   missing objects
-   font changes
-   wrapping changes

------------------------------------------------------------------------

# Continuous Improvement Memory

After successful generation:

Store reusable knowledge:

    successful-layouts/

    bad-patterns/

    industry-style-guidelines/

    brand-rules/

Future decks should reuse validated patterns.

------------------------------------------------------------------------

# Operating Principles

## Source of Truth

HTML/CSS \> PPTX

## Quality Loop

Generate → Render → Critique → Repair → Compile → Validate

## Design Philosophy

AI decides:

-   story
-   structure
-   information hierarchy

Design system decides:

-   colors
-   spacing
-   typography
-   components

Compiler decides:

-   PPT implementation

This separation improves reliability, consistency, and editability.
