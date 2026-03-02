# Skill: DocumentsTable

## Ubicación
```
src/presentation/agreement/components/DocumentsStep/DocumentsTable/
├── DocumentsTable.tsx          ← componente principal
├── DocumentsTable.scss
├── useDocumentTableHeaders.tsx ← hook: columnas legacy + lógica del modal
├── useHeadersRowsTable.tsx     ← hook: columnas y filas para el modo migrado
├── useMapDocumentToShowDetail.ts ← hook: mapeo de documentos al modal de detalle
└── __test__/
    └── useHeadersRowsTable.test.tsx
```

## Propósito
Tabla de documentos de cobro que permite al usuario seleccionar qué documentos pagar. Funciona en dos contextos:
- **`secondStep = false`** → flujo de convenios (`DocumentsStep`), modo selección simple o múltiple
- **`secondStep = true`** → flujo de PaymentLink (`DocumentTableStep`), modo multiselección con edición de valor a pagar

## Props

| Prop         | Tipo      | Requerida | Descripción                                               |
|--------------|-----------|-----------|-----------------------------------------------------------|
| `secondStep` | `boolean` | Sí        | `true` para el flujo PaymentLink; `false` para convenios  |

## Dónde se usa

| Consumidor | Contexto |
|---|---|
| `DocumentsStep.tsx` (`src/.../DocumentsStep/DocumentsStep.tsx`) | Flujo de convenios (paso 1) |
| `DocumentTableStep.tsx` (`src/.../paymentLink/components/DocumentTableStep/DocumentTableStep.tsx`) | Flujo PaymentLink (paso 2) |

## Feature Flag

| Flag | Constante | Efecto |
|---|---|---|
| `MIGRATE_PAYMENT_DOCUMENT_TABLE` | `src/common/constants/featureFlags.ts` | `true` → usa `headers`/`rows` de `useHeadersRowsTable` (modo migrado); `false` → usa `getHeadersColumns` de `useDocumentTableHeaders` (legacy) |

## Estado Redux leído

### AgreementReducer (`stateSelectorAgreementContext`)
- `filteredDocuments` — documentos filtrados para el flujo de convenios
- `selectedDocument` — IDs seleccionados en convenios
- `paymentStrategy` — reglas de la estrategia de pago
- `references`, `values`, `documents`, `selectedAgreement` — usados en los hooks internos

### PaymentLinkReducer (`stateSelectorPayMethodContext`)
- `filteredDocuments` (como `filteredDocumentStepper`) — documentos para PaymentLink
- `selectedDocument` (como `selectedDocumentStepper`) — IDs seleccionados en PaymentLink
- `previousSelected` — flag que evita re-autoselección
- `multipleEntryReferences`, `openEntryReferenceModal`, `paymentDetail` — para la lógica del segundo paso

## Acciones Redux disparadas

| Acción | Cuándo |
|---|---|
| `setSelectedDocument` (AgreementReducer) | Selección en flujo convenios; autoselección si hay 1 documento |
| `setSelectedDocumentPaymentLink` (PaymentLinkReducer) | Selección en flujo PaymentLink; autoselección inicial |
| `setPreviousSelected(true)` | Tras autoselección inicial en `secondStep` |

## Lógica central

### `isRowSelectable(value, endDate, documentType, state?)`
Determina si una fila es seleccionable. Reglas:
1. Si `type === RECEIVABLE` y `state === PAID` → **no seleccionable**
2. Si `type === RECEIVABLE` y `value === 0` y la regla de pago es `TOTAL_PAY` → **no seleccionable**
3. Delega a `isAllowToPay(endDate, documentType)` del hook `usePaymentRules`
4. Usa formato de fecha `"DD/MM/YYYY"` (flag migrado) o `"DD/MM/YY"` (legacy)

### `finalFilteredDocuments` (useMemo)
Enriquece cada documento con:
- `Icon`: `<ErrorIcon>` si la fecha de vencimiento ya pasó, `<>` si no
- `viewMore`: componente del link "ver más" via `useGetViewMoreLink`

### `getHeightTable()`
En `secondStep`: altura dinámica = `70 * cantDocumentos + (220 + entryReferences * 30)px`
En `!secondStep`: `"auto"`

### `showCounterSelectedDocuments()`
Muestra contador de documentos seleccionados solo cuando `isSm === true` (mobile) y hay al menos uno seleccionado.

### `getModalDetail()`
- `isSm === true` → `SideModalComponent` (lateral)
- `isSm === false` → `Modal` de MUI con `DetailInfoModalMobile` centrado en pantalla

