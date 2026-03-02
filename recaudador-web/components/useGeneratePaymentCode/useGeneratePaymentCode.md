# useGeneratePaymentCode — Agent Skill

## Propósito

Hook personalizado que encapsula la lógica para **generar y copiar al portapapeles un enlace de pago** a partir de un documento de cobro (receivable o instrucción). Gestiona el estado visual del mensaje de confirmación, el comportamiento específico de Safari (que no permite escritura asíncrona al portapapeles sin interacción directa del usuario), y las reglas de habilitación/deshabilitación del botón según el estado del documento y la estrategia de pago del acuerdo.

Vive en:

```
src/presentation/receivable/hooks/useGeneratePaymentCode.ts
```

---

## Firma

```ts
export const useGeneratePaymentCode = (
  documentId: number,
  type: ReceivablesTypesEnum,
  agreementId: number,
  documentState?: DocumentStateEnum,
) => { ... }
```

### Parámetros

| Parámetro | Tipo | Requerido | Descripción |
|---|---|---|---|
| `documentId` | `number` | ✅ | ID del documento de cobro a procesar |
| `type` | `ReceivablesTypesEnum` | ✅ | Tipo de documento: `RECEIVABLE` (0) o `INSTRUCTIONS` (1) |
| `agreementId` | `number` | ✅ | ID del acuerdo al que pertenece el documento |
| `documentState` | `DocumentStateEnum` | ❌ | Estado actual del documento. Controla si el botón está deshabilitado |

### Retorno

| Campo | Tipo | Descripción |
|---|---|---|
| `onGeneratePaymentCode` | `() => void` | Dispara la generación del código de pago y limpia el portapapeles |
| `showMessage` | `boolean` | Indica si debe mostrarse el tooltip de "copiado" |
| `setShowMessage` | `Dispatch<SetStateAction<boolean>>` | Permite al consumidor apagar el tooltip tras un timer |
| `isDisabled` | `boolean` | Si el botón debe estar deshabilitado según estado del documento y estrategia de pago |
| `ref` | `RefObject<any>` | Ref para controlar el `ButtonCard` (necesario en Safari para cerrar el card) |
| `processState` | `ProcessState` | Estado actual del thunk Redux (`action`, `current`) |
| `getClassnameByBrowser` | `() => string` | Retorna `"payment-code-container-safari"` o `"payment-code-container"` según el navegador y el estado |
| `copyLinkText` | `() => Promise<void>` | En Safari: copia el link y cierra el card cuando el usuario hace clic manual |
| `linkText` | `string` | URL completa del enlace de pago (solo se popula en Safari) |
| `disableShadowDocuments` | `() => boolean` | Retorna `true` si el documento es un shadow document no real (`is_document_shadow === false`) |
| `isSafari` | `boolean` | Indica si el navegador activo es Safari |

---

## Consumidores

El hook es usado en **tres lugares**:

| Componente / Página | Parámetros pasados | Qué usa |
|---|---|---|
| `GeneratePaymentCodeOption.tsx` | `documentId`, `type`, `agreementId`, `documentState` | Todos los retornos |
| `GeneratePaymentCodeLink.tsx` | `documentId`, `type`, `agreementId` (sin `documentState`) | `onGeneratePaymentCode`, `showMessage`, `setShowMessage` |
| `ReceivableForm.tsx` | `receivable.id`, `RECEIVABLE`, `agreement.id`, `receivable.state` | Solo `isDisabled` |

> **Regla crítica**: al ser usado en más de un lugar, **nunca modificar el hook original** si el cambio es específico a un consumidor. Crear una versión derivada con nombre descriptivo.

---

## `GeneratePaymentCodeOption` — props adicionales (BUG-008)

`GeneratePaymentCodeOption` es el único consumidor que agrega lógica de deshabilitado **por encima** del hook. Su función `getIsDisabled()` evalúa condiciones en este orden:

| Prioridad | Condición | Fuente |
|---|---|---|
| 1 | `isExternalApiDisabled === true` | Prop externa — indica que el acuerdo tiene `external_api_id === 1` |
| 2 | `disableShadowDocuments()` | Retorno del hook — documento shadow no real |
| 3 | Lógica de `isPayable` | Prop externa — si está definida, sobreescribe `isDisabled` del hook |
| 4 | `isDisabled` del hook | Retorno del hook — basado en estado del doc y estrategia de pago |

### Props del componente `GeneratePaymentCodeOption`

| Prop | Tipo | Descripción |
|---|---|---|
| `documentId` | `number` | ID del documento |
| `type` | `ReceivablesTypesEnum` | Tipo de documento |
| `agreementId` | `number` | ID del acuerdo |
| `documentState` | `DocumentStateEnum` | Estado del documento |
| `isPayable` | `boolean \| undefined` | Si está definido, sobreescribe la lógica de `isDisabled` |
| `isExternalApiDisabled` | `boolean \| undefined` | Si `true`, deshabilita el botón antes de cualquier otra evaluación |

