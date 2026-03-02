# Comando: /create-feature

## Uso

```
/create-feature
```

El agente te pedirá que pegues el texto de la historia de Jira y guiará el resto.

---

## Rutas

```
SKILLS    = C:\Users\Lenovo\skills\back-office-web\
FEATURES  = C:\Users\Lenovo\skills\back-office-web\features\
CMDS      = C:\Users\Lenovo\skills\back-office-web\commands\
COMPS     = C:\Users\Lenovo\skills\back-office-web\components\
RULES     = C:\Users\Lenovo\skills\back-office-web\GLOBAL_RULES.md
TEMPLATE  = C:\Users\Lenovo\skills\back-office-web\features\FEATURE-SPEC-TEMPLATE.md
```

---

## Flujo que debes seguir

### Paso 1 — Recibir la historia

1. Pedirle al dev que pegue el texto completo de la historia de Jira
2. Confirmar que recibiste el texto antes de continuar
3. Leer `RULES` — son de obligatorio cumplimiento durante todo el flujo

---

### Paso 2 — Asignar ID

1. Listar los archivos en `FEATURES\`
2. Identificar el último número usado (ej: FEAT-003)
3. Asignar el siguiente en secuencia (ej: FEAT-004)
4. Si la carpeta está vacía → empezar en FEAT-001

---

### Paso 3 — Parsear la historia

Extraer del texto de Jira:

| Campo | Dónde va |
|-------|----------|
| Título | Header del spec |
| Historia (Como / quiero / para) | §1 |
| Criterios de aceptación | §1 |
| Fecha de hoy | Metadata |

Si algún campo no está en la historia → dejar el placeholder del template y marcarlo con `[INFERIDO]` o `[PENDIENTE]`.

---

### Paso 4 — Analizar dependencias

1. Leer todos los specs existentes en `FEATURES\` (si los hay)
2. Para cada uno, revisar §3 (Contratos de salida)
3. Detectar si algún contrato de salida existente es necesario para esta feature
4. Completar §2 del nuevo spec con las dependencias encontradas
5. Si no hay dependencias → escribir "Ninguna"

---

### Paso 5 — Proponer descomposición en capas

Analizar el código del proyecto para:

1. **§4.1 Átomos** — Identificar:
   - Qué elementos UI base necesita esta feature
   - Cuáles ya existen en el proyecto (reutilizar)
   - Cuáles hay que crear desde cero

2. **§4.2 Moléculas** — Proponer combinaciones de átomos

3. **§4.3 Organismos** — Proponer la sección completa de pantalla

4. **§4.4 Lógica** — Identificar:
   - Hooks nuevos necesarios
   - Adapters de datos
   - Cambios al store (si aplica)
   - Llamadas a API

5. **§4.5 Integración** — Identificar:
   - Punto de montaje en el árbol de componentes
   - Routing (si aplica)

**Regla:** Marcar con `[INFERIDO]` todo lo que sea propuesta del agente, no confirmado por el dev.

---

### Paso 6 — Definir contratos de salida

Proponer §3 basándose en lo identificado en §4:
- Qué componentes nuevos podrían ser reutilizados por otras features
- Qué hooks o adapters serán "públicos" para el sistema

---

### Paso 7 — Presentar y pedir aprobación

Mostrar al dev:

```
══════════════════════════════════════════════
  NUEVA FEATURE: [ID] — [TITULO]
══════════════════════════════════════════════

  Historia parseada:   ✓
  Dependencias:        [N features] / Ninguna
  Contratos de salida: [N artefactos]

  Descomposición propuesta:
    Átomos:     [N] nuevos, [N] reutilizados
    Moléculas:  [N]
    Organismos: [N]
    Lógica:     [N artefactos]

  ⚠ Secciones con [INFERIDO]: [lista]

  ¿Aprobás esta descomposición para guardar el spec?
══════════════════════════════════════════════
```

Esperar confirmación del dev antes de guardar.

---

### Paso 8 — Guardar el spec

1. Copiar `TEMPLATE` como base
2. Llenar todas las secciones con lo construido en los pasos anteriores
3. Guardar en `FEATURES\[ID].md`
4. Confirmar al dev:

```
══════════════════════════════════════════════
  SPEC CREADO: [ID]
══════════════════════════════════════════════

  Archivo: FEATURES\[ID].md
  Estado:  ready

  Para implementar:
    /feature [ID]

══════════════════════════════════════════════
```

---

## Reglas de oro

1. Nunca guardar el spec sin aprobación del dev (Paso 7)
2. Todo lo propuesto por el agente va marcado con `[INFERIDO]`
3. Nunca asignar un ID que ya existe en `FEATURES\`
4. Las GLOBAL_RULES aplican desde el Paso 1
5. Si la historia de Jira es ambigua → preguntar al dev, no asumir

---

## Manejo de errores

| Situación | Acción |
|-----------|--------|
| Historia incompleta o ambigua | Preguntar al dev antes de continuar |
| No se puede determinar dependencias | Marcar §2 como `[PENDIENTE]` y continuar |
| Componente existente similar pero no igual | Proponer reutilizar, marcar con `[INFERIDO]`, dejar decisión al dev |
| Dev rechaza la descomposición propuesta | Ajustar según feedback y volver al Paso 7 |
