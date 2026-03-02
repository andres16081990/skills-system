# PaymentDocumentMain — Agent Skill

## Propósito

`PaymentDocumentMain` es la **página principal de gestión de documentos de cobro** de la aplicación. Muestra una lista paginada de documentos (recaudables e instrucciones), permite filtrarlos, exportarlos, eliminarlos individualmente o de forma masiva, marcarlos como pagados (individualmente o en masa) y crear nuevos documentos.

Vive en:
```
src/presentation/paymentDocument/pages/Main/PaymentDocumentMain.tsx
```

Pertenece a la ruta `RoutesPath.PAYMENT_DOCUMENT`. No recibe props — obtiene todo de Redux y React Router.

---

## Comunicación con el resto de la app

### Cómo recibe datos
| Fuente | Qué recibe |
|---|---|
| `PaymentDocumentState` (Redux) | Lista de documentos, paginación, IDs seleccionados, estado de procesos, reporte de exportación, `isEmptyResults`, `changedFilters`, `isRedirectFromDashboard`, `excludeIds`, `documentsPaid` |
| `ReceivableState` (Redux) | `processState` (para escuchar resultados de delete/create/update), `modalContent`, `emailNotificationInfo`, `previousFiltersAndPagination`, `paymentCodeToQrGenerator` |
| `AgreementState` (Redux) | `processState`, `filteredAgreements` (usado para construir el filtro de búsqueda) |
| `EntitlementState` (Redux) | `permissionsByUser` (para mostrar/ocultar el botón "Crear documento" y el conteo por rol) |
| `useLocation` (React Router) | `state` — puede venir con `source: DASHBOARD` o `source: UPLOAD_CENTER` para pre-popular filtros |
| FeatBit flags | `FRONT_MARK_AS_PAID_DOCUMENT` (habilita flujo de marcar como pagado), `FRONT_ENTITLEMENT` (muestra conteo `non_deletable_by_role` en mensajes de eliminación) |

### Cómo emite datos
- Despacha acciones Redux: `getPaymentDocumentList`, `exportPaymentDocumentList`, `setPagination`, `setSelectedIds`, `resetSelectedIds`, `setExcludeIds`, `setChangedFilters`, `setResetedIds`, `setIdleState`, `resetPaymentDocumentList`, `resetEmptyResults`, `resetPaymentDocumentReport`, `resetExternalPayments`, `resetOpenSideModalMarkAsPaid`, `resetTotalDocuments`, `setPaymenDocumentIsRedirectFromDashboard`, `setPreviousFilterAndPagination` (en ReceivableReducer), `getAgreementChannelsThunk`, `resetDigitalAgreementChannel`, `getExternalPaymentMethodsThunk`, `setIdleStateConciliation`, `setIdleStateReceivable`.
- Pasa callbacks hacia abajo a componentes hijos (`handleFilterChange`, `handleCleanFilters`, `handleOnSearch`, etc.).

### Con qué componentes se comunica directamente
| Componente | Módulo |
|---|---|
| `DocumentFilters` | `receivable/components/ReceivablesTabs/ReceivableTab/Filters/` |
| `DocumentTable` | `receivable/components/DocumentTable/` |
| `MassiveSelectionFilter` | `receivable/components/MassiveSelectionFilter/` |
| `MarkAsPaidSideModal` | `receivable/components/MarkAsPaidSideModal/` |
| `ConfirmModalComponent` | `receivable/components/MarkDocumentAsPaid/ConfirmModalComponent/` |
| `MarkAsPaidModalChild` | `receivable/components/MassiveSelectionFilter/MarkAsPaidModalChild/` |
| `EmailModalContent` | `receivable/components/ReceivablesTabs/EmailModalContent/` |
| `QrCodeGenerator` | `receivable/components/ReceivablesTabs/QrCodeGenerator/` |
| `DetailInfoModal` | `conciliation/components/DetailInfoModal` |
| `SideModalCreateDocument` | `home/pages/components/SideModalCreateDocument/` |
| `NotificationsModal` | `paymentStrategy/components/NotificationsModal/` |
| `ExportButton` | `paymentDocument/components/ExportButton/` |
| `RemoveBulkFilterModalContent` | `paymentDocument/components/RemoveBulkFilterModal/` |
| `ShowElementByPermission` | `common/components/ShowElementByPermission/` |

