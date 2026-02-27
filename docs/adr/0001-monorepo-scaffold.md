# ADR 0001: Initial monorepo scaffold

## Status
Accepted

## Context
The MVP requires a React Native client, Node/TypeScript backend, and shared contracts package.

## Decision
Create a pnpm workspace with `apps/mobile`, `services/api`, and `packages/contracts` using strict TypeScript and a shared base tsconfig.

## Consequences
- Enables incremental task implementation with a compile-ready baseline.
- Lint/test scripts are placeholders until later tasks provide full tooling.
