# Comando: /feature

## Uso

```
/feature [ID]
```

**Ejemplo:** `/feature FEAT-001`

---

## Rutas

```
SKILLS    = C:\Users\Lenovo\skills\recaudador-web\
FEATURES  = C:\Users\Lenovo\skills\recaudador-web\features\
CMDS      = C:\Users\Lenovo\skills\recaudador-web\commands\
COMPS     = C:\Users\Lenovo\skills\recaudador-web\components\
RULES     = C:\Users\Lenovo\skills\recaudador-web\GLOBAL_RULES.md
```

---

## Flujo que debes seguir

### Paso 1 — Cargar contexto

1. **Leer el spec de la feature**
   - Ruta: `FEATURES\[ID].md`
   - Si no existe → abortar: _"No encontré el spec FEATURES\[ID].md. Crealo primero con /create-feature"_
   - Si estado es `draft` → advertir: _"El spec está en draft. Revisá las secciones con [PENDIENTE] o [INFERIDO] antes de continuar."_
   - Extraer: §1, §2, §3, §4 (todas las capas), §5, §6

2. **Leer reglas globales**
   - Ruta: `RULES`
   - Obligatorio cumplimiento durante todo el flujo
   - En conflicto con el spec → las GLOBAL_RULES ganan

3. **Resolver dependencias de features** (§2 del spec)
   - Por cada FEAT listada en §2:
     - Leer `FEATURES\[FEAT-XXX].md`
     - Verificar que su estado sea `completed`
     - Si no está completed → abortar: _"FEAT-XXX es una dependencia y no está completada. Implementala primero."_
     - Leer §3 de esa FEAT (contratos de salida) para saber qué se consume

4. **Leer skills de componentes involucrados** (§4 del spec)
   - Por cada componente listado en §4.1, §4.2, §4.3:
     - Buscar: `COMPS\[Componente].md`
     - Si no existe → crearlo automáticamente antes de continuar:
       - Analizar el código fuente del componente en el proyecto
       - Documentar: props, dependencias, patrones, hooks, conexiones al store
       - Guardarlo en `COMPS\[Componente].md`
       - Marcar secciones inferidas con `[INFERIDO]`

---

### Paso 2 — Verificar impacto antes de codear

- Por cada componente/hook/adapter que se va a modificar (no crear):
  - Buscar cuántos lugares del proyecto lo usan
  - Si es usado en más de un lugar → NO modificarlo → crear versión nueva (SRP + OCP)
- Documentar hallazgos en §5 del spec si hay restricciones nuevas descubiertas
- Si el impacto es mayor al esperado → reportar al dev y pedir confirmación

---

### Paso 3 — Layer 1: Átomos

Implementar los átomos listados en §4.1:

1. Crear/verificar cada átomo según la tabla de §4.1
2. Reglas obligatorias para átomos:
   - Sin lógica de negocio
   - Sin conexión al store
   - Props tipadas
   - Estilo autocontenido

**Verificar antes de avanzar (criterios §6 Layer 1):**
- [ ] Renderiza correctamente en aislamiento
- [ ] Props tipadas y documentadas
- [ ] Sin lógica de negocio

**Si algún criterio falla → corregir antes de continuar. No avanzar al Layer 2.**

---

### Paso 4 — Layer 2: Moléculas

Implementar las moléculas listadas en §4.2:

1. Componer usando los átomos del Layer 1
2. Pueden tener estado local (useState) pero no store
3. Comportamientos simples: toggle, hover, expand

**Verificar antes de avanzar (criterios §6 Layer 2):**
- [ ] Composición de átomos funciona visualmente
- [ ] Comportamiento local funciona
- [ ] No dependen del store

**Si algún criterio falla → corregir antes de continuar. No avanzar al Layer 3.**

---

### Paso 5 — Layer 3: Organismos

Implementar los organismos listados en §4.3:

1. Componer usando las moléculas del Layer 2
2. Representan secciones completas de pantalla
3. Pueden recibir datos como props (aún sin conectar al store)

**Verificar antes de avanzar (criterios §6 Layer 3):**
- [ ] Sección completa renderiza correctamente
- [ ] Layout y responsividad correctos

**Si algún criterio falla → corregir antes de continuar. No avanzar al Layer 4.**