### A qué páginas o layouts pertenece
Se renderiza en `RoutesPath.PAYMENT_DOCUMENT`. [INFERIDO] Está envuelto en `ProtectedPageWithLayoutHOC` o equivalente dado el patrón de la app — validar en el router.

---

## Props

Este componente no recibe props. Es una página de ruta top-level.

---

## Hooks y dependencias

### Hooks propios
| Hook local | Propósito |
|---|---|
| `useState(showMessage / message)` | Controla el Toast de éxito/error |
| `useState(isExactFilter)` | Si la búsqueda de texto es exacta o parcial |
| `useState(qrLinkBase64 / qrLinkBase64Code)` | Datos del QR generado |
| `useState(openModal)` | Abre el `SideModalCreateDocument` |
| `useState(isMassiveSelectionDelete)` | Activa el modo de selección masiva para eliminación |
| `useState(isRemoveBulkModalOpen)` | Controla el modal de confirmación para quitar el filtro bulk |
| `useRef(csvLink)` | Referencia al componente `CsvExport` para disparar la descarga |
| `useRef(anchorTagRef)` | Referencia al tag anchor para descarga de archivos |
| `useRef(massiveDeletionMessage)` | Almacena el mensaje de resultado de eliminación masiva |
| `useRef(hasProcessedInitialState)` | Guard de un solo uso para no re-fetchear en mount cuando viene de UPLOAD_CENTER |
| `useRef(closeAlertRef)` | Callback para cerrar el alert de filtro bulk desde fuera del componente |
| `useState(documentTypeByFulfillmentRequest)` | Tipo de documento actual según el último resultado del API |

### Hooks externos consumidos

| Hook | Datos que toma | Acciones que dispara |
|---|---|---|
| `useAppDispatch` | — | Acceso al dispatcher tipado |
| `useTypedSelector(stateSelectorPaymentDocument)` | `processState`, `pagination`, `paymentDocumentList`, `paymentDocumentListReport`, `isEmptyResults`, `selectedIds`, `documentsPaid`, `changedFilters`, `isRedirectFromDashboard`, `excludeIds` | — |
| `useTypedSelector(stateSelectorReceivable)` | `processState`, `modalContent`, `emailNotificationInfo`, `previousFiltersAndPagination`, `paymentCodeToQrGenerator` | — |
| `useTypedSelector(stateSelectorAgreement)` | `processState` | — |
| `useTypedSelector(stateSelectorEntitlement)` | `permissionsByUser` | — |
| `useLocation` | `state` (fuente de navegación), `pathname` (para validar ruta activa) | — |
| `useFlags` (FeatBit) | `FRONT_MARK_AS_PAID_DOCUMENT`, `FRONT_ENTITLEMENT` | — |
| `useTranslation` (x4) | Keys de `properties.paymentDocument`, `properties.agreement.components.receivablesTabs.tabs`, `properties.common.components.counterSelectedRows`, `...markAsPaidComponent` | — |
| `usePaymentDocumentReport` | `documentFilters`, `filteredAgreements`, `setDocumentFilters`, `buildFilterToSearch`, `canCleanFilter`, `setCanCleanFilter`, `changeFiltersRef` | Carga acuerdos (`getAgreements`), despacha `getPaymentDocumentList` en cambio de state de navegación, persiste filtros en `ReceivableState` |
| `useNotificationsModal` | `notificaionsTranslate`, `showNotificationDisabledMessage`, `typeNotificationChannel` | `handleCloseNotificationModal` → resetea `UserContextState` |
| `usePrevoiusFilters` | — | Restaura filtros y paginación previos desde `ReceivableState.previousFiltersAndPagination` si `isFromEditedDocument` es true |
| `useDashboardRedirect` | — | `handleBackToDashboard` → navega al dashboard |
| `useDataDocumentTable` | `openSideModal`, `documentDetailData`, `documentDetail` | `getRowsDocumentsTable`, `getHeaderColumns`, `getDataRows`, `setOpenSideModal` |
| `useDeleteDocuments(selectedIds)` | `showModal`, `titleModal`, `descriptionModal`, `isDeletable` | `handleDelete`, `handleCloseModal`, `setShowModal`, `getDataModalMassive` |
| `useHandleOptionsMassiveSelect` | `openMarkAsPaidModal` | `getDropDownDataMassiveSelections`, `handleOnChangeMassiveOpt`, `handleCloseMarkAsPaidModal`, `handleOnConfirmMarkAsPaidModal` |
| `useSideModalMarkAsPaidOneToOne` | `openSideModalMarkAsPaid`, `currentIdOpened`, `disableConfirmButton` | `handleOnConfirmModal`, `handleOncloseMarkAsPaidModal` |

