# MassiveSelectionFilter — Agent Skill

## Propósito

`MassiveSelectionFilter` es la barra de acciones masivas sobre documentos seleccionados. Muestra iconos-botones (Eliminar, Marcar como pagado) en función de la selección activa y gestiona el flujo de eliminación masiva en dos pasos:

1. **Pre-confirmación** (`massiveDeletionDocuments`): envía los filtros y exclusiones al API, que responde con un resumen de cuántos documentos son eliminables y cuántos no.
2. **Confirmación** (`massiveDeletionDocumentsConfirm`): envía el `uuid` de la operación para ejecutar la eliminación real.

También embebe el componente `MassiveDeleteModals` que presenta los modales de confirmación (variant-a = aviso inicial, variant-b = resumen de eliminación).

Vive en:
```
src/presentation/receivable/components/MassiveSelectionFilter/MassiveSelectionFilter.tsx
```

---

## Comunicación con el resto de la app

### Cómo recibe datos

| Fuente | Qué recibe |
|---|---|
| Prop `data` | Lista de opciones a renderizar como botones (`DropdownData[]`) — construida externamente por el padre |
| Prop `isMassiveSelectionAllowedToDelete` | Booleano que indica si el tipo de documento activo permite eliminación masiva (solo tipos únicos, no "ALL") |
| Prop `isMassiveSelectionDelete` | Booleano que indica si el modo de selección masiva está activo |
| Prop `typeDocument` | Tipo de documento activo en la lista ("RECEIVABLE", "INSTRUCTION", "ALL", etc.) |
| Prop `isFromReceivable` | Booleano que indica si el contexto es el módulo de Recaudables/Instrucciones (`true`) o PaymentDocuments (`false`) |
| `PaymentDocumentState` (Redux) | `paymentDocumentrequest`, `excludeIds`, `pagination` — usados para construir el request de eliminación masiva |
| `ReceivableState` (Redux) | `processState`, `massiveDeletionRes`, `receivableListRequest`, `instructionListRequest`, `receivableList`, `instructionList`, `pagination` |
| `localStorage["showMassiveDeleteMessage"]` | Si `"true"`, omite el modal de confirmación `variant-a` y ejecuta directamente la eliminación |

### Cómo emite datos

| Mecanismo | Qué emite |
|---|---|
| `handleOnChange(value)` | Notifica al padre con el valor enum del botón pulsado cuando **no** es eliminación masiva (`isMassiveSelectionAllowedToDelete=false`) |
| `getModalMassiveDeletionOptions(modalsOptions)` | Notifica al padre en cada cambio de `modalsOptions` (vía `useEffect`) — el padre usa esto para actualizar su mensaje de resultado de eliminación |
| `dispatch(massiveDeletionDocuments)` | Thunk Redux — inicia el paso 1 de eliminación masiva |
| `dispatch(massiveDeletionDocumentsConfirm)` | Thunk Redux — ejecuta el paso 2 de eliminación masiva usando el uuid del paso 1 |
| `dispatch(setIdleState())` | Resetea el `processState` del slice Receivable al cerrar el modal `variant-b` |

### Con qué componentes se comunica directamente

| Componente | Rol |
|---|---|
| `MassiveDeleteModals` | Presenta los modales de confirmación de eliminación masiva (variant-a y variant-b). Recibe estado y callbacks del hook |
| `Tooltip` (design system) | Wrappea cada botón de acción — muestra el label o el texto de deshabilitado según `disableDeleteButton` |
| `BodyText` (design system) | Contenido del tooltip cuando el botón está deshabilitado |
| `DeleteIcon` / `PriceCheckIcon` (MUI) | Iconos de las acciones Eliminar / Marcar como pagado |

### A qué páginas o layouts pertenece

| Página / Componente padre | `isFromReceivable` | `typeDocument` | Opciones en `data` |
|---|---|---|---|
| `PaymentDocumentMain` | `false` | Varía (`"RECEIVABLE"`, `"INSTRUCTION"`, `"ALL"`) | Construido por `useHandleOptionsMassiveSelect` — puede incluir DELETE y/o MARK_AS_PAID según permisos y feature flag |
| `ReceivableTab` | `true` | `"RECEIVABLE"` o `"INSTRUCTION"` | Solo DELETE (hardcodeado en `ReceivableTab`) |

---

## Props