## Hooks internos

### `useDocumentTableHeaders(secondStep, isAllowToPay)`
Orquesta toda la lógica de columnas y modal. Devuelve:
- `getHeadersColumns(referenceName, isAllowToPay, secondStep, isSm)` — columnas legacy
- `headers`, `rows` — columnas y filas migradas (via `useHeadersRowsTable`)
- `showModalDetail`, `handleShowModalClose`
- `mappedDocument`
- `handleClickViewMore`
- `handleCloseMultipleEntryReferenceModal`

**Columnas legacy por modo:**

| Modo | Columnas |
|---|---|
| `!secondStep && isSm` | startDate, endDate, reference, state, value (valueToPay), actions |
| `!secondStep && !isSm` | startDate, endDate, reference, formattedValue, stateName, viewMore |
| `secondStep` | startDate, endDate, reference, state, valueTypeSelected (Dropdown), value (MoneyInputTable), actions |

### `useHeadersRowsTable(handleShowDetail, isSecondStep, isAllowToPay)`
Construye `headers` y `rows` para el modo migrado (flag activo):
- `headers`: endDate + referencias dinámicas + valores dinámicos + totalAmountPaid + amountPayments + state + startDate + (value o MoneyInputTable según paso)
- `rows`: mapea `documents_to_pay` (secondStep) o `receivable_data_list + receivable_instruction_data_list` (!secondStep) al modelo `PaymentDocument`
- ID de fila en `!secondStep`: `"${rec.id}/${rec.type}"`
- ID de fila en `secondStep`: `rec.pay_ref_id`

**⚠️ Distinción crítica en el modelo `Reference`:**

El modelo `Reference` de `ReferencesAgreementResponse` tiene dos campos que se confunden fácilmente:

| Campo | Contenido | Uso correcto |
|---|---|---|
| `Reference.name` | Nombre del tipo de referencia (ej: "Número de Factura") | Solo como `headerName` de columna |
| `Reference.value` | Valor real del documento (ej: "00123") | Como dato de celda (`REFTYPE_*`) |

- En `!isSecondStep`: se busca por `p.reference_type_code` y se usa **`.value`** para poblar `REFTYPE_1..5`
- En `isSecondStep`: se busca por `p.code` (campo distinto, porque vienen de `PaymentDetailResponse`) y también usa **`.value`**
- **Nunca usar `.name` como valor de celda** — producirá que las celdas muestren el nombre de la columna en lugar del dato real del documento

### `useMapDocumentToShowDetail()`
Gestiona qué documento se muestra en el modal lateral. Devuelve:
- `handleShowDetail(documentId)` — busca por ID en `receivable_data_list` o `receivable_instruction_data_list`
- `handleShowSecondStepDetail(params)` — extrae el documento del row del grid
- `mappedDocument` — objeto mapeado para el modal (`DetailDataProperty`)
- `renderDocumentIcon(document)` — devuelve `true` si falta una entry reference requerida
- `handleCardClick(id)` — controla apertura/cierre de tarjetas de entry references

## Componentes hijos renderizados

| Componente | Condición |
|---|---|
| `PaymentControlDisclaimer` | Siempre, encima de la tabla |
| `CustomGrid` | Siempre — tabla principal (MUI X Pro) |
| `SideModalComponent` | `isSm && showModalDetail` |
| `Modal` + `DetailInfoModalMobile` | `!isSm && showModalDetail` |
| `SideModal` + `EntryReferencesMultipleCard` | `openEntryReferenceModal === true` |
| `MoneyInputTable` | Columna `value` en secondStep |
| `MoreActionComponent` | Columna `actions` |
| `SavePaymentDocumentOption` | Dentro de `MoreActionComponent` si `payerSessionId` existe |

## Configuración del `CustomGrid`

| Prop | Valor |
|---|---|
| `singleSelection` | `!isSm` (radio buttons en desktop) |
| `checkboxSelection` | siempre |
| `maxAllowSelected` | `200` en `!secondStep`; `undefined` en `secondStep` |
| `rowHeight` | `70` en `secondStep`; `46` en `!secondStep` |
| `pinnedColumns` | `right: ["value", "actions"]` |
| `disabledFixedHeader` | `true` |
| `disableRowSelectionOnClick` | `true` |

## Traducciones (`keyPrefix: "properties.agreement.component.documentsTable"`)

