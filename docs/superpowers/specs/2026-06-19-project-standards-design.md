# Project Standards Design

**Date:** 2026-06-19
**Status:** Approved

## Overview

A root-level standards setup for an open-ended multi-project workspace. Two artifacts: a `CLAUDE.md` that Claude reads automatically every session, and a `README.md` pushed to GitHub as a human-readable record of decisions. Everything lives in the root of the `projects/` folder.

## Project Types and Tech Stack

| Type | Tools |
|---|---|
| Web app or website | TypeScript + Next.js + Tailwind CSS |
| Claude marketplace tool | TypeScript + Anthropic MCP SDK |
| Script or utility | TypeScript + Node.js |

- **Package manager:** npm
- **Deployment:** Vercel (web apps)

A single language across all project types eliminates per-project toolchain switching. New types can be added to this table as they arise.

## Folder Structure

```
projects/
├── CLAUDE.md
├── README.md
└── <project-name>/     ← one folder per project, added as needed
```

## Git Conventions

- **One repo per project** — each project has its own GitHub repository. The `projects/` root folder has its own repo for standards docs.
- **Branch strategy:** `main` is always deployable. Feature work goes on named branches (`feature/<description>`).
- **Commit messages:** Plain English, imperative verb — e.g. `Add homepage layout`.

## Artifacts to Produce

1. `CLAUDE.md` at `projects/` root — encodes stack, structure, and git rules for Claude to follow automatically
2. `README.md` at `projects/` root — human-readable version of the same standards, pushed to GitHub
