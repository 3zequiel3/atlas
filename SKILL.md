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
  "índice de changes", "regenerar CHANGES.md", "actualizar CHANGES.md",
  "generate CHANGES.md", "regenerate the roadmap", "what changes do I need".
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "3.2"
---

## When to Use

- Generar el **índice canónico** de changes para implementar un sistema desde cero hasta producción.
- Convertir una base de conocimiento estructurada en un plan operativo con paralelización explícita.
- Identificar **camino crítico**, **niveles de governance**, y **contratos con la KB** por change.
- **Regenerar** un `CHANGES.md` existente sin perder el estado de progreso.

**Don't use when:**
- No hay KB disponible: ni `knowledge-base/` en raíz, ni `state.kb.files` en un `.jr-orchestrator-state.json` v2. Corré primero la skill `chronicle`.
- El usuario quiere tocar **un solo change** de un `CHANGES.md` existente. Si el pedido es ambiguo entre "regenerá todo" y "cambiá este change", **preguntá antes de regenerar** — regenerar reescribe el archivo entero.

---

## Contrato de fuentes

Tres archivos, una responsabilidad cada uno. **No dupliques contenido entre ellos.**

| Archivo | Manda sobre |
|---------|-------------|
| `SKILL.md` (este) | Pre-checks, modos, reglas de inferencia, workflow. Contiene el path crítico completo. |
| [`assets/changes-template.md`](assets/changes-template.md) | **La forma exacta del output** + la checklist de validación. |
| [`assets/state-contract.md`](assets/state-contract.md) | Racional del schema orquestado y casos borde. Opcional. |

---

## Pre-checks obligatorios

Corré estos checks **antes** de leer la KB. Si un check marcado **HALT** falla, detené la ejecución con su mensaje. No generes nada parcialmente.

| # | Check | Si falla |
|---|-------|----------|
| 1 | Hay una KB accesible: `knowledge-base/` en raíz **o** `state.kb.files` no vacío | **HALT** — "Falta la KB. Corré primero la skill `chronicle` para generarla." |
| 2 | Existen `04_modelo_de_datos.md` **y** `08_arquitectura_propuesta.md` | **HALT** — "KB incompleta para atlas. Faltan los nodos mínimos: [lista]. Completá la KB con `chronicle`." |
| 3 | Existe `openspec/` en raíz, y adentro hay `openspec.config.*`, `config.yaml`, `config.json` o el directorio `specs/` | **HALT** — "OpenSpec no inicializado. Corré `npx @fission-ai/openspec@latest init` en la raíz del proyecto." |

**Check 2 vale en los dos modos.** En orquestado se evalúa contra los basenames de `state.kb.files`.

> **Solo 04 y 08 son bloqueantes.** `06_funcionalidades.md` y `07_flujos_principales.md` mejoran mucho el resultado pero no detienen. `chronicle` omite nodos a propósito según el perfil del sistema — nunca falles porque el total de nodos sea menor a 10.

> **Check 3 — cuándo no aplica.** Si el usuario indica explícitamente que no usa OpenSpec, o si estás en **modo orquestado** (no hay humano a quien preguntar) y `openspec/` no existe: salteá el HALT y generá en **modo sin OpenSpec** (ver §Modo sin OpenSpec). El HALT solo corre en standalone interactivo.

---

## Modo de operación

Detectá los dos ejes antes de generar.

**Eje 1 — ¿de dónde salen los archivos de la KB?**

- `.jr-orchestrator-state.json` con `version == 2` y `state.kb.files` no vacío → **input orquestado**.
- Cualquier otro caso → **input standalone** (glob de `knowledge-base/`).

**Eje 2 — ¿`CHANGES.md` ya existe en la raíz?**

- No → generación limpia.
- Sí → **Modo Update** (ver §Modo Update, que incluye el guard de archivo ajeno).

