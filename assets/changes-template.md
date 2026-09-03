# CHANGES Template — la forma exacta de `CHANGES.md`

Este archivo manda sobre **cómo se ve** el output. Las reglas de *qué* poner en cada sección viven en `SKILL.md`.

El bloque de abajo es un ejemplo **completo y verificable**: están los 7 changes, y cada número del Resumen se puede contar en el propio archivo. Adaptá dominio y nombres; respetá el orden y el formato.

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

> El checkbox `[x]` es la fuente de verdad del progreso para atlas. Si un change quedó
> archivado en OpenSpec pero sin tildar acá, atlas lo va a tratar como pendiente.

---

## Árbol de dependencias

> Un árbol no puede dibujar más de un padre por nodo. Cuando un change depende de varios,
> el padre de mayor profundidad es la rama y el resto se anota con `← + C-XX`. Esa
> anotación **es una arista real**, no un comentario.

```
C-01 foundation-setup
└── C-02 core-models
    └── C-03 auth
        ├── C-04 menu-catalog
        │   └── C-06 orders
        │       └── C-07 admin-dashboard   ← + C-05
        └── C-05 staff-management
```

### Paralelismo por fase

> Cada gate es un punto de sincronización real: abre ramas paralelas (fork) o exige que
> varias converjan (join). Los tramos lineales no son gates — ya están en el árbol.
> Cuando un gate abre 3 o más ramas, el primero lleva la anotación `← PRIMER FORK`.

```
GATE 1: C-03 auth ✓
  → C-04 menu-catalog                      [Agente A]
  → C-05 staff-management                  [Agente B]

GATE 2: C-05 + C-06 ✓
  → C-07 admin-dashboard                   [Agente A]
```

### Camino crítico (6 changes — cadena más larga)

```
C-01 → C-02 → C-03 → C-04 → C-06 → C-07
```

### Plan óptimo con 2 agentes

```
Paso │ Agente A (Backend Core)   │ Agente B (Backend Aux)
─────┼───────────────────────────┼────────────────────────
  1  │ C-01 foundation-setup     │           —
  2  │ C-02 core-models          │           —
  3  │ C-03 auth                 │           —
  4  │ C-04 menu-catalog         │ C-05 staff-management
  5  │ C-06 orders               │           —
  6  │ C-07 admin-dashboard      │           —
```

---

## FASE 0 — Cimientos

### [C-01] `foundation-setup`

- [ ] **Estado**
- **Scope**: Scaffolding del monorepo + infraestructura base
  - Estructura de directorios: `backend/`, `frontend/`, `docs/`
  - `backend/`: FastAPI con health check `/api/health`, Alembic inicializado, `shared/` con settings, logger, db
  - `frontend/`: Vite + React + TypeScript, Zustand, Tailwind
  - `.env.example` en cada sub-proyecto
  - GitHub Actions CI: jobs paralelos para backend y frontend
- **Dependencias**: ninguna
- **Governance**: BAJO
- **Leer antes**:
  - `knowledge-base/08_arquitectura_propuesta.md` §Estructura de directorios
  - `knowledge-base/04_modelo_de_datos.md` §Convenciones de nombrado
  - `knowledge-base/06_funcionalidades.md` §Alcance del MVP

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

> C-04 y C-05 se pueden proponer en paralelo. C-06 requiere que C-04 esté archivado.

### [C-04] `menu-catalog`

- [ ] **Estado**
- **Scope**: Catálogo del menú con endpoints admin y públicos
  - Modelos: `Category`, `Subcategory`, `Product`
  - CRUD admin: `/api/admin/categories`, `/subcategories`, `/products`
  - `GET /api/public/menu` — cacheado en Redis (TTL 5 min)
  - Paginación, precios en centavos
  - Migración 003: tablas catálogo
  - Tests: CRUD, aislamiento multi-tenant, invalidación de cache
- **Dependencias**: `C-03`
- **Governance**: MEDIO
- **Leer antes**:
  - `knowledge-base/04_modelo_de_datos.md` §Catálogo
  - `knowledge-base/07_flujos_principales.md` §Catálogo público
  - `knowledge-base/05_reglas_de_negocio.md` §Catálogo

---

### [C-05] `staff-management`

- [ ] **Estado**
- **Scope**: Alta, baja y asignación de roles del personal
  - Modelo: `StaffMember` con relación a `User` y `Role`
  - CRUD admin: `/api/admin/staff`
  - `POST /api/admin/staff/{id}/roles` — asignación y revocación de roles
  - Regla: no se puede revocar el último ADMIN del tenant
  - Migración 004: tablas de personal
  - Tests: escalada de privilegios, último admin, aislamiento por tenant
- **Dependencias**: `C-03`
- **Governance**: ALTO
- **Leer antes**:
  - `knowledge-base/03_actores_y_roles.md` §Personal
  - `knowledge-base/05_reglas_de_negocio.md` §Roles
  - `knowledge-base/04_modelo_de_datos.md` §Personal

---

### [C-06] `orders`

- [ ] **Estado**
- **Scope**: Ciclo de vida del pedido
  - Modelos: `Order`, `OrderItem` — referencian `Product` de C-04
  - Máquina de estados: `abierto → confirmado → en_preparacion → servido → cerrado`
  - `POST /api/orders`, `PATCH /api/orders/{id}/estado`
  - Evento WS `order.updated` al cambiar de estado
  - Migración 005: tablas de pedidos
  - Tests: transiciones válidas e inválidas, concurrencia en cierre
