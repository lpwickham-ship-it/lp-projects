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