| Prop | Tipo | Requerida | Qué hace | De dónde viene |
|---|---|---|---|---|
| `data` | `DropdownData[]` | Sí | Lista de opciones de acción masiva a renderizar. Si está vacía, no se muestra el contenedor de botones pero sí `MassiveDeleteModals`. | `useHandleOptionsMassiveSelect.getDropDownDataMassiveSelections()` en `PaymentDocumentMain`, o array hardcodeado en `ReceivableTab` |
| `handleOnChange` | `(value: string) => void` | Sí | Callback cuando se pulsa un botón y `isMassiveSelectionAllowedToDelete=false`. Recibe el valor del enum `MassiveSelectionValueEnum`. | `useHandleOptionsMassiveSelect.handleOnChangeMassiveOpt` en `PaymentDocumentMain`, o lambda inline en `ReceivableTab` |
| `isMassiveSelectionAllowedToDelete` | `boolean` | Sí | Cuando `true`, cualquier click en un botón dispara `handleMassiveDelete()` (flujo de eliminación masiva). Cuando `false`, se llama `handleOnChange`. | Calculado en el padre: `true` si el modo masivo está activo Y `typeDocument !== "ALL"` |
| `isMassiveSelectionDelete` | `boolean` | Sí | Cuando `true`, filtra `data` para mostrar solo los items con `value === DELETE`. | Estado local en `PaymentDocumentMain` (`isMassiveSelectionDelete`) |
| `translate` | `TFunction` | Sí | Función de traducción. Se usa para las dos líneas del tooltip cuando el botón está deshabilitado (`disableDeleteButtonTooltip`, `disableDeleteButtonTooltipTwo`). | `useTranslation` del padre con keyPrefix `properties.common.components.counterSelectedRows` |
| `typeDocument` | `string` | Sí | Tipo de documento activo. Determina qué campo de fecha usar al construir `buildMasiveDeletionReq` y qué lista buscar para `agreementSelected`. | `documentTypeByFulfillmentRequest` en `PaymentDocumentMain`, o derivado de `type` en `ReceivableTab` |
| `isFromReceivable` | `boolean` | Sí | Determina la fuente del request de eliminación: `true` usa `receivableListRequest`/`instructionListRequest`, `false` usa `paymentDocumentrequest`. | Hardcodeado: `false` en `PaymentDocumentMain`, `true` en `ReceivableTab` |
| `getModalMassiveDeletionOptions` | `(modalOptions: ModalsOptionsType) => void` | Sí | Callback invocado cada vez que `modalsOptions` cambia dentro del hook. El padre lo usa para leer el mensaje de resultado de la eliminación masiva. | `getModalMassiveDeletionOptions` en `PaymentDocumentMain`, `getModalMassiveDeletionOptions` en `ReceivableTab` |

---

## Hooks y dependencias

### Hook interno: `useMassiveSelectionFilter`

El componente delega toda la lógica a este hook. Recibe los mismos `isMassiveSelectionAllowedToDelete`, `isMassiveSelectionDelete`, `typeDocument`, `isFromReceivable`, `getModalMassiveDeletionOptions`.

#### Datos que consume del store

| Selector | Campos leídos |
|---|---|
| `stateSelectorPaymentDocument` | `paymentDocumentrequest`, `excludeIds`, `pagination` |
| `stateSelectorReceivable` | `processState`, `massiveDeletionRes`, `receivableListRequest`, `instructionListRequest`, `receivableList`, `instructionList`, `pagination` (como `receivablePagination`) |

#### Estado local del hook

| Estado | Propósito |
|---|---|
| `modalsOptions: ModalsOptionsType` | Controla qué modal está abierto, qué variante y con qué datos (`variantbStructure`) |
| `buildMasiveDeletionReq: MassiveDeletionRequest` | Request construido a partir de la última consulta activa, listo para enviarse en `confirmDeletion` |

#### Effects del hook

| Effect | Dependencias | Qué hace |
|---|---|---|
| `useEffect` #1 | `[modalsOptions]` | Llama `getModalMassiveDeletionOptions(modalsOptions)` — notifica al padre |
| `useEffect` #2 | `[isFromReceivable, typeDocument, excludeIds, receivableListRequest, instructionListRequest, paymentDocumentrequest]` | Reconstruye `buildMasiveDeletionReq` usando `adaptToListRequest` o `adaptPaymentToMassiveDeletion` |
| `useEffect` #3 | `[processState]` | Cuando `MASSIVE_DELETION_DOCUMENTS` → `success`: abre modal `variant-b` con datos de `massiveDeletionRes`. También llama `setIdleState()` **sin dispatch** (ver Puntos frágiles) |

