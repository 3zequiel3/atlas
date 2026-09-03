---
name: atlas
description: >
  Generates CHANGES.md — an operational index of all changes needed to implement a project,
  with dependency tree, parallelism gates, critical path, multi-agent plan, and per-change
  scope, governance level, dependencies, and "Leer antes" pointers to the knowledge base.
  Supports standalone and orchestrated modes. Preserves progress checkboxes on regeneration.
  Fire-and-forget in new projects; merge-safe on existing CHANGES.md.
  Trigger: "crear CHANGES.md", "generar roadmap", "armar CHANGES", "armar roadmap",
  "crear mapa de changes", "generar plan de implementación", "qué changes necesito",
  "índice de changes", "regenerar CHANGES.md", "actualizar CHANGES.md".
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "3.0"
---

## When to Use

- Generar el **índice canónico** de changes para implementar un sistema desde cero hasta producción.
- Convertir una base de conocimiento estructurada en un plan operativo con paralelización explícita.
- Identificar **camino crítico**, **niveles de governance**, y **contratos con la KB** por change.
- **Regenerar** un `CHANGES.md` existente sin perder el estado de progreso (`[x]`).

**Don't use when:**
- No existe `knowledge-base/` en raíz (corré primero la skill `chronicle` para generarla).
- Ya existe `CHANGES.md` y el usuario quiere modificar un solo change puntualmente — hacé una edición directa en su lugar.

---

## Critical Patterns

### Pre-checks obligatorios

Antes de generar, verificá las siguientes condiciones. Si alguna falla, **detené la ejecución** y devolvé el mensaje indicado. No generes nada parcialmente.

| # | Check | Si falla |
|---|-------|----------|
| 1 | `knowledge-base/` existe en raíz del proyecto | "Falta la KB. Corré primero la skill `chronicle` para generarla (`chronicle` → Mode B o Mode A)." |
| 2 | La KB tiene los **canónicos necesarios** para atlas (ver regla abajo) | "KB incompleta para atlas. Faltan los nodos mínimos: [lista]. Completá la KB con `chronicle`." |
| 3 | `openspec/` existe en raíz **Y** contiene al menos un archivo de configuración válido (`openspec.config.*`, `config.yaml`, `config.json`, o cualquier archivo no vacío en su raíz) | "OpenSpec no inicializado o vacío. Corré `npx @fission-ai/openspec@latest init` en la raíz del proyecto." |

> **IMPORTANTE — Pre-check 2 (canónicos mínimos):**  
> `chronicle` genera KBs con perfiles que omiten nodos intencionalmente (p. ej., perfil `cli` omite 06, 07; perfil `library_sdk` omite 03, 07). La condición no es "existen los 10 canónicos", sino que existan los **4 nodos que atlas necesita leer**:
> - `04_modelo_de_datos.md` — requerido siempre.
> - `06_funcionalidades.md` — requerido si el sistema tiene funcionalidades de usuario; omitido en `library_sdk` y `data_pipeline`.
> - `07_flujos_principales.md` — requerido si el sistema tiene flujos E2E; omitido en `cli` y `library_sdk`.
> - `08_arquitectura_propuesta.md` — requerido siempre.
>
> Si alguno de los requeridos falta, listá los que faltan y detené. Si un nodo es "requerido condicionalmente" y está ausente, lo notás al usuario y continuás con los que hay.
>
> **Nunca falles el pre-check solo porque el total de nodos es menor a 10.**

> **IMPORTANTE — Pre-check 3 (openspec/):**  
> Este pre-check es relevante porque `CHANGES.md` referencia comandos `/opsx:propose`. Si el usuario explícitamente indica que no va a usar OpenSpec (ej: "solo quiero el plan de changes"), omití este pre-check y generá el archivo eliminando las referencias a `/opsx:propose` del output. Documentá la omisión con una nota al final del archivo generado.

### Modo de operación: ¿CHANGES.md ya existe?

Antes de escribir, verificá si ya existe `CHANGES.md` en la raíz:

