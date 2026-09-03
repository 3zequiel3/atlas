# CHANGES Template — la forma exacta de `CHANGES.md`

Este archivo manda sobre **cómo se ve** el output. Las reglas de *qué* poner en cada sección viven en `SKILL.md`.

El bloque de abajo es un ejemplo completo y **verificado**: el árbol, los gates, el camino crítico, el plan y el Resumen son consistentes entre sí. Adaptá dominio y nombres; respetá el orden y el formato.

> **Cuidado con los fences.** El ejemplo va envuelto en un fence de **4 backticks** porque adentro usa fences de 3. Al escribir el `CHANGES.md` real, las secciones ASCII (árbol, gates, camino crítico, plan) van en fences normales de 3 backticks.

---

````markdown
# CHANGES — Secuencia de Implementación

> Índice canónico de todos los changes del proyecto **{NombreProyecto}**.
> Cada change es atómico: un agente puede implementarlo en una sesión (~4-6 horas).
> **Leer este archivo antes de ejecutar cualquier `/opsx:propose`.**

---

## Cómo usar este documento

1. Verificar que las dependencias del change target están en `openspec/changes/archive/`.
2. Leer los docs de la KB indicados en "Leer antes" del change.
3. Ejecutar `/opsx:propose <nombre-del-change>`.
4. Al terminar, archivar con `/opsx:archive <nombre-del-change>`.
5. Marcar el checkbox `[x]` del change en este archivo.

---

## Árbol de dependencias

> `←` marca dependencias **adicionales** a la arista del árbol. Un árbol no puede
> dibujar más de un padre por nodo: cuando un change depende de varios, el padre
> principal es la rama y el resto se anota con `← + C-XX`.

```
C-01 foundation-setup
└── C-02 core-models
    └── C-03 auth
        ├── C-04 menu-catalog
        │   ├── C-05 allergens
        │   └── C-06 sectors-tables
        ├── C-07 ingredients
        └── C-08 dashboard-shell
            └── C-09 dashboard-pages        ← + C-05
```

### Paralelismo por fase

> Cada gate es un punto de sincronización real: abre ramas paralelas (fork) o exige
> que varias converjan (join). Los tramos lineales no son gates — están en el árbol.

```
GATE 1: C-03 auth ✓                         ← PRIMER FORK
  → C-04 menu-catalog                       [Agente A]
  → C-07 ingredients                        [Agente B]
  → C-08 dashboard-shell                    [Agente C]

GATE 2: C-04 menu-catalog ✓
  → C-05 allergens                          [Agente B]
  → C-06 sectors-tables                     [Agente A]

GATE 3: C-05 + C-08 ✓                       ← JOIN
  → C-09 dashboard-pages                    [Agente C]
```

### Camino crítico (6 changes — cadena más larga)

```
C-01 → C-02 → C-03 → C-04 → C-05 → C-09
```

### Plan óptimo con 3 agentes

```
Paso │ Agente A (Backend Core)  │ Agente B (Backend Aux)  │ Agente C (Frontend)
─────┼──────────────────────────┼─────────────────────────┼──────────────────────
  1  │ C-01 foundation-setup    │           —             │          —
  2  │ C-02 core-models         │           —             │          —
  3  │ C-03 auth                │           —             │          —
  4  │ C-04 menu-catalog        │ C-07 ingredients        │ C-08 dashboard-shell
  5  │ C-06 sectors-tables      │ C-05 allergens          │          —
  6  │           —              │           —             │ C-09 dashboard-pages
```

---

## FASE 0 — Cimientos

### [C-01] `foundation-setup`

- [x] **Estado**
- **Scope**: Scaffolding del monorepo + infraestructura base
  - Estructura de directorios: `backend/`, `frontend/`, `docs/`, `knowledge-base/`
  - `backend/`: FastAPI con health check `/api/health`, Alembic inicializado, `shared/` con settings, logger, db, exceptions
  - `frontend/`: Vite + React + TypeScript, Zustand, Tailwind
  - `.env.example` en cada sub-proyecto
  - GitHub Actions CI: jobs paralelos para backend y frontend
  - Variables sensibles vía `${VAR}` sin defaults hardcodeados
- **Dependencias**: ninguna
- **Governance**: BAJO
- **Leer antes**:
  - `knowledge-base/02_descripcion_general.md` §Stack
  - `knowledge-base/08_arquitectura_propuesta.md` §Estructura de directorios
  - `knowledge-base/04_modelo_de_datos.md` §Convenciones

---

### [C-02] `core-models`

- [ ] **Estado**
- **Scope**: Modelos base + migraciones iniciales + seed mínimo
  - Modelos: `Tenant`, `User`, `Role`, `UserRole`
  - `AuditMixin` con `is_active`, `created_at`, `updated_at`, `deleted_at`
  - `BaseRepository[T]`, `UnitOfWork`
  - Migración 001: tablas core
  - Seed mínimo: 1 tenant, 1 usuario ADMIN
- **Dependencias**: `C-01`
- **Governance**: CRITICO
- **Leer antes**:
  - `knowledge-base/04_modelo_de_datos.md` §Entidades core
  - `knowledge-base/08_arquitectura_propuesta.md` §Patrones (Repository, UoW)
  - `knowledge-base/05_reglas_de_negocio.md` §Multi-tenancy

---

## FASE 1 — Autenticación

### [C-03] `auth`

