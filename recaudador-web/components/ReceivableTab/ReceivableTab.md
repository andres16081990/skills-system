# ReceivableTab — Agent Skill

## Propósito

Panel de tabla para gestionar documentos de cobro de un acuerdo: **cuentas por cobrar** (receivables) o **instrucciones** (instructions). Es uno de los dos tabs del componente `ReceivablesTabs`.

Resuelve la visualización filtrada, la selección masiva, la eliminación, la exportación y las acciones individuales (notificación, QR, detalle) de documentos. Vive en:

```
src/presentation/receivable/components/ReceivablesTabs/ReceivableTab/ReceivableTab.tsx
```

---

## Comunicación con el resto de la app

### Recibe datos
- **Props** del padre `ReceivablesTabs`: filtros, paginación, handlers de búsqueda y cambio de filtros.
- **Redux** — cuatro slices:
  - `ReceivableState` → `receivableList`, `instructionList`, `processState`, `documentCsv`, `isEmptyReceivablesResults`, `isEmptyInstructionsResults`, `modalContent`, `paymentCodeToQrGenerator`
  - `PaymentDocumentState` → `selectedIds` (alias `receivableIds`), `excludeIds`, `paymentDocumentListReport`
  - `AgreementState` → `agreementDetail`, `processState` (para loading)
  - `MassiveLoadState` → `pendingLoad`
- **FeatBit** → flag `FRONT_ENTITLEMENT` que controla si se muestra el contador de `non_deletable_by_role`.

### Emite datos / mutations al store
| Acción despachada | Cuándo |
|---|---|
| `setSelectReceivableIds(ids)` | Al cambiar `selectedIds` local o al resetear |
| `setExcludeIds(ids)` | En modo selección masiva, al cambiar página o deseleccionar |
| `resetPaymentDocumentReport()` | Después de disparar descarga de CSV/XLSX |
| `resetDocumentCsv()` | Después de disparar descarga de CSV de receivables |
| `exportPaymentDocumentList(request)` | Al exportar documentos |
| `setShowComponentLoader(true)` | Al iniciar carga masiva |
| `setMassiveLoadType(type)` | Al iniciar carga masiva |
| `setMassiveDeletionMessage(msg)` | Al recibir resultado de eliminación masiva |
| `setSelectReceivableIds([]) + setExcludeIds([])` | Al desmontar (cleanup) |

### Componentes hijos directos
- `DocumentFilters` — filtros de fecha, estado, tipo, acuerdo
- `SearchInput` (DS) — búsqueda por referencia
- `PrimaryButton` + `Menu`/`MenuItem` (MUI) — botón "Agregar" con opciones masiva/manual
- `CsvExport` + `DownloadFile` — exportación de archivos
- `CounterRows` (DS) — contadores de selección
- `LinkText` (DS) — link "seleccionar todos" para eliminación masiva
- `MassiveSelectionFilter` — acciones sobre selección masiva
- `DocumentTable` — tabla principal (MUI DataGrid)
- `GenericModal` (DS) — modal de confirmación de borrado individual
- `SideModal` (DS) — contenedor de modales laterales
- `DetailInfoModal` — detalle del documento
- `EmailModalContent` — envío de notificación por email
- `QrCodeGenerator` — visualización del código QR
- `NotificationsModal` — mensaje de canal de notificación deshabilitado
- `ShowElementByPermission` — wrapper de permisos

### Páginas / layouts
- Vive dentro de `ReceivablesTabs`, que a su vez vive en la página de detalle de acuerdo (`AgreementDetail`).
- El tab 0 renderiza RECEIVABLE, el tab 1 renderiza INSTRUCTIONS.

---

## Props