---

## Flujo de datos

### 1. Carga inicial (`useEffect` con `[]`)
1. Se despacha `setIdleStateConciliation()` y `setIdleStateReceivable()` para limpiar estados de otras páginas.
2. `resetOpenSideModalMarkAsPaid()` y `resetSelectedIds()`.
3. Si `FRONT_MARK_AS_PAID_DOCUMENT` está activo → `getExternalPaymentMethodsThunk()`.
4. Si **no** viene de `UPLOAD_CENTER` → `buildFilterToSearch(pagination, isExactFilter)` → `getPaymentDocumentList(filterRequest)`.
5. **Cleanup (unmount):** resetea paginación, estado de modal, resultados vacíos, `changedFilters`, `resetedIds`, pagos externos, modal de notificaciones, `isRedirectFromDashboard`, `excludeIds`, lista de documentos.

### 2. Navegación con estado previo (`usePaymentDocumentReport`)
- `state.source === DASHBOARD` → extrae fechas y estado del `cardSelected`, pre-popula `documentFilters`, dispara búsqueda.
- `state.source === UPLOAD_CENTER` → pre-popula `agreementSelected`, `type`, `code_process`, dispara búsqueda y abre modal si corresponde.
- Si `isFromEditedDocument` (vuelta desde edición de documento) → `usePrevoiusFilters` restaura los filtros y la paginación previos.

### 3. Cambio de filtros
```
Usuario interactúa con DocumentFilters
  → handleFilterChange(name, value)
    → si es campo de fecha (TypeDateAgreement): parsea el JSON y separa en _initial / _final
    → si es campo regular: actualiza key directamente
  → setDocumentFilters({ ...documentFilters, [name]: value, isChangedFilters: true })
  → setCanCleanFilter(false), changeFiltersRef.current = false
```
La búsqueda **no** se lanza automáticamente al cambiar filtros — el usuario debe presionar el botón de búsqueda en `DocumentFilters` (`getFilteredDocuments`) o en el `SearchInput` (`handleOnSearch`).

### 4. Búsqueda de documentos
```
handleOnSearch(exact: boolean)  /  getFilteredDocuments()  /  searchPaymentDocumentList(pagination)
  → buildFilterToSearch(pagination, exact, ...)
      → buildDocumentRequest(filters, filteredAgreements.flatMap(p => p.agreement_code), ...)
  → dispatch(getPaymentDocumentList(filterRequest))
  → Reducer: loading → success → paymentDocumentList, pagination, isEmptyResults actualizados
  → useEffect([processStatePaymentDocument]) detecta success
      → setDocumentTypeByFulfillmentRequest(documentFilters.type)
      → dispatch(setIdleStatePaymentDocument())
```

### 5. Paginación
```
handleOnChangePagination(newPage) / handleOnChangePageSize(newPageSize)
  → dispatch(setPagination(newPagination))
  → useEffect([pagination.page, pagination.page_size])  ← se dispara automáticamente
      → searchPaymentDocumentList(pagination, isExactFilter)
      ⚠ Excepción: si viene de UPLOAD_CENTER y hasProcessedInitialState.current === false, salta
```

