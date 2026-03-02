# Comando: Init Skills

## Uso
/init-skills [NOMBRE-PROYECTO]   →   ejemplo: /init-skills mi-proyecto

## Flujo que debes seguir

### Paso 1 — Crear la estructura de carpetas
Crea las siguientes carpetas:
  ~/skills/[NOMBRE-PROYECTO]/components/
  ~/skills/[NOMBRE-PROYECTO]/bugs/
  ~/skills/[NOMBRE-PROYECTO]/commands/
  ~/skills/[NOMBRE-PROYECTO]/features/

---

### Paso 2 — Copiar el sistema base
Copia los siguientes archivos de ~/skills/_base/
hacia ~/skills/[NOMBRE-PROYECTO]/:

  - commands/bugfix.md
  - commands/create-feature.md
  - commands/feature.md
  - bugs/BUGFIX-SPEC-TEMPLATE.md
  - features/FEATURE-SPEC-TEMPLATE.md
  - GLOBAL_RULES.md

Luego reemplaza el placeholder `[NOMBRE-PROYECTO]` en cada archivo copiado
con el nombre real del proyecto.

**Importante:** Usa Python para el reemplazo — evita problemas con rutas en Windows:

```bash
python3 -c "
import sys; path=sys.argv[1]
with open(path,'r',encoding='utf-8') as f: c=f.read()
c=c.replace('[NOMBRE-PROYECTO]','NOMBRE_REAL')
with open(path,'w',encoding='utf-8') as f: f.write(c)
" "$archivo"
```

---

### Paso 3 — Instalar archivos de configuración de agentes

Para cada agente, copia el archivo de `~/skills/_base/agents/` a la raíz del proyecto
y reemplaza `[NOMBRE-PROYECTO]` con el nombre real del proyecto.

| Agente | Archivo en `_base/agents/` | Destino en el proyecto |
|--------|---------------------------|------------------------|
| Claude Code | `CLAUDE.md` | Agregar al final de `CLAUDE.md` existente (o crear) |
| OpenAI Codex CLI | `AGENTS.md` | Agregar al final de `AGENTS.md` existente (o crear) |
| Cursor | `.cursorrules` | Agregar al final de `.cursorrules` existente (o crear) |
| GitHub Copilot | `copilot-instructions.md` | Agregar al final de `.github/copilot-instructions.md` (crear `.github/` si no existe) |

**Regla:** Si el archivo ya existe en el proyecto → agregar el contenido al final.
Si no existe → crearlo con el contenido del template.

El equipo puede tener todos los archivos activos al mismo tiempo.
Cada dev usa el agente de su preferencia; todos comparten los mismos skills.

---

### Paso 4 — Confirmar
Muestra un resumen de todo lo que creaste y modificaste:

```
══════════════════════════════════════════════
  SKILLS INICIALIZADOS: [NOMBRE-PROYECTO]
══════════════════════════════════════════════

  Carpetas creadas:
    ✓ ~/skills/[NOMBRE-PROYECTO]/components/
    ✓ ~/skills/[NOMBRE-PROYECTO]/bugs/
    ✓ ~/skills/[NOMBRE-PROYECTO]/commands/
    ✓ ~/skills/[NOMBRE-PROYECTO]/features/

  Archivos copiados (con rutas actualizadas):
    ✓ commands/bugfix.md
    ✓ commands/create-feature.md
    ✓ commands/feature.md
    ✓ bugs/BUGFIX-SPEC-TEMPLATE.md
    ✓ features/FEATURE-SPEC-TEMPLATE.md
    ✓ GLOBAL_RULES.md

  Configuración de agentes instalada en el proyecto:
    ✓ CLAUDE.md          (Claude Code)
    ✓ AGENTS.md          (OpenAI Codex CLI)
    ✓ .cursorrules       (Cursor)
    ✓ .github/copilot-instructions.md  (GitHub Copilot)

  Comandos disponibles (por agente):
    Claude Code  →  /bugfix [ID]  /create-feature  /feature [ID]
    Codex CLI    →  bugfix [ID]   create-feature   feature [ID]
    Cursor       →  @bugfix [ID]  @create-feature  @feature [ID]
    Copilot      →  "bugfix [ID]" "crear feature"  "feature [ID]"
══════════════════════════════════════════════
```
