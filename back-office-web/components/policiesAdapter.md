# Skill: policiesAdapter

## Ubicación
`src/presentation/adminSettings/TermsAndConditions/helpers/policiesAdapter.ts`

## Propósito
Helper puro que transforma la respuesta de la API (`PoliciesResponse`) en las estructuras de datos que consume el formulario Formik y el componente de previsualización de políticas (Términos y Condiciones).

## Dónde se usa

| Consumidor | Qué usa |
|---|---|
| `hooks/useCreatePolicie.tsx` | `mapApiResponseToFormValues`, `mapApiResponseToPreview` |

**Usado en 1 solo lugar** → se puede modificar directamente, pero documentar cualquier cambio.

## Funciones exportadas

### `mapApiResponseToFormValues(apiResponse: PoliciesResponse): FormValues`

Convierte cada `PolicyDto` de la respuesta en un `PolicyForm` listo para Formik.

**Mapeo campo por campo:**

| Campo destino (`PolicyForm`) | Fuente (`PolicyDto`) | Notas |
|---|---|---|
| `id` | `idx + 1` | Índice 1-based; no es el ID de base de datos |
| `title` | `item.name` | |
| `content` | `item.content` | HTML crudo tal como viene del API |
| `contentLength` | `item.content?.replace(/<[^>]*>/g, "").trim().length` | Longitud del texto plano sin tags HTML; `0` si es nullish |
| `open` | `false` | Acordeón colapsado por defecto |
| `linkType` | `item.link.type.toLowerCase()` | Siempre en minúsculas: `"url"` o `"s3"` |
| `linkName` | `item.link.name` | **No confundir con `link.url`** |
| `linkUrl` | `item.link.url` | **No confundir con `link.name`** |
| `files` | `[]` | Siempre vacío; los archivos viven solo en estado local del form |
| `appliesPortal` | `channels.includes("PAYER")` | Via `mapChannelsToFlags` |
| `appliesMicrosite` | `channels.includes("PAYMENT_LINK")` | Via `mapChannelsToFlags` |

> **Bug histórico resuelto:** En versiones anteriores `linkName` y `linkUrl` estaban invertidos. Los tests de regresión en `__test__/policiesAdapter.test.ts` protegen este comportamiento.

---

### `mapApiResponseToPreview(apiResponse: PoliciesResponse): PreviewPolicyItem[]`

Convierte la respuesta de la API en el formato que consume el componente de previsualización.

**Tipo de retorno `PreviewPolicyItem`** (también exportado desde este archivo):

```ts
export interface PreviewPolicyItem {
  id: string;           // String(idx + 1)
  label: string;        // item.name
  htmlContent: string;  // item.content (HTML crudo)
  disabled?: boolean;   // false siempre en carga inicial
  link?: {
    url?: string;       // item.link.url
    name?: string;      // item.link.name
    type: TypePolicie.URL | TypePolicie.S3;  // item.link.type.toLowerCase()
  };
}
```

> **Advertencia:** `PreviewPolicyItem` también está definido inline en `useCreatePolicie.tsx`. Usar el de este adapter cuando se necesite fuera del hook para mantener consistencia.

---

## Función interna (no exportada)

### `mapChannelsToFlags(channels: string[])`

```ts
const mapChannelsToFlags = (channels: string[]) => ({
  appliesMicrosite: channels.includes("PAYMENT_LINK"),
  appliesPortal: channels.includes("PAYER"),
});
```

Convierte el array de strings de canales en flags booleanos para el formulario.

## Tipos relacionados

| Tipo | Ubicación |
|---|---|
| `PoliciesResponse`, `PolicyLinkDto` | `src/domain/model/response/PoliciesResponse.ts` |
| `PolicyForm`, `FormValues` | `src/presentation/adminSettings/TermsAndConditions/models/policiesModels.ts` |
| `TypePolicie` | `src/presentation/adminSettings/TermsAndConditions/components/AttachmenComponent/constants/constants.ts` |

## Tests

```
src/presentation/adminSettings/TermsAndConditions/helpers/__test__/policiesAdapter.test.ts
```

Casos cubiertos:
- `linkName` se mapea desde `item.link.name` (no desde `item.link.url`)
- `linkUrl` se mapea desde `item.link.url` (no desde `item.link.name`)
- Políticas tipo `s3` no invierten los campos
- Nombres que parecen URLs y URLs que parecen nombres no se confunden
- `channels: ["PAYER", "PAYMENT_LINK"]` → `appliesPortal: true`, `appliesMicrosite: true`
- `files` siempre se inicializa como `[]`
- Múltiples políticas conservan orden e índices correctos

## Edge cases y advertencias

- `contentLength` mide el texto plano (sin HTML). Si el contenido es `null`/`undefined`, queda en `0` (no lanza error).
- `linkType` siempre se convierte a minúsculas. La API puede devolver `"URL"` o `"S3"` en mayúsculas.
- `files` siempre es `[]` tras el mapeo. Los archivos subidos viven solo en el estado local del form, nunca vienen del API.
- `id` es 1-based (`idx + 1`), no el ID real de base de datos.
- El mapeo inverso (UI → API) vive en `helpers/createPolicyAdapter.ts`, no en este archivo.