> **El hook de estado no depende del eje 1.** Corre siempre que exista `.jr-orchestrator-state.json` con `version == 2`, incluso si el input cayó a standalone por `kb.files` vacío. Si no, el orquestador espera para siempre una señal que nunca llega. Ver §State integration.

---

## Input — qué leer de la KB

### Standalone

Leé de `knowledge-base/`:

| Nodo | Para qué | Si falta |
|------|----------|----------|
| `04_modelo_de_datos.md` | Entidades y relaciones → orden de creación de tablas | HALT (pre-check 2) |
| `08_arquitectura_propuesta.md` | Patrones → infraestructura previa necesaria | HALT (pre-check 2) |
| `06_funcionalidades.md` | US y épicas → unidad de cada change | Avisá y seguí |
| `07_flujos_principales.md` | Flujos E2E → changes atómicos vs compuestos | Avisá y seguí |
| `03_actores_y_roles.md` | Changes de auth y RBAC | Usalo si está, sin aviso |
| `05_reglas_de_negocio.md` | Reglas que cruzan changes → aristas de dependencia y bullets de Scope | Usalo si está, sin aviso |
| `10_preguntas_abiertas.md` | Dependencias inciertas → bullet `⚠ incierto:` en el Scope del change afectado | Usalo si está, sin aviso |

**Formato único de aviso**, en los dos modos: `⚠ nodo ausente: {path} — continuando sin él`.

> Avisá **solo** por `06` y `07`: su ausencia cambia la calidad del output. Los tres opcionales se omiten en perfiles enteros de `chronicle` (`cli`, `library_sdk`, `data_pipeline`), así que avisar por ellos convierte una KB correcta en una pared de warnings falsos.

### Orquestado

1. Leé `state.kb.files` como **lista autoritativa de paths**.
2. **Antes de leer cada archivo**, verificá que existe en disco. Si falta, emití el aviso y seguí con los que sí están.
3. Aplicá el pre-check 2 contra los basenames de esa lista.
4. Leé `state.kb.discovery` como **hint de nombrado, nunca como fuente de verdad**:
   - `needs_infra: false` → el sistema ya tiene infraestructura; `C-01` es el primer change de dominio en vez de `foundation-setup`.
   - `system_type` → nombrá las FASEs y los roles de agente acorde al dominio.
   - `domain` → inferí changes específicos del dominio.
   - `scale` → inferí si se necesitan changes de RBAC o multi-tenancy.

> **Paths en "Leer antes" (orquestado):** usá los paths **reales** de `state.kb.files`, no `knowledge-base/` hardcodeado. Si el orquestador apunta a `docs/kb/`, cada puntero del output tiene que decir `docs/kb/...`.

---

## Formato del output

**La forma exacta de `CHANGES.md` vive en [`assets/changes-template.md`](assets/changes-template.md). Leelo antes de escribir.** Las reglas de abajo definen *qué* poner en cada sección; el template define *cómo* se ve.

Secciones de primer nivel, en este orden fijo:

1. `# CHANGES — Secuencia de Implementación` + blockquote de intro
2. `## Cómo usar este documento`
3. `## Árbol de dependencias`
4. `## FASE {N} — {Nombre semántico}` × M
5. `## ⚠️ Changes eliminados (tenían progreso)` — **solo en Modo Update con bajas**
6. `## Resumen` — **siempre la última sección del archivo**

No agregues ni quites secciones fuera de esa lista. La #5 es la única condicional.

Dentro de `## Árbol de dependencias` van tres `###`, y **sus paréntesis son parte del encabezado**:

- `### Paralelismo por fase`
- `### Camino crítico ({K} changes — cadena más larga)`
- `### Plan óptimo con {N} agentes`

Los dos últimos se omiten juntos cuando no hay gates (ver §Reglas para GATES).

### Reglas para nombrar changes

