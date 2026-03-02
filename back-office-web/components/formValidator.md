# Skill: formValidator (TermsAndConditions)

## Ubicación
`src/presentation/adminSettings/TermsAndConditions/helpers/formValidator.ts`

## Propósito
Helper puro que implementa la lógica de validación del formulario Formik de políticas (Términos y Condiciones). Devuelve objetos compatibles con `FormikErrors`.

## Dónde se usa

| Consumidor | Qué usa |
|---|---|
| `components/PoliciesPage/CreatePolicie/CreatePolicie.tsx` | `validatePolicies` (pasada como prop `validate` de Formik) |

**Usado en 1 solo lugar** → se puede modificar directamente, pero documentar cualquier cambio.

## Constantes internas

```ts
const urlRegex = /^(https?:\/\/)([\w-]+(\.[\w-]+)+)(:\d{2,5})?([/?#][^\s]*)?$/i;
const MAX_FILE_SIZE_BYTES = 800 * 1024; // 800 KB
```

## Funciones exportadas

### `validateSinglePolicy(p, isCollector, translate): FormikErrors<PolicyForm>`

Valida una política individual. Ejecuta en orden:

1. `validateTitle` → requiere `title` no vacío
2. `validateContent` → requiere `contentLength > 0` y `<= 2000`
3. `validateChannels` → si `!isCollector`, requiere al menos un canal activo
4. Según `p.linkType`:
   - `"url"` → `validateUrl`
   - `"s3"` → `validateS3`
   - otro valor → `errors.linkType = "validations.attachmentTypeInvalid"`

---

### `validatePolicies(values, opts): FormikErrors<FormValues>`

Valida el formulario completo. Flujo:

1. Si `values.policies` está vacío → error global `"validations.atLeastOnePolicy"` y retorno inmediato.
2. Ejecuta `validateSinglePolicy` sobre cada política.
3. Detecta títulos duplicados (case-insensitive, trimmed) y agrega `"validations.duplicateTitle"` a cada política duplicada.
4. Si alguna política tiene errores → asigna `errors.policies` con el array de errores indexado.

**Parámetro `opts`:**
```ts
{ isCollector: boolean; translate: TFunction }
```

---

## Validaciones internas (no exportadas)

### `validateTitle(p, translate, errors)`
| Condición | Campo con error | Clave i18n |
|---|---|---|
| `title` es blank (vacío o solo espacios) | `title` | `"validations.titleRequired"` |

### `validateContent(p, translate, errors)`
| Condición | Campo con error | Clave i18n |
|---|---|---|
| `contentLength` es `0` o falsy | `content` | `"validations.contentRequired"` |
| `contentLength > 2000` | `content` | `"validations.contentMax"` |

> Valida `contentLength` (texto plano sin HTML), **no** la longitud del campo `content` directamente.

### `validateChannels(p, isCollector, translate, errors)`
| Condición | Campos con error | Clave i18n |
|---|---|---|
| `!isCollector` y ni `appliesPortal` ni `appliesMicrosite` activos | `appliesPortal` y `appliesMicrosite` | `"validations.channelRequired"` |

> Solo actúa cuando `isCollector === false`. Las políticas de tipo Collector no tienen restricción de canales.

### `validateUrl(p, translate, errors)`
| Campo | Condición | Clave i18n |
|---|---|---|
| `linkName` | Blank | `"validations.linkNameRequired"` |
| `linkUrl` | Blank | `"validations.urlRequired"` |
| `linkUrl` | No pasa `urlRegex` | `"validations.urlInvalid"` |
| `files` | `files.length > 0` | `"validations.filesNotAllowedForUrl"` |

### `validateS3(p, translate, errors)`
| Campo | Condición | Clave i18n |
|---|---|---|
| `files` | Sin archivos nuevos **Y** `linkUrl` vacío | `"validations.fileRequired"` |
| `linkName` | Blank | `"validations.fileLinkNameRequired"` |
| `files` | Algún archivo > 800 KB | `"validations.fileTooLarge"` |
| `fileSize` | `p.fileSize > MAX_FILE_SIZE_BYTES` | `"validations.fileTooLarge"` (en `errors.fileSize`) |

**Lógica de archivo requerido en S3:**
```ts
const hasNewFiles = p.files && p.files.length > 0;
const hasExistingFile = !isBlank(p.linkUrl); // URL del archivo ya guardado en S3
// Error solo si ninguno de los dos aplica
```
- Si hay archivo grande, el error `fileTooLarge` **sobreescribe** al de `fileRequired`.

## Tipos relacionados

| Tipo | Ubicación |
|---|---|
| `PolicyForm`, `FormValues` | `src/presentation/adminSettings/TermsAndConditions/models/policiesModels.ts` |
| `FormikErrors` | `formik` |
| `TFunction` | `i18next` |

## Tests

```
src/presentation/adminSettings/TermsAndConditions/helpers/__test__/formValidator.test.ts
```

Casos cubiertos (validateS3):
- `files` vacío + `linkUrl` vacío → error `fileRequired`
- Archivos nuevos + `linkUrl` vacío → sin error `fileRequired`
- `files` vacío + `linkUrl` existente → sin error `fileRequired`
- Ambos presentes → sin error
- `linkUrl` existente pero `linkName` blank → error `fileLinkNameRequired`
- Archivo > 800 KB → error `fileTooLarge` (no `fileRequired`)

Casos cubiertos (validateUrl):
- URL válida → sin errores en `linkUrl` ni `linkName`
- URL inválida (`"no-es-url"`) → error `urlInvalid`

## Edge cases y advertencias

- `validateChannels` **no actúa** cuando `isCollector === true`.
- `validateContent` usa `contentLength` (longitud del texto plano). Si el rich text editor no actualiza `contentLength` correctamente, la validación fallará aunque el usuario haya escrito contenido.
- En S3, `fileSize` es distinto de `files[].size`: representa el tamaño del archivo **ya guardado en servidor**. Permite detectar si un archivo previo supera el límite sin que el usuario suba uno nuevo.
- La detección de títulos duplicados en `validatePolicies` es case-insensitive y trim-aware (`p.title?.trim().toLowerCase()`).
- El error de duplicado sobreescribe el error de `titleRequired` en `itemsErrors[i].title`.
- `validateSinglePolicy` es útil para validar policies individualmente fuera del contexto completo del form (e.g., en tests unitarios).