#### Funciones expuestas por el hook

| Función | Qué hace |
|---|---|
| `disableDeleteButton(value)` | Retorna `true` si `value === DELETE && !isMassiveSelectionAllowedToDelete && isMassiveSelectionDelete` |
| `handleMassiveDelete()` | Verifica localStorage `showMassiveDeleteMessage`; si es `"true"` llama `confirmDeletion()` directamente, si no, abre el modal `variant-a` |
| `handleOnCloseModal()` | Cierra el modal; si era `variant-b` también resetea a `variant-a` y despacha `setIdleState()` |
| `confirmDeletion()` | Si `variant-b`: despacha `massiveDeletionDocumentsConfirm({ type, uuid })`; si no: despacha `massiveDeletionDocuments({ request, type })` |
| `showMassiveDeleteMoldal()` | Lee `localStorage["showMassiveDeleteMessage"]` y retorna `true` si es `"true"` |

#### `useLocalStorage` (de `common/hooks`)

Wrappea `localStorage.getItem` / `localStorage.setItem`. Se usa para:
- Leer `showMassiveDeleteMessage` (omitir modal variant-a)
- Escribir `showMassiveDeleteMessage` cuando el usuario marca la opción "no mostrar de nuevo" en `useMassiveDeleteModals`

---

## Flujo de datos

### Escenario A — Botón no DELETE (ej: Marcar como pagado)

```
Usuario pulsa el botón MARK_AS_PAID
  → isMassiveSelectionAllowedToDelete=false → handleOnChange("MARK_AS_PAID") es llamado
  → Padre (PaymentDocumentMain via useHandleOptionsMassiveSelect) recibe el valor
      → setOpenMarkAsPaidModal(true) → abre el modal de confirmación de pago
```

### Escenario B — Botón DELETE en modo no-masivo (`isMassiveSelectionAllowedToDelete=false`)

```
Usuario pulsa el botón DELETE (selección parcial, tipo ALL o sin modo masivo activo)
  → handleOnChange("DELETE") es llamado en el padre
  → Padre llama getDeleteData() → construye y presenta el modal de confirmación de su propia lógica
```

### Escenario C — Eliminación masiva completa (flujo de 2 pasos)

```
Usuario pulsa DELETE con isMassiveSelectionAllowedToDelete=true
  → handleMassiveDelete() es llamado

  Rama 1 — localStorage["showMassiveDeleteMessage"] !== "true":
    → setmodalsOptions({ isOpenModal: true, variantModal: "variant-a" })
    → MassiveDeleteModals muestra modal variant-a (aviso de confirmación)
    → Usuario puede marcar "No mostrar de nuevo" → localStorage se actualiza
    → Usuario presiona Confirmar → confirmDeletion()

  Rama 2 — localStorage["showMassiveDeleteMessage"] === "true":
    → confirmDeletion() es llamado directamente (sin modal)

  confirmDeletion() — primer paso:
    → dispatch(massiveDeletionDocuments({ request: buildMasiveDeletionReq, type: typeDocument }))
    → Reducer: loading → success → massiveDeletionRes = { uuid, deletable_count, non_deletable_count, non_deletable_by_role }
    → useEffect([processState]) detecta MASSIVE_DELETION_DOCUMENTS success
        → setmodalsOptions({ isOpenModal: true, variantModal: "variant-b", variantbStructure: { modalInfo, documentCount } })
    → useEffect([modalsOptions]) dispara → getModalMassiveDeletionOptions(modalsOptions) → padre actualiza su mensaje

  MassiveDeleteModals muestra modal variant-b (resumen de eliminables / no eliminables)
    → Si deletable_count === 0: el botón de confirmar está oculto (solo cerrar)
    → Si deletable_count > 0: usuario puede confirmar

  Usuario confirma en variant-b → confirmDeletion() de nuevo:
    → dispatch(massiveDeletionDocumentsConfirm({ type, uuid: modalsOptions.variantbStructure.modalInfo.uuid }))
    → Reducer: loading → success → processState.action === MASSIVE_DELETION_DOCUMENTS_CONFIRM
    → El padre (PaymentDocumentMain) detecta este success en su propio useEffect y lanza un nuevo getPaymentDocumentList
```

### Flujo de construcción del request (`buildMasiveDeletionReq`)

