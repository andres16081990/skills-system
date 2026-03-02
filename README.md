# Skills System

Sistema de conocimiento compartido para equipos de desarrollo.
Centraliza specs de bugs, features, skills de componentes y comandos de agente
fuera de los repositorios de producto — en una carpeta local estándar que
cualquier agente de IA puede leer.

Compatible con **Claude Code**, **OpenAI Codex CLI**, **Cursor** y **GitHub Copilot**.

---

## Cómo funciona

```
~/skills/                        ← este repo, clonado siempre en esta ruta
  _base/                         ← plantillas genéricas (no tocar)
  [nombre-proyecto]/             ← una carpeta por proyecto del equipo
    GLOBAL_RULES.md
    commands/                    ← bugfix, feature, create-feature
    components/                  ← skills de componentes
    bugs/                        ← specs de bugs
    features/                    ← specs de features
```

Los proyectos reales (en Azure, GitHub, etc.) tienen en su raíz un archivo
de configuración de agente (`CLAUDE.md`, `AGENTS.md`, `.cursorrules`, etc.)
que apunta a esta carpeta. El código del proyecto nunca se mezcla con los skills.

---

## Setup inicial (nuevo integrante del equipo)

### 1. Clonar este repo en la ruta estándar

```bash
git clone git@github.com:andres16081990/skills-system.git ~/skills
```

> La ruta `~/skills` es obligatoria. Todos los archivos de configuración
> de agente apuntan a esa ruta.

### 2. Verificar la estructura

```bash
ls ~/skills
```

Deberías ver: `_base/`, `init-skills.md`, `README.md` y las carpetas de cada proyecto.

### 3. Abrir el proyecto en el que vas a trabajar

Abrí tu agente de IA (Claude Code, Cursor, etc.) apuntando a la carpeta
del proyecto de producto (no a `~/skills`). El archivo de configuración
del agente ya está en ese proyecto con las instrucciones correctas.

---

## Inicializar un proyecto nuevo

Cuando un proyecto aún no tiene su carpeta de skills, ejecutá desde
el agente parado en la raíz de `~/skills`:

| Agente | Comando |
|--------|---------|
| Claude Code | `/init-skills [nombre-proyecto]` |
| Codex CLI | `init-skills [nombre-proyecto]` |
| Cursor | `@init-skills [nombre-proyecto]` |
| Copilot | `"inicializar skills para [nombre-proyecto]"` |

El agente va a leer `~/skills/init-skills.md` y ejecutar el flujo completo:

1. Crear la carpeta `~/skills/[nombre-proyecto]/` con su estructura
2. Copiar y configurar los archivos base desde `_base/`
3. Instalar los archivos de configuración de agente en la raíz del proyecto
4. Agregar al `CLAUDE.md` (u otro archivo de agente) del proyecto el siguiente
   bloque de referencia al sistema de skills:

```
## Sistema de Skills
Los skills, bugs, features y comandos de este proyecto están en:

  ~/skills/[nombre-proyecto]/

Antes de ejecutar cualquier comando, resolver bugs, implementar features
o modificar código, lee los archivos correspondientes desde esa ruta:

  - ~/skills/[nombre-proyecto]/GLOBAL_RULES.md   ← léelo SIEMPRE primero
  - ~/skills/[nombre-proyecto]/commands/          ← comandos disponibles
  - ~/skills/[nombre-proyecto]/components/        ← skills de componentes
  - ~/skills/[nombre-proyecto]/bugs/              ← specs de bugs
  - ~/skills/[nombre-proyecto]/features/          ← specs de features

Para crear bugs usa ~/skills/[nombre-proyecto]/bugs/BUGFIX-SPEC-TEMPLATE.md como base.
Para crear features usa ~/skills/[nombre-proyecto]/features/FEATURE-SPEC-TEMPLATE.md como base.
Para crear skills guárdalos en ~/skills/[nombre-proyecto]/components/

## Capas de implementación (Atomic Design)
Todo desarrollo nuevo sigue este orden estricto:

  Layer 1 → Átomos       (componentes base, sin lógica, sin store)
  Layer 2 → Moléculas    (composición de átomos, estado local permitido)
  Layer 3 → Organismos   (secciones completas, datos por props)
  Layer 4 → Lógica       (hooks, adapters, store slices, servicios)
  Layer 5 → Integración  (montar en el árbol, conectar store, routing)

No avanzar al siguiente layer sin verificar los criterios del actual.

Nunca guardes skills, bugs ni features dentro de este repositorio.
```

---

## Comandos disponibles por agente

Una vez inicializado el proyecto, estos son los comandos disponibles
desde el agente parado en la raíz del proyecto de producto:

| Acción | Claude Code | Codex CLI | Cursor | Copilot |
|--------|-------------|-----------|--------|---------|
| Resolver un bug | `/bugfix BUG-001` | `bugfix BUG-001` | `@bugfix BUG-001` | `"resolver bug BUG-001"` |
| Crear spec de bug | `/create-bug` | `create-bug` | `@create-bug` | `"crear bug"` |
| Crear spec de feature | `/create-feature` | `create-feature` | `@create-feature` | `"crear feature"` |
| Implementar feature | `/feature FEAT-001` | `feature FEAT-001` | `@feature FEAT-001` | `"implementar FEAT-001"` |

---

## Agregar soporte para un agente nuevo

1. Crear `~/skills/_base/agents/[archivo-del-agente]` con el mismo contenido
   que los archivos existentes, usando `[NOMBRE-PROYECTO]` como placeholder
2. Agregar la fila correspondiente en la tabla del Paso 3 de `init-skills.md`
3. Agregar la fila en el resumen del Paso 5 de `init-skills.md`

---

## Mantener sincronizado con el equipo

```bash
# Traer cambios (nuevos skills, bugs, features de otros devs)
cd ~/skills && git pull

# Pushear cambios propios
cd ~/skills && git add . && git commit -m "feat: ..." && git push
```

> Los skills evolucionan con el proyecto. Cada bug resuelto y cada feature
> implementada deja conocimiento nuevo en esta carpeta.