| Nombre | Tipo | Requerida | Qué hace | De dónde viene |
|---|---|---|---|---|
| `tabValue` | `number` | ✅ | Valor activo del tab (0 = RECEIVABLE, 1 = INSTRUCTIONS). Controla qué lista mostrar y con qué parámetros operar | `ReceivablesTabs` |
| `type` | `ReceivablesTypesEnum` | ✅ | Determina si el tab es de cuentas por cobrar o instrucciones. Usado en condicionales en toda la lógica | `ReceivablesTabs` |
| `handleOnSearch` | `(exactFilter?: boolean) => void` | ✅ | Dispara la búsqueda con el filtro actual. Se llama desde `SearchInput` y desde `DocumentFilters` | `ReceivablesTabs` |
| `disabledAddDocument` | `boolean` | ✅ | Deshabilita el botón de agregar documentos | `ReceivablesTabs` |
| `handleOnChangePagination` | `(page: number) => void` | ✅ | Cambia la página activa | `ReceivablesTabs` |
| `pagination` | `PaginationMetadata` | ✅ | Metadata de paginación: `page`, `page_size`, `total_rows` | `ReceivablesTabs` |
| `canCleanFilter` | `boolean` | ✅ | Controla si el botón limpiar filtros está habilitado | `ReceivablesTabs` |
| `documentFilters` | `DocumentFilter` | ✅ | Estado actual de los filtros activos | `ReceivablesTabs` |
| `handleFilterChange` | `(name: string, value?: string \| Date) => void` | ✅ | Actualiza un campo individual del filtro | `ReceivablesTabs` |
| `handleCleanFilters` | `() => void` | ✅ | Resetea todos los filtros a sus valores iniciales | `ReceivablesTabs` |
| `setDocumentFilters` | `(newData: DocumentFilter) => void` | ✅ | Reemplaza el objeto completo de filtros (usado al escribir en `SearchInput`) | `ReceivablesTabs` |
| `handleOnChangePageSize` | `(tabValue: number, newPageSize: number) => void` | ✅ | Cambia el tamaño de página | `ReceivablesTabs` |

---

## Hooks y dependencias

### Hooks propios (internos)
No hay hooks definidos dentro del propio archivo. Toda la lógica vive en el cuerpo del componente con `useState`/`useEffect`/`useMemo`/`useCallback`.

### Hooks externos consumidos

| Hook | Datos que toma | Acciones que dispara |
|---|---|---|
| `useFlags()` | Flag `FRONT_ENTITLEMENT` | — |
| `useNavigate()` | — | Navega a `RECEIVABLE_CREATE` o `INSTRUCTIONS_CREATE` |
| `useAppDispatch()` | — | Despacha todas las acciones Redux listadas arriba |
| `useTypedSelector(stateSelectorReceivable)` | `receivableList`, `instructionList`, `documentCsv`, `isEmptyReceivablesResults`, `isEmptyInstructionsResults`, `processState`, `modalContent`, `paymentCodeToQrGenerator` | — |
| `useTypedSelector(stateSelectorPaymentDocument)` | `selectedIds` (alias `receivableIds`), `excludeIds`, `paymentDocumentListReport` | — |
| `useTypedSelector(stateSelectorAgreement)` | `agreementDetail`, `processState` | — |
| `useTypedSelector(stateSelectorMassiveLoad)` | `pendingLoad` | — |
| `useDeleteDocuments(selectedIds, type)` | `showModal`, `titleModal`, `descriptionModal`, `isDeletable` | `handleDelete`, `setShowModal`, `handleCloseModal`, `getDataModalMassive` |
| `useDataDocumentTable()` | `getRowsReceivablesTable`, `getHeadersReceivablesTable`, `getRowsInstructionsTable`, `getHeadersInstructionsTable`, `getHeaderColumns`, `getDataRows`, `openSideModal`, `documentDetailData` | `setOpenSideModal` |
| `useNotificationsModal()` | `notificaionsTranslate`, `showNotificationDisabledMessage`, `typeNotificationChannel` | `handleCloseNotificationModal` |
| `useShowElementByPermission()` | — | `checkAgreementHasPermission(keys, agreementCode)` |
| `useTranslation("translation", { keyPrefix: "...tabs" })` | Traducciones UI del tab | — |
| `useTranslation("translation", { keyPrefix: "...counterSelectedRows" })` | Traducciones de contadores | — |
| `useTranslation("translation", { keyPrefix: "...paymentDocument" })` | Traducciones generales de documentos | — |

---

## Flujo de datos

### Renderizado inicial
1. El padre `ReceivablesTabs` pasa `tabValue`, `type`, `documentFilters`, `pagination` y todos los handlers.
2. El componente lee los cuatro slices Redux para obtener el estado de listas, proceso y selección.
3. `useDataDocumentTable` transforma las listas en columnas/filas del DataGrid.
4. `DocumentTable` renderiza la tabla.