- **Dependencias**: `C-04`
- **Governance**: MEDIO
- **Leer antes**:
  - `knowledge-base/04_modelo_de_datos.md` §Pedidos
  - `knowledge-base/07_flujos_principales.md` §Ciclo del pedido
  - `knowledge-base/05_reglas_de_negocio.md` §Pedidos

---

## FASE 3 — Panel de administración

### [C-07] `admin-dashboard`

- [ ] **Estado**
- **Scope**: Panel web que consume catálogo, personal y pedidos
  - Shell con layout, routing protegido y guard de rol ADMIN
  - Vistas: catálogo, personal, pedidos en vivo (suscripción a `order.updated`)
  - Estado global con Zustand, un store por dominio
  - Tests: render por rol, reconexión del WS
- **Dependencias**: `C-05, C-06`
- **Governance**: BAJO
- **Leer antes**:
  - `knowledge-base/08_arquitectura_propuesta.md` §Frontend
  - `knowledge-base/06_funcionalidades.md` §Panel de administración
  - `knowledge-base/03_actores_y_roles.md` §Personal

---

## Resumen

| Métrica | Valor |
|---------|-------|
| Total changes | 7 |
| Fases | 4 |
| Camino crítico | 6 changes |
| Gates de paralelismo | 2 |
| Governance CRITICO | 2 |
| Governance ALTO | 1 |
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
| Leer antes | `- **Leer antes**:` + sub-bullets | hasta 5 paths verificados en disco |

Encabezado del change: ``### [C-NN] `nombre-kebab-case` `` — `NN` es el número real con padding de 2, el nombre va en backticks.

---

## Checklist de validación

**Correla en el paso 15 del Workflow, contra el contenido en memoria, antes de escribir.** Cada ítem que falle se corrige antes de seguir.

**Estructura**

- [ ] El header dice `# CHANGES — Secuencia de Implementación`.
- [ ] Las secciones de primer nivel siguen el orden fijo, y `## Resumen` es la última del archivo.
- [ ] `## Cómo usar este documento` tiene 5 pasos (o 4 en modo sin OpenSpec).
- [ ] Los changes están agrupados en `## FASE {N}` con nombres semánticos y numeración entera sin sufijos de letra.
- [ ] Ninguna FASE contiene un change que dependa de otro de una FASE posterior.

**Grafo**

- [ ] El árbol usa ASCII con `└──` y `│`, dentro de un fence de 3 backticks.
- [ ] Todo change con más de un padre tiene su anotación `← + C-XX`.
- [ ] Cada arista del camino crítico existe en el árbol, contando las anotaciones `←`.
- [ ] El camino crítico es la cadena **más larga**: ninguna otra cadena del grafo tiene más changes.
- [ ] Si hay gates: cada uno es un fork (2+ hijos) o un join (2+ predecesores). Ningún tramo lineal figura como gate.
- [ ] `← PRIMER FORK` aparece solo si algún gate abre 3 o más ramas, y solo en el primero.
- [ ] El plan tiene **al menos** tantos pasos como changes el camino crítico.
- [ ] Cada `[Agente X]` de los gates coincide con la columna de ese change en el plan.
- [ ] Ninguna columna del plan está vacía en todos los pasos.
- [ ] Si no hay forks ni joins: `### Camino crítico` y `### Plan óptimo` están omitidos, con la nota correspondiente en su lugar.

**Coherencia numérica**

- [ ] `Total changes` = cantidad de encabezados `### [C-NN]`.
- [ ] `Fases` = cantidad de encabezados `## FASE`.
- [ ] `Gates de paralelismo` = cantidad de bloques `GATE`.
- [ ] `Camino crítico` = cantidad de changes de la cadena.
- [ ] `Governance CRITICO` y `ALTO` = conteo real de changes con ese nivel.
- [ ] El ID de cada change es mayor que el de todas sus dependencias.

**Por change**

- [ ] Exactamente 5 campos: Estado, Scope, Dependencias, Governance, Leer antes.
- [ ] El Estado es un task item GFM real (`- [ ] **Estado**`), no `` `[ ]` `` entre backticks.
- [ ] En una generación limpia, **todos** los Estados están en `[ ]`.
- [ ] Los nombres kebab-case son únicos en el archivo.
- [ ] El Scope tiene bullets operacionales (modelos, endpoints, migraciones, tests).
- [ ] Ningún change que implemente control de acceso, secretos o aislamiento entre tenants está marcado BAJO.
- [ ] Cada "Leer antes" tiene hasta 5 paths, **todos verificados en disco**. Con una KB mínima puede tener menos de 3 — nunca un path inventado para llegar al piso.
- [ ] En modo orquestado, los paths de "Leer antes" son los reales de `state.kb.files`.

**Modo Update**

- [ ] El archivo anterior pasó el guard de archivo ajeno antes de tocarse.
- [ ] Los `[x]` preservados se matchearon **por nombre**, no por `C-NN`, aceptando también el formato legacy `` `[x]` ``.
- [ ] Si hubo bajas con progreso: existe `## ⚠️ Changes eliminados (tenían progreso)` como tabla `Change | ID anterior`, insertada antes del Resumen.
- [ ] El contenido en memoria ya tiene los checkboxes restaurados — el write todavía no ocurrió.

**Higiene**

- [ ] No quedó ningún placeholder literal (`{NombreProyecto}`, `C-NN`, `{N}`) en el output.