### Origen de `isExternalApiDisabled`

**No viene de `agreementDetail.agreement`** — el endpoint de detalle no devuelve `external_api_id`. Se calcula en los hooks de tabla leyendo desde `AgreementState.agreements` (lista):

```ts
// En useDataDocumentTable y useMapDataDocumentTable:
const isExternalApiDisabled =
  agreements.find((a) => a.agreement_code === agreement.code)
    ?.external_api_id === 1;
```

Luego se propaga por: `moreActionsProps` → `getMoreActions` → `GeneratePaymentCodeOption`.

---

## Redux — Slices que consume

| Slice | Selector | Campos que lee |
|---|---|---|
| `ReceivableState` | `stateSelectorReceivable` | `lastPaymentCode`, `lastPaymentCodeDocumentId`, `processState` |
| `AgreementState` | `stateSelectorAgreement` | `agreementDetail.agreement.payment_strategy.strategy_rules` |
| `UserContextState` | `stateSelectorUserContext` | `businessIdentifier` |
| `PaymentDocumentState` | `stateSelectorPaymentDocument` | `paymentDocumentList` |

## Redux — Acciones que despacha

| Acción | Cuándo |
|---|---|
| `resetLastPaymentCode()` | Después de copiar al portapapeles (non-Safari), o después del copy manual (Safari) |
| `setLastPaymentCodeDocumentId(documentId)` | Al iniciar la generación del código, para rastrear a qué documento pertenece |
| `generatePaymentCode(request)` | Thunk principal que llama a la API |

---

## Flujo principal (non-Safari)

```
onGeneratePaymentCode()
  ├─ validateLastPaymentCode() → si hay código vigente en Safari, sale (guard)
  ├─ dispatch(setLastPaymentCodeDocumentId(documentId))
  ├─ navigator.clipboard.writeText("")   ← limpia portapapeles
  └─ dispatch(generatePaymentCode(request))
         │
         └─ [useEffect] processState.current === "success" &&
                          lastPaymentCode && documentId === lastPaymentCodeDocumentId
              ├─ navigator.clipboard.writeText(`${path}${lastPaymentCode}`)
              ├─ setShowMessage(true)   ← muestra tooltip
              └─ dispatch(resetLastPaymentCode())
```

## Flujo Safari

En Safari, la escritura al portapapeles solo funciona en el contexto directo del evento `click`. Por eso el flujo es diferente:

```
onGeneratePaymentCode()
  └─ dispatch(generatePaymentCode(request))
         │
         └─ [useEffect] processState === "success"
              └─ setLinkText(`${path}${lastPaymentCode}`)
                   ← NO copia al portapapeles aquí

copyLinkText()   ← llamada desde el botón del ButtonCard (interacción directa del usuario)
  ├─ navigator.clipboard.writeText(linkText)
  ├─ setShowMessage(true)
  ├─ dispatch(resetLastPaymentCode())
  └─ ref.current.closeCard()
```

`getClassnameByBrowser()` retorna `"payment-code-container-safari"` cuando el estado es `success` + hay código + es Safari, lo que hace que `GeneratePaymentCodeOption` renderice el `ButtonCard` con `variant="variant-a"` y el offset de posición.

---

## Lógica de `isDisabled`

```
documentState === OVERDUE
  ├─ type === INSTRUCTIONS → true (siempre deshabilitado)
  └─ type === RECEIVABLE
       └─ agreementDetail tiene regla ALLOW_PAYMENT_UNTIL_DUE_DATE → true

documentState ∈ [DISABLED, EXPIRED, PAID] → true

todos los demás casos → false
```

> `GeneratePaymentCodeOption` tiene su propia función `getIsDisabled()` que **sobreescribe** este valor si `isPayable` está definido (viene como prop). Usa `isDisabled` del hook como fallback cuando `isPayable === undefined`.

---

## Lógica de `disableShadowDocuments`

Busca en `paymentDocumentList` si existe un documento cuyo `id` (parte antes del `-`) coincide con `documentId`. Si el documento encontrado tiene `is_document_shadow === false` (explícitamente `false`, no `undefined`), retorna `true` (deshabilitar).

Esto deshabilita el botón para documentos que son "sombra" (duplicados no reales), evitando la generación de links incorrectos.

---

## Variable de entorno requerida

```
REACT_APP_HOST_PAYMENT_LINK_URL
```
Se usa para construir la URL completa del enlace: `${REACT_APP_HOST_PAYMENT_LINK_URL}${lastPaymentCode}`.