### Búsqueda / filtrado
1. El usuario modifica filtros en `DocumentFilters` → llama a `handleFilterChange` (prop) → actualiza `documentFilters` en el padre.
2. Al hacer clic en buscar → `handleOnSearch(exactFilter)` → el padre despacha `getReceivableList` o `getInstructionList`.
3. Al completarse (`processState.current === "success"` + acción GET_*), se hace `setIsLoadingLocal(false)`.

### Selección individual
1. El usuario marca checkboxes en `DocumentTable` → `setSelectedIds(ids)`.
2. `useEffect([selectedIds])` → despacha `setSelectReceivableIds(selectedIds)`.
3. Aparece `CounterRows` y `LinkText` con acciones.

### Selección masiva (eliminación global)
1. El usuario hace clic en "seleccionar todos" (`LinkText`) → `handleMassiveSelectionDelete()` → `setIsMassiveSelectionDelete(true)`.
2. `useEffect([..., isMassiveSelectionDelete])` calcula todos los IDs seleccionables de la lista actual y los despacha como `setSelectReceivableIds`.
3. Otro `useEffect([receivableIds, ..., isMassiveSelectionDelete])` recalcula `excludeIds` por página.
4. `massiveSelectionCount` (memo) calcula el total considerando `pagination.total_rows - excludeIds.length`.
5. `MassiveSelectionFilter` muestra la acción de eliminar.
6. Al confirmar, `useMassiveSelectionFilter` (interno a `MassiveSelectionFilter`) despacha `massiveDeletionDocuments`.
7. Si `processState === success + DELETE/DELETION_CONFIRM` → resetea selección y refresca la página.

### Exportación
1. El usuario hace clic en `CsvExport` → `handleGetFile(formatKey, typeFile)`.
2. Guard: solo ejecuta si `typeFile === FileTypeEnum.DOCUMENTS`.
3. `formatKey === "xlsx"` → `isCsv = true`; `"csv"` → `isCsv = false`. [INFERIDO: el nombre `isCsv` parece invertido — `xlsx` produce `isCsv=true`. Validar con el backend.]
4. `buildFilterToSearch({ ...pagination, page: 1 }, undefined, isCsv)` construye el request.
5. Despacha `exportPaymentDocumentList({ ...request, type: MassivePaymentDocumentEnum.RECEIVABLE | INSTRUCTION })`.
6. `useEffect([paymentDocumentListReport])` → llama a `csvLink.current.link.click()` para descargar.
7. Despacha `resetPaymentDocumentReport()`.

### Descarga de CSV de receivables
- Similar al flujo anterior pero usando `documentCsv` del slice `ReceivableState` y `resetDocumentCsv()`.

### Modales laterales
- Controlados por `modalContent` del slice Receivable: `DOCUMENT`, `NOTIFICATION`, `QR_CODE`.
- `openSideModal` local (de `useDataDocumentTable`) controla si el `SideModal` está abierto.

---

## Dependencias críticas

| Dependencia | Propósito |
|---|---|
| `buildDocumentRequest` (`paymentDocuments.ts`) | Construye el `PaymentDocumentRequest` para la exportación |
| `useDataDocumentTable` | Transforma listas de receivables/instrucciones en filas y columnas del DataGrid |
| `useDeleteDocuments` | Lógica del modal de confirmación de borrado individual |
| `useMassiveSelectionFilter` | Orquesta el flujo de eliminación masiva (incluido `adaptPaymentToMassiveDeletion`) |
| `DocumentStateEnum` (`StateFilter.tsx`) | Usado para filtrar qué filas son seleccionables (`PAID`, `PARTIALLY_PAID`) |
| `getReceivablesHeaders` / `getInstructionsHeaders` | Generan las cabeceras legacy de la tabla |
| `getHeaderColumns` / `getDataRows` | Generan columnas/filas para el DataGrid moderno (MUI X) |
| `ReceivablesTypesEnum.RECEIVABLE = 0`, `.INSTRUCTIONS = 1` | Discrimina qué tipo de documento opera el tab |
| `FRONT_ENTITLEMENT` (feature flag) | Controla visibilidad de `non_deletable_by_role` en el mensaje de eliminación masiva |
| `AgreementState.agreementDetail.agreement.code` | Código del acuerdo para permisos y para construir el request de exportación |