- **No existe** → generación limpia (fire-and-forget normal).
- **Ya existe** → activá el **Modo Update (merge-safe)**:
  1. Leé el `CHANGES.md` actual y extraé todos los checkboxes en estado `[x]` (con sus IDs `C-NN`).
  2. Generá el nuevo `CHANGES.md` normalmente.
  3. Para cada `C-NN` que estaba `[x]` en el archivo anterior **y sigue existiendo en el nuevo**, restaurá su estado a `[x]`.
  4. Si un `C-NN` anterior ya no existe en el nuevo (fue renombrado o eliminado), incluí una nota al final del archivo bajo `## ⚠️ Changes eliminados (tenían progreso)` listando los IDs y nombres originales.
  5. Al cerrar, indicá cuántos checkboxes se preservaron y cuántos se perdieron por restructuración.

---

## Input — qué leer de la KB

### Modo orquestado (`.jr-orchestrator-state.json` presente, `version == 2`)

1. Leé `state.kb.files` como lista autoritativa de paths.
2. **Antes de leer cada archivo**: verificá que existe en disco. Si alguno no existe, reportá `⚠ archivo ausente: {path}` y continuá con los que sí existen.
3. Si `state.kb.files` está vacío o ausente → caé al modo standalone (punto siguiente).
4. Leé `state.kb.discovery` como contexto de nombrado (hint only, no fuente de verdad):
   - `needs_infra: true` → esperá un change de `foundation-setup` (C-01).
   - `system_type` → nombre las FASEs acorde al dominio.
   - `domain` → inferí changes específicos del dominio.
   - `scale` → inferí si se necesitan changes de RBAC o multi-tenancy.
5. Pre-checks en modo orquestado:
   - Saltear el check de existencia de `knowledge-base/` (los paths vienen del estado).
   - Verificar existencia en disco de cada archivo de `state.kb.files` (punto 2).
   - Ejecutar el pre-check de `openspec/` normalmente (salvo excepción del usuario).

### Modo standalone (sin estado, o fallback)

Ejecutá los 3 pre-checks normales. Luego leé de `knowledge-base/`:

**Siempre leer (si existen):**
1. `04_modelo_de_datos.md` → entidades y relaciones (orden de creación de tablas).
2. `06_funcionalidades.md` → US y épicas (unidad de cada change).
3. `07_flujos_principales.md` → flujos E2E (changes atómicos vs compuestos).
4. `08_arquitectura_propuesta.md` → patrones (infraestructura previa necesaria).

**Si alguno de los anteriores no existe:** notificá al usuario (`⚠ nodo ausente: {nombre} — continuando sin él`) y generá con la información disponible.

**Leer si están:**
- `03_actores_y_roles.md` → para changes de auth y RBAC.
- `05_reglas_de_negocio.md` → para detectar reglas que cruzan changes.
- `10_preguntas_abiertas.md` → para flaggear changes con dependencias inciertas.

---

## Formato obligatorio de CHANGES.md

El archivo SIEMPRE tiene esta estructura. **No agregues ni quites secciones de primer nivel** — son contrato.

```markdown
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
5. Marcar el checkbox `[x]` en este archivo.

---

## Árbol de dependencias

(ASCII art jerárquico con └── y │ mostrando la cadena de dependencias)

### Paralelismo por fase

(GATES numerados desde GATE 0. Formato: ver §Reglas GATES)

### Camino crítico ({N} changes — mínimo irreducible)

(cadena lineal más corta hacia producción)

### Plan óptimo con {X} agentes

(tabla: Paso | Agente A | Agente B | Agente C — X puede ser 1, 2 o 3 según el tamaño del proyecto)

---

## FASE {N} — {Nombre semántico de la fase}

> Nota opcional sobre paralelismo dentro de la fase.

### [C-NN] `nombre-kebab-case`

- **Estado**: `[ ]` pendiente
- **Scope**: descripción densa con bullets operacionales (modelos, endpoints, eventos, migraciones, tests)
- **Dependencias**: ninguna | `C-NN` | `C-NN, C-MM`
- **Governance**: BAJO | MEDIO | ALTO | CRITICO
- **Leer antes**:
  - `knowledge-base/0X_archivo.md` §{sección}
  - `knowledge-base/0Y_archivo.md`

---

## Resumen

| Métrica | Valor |
|---------|-------|
| Total changes | N |
| Fases | M |
| Camino crítico | K changes |
| Gates de paralelismo | G |
| Governance CRITICO | X changes |
| Governance ALTO | Y changes |
```

