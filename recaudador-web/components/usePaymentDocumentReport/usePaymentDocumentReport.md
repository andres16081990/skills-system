# usePaymentDocumentReport — Agent Skill

## Propósito

`usePaymentDocumentReport` es el hook central de `PaymentDocumentMain`. Gestiona:
- El estado local de filtros (`documentFilters`)
- La carga de acuerdos según el rol del usuario
- La detección del origen de navegación (`DASHBOARD` / `UPLOAD_CENTER`) y la pre-población de filtros correspondiente
- La construcción del request de búsqueda (`buildFilterToSearch` → `buildDocumentRequest`)
- La persistencia de filtros en Redux para restaurarlos al volver desde edición de documento
- La señal `changeFiltersRef` que indica si el contexto de entrada es desde el dashboard

Vive en:
```
src/presentation/paymentDocument/hooks/usePaymentDocumentReport.ts
```

Solo es usado por `PaymentDocumentMain.tsx`. No se reutiliza en ningún otro lugar.

---

## Firma

```ts
export const usePaymentDocumentReport = (
  state: any,                                          // useLocation().state de React Router
  pagination: PaginationMetadata,                      // paginación actual desde Redux
  previousFiltersAndPagination: PreviousFiltersAndPagination, // desde ReceivableState
  setOpenModal: Dispatch<SetStateAction<boolean>>,     // para abrir modal al venir de UPLOAD_CENTER
)
```

### Lo que retorna

| Campo | Tipo | Descripción |
|---|---|---|
| `documentFilters` | `DocumentFilter` | Estado local de filtros activos |
| `filteredAgreements` | `Agreement[]` | Acuerdos filtrados desde AgreementState |
| `setDocumentFilters` | `Dispatch` | Setter directo de filtros locales |
| `buildFilterToSearch` | `function` | Construye el PaymentDocumentRequest listo para despachar |
| `setStateFilterByNavigatorState` | `function` | Convierte `CardSelected` → valor de estado para el filtro |
| `setCanCleanFilter` | `Dispatch` | Controla si el botón de limpiar filtros está habilitado |
| `canCleanFilter` | `boolean` | Estado del botón limpiar filtros |
| `changeFiltersRef` | `MutableRefObject<boolean>` | Flag de contexto de navegación dashboard |

---

## Estado interno y refs

| Nombre | Tipo | Propósito |
|---|---|---|
| `documentFilters` | `useState<DocumentFilter>` | Filtros activos en la pantalla. Inicializa en `documentFilterInitialize` |
| `canCleanFilter` | `useState<boolean>` | `true` por defecto. Se pone en `false` al recibir filtros del dashboard |
| `filtersFromDashboard` | `useState<boolean>` | Flag temporal para disparar la búsqueda inicial cuando viene del dashboard |
| `changeFiltersRef` | `useRef<boolean>(false)` | `true` cuando el contexto de navegación es dashboard. Se usa como argumento `fromDashboardCardSelected` en `buildDocumentRequest` |

---

## Efectos y cuándo se ejecutan

### 1. Carga de acuerdos (`[]`)
Se ejecuta al montar. Carga acuerdos con `page_size: 500` si el usuario es admin, o con `roleId` si no lo es.

### 2. Efecto de `processState` (`[processState, documentFilters]`)
Se activa cuando `GET_PAYMENT_DOCUMENT_LIST` termina en `"success"`.
- Calcula `filtersChanged` comparando filtros actuales con `documentFilterInitialize`
- Actualiza `documentFilters.isChangedFilters` y despacha `setChangedFilters`
- Despacha `setIdleState()`
- **⚠️ Guarda crítica:** Solo resetea `changeFiltersRef.current = false` si NO estamos en el flujo dashboard + "por vencerse" (`CardSelected.PENDING_COLLECTION_IN_CENTS`). Ver sección "Puntos frágiles".

### 3. Efecto de navegación (`[state]`)
Se activa cuando cambia el `state` de `useLocation`.

**Si `state.source === DASHBOARD`:**
- Calcula fechas desde `metaData` según el card seleccionado
- Setea `documentFilters` con el estado y las fechas correspondientes
- Pone `filtersFromDashboard = true` para disparar la búsqueda en el siguiente efecto
- Pone `canCleanFilter = false`
- Pone `changeFiltersRef.current = true`