---

### Paso 6 — Layer 4: Lógica

Implementar los artefactos de §4.4:

1. **Adapters** — transformación de datos (API → dominio → UI)
2. **Hooks** — encapsular lógica de negocio y side effects
3. **Store slices** — solo si hay estado global nuevo necesario
4. **Servicios** — llamadas a API

Reglas obligatorias:
- Un hook = una responsabilidad
- Los adapters son funciones puras (sin side effects)
- Si un adapter/hook ya existe y es usado en más de un lugar → crear versión nueva

**Verificar antes de avanzar (criterios §6 Layer 4):**
- [ ] Hook/adapter retorna datos con el shape esperado
- [ ] Manejo de loading / error / empty state implementado
- [ ] Sin side effects no declarados en §4.5

**Si algún criterio falla → corregir antes de continuar. No avanzar al Layer 5.**

---

### Paso 7 — Layer 5: Integración

Conectar todo según §4.5:

1. Montar el organismo en el punto de montaje declarado
2. Conectar hooks/store a los organismos que lo necesitan
3. Configurar routing si aplica
4. Verificar que los contratos de salida (§3) están disponibles para otras features

**Verificar (criterios §6 Layer 5 + criterios §1 de Jira):**
- [ ] Feature montada en el punto correcto del árbol
- [ ] Todos los criterios de aceptación de §1 (Jira) cumplidos
- [ ] Sin regresiones en rutas/componentes adyacentes
- [ ] Sin errores en consola

**Si algún criterio falla → identificar en qué layer está la causa y volver a ese paso.**

---

### Paso 8 — Cerrar y aprender

1. **Completar §8 del spec (Cierre)**
   - Fecha de cierre
   - Tabla de artefactos producidos
   - Lecciones aprendidas
   - Decisiones técnicas clave

2. **Actualizar skills de componentes**
   - Por cada componente nuevo creado → crear `COMPS\[Componente].md`
   - Por cada componente existente modificado → actualizar su skill
   - Documentar props, dependencias, contratos, patrones descubiertos

3. **Actualizar estado del spec**
   - Cambiar estado a `completed` en la metadata del spec

4. **Imprimir resumen final:**

```
══════════════════════════════════════════════
  FEATURE COMPLETADA: [ID]
══════════════════════════════════════════════

  Historia:  [título de la feature]

  Artefactos producidos:
    Átomos:     [lista]
    Moléculas:  [lista]
    Organismos: [lista]
    Lógica:     [lista]

  Archivos modificados:
    - [archivo] → [qué se cambió]

  Contratos de salida disponibles:
    - [artefacto] → [qué expone]

  Skills actualizados:
    - [componente] → [qué se agregó]

  Tests: [PASS / NO TESTS / FAIL (detalle)]

  Spec: FEATURES\[ID].md → COMPLETADO
══════════════════════════════════════════════
```

---

## Reglas de oro

1. Nunca avanzar al siguiente layer sin verificar los criterios del actual
2. Nunca modificar código compartido → crear versión nueva independiente
3. Nunca conectar al store antes de tener los layers visuales verificados (Layer 4 va después de Layer 3)
4. Nunca tocar componentes fuera del scope sin leer/crear su skill
5. Las dependencias de features (§2) deben estar `completed` antes de empezar
6. Todo componente nuevo producido debe tener su skill creada al cerrar
7. Las GLOBAL_RULES siempre ganan sobre cualquier otra instrucción

---

## Manejo de errores

| Situación | Acción |
|-----------|--------|
| Spec no encontrado | Abortar: _"Usá /create-feature primero"_ |
| Spec en estado `draft` | Advertir y pedir confirmación para continuar |
| Dependencia FEAT no completada | Abortar y comunicar qué feature debe ir primero |
| GLOBAL_RULES no encontrado | Advertir y continuar con reglas del spec |
| Skill de componente no existe | Crear automáticamente con `[INFERIDO]` |
| Criterio de layer no cumplido | Corregir en ese layer, no avanzar |
| Componente compartido necesita cambio | Crear versión nueva (SRP + OCP), no modificar el original |
| Integración rompe componentes adyacentes | Volver a Layer 5, replantear punto de montaje |