### 6. Selección masiva
```
Usuario hace click en "seleccionar todos"
  → handleMassiveSelectionDelete() → isMassiveSelectionDelete = true
  → useEffect([documentTypeByFulfillmentRequest, paymentDocumentList, isMassiveSelectionDelete])
      → Si type !== "ALL": selecciona IDs de docs no-PAID de la página actual, excluyendo excludeIds
      → Si type === "ALL": selecciona todos los IDs no-PAID de la página
  → useEffect([selectedIds, paymentDocumentList, isMassiveSelectionDelete, ...])
      → Mantiene excludeIds (documentos visibles que NO están en selectedIds)
  → massiveSelectionCount: calcula cuántos docs hay seleccionados en total (totalRows - excludeIds.length)
```

### 7. Exportación
```
handleGetFile(formatKey, type)
  → si type === DOCUMENTS:
      isCsv = (formatKey === "xlsx")
      dispatch(exportPaymentDocumentList(buildFilterToSearch({ ...pagination, page: 1 }, undefined, isCsv)))
  → Reducer: success → paymentDocumentListReport = base64/URL del archivo
  → useEffect([paymentDocumentListReport])
      → csvLink.current.link.click()  ← dispara la descarga del archivo
      → dispatch(resetPaymentDocumentReport())
```

### 8. Respuesta a acciones sobre documentos
```
useEffect([processStateReceivable])
  → Condición: current === "success" && pathname === path && action === uno_de_los_ReceivableActions
  → Si es eliminación masiva: mensaje = massiveDeletionMessage.current
  → Si es otra acción: mensaje = processStateReceivable.message
  → setShowModal(false), dispatch(setSelectedIds([])), setShowMessage(true), setMessage(...)
  → Si fue eliminación → relanza getPaymentDocumentList para refrescar la lista
  → dispatch(setIdleStateReceivable())
```

### 9. Canales de acuerdo
```
useEffect([paymentDocumentList])
  → Si lista tiene elementos → dispatch(getAgreementChannelsThunk({ agreement_ids: [...] }))
  → Cleanup → dispatch(resetDigitalAgreementChannel())
```

---

## Dependencias críticas

| Dependencia | Rol |
|---|---|
| `PaymentDocumentProvider` | Service locator: provee `getPaymentDocumentListUseCase`, `getPaymentDocumentDetailUseCase`, `exportPaymentDocumentListUseCase`, `setExternalPaymentsUseCase`, `getExternalDocumentsUseCase` |
| `PaymentProvider` | Service locator: provee `getInternalExternalPaymentsByDocumentIdUseCase` |
| `buildDocumentRequest` (`utils/paymentDocuments.ts`) | Construye el `PaymentDocumentRequest` a partir de los filtros locales y los códigos de acuerdo |
| `hasAnyAgreementWithCreatePermission` | Determina si el botón "Crear documento" es visible |
| `getStateFilterValue` | Convierte el estado de filtro ("ALL", "SOON_DUE", "DUE", etc.) a array de IDs numéricos para la API |
| `DocumentStateEnum.PAID` | Se usa para excluir documentos ya pagados de la selección masiva — si cambia su valor en la API, la selección masiva se rompe |
| `RoutesPath.PAYMENT_DOCUMENT` | Usado como guard en el effect de `processStateReceivable` para evitar responder cuando el usuario ya navegó a otra ruta |
| `FRONT_MARK_AS_PAID_DOCUMENT` (feature flag) | Habilita `getExternalPaymentMethodsThunk` en mount y toda la UI de "marcar como pagado" |
| `FRONT_ENTITLEMENT` (feature flag) | Muestra/oculta el conteo `non_deletable_by_role` en el mensaje de eliminación masiva |
| `ComponentSource.UPLOAD_CENTER / DASHBOARD` | Constantes que determinan el comportamiento al recibir estado de navegación |

---

## Puntos frágiles