---

## Dependencias externas

| Dependencia | Uso |
|---|---|
| `react-device-detect` → `isSafari` | Bifurca el comportamiento de clipboard |
| `ReceivablesTypesEnum` | Determina `isReceivable` en el request: `type === RECEIVABLE` |
| `PaymentStrategyRulesCodeEnum.ALLOW_PAYMENT_UNTIL_DUE_DATE` | Regla de estrategia que bloquea pagos vencidos |
| `DocumentStateEnum` | Estados del documento que controlan `isDisabled` |

---

## Puntos frágiles

### 1. `useEffect` con dependencias incompletas

```ts
useEffect(() => { ... }, [processState.action, lastPaymentCode]);
```

No incluye `documentId` ni `lastPaymentCodeDocumentId` en el array de dependencias, aunque ambos se leen dentro del efecto. Si `documentId` cambia entre renders sin que cambie `lastPaymentCode`, el efecto no se dispara. Esto es especialmente relevante si el mismo componente re-renderiza con otro `documentId`.

### 2. Guard `validateLastPaymentCode` solo aplica en Safari

```ts
const validateLastPaymentCode = () => {
  return lastPaymentCode && documentId === lastPaymentCodeDocumentId && isSafari;
};
```

En non-Safari no hay guard: si el usuario hace clic rápidamente dos veces, se despachan dos thunks. El segundo sobreescribe `lastPaymentCodeDocumentId` y el link del primero puede no copiarse correctamente.

### 3. `disableShadowDocuments` usa split por `"-"` en `p.id`

```ts
const [id] = p.id.split("-");
return +id === documentId;
```

Si el formato de `id` cambia (e.g., UUID sin guión, o guión en otra posición), la comparación falla silenciosamente y todos los documentos quedan habilitados.

### 4. `copyLinkText` depende de `ref.current.closeCard()`

Si `ref.current` es `null` o no tiene el método `closeCard`, el `await navigator.clipboard.writeText(linkText)` se ejecuta pero el card no se cierra. No hay manejo de error para ese caso.

---

## Reglas — Lo que NUNCA debes hacer

- **No muevas la lógica de clipboard a fuera del `useEffect`** para non-Safari — la copia debe ocurrir en el contexto del efecto reactivo, no en el click handler, porque en ese punto el thunk aún no ha resuelto.
- **No elimines el `navigator.clipboard.writeText("")` del click handler** — limpia el portapapeles antes de despachar para evitar que el usuario vea un link anterior si el thunk falla.
- **No cambies `isSafari` por una detección más granular** sin revisar los tres consumidores — el comportamiento de `copyLinkText` y `getClassnameByBrowser` depende de este valor de módulo.
- **No confundas `lastPaymentCode` (el código) con `lastPaymentCodeDocumentId` (el ID del doc)** — son dos campos distintos del slice y ambos se validan en conjunto para confirmar que el código pertenece al documento activo.
- **No modifiques este hook directamente para cambiar comportamiento en un solo consumidor** — créa una versión derivada con nombre descriptivo (ej. `useGeneratePaymentCodeForForm`).

---

## Tests

### Ubicación

```
src/presentation/receivable/hooks/__test__/useGeneratePaymentCode.test.tsx
```

### Cómo correrlos

```bash
pnpm test -- src/presentation/receivable/hooks/__test__/useGeneratePaymentCode.test.tsx
```

### Qué cubren (5 casos)

| Caso | Qué verifica |
|---|---|
| `copies to clipboard and dispatches resetLastPaymentCode when not Safari` | El `useEffect` de mount copia al portapapeles y despacha `resetLastPaymentCode` cuando `processState === success` |
| `dispatches setLastPaymentCodeDocumentId and generatePaymentCode on onGeneratePaymentCode` | Orden y contenido de los 3 dispatches al llamar `onGeneratePaymentCode` |
| `isDisabled=true for INSTRUCTIONS when documentState is OVERDUE` | La lógica de `isDisabled` para el caso INSTRUCTIONS + OVERDUE |
| `getClassnameByBrowser returns default class when isSafari=false` | Retorna `"payment-code-container"` cuando no es Safari |
| `copyLinkText does nothing when linkText is empty` | No escribe al portapapeles ni hace dispatch adicional si `linkText` está vacío |

> Los tests usan `renderHook` de `@testing-library/react-hooks`. Mockean `react-device-detect` con `isSafari: false` a nivel de módulo (no por test individual), lo que significa que **no hay tests del flujo Safari** actualmente.

---

## Historial de cambios

| Fecha | Motivo | Resumen |
|---|---|---|
| 2026-02-27 | Creación del skill | Documentación inicial del hook a partir de lectura directa del código. |
