# helpdesk-api — Backend

Se carga junto con el CLAUDE.md global y el de la carpeta padre (`HelpDesk/CLAUDE.md`), que tiene el objetivo del proyecto, el stack y la estructura de repos. Acá va solo lo específico de la API.

## Estado

**Fase 0 completada** (alcance, modelo de datos y arquitectura decididos). Falta la **Fase 1 (setup)**: scaffold de NestJS, instalar Prisma, conectar una rama de Neon y correr la primera migración. Todavía no hay código de aplicación.

Ya existe un borrador de `prisma/schema.prisma`, **sin validar**: la CLI de Prisma no estaba instalada y `npx prisma` falló con un error interno de npm (`edgesOut`), no del schema. Validarlo apenas Prisma esté instalado en el proyecto, y ajustar el `output` del generator al layout final de NestJS.

## Documentación de diseño (`docs/`)

- `docs/modelo-de-datos.html`: diagrama entidad-relación del schema, con reglas de borrado, índices y enums.
- `docs/arquitectura-backend.html`: módulos de NestJS, camino de un request, login y cookie, filtro por organización, transacciones, migraciones, adjuntos en S3, cola con SQS y Lambda, despliegue en AWS y testing. Cada sección trae preguntas de estudio.

Se abren en el navegador (necesitan internet para las fuentes). Son diseño propuesto, no código: si contradicen al código o al schema, manda el código.

**Mantenerlos al día:** si un cambio modifica el schema (tabla, campo, índice, regla de borrado, enum), actualizar `docs/modelo-de-datos.html` en el mismo commit y avisarlo. Lo mismo con `docs/arquitectura-backend.html` si se cambia una decisión que allí aparece (por ejemplo dónde vive el filtro por organización). Los diagramas tienen las coordenadas escritas a mano en un script al final del archivo: al editarlos, revisar que no queden solapamientos.

## Stack

- NestJS + TypeScript
- Prisma ORM sobre PostgreSQL en Neon (branching por feature; no Prisma Postgres). Prisma 7: el generator `prisma-client` exige `output`, y la URL de conexión va en `prisma.config.ts`, no en el schema.
- Auth: JWT en cookie httpOnly/secure/sameSite. El guard extrae el token de `req.cookies`, no del header `Authorization` (ver resolución en el CLAUDE.md global).
- Testing: Jest
- Cloud (Fase 8): imagen Docker en ECR corriendo en ECS Fargate, S3 para adjuntos, SQS + Lambda para trabajo asíncrono (emails), Secrets Manager, CloudWatch. La base sigue en Neon; RDS solo como experimento opcional al final. Verificar costos antes de crear recursos.

## Decisiones tomadas

El detalle de tablas, índices y borrados está en `prisma/schema.prisma`; acá solo lo que no se deduce de leerlo.

- **Multi-tenancy:** tablas compartidas, con `organizationId` en cada tabla de negocio. El `organizationId` sale **siempre** del JWT (`req.user`), nunca del body, params ni query, y toda query filtra por él.
- **IDs:** UUID v7 (`uuid(7)`).
- **Email de `User`:** único global, guardado en minúsculas (normalizar en el service).
- **Customer** es un contacto que abre tickets, no inicia sesión. Roles de usuario: `ADMIN`, `AGENT`.
- **Estado y prioridad** son enums. **Category** es una tabla por organización.
- Los usuarios no se borran: se desactivan (`isActive`), porque comentarios e historial los referencian.
- **Número de ticket** legible por organización: `Organization.ticketSeq` + `Ticket.number`, incrementado de forma atómica dentro de la transacción que crea el ticket (el `UPDATE` bloquea la fila). Único por `(organizationId, number)`.
- **Transacciones que ya están identificadas:** el registro crea `Organization` + primer `User` ADMIN; crear un ticket toca contador + ticket + log; cambiar el estado escribe `Ticket` + `ActivityLog`.
- **Adjuntos:** van directo del navegador a S3 con URL prefirmada. En Postgres queda solo la metadata y `storageKey`; la key lleva el `organizationId` como prefijo.

## Decisiones abiertas (no asumir: resolver con el usuario cuando toque)

- **Versión de Prisma:** la documentación consultada muestra Prisma 8 como última versión (con un flujo de migraciones distinto) y trata a la 7 como la anterior. El `schema.prisma` y las notas de este archivo están escritos para la 7. Verificar y decidir en el bloque H1.1, antes de instalar.
- Dónde vive el filtro por organización: explícito en cada query o centralizado (p. ej. una extensión del cliente de Prisma). Se decide en la Fase 2 con casos reales.
- Qué base usan los tests de integración: rama de Neon o Postgres en Docker.
- Cookie entre el front (Vercel) y la API (AWS): si quedan en sitios distintos, el navegador la trata como de terceros. Lo más limpio es un dominio común (`app.` y `api.`). Verificar al integrar el front; afecta CORS y `SameSite`.
- Qué pasa con un JWT ya emitido cuando se desactiva al usuario (vencimiento corto, chequeo en cada request, etc.).

## Alcance y orden de trabajo

Acordado:
- **Cortes verticales** con el front: primero auth de punta a punta, después tickets; no todo el back y después todo el front.
- **Tests desde el primer corte.** Prioridad: integración (auth, permisos, aislamiento entre organizaciones, transacciones); unit solo donde haya lógica real.
- **Despliegue mínimo después del H1.**
- Hitos y bloques de trabajo: `../estudio/GUIA.md` y `../estudio/H1.md` (leer solo el bloque que se indique). Las decisiones a resolver en cada bloque están marcadas ahí.

**Fuera del MVP:** Knowledge Base, SLA, tags, integraciones, realtime, IA, Row-Level Security de Postgres, foreign key compuesta para garantizar consistencia entre tenants (por ahora se valida en el service) y el patrón outbox.

## Cómo trabajar acá

El usuario viene de frontend. Al introducir un concepto de backend (transacciones, guards, inyección de dependencias, migraciones, etc.), explicarlo antes de aplicarlo: el objetivo es entender y poder defender cada decisión, no solo que funcione.

## Convenciones

- Git: ramas cortas desde `develop`, conventional commits. `main` solo se actualiza con un release explícito.
- Skills a usar según la tarea: `nestjs-best-practices`, `prisma-cli`, `prisma-client-api`, `postgresql-table-design`, `javascript-typescript-jest`, `auth-implementation-patterns`.
- Los cambios de contrato (DTOs, endpoints, errores, cookies) se avisan explícitamente: `helpdesk-web` es otro repo y se actualiza aparte.
