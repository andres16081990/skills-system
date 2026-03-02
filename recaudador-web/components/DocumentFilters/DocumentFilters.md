# DocumentFilters — Agent Skill

## Propósito

`DocumentFilters` es el panel de filtros compartido para búsqueda de documentos de cobro. Permite al usuario filtrar por rango de fechas (emisión y vencimiento), estado del documento, tipo de documento y acuerdo. También gestiona la visualización de una alerta de carga masiva cuando el usuario navega desde el Upload Center.

El componente **soporta dos layouts distintos** controlados por la prop `documentType`:
- **`paymentDocuments`**: Layout completo con Alert de carga masiva, filtros de acuerdo y tipo de documento. Ancho fijo de 800px.
- **`receivable` / `instructions`**: Layout simplificado sin Alert, sin filtro de acuerdo ni de tipo de documento.

Vive en:
```
src/presentation/receivable/components/ReceivablesTabs/ReceivableTab/Filters/DocumentFilters.tsx
```

Es un componente **controlado**: no gestiona el estado de los filtros internamente — lo recibe y lo devuelve mediante callbacks.

---

## Comunicación con el resto de la app

### Cómo recibe datos

| Fuente | Qué recibe |
|---|---|
| Props (`documentFilters`) | Estado completo de los filtros activos (`DocumentFilter`) |
| Props (`canCleanFilter`) | Booleano que controla si el botón "Limpiar filtros" está habilitado |
| Props (`tabValue`) | Determina qué opciones de estado muestra `StateFilter` (0=recaudables, 1=instrucciones, `undefined`=todos) |
| Props (`documentType`) | Determina el layout a renderizar y las etiquetas de los campos de fecha |
| `useLocation` (dentro de `useBulkAlert`) | Lee `state.data.upload_id` y `state.data.file_name` del router para mostrar la alerta de carga masiva |

### Cómo emite datos

| Mecanismo | Qué emite |
|---|---|
| `handleFilterChange(name, value)` | Notifica al padre cada cambio individual de filtro |
| `getFilteredDocuments()` | Notifica al padre que debe ejecutar la búsqueda (botón "Aplicar filtros") |
| `handleCleanFilters()` | Notifica al padre que debe limpiar todos los filtros |
| `handleRemoveBulkFilter()` | Alternativa a `handleCleanFilters` usada en el botón confirmar del Alert (si se provee) |
| `onAlertClose(closeAlertFn)` | Expone la función `closeAlert` del hook al padre para que pueda cerrar el Alert externamente |

### Con qué componentes se comunica directamente

| Componente | Módulo | Presente en |
|---|---|---|
| `DateFilter` | `./DateFilter/DateFilter` | Ambos layouts |
| `StateFilter` | `./StateFilter/StateFilter` | Ambos layouts |
| `AgreementFilter` | `./AgreementFilter/AgreementFilter` | Solo `paymentDocuments` |
| `DocumetTypeFilter` | `./DocumetTypeFilter/DocumetTypeFilter` | Solo `paymentDocuments` |
| `Alert` | `@recaudify/recaudify-design-systems` | Solo `paymentDocuments` |
| `LinkText` (limpiar filtros) | `@recaudify/recaudify-design-systems` | Ambos layouts |
| `SecondaryButton` (aplicar filtros) | `@recaudify/recaudify-design-systems` | Ambos layouts |

### A qué páginas o layouts pertenece

| Página / Componente padre | `documentType` que usa |
|---|---|
| `PaymentDocumentMain` (`src/presentation/paymentDocument/pages/Main/`) | `"paymentDocuments"` |
| `ReceivableTab` (`src/presentation/receivable/components/ReceivablesTabs/ReceivableTab/`) | `"receivable"` o `"instructions"` |

---

## Props