### Reglas para nombrar changes

- Códigos secuenciales `C-01`, `C-02`, ..., `C-NN`. **Siempre padding de 2 dígitos.**
- Nombre en kebab-case, descriptivo, sin prefijos `us-NNN-`.
- Si un change cubre varias US, mencionarlas en el **Scope** (no en el nombre).
- Changes transversales de infra igual reciben `C-NN` normal.

### Reglas para inferir dependencias

Jerarquía obligatoria (en este orden):

1. **Infra primero**: `C-01` es siempre el `foundation-setup`. No depende de nada.
2. **Modelos core antes que features**: `C-02` suele ser `core-models` (entidades base, mixins, repositorios).
3. **Auth antes que recursos protegidos**: cualquier change que requiera usuario logueado depende del change de auth.
4. **Entidad referenciada antes que la que referencia**: `categorias` antes que `productos`.
5. **Backend antes que frontend acoplado**: si una vista consume un endpoint, depende del change del endpoint.
6. **Integraciones externas / pagos / webhooks al final**: dependen de las entidades del dominio.
7. **Admin / dashboards al final**: dependen de los datos que muestran.
8. **Refactors visuales / UI restyle al final del todo**: requieren producto estable.

### Reglas para GATES de paralelismo

Un GATE se genera cuando completar un change (o grupo de changes) **desbloquea múltiples changes que no dependen entre sí**.

Formato exacto:

```
GATE N: C-XX nombre-del-change ✓     ← anotación opcional: PRIMER FORK
  → C-YY nombre-del-change             [Agente A]
  → C-ZZ nombre-del-change             [Agente B]
  → C-WW nombre-del-change             [Agente C — si C-MM ✓]
```

- Si solo hay un change paralelo en el gate, aún lo incluís (aclara el punto de sync).
- Si un change dentro del gate depende adicionalmente de otro, anotalo: `[Agente C — si C-NN ✓]`.
- Marcá `← PRIMER FORK` en el gate donde se abren 3+ ramas paralelas por primera vez.

> **Proyectos pequeños (<5 changes):** Si no hay forks reales de paralelismo, omití la sección GATES y Plan con N agentes. Escribí en su lugar: `> Este proyecto tiene pocas dependencias lineales. No se identificaron oportunidades de paralelización significativa.`

### Reglas para Camino crítico

- La **cadena lineal más corta** desde `C-01` hasta el último change indispensable para producción.
- **No incluye** refactors visuales, dashboards extra, ni features opcionales.
- Si dos changes pueden ser el "último", anotá ambos con asterisco `*`.
- Formato: `C-01 → C-02 → C-03 → C-NN`

### Reglas para Plan con N agentes

- `N` se ajusta al tamaño real: 1 agente si < 4 changes, 2 agentes si 4-7 changes, 3 agentes si 8+ changes.
- Tabla: `Paso | Agente A (Backend Core) | Agente B (Backend Aux) | Agente C (Frontend)`
- Si un agente queda libre en un paso, marcá con `—`.
- Apuntá a que los agentes terminen aproximadamente al mismo tiempo.

### Reglas para Governance

| Nivel | Cuándo |
|-------|--------|
| **BAJO** | Scaffolding, CRUDs simples, pages frontend sin lógica crítica, configuración. |
| **MEDIO** | Flujos con estado, sesiones, máquinas de estado, eventos WebSocket no críticos. |
| **ALTO** | Sistemas de notificaciones, gestión de roles, WS gateway, observabilidad. |
| **CRITICO** | Auth, pagos, datos de seguridad, audit trail, modelos core que todo lo demás referencia. |