```
Cuando cambian: isFromReceivable, typeDocument, excludeIds, receivableListRequest, instructionListRequest, paymentDocumentrequest

  isFromReceivable=false:
    → adaptPaymentToMassiveDeletion(paymentDocumentrequest, excludeIds)
        → exclude_ids = excludeIds.map(id => +id.split("-")[0])   ← parsea formato "id-type"

  isFromReceivable=true, typeDocument="RECEIVABLE":
    → adaptToListRequest(receivableListRequest, excludeIds)

  isFromReceivable=true, typeDocument="INSTRUCTION":
    → adaptToListRequest(instructionListRequest, excludeIds)

  adaptToListRequest:
    1. Traduce los state values (ej: "TO_PAY") a nombres (ej: "Por pagar") usando RECEIVABLE_STATES + INSTRUCIONS_STATES
    2. Busca esos nombres en ALL_DOCUMENTS_STATES para obtener los IDs numéricos del API
    3. Busca agreement_code del primer documento de la lista actual (receivableList/instructionList)
       que coincida con el agreementId del request
```

---

## Dependencias críticas

| Dependencia | Rol |
|---|---|
| `useMassiveSelectionFilter` | Hook central — contiene toda la lógica de estado, Redux y flujo de eliminación |
| `MassiveDeleteModals` | Presenta los modales de confirmación; lee variante y opciones de `modalsOptions` |
| `MassiveSelectionValueEnum` (`receivable/types/receivable.d.ts`) | `DELETE = "DELETE"`, `MARK_AS_PAID = "MARK_AS_PAID"`. Usado para filtrar `data`, identificar acciones y mapear el enum en el handler |
| `massiveDeletionDocuments` (thunk) | Paso 1: obtiene el resumen de eliminación del API |
| `massiveDeletionDocumentsConfirm` (thunk) | Paso 2: confirma y ejecuta la eliminación usando el uuid |
| `ReceivableAction.MASSIVE_DELETION_DOCUMENTS` | Acción vigilada en `useEffect([processState])` para detectar cuándo abrir el modal `variant-b` |
| `RECEIVABLE_STATES`, `INSTRUCIONS_STATES`, `ALL_DOCUMENTS_STATES` | Usados para traducir valores de estado entre los formatos de Receivable y PaymentDocument |
| `localStorage["showMassiveDeleteMessage"]` | Preferencia del usuario para omitir el modal `variant-a`. Gestionada por `useMassiveDeleteModals` |
| `PaymentDocumentState.paymentDocumentrequest` | Request de la última búsqueda en PaymentDocuments — base para `adaptPaymentToMassiveDeletion` |
| `ReceivableState.receivableListRequest` / `instructionListRequest` | Request de la última búsqueda en Recaudables/Instrucciones — base para `adaptToListRequest` |
| `excludeIds` (Redux) | IDs excluidos de la selección masiva. Se mapean a `exclude_ids: number[]` en el request |

---

## Puntos frágiles

### 1. `setIdleState()` sin `dispatch` en `useEffect([processState])`

```ts
// useMassiveSelectionFilter.tsx — línea ~276
useEffect(() => {
  if (
    processState.current === "success" &&
    processState.action === ReceivableAction.MASSIVE_DELETION_DOCUMENTS
  ) {
    setmodalsOptions({ ... });
  }
  setIdleState(); // ← NO está wrapeado en dispatch(). La acción se crea pero no se despacha.
}, [processState]);
```

Esto significa que el `processState` de Receivable **nunca se resetea a idle** después de que `MASSIVE_DELETION_DOCUMENTS` resuelve con éxito. Si otro componente observa ese `processState` sin un guard adicional, puede reaccionar incorrectamente.

### 2. `getModalMassiveDeletionOptions` se dispara en TODOS los cambios de `modalsOptions`

```ts
useEffect(() => {
  getModalMassiveDeletionOptions(modalsOptions);
}, [modalsOptions]);
```

Esto incluye cuando el modal simplemente se cierra (`isOpenModal: false`). El padre recibirá la notificación aunque no haya cambiado el contenido relevante del mensaje. El padre debe ser idempotente o guardar el mensaje solo cuando `variantbStructure` esté presente.

### 3. Traducción de estados en `adaptToListRequest` depende de coincidencia exacta de nombres

La función traduce values de estado (ej: `"TO_PAY"`) a nombres en español (ej: `"Por pagar"`) y luego los busca en `ALL_DOCUMENTS_STATES` por nombre. Si algún `name` difiere entre listas (tildes, mayúsculas, espacios), el estado se pierde silenciosamente y la eliminación masiva operará sobre todos los estados.