**Si `state.source === UPLOAD_CENTER`:**
- Resuelve el `agreementSelected` a partir de `state.data.agreement_id`
- Setea `documentFilters` con `type`, `code_process`, `agreementSelected`
- Pone `changeFiltersRef.current = false`
- Despacha `getPaymentDocumentList` directamente (sin esperar al efecto de `filtersFromDashboard`)
- Llama `setOpenModal(state.openModal)`

### 4. Efecto de búsqueda desde dashboard (`[filtersFromDashboard, documentFilters, ...]`)
Se activa cuando `filtersFromDashboard` es `true`. Llama `buildFilterToSearch` con el `dashboardState` como `overrideState`, despacha `getPaymentDocumentList` y resetea `filtersFromDashboard = false`.

### 5. Efecto de persistencia de filtros (`[documentFilters]`)
Se activa en cada cambio de `documentFilters`. Persiste los filtros actuales en `ReceivableState.previousFiltersAndPagination` para restaurarlos al volver desde edición. **Solo actúa si `isFromEditedDocument === false`** para evitar bucles.

---

## `buildFilterToSearch` — función clave

```ts
buildFilterToSearch(
  pagination: PaginationMetadata,
  exactFilter?: boolean,
  isFormExportFile?: boolean,
  overrideState?: string,      // sobreescribe documentFilters.state
  overrideFilters?: DocumentFilter, // sobreescribe todos los filtros
): PaymentDocumentRequest
```

Internamente:
1. Usa `overrideFilters ?? documentFilters` como base
2. Si `overrideState` está presente, lo aplica como `.state`
3. Despacha `setDocumentFilterState` con el estado resultante
4. Llama `buildDocumentRequest` pasando:
   - Los filtros construidos
   - `filteredAgreements.flatMap(p => p.agreement_code)` como acuerdos
   - La paginación
   - `exactFilter`, `isFormExportFile`
   - `false` (hardcoded — reservado para uso interno de `buildDocumentRequest`)
   - **`state?.source === ComponentSource.DASHBOARD && changeFiltersRef.current`** → el `fromDashboardCardSelected`

---

## `setStateFilterByNavigatorState` — mapeo de cards

| `CardSelected` | Resultado |
|---|---|
| `PENDING_COLLECTION_IN_CENTS` | valor de `PayStateEnum.SOON_DUE` ("SOON_DUE") |
| `OVERDUE_IN_CENTS` | valor de `PayStateEnum.DUE` |
| cualquier otro | `"ALL"` |

---

## `getDatesFromDashboardCardSelected` — mapeo de fechas

| Card | Campos que setea |
|---|---|
| `PENDING_COLLECTION_IN_CENTS` o `TOTAL_PAYMENT_DOCUMENTS_IN_CENTS` | `emition_date_initial` / `emition_date_final` |
| cualquier otro (ej. `OVERDUE_IN_CENTS`) | `due_date_initial` / `due_date_final` |

Normaliza el rango: si `initialDate > finalDate`, los intercambia.

---

## Puntos frágiles

### 1. `changeFiltersRef` — flag de contexto de navegación (crítico)

`changeFiltersRef.current` es el mecanismo por el que `buildFilterToSearch` sabe si debe pasar `fromDashboardCardSelected = true` a `buildDocumentRequest`. Su ciclo de vida:

```
Mount:            changeFiltersRef.current = false
state [DASHBOARD]: changeFiltersRef.current = true   ← se activa al entrar del dashboard
processState success:
  si NOT (dashboard + PENDING_COLLECTION_IN_CENTS):  changeFiltersRef.current = false
  si SÍ  (dashboard + PENDING_COLLECTION_IN_CENTS):  se mantiene true (permite paginar correctamente)
handleFilterChange / handleCleanFilters (en PaymentDocumentMain):
  changeFiltersRef.current = false                   ← el usuario tocó los filtros, se resetea
```

**Regla:** No setear `changeFiltersRef.current = false` en el efecto de `processState` cuando el contexto es dashboard + "por vencerse" (`CardSelected.PENDING_COLLECTION_IN_CENTS`). Hacerlo rompe la paginación en ese flujo.