### Reglas para "Leer antes"

- Listá **3 a 5 archivos** de la KB por change. Pueden repetirse entre changes — es intencional.
- Incluí sección específica cuando aplique (`§2.1`, `§Auth`, `§Round lifecycle`).
- Priorizar:
  - Auth → `03_actores_y_roles.md`, `05_reglas_de_negocio.md §Auth`.
  - Entidades → `04_modelo_de_datos.md §entidad`.
  - Endpoints → `07_flujos_principales.md`, archivo de API si existe.
  - Frontend → `08_arquitectura_propuesta.md §frontend`, archivo de convenciones.
  - Pagos / integraciones → `05_reglas_de_negocio.md §Pagos`.

### Reglas para Scope

Bullets **operacionales**, no descriptivos. Cada bullet describe algo que el agente va a generar:

- ✅ `POST /api/auth/login — JWT access + refresh, rate limiting 5/60s por IP+email`
- ❌ `Sistema de autenticación completo con tokens`

- ✅ `Migración 003: tablas allergen, product_allergen`
- ❌ `Crear las tablas de alérgenos`

Siempre mencioná explícitamente: modelos/entidades nuevas, endpoints clave, migración numerada, eventos WS si existen, tests esperados.

---

## Workflow

1. Detectar modo de operación (orquestado vs standalone).
2. Detectar si `CHANGES.md` ya existe → activar Modo Update si corresponde.
3. Ejecutar pre-checks según el modo (ver §Pre-checks).
4. Leer los nodos de la KB según el modo (ver §Input).
5. Identificar capacidades atómicas del sistema (cada una será un `C-NN`).
6. Asignar `C-NN` secuenciales agrupados en FASEs semánticas.
7. Inferir dependencias aplicando las 8 reglas.
8. Calcular GATES de paralelismo (omitir si <5 changes).
9. Identificar camino crítico.
10. Diseñar plan con N agentes (N ajustado al tamaño).
11. Para cada change: escribir Estado, Scope, Dependencias, Governance, Leer antes.
12. Escribir `CHANGES.md` en la raíz, incluyendo la tabla **Resumen** al final.
13. En Modo Update: restaurar checkboxes preservados, agregar sección de changes eliminados si aplica.
14. **Si orquestado**: ejecutar el hook de estado → actualizar `state.roadmap`. Ver [`assets/state-contract.md`](assets/state-contract.md).
15. Emitir el bloque de cierre al usuario.

### Output al usuario al cerrar

```markdown
## ✅ atlas — CHANGES.md generado

`CHANGES.md` creado en la raíz con **{N} changes** en **{M} fases**.

| Métrica | Valor |
|---------|-------|
| Camino crítico | {K} changes |
| Gates de paralelismo | {G} |
| Governance CRITICO | {X} changes |
| Primer change recomendado | `C-01` ({nombre}) |

Para arrancar: `/opsx:propose C-01-{nombre}`
```

Si fue Modo Update, agregar:

```markdown
**Checkboxes preservados**: {P} de {T} anteriores.
**Changes eliminados con progreso**: {lista o "ninguno"}.
```

---

## State integration (orchestrated)

Hook **condicional y standalone-safe**: solo corre si `.jr-orchestrator-state.json` existe con `version == 2`.

**Qué escribe**: solo `state.roadmap` — nunca toca `step`, `owner`, `kb`, `skills`, ni `agents`.

| Campo | Valor |
|-------|-------|
| `created_by` | `"atlas"` |
| `source` | `"CHANGES.md"` |
| `changes` | lista de IDs `C-NN` generados, en orden |

Algoritmo completo, manejo de JSON malformado, y fallback: [`assets/state-contract.md`](assets/state-contract.md).

---

## Resources

- **Template**: [`assets/changes-template.md`](assets/changes-template.md) — plantilla completa con ejemplos.
- **State contract**: [`assets/state-contract.md`](assets/state-contract.md) — schema, algoritmo condicional, reglas de input orquestado.
