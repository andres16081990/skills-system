# Skill: useCreatePolicie

## Ubicación
`src/presentation/adminSettings/TermsAndConditions/hooks/useCreatePolicie.tsx`

## Propósito
Hook central de la pantalla de creación/edición de políticas de Términos y Condiciones. Gestiona los valores iniciales de Formik, el flujo de guardado con diálogo de confirmación, el modal de preview, y despacha el thunk `createPolicies` hacia Redux.

## Dónde se usa

| Consumidor | Qué usa |
|---|---|
| `CreatePolicie.tsx` (`src/.../PoliciesPage/CreatePolicie/CreatePolicie.tsx`) | Todos los retornos excepto `onApplyCreate` y `setIsEditableVersion` |

**Usado en 1 solo lugar** → se puede modificar directamente, pero documentar cualquier cambio.

## Parámetro de entrada

| Parámetro | Tipo | Descripción |
|---|---|---|
| `isCollector` | `boolean` | `true` = modo Collector; `false` = modo Payment Channels (Payer). Afecta los valores iniciales, las reglas de canal y el `typePortal` enviado al API. |

## Valor retornado

| Campo | Tipo | Descripción |
|---|---|---|
| `initialFormValues` | `FormValues` | Valores iniciales para Formik, derivados del estado Redux (`policies_data`). |
| `isEditableVersion` | `boolean` | `true` cuando hay políticas guardadas (modo lectura). `false` cuando se está creando o editando. |
| `setIsEditableVersion` | `Dispatch<SetStateAction<boolean>>` | Setter directo del flag de edición. |
| `handleSubmit` | `(values: FormValues) => void` | Transforma el formulario a `PolicyRequest` y despacha `createPolicies`. |
| `createNewPolicy` | `(policies: PolicyForm[]) => PolicyForm` | Genera un `PolicyForm` vacío con ID auto-incremental. |
| `isPreviewOpen` | `boolean` | Controla visibilidad del modal de preview. |
| `previewPolicies` | `PreviewPolicyItem[]` | Políticas formateadas para el preview (clases Quill → estilos inline). |
| `openPreview` | `(pols: PolicyForm[]) => void` | Construye `previewPolicies` y abre el modal. |
| `closePreview` | `() => void` | Cierra el modal de preview. |
| `handleContinue` | `() => void` | Alias de `closePreview` (botón "Continuar" del modal). |
| `handleCancel` | `() => void` | Alias de `closePreview` (botón "Cancelar" del modal). |
| `confirmOpen` | `boolean` | Controla visibilidad del diálogo de confirmación de guardado. |
| `onConfirmCreate` | `() => void` | Abre el diálogo de confirmación. |
| `onCancelCreate` | `() => void` | Cierra el diálogo sin guardar. |
| `onApplyCreate` | `() => void` | Cierra el diálogo (usado internamente antes de `handleSubmit`). |
| `onEditPolicy` | `() => void` | Vuelve al modo edición (`isEditableVersion = false`). |
| `handleClickConfirm` | ver firma abajo | Valida antes de confirmar; si hay errores marca todos los campos como touched. |

### Firma de `handleClickConfirm`
```ts
handleClickConfirm(
  isValid: boolean,
  setTouched: (touched: FormikTouched<FormValues>, shouldValidate?: boolean) => Promise<void | FormikErrors<FormValues>>,
  values: FormValues,
  setSubmitting: (isSubmitting: boolean) => void,
): void
```
- `isValid = false` → cierra el confirm, activa `touched` en todos los campos de todas las políticas para mostrar errores.
- `isValid = true` → cierra el confirm, despacha `handleSubmit`, pone `setSubmitting(true)` e `isEditableVersion = true`.

## Estado Redux leído

### `policiesSelector` (slice `policies`)
| Campo | Uso |
|---|---|
| `policies_data` | Fuente de datos para `initialFormValues`. Se mapea con `mapApiResponseToFormValues`. |
| `processState.current` | Detecta `"success"` para saber cuándo actualizar los valores iniciales. |
| `processState.action` | Se compara con `PoliciesAction.GET_POLICIES_ACTION` para filtrar el efecto. |

## Acciones Redux disparadas

| Acción / Thunk | Cuándo |
|---|---|
| `createPolicies({ data, typePortal })` | En `handleSubmit`, al confirmar el guardado con formulario válido. |

## Lógica de `initialFormValues` según `isCollector`

El `useEffect` que actualiza los valores solo actúa cuando:
- `processState.current === "success"`
- `processState.action === PoliciesAction.GET_POLICIES_ACTION`
- `policies_data.policies.length > 0`