- Códigos secuenciales `C-01`, `C-02`, ..., `C-NN`. **Siempre padding de 2 dígitos.**
- **Orden de asignación**: topológico. Invariante verificable: **el ID de un change es siempre mayor que el de todas sus dependencias.** Recorré las FASEs en orden y numerá dentro de cada una respetando esa condición.
- Nombre en kebab-case, descriptivo, sin prefijos numéricos ni `us-NNN-`.
- **Los nombres son únicos dentro del archivo.** Si dos changes quedan con el mismo nombre, desambiguá por dominio (`orders-api` / `orders-ui`), nunca por número.
- **El nombre es la identidad del change**, no el ID. Los IDs se reasignan en cada regeneración; los nombres persisten.
- Si un change cubre varias US, mencionalas en el **Scope** (no en el nombre).

### Reglas para inferir dependencias

Jerarquía obligatoria. Cuando dos reglas aplican al mismo change, **gana la de número más bajo**:

1. **Infra primero**: si el proyecto necesita scaffolding, `C-01` es `foundation-setup` y no depende de nada.
2. **Modelos core antes que features**: entidades base, mixins, repositorios.
3. **Auth antes que recursos protegidos**: cualquier change que requiera usuario logueado.
4. **Entidad referenciada antes que la que referencia**: `productos` antes que `pedidos`.
5. **Backend antes que frontend acoplado**: si una vista consume un endpoint, depende del change del endpoint.
6. **Integraciones externas / pagos / webhooks al final**: dependen de las entidades del dominio.
7. **Admin / dashboards al final**: dependen de los datos que muestran.
8. **Refactors visuales / UI restyle al final del todo**: requieren producto estable.

### Reglas para FASEs

- Una FASE agrupa changes que **comparten tema** y caen en la misma franja del grafo. El nombre semántico es obligatorio (`## FASE 2 — Dominio principal`, nunca `## FASE 2`).
- **Restricción dura**: ninguna FASE puede contener un change que dependa de otro de una FASE **posterior**. Las FASEs están topológicamente ordenadas — de eso depende que el invariante de IDs sea satisfacible.
- Numeración entera desde `0`, sin sufijos de letra. Dos grupos paralelos son **dos FASEs**, no `1A`/`1B`.
- Una FASE puede llevar un blockquote opcional de una línea aclarando su paralelismo interno.
- La métrica `Fases` del Resumen = cantidad de encabezados `## FASE`.

### Reglas para GATES de paralelismo

Un GATE es un **punto de sincronización real**. Emitilo cuando, y solo cuando:

- **Fork** — completar un change desbloquea **2 o más** changes que no dependen entre sí, o
- **Join** — un change requiere que **2 o más** changes converjan.

Los tramos lineales (un change desbloquea exactamente uno) **no son gates**: ya están en el árbol.

- Numerá desde `GATE 1`, en **orden topológico** del change que los dispara.
- `← PRIMER FORK` va en el primer gate que abre **3 o más** ramas. Si ningún gate abre 3+, la anotación no se usa.
- Si hay Plan con N agentes, cada change del gate lleva su `[Agente X]`, **coherente con la tabla del plan**. Si no hay plan, se omiten las etiquetas.
- La métrica `Gates de paralelismo` del Resumen = cantidad de bloques `GATE`.
- Si no hay ningún fork ni join, omití `### Camino crítico` y `### Plan óptimo` y escribí en su lugar:
  `> Este proyecto tiene dependencias lineales. No se identificaron oportunidades de paralelización.`

### Reglas para Camino crítico

- Es la **cadena de dependencias más larga** del grafo, desde un change sin dependencias hasta un change del que nada depende. Es la cota inferior de la duración del proyecto: por definición **no puede saltear una dependencia**.
- Se calcula sobre el **grafo completo**, sin excluir nada por criterio subjetivo. Si una feature es opcional, la decisión correcta es no incluirla como change — no filtrarla acá.
- Verificá que cada flecha del resultado sea una arista real del árbol (una anotación `← + C-XX` cuenta como arista).
- Si dos cadenas empatan en largo, elegí la que contiene más changes `CRITICO`; si sigue el empate, la que termina en el ID más bajo. Una sola cadena, sin ramas ni asteriscos.

