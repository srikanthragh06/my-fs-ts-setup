# my-fs-ts-setup

Canonical setup guide for full-stack TypeScript projects.

`CLAUDE.md` documents the monorepo structure, tech stack, coding patterns, and scaffolding checklist used across projects. Point a new Claude Code instance at it to replicate the setup in any new project.

## Usage

When starting a new project, tell Claude Code:

> Follow the setup guide in `/home/sr1k5/myStuff/my-fs-ts-setup/CLAUDE.md` exactly and scaffold the project from the checklist at the bottom.

## Stack

- **Backend** — NestJS, Kysely, PostgreSQL, Redis, Socket.io
- **Frontend** — React 19, Vite, Jotai, Mantine, Tailwind CSS
- **Shared** — Zod schemas and TypeScript types via pnpm workspace
- **Infra** — Docker Compose, Nginx (blue-green deployment)
