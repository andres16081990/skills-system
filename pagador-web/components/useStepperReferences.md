# Skill: useStepperReference

## Ubicación
`src/presentation/agreement/hooks/useStepperReferences.ts`

> Nota: el archivo se llama `useStepperReferences.ts` (plural) pero el hook exportado se llama `useStepperReference` (singular).

## Propósito
Hook central del flujo de convenios. Orquesta la navegación, el avance entre pasos del stepper y el envío de solicitudes de pago. Es el puente entre la UI del stepper y las acciones Redux/thunks del módulo `agreement`.

## Dónde se usa

| Consumidor | Qué usa |
|---|---|
| `References.tsx` (`src/.../pages/References/References.tsx`) | `onHandleButtonBack`, `onHandleSubmit`, `activeStep`, `isDisabledNextButton` |
| `Documents.tsx` (`src/.../pages/Documents/Documents.tsx`) | `onHandleButtonBack`, `onHandleSubmit`, `activeStep`, `isDisabledNextButton` |
| `ReferenceGroup.tsx` (`src/.../ReferencesGroup/ReferenceGroup.tsx`) | Solo `onHandleSubmit` |

**Usado en 3 lugares** → no modificar directamente si el cambio afecta solo un consumidor.

## Valor retornado

| Campo | Tipo | Descripción |
|---|---|---|
| `onHandleButtonBack` | `() => void` | Navega hacia atrás según el paso activo |
| `onHandleSubmit` | `() => void` | Avanza al siguiente paso o genera el código de pago |
| `activeStep` | `number` | Paso actual del stepper (0 = referencias, 1 = documentos) |
| `isDisabledNextButton` | `() => boolean` | Indica si el botón "Siguiente" debe estar deshabilitado |

## Estado Redux leído

### AgreementReducer (`stateSelectorAgreementContext`)
| Campo | Uso |
|---|---|
| `businessIdentifier` | Identificador del negocio para construir URLs de navegación |
| `references` | Detecta la referencia activa (`checked === true`) |
| `selectedAgreement` | Obtiene `agreement_code` e `id` del convenio seleccionado |
| `activeStep` | Paso actual del stepper |
| `selectedDocument` | IDs de documentos seleccionados para construir el request de pago |
| `filteredDocuments` | Busca datos de cada documento seleccionado |
| `agreements` | Verifica si el convenio es externo via `external_api_id` |
| `documents` | Obtiene `key`, `receivable_data_list`, `receivable_instruction_data_list` para convenios externos |
| `processState` | Escucha el resultado de `GET_SHADOW_DOCUMENTS_TO_LINK` |
| `shadowDocuments` | Documentos shadow ya vinculados, usados en `sendGeneratePaymentCode` |

### CommonsReducer (`stateSelectorCommonContext`)
| Campo | Uso |
|---|---|
| `payerSessionId` | Determina la URL de retorno en `onHandleButtonBack` cuando `activeStep === 0` |

## Acciones Redux disparadas

| Acción | Cuándo |
|---|---|
| `setIdleState()` | Al inicio de `onHandleSubmit` para limpiar estado anterior |
| `setActiveStep(0)` | En `onHandleButtonBack` cuando `activeStep > 0` |
| `setSelectedDocument(undefined)` | En `onHandleButtonBack` cuando regresa al paso 0 |
| `getShadowDocumentsToLink(request)` | En `onHandleSubmit` step 1 para convenios externos con `documents.key` |
| `generatePaymentCode(request)` | En `sendGeneratePaymentCode` para generar el código de pago |

## Thunks disparados

| Thunk | Condición |
|---|---|
| `getDocumentsByReference` | Step 0, convenio **no externo** |
| `getExternalDocumentsByReferences` | Step 0, convenio **externo** |
| `getShadowDocumentsToLink` | Step 1, convenio externo con `documents.key` presente |
| `generatePaymentCode` | Step 1, flujo normal o tras éxito de `getShadowDocumentsToLink` |

Todos los thunks de documentos (`getDocumentsByReference`, `getExternalDocumentsByReferences`) se despachan envueltos en `handleReCaptchaVerify` del hook `useHeaders`.

## Lógica de navegación — `onHandleButtonBack()`