| Prop | Tipo | Requerida | Qué hace | De dónde viene |
|---|---|---|---|---|
| `documentFilters` | `DocumentFilter` | Sí | Estado completo de filtros activos. Controlado externamente. | Estado local del padre (`useState`) o hook `usePaymentDocumentReport` / `useReceivablesTabs` |
| `handleFilterChange` | `(name: string, value?: string) => void` | Sí | Callback invocado por cada sub-filtro cuando el usuario cambia un valor. El padre actualiza `documentFilters`. | Definido en el componente padre |
| `getFilteredDocuments` | `() => void` | Sí | Callback invocado al presionar "Aplicar filtros". Dispara la búsqueda en el padre. | Definido en el componente padre |
| `handleCleanFilters` | `() => void` | Sí | Callback para limpiar todos los filtros. También se pasa a `useBulkAlert`. | Definido en el componente padre |
| `canCleanFilter` | `boolean` | Sí | Controla si el botón "Limpiar filtros" está habilitado. `true` = deshabilitado. | Estado del padre (se activa después de limpiar, se desactiva al cambiar un filtro) |
| `documentType` | `"receivable" \| "instructions" \| "paymentDocuments"` | Sí | Determina el layout a renderizar y las etiquetas de los campos de fecha. | Hardcodeado en el padre según el contexto de uso |
| `tabValue` | `number` | No | Índice de la pestaña activa. Controla qué opciones muestra `StateFilter`. | Prop del `ReceivableTab` o estado del padre |
| `onAlertClose` | `(closeAlertFn: () => void) => void` | No | Permite al padre obtener una referencia a `closeAlert` para cerrarlo externamente. Solo relevante en `paymentDocuments`. | Callback en `PaymentDocumentMain` que almacena la función en un `useRef` |
| `handleRemoveBulkFilter` | `() => void` | No | Reemplaza a `handleCleanFilters` como acción del botón "confirmar" del Alert. Si no se provee, el Alert usa `handleCleanFilters`. | Definido en `PaymentDocumentMain` (`handleCleanFiltersWrapper`) |

---

## Hooks y dependencias

### Hooks propios del componente

| Hook | Propósito |
|---|---|
| `useCallback(disbleCleanFilter)` | Memoiza la función que calcula si el botón "Limpiar" debe estar deshabilitado |
| `useEffect([closeAlert, onAlertClose])` | Expone `closeAlert` al padre llamando `onAlertClose(closeAlert)` cuando cambia |

### Hooks externos que consume

| Hook | Datos que toma | Acciones que dispara |
|---|---|---|
| `useBulkAlert(handleCleanFilters)` | Ninguno de entrada — lee `useLocation().state` internamente | Llama `handleCleanFilters` vía `removeBulkFilter` cuando el usuario confirma quitar el filtro de carga masiva. Expone `closeAlert` para que el padre lo use. |
| `useTranslation` | `properties.agreement.components.receivablesTabs.filters` | Provee etiquetas: `cleanFilter`, `applyFilters`, etiquetas de fechas, mensaje del Alert |

#### Detalle de `useBulkAlert`

- Lee `useLocation().state.data.upload_id` y `state.data.file_name`
- Si `upload_id` existe → `showAlert = true`, `alertType = "info"`, `alertMessageParams = { batchId, filename }`
- Expone: `showAlert`, `alertType`, `alertMessageParams`, `closeAlert`, `openAlert`, `removeBulkFilter`
- En `DocumentFilters` solo se usan: `showAlert`, `alertType`, `alertMessageParams`, `closeAlert`
- `showAlert = true` deshabilita `AgreementFilter` y `DocumetTypeFilter` (prop `disabled`)

#### Detalle de `AgreementFilter` (sub-componente con acceso a Redux)

- Llama `useTypedSelector(stateSelectorAgreement)` internamente para obtener `filteredAgreements`
- Solo muestra acuerdos en estado `ACTIVE`
- Emite hacia `DocumentFilters` vía `handleFilterChange("agreementSelected", value)`

#### Detalle de `DateFilter` (sub-componente con hook propio)