---

## Puntos frágiles

### 1. `hasRowsLength` no se actualiza cuando cambian las listas
```ts
const hasRowsLength = useMemo(() => {
  if (tabValue === 0) return receivableList.length === 0;
  return instructionList.length === 0;
}, [tabValue, documentFilters]); // ← documentFilters en lugar de receivableList/instructionList
```
El memo depende de `documentFilters` en lugar de las listas. El botón de exportar puede quedar habilitado/deshabilitado incorrectamente si las listas cambian sin que cambien los filtros.

### 2. `isMassiveSelectionDelete` y `excludeIds` se sincronizan en tres `useEffect` distintos
Si se modifica uno solo, los otros dos pueden producir estados inconsistentes. La lógica de exclusión es delicada: asume que `receivableIds` (Redux) refleja exactamente los IDs seleccionables de la página visible.

### 3. Nombre invertido de `isCsv`
En `handleGetFile`, `formatKey === "xlsx"` produce `isCsv = true`. El nombre sugiere lo contrario. Si alguien "corrige" esto sin conocer la lógica del backend, romperá el formato del archivo exportado.

### 4. Modal de QR tiene un condicional siempre-falso
```tsx
{modalContent === ContentTypesEnum.QR_CODE && (
  ...
  {modalContent !== ContentTypesEnum.QR_CODE && ( // ← siempre false
    <EmailModalContent ... />
  )}
)}
```
El bloque de `EmailModalContent` dentro del modal QR nunca se renderiza. [INFERIDO: podría ser un bug latente o feature incompleta.]

### 5. Dependencias de `useEffect` para selección masiva
Varios `useEffect` dependen de `[tabValue]`, `[pagination, receivableList, ...]`, `[receivableIds, tabValue, ...]` simultáneamente. Un cambio de página puede disparar todos en cascada produciendo múltiples dispatches de `setSelectReceivableIds` y `setExcludeIds`.

### 6. Limpieza de store al desmontar
El `useEffect` de cleanup solo despacha `setSelectReceivableIds([])` y `setExcludeIds([])`. No limpia `documentFilterState` ni `paymentDocumentrequest`. Si el componente se monta con datos residuales, `adaptPaymentToMassiveDeletion` puede construir un request incorrecto.

---

## Reglas — Lo que NUNCA debes hacer

- **No uses `tabValue` como índice de tipo directamente** — usa `type` (`ReceivablesTypesEnum`) para discriminar entre RECEIVABLE e INSTRUCTIONS. `tabValue` es el valor del tab UI y no siempre es igual al valor del enum.
- **No modifiques `selectedIds` local sin también actualizar el store** — ambos se mantienen en sincronía a través del `useEffect([selectedIds])`. Si rompes esa sincronía, `useMassiveSelectionFilter` trabajará con IDs desactualizados.
- **No toques `excludeIds` directamente en este componente** — solo se actualiza via los `useEffect` de modo masivo. Modificarlo en otro lugar desincronizaría el conteo.
- **No llames a `handleOnSearch` con `exactFilter=true` desde `DocumentFilters`** — el botón de búsqueda exacta usa `handleClickSearch={() => handleOnSearch(true)}`; el cambio de filtros usa `false`. Invertirlo produce búsquedas incorrectas.
- **No agregues lógica de exportación fuera de `handleGetFile`** — el guard `if (typeFile === FileTypeEnum.DOCUMENTS)` con `return` al final es intencional.
- **No cambies el nombre ni la lógica de `isCsv`** en `handleGetFile` sin validar con el backend el significado real de ese flag.
- **`buildFilterToSearch` siempre pasa `isReportDetail=true` como último argumento a `buildDocumentRequest`** — esto hace que internamente se use `getStateFilterValueReceivables` en lugar de `getStateFilterValue`. No cambies ese valor a `false` sin validar el comportamiento del request resultante en el backend.

---

## Cómo diagnosticar y corregir bugs

1. **Identificar si el bug es de renderizado o de datos**: revisar si el valor incorrecto viene de Redux (usar DevTools) o de `documentFilters` (prop).

