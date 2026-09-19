# helpdesk-api — Backend

Se carga junto con el CLAUDE.md global y el de la carpeta padre (`HelpDesk/CLAUDE.md`), que tiene el objetivo del proyecto, el stack y la estructura de repos. Acá va solo lo específico de la API.

## Estado

Repo vacío, sin código todavía. Falta la **Fase 0** (ver CLAUDE.md padre) antes de scaffoldear.

## Stack

- NestJS + TypeScript
- Prisma ORM sobre PostgreSQL en Neon (branching por feature; no Prisma Postgres)
- Auth: JWT en cookie httpOnly/secure/sameSite. El guard extrae el token de `req.cookies`, no del header `Authorization` (ver resolución en el CLAUDE.md global).
- Testing: Jest

## Convenciones

- Git: ramas cortas desde `develop`, conventional commits. `main` solo se actualiza con un release explícito.
- Skills a usar según la tarea: `nestjs-best-practices`, `prisma-cli`, `prisma-client-api`, `postgresql-table-design`, `javascript-typescript-jest`, `auth-implementation-patterns`.
- Los cambios de contrato (DTOs, endpoints, errores, cookies) se avisan explícitamente: `helpdesk-web` es otro repo y se actualiza aparte.
