# CHANGES Template — Plantilla completa de `CHANGES.md`

Esta es la **estructura exacta** que tenés que generar en la raíz del proyecto. Adaptá nombres y contenido al dominio del proyecto, pero respetá el orden y el formato de cada sección.

---

```markdown
# CHANGES — Secuencia de Implementación

> Índice canónico de todos los changes del proyecto **{NombreProyecto}**.
> Cada change es atómico: un agente puede implementarlo en una sesión (~4-6 horas).
> **Leer este archivo antes de ejecutar cualquier `/opsx:propose`.**

---

## Cómo usar este documento

1. Identificar el change a implementar (verificar que sus dependencias están en `openspec/changes/archive/`).
2. Leer los docs de la knowledge-base indicados en "Leer antes".
3. Ejecutar `/opsx:propose <nombre-del-change>`.
4. Al terminar el change, archivarlo con `/opsx:archive <nombre-del-change>`.
5. Marcar el checkbox `[x]` en este archivo.

---

## Árbol de dependencias

```
C-01 foundation-setup
  └── C-02 core-models
        └── C-03 auth                          ← desbloquea TODO lo demás
              │
              ├── C-04 menu-catalog
              │     ├── C-05 allergens
              │     └── C-06 sectors-tables
              │
              ├── C-07 ingredients             ← paralelo con C-04
              │
              └── C-08 dashboard-shell         ← solo necesita C-03
                    └── C-09 dashboard-pages   ← + C-04 + C-05
```

### Paralelismo por fase

> Cada "gate" es un punto de sincronización. Los changes dentro de un gate pueden ejecutarse en paralelo.

```
GATE 0: ninguna
  → C-01 (solo)

GATE 1: C-01 ✓
  → C-02 (solo)

GATE 2: C-02 ✓
  → C-03 (solo)

GATE 3: C-03 ✓                     ← PRIMER FORK (3 paralelos)
  → C-04 menu-catalog              [Agente A]
  → C-07 ingredients               [Agente B]
  → C-08 dashboard-shell           [Agente C]

GATE 4: C-04 ✓
  → C-05 allergens                 [Agente B]
  → C-06 sectors-tables            [Agente A]

GATE 5: C-08 + C-04 + C-05 ✓
  → C-09 dashboard-pages           [Agente C]
```

### Camino crítico (5 changes — mínimo irreducible)

```
C-01 → C-02 → C-03 → C-04 → C-09
```

### Plan óptimo con 3 agentes

```
Paso │ Agente A (Backend Core)  │ Agente B (Backend Aux)  │ Agente C (Frontend)
─────┼──────────────────────────┼─────────────────────────┼─────────────────────
  1  │ C-01 foundation-setup    │          —              │         —
  2  │ C-02 core-models         │          —              │         —
  3  │ C-03 auth                │          —              │         —
  4  │ C-04 menu-catalog        │ C-07 ingredients        │ C-08 dashboard-shell
  5  │ C-06 sectors-tables      │ C-05 allergens          │         —
  6  │          —               │          —              │ C-09 dashboard-pages
