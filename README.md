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
