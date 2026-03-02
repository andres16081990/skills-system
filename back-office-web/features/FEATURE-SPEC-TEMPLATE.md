# Feature Spec: [TITULO_CORTO]

> **ID:** FEAT-[XXXX]
> **Fecha:** [YYYY-MM-DD]
> **Proyecto:** [nombre-proyecto]
> **Prioridad:** [critical | high | medium | low]
> **Estado:** [draft | ready | in-progress | completed]

---

## §1. Historia de usuario

<!-- Pegar aquí el texto raw de Jira sin modificar -->

**Como** [rol]
**quiero** [funcionalidad]
**para** [beneficio]

### Criterios de aceptación (Jira)

<!-- Los criterios tal cual vienen de Jira -->

- [ ] [criterio 1]
- [ ] [criterio 2]

---

## §2. Dependencias de features

<!-- Qué produce otra FEAT que esta necesita consumir -->
<!-- Si no depende de ninguna, escribir "Ninguna" -->

| FEAT | Qué produce | Qué consume esta feature |
|------|-------------|--------------------------|
| FEAT-XXX | [componente/hook/adapter] | [para qué lo usa] |

---

## §3. Contratos de salida

<!-- Qué produce esta FEAT que otras podrían consumir -->
<!-- Define la "API pública" de esta feature para el resto del sistema -->

| Tipo | Nombre | Descripción |
|------|--------|-------------|
| Componente | `[NombreComponente]` | [qué hace, qué props expone] |
| Hook | `[useNombre]` | [qué retorna] |
| Adapter | `[nombreAdapter]` | [input → output] |
| Store | `[slice/selector]` | [qué estado maneja] |

---

## §4. Diseño por capas

<!-- El agente completa y propone esta sección en /create-feature -->
<!-- El dev aprueba antes de que comience /feature -->

### §4.1 Átomos

<!-- Componentes base: sin lógica, solo UI -->
<!-- Indicar si se crea nuevo o se reutiliza uno existente -->

| Componente | Nuevo / Reutilizar | Descripción |
|------------|--------------------|-------------|
| `[Nombre]` | Nuevo | [qué representa] |
| `[Nombre]` | Reutilizar de [ruta] | [cómo se usa] |

### §4.2 Moléculas

<!-- Combinaciones de átomos con comportamiento simple -->

| Componente | Átomos que compone | Descripción |
|------------|--------------------|-------------|
| `[Nombre]` | [Átomo1, Átomo2] | [qué hace] |

### §4.3 Organismos

<!-- Secciones completas compuestas de moléculas -->

| Componente | Moléculas que compone | Descripción |
|------------|-----------------------|-------------|
| `[Nombre]` | [Molécula1, Molécula2] | [qué representa en la pantalla] |

### §4.4 Lógica

<!-- Hooks, adapters, store slices, servicios -->
<!-- Se implementa DESPUÉS de tener la estructura visual verificada -->

| Artefacto | Tipo | Responsabilidad |
|-----------|------|-----------------|
| `[nombre]` | hook / adapter / slice / service | [qué hace] |

### §4.5 Integración

<!-- Cómo se ensambla todo y dónde se conecta al árbol existente -->

- **Punto de montaje:** `[ruta del componente padre donde se integra]`
- **Conexión al store:** `[qué acciones/selectores usa]`
- **Routing:** `[si aplica, qué ruta nueva se agrega]`
- **Side effects:** `[llamadas a API, eventos, etc.]`

---

## §5. Restricciones

<!-- Qué NO se debe tocar ni cambiar -->

- ❌ No modificar componentes existentes usados en más de un lugar
- ❌ No alterar el shape del store fuera del slice nuevo
- ❌ No agregar lógica en los átomos
- ✅ [Libertades permitidas]

---

## §6. Criterios de aceptación por capa

<!-- El agente verifica cada uno antes de avanzar a la siguiente capa -->

### Layer 1 — Átomos
- [ ] Renderizan correctamente en aislamiento
- [ ] Props tipadas y documentadas
- [ ] Sin lógica de negocio

### Layer 2 — Moléculas
- [ ] Composición de átomos funciona visualmente
- [ ] Comportamiento local (ej: toggle, hover) funciona
- [ ] No dependen del store

### Layer 3 — Organismos
- [ ] Sección completa renderiza correctamente
- [ ] Layout y responsividad correctos

### Layer 4 — Lógica
- [ ] Hook/adapter retorna los datos con el shape esperado
- [ ] Manejo de loading / error / empty state
- [ ] No hay side effects no declarados en §4.5

### Layer 5 — Integración
- [ ] Feature montada en el punto correcto del árbol
- [ ] Criterios de §1 (Jira) todos cumplidos
- [ ] No hay regresiones en rutas/componentes adyacentes
- [ ] Sin errores en consola

---

## §7. Testing

### Verificación manual

```
1. [Paso de verificación]
2. [Paso de verificación]
```

### Tests automatizados

| Test | Descripción | Capa | Estado |
|------|-------------|------|--------|
| `[test file]` | [qué valida] | [Layer N] | [ ] pendiente |

---

## §8. Cierre

<!-- Se completa DESPUÉS de finalizar la implementación -->

### Fecha de cierre

[YYYY-MM-DD]

### Resumen de lo producido

| Artefacto | Ruta | Tipo |
|-----------|------|------|
| `[Nombre]` | `src/...` | Átomo / Molécula / Organismo / Hook / Adapter |

### Lecciones aprendidas

- **Patrón identificado:** [descripción]
- **Decisión técnica clave:** [qué se decidió y por qué]
- **Skills a actualizar:** [componente → qué se agregó]

### ¿Produce contratos para otras features?

- [ ] Sí → Documentar en §3 y notificar en specs de features dependientes
- [ ] No
