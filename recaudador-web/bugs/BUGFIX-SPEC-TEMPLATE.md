# 🐛 Bugfix Spec: [TITULO_CORTO]

> **ID:** BUG-[XXXX]
> **Fecha:** [YYYY-MM-DD]
> **Proyecto:** [nombre-proyecto]
> **Prioridad:** [critical | high | medium | low]
> **Estado:** [draft | ready | in-progress | resolved]

---

## 1. Resumen

<!-- Una oración clara describiendo el bug. Si no podés explicarlo en una oración, probablemente son dos bugs. -->

[Componente X falla cuando Y bajo la condición Z]

---

## 2. Contexto

### Archivos involucrados

| Archivo | Rol |
|---------|-----|
| `src/components/...` | Componente afectado |
| `src/adapters/...` | Adapter/lógica relacionada |
| `src/store/...` | Estado de Redux (si aplica) |

### Dependencias clave

<!-- Qué otros componentes, hooks, stores o servicios están conectados al área del bug -->

- Componente padre: `...`
- Store/slice: `...`
- Hooks: `...`
- Servicios/API: `...`

---

## 3. Reproducción

### Pasos

1. Ir a `[ruta/pantalla]`
2. Realizar `[acción específica]`
3. Observar `[comportamiento actual]`

### Comportamiento actual

<!-- Qué pasa ahora (el bug) -->

```
[Error message, console output, o descripción del fallo]
```

### Comportamiento esperado

<!-- Qué debería pasar -->

[Descripción clara del resultado correcto]

---

## 4. Análisis de causa raíz

<!-- Completar durante investigación. El agente debe llenar esta sección antes de codear. -->

### Hipótesis

- [ ] [Hipótesis 1: ej. "El adapter no maneja undefined"]
- [ ] [Hipótesis 2: ej. "Race condition en el dispatch"]

### Causa confirmada

<!-- Se llena cuando se identifica la causa real -->

```
[Explicación técnica de la causa raíz]
```

**Archivo raíz:** `[path al archivo donde está la causa]`
**Línea(s):** `[rango de líneas]`

---

## 5. Plan de fix

### Estrategia

<!-- Describir el approach a alto nivel -->

[Ej: "Agregar validación de null check en el adapter antes de mapear los items"]

### Cambios requeridos

<!-- Lista precisa de qué modificar. Cada cambio = una responsabilidad. -->

| # | Archivo | Cambio | Principio |
|---|---------|--------|-----------|
| 1 | `src/...` | [descripción del cambio] | SRP / OCP |
| 2 | `src/...` | [descripción del cambio] | SRP / OCP |

### Restricciones

<!-- Qué NO se debe tocar o cambiar -->

- ❌ No modificar la interfaz pública del componente
- ❌ No alterar el shape del store
- ❌ No cambiar archivos fuera del scope listado
- ✅ [Cualquier libertad permitida]

---

## 6. Criterios de aceptación

<!-- Cada criterio debe ser verificable. El agente valida contra esta lista. -->

- [ ] El bug original ya no se reproduce siguiendo los pasos de §3
- [ ] No hay errores en consola
- [ ] No hay regresiones en funcionalidades relacionadas
- [ ] [Criterio específico del fix]
- [ ] [Criterio específico del fix]

---

## 7. Testing

### Verificación manual

```
1. [Paso de verificación manual]
2. [Paso de verificación manual]
```

### Tests automatizados (si aplica)

| Test | Descripción | Estado |
|------|-------------|--------|
| `[test file]` | [qué valida] | [ ] pendiente |

---

## 8. Resolución

<!-- Se completa DESPUÉS del fix. Sirve como documentación y alimenta las skills. -->

### Fecha de resolución

[YYYY-MM-DD]

### Cambios realizados

```diff
// Resumen de los cambios clave en formato diff
- código viejo
+ código nuevo
```

### Lecciones aprendidas

<!-- Esto alimenta el sistema de skills para futuros bugs similares -->

- **Patrón del bug:** [Ej: "Falta de null check en adapter de datos"]
- **Skill relacionada:** [Ej: "component-skills/MassiveSelectionFilter"]
- **Regla nueva:** [Ej: "Todo adapter debe manejar undefined/null como input"]

### ¿Se debe actualizar alguna skill?

- [ ] Sí → Skill: `[nombre]` | Cambio: `[descripción]`
- [ ] No