### 4. `getAgreementSelectedByAgreementId` solo busca en la página actual

```ts
receivableList.find((p) => p.agreement_id === +agreementSelected)?.agreement_code ?? ""
```

`receivableList` solo contiene los documentos de la página actual. Si el acuerdo del request no aparece en la página visible, `agreementSelected` será `[""]`, generando un request de eliminación incorrecto o vacío.

### 5. El nombre del tipo y del componente son idénticos

```ts
export type MassiveSelectionFilter = { ... };     // tipo de props
export const MassiveSelectionFilter = (...) => { ... }; // componente
```

TypeScript resuelve el conflicto por contexto, pero puede generar confusión al importar. Al agregar nuevas props, asegúrate de editar el `type`, no solo la desestructuración del componente.

### 6. `adaptPaymentToMassiveDeletion` parsea IDs con `split("-")[0]`

```ts
exclude_ids: excludeIds.map((p) => +p.split("-")[0]),
```

Los IDs en `PaymentDocumentState` tienen formato `"${id}-${type}"`. Si el formato cambia, este parseo fallará silenciosamente (producirá `NaN` convertido a `0`). Está acoplado al formato establecido en `PaymentDocumentReducer`.

### 7. El botón de confirmación del modal `variant-b` se oculta condicionalmente en `MassiveDeleteModals`

```ts
hidePrimaryButton={
  modalOptions.variantModal === "variant-b" &&
  !modalOptions.variantbStructure?.modalInfo.deletable_count
}
```

Si `deletable_count` es `0` o `undefined`, el botón confirmar se oculta. Esto es correcto para el caso de que no haya nada eliminable, pero si `variantbStructure` no se pobló correctamente, el botón desaparece aunque debería mostrarse.

---

## Reglas — Lo que NUNCA debes hacer

1. **No dispatches `massiveDeletionDocuments` o `massiveDeletionDocumentsConfirm` fuera del hook `useMassiveSelectionFilter`**. El estado `modalsOptions` y el `buildMasiveDeletionReq` viven en el hook y deben estar sincronizados con los dispatches.

2. **No modifiques el formato del id en `PaymentDocumentState` (`"${id}-${type}"`) sin actualizar `adaptPaymentToMassiveDeletion`**. El parseo `split("-")[0]` depende de ese formato exacto.

3. **No agregues un tercer paso al flujo de eliminación masiva sin actualizar `confirmDeletion`**. La función distingue entre `variant-b` y "primer paso" usando `modalsOptions.variantModal`. Un tercer paso requeriría una nueva variante.

4. **No cambies los `name` de los estados en `RECEIVABLE_STATES`, `INSTRUCIONS_STATES` o `ALL_DOCUMENTS_STATES` sin verificar `adaptToListRequest`**. La traducción cross-lista depende de coincidencia exacta de nombres en español.

5. **No pases `isFromReceivable=true` desde `PaymentDocumentMain`** (ni viceversa). Cada página tiene su propia fuente de request y las funciones de adaptación son incompatibles entre sí.

