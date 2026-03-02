# Skill: PolicyItem + usePolicyItem

## Ubicación

| Archivo | Ruta |
|---|---|
| Componente | `src/presentation/adminSettings/TermsAndConditions/components/PolicyItem/PolicyItem.tsx` |
| Hook | `src/presentation/adminSettings/TermsAndConditions/hooks/usePolicyItem.tsx` |
| Estilos | `src/presentation/adminSettings/TermsAndConditions/components/PolicyItem/PolicyItem.scss` |

## Propósito

`PolicyItem` es el acordeón que representa una política individual dentro del formulario de creación/edición de Términos y Condiciones. Toda la lógica de campos Formik está encapsulada en `usePolicyItem`.

## Dónde se usa

| Consumidor | Uso |
|---|---|
| `CreatePolicie.tsx` | Renderiza uno por cada política en el `FieldArray` de Formik |

**`PolicyItem` usado en 1 solo lugar** → se puede modificar directamente.
**`usePolicyItem` usado en 1 solo lugar** (`PolicyItem.tsx`) → se puede modificar directamente.

---

## Props de `PolicyItem`

| Prop | Tipo | Descripción |
|---|---|---|
| `index` | `number` | Índice de la política dentro del `FieldArray` de Formik. Se usa para construir los nombres de campo (`policies.${index}.*`). |
| `policy` | `PolicyForm` | Objeto completo de la política. Usado para leer `policy.open` (estado del acordeón). |
| `remove` | `() => void` | Función del `FieldArray` de Formik envuelta por el padre: `() => remove(idx)`. Elimina la política del array. |
| `fieldError` | `{ [K in keyof PolicyForm]?: string }` | Mapa de errores de validación para esta política (viene de `errors.policies[idx]`). |
| `isCollector` | `boolean` | `true` = modo Collector (oculta la sección de canales Portal/Micrositio). |
| `isEditableVersion` | `boolean` | `true` = modo lectura; `false` = modo edición. Deshabilita todos los campos cuando es `true`. |
| `setSubmitting` | `(isSubmitting: boolean) => void` | Setter de Formik. Se llama con `false` al confirmar la eliminación de una política. |

---

## Modelo `PolicyForm`

```ts
// src/presentation/adminSettings/TermsAndConditions/models/policiesModels.ts
interface PolicyForm {
  id: number;
  title: string;
  content: string;
  open: boolean;
  linkType: string;         // "url" | "s3"
  linkName: string;
  files: File[];
  appliesPortal: boolean;
  appliesMicrosite: boolean;
  linkUrl: string;
  originalS3Url?: string;
  contentLength?: number;
  fileSize?: number;
}
```

---

## Retornos de `usePolicyItem`

| Campo | Tipo | Descripción |
|---|---|---|
| `fieldLink` | `FieldInputProps<string>` | Campo Formik `policies.${index}.title` |
| `fieldContent` | `FieldInputProps<string>` | Campo Formik `policies.${index}.content` |
| `fieldLinkType` | `FieldInputProps<"url" \| "s3">` | Campo Formik `policies.${index}.linkType` |
| `fieldLinkName` | `FieldInputProps<string>` | Campo Formik `policies.${index}.linkName` |
| `fieldLinkUrl` | `FieldInputProps<string>` | Campo Formik `policies.${index}.linkUrl` |
| `fieldFiles` | `FieldInputProps<File[]>` | Campo Formik `policies.${index}.files` |
| `fieldPortal` | `FieldInputProps<boolean>` | Campo Formik `policies.${index}.appliesPortal` |
| `fieldMicrosite` | `FieldInputProps<boolean>` | Campo Formik `policies.${index}.appliesMicrosite` |
| `toggleOpen` | `() => void` | Alterna `policies.${index}.open` (expande/colapsa el acordeón) |
| `handleContentChange` | `(html: string) => void` | Actualiza `content` desde el `RichTextEditor` |
| `handleTypeChange` | `(type: "url" \| "s3") => void` | Actualiza `linkType` |
| `handleLinkNameChange` | `(val: string) => void` | Actualiza `linkName` |
| `handleLinkUrlChange` | `(val: string) => void` | Actualiza `linkUrl` |
| `handleFilesChange` | `(files: File[], fileSize: number) => void` | Actualiza `files`, `fileSize` y pone `originalS3Url = undefined` |
| `handleValidation` | `(err: { file?: string; link?: string }) => void` | Pone errores de campo en `files` y `linkName` vía `setFieldError` |
| `handlePortalChange` | `(_e: any, checked: boolean) => void` | Actualiza `appliesPortal` |
| `handleMicrositeChange` | `(_e: any, checked: boolean) => void` | Actualiza `appliesMicrosite` |
| `getLengthText` | `(text: number) => void` | Actualiza `contentLength` (callback del `RichTextEditor`) |
| `translateText` | `(k: string) => string` | Wrapper de `t()` con keyPrefix `properties.termsAndConditions.pages.main` |
| `confirmOpenDelete` | `boolean` | Controla visibilidad del `GenericModal` de confirmación de eliminación |
| `onDeleteClick` | `() => void` | Abre el modal de confirmación de eliminación |
| `onCancelDelete` | `() => void` | Cierra el modal sin eliminar |
| `onConfirmDelete` | `() => void` | Llama `remove()`, cierra el modal y llama `setSubmitting(false)` |

