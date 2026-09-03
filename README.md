# atlas

Skill para generar **`CHANGES.md`** — el índice operativo y canónico de todos los changes necesarios para implementar un sistema desde cero hasta producción. Sucesor de `roadmap-generator` (v2 → v3).

---

## ¿Qué hace?

Lee tu base de conocimiento estructurada en `knowledge-base/` y genera `CHANGES.md` en la raíz del proyecto con:

- **Árbol de dependencias** visual (ASCII art jerárquico).
- **Paralelismo por fase** con gates explícitos y asignación sugerida de agentes.
- **Camino crítico**: mínimo irreducible de changes para llegar a producción.
- **Plan óptimo con N agentes**: tabla de asignaciones ajustada al tamaño del proyecto.
- **Por cada change**: estado checkbox, scope operacional, dependencias, nivel de governance y archivos KB a leer antes.
- **Tabla Resumen** al final: métricas del plan generado de un vistazo.

Soporta dos modos de operación:

| Modo | Cuándo | Comportamiento |
|------|--------|----------------|
| **Fire-and-forget** | `CHANGES.md` no existe | Genera el archivo limpio. |
| **Merge-safe (Update)** | `CHANGES.md` ya existe | Regenera preservando los checkboxes `[x]` del archivo anterior. |

---

## ¿Por qué no es solo "un roadmap"?

Un roadmap clásico es informativo (qué viene y en qué orden). `CHANGES.md` es **operativo**.

| Aspecto | Roadmap simple | CHANGES.md (atlas) |
|---------|---------------|---------------------|
| Dependencias | Tabla 1-a-1 | Árbol jerárquico + gates |
| Paralelización | Implícita o nula | Explícita, asignada a agentes (N ajustado al proyecto) |
| Priorización ante tiempo limitado | Difícil de inferir | Camino crítico marcado |
| Contrato con la KB | Implícito | Sección "Leer antes" por change |
| Nivel de riesgo | No declarado | Governance: BAJO / MEDIO / ALTO / CRITICO |
| Scope por change | Descriptivo | Operacional (modelos, endpoints, migraciones, tests) |
| Tracking de progreso | No | Checkboxes `[ ]` / `[x]` preservados en regeneraciones |
| Métricas del plan | No | Tabla Resumen al final |

---

## Pre-requisitos

Antes de invocar atlas necesitás:

1. **Base de conocimiento** en `knowledge-base/` (raíz). Si no la tenés, generala con la skill [`chronicle`](https://github.com/gentleman-programming/chronicle).
   - atlas no requiere los 10 nodos canónicos — solo los que aplican al perfil de tu proyecto.
   - Mínimo requerido: `04_modelo_de_datos.md` y `08_arquitectura_propuesta.md`.

2. **OpenSpec inicializado** en el proyecto:
   ```bash
   npx @fission-ai/openspec@latest init
   ```
   - atlas verifica que `openspec/` exista y contenga al menos un archivo de configuración.
   - Si no usás OpenSpec, indicáselo al agente y atlas generará el plan sin referencias a `/opsx:propose`.

Si falta la KB, atlas se detiene y avisa. Si falta OpenSpec y no se indicó omisión, también se detiene.

---

## Instalación

```bash
npx skills add https://github.com/gentleman-programming/atlas
```

---

## Uso

```
Tu repo:
proyecto/
├── docs/                        # Documentos fuente
├── knowledge-base/              # KB generada por chronicle
└── openspec/                    # OpenSpec inicializado

Le decís al agente:
"generá el CHANGES.md del proyecto"
"regenerá el CHANGES.md, cambié la KB"
```

→ El agente lee la KB y escribe (o actualiza) `CHANGES.md` en la raíz.

---

## Estructura del output

```markdown
# CHANGES — Secuencia de Implementación

> Índice canónico de todos los changes del proyecto X.

## Cómo usar este documento
(5 pasos)

## Árbol de dependencias
(ASCII art jerárquico)

### Paralelismo por fase
(GATES con asignación a agentes)

### Camino crítico
(cadena lineal irreducible)

### Plan óptimo con N agentes
(tabla paso × agente; N ajustado al tamaño del proyecto)

## FASE 0 — Cimientos
### [C-01] `foundation-setup`
- Estado: [ ] pendiente
- Scope: bullets operacionales
- Dependencias: ninguna
- Governance: BAJO
- Leer antes: ...

## FASE 1A — Autenticación
### [C-03] `auth`
- ...

## Resumen
| Total changes | 9 |
| Fases         | 4 |
| ...           | . |
```

---

## Reglas que aplica para inferir dependencias

1. **Infra primero**: C-01 nunca depende de nada.
2. **Modelos core antes que features**: C-02 es típicamente `core-models`.
3. **Auth antes que recursos protegidos**.
4. **Entidad referenciada antes que la que referencia** (`categorías` antes que `productos`).
5. **Backend antes que frontend acoplado**.
6. **Integraciones externas (pagos, webhooks) al final**.
7. **Admin / dashboards al final** (dependen de datos que muestran).
8. **Refactors visuales / UI restyle al final del todo** (requieren producto estable).

## Niveles de governance

| Nivel | Cuándo se usa |
|-------|---------------|
| **BAJO** | Scaffolding, CRUDs simples, pages frontend sin lógica crítica. |
| **MEDIO** | Flujos con estado, sesiones, máquinas de estado, WS no críticos. |
| **ALTO** | Notificaciones, gestión de roles, WS gateway, observabilidad. |
| **CRITICO** | Auth, pagos, datos de seguridad, audit trail, modelos core. |

---

## Output al cerrar

```
✅ atlas — CHANGES.md generado

CHANGES.md creado en la raíz con 9 changes en 4 fases.

| Métrica              | Valor                          |
|----------------------|--------------------------------|
| Camino crítico       | 5 changes                      |
| Gates de paralelismo | 5                              |
| Governance CRITICO   | 2 changes                      |
| Primer change        | C-01 (foundation-setup)        |

Para arrancar: /opsx:propose C-01-foundation-setup
```

Si fue una regeneración (Modo Update):
```
Checkboxes preservados: 3 de 4 anteriores.
Changes eliminados con progreso: C-07 (ingredients).
```

---

## Changelog

### v3.0 (atlas)
- Renombrado de `roadmap-generator` a `atlas`.
- **Fix**: referencias a `kb-creator` reemplazadas por `chronicle` (skill correcta).
- **Fix**: pre-check de "10 canónicos" reemplazado por validación de perfil — ya no falla en KBs generadas con chronicle que omiten slots intencionalmente.
- **Fix**: modo Update merge-safe — regenerar preserva checkboxes `[x]` del archivo anterior.
- **Fix**: lógica orquestada self-contained en SKILL.md (ya no requiere leer state-contract.md para el path crítico).
- **Fix**: verificación de existencia en disco de archivos de `state.kb.files` en modo orquestado.
- **Fix**: pre-check de `openspec/` es contextual — omitible si el usuario no usa OpenSpec.
- **Fix**: manejo de JSON malformado o vacío en el algoritmo de escritura del estado.
- **Fix**: plan con N agentes ajustado al tamaño del proyecto (no siempre 3).
- **Fix**: sección GATES y plan multi-agente omitidos en proyectos con <5 changes.
- **Fix**: tabla Resumen definida y obligatoria en el output.
- **Fix**: nodos "obligatorios" ausentes notifican y continúan en vez de fallar silenciosamente.
- **Fix**: formato del header `### [C-NN] \`nombre\`` unificado entre SKILL.md y template.

### v2.0 (roadmap-generator)
- Versión inicial con estado integrado para jr-orchestrator.

---

## Licencia

Apache-2.0