### Reglas para Plan con N agentes

- Solo si hay al menos un GATE.
- `N` arranca en **2** si hay menos de 8 changes, **3** si hay 8 o más.
- **Después, acotá `N` por el ancho máximo del grafo**: si nunca hay más de 2 changes simultáneamente desbloqueados, `N = 2` aunque haya 20 changes. El ancho manda sobre el conteo — ninguna columna puede quedar vacía todo el plan.
- Los roles salen del sistema real, no de una lista fija: `Backend Core / Backend Aux / Frontend` sirve para una web app; en un CLI o un SDK usá `Core / Comandos / Docs+CI` o lo que corresponda.
- Si un agente queda libre en un paso, marcá con `—`.
- El plan **nunca puede tener menos pasos que el camino crítico**. Puede tener más: con `N` acotado, varios changes listos compiten por el mismo agente. No fuerces que coincidan.

### Reglas para Governance

| Nivel | Cuándo |
|-------|--------|
| **BAJO** | Scaffolding, CRUDs simples sin aislamiento, pages frontend sin lógica crítica, configuración. |
| **MEDIO** | Flujos con estado, sesiones, máquinas de estado, aislamiento multi-tenant, eventos WebSocket no críticos. |
| **ALTO** | Sistemas de notificaciones, gestión de roles, WS gateway, observabilidad. |
| **CRITICO** | Auth, pagos, datos de seguridad, audit trail, modelos core que todo lo demás referencia. |

> Si el Scope de un change **implementa** control de acceso, manejo de secretos o aislamiento de datos entre tenants, no puede ser BAJO. Consumir una convención existente (un `.env.example`, un decorador de permisos ya construido) no sube el nivel.

### Reglas para "Leer antes"

- Apuntá a **3 a 5 archivos** de la KB por change. Pueden repetirse entre changes — es intencional.
- **Verificá en disco que cada path exista antes de escribirlo.** Nunca apuntes a un nodo que no leíste.
- **La KB manda sobre el piso**: si la KB solo tiene los 2 nodos mínimos, listá esos 2. Nunca inventes un path para llegar a 3. El piso aplica solo hasta donde llegue la KB real.
- Incluí la sección específica cuando aplique (`§2.1`, `§Auth`, `§Catálogo`).
- Priorizar: auth → `03` + `05 §Auth`; entidades → `04 §entidad`; endpoints → `07` + `04`; frontend → `08 §frontend`; pagos e integraciones → `05 §Pagos` + `07`.

### Reglas para Scope

Bullets **operacionales**, no descriptivos. Cada bullet describe algo que el agente va a generar:

- ✅ `POST /api/auth/login — JWT access + refresh, rate limiting 5/60s por IP+email`
- ❌ `Sistema de autenticación completo con tokens`
- ✅ `Migración 003: tablas allergen, product_allergen`
- ❌ `Crear las tablas de alérgenos`

Siempre mencioná explícitamente: modelos/entidades nuevas, endpoints clave, migración numerada, eventos WS si existen, tests esperados. Si `10_preguntas_abiertas.md` marca una incógnita que afecta al change, agregá un bullet `⚠ incierto: {pregunta}`.

---

## Modo Update — merge-safe

Se activa cuando `CHANGES.md` ya existe en la raíz.

### Guard de archivo ajeno

`CHANGES.md` es un nombre de changelog común. **Antes de tocar nada**, verificá que el archivo sea de atlas: tiene el header `# CHANGES — Secuencia de Implementación` **o** al menos un encabezado `### [C-NN]`.

Si no lo es, **detené** y preguntá:
`"Ya existe un CHANGES.md que no parece generado por atlas. ¿Lo reemplazo, o preferís otro nombre de archivo?"`
Nunca lo sobrescribas por tu cuenta: sería la destrucción silenciosa e irrecuperable de un archivo ajeno.

