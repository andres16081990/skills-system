# Skills System — Instrucciones para el agente

Este repositorio es el sistema de conocimiento del equipo de desarrollo.
No contiene código de producto. Contiene specs, comandos y contexto
que los agentes de IA usan para trabajar en los proyectos reales.

---

## Estructura del repositorio

```
~/skills/
  _base/                    ← plantillas genéricas para nuevos proyectos
    agents/                 ← archivos de config por agente (con placeholder [NOMBRE-PROYECTO])
      CLAUDE.md             → Claude Code
      AGENTS.md             → OpenAI Codex CLI
      .cursorrules          → Cursor
      copilot-instructions.md → GitHub Copilot
    commands/               ← flujos de trabajo del agente (bugfix, feature, create-feature)
    bugs/                   ← template de spec de bug
    features/               ← template de spec de feature
    GLOBAL_RULES.md         ← reglas de modificación de código

  [nombre-proyecto]/        ← una carpeta por proyecto del equipo
    commands/               ← comandos copiados y configurados desde _base
    components/             ← skills de componentes del proyecto
    bugs/                   ← specs de bugs (BUG-001.md, BUG-002.md, ...)
    features/               ← specs de features (FEAT-001.md, FEAT-002.md, ...)
    GLOBAL_RULES.md         ← reglas del proyecto (copiadas desde _base)
```

---

## Cómo inicializar un proyecto nuevo

Lee las instrucciones completas en:

  ~/skills/init-skills.md

Invocación según el agente:

  Claude Code  →  /init-skills [nombre-proyecto]
  Codex CLI    →  init-skills [nombre-proyecto]
  Cursor       →  @init-skills [nombre-proyecto]
  Copilot      →  "inicializar skills para [nombre-proyecto]"

El comando:
1. Crea la carpeta ~/skills/[nombre-proyecto]/ con su estructura
2. Copia y configura los archivos desde _base/
3. Instala los archivos de config de agente en la raíz del proyecto

---

## Cómo agregar soporte para un agente nuevo

1. Crear el archivo de config en ~/skills/_base/agents/[archivo-del-agente]
   con el mismo contenido que los existentes y el placeholder [NOMBRE-PROYECTO]
2. Agregar la entrada correspondiente en la tabla del Paso 3 de init-skills.md
3. Agregar la fila en el resumen del Paso 4 de init-skills.md

---

## Regla principal

Los archivos de esta carpeta son plantillas y documentación del sistema.
Nunca contienen código de producto.
Los specs de bugs y features de cada proyecto viven en su carpeta correspondiente
y no se mezclan entre proyectos.