### 1. Transformación del ID del documento en el reducer
En `PaymentDocumentReducer.ts` el id se reescribe como:
```ts
id: `${document.id}-${document.type}`
```
Y luego en el componente se parsea con:
```ts
paymentDocument.id.split("-")[0]
```
**Si el `type` contiene guiones** (`-`), el split extraerá solo la primera parte del tipo y no el id original. También afecta a cualquier componente que reciba un id de este store y lo envíe a la API.

### 2. Guard `hasProcessedInitialState` de un solo uso
```ts
const hasProcessedInitialState = useRef(false);
```
Se setea a `true` en el primer effect de paginación cuando viene de UPLOAD_CENTER. Solo funciona en el primer render — si el usuario navega a la misma ruta dos veces sin desmontar el componente, el guard ya está consumido y podría ejecutarse el fetch inesperado.

### 3. Cálculo de `massiveSelectionCount`
El cálculo depende de `pagination.total_rows`, `excludeIds.length`, `documentTypeByFulfillmentRequest` y `paymentDocumentList`. Si `total_rows` está desactualizado (ej: entre la respuesta del API y la interacción del usuario), el contador mostrará un valor incorrecto.

### 4. `closeAlertRef` desconectado
`handleAlertClose` almacena el callback `closeAlertFn` recibido desde `DocumentFilters` a través del prop `onAlertClose`. Si `DocumentFilters` no llama a `onAlertClose` (o cambia su API), `closeAlertRef.current` permanecerá `null` y `handleConfirmRemoveBulkFilter` no cerrará el alert interno del componente.

### 5. Routing en el effect de `processStateReceivable`
El guard `pathname === path` previene respuestas en otras rutas, pero la lista de `ReceivableAction` que se chequea es extensa. **Agregar una nueva acción al módulo receivable sin actualizar este effect** hará que el componente ignore silenciosamente el éxito de esa acción.

### 6. `sanitizedTypeDocument` puede producir un string con coma
```ts
const sanitizedTypeDocument =
  typeDocumentToSelect.length > 1
    ? typeDocumentToSelect[0]   // solo toma el primero
    : typeDocumentToSelect.toString();  // si hay 1 elemento, es string limpio
```
La rama `length > 1` toma `[0]` que es correcto, pero la documentación del flujo y los tests deberían verificar que los componentes downstream manejan bien cuando `typeDocument` tiene más de un valor.

---

## Reglas — Lo que NUNCA debes hacer

1. **No llames `getPaymentDocumentList` sin pasar por `buildFilterToSearch`**. Esa función garantiza que los `agreementCodes` vengan de `filteredAgreements` y que las fechas estén formateadas correctamente.

2. **No modifiques la transformación `id: \`${document.id}-${document.type}\`` en el reducer** sin actualizar todos los `split("-")[0]` en el componente y componentes hijos que usen ese id para llamadas a la API.

3. **No elimines ni cortocircuites el cleanup del `useEffect` con `[]`**. El reseteo de estado en unmount es crítico — si se omite, la próxima visita a la página arrancará con datos sucios (lista, seleccionados, paginación, filtros).

4. **No agregues lógica de efecto que dependa de `processStateReceivable` sin incluir el guard `pathname === path`**. Sin él, el componente responderá a acciones de otras páginas que usan el mismo slice.

5. **No llames `dispatch(setSelectedIds([]))` directamente sin también llamar `dispatch(setResetedIds(true))`**. `idsReseted` es la señal que usa `DocumentTable` para limpiar su estado interno de selección.

6. **No toques `excludeIds` fuera de los handlers designados** (`setExcludeIds`, `resetExcludeIds`). Su cálculo está sincronizado con `selectedIds` y `paymentDocumentList` en dos effects separados — un cambio externo puede desincronizar el conteo.

---

## Cómo diagnosticar y corregir bugs

