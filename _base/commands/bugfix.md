# Comando: /bugfix

## Uso

```
/bugfix [ID]
```

**Ejemplo:** `/bugfix BUG-001`

---

## Rutas

```
SKILLS   = ~/skills/[NOMBRE-PROYECTO]/
BUGS     = ~/skills/[NOMBRE-PROYECTO]/bugs/
CMDS     = ~/skills/[NOMBRE-PROYECTO]/commands/
COMPS    = ~/skills/[NOMBRE-PROYECTO]/components/
RULES    = ~/skills/[NOMBRE-PROYECTO]/GLOBAL_RULES.md
```

---

## Flujo que debes seguir

### Paso 1 — Cargar contexto

1. **Leer el spec del bug**
   - Ruta: `BUGS/[ID].md`
   - Si no existe → abortar: _"No encontré el spec BUGS/[ID].md. Crealo primero con /create-bug"_
   - Extraer del spec:
     - §1 Resumen (qué pasa)
     - §2 Archivos involucrados y dependencias
     - §3 Reproducción y comportamiento esperado
     - §5 Restricciones (si ya están definidas)

2. **Leer reglas globales**
   - Ruta: `RULES`
   - Son de **obligatorio cumplimiento** durante todo el flujo
   - Si hay conflicto entre el spec y las reglas globales → las reglas globales ganan

3. **Leer skills de componentes involucrados**
   - Por cada componente listado en §2 del spec:
     - Buscar: `COMPS/[Componente].md`
     - Si no existe → crearlo automáticamente antes de continuar:
       - Analizar el código fuente del componente en el proyecto
       - Documentar: props, dependencias, patrones, hooks, conexiones al store
       - Guardarlo en `COMPS/[Componente].md`
       - Marcar secciones inferidas con `[INFERIDO]`

---

### Paso 2 — Investigar causa raíz

- Analizar el código de cada componente involucrado
- Seguir las pistas de §3 del spec (pasos de reproducción)
- Rastrear componentes hijos que puedan estar relacionados
- **Verificar impacto:** antes de planificar cambios, buscar cuántos lugares usan cada método/componente que se va a modificar (regla global)
- Formular hipótesis y confirmar contra el código

**Al terminar este paso, completar en el spec:**
- **§4 Causa raíz:** hipótesis evaluadas + causa confirmada con archivo y líneas exactas

---

### Paso 3 — Planificar el fix

Completar **§5 del spec** antes de tocar código:
- Estrategia a alto nivel
- Tabla de cambios requeridos (archivo + cambio + principio aplicado)
- Restricciones de lo que NO se debe tocar

**Principios obligatorios al planificar:**
- Si un componente/método es usado en más de un lugar → NO modificarlo, crear versión nueva (SRP + OCP)
- Un cambio = una responsabilidad
- Crear nuevo antes de modificar existente

---

### Paso 4 — Aplicar el fix

- Ejecutar el plan definido en §5
- Respetar estrictamente:
  - Las restricciones del spec (§5)
  - Las reglas de cada skill de componente
  - Las GLOBAL_RULES
- **Si necesitás tocar un componente fuera del scope del bug:**
  - Leer su skill en `COMPS/[Componente].md`
  - Si no tiene skill → crearlo primero
  - Agregar el componente a §2 del spec

---

### Paso 5 — Verificar

- Recorrer cada criterio de §6 del spec y verificar que se cumple
- Correr los tests de los componentes modificados
- Si no hay tests → advertirlo en el resumen
- **Si algún criterio no se cumple → volver al Paso 4 e iterar**

---

### Paso 6 — Cerrar y aprender

1. **Completar §8 del spec (Resolución)**
   - Fecha de resolución
   - Diff resumido de cambios clave
   - Lecciones aprendidas
   - Patrón del bug identificado

2. **Actualizar skills de componentes modificados**
   - Agregar lección aprendida al skill en `COMPS/[Componente].md`
   - Si se descubrió un nuevo patrón de error → documentarlo
   - Si se descubrió una nueva dependencia → agregarla

3. **Imprimir resumen final:**

```
══════════════════════════════════════════════
  BUGFIX COMPLETADO: [ID]
══════════════════════════════════════════════

  Causa raíz:     [resumen en 1 línea]

  Archivos modificados:
    - [archivo] → [qué se cambió]

  Archivos creados:
    - [archivo] → [por qué]

  Skills actualizados:
    - [componente] → [qué se agregó]

  Tests: [PASS / NO TESTS / FAIL (detalle)]

  Spec: BUGS/[ID].md → COMPLETADO
══════════════════════════════════════════════
```

---

## Reglas de oro

1. Nunca modificar código compartido → crear versión nueva independiente
2. Nunca aplicar fix sin haber completado §4 y §5 del spec primero
3. Nunca tocar componentes fuera del scope sin leer/crear su skill
4. Si algo del skill está marcado como `[INFERIDO]`, advertirlo antes de actuar
5. Si un criterio de §6 no se cumple, iterar — nunca cerrar el bug incompleto
6. Todo fix debe dejar el spec completamente lleno al terminar
7. Las GLOBAL_RULES siempre ganan sobre cualquier otra instrucción

---

## Manejo de errores

| Situación | Acción |
|-----------|--------|
| Spec no encontrado | Abortar: _"Usá /create-bug primero"_ |
| GLOBAL_RULES no encontrado | Advertir y continuar con reglas del spec |
| Skill de componente no existe | Crear automáticamente con `[INFERIDO]` |
| Causa raíz no identificable | Documentar hipótesis en §4, pedir input al dev |
| Fix rompe tests existentes | Revertir, replantear §5, iterar |
| Componente fuera de scope | Leer/crear skill, agregar a §2, continuar |