- Usa `useDateFilters(documentFilters, handleFilterChange)` para manejar rangos de fechas locales
- Sincroniza los valores del calendario con `documentFilters.emition_date_*` y `documentFilters.due_date_*`
- Emite hacia `DocumentFilters` vía `handleFilterChange(TypeDateAgreement.EMITION_DATE | DUE_DATE, JSON.stringify([Date, Date]))`
- El prop `canCleanFilter = true` fuerza `value = null` en los `DateRangePicker`, limpiando visualmente el calendario

---

## Flujo de datos

### 1. Cambio de un filtro individual

```
Usuario interactúa con un sub-filtro (DateFilter / StateFilter / AgreementFilter / DocumetTypeFilter)
  → Sub-filtro llama handleFilterChange(name, value) — prop recibida de DocumentFilters
  → DocumentFilters reenvía la llamada al padre sin transformaciones
  → Padre actualiza documentFilters en su estado
  → React re-renderiza DocumentFilters con los nuevos documentFilters
  → canCleanFilter pasa a false (el padre lo controla)
```

**Excepción — DateFilter**: el formato del `value` que sube es `JSON.stringify([Date, Date])`. El padre (`PaymentDocumentMain.handleFilterChange`) lo parsea si el campo está en `TypeDateAgreement`.

### 2. Aplicar filtros

```
Usuario presiona "Aplicar filtros" (SecondaryButton)
  → DocumentFilters llama getFilteredDocuments()
  → Padre ejecuta la búsqueda (despacha getPaymentDocumentList o equivalente)
```

El botón está **deshabilitado** si `onDisabledBtnReport()` retorna `true` (ver lógica de validación de fechas abajo).

### 3. Limpiar filtros

```
Usuario presiona "Limpiar filtros" (LinkText)
  → DocumentFilters llama handleCleanFilters()
  → Padre resetea documentFilters a documentFilterInitialize
  → canCleanFilter pasa a true
  → DateFilter detecta canCleanFilter=true y pone value=null en los DateRangePicker
```

### 4. Alerta de carga masiva (solo `paymentDocuments`)

```
Padre navega a PaymentDocuments con state.data.upload_id (desde Upload Center)
  → useBulkAlert detecta upload_id en useLocation().state → showAlert = true
  → DocumentFilters muestra el Alert con batchId y filename
  → AgreementFilter y DocumetTypeFilter quedan disabled={true}
  → onAlertClose prop → useEffect llama onAlertClose(closeAlert)
      → Padre almacena closeAlert en closeAlertRef.current
  → Usuario presiona "Quitar filtro" en el Alert
      → Alert llama handleRemoveBulkFilter ?? handleCleanFilters
      → Padre ejecuta la lógica de limpieza del filtro bulk
  → Usuario presiona la X del Alert
      → closeAlert() → showAlert = false
```

### 5. Validación de fechas (`isInvalidDate`)

El botón "Aplicar filtros" se deshabilita cuando `isInvalidDate()` retorna `true`:

```
Caso A: ambas fechas (emition_date_initial Y due_date_initial) están presentes
  → Si alguna es más reciente que MIN_DATE (hace 6 meses):
      → retorna true SI due_date < emition_date  (rango invertido)
      → retorna false si due_date >= emition_date
  → Si NINGUNA es más reciente que MIN_DATE:
      → retorna true siempre (fechas demasiado antiguas consideradas inválidas)

Caso B: solo una fecha presente
  → retorna true si la fecha no es un momento válido

Caso C: ninguna fecha presente
  → retorna false (sin fechas = válido, puede buscar)
```

---

## Dependencias críticas