### Formato del Estado

**El estado de un change es un task item real de GFM**, para que renderice y se pueda clickear en GitHub:

```
- [ ] **Estado**        ← pendiente
- [x] **Estado**        ← completado
```

**Regla de asociación:** un `[x]` pertenece al change del encabezado `### [C-NN]` inmediatamente anterior. Ignorá cualquier `[x]` que no esté en esa posición.

**Compatibilidad hacia atrás:** las versiones ≤ 3.0 escribían el estado como `` - **Estado**: `[x]` completado ``. Al **leer**, aceptá las dos formas. Al **escribir**, emití siempre el task item de GFM. Sin esto, la primera regeneración sobre un archivo viejo matchea cero checkboxes y reporta "0 de 0" — pierde todo el progreso y encima afirma que no había nada.

### Algoritmo

Todo en memoria, **un solo write al disco**:

1. Leé el `CHANGES.md` actual. Para cada change con el estado en `[x]`, guardá el par **`{nombre, ID}`** — el nombre es la clave de matching, el ID solo se usa para reportar bajas.
2. Generá el nuevo contenido completo en memoria, con el `## Resumen` al final.
3. Para cada nombre de ese conjunto que **existe en el contenido nuevo**, poné su Estado en `[x]`. El matching es **por nombre, nunca por `C-NN`** — los IDs se reasignan en cada regeneración y matchear por ID marca como hecho trabajo que nadie tocó.
4. Para cada nombre que estaba en `[x]` y **no existe** en el contenido nuevo, agregá su fila a `## ⚠️ Changes eliminados (tenían progreso)`, insertada **inmediatamente antes de `## Resumen`**.
5. **Recién ahora** escribí el archivo, una sola vez, con los checkboxes ya restaurados. Nunca escribas la versión sin restaurar: entre ese write y el merge, el progreso no existiría en ningún lado.
6. Al cerrar, reportá cuántos se preservaron y cuántos se perdieron por restructuración.

La sección de bajas es una tabla de dos columnas:

```markdown
## ⚠️ Changes eliminados (tenían progreso)

| Change | ID anterior |
|--------|-------------|
| `ingredients` | `C-07` |
```

---

## Modo sin OpenSpec

Si el usuario optó por no usar OpenSpec, o si estás orquestado sin `openspec/`, el output cambia así — y solo así:

- El blockquote de intro pierde la línea de `/opsx:propose`.
- Se agrega `> Nota: generado sin integración OpenSpec.` justo debajo del blockquote de intro.
- `## Cómo usar este documento` tiene **4 pasos** en vez de 5:
  1. Verificar que las dependencias del change target estén en `[x]`.
  2. Leer los docs de la KB indicados en "Leer antes" del change.
  3. Implementar el change.
  4. Marcar el checkbox `[x]` en este archivo.
- El bloque de cierre omite la línea `Para arrancar:`. La fila `Primer change recomendado` de la tabla ya dice lo mismo.

Todo lo demás — árbol, gates, camino crítico, campos por change, Resumen — es idéntico.

---

## Workflow

1. Detectar los dos ejes de modo (§Modo de operación).
2. Ejecutar los pre-checks (§Pre-checks). HALT si corresponde.
3. Si `CHANGES.md` existe: correr el **guard de archivo ajeno** (§Modo Update). Detené y preguntá si falla.
4. Leer los nodos de la KB según el modo (§Input).
5. Si es Modo Update: extraer los pares `{nombre, ID}` de los changes en `[x]`, aceptando el formato legacy.
6. Identificar capacidades atómicas — cada una será un `C-NN`. Una capacidad que no entra en una sesión de agente se parte en dos changes.
7. Agrupar en FASEs semánticas topológicamente ordenadas y asignar `C-NN`.
8. Inferir dependencias aplicando las 8 reglas por prioridad.
9. Calcular GATES (solo forks y joins reales).
10. Calcular el camino crítico como la cadena **más larga**, y verificar cada arista contra el árbol.
11. Diseñar el plan con N agentes, si hay al menos un GATE.
12. Para cada change: Estado, Scope, Dependencias, Governance, Leer antes — verificando en disco cada path de "Leer antes".
13. Componer el contenido completo en memoria, con el Resumen como última sección.
14. Si es Modo Update: aplicar los checkboxes preservados, e insertar la sección de bajas **antes del Resumen**.
15. **Correr la checklist de validación** de [`assets/changes-template.md`](assets/changes-template.md) contra el contenido en memoria. Corregí lo que falle antes de seguir.
16. Escribir `CHANGES.md` en la raíz — **un solo write**.
17. Ejecutar el hook de estado (§State integration).
18. Emitir el bloque de cierre.