```
activeStep === 0
  ├── payerSessionId existe → navega a PORTAL_PAYMENTS + businessIdentifier?payerSessionId=...
  └── no existe             → navega a PORTAL_PAYMENTS + businessIdentifier

activeStep > 0
  └── dispatch setActiveStep(0) + setSelectedDocument(undefined)
      → navega a PORTAL_PAYMENTS_REFERENCES
```

## Lógica de avance — `onHandleSubmit()`

```
dispatch setIdleState()

activeStep === 0
  ├── convenio no externo → handleReCaptchaVerify(getDocumentsByReference, { agreementCode, ... })
  └── convenio externo    → handleReCaptchaVerify(getExternalDocumentsByReferences, { agreementId, ... })

activeStep === 1
  ├── convenio externo && documents.key existe
  │     → dispatch getShadowDocumentsToLink(request)   [continúa en useEffect]
  └── caso normal
        → sendGeneratePaymentCode()
```

## useEffect — encadenamiento tras `getShadowDocumentsToLink`

```tsx
useEffect(() => {
  if (
    processState.action === AgreementActions.GET_SHADOW_DOCUMENTS_TO_LINK &&
    processState.current === "success"
  ) {
    dispatch(setIdleState());
    sendGeneratePaymentCode();  // ← continúa el flujo de pago
  }
}, [processState]);
```

Permite que el flujo de convenios externos complete la vinculación de shadow documents antes de generar el código de pago.

## Lógica de convenio externo — `isCurrentAgreemnetExternal()`

Busca el convenio activo en `agreements` y consulta `external_api_id`:
- `external_api_id = 0` → `isAllowedIdsMap.get(0)` = `false` → **no externo**
- `external_api_id = 1` → `isAllowedIdsMap.get(1)` = `true` → **externo**
- No encontrado → `false`

El mapa `isAllowedIdsMap` está definido en `DocumentLite.ts`.

## Construcción de requests

### `buildDocumentsByReferenceRequest()`
Toma la referencia activa (`references.find(r => r.checked)`) y construye:
```ts
{ references: [{ code: activeReference.reference_type_code, value: activeReference.value }] }
```

### `buildDocumentsToPay()` — convenios no externos
Mapea `selectedDocument` (IDs) a `PaymentRef[]`:
- `payment_ref_id` = parte numérica del ID (`id.split("/")[0]`)
- `payment_ref_type_id` = `1` (RECEIVABLE) o `2` (INSTRUCTION)

### `buildShadowDocumentsToLink(documentsToFilter, selectedIds)` — convenios externos
Extrae los UUIDs de los documentos seleccionados que tienen `uuid`. Recibe tanto `receivable_instruction_data_list` como `receivable_data_list` por separado.

### `sendGeneratePaymentCode()`
Construye `GeneratePaymentCodeRequest`:
- Convenio **no externo**: usa `buildDocumentsToPay()`
- Convenio **externo**: usa `shadowDocuments` del estado (ya vinculados)
- `origin_channel` siempre = `"PAYMENT_PORTAL"`

## Lógica de deshabilitación — `isDisabledNextButton()`

| Paso | Condición para deshabilitar |
|---|---|
| Step 0 | Ninguna referencia tiene `value` Y `checked === true` |
| Step 1 | `selectedDocument` es `undefined` o tiene longitud 0 |

## Dependencias

| Hook/util | Qué aporta |
|---|---|
| `useHeaders` | `getClientIp()`, `handleReCaptchaVerify()`, `ipAddress` |
| `useNavigate` (react-router-dom) | Navegación programática |
| `useAppDispatch` / `useTypedSelector` | Acceso a Redux |

## Tests existentes
No hay tests unitarios para este hook.

## Reglas para modificar este hook

- Usado en **3 lugares** — cualquier cambio en la firma o comportamiento impacta `References.tsx`, `Documents.tsx` y `ReferenceGroup.tsx`
- Si se necesita un comportamiento distinto para un solo consumidor, crear un hook derivado que llame a `useStepperReference` internamente
- El flujo de convenios externos tiene dos pasos asíncronos encadenados (`getShadowDocumentsToLink` → `sendGeneratePaymentCode`); no cortocircuitar el `useEffect` sin entender ese flujo
- `isCurrentAgreemnetExternal` tiene un typo intencional en el nombre (`Agreemnet`) — no renombrar sin actualizar los 3 consumidores
