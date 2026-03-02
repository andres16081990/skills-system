# Reglas Globales del Agente

## Antes de modificar cualquier código

1. Busca en todo el proyecto cuántos archivos importan o usan
   el componente, método, hook o función que vas a modificar.

2. Si está siendo usado en MÁS DE UN LUGAR:
   - NO modifiques el original
   - Crea una versión nueva e independiente con un nombre descriptivo
   - Usa la versión nueva solo donde se necesita el cambio

3. Si está siendo usado en UN SOLO LUGAR:
   - Puedes modificarlo directamente
   - Aun así documenta qué cambiaste y por qué

4. Esta regla aplica sin excepciones para:
   - Componentes React/Vue
   - Custom hooks
   - Actions y reducers de Redux
   - Servicios y llamadas a API
   - Utils y helpers
   - Tipos y interfaces compartidas

## Por qué existe esta regla
Modificar código compartido puede romper funcionalidades no relacionadas
con el bug que se está resolviendo. Siempre es más seguro crear
que modificar cuando hay reutilización.