### Bloque de cierre

```markdown
## ✅ atlas — CHANGES.md generado

`CHANGES.md` {creado|actualizado} en la raíz con **{N} changes** en **{M} fases**.

| Métrica | Valor |
|---------|-------|
| Camino crítico | {K} changes |
| Gates de paralelismo | {G} |
| Governance CRITICO | {X} changes |
| Primer change recomendado | `C-01` ({nombre}) |

Para arrancar: `/opsx:propose {nombre}`
```

El comando usa el **nombre kebab-case pelado**, sin prefijo `C-NN-`: es el mismo identificador que usa `## Cómo usar este documento` y el que va a quedar en `openspec/changes/archive/`. En modo sin OpenSpec, omití esa línea.

Si fue Modo Update, agregá:

```markdown
**Checkboxes preservados**: {P} de {T} anteriores.
**Changes eliminados con progreso**: {lista de nombres o "ninguno"}.
```

Si el hook de estado encontró un `roadmap.created_by` que no es `"atlas"`, sumá una línea:
`**Nota**: se reemplazó un state.roadmap creado por \`{owner}\`.`

---

## State integration (orquestado)

Hook **condicional y standalone-safe**. Corre siempre que exista `.jr-orchestrator-state.json` con `version == 2` — **también si el input cayó a standalone** por `kb.files` vacío.

```
1. Buscar `.jr-orchestrator-state.json` en la raíz del proyecto.
   ├─ AUSENTE → no-op. NO lo crees. Listo.
   ├─ PRESENTE pero vacío o JSON malformado →
   │    "⚠ No se pudo parsear el archivo de estado — se omite la actualización." Listo.
   ├─ PRESENTE con version != 2 →
   │    "⚠ El archivo de estado no es v2 — se omite la actualización." Listo.
   └─ PRESENTE con version == 2 →
        a. Re-leé el archivo completo AHORA (el orquestador pudo escribir mientras generabas).
        b. Si `roadmap.created_by` existe y no es "atlas", anotalo para el bloque de cierre.
        c. Reemplazá únicamente el valor de la clave `roadmap`.
        d. Conservá toda otra clave con su valor intacto, incluidas las que no conocés.
        e. Serializá y escribí el archivo de vuelta.
```

**Qué escribe**: solo `state.roadmap`. Nunca toca `step`, `owner`, `kb`, `skills`, ni `agents`.

| Campo | Valor |
|-------|-------|
| `created_by` | `"atlas"` |
| `source` | `"CHANGES.md"` |
| `changes` | lista de objetos `{"id": "C-01", "name": "foundation-setup"}`, en orden. El `id` es posicional y solo vale para esta generación; el `name` es la identidad estable. |
| `preserved_checkboxes` | cantidad de `[x]` restaurados (0 en la primera corrida) |
| `eliminated_with_progress` | nombres que tenían `[x]` y desaparecieron (`[]` si ninguno) |

**Nunca lo crees**: el orquestador es el único creador. atlas solo actualiza, así dos escritores no compiten por crearlo.

Racional del schema y casos borde: [`assets/state-contract.md`](assets/state-contract.md).