### 2. Doble disparo del efecto de `filtersFromDashboard`
El efecto tiene como dependencias `[filtersFromDashboard, documentFilters, pagination, dispatch, state, filteredAgreements]`. Si alguna de esas deps cambia mientras `filtersFromDashboard` está en `true`, el efecto puede dispararse más de una vez. La guarda `if (filtersFromDashboard)` previene el segundo dispatch porque el primer disparo pone `filtersFromDashboard = false`.

### 3. `state` es `any`
El parámetro `state` no tiene tipo fuerte. Si el dashboard cambia la forma del objeto que pasa como location state, ningún error de compilación lo detectará. Los campos usados son: `state.source`, `state.cardSelected`, `state.metaData.initialDate`, `state.metaData.finalDate`, `state.data.agreement_id`, `state.data.origin`, `state.data.upload_id`, `state.openModal`.

### 4. `filteredAgreements` se usa directamente en UPLOAD_CENTER
En el flujo UPLOAD_CENTER, `buildDocumentRequest` recibe `filteredAgreements` (el array completo de acuerdos) en vez de `filteredAgreements.flatMap(p => p.agreement_code)`. Esto difiere del comportamiento de `buildFilterToSearch`. Validar si es intencional antes de tocar ese flujo.

### 5. Efecto de persistencia puede correr en loop si `isFromEditedDocument` nunca se resetea
El efecto `[documentFilters]` persiste filtros siempre que `isFromEditedDocument === false`. Si `usePrevoiusFilters` falla al resetear `isFromEditedDocument` a `false` (o si `previousFiltersAndPagination` se actualiza por este mismo dispatch), se puede generar un loop de renders.

---

## Reglas — Lo que NUNCA debes hacer

1. **No resetees `changeFiltersRef.current = false` en el efecto de `processState` sin la guarda del flujo dashboard + `PENDING_COLLECTION_IN_CENTS`**. Eso rompe la paginación al entrar desde el dashboard con el filtro "por vencerse".

2. **No llames `buildDocumentRequest` directamente desde `PaymentDocumentMain`**. Siempre usa `buildFilterToSearch` para garantizar que `fromDashboardCardSelected`, los `agreementCodes` y `setDocumentFilterState` estén correctamente sincronizados.

3. **No modifiques `documentFilters` directamente sin pasar por `setDocumentFilters`**. El efecto de persistencia `[documentFilters]` depende del state de React; una mutación directa no lo dispararía.

4. **No agregues lógica de fuente de navegación (`DASHBOARD`/`UPLOAD_CENTER`) fuera de este hook**. `PaymentDocumentMain` no debe chequear `state.source` para construir filtros — eso es responsabilidad exclusiva de este hook.

---

## Tests

**Ubicación:**
```
src/presentation/paymentDocument/hooks/__test__/usePaymentDocumentReport.test.tsx
```

**Cómo correrlos:**
```bash
pnpm test -- src/presentation/paymentDocument/hooks/__test__/usePaymentDocumentReport.test.tsx
```

**Cobertura actual:**

| Describe | Qué valida |
|---|---|
| `initialization and getAgreements` | `getAgreements` con `roleId` (no-admin) y sin él (admin) |
| `processState effect` | Que `setChangedFilters` e `setIdleState` se llaman en success |
| `buildFilterToSearch` | Que llama `buildDocumentRequest` con los parámetros correctos y respeta `overrideState` |
| `setStateFilterByNavigatorState` | Mapeo correcto para `PENDING_COLLECTION_IN_CENTS`, `OVERDUE_IN_CENTS`, y otros |
| `state effect - Dashboard navigation` | Que setea `isChangedFilters: true` y `canCleanFilter: false` al venir del dashboard |
| `state effect - Upload Center navigation` | Que despacha `getPaymentDocumentList`, llama `setOpenModal`, resuelve `agreementSelected` |
| `previousFiltersAndPagination effect` | Que llama `setPreviousFilterAndPagination` al cambiar `documentFilters`, y no lo hace si `isFromEditedDocument: true` |
| `canCleanFilter` | Estado inicial `true` y actualización vía `setCanCleanFilter` |
| `filteredAgreements` | Que retorna los acuerdos del store correctamente |

> **Nota:** El test de `processState effect` verifica que los dispatches **no** se llaman durante el render inicial (cuando `processState` viene en `idle`), no el caso de `success` con re-render. Para testear el caso `success` correctamente se necesitaría `act` + `rerender` con el state actualizado.