---

## Estructura visual del componente

```
<div className="policy-item">
  ├── .policy-item__header
  │     ├── <CheckBoxIcon />
  │     ├── isEditableVersion → <BodyText> con el título (readonly)
  │     ├── !isEditableVersion → <OutlinedInput> editable (max 100 chars)
  │     └── <IconButton> toggle acordeón (ExpandLess/ExpandMore)
  │
  ├── <Collapse in={policy.open}>
  │     └── .policy-item__body
  │           ├── .policy-item__attachment → <PolicyAttachmentCollapse>
  │           ├── !isCollector → .policy-item__payer-channel
  │           │     ├── <Checkbox> appliesPortal
  │           │     └── <Checkbox> appliesMicrosite
  │           ├── .policy-item__tooltip → <Tooltip> info adicional
  │           ├── .policy-item__editor → <RichTextEditor>
  │           └── !isEditableVersion → .policy-item__footer
  │                 └── <LinkText> "Eliminar política"
  │
  ├── <Divider />
  └── <GenericModal> confirmación de eliminación
```

---

## Lógica de `handleFilesChange`

Al cambiar archivos se fuerza `originalS3Url = undefined`. Esto indica que el archivo ya no es el que estaba en S3; el adapter de creación (`createPolicyAdapter.ts`) usa esta ausencia para saber que debe enviar el archivo nuevo en lugar de la URL existente.

---

## Lógica de `onConfirmDelete`

```
onConfirmDelete()
  → remove()           # elimina la política del FieldArray de Formik
  → setConfirmOpenDelete(false)   # cierra el modal
  → setSubmitting(false)          # permite que el formulario vuelva a enviarse
```

El `setSubmitting(false)` es crítico: si se elimina una política mientras Formik estaba en estado `isSubmitting = true`, sin resetear este flag el botón de guardar quedaría bloqueado.

---

## Dependencias clave

| Archivo | Rol |
|---|---|
| `hooks/usePolicyItem.tsx` | Toda la lógica de campos y handlers |
| `components/AttachmenComponent/` | `PolicyAttachmentCollapse` — sección de adjunto/link |
| `components/AttachmenComponent/constants/constants.ts` → `MAX_LENGTH_TEXT` | Límite de caracteres del `RichTextEditor` |
| `models/policiesModels.ts` → `PolicyForm` | Tipo del objeto de política |

---

## Edge cases y advertencias

- **`fieldError` viene del padre.** `PolicyItem` no llama a Formik directamente para leer errores de `title` y `content`; los recibe como prop. El padre (`CreatePolicie.tsx`) extrae `errors.policies` y pasa `errors.policies[idx]`.
- **`isCreating={true}` siempre.** La prop `isCreating` de `PolicyAttachmentCollapse` está hardcodeada a `true`; no depende de `isEditableVersion`.
- **`onApply={() => {}}` hardcodeado.** El callback `onApply` de `PolicyAttachmentCollapse` es un no-op; la lógica de aplicar cambios del adjunto se maneja vía los callbacks individuales (`onTypeChange`, `onFilesChange`, etc.).
- **`onCancel` cierra el acordeón.** El botón cancelar del adjunto llama a `toggleOpen()`, colapsando toda la política.
- **El `RichTextEditor` reporta longitud** vía `onLengthChange={getLengthText}`, que guarda en `contentLength`. Este valor se usa probablemente en validación o en el `createPolicyAdapter`.
