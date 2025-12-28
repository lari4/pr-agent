# PR-Agent Pipeline Documentation

This document describes all agent workflows (pipelines) in PR-Agent, detailing the sequence of operations, data flow between stages, and which prompts are used at each step.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [PR Review Pipeline](#1-pr-review-pipeline)
3. [PR Description Pipeline](#2-pr-description-pipeline)
4. [Code Suggestions Pipeline](#3-code-suggestions-pipeline)
5. [Add Docs Pipeline](#4-add-docs-pipeline)
6. [Questions Pipeline](#5-questions-pipeline)
7. [Changelog Update Pipeline](#6-changelog-update-pipeline)
8. [Labels Generation Pipeline](#7-labels-generation-pipeline)
9. [Help Pipeline](#8-help-pipeline)
10. [Common Components](#common-components)

---

## Architecture Overview

### High-Level Request Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              REQUEST FLOW                                    │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌──────────┐     ┌───────────────────┐     ┌──────────────────┐
  │  Webhook │────▶│ PRAgent.handle_   │────▶│  Route to Tool   │
  │  or CLI  │     │ request()         │     │  (Review, etc.)  │
  └──────────┘     └───────────────────┘     └────────┬─────────┘
                                                      │
       ┌──────────────────────────────────────────────┼──────────────────────┐
       │                                              ▼                      │
       │  ┌─────────────────────────────────────────────────────────────┐   │
       │  │                    TOOL EXECUTION                            │   │
       │  │                                                              │   │
       │  │  1. Initialize git provider, language, token handler         │   │
       │  │                         │                                    │   │
       │  │                         ▼                                    │   │
       │  │  2. Collect PR data (diff, metadata, comments)               │   │
       │  │                         │                                    │   │
       │  │                         ▼                                    │   │
       │  │  3. Prepare prompts (Jinja2 template rendering)              │   │
       │  │                         │                                    │   │
       │  │                         ▼                                    │   │
       │  │  4. Call retry_with_fallback_models()                        │   │
       │  │     ├── Try primary model                                    │   │
       │  │     └── On failure: try fallback models                      │   │
       │  │                         │                                    │   │
       │  │                         ▼                                    │   │
       │  │  5. Post-process response (YAML parse, format, validate)     │   │
       │  │                         │                                    │   │
       │  │                         ▼                                    │   │
       │  │  6. [Optional] Self-reflection cycle                         │   │
       │  │                         │                                    │   │
       │  │                         ▼                                    │   │
       │  │  7. Publish output via git provider API                      │   │
       │  │                                                              │   │
       │  └─────────────────────────────────────────────────────────────┘   │
       │                                                                      │
       └──────────────────────────────────────────────────────────────────────┘
```

### Component Interactions

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                           COMPONENT ARCHITECTURE                                │
└────────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────┐          ┌─────────────┐          ┌─────────────────┐
  │   CLI /     │          │   Tool      │          │   AI Handler    │
  │   Webhook   │─────────▶│   Class     │─────────▶│   (LiteLLM)     │
  └─────────────┘          └──────┬──────┘          └────────┬────────┘
                                  │                          │
                                  │                          │
       ┌──────────────────────────┼──────────────────────────┼───────────────┐
       │                          │                          │               │
       ▼                          ▼                          ▼               │
  ┌─────────────┐          ┌─────────────┐          ┌─────────────────┐     │
  │  Git        │          │   Token     │          │   Model         │     │
  │  Provider   │◀────────▶│   Handler   │          │   Response      │     │
  │  (GitHub,   │          │             │          │                 │     │
  │   GitLab)   │          └─────────────┘          └─────────────────┘     │
  └─────────────┘                                                            │
       │                                                                      │
       │                   ┌─────────────────────────────────────────────────┐
       │                   │              PROMPT TEMPLATES                   │
       │                   │                                                 │
       │                   │  ┌─────────────────────────────────────────┐   │
       │                   │  │  pr_reviewer_prompts.toml               │   │
       │                   │  │  pr_description_prompts.toml            │   │
       ▼                   │  │  pr_code_suggestions_prompts.toml       │   │
  ┌─────────────┐          │  │  pr_add_docs.toml                       │   │
  │  Output     │          │  │  pr_questions_prompts.toml              │   │
  │  (Comment,  │          │  │  pr_update_changelog_prompts.toml       │   │
  │   Labels,   │          │  │  pr_custom_labels.toml                  │   │
  │   Files)    │          │  │  pr_help_prompts.toml                   │   │
  └─────────────┘          │  └─────────────────────────────────────────┘   │
                           └─────────────────────────────────────────────────┘
```

---

## 1. PR Review Pipeline

**Command:** `review`, `review_pr`
**Class:** `PRReviewer` (`pr_agent/tools/pr_reviewer.py`)
**Prompt File:** `pr_agent/settings/pr_reviewer_prompts.toml`

### Pipeline Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           PR REVIEW PIPELINE                                     │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 1: INITIALIZATION & DATA COLLECTION                                    │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
  ┌───────────────────────────────────┼───────────────────────────────────────────┐
  │                                   ▼                                           │
  │   ┌─────────────────────────────────────────────────────────────────────┐    │
  │   │  • Parse command arguments (incremental flag, etc.)                  │    │
  │   │  • Initialize git provider with PR context                           │    │
  │   │  • Detect main programming language from PR files                    │    │
  │   │  • Load PR description                                               │    │
  │   │  • Fetch related tickets (if enabled)                                │    │
  │   │  • Initialize token handler                                          │    │
  │   └─────────────────────────────────────────────────────────────────────┘    │
  │                                   │                                           │
  │                                   ▼                                           │
  │   ┌─────────────────────────────────────────────────────────────────────┐    │
  │   │                    COLLECTED DATA                                    │    │
  │   │  ─────────────────────────────────────────────────────────────────  │    │
  │   │  • title         : PR title                                          │    │
  │   │  • branch        : Target branch name                                │    │
  │   │  • description   : PR description text                               │    │
  │   │  • language      : Main programming language                         │    │
  │   │  • diff          : Code changes (+ and - lines)                      │    │
  │   │  • commit_messages: All commit messages                              │    │
  │   │  • related_tickets: Linked issues/tickets                            │    │
  │   │  • date          : Current date                                      │    │
  │   └─────────────────────────────────────────────────────────────────────┘    │
  └───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 2: DIFF GENERATION                                                      │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
  ┌───────────────────────────────────┼───────────────────────────────────────────┐
  │                                   ▼                                           │
  │   ┌─────────────────────────────────────────────────────────────────────┐    │
  │   │  get_pr_diff()                                                       │    │
  │   │  ─────────────────────────────────────────────────────────────────  │    │
  │   │  1. Extract diff files from git provider                             │    │
  │   │  2. Sort files by primary language                                   │    │
  │   │  3. Generate extended diffs with context lines                       │    │
  │   │  4. Apply token budgeting:                                           │    │
  │   │     • If under limit → return full diff                              │    │
  │   │     • If over limit → prune less important files                     │    │
  │   │  5. Add line numbers to hunks for precise references                 │    │
  │   └─────────────────────────────────────────────────────────────────────┘    │
  │                                   │                                           │
  │                                   ▼                                           │
  │            ┌───────────────────────────────────────────┐                      │
  │            │         STRUCTURED DIFF FORMAT            │                      │
  │            │  ─────────────────────────────────────── │                      │
  │            │  ## File: 'src/file.py'                   │                      │
  │            │  @@ ... @@ def func():                    │                      │
  │            │  __new hunk__                             │                      │
  │            │  11  unchanged line                       │                      │
  │            │  12 +new code added                       │                      │
  │            │  __old hunk__                             │                      │
  │            │  -old code removed                        │                      │
  │            └───────────────────────────────────────────┘                      │
  └───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 3: PROMPT PREPARATION                                                   │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
  ┌───────────────────────────────────┼───────────────────────────────────────────┐
  │                                   ▼                                           │
  │   ┌─────────────────────────────────────────────────────────────────────┐    │
  │   │  Jinja2 Template Rendering                                           │    │
  │   │  ─────────────────────────────────────────────────────────────────  │    │
  │   │                                                                      │    │
  │   │  Variables dict:                                                     │    │
  │   │  ┌────────────────────────────────────────────────────────────────┐ │    │
  │   │  │ title, branch, description, language                           │ │    │
  │   │  │ diff, date, extra_instructions                                 │ │    │
  │   │  │ related_tickets, commit_messages_str                           │ │    │
  │   │  │ require_score, require_tests, require_security_review          │ │    │
  │   │  │ require_can_be_split_review, require_todo_scan                 │ │    │
  │   │  │ num_max_findings, question_str, answer_str                     │ │    │
  │   │  └────────────────────────────────────────────────────────────────┘ │    │
  │   │                              │                                       │    │
  │   │                              ▼                                       │    │
  │   │  ┌────────────────────────────────────────────────────────────────┐ │    │
  │   │  │        pr_reviewer_prompts.toml                                │ │    │
  │   │  │  ──────────────────────────────────────────────────────────── │ │    │
  │   │  │  [pr_review_prompt]                                            │ │    │
  │   │  │  system = "You are PR-Reviewer..."                             │ │    │
  │   │  │  user = "PR Info: Title: '{{title}}'..."                       │ │    │
  │   │  └────────────────────────────────────────────────────────────────┘ │    │
  │   │                              │                                       │    │
  │   │                              ▼                                       │    │
  │   │              RENDERED SYSTEM + USER PROMPTS                          │    │
  │   └─────────────────────────────────────────────────────────────────────┘    │
  └───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 4: AI CALL                                                              │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
  ┌───────────────────────────────────┼───────────────────────────────────────────┐
  │                                   ▼                                           │
  │   ┌─────────────────────────────────────────────────────────────────────┐    │
  │   │  retry_with_fallback_models()                                        │    │
  │   │  ─────────────────────────────────────────────────────────────────  │    │
  │   │                                                                      │    │
  │   │   ┌───────────────┐    Fail    ┌───────────────┐    Fail    ...     │    │
  │   │   │ Primary Model │───────────▶│ Fallback #1   │───────────▶        │    │
  │   │   │ (e.g., GPT-4) │            │               │                    │    │
  │   │   └───────┬───────┘            └───────────────┘                    │    │
  │   │           │ Success                                                  │    │
  │   │           ▼                                                          │    │
  │   │   ┌───────────────────────────────────────────────────────────────┐ │    │
  │   │   │  ai_handler.chat_completion()                                 │ │    │
  │   │   │  • Model: configured model name                               │ │    │
  │   │   │  • Temperature: 0.2 (default)                                 │ │    │
  │   │   │  • System prompt: rendered system prompt                      │ │    │
  │   │   │  • User prompt: rendered user prompt                          │ │    │
  │   │   │  → Returns: (response_text, finish_reason)                    │ │    │
  │   │   └───────────────────────────────────────────────────────────────┘ │    │
  │   └─────────────────────────────────────────────────────────────────────┘    │
  │                                   │                                           │
  │                                   ▼                                           │
  │            ┌───────────────────────────────────────────┐                      │
  │            │         AI RESPONSE (YAML)                │                      │
  │            │  ─────────────────────────────────────── │                      │
  │            │  review:                                  │                      │
  │            │    estimated_effort_to_review_[1-5]: 3    │                      │
  │            │    score: 85                              │                      │
  │            │    relevant_tests: "No"                   │                      │
  │            │    key_issues_to_review:                  │                      │
  │            │      - relevant_file: "src/file.py"       │                      │
  │            │        issue_header: "Possible Bug"       │                      │
  │            │        issue_content: "..."               │                      │
  │            │        start_line: 42                     │                      │
  │            │        end_line: 45                       │                      │
  │            │    security_concerns: "No"                │                      │
  │            └───────────────────────────────────────────┘                      │
  └───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 5: POST-PROCESSING                                                      │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
  ┌───────────────────────────────────┼───────────────────────────────────────────┐
  │                                   ▼                                           │
  │   ┌─────────────────────────────────────────────────────────────────────┐    │
  │   │  1. Parse YAML response with load_yaml()                             │    │
  │   │  2. Fix known YAML parsing issues                                    │    │
  │   │  3. Reorder dict (key_issues_to_review at end)                       │    │
  │   │  4. Convert to Markdown (convert_to_markdown_v2())                   │    │
  │   │  5. Add help text if enabled                                         │    │
  │   │  6. Extract labels (effort, security)                                │    │
  │   └─────────────────────────────────────────────────────────────────────┘    │
  └───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 6: PUBLISHING                                                           │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
  ┌───────────────────────────────────┼───────────────────────────────────────────┐
  │                                   ▼                                           │
  │   ┌─────────────────────────────────────────────────────────────────────┐    │
  │   │  Publishing Modes:                                                   │    │
  │   │  ─────────────────────────────────────────────────────────────────  │    │
  │   │                                                                      │    │
  │   │  ┌──────────────────────┐    ┌──────────────────────┐               │    │
  │   │  │ Persistent Comment   │    │ Regular Comment      │               │    │
  │   │  │ (default)            │    │                      │               │    │
  │   │  │                      │    │                      │               │    │
  │   │  │ Updates same comment │    │ Posts new comment    │               │    │
  │   │  │ on each review       │    │ each time            │               │    │
  │   │  └──────────────────────┘    └──────────────────────┘               │    │
  │   │                                                                      │    │
  │   │  Additional actions:                                                 │    │
  │   │  • Publish labels (merge with user labels)                           │    │
  │   │  • Remove temporary "Preparing review..." comment                    │    │
  │   │  • Handle incremental reviews (link to previous)                     │    │
  │   └─────────────────────────────────────────────────────────────────────┘    │
  └───────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow Summary

| Stage | Input | Output | Prompt Used |
|-------|-------|--------|-------------|
| Data Collection | PR URL/context | Metadata, diff files | - |
| Diff Generation | Raw diff files | Formatted diff with line numbers | - |
| Prompt Preparation | Metadata + diff | Rendered prompts | `pr_reviewer_prompts.toml` |
| AI Call | System + User prompts | YAML response | - |
| Post-Processing | YAML response | Markdown text + labels | - |
| Publishing | Markdown + labels | PR comment + labels | - |

---

## 2. PR Description Pipeline

**Command:** `describe`, `describe_pr`
**Class:** `PRDescription` (`pr_agent/tools/pr_description.py`)
**Prompt File:** `pr_agent/settings/pr_description_prompts.toml`

### Pipeline Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         PR DESCRIPTION PIPELINE                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 1: INITIALIZATION & DATA COLLECTION                                    │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  • Initialize git provider                                                    │
  │  • Detect main language                                                       │
  │  • Load original PR description                                               │
  │  • Extract related tickets (if enabled)                                       │
  │  • Get semantic file types (if enabled)                                       │
  │  • Initialize token handler                                                   │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 2: DIFF GENERATION                                                      │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  get_pr_diff() OR get_pr_diff_multiple_patchs()                               │
  │  ─────────────────────────────────────────────────────────────────────────── │
  │                                                                               │
  │  For large PRs (large_pr_handling enabled):                                   │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │  Split into multiple patches                                             │ │
  │  │  Each patch processed separately                                         │ │
  │  │  Results combined at end                                                 │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 3: PROMPT PREPARATION                                                   │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Variables dict:                                                              │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │ title (original), description (original), branch                         │ │
  │  │ commit_messages_str, diff, language                                      │ │
  │  │ enable_custom_labels, custom_labels_class                                │ │
  │  │ enable_semantic_files_types, include_file_summary_changes                │ │
  │  │ enable_pr_diagram, related_tickets                                       │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  │                              │                                                │
  │                              ▼                                                │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │        pr_description_prompts.toml                                       │ │
  │  │  ───────────────────────────────────────────────────────────────────── │ │
  │  │  [pr_description_prompt]                                                 │ │
  │  │  system = "You are PR-Reviewer..."                                       │ │
  │  │  user = "PR Info: Previous title: '{{title}}'..."                        │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 4: AI CALL (WEAK Model - faster/cheaper)                                │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  retry_with_fallback_models() with WEAK model type                            │
  │                                                                               │
  │  For multiple patches:                                                        │
  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐                     │
  │  │   Patch 1     │  │   Patch 2     │  │   Patch N     │                     │
  │  │   AI Call     │  │   AI Call     │  │   AI Call     │  (parallel if       │
  │  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘   enabled)          │
  │          │                  │                  │                              │
  │          └──────────────────┼──────────────────┘                              │
  │                             ▼                                                 │
  │                    COMBINE RESULTS                                            │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
           ┌───────────────────────────────────────────┐
           │         AI RESPONSE (YAML)                │
           │  ─────────────────────────────────────── │
           │  type:                                    │
           │    - Bug fix                              │
           │    - Enhancement                          │
           │  description: |                           │
           │    - Added new feature X                  │
           │    - Fixed bug in Y                       │
           │  title: |                                 │
           │    Add feature X and fix bug Y            │
           │  changes_diagram: |                       │
           │    ```mermaid                             │
           │    flowchart LR...                        │
           │    ```                                    │
           │  pr_files:                                │
           │    - filename: "src/file.py"              │
           │      changes_title: "Add new handler"     │
           │      label: "enhancement"                 │
           └───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 5: POST-PROCESSING                                                      │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  1. Parse AI response into components (title, description, files)             │
  │  2. If semantic file types: add icons/classification to files                 │
  │  3. Generate PR labels from description if enabled                            │
  │  4. Format for publishing                                                     │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 6: PUBLISHING                                                           │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Two modes:                                                                   │
  │  ┌─────────────────────────────┐  ┌─────────────────────────────┐            │
  │  │ Direct PR Update            │  │ Comment Mode                │            │
  │  │ ─────────────────────────── │  │ ─────────────────────────── │            │
  │  │ • Updates PR title          │  │ • Posts as comment          │            │
  │  │ • Updates PR description    │  │ • Can be persistent         │            │
  │  │ • Posts update confirmation │  │   or new each time          │            │
  │  └─────────────────────────────┘  └─────────────────────────────┘            │
  │                                                                               │
  │  Additional:                                                                  │
  │  • Publish labels (merge with existing)                                       │
  │  • Inline file summaries (optional)                                           │
  └───────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Code Suggestions Pipeline

**Command:** `improve`, `improve_code`
**Class:** `PRCodeSuggestions` (`pr_agent/tools/pr_code_suggestions.py`)
**Prompt Files:**
- `pr_agent/settings/code_suggestions/pr_code_suggestions_prompts.toml`
- `pr_agent/settings/code_suggestions/pr_code_suggestions_reflect_prompts.toml`

### Pipeline Diagram (with Self-Reflection)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      CODE SUGGESTIONS PIPELINE                                   │
│                    (with Self-Reflection Feature)                                │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 1: INITIALIZATION                                                       │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  • Initialize git provider, language detector                                 │
  │  • Load PR description with changes walkthrough                               │
  │  • Add AI metadata if enabled                                                 │
  │  • Determine prompt type (decoupled vs non-decoupled hunks)                   │
  │  • Configure num_code_suggestions per chunk                                   │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 2: MULTI-DIFF GENERATION                                                │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  get_pr_multi_diffs()                                                         │
  │  ─────────────────────────────────────────────────────────────────────────── │
  │                                                                               │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │  1. Extract all diff files                                               │ │
  │  │  2. Sort by programming language                                         │ │
  │  │  3. If decoupled hunks enabled:                                          │ │
  │  │     • Generate diffs with __new hunk__ and __old hunk__ sections         │ │
  │  │     • Add line numbers for each hunk                                     │ │
  │  │  4. Split into multiple patches based on token limits                    │ │
  │  │  5. Return list of patches                                               │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  │                              │                                                │
  │                              ▼                                                │
  │         ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                      │
  │         │  Patch 1    │ │  Patch 2    │ │  Patch N    │                      │
  │         └─────────────┘ └─────────────┘ └─────────────┘                      │
  └───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 3: PARALLEL AI CALLS (Initial Suggestions)                              │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │                                                                               │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                           │
  │  │   Patch 1   │  │   Patch 2   │  │   Patch N   │                           │
  │  │   ───────   │  │   ───────   │  │   ───────   │                           │
  │  │  Variables: │  │  Variables: │  │  Variables: │                           │
  │  │  • diff     │  │  • diff     │  │  • diff     │                           │
  │  │  • title    │  │  • title    │  │  • title    │                           │
  │  │  • date     │  │  • date     │  │  • date     │                           │
  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                           │
  │         │                │                │                                   │
  │         ▼                ▼                ▼                                   │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │        pr_code_suggestions_prompts.toml                                  │ │
  │  │  ───────────────────────────────────────────────────────────────────── │ │
  │  │  [pr_code_suggestions_prompt]                                            │ │
  │  │  system = "You are PR-Reviewer, an AI specializing..."                   │ │
  │  │  user = "The PR Diff: ..."                                               │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  │         │                │                │                                   │
  │         ▼                ▼                ▼                                   │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                           │
  │  │ Suggestions │  │ Suggestions │  │ Suggestions │                           │
  │  │  List 1     │  │  List 2     │  │  List N     │                           │
  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                           │
  │         │                │                │                                   │
  │         └────────────────┼────────────────┘                                   │
  │                          ▼                                                    │
  │                 AGGREGATE ALL SUGGESTIONS                                     │
  └───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 4: SELF-REFLECTION (Quality Improvement)                                │
  │ ═══════════════════════════════════════════════════════════════════════════ │
  │ This is a KEY INNOVATION - the model evaluates its own suggestions           │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │                                                                               │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │  REFLECTION CALL                                                         │ │
  │  │  ─────────────────────────────────────────────────────────────────────  │ │
  │  │                                                                          │ │
  │  │  Uses REASONING model (or primary if unavailable)                        │ │
  │  │                                                                          │ │
  │  │  Input:                                                                  │ │
  │  │  ┌────────────────────────────────────────────────────────────────────┐ │ │
  │  │  │ • Full PR diff with line numbers                                   │ │ │
  │  │  │ • All suggestions from Phase 3 (formatted as string)               │ │ │
  │  │  │ • Number of suggestions                                            │ │ │
  │  │  └────────────────────────────────────────────────────────────────────┘ │ │
  │  │                                                                          │ │
  │  │  Prompt (pr_code_suggestions_reflect_prompts.toml):                      │ │
  │  │  ┌────────────────────────────────────────────────────────────────────┐ │ │
  │  │  │ "You are an AI language model specialized in reviewing and        │ │ │
  │  │  │  evaluating code suggestions for a Pull Request..."               │ │ │
  │  │  │                                                                    │ │ │
  │  │  │  Tasks:                                                            │ │ │
  │  │  │  1. Evaluate each suggestion's quality, relevance, accuracy        │ │ │
  │  │  │  2. Detect correct line numbers in __new hunk__                    │ │ │
  │  │  │  3. Score each suggestion 0-10                                     │ │ │
  │  │  │  4. Explain reasoning for score                                    │ │ │
  │  │  └────────────────────────────────────────────────────────────────────┘ │ │
  │  │                                                                          │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  │                              │                                                │
  │                              ▼                                                │
  │           ┌───────────────────────────────────────────┐                      │
  │           │      REFLECTION RESPONSE (YAML)           │                      │
  │           │  ─────────────────────────────────────── │                      │
  │           │  code_suggestions:                        │                      │
  │           │    - suggestion_summary: "Add null check" │                      │
  │           │      relevant_file: "src/handler.py"      │                      │
  │           │      relevant_lines_start: 42             │                      │
  │           │      relevant_lines_end: 45               │                      │
  │           │      suggestion_score: 8                  │                      │
  │           │      why: "Critical - prevents NPE"       │                      │
  │           │    - suggestion_summary: "Rename var"     │                      │
  │           │      suggestion_score: 3                  │                      │
  │           │      why: "Minor style improvement"       │                      │
  │           └───────────────────────────────────────────┘                      │
  │                              │                                                │
  │                              ▼                                                │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │  analyze_self_reflection_response()                                      │ │
  │  │  ─────────────────────────────────────────────────────────────────────  │ │
  │  │  1. Parse reflection response YAML                                       │ │
  │  │  2. Merge scores back into original suggestions                          │ │
  │  │  3. Update line numbers if provided                                      │ │
  │  │  4. Mark invalid suggestions (score -1)                                  │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  │                                                                               │
  └───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 5: FILTERING & RANKING                                                  │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Filtering steps:                                                             │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │  1. Remove suggestions with identical existing/improved code             │ │
  │  │  2. Remove duplicate summaries                                           │ │
  │  │  3. Remove invalid suggestions (wrong keys, negative scores)             │ │
  │  │  4. Filter by score threshold (configurable)                             │ │
  │  │  5. Truncate if exceeds max suggestions                                  │ │
  │  │  6. Sort by score (descending)                                           │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  │                              │                                                │
  │                              ▼                                                │
  │         HIGH-QUALITY SUGGESTIONS (filtered and ranked)                        │
  └───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 6: PUBLISHING                                                           │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Two publishing modes:                                                        │
  │                                                                               │
  │  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐    │
  │  │ Summarized Table (default)      │  │ Inline Code Suggestions         │    │
  │  │ ─────────────────────────────── │  │ ─────────────────────────────── │    │
  │  │                                 │  │                                 │    │
  │  │ ┌─────────────────────────────┐│  │ Posted on specific lines:       │    │
  │  │ │ Category │ Suggestion │Score││  │                                 │    │
  │  │ ├──────────┼────────────┼─────┤│  │ ```suggestion                   │    │
  │  │ │ Security │ Add check  │  8  ││  │ fixed_code_here                 │    │
  │  │ │ Bug      │ Fix NPE    │  7  ││  │ ```                             │    │
  │  │ │ Perf     │ Cache call │  5  ││  │                                 │    │
  │  │ └─────────────────────────────┘│  │ Includes existing + improved    │    │
  │  │                                 │  │ code blocks                     │    │
  │  │ • Grouped by label              │  │                                 │    │
  │  │ • Sorted by quality score       │  │                                 │    │
  │  │ • Persistent comment            │  │                                 │    │
  │  └─────────────────────────────────┘  └─────────────────────────────────┘    │
  │                                                                               │
  │  Optional: Dual publishing (high scores inline, others in table)              │
  └───────────────────────────────────────────────────────────────────────────────┘
```

### Self-Reflection Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     SELF-REFLECTION DATA FLOW                                    │
└─────────────────────────────────────────────────────────────────────────────────┘

  Initial Suggestions                    Reflection Call
  ─────────────────────                  ─────────────────────

  ┌────────────────────┐                 ┌────────────────────┐
  │ Suggestion 1       │                 │ Input:             │
  │ ─────────────────  │                 │ • diff             │
  │ file: handler.py   │────────────────▶│ • suggestions_str  │
  │ existing: "..."    │                 │ • num_suggestions  │
  │ improved: "..."    │                 └─────────┬──────────┘
  │ summary: "..."     │                           │
  │ label: "bug"       │                           │
  │ [no score yet]     │                           ▼
  └────────────────────┘                 ┌────────────────────┐
                                         │ Reflection Prompt  │
  ┌────────────────────┐                 │ (REASONING model)  │
  │ Suggestion 2       │                 └─────────┬──────────┘
  │ ─────────────────  │                           │
  │ file: utils.py     │────────────────▶          │
  │ existing: "..."    │                           ▼
  │ improved: "..."    │                 ┌────────────────────┐
  │ summary: "..."     │                 │ Reflection Output  │
  │ label: "perf"      │                 │ ─────────────────  │
  │ [no score yet]     │                 │ suggestion_score: 8│
  └────────────────────┘                 │ lines: 42-45       │
                                         │ why: "Critical..." │
                                         └─────────┬──────────┘
                                                   │
                                                   ▼
  MERGED SUGGESTIONS                     ┌────────────────────┐
  ─────────────────────                  │ analyze_self_      │
                                         │ reflection_response│
  ┌────────────────────┐                 └─────────┬──────────┘
  │ Suggestion 1       │                           │
  │ ─────────────────  │◀──────────────────────────┘
  │ file: handler.py   │
  │ existing: "..."    │
  │ improved: "..."    │
  │ summary: "..."     │
  │ label: "bug"       │
  │ score: 8           │◀── FROM REFLECTION
  │ lines: 42-45       │◀── FROM REFLECTION
  │ why: "Critical..." │◀── FROM REFLECTION
  └────────────────────┘
```

---

## 4. Add Docs Pipeline

**Command:** `add_docs`
**Class:** `PRAddDocs` (`pr_agent/tools/pr_add_docs.py`)
**Prompt File:** `pr_agent/settings/pr_add_docs.toml`

### Pipeline Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           ADD DOCS PIPELINE                                      │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 1: INITIALIZATION                                                       │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  • Initialize git provider                                                    │
  │  • Detect main programming language                                           │
  │  • Determine documentation style for language:                                │
  │                                                                               │
  │    ┌────────────────────────────────────────────────────────────────────────┐│
  │    │  Language        │  Documentation Format                               ││
  │    │  ──────────────────────────────────────────────────────────────────── ││
  │    │  Python          │  Docstrings                                         ││
  │    │  Java            │  Javadocs                                           ││
  │    │  JavaScript/TS   │  JSdocs                                             ││
  │    │  C++             │  Doxygen                                            ││
  │    │  Others          │  Generic "Docs"                                     ││
  │    └────────────────────────────────────────────────────────────────────────┘│
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 2: DIFF GENERATION                                                      │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  get_pr_diff() with line numbers                                              │
  │  ───────────────────────────────────────────────────────────────────────────  │
  │  • Extracts diff with __new hunk__ and __old hunk__ format                    │
  │  • Includes line numbers for precise documentation placement                  │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 3: PROMPT PREPARATION                                                   │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Variables dict:                                                              │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │ title, branch, description, language                                     │ │
  │  │ diff, docs_for_language (e.g., "Docstrings")                             │ │
  │  │ extra_instructions                                                       │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  │                              │                                                │
  │                              ▼                                                │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │        pr_add_docs.toml                                                  │ │
  │  │  ───────────────────────────────────────────────────────────────────── │ │
  │  │  [pr_add_docs_prompt]                                                    │ │
  │  │  system = "You are PR-Doc, a language model that specializes..."        │ │
  │  │  user = "PR Info: Title: '{{ title }}'..."                               │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 4: AI CALL                                                              │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
           ┌───────────────────────────────────────────┐
           │         AI RESPONSE (YAML)                │
           │  ─────────────────────────────────────── │
           │  Code Documentation:                      │
           │    - relevant file: "src/handler.py"      │
           │      relevant line: 42                    │
           │      doc placement: "after"               │
           │      documentation: |                     │
           │        """                                │
           │        Handle the incoming request.       │
           │                                           │
           │        Args:                              │
           │            request: The HTTP request      │
           │        Returns:                           │
           │            Response object                │
           │        """                                │
           └───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 5: POST-PROCESSING                                                      │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  For each documentation entry:                                                │
  │  ─────────────────────────────────────────────────────────────────────────── │
  │  1. Locate file and line number                                               │
  │  2. Extract surrounding code for context                                      │
  │  3. Adjust indentation to match existing code                                 │
  │  4. Prepend/append based on doc placement ("before" or "after")               │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 6: PUBLISHING                                                           │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  For each documentation:                                                      │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │  Create inline code suggestion at specific line                          │ │
  │  │                                                                          │ │
  │  │  Format:                                                                 │ │
  │  │  ```suggestion                                                           │ │
  │  │  def handle_request(self, request):                                      │ │
  │  │      """                                                                 │ │
  │  │      Handle the incoming request.                                        │ │
  │  │      ...                                                                 │ │
  │  │      """                                                                 │ │
  │  │  ```                                                                     │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  │                                                                               │
  │  • Batch publish all suggestions                                              │
  │  • If batch fails, publish individually                                       │
  │  • Remove temporary progress comment                                          │
  └───────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Questions Pipeline

**Command:** `ask`, `ask_question`
**Class:** `PRQuestions` (`pr_agent/tools/pr_questions.py`)
**Prompt File:** `pr_agent/settings/pr_questions_prompts.toml`

### Pipeline Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           QUESTIONS PIPELINE                                     │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 1: QUESTION PARSING                                                     │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Parse command arguments                                                      │
  │  ───────────────────────────────────────────────────────────────────────────  │
  │                                                                               │
  │  Command examples:                                                            │
  │  • /ask What does this function do?                                           │
  │  • /ask ![image](path/to/screenshot.png) What's wrong here?                   │
  │                                                                               │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │  Check for image markdown: ![image](path)                                │ │
  │  │  If found:                                                               │ │
  │  │  • Extract image path (local or URL)                                     │ │
  │  │  • Store for AI handler (multimodal support)                             │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 2: DATA COLLECTION                                                      │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  • Initialize git provider                                                    │
  │  • Detect main language                                                       │
  │  • Get PR diff (standard format, no line numbers)                             │
  │  • Collect: title, branch, description, language, commit_messages             │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 3: PROMPT PREPARATION                                                   │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Variables: title, branch, description, language, diff, questions            │
  │                              │                                                │
  │                              ▼                                                │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │        pr_questions_prompts.toml                                         │ │
  │  │  ───────────────────────────────────────────────────────────────────── │ │
  │  │  [pr_questions_prompt]                                                   │ │
  │  │  system = "You are PR-Reviewer, a language model designed to answer..."  │ │
  │  │  user = "The PR Questions: {{ questions|trim }}"                         │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 4: AI CALL (with optional image support)                                │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  _get_prediction()                                                            │
  │  ───────────────────────────────────────────────────────────────────────────  │
  │                                                                               │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │  If image detected:                                                      │ │
  │  │  • Pass image_path to ai_handler                                         │ │
  │  │  • AI handler processes image (varies by provider)                       │ │
  │  │  • Model receives both text prompt and image                             │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  │                                                                               │
  │  Uses WEAK model (faster/cheaper for Q&A)                                     │
  │                                                                               │
  │  Response: Free-form text answer                                              │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 5: POST-PROCESSING (Security)                                           │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Sanitize response to prevent command injection:                              │
  │  ───────────────────────────────────────────────────────────────────────────  │
  │  • Replace "\n/" with "\n /" (prevent quick actions)                          │
  │  • For GitLab: remove GitHub quick actions                                    │
  │  • If starts with "/": prepend space                                          │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 6: PUBLISHING                                                           │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Format:                                                                      │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │  ### **Ask**❓                                                           │ │
  │  │  [User's question here]                                                  │ │
  │  │                                                                          │ │
  │  │  ### **Answer:**                                                         │ │
  │  │  [AI's response here]                                                    │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  │                                                                               │
  │  • Publish as PR comment                                                      │
  │  • Add help text if enabled                                                   │
  └───────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Changelog Update Pipeline

**Command:** `update_changelog`
**Class:** `PRUpdateChangelog` (`pr_agent/tools/pr_update_changelog.py`)
**Prompt File:** `pr_agent/settings/pr_update_changelog_prompts.toml`

### Pipeline Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        CHANGELOG UPDATE PIPELINE                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 1: INITIALIZATION                                                       │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  • Initialize git provider                                                    │
  │  • Detect main language                                                       │
  │  • Retrieve existing CHANGELOG.md:                                            │
  │    ┌────────────────────────────────────────────────────────────────────────┐│
  │    │  If found:                                                              ││
  │    │  • Read first 50 lines (truncate for context)                           ││
  │    │  If not found:                                                          ││
  │    │  • Use default template                                                 ││
  │    └────────────────────────────────────────────────────────────────────────┘│
  │  • Parse configuration for push strategy                                      │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 2: DATA COLLECTION                                                      │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Variables:                                                                   │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │ title, branch, description, language                                     │ │
  │  │ diff, commit_messages_str                                                │ │
  │  │ today (current date)                                                     │ │
  │  │ changelog_file_str (existing content, truncated)                         │ │
  │  │ pr_link (optional - for clickable links)                                 │ │
  │  │ extra_instructions                                                       │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 3: PROMPT PREPARATION                                                   │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │        pr_update_changelog_prompts.toml                                  │ │
  │  │  ───────────────────────────────────────────────────────────────────── │ │
  │  │  [pr_update_changelog_prompt]                                            │ │
  │  │  system = "You are PR-Changelog-Updater. Your task is to add a brief     │ │
  │  │           summary of this PR's changes to CHANGELOG.md..."               │ │
  │  │  user = "The current 'CHANGELOG.md' file: {{ changelog_file_str }}"      │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 4: AI CALL (WEAK model)                                                 │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
           ┌───────────────────────────────────────────┐
           │         AI RESPONSE (Markdown)            │
           │  ─────────────────────────────────────── │
           │  ## [1.2.0] - 2024-01-15 [*](pr_link)     │
           │  ### Added                                │
           │  - New authentication flow                │
           │  ### Fixed                                │
           │  - Bug in user registration               │
           └───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 5: CHANGELOG ASSEMBLY                                                   │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  1. Strip whitespace and backticks from response                              │
  │  2. Remove code block markers if present                                      │
  │  3. Combine: new_entry + "\n\n" + existing_content                            │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 6: PUBLISHING                                                           │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Two modes:                                                                   │
  │                                                                               │
  │  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐    │
  │  │ Push to File                    │  │ Comment Only                    │    │
  │  │ (if git provider supports)      │  │                                 │    │
  │  │ ─────────────────────────────── │  │ ─────────────────────────────── │    │
  │  │                                 │  │                                 │    │
  │  │ 1. create_or_update_pr_file()   │  │ Posts changelog as comment      │    │
  │  │    with optional "[skip ci]"    │  │ with instruction to push        │    │
  │  │ 2. Wait 5 seconds               │  │ manually                        │    │
  │  │ 3. Create review comment on     │  │                                 │    │
  │  │    CHANGELOG changes            │  │                                 │    │
  │  │ 4. Fallback to comment if       │  │                                 │    │
  │  │    review creation fails        │  │                                 │    │
  │  └─────────────────────────────────┘  └─────────────────────────────────┘    │
  └───────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Labels Generation Pipeline

**Command:** `generate_labels`
**Class:** `PRGenerateLabels` (`pr_agent/tools/pr_generate_labels.py`)
**Prompt File:** `pr_agent/settings/pr_custom_labels.toml`

### Pipeline Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        LABELS GENERATION PIPELINE                                │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 1: INITIALIZATION                                                       │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  • Initialize git provider                                                    │
  │  • Detect main language                                                       │
  │  • Initialize token handler with label prompt                                 │
  │  • Load custom labels configuration (if enabled)                              │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 2: LABEL CONFIGURATION                                                  │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Default Labels:                     Custom Labels (if enabled):              │
  │  ┌────────────────────────┐          ┌────────────────────────────────────┐  │
  │  │ • Bug fix              │          │ Defined in configuration:          │  │
  │  │ • Tests                │    OR    │ • Custom label names               │  │
  │  │ • Enhancement          │          │ • Custom descriptions              │  │
  │  │ • Documentation        │          │ • Custom Pydantic class            │  │
  │  │ • Other                │          └────────────────────────────────────┘  │
  │  └────────────────────────┘                                                   │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 3: PROMPT PREPARATION                                                   │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Variables:                                                                   │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │ title, branch, description, language                                     │ │
  │  │ diff, commit_messages_str                                                │ │
  │  │ enable_custom_labels (flag)                                              │ │
  │  │ custom_labels_class (Pydantic class definition)                          │ │
  │  │ extra_instructions                                                       │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  │                              │                                                │
  │                              ▼                                                │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │        pr_custom_labels.toml                                             │ │
  │  │  ───────────────────────────────────────────────────────────────────── │ │
  │  │  [pr_custom_labels_prompt]                                               │ │
  │  │  system = "Your task is to provide labels that describe the PR..."       │ │
  │  │  user = "The PR Git Diff: {{ diff|trim }}"                               │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 4: AI CALL                                                              │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
           ┌───────────────────────────────────────────┐
           │         AI RESPONSE (YAML)                │
           │  ─────────────────────────────────────── │
           │  labels:                                  │
           │    - "Bug fix"                            │
           │    - "Enhancement"                        │
           └───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 5: POST-PROCESSING                                                      │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  1. Parse YAML response                                                       │
  │  2. Extract labels (handle list or comma-separated string)                    │
  │  3. Convert to original case using labels_minimal_to_labels_dict              │
  │  4. Strip and clean label strings                                             │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 6: PUBLISHING                                                           │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Label merging:                                                               │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │  1. Get current PR labels                                                │ │
  │  │  2. Extract user-created labels (filter out system labels)               │ │
  │  │  3. Merge AI-generated labels with user labels                           │ │
  │  │  4. Remove duplicates                                                    │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  │                                                                               │
  │  Publishing modes:                                                            │
  │  ┌──────────────────────────┐  ┌──────────────────────────────────┐          │
  │  │ If supports get_labels   │  │ Otherwise                        │          │
  │  │ ────────────────────────│  │ ─────────────────────────────── │          │
  │  │ Publish labels directly  │  │ Publish labels as comment        │          │
  │  │ to PR via API            │  │                                  │          │
  │  └──────────────────────────┘  └──────────────────────────────────┘          │
  └───────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Help Pipeline

**Command:** `help`
**Class:** `PRHelpMessage` (`pr_agent/tools/pr_help_message.py`)
**Prompt File:** `pr_agent/settings/pr_help_prompts.toml`

### Pipeline Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              HELP PIPELINE                                       │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 1: QUESTION PARSING                                                     │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Parse command arguments to extract help question                             │
  │                                                                               │
  │  Command examples:                                                            │
  │  • /help How do I configure the review settings?                              │
  │  • /help What labels can I use?                                               │
  │                                                                               │
  │  If no question provided: skip processing                                     │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 2: DOCUMENTATION LOADING                                                │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Load PR-Agent documentation content                                          │
  │  (stored in snippets variable)                                                │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 3: PROMPT PREPARATION                                                   │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Variables: question, snippets (documentation content)                        │
  │                              │                                                │
  │                              ▼                                                │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │        pr_help_prompts.toml                                              │ │
  │  │  ───────────────────────────────────────────────────────────────────── │ │
  │  │  [pr_help_prompts]                                                       │ │
  │  │  system = "You are Doc-helper, a language model designed to answer       │ │
  │  │           questions about PR-Agent (Qodo Merge)..."                       │ │
  │  │  user = "User's Question: {{ question|trim }}"                           │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  └───────────────────────────────────┬───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 4: AI CALL                                                              │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
           ┌───────────────────────────────────────────┐
           │         AI RESPONSE (YAML)                │
           │  ─────────────────────────────────────── │
           │  user_question: |                         │
           │    How do I configure review settings?    │
           │  response: |                              │
           │    You can configure review settings...   │
           │  relevant_sections:                       │
           │    - file_name: "review.md"               │
           │      relevant_section_header_string: |    │
           │        ## Configuration                   │
           └───────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ PHASE 5: PUBLISHING                                                           │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Two modes:                                                                   │
  │  ┌──────────────────────────────────────────────────────────────────────────┐│
  │  │ • Post response as PR comment (if publish_output enabled)                ││
  │  │ • Return as string (if return_as_string=True)                            ││
  │  └──────────────────────────────────────────────────────────────────────────┘│
  └───────────────────────────────────────────────────────────────────────────────┘
```

---

## Common Components

### Token Handler

The `TokenHandler` class manages token counting and budget allocation across all pipelines:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            TOKEN HANDLER                                         │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Responsibilities:                                                            │
  │  ─────────────────────────────────────────────────────────────────────────── │
  │  • Count tokens using tiktoken encoder (model-specific)                       │
  │  • Calculate prompt_tokens from system + user prompts                         │
  │  • Validate total tokens vs model limits                                      │
  │  • For Claude: uses Anthropic API for accurate count                          │
  │  • For OpenAI: uses tiktoken                                                  │
  │  • For others: estimation with configurable factor                            │
  └───────────────────────────────────────────────────────────────────────────────┘

  Token Budgeting Flow:
  ─────────────────────

  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
  │ Calculate       │────▶│ Compare to      │────▶│ Prune if        │
  │ Token Count     │     │ Model Limit     │     │ Over Budget     │
  └─────────────────┘     └─────────────────┘     └─────────────────┘
         │                                               │
         ▼                                               ▼
  ┌─────────────────┐                           ┌─────────────────┐
  │ System prompt   │                           │ Remove files:   │
  │ + User prompt   │                           │ 1. Added files  │
  │ + Diff content  │                           │ 2. Modified     │
  │ = Total tokens  │                           │ 3. Deleted      │
  └─────────────────┘                           └─────────────────┘
```

### AI Handler

The `LiteLLMAIHandler` provides a unified interface to multiple LLM providers:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         AI HANDLER (LiteLLM)                                     │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Supported Providers:                                                         │
  │  ─────────────────────────────────────────────────────────────────────────── │
  │  • OpenAI (GPT-4, GPT-3.5, etc.)                                              │
  │  • Anthropic (Claude 3, Claude 2)                                             │
  │  • Azure OpenAI                                                               │
  │  • Google (Gemini)                                                            │
  │  • AWS Bedrock                                                                │
  │  • Ollama (local models)                                                      │
  │  • And many more via LiteLLM                                                  │
  └───────────────────────────────────────────────────────────────────────────────┘

  Chat Completion Flow:
  ─────────────────────

  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │ System +     │────▶│ LiteLLM      │────▶│ Provider     │────▶│ Response +   │
  │ User Prompts │     │ Wrapper      │     │ API Call     │     │ finish_reason│
  └──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
         │                    │
         ▼                    ▼
  ┌──────────────┐     ┌──────────────┐
  │ Optional:    │     │ Parameters:  │
  │ Image path   │     │ • model      │
  │ (multimodal) │     │ • temperature│
  └──────────────┘     │ • max_tokens │
                       └──────────────┘
```

### Fallback Model Retry

The `retry_with_fallback_models()` function ensures reliability by trying multiple models:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       FALLBACK MODEL RETRY                                       │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────────────────────┐
  │                                                                               │
  │   ┌─────────────────┐    Fail    ┌─────────────────┐    Fail    ┌──────────┐ │
  │   │ Primary Model   │───────────▶│ Fallback #1     │───────────▶│ Fallback │ │
  │   │ (e.g., GPT-4)   │            │ (e.g., GPT-3.5) │            │ #2 ...   │ │
  │   └────────┬────────┘            └─────────────────┘            └──────────┘ │
  │            │ Success                                                          │
  │            ▼                                                                  │
  │   ┌─────────────────┐                                                         │
  │   │ Return Result   │                                                         │
  │   └─────────────────┘                                                         │
  │                                                                               │
  │   Only throw exception after ALL models fail                                  │
  └───────────────────────────────────────────────────────────────────────────────┘
```

### Git Provider Abstraction

All pipelines use a common git provider interface supporting multiple platforms:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       GIT PROVIDER ABSTRACTION                                   │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  Supported Platforms:                                                         │
  │  ─────────────────────────────────────────────────────────────────────────── │
  │                                                                               │
  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐  │
  │  │  GitHub    │  │  GitLab    │  │  Bitbucket │  │  Azure DevOps          │  │
  │  └────────────┘  └────────────┘  └────────────┘  └────────────────────────┘  │
  │                                                                               │
  │  Common Interface:                                                            │
  │  ┌─────────────────────────────────────────────────────────────────────────┐ │
  │  │ • get_diff_files()        - Get PR diff files                           │ │
  │  │ • publish_comment()       - Post comment on PR                          │ │
  │  │ • publish_description()   - Update PR description                       │ │
  │  │ • publish_labels()        - Add labels to PR                            │ │
  │  │ • create_inline_comment() - Post comment on specific line               │ │
  │  │ • get_pr_description()    - Get current PR description                  │ │
  │  │ • get_user_description()  - Get user-entered description                │ │
  │  │ • create_or_update_pr_file() - Push file changes to PR branch           │ │
  │  └─────────────────────────────────────────────────────────────────────────┘ │
  └───────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary: Prompt to Pipeline Mapping

| Pipeline | Main Prompt File | Model Type | Key Features |
|----------|------------------|------------|--------------|
| PR Review | `pr_reviewer_prompts.toml` | REGULAR | Ticket compliance, effort estimation, security analysis |
| PR Description | `pr_description_prompts.toml` | WEAK | Mermaid diagrams, semantic file types, labels |
| Code Suggestions | `pr_code_suggestions_prompts.toml` + `reflect_prompts.toml` | REGULAR + REASONING | Self-reflection, score-based filtering |
| Add Docs | `pr_add_docs.toml` | REGULAR | Language-specific docstrings, inline suggestions |
| Questions | `pr_questions_prompts.toml` | WEAK | Image support (multimodal), free-form answers |
| Changelog | `pr_update_changelog_prompts.toml` | WEAK | File push or comment, existing format matching |
| Labels | `pr_custom_labels.toml` | REGULAR | Custom labels support, label merging |
| Help | `pr_help_prompts.toml` | REGULAR | Documentation search, relevant sections |

---