| Dependencia | Rol |
|---|---|
| `DocumentFilter` (`receivable/types/receivable.d.ts`) | Tipo del estado de filtros. Cambiar sus campos puede romper todas las referencias a `documentFilters.*` en el componente |
| `TypeDateAgreement` (`receivable/types/receivable.d.ts`) | Enum `EMITION_DATE = "emition_date"` / `DUE_DATE = "due_date"`. Usado en `DateFilter` y esperado por `handleFilterChange` del padre |
| `MIN_DATE_AGREEMENT = 6` (`receivable/types/receivable.d.ts`) | Meses hacia atrás que el `DateRangePicker` permite seleccionar |
| `useBulkAlert` (`receivable/hooks/useBulkAlert.tsx`) | Gestiona la visibilidad y datos del Alert de carga masiva |
| `useDateFilters` (`paymentDocument/hooks/useDateFilters.tsx`) | Usado dentro de `DateFilter`. Convierte los rangos a formato local y maneja la limpieza visual del calendario |
| `AgreementState.filteredAgreements` (Redux) | `AgreementFilter` lo lee directamente del store. Si está vacío, el dropdown solo mostrará "Todos los acuerdos" |
| `moment` | Usado para calcular `MIN_DATE` (al módulo se importa como `moments`) y para validar fechas en `isInvalidDate` |
| i18n key `properties.agreement.components.receivablesTabs.filters` | Todas las etiquetas del componente. Si cambia la estructura de este key, los textos se muestran vacíos |

---

## Puntos frágiles

### 1. `disbleCleanFilter` tiene dependencias incompletas en `useCallback`

```ts
const disbleCleanFilter = useCallback(() => {
  if (documentFilters.code_process) return true;
  return canCleanFilter;
}, [documentFilters]); // ← canCleanFilter NO está en el array de deps
```

Si `canCleanFilter` cambia sin que cambie `documentFilters`, la función memoizada devolverá un valor desactualizado (stale closure). Solo afecta al layout `paymentDocuments`.

### 2. Comportamiento inconsistente de "Limpiar filtros" entre layouts

En el layout **`paymentDocuments`**, el botón "Limpiar" usa `disbleCleanFilter()` que incluye el guard de `code_process`.
En el layout **`receivable` / `instructions`**, el botón usa `canCleanFilter` directamente y no tiene guard de `code_process`.

Si en el futuro `receivable`/`instructions` también puede tener `code_process`, el guard deberá agregarse manualmente al segundo layout.

### 3. `onAlertClose` puede disparar el `useEffect` en cada render del padre

Si el padre no envuelve la función en `useCallback`, `onAlertClose` será una nueva referencia en cada render, y el `useEffect([closeAlert, onAlertClose])` se ejecutará en cada render del padre. Esto no produce errores visibles pero sí llamadas redundantes.

### 4. La lógica de `isInvalidDate` es no intuitiva

Cuando ambas fechas están presentes y **ninguna** cae dentro de los últimos 6 meses, la función retorna `true` (inválido). Esto significa que un rango de fechas completamente en el pasado (más de 6 meses atrás) deshabilitará el botón "Aplicar", aunque el rango en sí sea lógicamente coherente (initial ≤ final). [INFERIDO] — validar si este es el comportamiento de negocio deseado.

### 5. El nombre del sub-componente tiene un typo

El archivo y el componente se llaman `DocumetTypeFilter` (falta una `n`). Esto es un typo histórico — **no lo corrijas** sin actualizar todas las importaciones, ya que son 3 archivos (`DocumentFilters.tsx`, `DocumentFilters.test.tsx` y el propio archivo del componente).

### 6. `MIN_DATE` se calcula en el módulo, fuera del componente

```ts
const MIN_DATE = moments().add(-6, "months");
```

Este valor se calcula **una sola vez cuando el módulo se carga** (no en cada render). Si la aplicación permanece abierta por más de 6 meses sin recarga, `MIN_DATE` quedará desactualizado. En la práctica no es un problema, pero los tests deben congelar el tiempo para resultados deterministas (como hace `DocumentFilters.test.tsx` con `jest.useFakeTimers()`).

---

## Reglas — Lo que NUNCA debes hacer