2. **Bugs de selección masiva**: verificar los tres `useEffect` relacionados con `isMassiveSelectionDelete`. Revisar el orden de ejecución en el ciclo de render. Confirmar que `receivableIds` y `selectedIds` local están sincronizados.

3. **Bugs de eliminación masiva**: seguir el flujo de `useMassiveSelectionFilter` → `adaptPaymentToMassiveDeletion` → `documentFilterState` en Redux. Verificar que `buildFilterToSearch` despacha `setDocumentFilterState` con el estado de filtro correcto **antes** de que se abra el modal.

4. **Bugs de exportación**: verificar el valor de `isCsv` (recordar que `xlsx → true`), que `page` se fuerza a `1`, y que el `type` del request coincide con el enum correcto (`MassivePaymentDocumentEnum.RECEIVABLE` vs `ExternalPaymentsDocumentEnum.INSTRUCTION`).

5. **Bugs de loading infinito**: revisar `isLoadingTable()`. Puede bloquearse si `processState` de `AgreementState` queda en `"loading"`, o si `isLoadingLocal` nunca se resetea (depende de recibir `GET_RECEIVABLE_LIST_ACTION/GET_INSTRUCTION_LIST_ACTION` en success).

6. **Bugs de modales**: verificar `modalContent` en el slice `ReceivableState`. El `SideModal` se abre/cierra con `openSideModal` de `useDataDocumentTable`; `modalContent` determina **qué** renderiza adentro.

7. **Antes de modificar cualquier función compartida** (`buildDocumentRequest`, `getStateFilterValue`, etc.) verificar cuántos lugares la usan. Seguir la regla global: si hay más de un uso, crear versión independiente.

---

## Tests

### Ubicación
```
src/presentation/receivable/components/ReceivablesTabs/ReceivableTab/__test__/ReceivableTab.test.tsx
```

### Cómo correrlos
```bash
pnpm test -- src/presentation/receivable/components/ReceivablesTabs/ReceivableTab/__test__/ReceivableTab.test.tsx
```

### Qué cubren (45 casos)
- **`isCsv` derivation**: `xlsx` → `true`, `csv` → `false`
- **`buildFilterToSearch` arguments**: verifica que `page` se fuerza a `1`, que `page_size` se preserva, que `exactFilter` es siempre `undefined`, que `isCsv` se pasa correctamente
- **`exportPaymentDocumentList` dispatch**: tipo correcto para RECEIVABLE (`MassivePaymentDocumentEnum.RECEIVABLE`) y para INSTRUCTIONS (`ExternalPaymentsDocumentEnum.INSTRUCTION`); request completo (spread + type)
- **Guard `FileTypeEnum.DOCUMENTS`**: solo ejecuta cuando `typeFile === DOCUMENTS`; no ejecuta para otros tipos
- **Simulación completa de `handleGetFile`**: combinaciones xlsx+RECEIVABLE y csv+INSTRUCTIONS
- **`exactFilterNotApplicable` siempre es `undefined`**: para todos los formatos y tipos

> **Nota**: los tests son de tipo "lógica pura" — no renderizan el componente con React Testing Library. Testean las derivaciones y llamadas a funciones simuladas mediante mocks. No hay tests de integración del componente completo.

---

## Historial de cambios

| Fecha | Bug / Motivo | Resumen |
|---|---|---|
| 2026-02-23 | Revisión de skill | Corrección de documentación incorrecta: `buildFilterToSearch` pasa `isReportDetail=true` (no `false`) a `buildDocumentRequest`. Se actualizó la sección "Reglas" para reflejar el comportamiento real del código. |
| 2026-02-23 | Bug CSV "Por vencerse" | Fix en `getStateFilterValueReceivables` (`paymentDocuments.ts`): (1) se agregó conciencia del tipo de tab vía `documentFilters.type` ("RECEIVABLE"/"INSTRUCTION") — "ALL" ahora filtra solo los estados del tab activo; (2) caso especial para `"TO_PAY,PARTIALLY_PAID"` (Por vencerse) igual a la lógica SOON_DUE → retorna `[11, 13]`; (3) se corrigió el match de tokens de `String.includes()` (subcadena) a `stateTokens.includes()` (exacto). Cuando `type` no es RECEIVABLE ni INSTRUCTION se mantiene comportamiento anterior (fallback a todos los estados). |