- [ ] **Estado**
- **Scope**: Autenticación JWT con RBAC
  - `POST /api/auth/login` — JWT access + refresh, rate limiting 5/60s por IP+email
  - `POST /api/auth/refresh` — rotación de refresh token, blacklist del anterior
  - `POST /api/auth/logout` — blacklist del access token
  - `GET /api/auth/me` — info del usuario actual
  - `PermissionContext`: `require_role()`, `require_admin()`
  - Claims JWT: `sub`, `tenant_id`, `roles`, `email`, `jti`, `type`, `iat`, `exp`
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

## FASE 2 — Dominio principal

> C-04 y C-07 se pueden proponer en paralelo. C-05 y C-06 dependen ambos de C-04.

### [C-04] `menu-catalog`

- [ ] **Estado**
- **Scope**: Catálogo del menú con endpoints admin y públicos
  - Modelos: `Category`, `Subcategory`, `Product`
  - CRUD admin: `/api/admin/categories`, `/subcategories`, `/products`
  - `GET /api/public/menu` — cacheado en Redis (TTL 5 min)
  - Paginación, precios en centavos
  - Migración 003: tablas catálogo
  - Tests: CRUD, aislamiento multi-tenant, cache invalidation
- **Dependencias**: `C-03`
- **Governance**: MEDIO
- **Leer antes**:
  - `knowledge-base/04_modelo_de_datos.md` §Catálogo
  - `knowledge-base/07_flujos_principales.md` §Catálogo público
  - `knowledge-base/05_reglas_de_negocio.md` §Catálogo

---

*(FASE 2 continúa con C-05, C-06 y C-07; FASE 3 contiene C-08 y C-09. Se omiten acá por brevedad —
en un `CHANGES.md` real van los 9 completos, y el Resumen de abajo cuenta el archivo entero.)*

---

## Resumen

| Métrica | Valor |
|---------|-------|
| Total changes | 9 |
| Fases | 4 |
| Camino crítico | 6 changes |
| Gates de paralelismo | 3 |
| Governance CRITICO | 2 |
| Governance ALTO | 0 |
````

---

## Esqueleto de un change

Los cinco campos, en este orden exacto. **No lo copies literal** — es la forma, no contenido:

| Campo | Forma | Valores |
|-------|-------|---------|
| Estado | `- [ ] **Estado**` / `- [x] **Estado**` | task item GFM real, primer bullet del change |
| Scope | `- **Scope**: {resumen}` + sub-bullets | bullets operacionales |
| Dependencias | ``- **Dependencias**: `C-NN` `` | `ninguna`, un ID, o varios separados por coma |
| Governance | `- **Governance**: {nivel}` | BAJO, MEDIO, ALTO o CRITICO |
| Leer antes | `- **Leer antes**:` + sub-bullets | 3 a 5 paths verificados en disco |

Encabezado del change: ``### [C-NN] `nombre-kebab-case` `` — `NN` es el número real con padding de 2, el nombre va en backticks.

---

## Checklist de validación

**Correla en el paso 14 del Workflow, antes de escribir el archivo.** Cada ítem que falle se corrige antes de seguir.

**Estructura**

- [ ] El header dice `# CHANGES — Secuencia de Implementación`.
- [ ] Las secciones de primer nivel siguen el orden fijo, y `## Resumen` es la última del archivo.
- [ ] `## Cómo usar este documento` tiene 5 pasos (o 4 en modo sin OpenSpec).
- [ ] Los changes están agrupados en `## FASE {N}` con nombres semánticos y numeración entera sin sufijos de letra.

**Grafo**

- [ ] El árbol usa ASCII con `└──` y `│`, dentro de un fence de 3 backticks.
- [ ] Todo change con más de un padre tiene su anotación `← + C-XX`.
- [ ] Cada arista del camino crítico existe en el árbol.
- [ ] El camino crítico es la cadena **más larga**: ninguna otra cadena del árbol tiene más changes.
- [ ] Si hay gates: cada uno es un fork (2+ hijos) o un join (2+ predecesores). Ningún tramo lineal figura como gate.
- [ ] Si no hay forks ni joins: `### Paralelismo por fase` y `### Plan óptimo` están omitidos, con la nota correspondiente en su lugar.

**Coherencia numérica**

- [ ] `Total changes` = cantidad de encabezados `### [C-NN]`.
- [ ] `Fases` = cantidad de encabezados `## FASE`.
- [ ] `Gates de paralelismo` = cantidad de bloques `GATE`.
- [ ] `Camino crítico` = cantidad de changes de la cadena = cantidad de pasos del Plan con N agentes.
- [ ] `Governance CRITICO` y `ALTO` = conteo real de changes con ese nivel.
- [ ] El ID de cada change es mayor que el de todas sus dependencias.

**Por change**

- [ ] Exactamente 5 campos: Estado, Scope, Dependencias, Governance, Leer antes.
- [ ] El Estado es un task item GFM real (`- [ ] **Estado**`), no `` `[ ]` `` entre backticks.
- [ ] El Scope tiene bullets operacionales (modelos, endpoints, migraciones, tests).
- [ ] Ningún change con seguridad, permisos o aislamiento de datos en el Scope está marcado BAJO.
- [ ] Cada "Leer antes" tiene 3 a 5 paths, **todos verificados en disco**.
- [ ] En modo orquestado, los paths de "Leer antes" son los reales de `state.kb.files`.

**Modo Update**

- [ ] Los `[x]` preservados se matchearon **por nombre**, no por `C-NN`.
- [ ] Si hubo changes con progreso que desaparecieron: existe `## ⚠️ Changes eliminados (tenían progreso)`, ubicada antes del Resumen.
- [ ] El archivo se escribió **una sola vez**, ya con los checkboxes restaurados.

**Higiene**

- [ ] No quedó ningún placeholder literal (`{NombreProyecto}`, `C-NN`, `0X_archivo.md`) en el output.