| Clave | Texto |
|---|---|
| `headers.startDate` | Emisión/Inicio |
| `headers.endDate` | Vencimiento/Fin |
| `headers.state` | Estado |
| `headers.valueName` | Nombre valor |
| `headers.valueToPay` | Valor a pagar |
| `headers.pendingValue` | Valor pendiente de pago |
| `headers.totalAmountPaid` | Valor pagado |
| `headers.totalPayments` | Pagos asociados |
| `headers.showMore` | Ver más |
| `headers.saveDocument.saveTitle` | Guardar y pagar después |
| `headers.saveDocument.removeSave` | Quitar de los guardados |
| `documentSelected` | Has seleccionado |
| `paymentDocument` | documento de cobro (singular) |
| `paymentsDocument` | documentos de cobro (plural) |
| `sideModal.title` | Detalle del documento |
| `sideModal.donwloadLink` | Descargar en PDF |
| `entryRefereneceMultipleTitle` | (título del modal de entry references múltiples) |

## Tests existentes
- `__test__/useHeadersRowsTable.test.tsx` — cubre generación de headers y rows en ambos modos (`secondStep: true/false`)
- No hay tests directos para `DocumentsTable.tsx`, `useDocumentTableHeaders.tsx` ni `useMapDocumentToShowDetail.ts`

## Clases CSS relevantes

| Clase | Descripción |
|---|---|
| `.payment-control-disclaimer` | Contenedor del disclaimer, flex centrado |
| `.documents-table--icon` | Icono de error (vencimiento) — color `semantic error 500` |
| `.documents-table__modal` | Modal de detalle (mobile) centrado en pantalla |
| `.documents-table__modal__container` | Caja blanca con sombra y border-radius 16px |
| `.counter-selected-document` | Texto contador de documentos seleccionados |
| `.references-counter-container` | Contenedor flex row para el contador en secondStep |
| `.actions-container-second-step-paid` | Acción "ver más" en fila de secondStep con margin-left 33px |

## Reglas para modificar este componente

- Es usado en **dos lugares** (`DocumentsStep` y `DocumentTableStep`) — evaluar impacto en ambos flujos antes de modificar
- El flag `MIGRATE_PAYMENT_DOCUMENT_TABLE` divide dos caminos de renderizado de tabla; cualquier cambio en columnas debe contemplar ambos
- `isRowSelectable` es crítico para el flujo de pago; modificarlo puede afectar qué documentos el usuario puede seleccionar
- `useHeadersRowsTable` y `useDocumentTableHeaders` están separados intencionalmente (migración progresiva); no fusionarlos sin coordinar
- La autoselección en `secondStep` ocurre una sola vez gracias a `previousSelected`; cuidado al modificar ese `useEffect`
- `MAX_ALLOW_SELECTED = 200` aplica solo en `!secondStep`; cambiar este valor afecta el límite de documentos seleccionables en convenios
- En `useHeadersRowsTable`, al mapear referencias usar siempre `Reference.value` (dato del documento), nunca `Reference.name` (nombre del tipo = header de columna)
- En `useHeadersRowsTable` path `!isSecondStep`, el ID de row **debe** construirse con `rec.uuid ?? rec.id` (no solo `rec.id`). Para convenios externos los documentos tienen `uuid` como identificador canónico; usar solo `rec.id` rompe la selección y el envío de shadow documents

## ⚠️ ID de row en convenios externos (path migrado)

`getFilteredDocuments` (que genera `filteredDocuments`) usa `p.uuid ?? p.id` como ID base. `useHeadersRowsTable` debe usar el mismo criterio para que no haya mismatch entre:
- El `rowSelectionModel` (que viene de `filteredDocuments`)
- Los IDs reales de las rows del grid

**Si los IDs difieren:**
1. Auto-select almacena UUID pero el grid tiene ID numérico → row aparece visualmente no seleccionada
2. `useEffect` de validación borra la selección porque no encuentra el ID numérico en `filteredDocuments` (UUID-based)
3. `buildShadowDocumentsToLink` en `useStepperReferences` no encuentra los UUIDs → envía arrays vacíos al API

## Historial de cambios

| Fecha | Bug | Resumen |
|---|---|---|
| 2026-02-23 | BUG-001 | Fix en `useHeadersRowsTable`: `REFTYPE_1..5` usaban `Reference.name` en lugar de `Reference.value` en el path `!isSecondStep`, mostrando el header de columna como valor de celda |
| 2026-02-23 | BUG-002 | Fix en `useHeadersRowsTable`: construcción del ID de row cambiada de `rec.id` a `rec.uuid ?? rec.id` en path `!isSecondStep`, para alinear con `getFilteredDocuments` y soportar convenios externos con UUID |