1. **No gestiones el estado de `documentFilters` dentro de `DocumentFilters`**. Este componente es totalmente controlado. El estado vive en el padre. Si necesitas lógica adicional de filtros, agrégala en el padre o en su hook (`usePaymentDocumentReport` / `useReceivablesTabs`).

2. **No llames `getFilteredDocuments()` desde dentro del componente** salvo en el click del botón "Aplicar filtros". La búsqueda automática ante cualquier cambio de filtro es responsabilidad del padre.

3. **No modifiques el layout de `receivable`/`instructions` para agregar `AgreementFilter` o `DocumetTypeFilter`** sin verificar que el padre provee los props necesarios. Estos filtros solo existen en el layout `paymentDocuments` porque solo esa página tiene multi-acuerdo y multi-tipo.

4. **No elimines la prop `disabled={showAlert}` de `AgreementFilter` y `DocumetTypeFilter`** sin coordinar con el flujo del Upload Center. Cuando hay un filtro de carga masiva activo (`showAlert = true`), cambiar el acuerdo o el tipo de documento puede romper la coherencia del filtro de `code_process`.

5. **No agregues lógica de Redux directamente en `DocumentFilters`**. El único acceso a Redux es indirecto (a través de `AgreementFilter` que lee `filteredAgreements`). `DocumentFilters` debe seguir siendo un componente de presentación pura con comunicación solo por props y callbacks.

6. **No corrijas el typo `DocumetTypeFilter` en un cambio parcial**. Debe hacerse en todos los archivos que lo referencian simultáneamente o no hacerse.

---

## Cómo diagnosticar y corregir bugs

### El botón "Aplicar filtros" está siempre deshabilitado

1. Verificar `isInvalidDate()`: loguear `documentFilters.emition_date_initial`, `documentFilters.due_date_initial` y `MIN_DATE`.
2. Caso frecuente: el padre está pasando un rango de fechas donde ambas fechas son anteriores a 6 meses — la lógica las considera inválidas.
3. Verificar que el padre no esté pasando fechas en un formato que `moment` no parsea correctamente (ej: formato sin zona horaria).

### El botón "Limpiar filtros" está siempre deshabilitado en `paymentDocuments`