### La lista no carga al entrar
1. Verificar que `filteredAgreements` en `AgreementState` no esté vacío — si el fetch de acuerdos falla, `buildFilterToSearch` enviará `agreementSelected: []` y el API puede retornar resultados vacíos o error.
2. Revisar si `previousFiltersAndPagination.isFromEditedDocument` está en `true` de forma inesperada — haría que el effect de paginación (`[pagination.page, pagination.page_size]`) no ejecute la búsqueda.
3. Comprobar si `state.source === UPLOAD_CENTER` en `useLocation` — en ese caso el mount no lanza el fetch y espera que `usePaymentDocumentReport` lo haga.

### El Toast no aparece después de una acción
1. Comparar `processStateReceivable.action` con la lista de `ReceivableAction` del effect — si la acción no está en la lista, el effect no se activa.
2. Verificar que `pathname === path` — si el usuario navegó antes de que resolviera la llamada, el guard lo bloquea.
3. Si es eliminación masiva, verificar que `massiveDeletionMessage.current` tenga contenido (se setea en `getModalMassiveDeletionOptions`).

### La exportación no descarga el archivo
1. Verificar que `paymentDocumentListReport` en el store cambia a un valor truthy después de `exportPaymentDocumentList.fulfilled`.
2. Verificar que `csvLink.current` no sea null — si `ExportButton` no montó correctamente, la ref estará vacía.
3. Verificar que `isChangedFilters` sea `true` — `ExportButton` deshabilita el botón si `isChangedFilters` es false.

### El contador de selección masiva es incorrecto
1. Loguear: `isMassiveSelectionDelete`, `documentTypeByFulfillmentRequest`, `pagination.total_rows`, `excludeIds.length`, `paymentDocumentList.length`.
2. Verificar que `total_rows` en el store refleja el resultado de la última búsqueda y no un valor previo.
3. Verificar que `excludeIds` no acumula IDs duplicados entre cambios de página — revisar los effects de sincronización.

### Los filtros previos no se restauran al volver de la edición
1. Verificar que `previousFiltersAndPagination.isFromEditedDocument` se setea a `true` antes de navegar a la página de edición.
2. Revisar `usePrevoiusFilters`: carga los filtros y luego llama `setPreviousFilterAndPagination(initialPreviousFilterAndPagination)` para resetear el flag — si esto falla, los filtros se restaurarán infinitamente.

---

## Tests

**Ubicación:**
```
src/presentation/paymentDocument/pages/Main/__test__/PaymentDocumentMain.test.tsx
```

**Cómo correrlos:**
```bash
pnpm test -- src/presentation/paymentDocument/pages/Main/__test__/PaymentDocumentMain.test.tsx
```

**Estructura de los tests:**

| Describe | Enfoque | Tipo |
|---|---|---|
| `Filter Logic Tests > handleFilterChange logic` | Verifica que los campos de fecha separan _initial/_final y que los campos normales se actualizan directamente | Lógica pura (sin render) |
| `Filter Logic Tests > handleCleanFilters logic` | Verifica el reset a `documentFilterInitialize`, los dispatches de `resetSelectedIds`/`setResetedIds`, y la llamada condicional a `searchPaymentDocumentList` si `code_process` está activo | Lógica pura |
| `Filter Logic Tests > getModalMassiveDeletionOptions logic` | Verifica el mensaje de eliminación masiva con distintas combinaciones de `deletable_count`, `non_deletable_count`, `non_deletable_by_role` y el flag `FRONT_ENTITLEMENT` | Lógica pura |
| `Filter Logic Tests > TypeDateAgreement enum check` | Verifica que `due_date` y `emition_date` se reconocen como campos de fecha y que `state`/`type` no | Lógica pura |
| `Filter Logic Tests > buildFilterToSearch / handleGetFile` | Verifica la construcción del filtro con `page: 1` para exportación y el flag `isCsv` | Lógica pura |
| `handleCleanFilters branch coverage` | Render real del componente con todos los hooks mockeados; prueba el flujo completo de eliminar filtro bulk (click en `DocumentFilters` → modal → confirmar → `dispatch`) | Render con RTL |

> **Nota importante:** La mayoría de los tests simulan la lógica del componente directamente (copiando el código) en lugar de testear el componente renderizado. Esto significa que un refactor interno del componente puede no romper los tests aunque la lógica cambié.
