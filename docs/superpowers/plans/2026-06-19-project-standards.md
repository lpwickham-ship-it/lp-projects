# Project Standards Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Set up a root-level `CLAUDE.md` and `README.md` that establish consistent coding standards across all current and future projects, then push them to GitHub.

**Architecture:** Two files live at the root of the `projects/` folder. `CLAUDE.md` is read automatically by Claude Code at the start of every session. `README.md` is the human-readable equivalent committed to GitHub as a permanent record.

**Tech Stack:** TypeScript, Next.js, Tailwind CSS, Anthropic MCP SDK, npm, Vercel, Git, GitHub CLI (`gh`)

## Prerequisites

Before starting:

1. Initialize a git repo at the `projects/` root:
   ```bash
   cd /Users/louis-philippe/Desktop/projects
   git init
   ```
2. Install and authenticate the GitHub CLI:
   ```bash
   brew install gh
   gh auth login
   ```

## Global Constraints

- All projects use TypeScript — no plain `.js` files
- Package manager is always npm
- One GitHub repository per project
- `main` branch is always deployable

---

### Task 1: Create CLAUDE.md

**Files:**
- Create: `projects/CLAUDE.md`

**Interfaces:**
- Produces: A file Claude Code reads automatically at session start, encoding all project-wide standards

- [ ] **Step 1: Create the file**

Create `projects/CLAUDE.md` with this exact content:

```markdown
# Project Standards

## Who I Am
I am a complete beginner who codes exclusively with Claude. Always explain things simply, avoid jargon, and favor the simplest solution that works.

## Tech Stack
- **Language:** TypeScript (all projects — never plain JavaScript)
- **Web apps & websites:** Next.js + Tailwind CSS
- **Claude MCP tools:** TypeScript + Anthropic MCP SDK
- **Scripts / utilities:** TypeScript + Node.js
- **Package manager:** npm (never yarn or pnpm)
- **Deployment:** Vercel for web apps

## Project Structure
Every project lives in its own folder at the root of `projects/`:

```
projects/
├── CLAUDE.md
├── README.md
└── <project-name>/
    ├── src/         ← all source code
    ├── public/      ← static assets (images, icons)
    ├── package.json
    └── README.md
```

Each project has its own GitHub repository. The `projects/` root folder has its own repo for standards docs only.

## Git Conventions
- `main` is always deployable — never commit broken code to main
- New features go on a branch: `feature/<short-description>`
- Commit messages: plain English, start with a verb — e.g. `Add homepage layout`
- Commit small and often

## General Rules
- Keep files small and focused — one clear responsibility per file
- Build only what is needed — no speculative features
- Every project must have a `README.md` explaining what it does and how to run it
- Always use TypeScript strict mode
```

---

### Task 2: Create README.md and commit both files

**Files:**
- Create: `projects/README.md`

**Interfaces:**
- Consumes: `CLAUDE.md` written in Task 1
- Produces: A human-readable standards doc suitable for GitHub; both files committed to git

- [ ] **Step 1: Create the file**

Create `projects/README.md` with this exact content:

```markdown
# LP Projects — Standards & Conventions

This is the root workspace for all of Louis-Philippe's projects. Every project lives as its own folder here and has its own GitHub repository.

---

## Tech Stack

| Use case | Tools |
|---|---|
| Web app or website | TypeScript + Next.js + Tailwind CSS |
| Claude marketplace tool | TypeScript + Anthropic MCP SDK |
| Script or utility | TypeScript + Node.js |

- **Package manager:** npm
- **Deployment:** Vercel (web apps)

---

## Folder Structure

```
projects/
├── CLAUDE.md          ← read by Claude Code automatically
├── README.md          ← this file
└── <project-name>/    ← one folder per project
    ├── src/
    ├── public/
    ├── package.json
    └── README.md
```

---

## Git Conventions

- `main` is always deployable
- Feature branches: `feature/<description>`
- Commit messages: plain English, imperative verb — e.g. `Add homepage layout`
- Commit small and often

---

## Projects

| Project | Type | Repo |
|---|---|---|
| *(project name)* | *(Web app / Claude MCP / Script)* | *(GitHub URL)* |

*(Add a row for each new project)*
```

- [ ] **Step 2: Commit both files**

```bash
git add CLAUDE.md README.md
git commit -m "Add project standards (CLAUDE.md, README.md)"
```

---

### Task 3: Commit docs and push to GitHub

**Files:**
- Stage: `docs/` folder

**Interfaces:**
- Consumes: git repo from Prerequisites; commits from Tasks 1–2

- [ ] **Step 1: Commit the docs folder**

```bash
git add docs/
git commit -m "Add brainstorming spec and implementation plan"
```

- [ ] **Step 2: Create GitHub repository and push**

```bash
gh repo create lp-projects --public --source=. --remote=origin --push
```

Expected: GitHub prints a URL like `https://github.com/<your-username>/lp-projects`

- [ ] **Step 3: Verify**

```bash
gh repo view lp-projects
```

Expected: shows the repo URL and confirms the latest commit is visible.

---

## Done

Standards are live — Claude reads them automatically each session, and they are version-controlled on GitHub.