```

---

## FASE 0 — Cimientos

### [C-01] `foundation-setup`
- **Estado**: `[ ]` pendiente
- **Scope**: Scaffolding completo del monorepo + infraestructura base
  - Estructura de directorios: `backend/`, `frontend/`, `docs/`, `knowledge-base/`
  - `backend/`: FastAPI app mínima con health check `/api/health`, Alembic inicializado, `shared/` con settings, logger, db, exceptions
  - `frontend/`: Vite + React + TypeScript scaffolding, Zustand instalado, Tailwind
  - `.env.example` en cada sub-proyecto
  - GitHub Actions CI: jobs paralelos para backend y frontend
  - Variables sensibles vía `${VAR}` sin defaults hardcodeados
- **Dependencias**: ninguna
- **Governance**: BAJO
- **Leer antes**:
  - `knowledge-base/01_vision_y_objetivos.md` (qué es el sistema y por qué existe)
  - `knowledge-base/02_descripcion_general.md` §Stack
  - `knowledge-base/08_arquitectura_propuesta.md` §Estructura de directorios

---

### [C-02] `core-models`
- **Estado**: `[ ]` pendiente
- **Scope**: Modelos base + migraciones iniciales + seed mínimo
  - Modelos: `Tenant`, `User`, `Role`, `UserRole`
  - `AuditMixin` con `is_active`, `created_at`, `updated_at`, `deleted_at`
  - `BaseRepository[T]`, `UnitOfWork`
  - Migración 001: tablas core
  - Seed mínimo: 1 tenant, 1 usuario ADMIN
- **Dependencias**: `C-01`
- **Governance**: CRITICO
- **Leer antes**:
  - `knowledge-base/04_modelo_de_datos.md` §entidades core
  - `knowledge-base/08_arquitectura_propuesta.md` §Patrones (Repository, UoW)
  - `knowledge-base/09_decisiones_y_supuestos.md` (decisiones de modelado)

---

## FASE 1A — Autenticación

### [C-03] `auth`
- **Estado**: `[ ]` pendiente
- **Scope**: Sistema completo de autenticación JWT con RBAC
  - `POST /api/auth/login` — JWT access + refresh, rate limiting 5/60s por IP+email
  - `POST /api/auth/refresh` — rotación de refresh token, blacklist del anterior
  - `POST /api/auth/logout` — blacklist del access token
  - `GET /api/auth/me` — info del usuario actual
  - `PermissionContext`: `require_role()`, `require_admin()`
  - Campos JWT: `sub`, `tenant_id`, `roles`, `email`, `jti`, `type`, `iat`, `exp`
  - Refresh token en cookie HttpOnly (secure, samesite=lax)
  - Migración 002: tablas auth
  - Tests: login correcto, token expirado, rate limit, refresh rotation
- **Dependencias**: `C-02`
- **Governance**: CRITICO
- **Leer antes**:
  - `knowledge-base/03_actores_y_roles.md`
  - `knowledge-base/05_reglas_de_negocio.md` §Autenticación
  - `knowledge-base/07_flujos_principales.md` §Auth flow

---

## FASE 1B — Dominio principal

> Los changes C-04 y C-07 pueden proponerse en paralelo. C-04 debe archivarse antes de C-06.

### [C-04] `menu-catalog`
- **Estado**: `[ ]` pendiente
- **Scope**: Catálogo del menú con endpoints admin y públicos
  - Modelos: `Category`, `Subcategory`, `Product`
  - Endpoints admin CRUD (`/api/admin/categories`, `/subcategories`, `/products`)
  - Endpoint público: `GET /api/public/menu` — cacheado en Redis (TTL 5 min)
  - Paginación, precios en centavos, validación anti-SSRF en URLs de imagen
  - Migración 003: tablas catálogo
  - Tests: CRUD, aislamiento multi-tenant, cache invalidation
- **Dependencias**: `C-03`
- **Governance**: BAJO
- **Leer antes**:
  - `knowledge-base/04_modelo_de_datos.md` §Catálogo
  - `knowledge-base/07_flujos_principales.md` §Catálogo público
  - `knowledge-base/05_reglas_de_negocio.md` §Catálogo

---

### [C-NN] `nombre-del-change`
- **Estado**: `[ ]` pendiente
- **Scope**: (bullets operacionales — modelos, endpoints, migraciones, eventos, tests)
- **Dependencias**: `C-NN` | ninguna
- **Governance**: BAJO | MEDIO | ALTO | CRITICO
- **Leer antes**:
  - `knowledge-base/0X_archivo.md` §sección
  - ...

---

## FASE {M} — {Nombre semántico}

(Continuar el patrón hasta cubrir todos los changes del proyecto)

---

## Resumen

| Métrica | Valor |
|---------|-------|
| Total changes | 9 |
| Fases | 4 |
| Camino crítico | 5 changes (C-01→C-02→C-03→C-04→C-09) |
| Gates de paralelismo | 5 |
| Governance CRITICO | 2 (C-02, C-03) |
| Governance ALTO | 0 |
```

---

## Checklist de validación al cerrar

Antes de devolver el archivo al usuario, verificá:

- [ ] El header dice `# CHANGES — Secuencia de Implementación`.
- [ ] La sección "Cómo usar este documento" tiene los 5 pasos numerados.
- [ ] El "Árbol de dependencias" usa ASCII art con `└──` y `│`.
- [ ] Si hay 5+ changes: existe al menos un GATE por cada fork de paralelismo detectado.
- [ ] Si hay <5 changes: la sección GATES fue omitida con la nota correspondiente.
- [ ] El camino crítico tiene una flecha lineal sin ramas (`C-01 → ... → C-NN`).
- [ ] El "Plan óptimo con N agentes" tiene N columnas y pasos enumerados (N ajustado al tamaño).
- [ ] Cada change tiene exactamente 5 campos: Estado, Scope, Dependencias, Governance, Leer antes.
- [ ] Cada "Leer antes" tiene 3 a 5 archivos KB (con sección cuando aplique).
- [ ] El Scope tiene bullets operacionales (modelos, endpoints, migraciones, tests).
- [ ] Cada Governance es uno de: BAJO, MEDIO, ALTO, CRITICO.
- [ ] Los changes están agrupados en FASES con nombres semánticos (no "Fase 1, 2, 3" pelados).
- [ ] La tabla **Resumen** está al final con todas sus filas completas.
- [ ] Si fue Modo Update: los checkboxes `[x]` preservados están restaurados en el nuevo archivo.
- [ ] Si fue Modo Update y hubo changes eliminados con `[x]`: existe la sección `## ⚠️ Changes eliminados (tenían progreso)`.
- [ ] Los encabezados de change usan el formato exacto: `### [C-NN] \`nombre-kebab-case\`` (sin llaves, NN = número real con padding).