**`isCollector = false` (Payment Channels):** si alguna política tiene ambos flags `appliesPortal` y `appliesMicrosite` en `false`, se fuerzan ambos a `true`.

**`isCollector = true` (Collector):** los valores del API se mapean sin modificación.

Un segundo `useEffect` activa `isEditableVersion = true` en cuanto `initialFormValues.policies` tenga al menos un elemento.

## Mapeo de canales UI ↔ API

| Flag UI | Canal API |
|---|---|
| `appliesPortal = true` | `"PAYER"` |
| `appliesMicrosite = true` | `"PAYMENT_LINK"` |
| `isCollector = true` | `channels: []` (sin canales) |

El mapeo inverso (API → UI) vive en `helpers/policiesAdapter.ts`:
- `channels.includes("PAYER")` → `appliesPortal`
- `channels.includes("PAYMENT_LINK")` → `appliesMicrosite`

## Flujo completo de guardado

```
Usuario hace clic en "Guardar"
  → onConfirmCreate()          # abre GenericModal de confirmación
  → Usuario confirma
  → handleClickConfirm(isValid, setTouched, values, setSubmitting)
      ├── isValid = false → onApplyCreate() + setTouched (muestra errores), return
      └── isValid = true
            → onApplyCreate()      # cierra modal
            → handleSubmit(values) # transforma y despacha thunk
            → setSubmitting(true)
            → setIsEditableVersion(true)
```

## `createNewPolicy` — generación de nueva política

- ID = `Math.max(...policies.map(p => p.id)) + 1` (o `1` si la lista está vacía).
- `appliesPortal` y `appliesMicrosite` se inicializan en `!isCollector`.
- `open: true` — el acordeón se expande automáticamente.

## Dependencias clave

| Archivo | Rol |
|---|---|
| `helpers/createPolicyAdapter.ts` → `formValuesToPolicyRequest` | Convierte `FormValues` a `PolicyRequest` para el API. |
| `helpers/policiesAdapter.ts` → `mapApiResponseToFormValues` | Convierte la respuesta del API a `FormValues`. |
| `helpers/formHelpers.ts` → `replaceContentStyle` | Convierte clases Quill (`ql-align-*`) a estilos inline para el preview. |
| `state/PoliciesThunk.ts` → `createPolicies` | Thunk que llama al caso de uso `createPoliciesUseCase`. |
| `state/PoliciesReducer.ts` → `policiesSelector` | Selector del slice `policies`. |
| `state/PoliciesActions.ts` → `PoliciesAction` | Constantes de acción para filtrar el `useEffect`. |
| `constants/constants.ts` → `TypePolicie`, `PortalPayerType` | Enums de tipo de link (`s3`/`url`/`v1`) y claves de canal. |
| `helpers/termsAndConditionsOptions.ts` → `TermsCardEnum` | `COLLECTOR` / `PAYMENT_CHANNELS` — determina el `typePortal` del thunk. |

## Edge cases y advertencias

- **No llamar a `handleSubmit` directamente desde el formulario.** El flujo correcto pasa siempre por `handleClickConfirm`, que valida antes de despachar.
- **El hook no despacha `getPolicies`.** La carga inicial de datos es responsabilidad del componente padre o de un hook separado.
- **`previewPolicies` es estado local**, construido desde los valores en memoria del formulario. No refleja el último guardado en servidor hasta que el padre vuelva a llamar a `openPreview`.
- **`TypePolicie.V1`** se usa siempre como versión al crear (`"v1"`); no hay soporte multi-versión en este hook.
- El tipo `PreviewPolicyItem` está **definido dos veces**: inline en el hook y exportado desde `helpers/policiesAdapter.ts`. Usar el del adapter para consistencia si se necesita fuera del hook.
- `Formik` en `CreatePolicie.tsx` usa `enableReinitialize`, por lo que `initialFormValues` se aplica cada vez que Redux actualiza `policies_data`.
- **Para links tipo S3, `p.linkUrl` siempre contiene la URL del archivo ya subido a S3.** Debe asignarse al campo `url` del request, nunca a `b64`. El campo `b64` (`PolicyLinkRequest`) existe para enviar un PDF en base64 crudo, que no es el flujo real implementado.

## Historial de cambios

| Fecha | Bug | Resumen |
|---|---|---|
| 2026-02-24 | BUG-001 | Fix en `buildLink` (`createPolicyAdapter.ts`): URL S3 movida de `b64` a `url` para que el backend acepte el request |
