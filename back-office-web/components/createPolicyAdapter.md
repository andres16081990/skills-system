# Skill: createPolicyAdapter

## Ubicación
`src/presentation/adminSettings/TermsAndConditions/helpers/createPolicyAdapter.ts`

## Propósito
Helper puro que transforma `FormValues` (estado interno del formulario Formik) en `PolicyRequest` (payload enviado al backend). También expone una utilidad para detectar títulos duplicados.

## Dónde se usa

| Consumidor | Qué usa |
|---|---|
| `hooks/useCreatePolicie.tsx` | `formValuesToPolicyRequest` |

**Usado en 1 solo lugar** → se puede modificar directamente, pero documentar cualquier cambio.

## Funciones exportadas

### `formValuesToPolicyRequest(values, opts): PolicyRequest`

Convierte `FormValues` en el payload `PolicyRequest` listo para el API.

**Parámetro `opts`:**
```ts
interface BuildReqOpts {
  version: string;
  isCollector: boolean;
  channelsMap: { portalKey: string; micrositeKey: string };
  now?: () => string;  // default: () => new Date().toISOString()
}
```

**Estructura del request generado:**
```ts
{
  policies: PolicyRequestItem[],
  version: string,
  created_date: string  // ISO 8601
}
```

Cada `PolicyRequestItem` se construye como:
```ts
{
  name: p.title.trim(),
  content: p.content,
  link: buildLink(p),
  channels: buildChannels(p, isCollector, channelsMap),
}
```

---

### `getDuplicateTitleIndices(policies: PolicyForm[]): number[]`

Detecta índices de políticas con títulos duplicados (case-insensitive, trimmed).
Retorna el array de índices que tienen duplicado. Si no hay duplicados, retorna `[]`.

---

## Funciones internas (no exportadas)

### `buildLink(p: PolicyForm): PolicyLinkRequest`

| `p.linkType` | Condición | Campos generados |
|---|---|---|
| `"url"` | — | `{ name: p.linkName, url: p.linkUrl, type: "URL" }` |
| `"s3"` | `p.files.length > 0` (archivo nuevo) | `{ name: p.linkName, url: "", b64: p.linkUrl, type: "S3" }` |
| `"s3"` | `p.files.length === 0` (archivo existente) | `{ name: p.linkName, url: p.linkUrl, type: "S3" }` |

**¿Por qué `p.linkUrl` puede contener base64 o una URL S3?**

- **Archivo nuevo (upload):** `usePolicyAttachmentCollapse.handleApplyClick` convierte el PDF a base64 y llama `onLinkChange(b64)`, que escribe la cadena base64 en `p.linkUrl`. En este caso `p.files.length > 0`.
- **Archivo existente (cargado del API):** `mapApiResponseToFormValues` escribe la URL S3 (`item.link.url`) en `p.linkUrl`. En este caso `p.files` es `[]`.

El discriminador correcto es `p.files.length > 0`:
- `true` → `p.linkUrl` = base64 → campo `b64`
- `false` → `p.linkUrl` = URL S3 → campo `url`

### `buildChannels(p, isCollector, map): string[]`

| Condición | Resultado |
|---|---|
| `isCollector === true` | `[]` (sin canales) |
| `p.appliesPortal === true` | push `map.portalKey` |
| `p.appliesMicrosite === true` | push `map.micrositeKey` |

## Tipos relacionados

| Tipo | Ubicación |
|---|---|
| `PolicyRequest`, `PolicyRequestItem`, `PolicyLinkRequest` | `src/domain/model/request/PolicyRequest.ts` |
| `FormValues`, `PolicyForm` | `src/presentation/adminSettings/TermsAndConditions/models/policiesModels.ts` |

## Edge cases y advertencias

- `now` es inyectable para facilitar tests. En producción siempre usa `new Date().toISOString()`.
- `p.title.trim()` se aplica al construir `name`. Si el título tiene espacios iniciales/finales, se limpian en el request.
- El campo `b64` de `PolicyLinkRequest` es opcional (`b64?: string`). Para S3 se omite completamente; solo se usa `url`.

## Historial de cambios

| Fecha | Bug | Resumen |
|---|---|---|
| 2026-02-24 | BUG-001 | Fix en `buildLink` caso S3: URL S3 movida de `b64` a `url`; `b64` eliminado del return |