6. **No envuelvas `setIdleState()` en un `dispatch()` en el `useEffect([processState])` del hook sin antes revisar si el padre o algún sibling ya lo está haciendo**. Actualmente no se despacha (ver punto frágil #1). Arreglarlo requiere confirmar que no hay doble reset.

---

## Cómo diagnosticar y corregir bugs

### La eliminación masiva no se dispara al confirmar en el modal variant-a

1. Verificar que `isMassiveSelectionAllowedToDelete=true` llegue al componente — sin eso, el click llama `handleOnChange` en lugar de `handleMassiveDelete`.
2. Verificar `localStorage["showMassiveDeleteMessage"]` — si es `"true"`, el modal se omite y `confirmDeletion()` se llama directamente; si la llamada API falla, puede parecer que el modal no funciona.
3. Verificar que `buildMasiveDeletionReq` está correctamente construido: loguear su valor en el `useEffect` #2 del hook.

### El modal variant-b no aparece después de dispatchar `massiveDeletionDocuments`

1. Verificar que `processState.action === ReceivableAction.MASSIVE_DELETION_DOCUMENTS` y `processState.current === "success"` en el momento que el `useEffect` corre.
2. Verificar que `massiveDeletionRes` en `ReceivableState` tiene datos: si el API retornó un payload vacío, `variantbStructure.modalInfo` estará vacío.
3. Recordar el punto frágil #1: `setIdleState()` sin dispatch hace que `processState` no se resetee, por lo que el `useEffect` podría no re-dispararse si el usuario intenta la operación nuevamente (las dependencias no cambian).

### El mensaje de resumen en el padre (`getModalMassiveDeletionOptions`) siempre llega vacío

1. Verificar que el `useEffect([modalsOptions])` está corriendo: si `modalsOptions` no cambia, el callback no se invoca.
2. Verificar que `modalsOptions.variantbStructure` esté presente cuando el padre intenta leer el mensaje — el effect se dispara también al cerrar el modal (cuando `variantbStructure` puede ya no existir).
3. El padre debe leer `modalsOptions.variantbStructure?.modalInfo` solo cuando `variantModal === "variant-b"`.

### El request de eliminación masiva envía `agreementSelected: [""]`

1. Este es el escenario del punto frágil #4. Verificar que `receivableList` o `instructionList` contiene documentos del acuerdo buscado.
2. Verificar el valor de `request.agreementId` en `receivableListRequest` o `instructionListRequest` — si es `0` o undefined, `getAgreementSelectedByAgreementId` nunca encontrará coincidencia.
3. En `isFromReceivable=false` (PaymentDocuments), esto no aplica — `adaptPaymentToMassiveDeletion` usa `req.agreementSelected` directamente del `paymentDocumentrequest`.

### El botón DELETE está siempre deshabilitado en modo masivo

1. Verificar `disableDeleteButton("DELETE")`: retorna `true` cuando `!isMassiveSelectionAllowedToDelete && isMassiveSelectionDelete`.
2. Si `isMassiveSelectionAllowedToDelete=false` y `isMassiveSelectionDelete=true`, el botón se deshabilita porque el tipo de documento es "ALL" (mixed). Esto es el comportamiento esperado — la eliminación masiva no está permitida cuando hay mezcla de tipos.
3. Verificar `typeDocument` en el padre — si es `"ALL"`, el padre debe calcular `isMassiveSelectionAllowedToDelete=false`.

---

## Tests

**Ubicación:**
```
src/presentation/receivable/components/MassiveSelectionFilter/__test__/MassiveSelectionFilter.test.tsx
```

**Cómo correrlos:**
```bash
pnpm test -- src/presentation/receivable/components/MassiveSelectionFilter/__test__/MassiveSelectionFilter.test.tsx
```

**Estructura de los tests:**

| Test | Qué verifica |
|---|---|
| `renders the main container, the button list, and the modal` | Que el contenedor, ambos botones (DELETE y APPROVE) y `MassiveDeleteModals` se renderizan cuando `isMassiveSelectionDelete=false` |
| `when isMassiveSelectionDelete=true, it only shows the DELETE button` | Que el filtro por `value === DELETE` funciona correctamente |
| `when isMassiveSelectionAllowedToDelete=false, clicking DELETE calls handleOnChange` | Que el click llama `handleOnChange("DELETE")` y no `handleMassiveDelete` |
| `when isMassiveSelectionAllowedToDelete=true, clicking DELETE calls handleMassiveDelete` | Que el click llama `handleMassiveDelete` y no `handleOnChange` |
| `disables the DELETE button when disableDeleteButton returns true` | Que el botón queda `disabled`, la clase CSS de deshabilitado se aplica y el Tooltip muestra las dos claves de traducción del mensaje |

**Cobertura ausente (gaps detectados):**
- No hay tests para el flujo completo de eliminación en 2 pasos (requeriría mockear el store Redux y los thunks)
- No hay tests para `useMassiveSelectionFilter` directamente (solo está mockeado en los tests del componente)
- No hay tests para `adaptToListRequest` ni `adaptPaymentToMassiveDeletion`
- No hay tests para el comportamiento con `data=[]` (el contenedor de botones no debe renderizarse pero `MassiveDeleteModals` sí)
- No hay tests para la integración con `localStorage["showMassiveDeleteMessage"]`

**Notas sobre los mocks:**
- `useMassiveSelectionFilter` está completamente mockeado — los tests verifican el comportamiento del componente dado un retorno controlado del hook
- `MassiveDeleteModals` está mockeado como `<div data-testid="massive-delete-modals" />` — los tests no verifican su apertura ni contenido
- El mock de `Tooltip` renderiza el `title` directamente, lo que permite verificar el contenido del tooltip deshabilitado
