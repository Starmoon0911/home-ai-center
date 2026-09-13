# Repository Guidelines

## Project Structure & Module Organization

This repository is currently in the specification stage; `Home_AI_Platform_Spec_v0.1.docx` is the architecture source of truth. The planned layout places Next.js applications in `apps/`, backend components in `services/`, the host daemon in `agents/node-agent/`, shared TypeScript libraries in `packages/`, deployment configuration in `infrastructure/`, service manifests in `registry/`, and supporting material in `docs/`. Keep infrastructure-specific logic out of application packages, and expose reusable contracts through `packages/shared/` or the relevant SDK.

## Build, Test, and Development Commands

No build system or package scripts have been committed yet. When scaffolding the workspace, expose consistent root-level commands, preferably through `package.json`:

- `npm install` installs workspace dependencies.
- `npm run dev` starts local development services.
- `npm run build` builds all applications, services, and packages.
- `npm test` runs the complete automated test suite.
- `npm run lint` checks formatting and static-analysis rules.
- `docker compose up -d` starts local infrastructure once a Compose file exists.

Document any service-specific setup in that service's README.

## Coding Style & Naming Conventions

Use TypeScript for the platform, Portal, and orchestration layers; reserve Python/FastAPI for ML-specific services. Follow two-space indentation in TypeScript, four spaces in Python, `camelCase` for variables and functions, `PascalCase` for types and React components, and kebab-case for service directories and manifest names. Validate external configuration with Zod or JSON Schema. Add formatter and linter configurations with the first implementation rather than relying on editor defaults.

## Testing Guidelines

Place unit tests beside their modules as `*.test.ts` or under a service-level `tests/` directory; use `test_*.py` for Python. Cover manifest validation, authorization, RAG scope isolation, service lifecycle transitions, and health checks. Every bug fix should include a regression test. Integration tests must avoid requiring production secrets or fixed node addresses.

## Commit & Pull Request Guidelines

There is no Git history yet. Use Conventional Commit subjects such as `feat(registry): validate service manifests` or `docs: describe node enrollment`. Keep commits focused. Pull requests should explain the change, affected services, verification performed, configuration or migration impact, and linked issues. Include screenshots for Portal changes and sample manifests for registry changes.

## Security & Configuration Tips

Never commit secrets, tokens, model credentials, or private data. Route external access through Traefik and authentication, enforce permissions server-side for MCP tools, and test RAG ACL boundaries. Do not hard-code node IPs or direct vLLM endpoints; use service discovery and logical LiteLLM model aliases.