1. Verificar `documentFilters.code_process` — si tiene valor, `disbleCleanFilter()` retorna `true` siempre.
2. Verificar `canCleanFilter` en el padre — si es `true`, también deshabilita el botón.
3. Recordar que `disbleCleanFilter` puede tener un valor stale de `canCleanFilter` (ver punto frágil #1). Si solo cambia `canCleanFilter` sin cambiar `documentFilters`, forzar un re-render del padre.

### El Alert de carga masiva no aparece al navegar desde Upload Center

1. Verificar que el padre navega con `state.data.upload_id` en el objeto de navegación (`navigate(path, { state: { data: { upload_id: ... } } })`).
2. Verificar que `documentType === "paymentDocuments"` — el Alert solo se renderiza en ese layout.
3. Verificar que `useBulkAlert` está leyendo correctamente `useLocation().state` — si el router no preserva el state (ej: redirección extra), `upload_id` llegará vacío.

### El Alert no se cierra cuando el padre llama `closeAlertRef.current()`

1. Verificar que `onAlertClose` está siendo pasada como prop a `DocumentFilters`.
2. Verificar que el `useEffect([closeAlert, onAlertClose])` se ejecutó y llamó `onAlertClose(closeAlert)`.
3. Verificar que el padre almacena la función recibida (y no sobrescribe con `undefined`).
4. Si el padre no pasa `onAlertClose`, el componente nunca expone `closeAlert` y el padre no puede cerrar el Alert externamente.

### Los valores del calendario no se actualizan al cambiar `documentFilters` externamente

Este comportamiento lo gestiona `useDateFilters` (dentro de `DateFilter`):
1. Verificar que los campos `emition_date_initial`, `emition_date_final`, `due_date_initial`, `due_date_final` del `documentFilters` están cambiando correctamente en el padre.
2. Verificar que `canCleanFilter` no está en `true` — cuando es `true`, `DateFilter` fuerza `value = null` sin importar los filtros.
3. Si los filtros vienen de un restore (edición de documento), verificar que `filters.isFromEditedDocument = true` — `useDateFilters` usa ese flag para mostrar el rango por defecto.

### Los acuerdos no aparecen en el dropdown de `AgreementFilter`

1. Verificar que `AgreementState.filteredAgreements` en Redux tiene datos.
2. `AgreementFilter` solo muestra acuerdos con `state.name === AgreementStateEnum.ACTIVE`. Verificar que los acuerdos tienen ese estado.
3. Verificar que `getAgreements` fue despachado antes de que `DocumentFilters` se montara (lo hace `usePaymentDocumentReport` en su `useEffect([])`).

---

## Tests

**Ubicación:**
```
src/presentation/receivable/components/ReceivablesTabs/ReceivableTab/Filters/__test__/DocumentFilters.test.tsx
```

**Cómo correrlos:**
```bash
pnpm test -- src/presentation/receivable/components/ReceivablesTabs/ReceivableTab/Filters/__test__/DocumentFilters.test.tsx
```

**Estructura de los tests:**

| Test | Qué verifica |
|---|---|
| `renders non-payment layout (receivable)` | Que el layout `receivable` renderiza `DateFilter` + `StateFilter` + botones, sin `AgreementFilter` ni `DocumetTypeFilter` |
| `clicking applyFilters calls getFilteredDocuments (non-payment)` | Que el botón "Aplicar filtros" llama `getFilteredDocuments` |
| `clicking cleanFilter calls handleCleanFilters (non-payment)` | Que el botón "Limpiar" llama `handleCleanFilters` |
| `disables applyFilters when isChangedFilters is false` | Que `onDisabledBtnReport()` deshabilita el botón cuando `isChangedFilters` es false |
| `disables applyFilters when date range is invalid` | Que un rango con `due < emition` deshabilita el botón (solo verifica que el botón existe — no verifica `disabled` explícitamente) |
| `renders paymentDocuments layout` | Que el layout `paymentDocuments` incluye los 4 filtros + Alert area + botones |
| `paymentDocuments: shows bulk alert when showAlert is true` | Que el `Alert` aparece con tipo y mensaje correctos cuando `useBulkAlert` retorna `showAlert=true` |
| `paymentDocuments: alert confirm uses handleRemoveBulkFilter` | Que el botón confirmar del Alert llama `handleRemoveBulkFilter` si está provisto |
| `paymentDocuments: alert confirm falls back to handleCleanFilters` | Que el botón confirmar llama `handleCleanFilters` cuando `handleRemoveBulkFilter` no se provee |
| `calls onAlertClose with closeAlert function` | Que `onAlertClose` es llamado con la función `closeAlert` del hook |
| `paymentDocuments: disables cleanFilter when code_process exists` | Que el botón "Limpiar" está deshabilitado cuando `code_process` tiene valor |
| `paymentDocuments: disables AgreementFilter and DocumetTypeFilter when showAlert is true` | Que el layout sigue renderizando ambos filtros aunque estén deshabilitados |

**Notas sobre los tests:**
- Todos los sub-componentes (`DateFilter`, `AgreementFilter`, `StateFilter`, `DocumetTypeFilter`) están mockeados como `<div data-testid="..." />`. Los tests no verifican sus props (como `disabled`).
- `useBulkAlert` está mockeado globalmente — los tests controlan su retorno con `mockUseBulkAlert.mockReturnValue(...)`.
- El tiempo está congelado a `2026-02-17T12:00:00Z` para hacer determinista el cálculo de `MIN_DATE`.
- El test de fecha inválida verifica que el botón "existe" (`toBeDefined`) pero no que esté `disabled` — este es un gap de cobertura.
