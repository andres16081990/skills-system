# DashboardTabComponent — Agent Skill

## Propósito

Barra de navegación del dashboard de la página de inicio (`Home`). Permite al usuario seleccionar un período de tiempo predefinido (tabs) o un rango personalizado (DateRangePicker) para filtrar los datos que muestran las tarjetas de resumen (`DashboardCardsComponent`).

Vive en:

```
src/presentation/home/pages/components/dashboardTabComponent/DashboardTabComponent.tsx
```

Toda la lógica de estado y negocio vive en el hook:

```
src/presentation/home/pages/hooks/useDashboardTab.ts
```

---

## Comunicación con el resto de la app

### Recibe datos

- **Props** del padre `Home` (función `dashboardComponent`):
  - `translate: TFunction` — función de traducción con keyPrefix `"properties.home.components"`.
- **Redux** — dos slices (consumidos por `useDashboardTab`):
  - `ConciliationState` → `dashboardConfiguration` (tabValue, metaData con fechas) y `keepDashboardConfiguration` (flag para restaurar estado).
  - `EntitlementState` → `permissionsByUser` (name, id) para determinar si es admin y construir el request con o sin `roleId`.

### Emite datos / mutations al store

| Acción despachada | Cuándo |
|---|---|
| `getDashboardHomeReport(request)` | Cada vez que cambia `dashboardDate` (useEffect) |
| `setConciliationDashboardConfiguration({...metaData: dateRange})` | Al cambiar el rango del DateRangePicker |
| `setConciliationDashboardConfiguration({...tabValue})` | Al cambiar de tab |
| `setConciliationDashboardConfiguration(initialDashboardConfiguration)` | Al montar, si `keepDashboardConfiguration === false` |
| `setKeepDashboardConfiguration(false)` | Al desmontar (cleanup del useEffect de montaje) |

### Componentes hermanos directos

- `DashboardCardsComponent` — recibe implícitamente los datos del slice `ConciliationState` que este componente actualiza vía dispatch.

### Dónde vive en el árbol

```
Home
└── dashboardComponent() [condicional: acuerdo activo + permiso "Home_dashboard.Ver/Consultar.Ver"]
    ├── DashboardTabComponent ← aquí
    └── DashboardCardsComponent
```

---

## Props

| Nombre | Tipo | Requerida | Qué hace |
|---|---|---|---|
| `translate` | `TFunction` | ✅ | Traduce labels de tabs, DateRangePicker, botones y título dinámico |

---

## Hooks consumidos

| Hook | Datos que retorna | Acciones que dispara |
|---|---|---|
| `useDashboardTab()` | `handleChange`, `tabValue`, `handleChangeRangeDataPicker`, `handleOnSubmit`, `handleOnclean`, `disableSubmit`, `resetCalendar`, `setResetCalendar`, `showDateFilterModal`, `setShowDateFilterModal`, `handleResetFilters`, `getDefaultDateValues`, `disabledResetFilters` | Despacha acciones Redux listadas arriba |
| `useQuery()` | `isTablet` | — |

---

## Estructura visual

### Desktop (isTablet = true)
```
[TabContainer: Tab1 | Tab2 | Tab3 | Tab4 | Tab5] [DateRangePicker] [SecondaryButton: Consultar]
[LinkText: Restablecer filtros]
[H2: Título dinámico según tab o rango]
```

### Mobile (isTablet = false)
```
[IconButton: DateRangeIcon]  ← abre Modal
[TabContainer: scrollable]
[LinkText: Restablecer filtros]
[H2: Título dinámico]

Modal (MUI):
  [DateRangePicker] [SecondaryButton: Consultar]  ← disabledOpenModal=true
```

---

## Flujo de datos

### Selección por tab
1. Usuario hace clic en un tab → `handleChange(_, newValue)` → `setTabValue(newValue)` + `setResetCalendar(true)`.
2. `useEffect([tabValue])` → `setDashboardDate(getDatesByTabvalue(tabValue))` + dispatch `setConciliationDashboardConfiguration({tabValue})`.
3. `useEffect([dashboardDate])` → dispatch `getDashboardHomeReport(request)` (con o sin `roleId` según rol).
4. `DashboardCardsComponent` reacciona al nuevo estado de Conciliación.

### Selección por rango de fechas
1. Usuario selecciona fechas en `DateRangePicker` → `handleChangeRangeDataPicker(dates)` → `setDashboardDateRange(getDateByRange(dates))` + dispatch `setConciliationDashboardConfiguration({metaData: dateRange})`.
2. Usuario hace clic en "Consultar" → `handleOnSubmit()` → `setDashboardDate(dashboardDateRange)` → el `useEffect([dashboardDate])` dispara el report.
3. `tabValue` se resetea a `null` en `handleOnSubmit`.

### Reset de filtros
- `handleResetFilters()` → `setDashboardDateRange(DashboardHomeInitial)` + `setResetCalendar(true)` + `setTabValue(null)`.
- Cuando `tabValue === null` y no hay rango: `disabledResetFilters()` retorna `true` → `LinkText` queda deshabilitado.
- Al quedar `tabValue === null`: `useEffect([tabValue])` llama `setDashboardDate(DashboardHomeInitial)` → dispara nuevo report con fechas vacías.

### Restauración de estado (keepDashboardConfiguration)
- Si `keepDashboardConfiguration === true` al montar, `setValuesByConfiguration()` restaura el `tabValue` o el rango de fechas del slice `dashboardConfiguration`.
- Útil cuando el usuario navega a otra sección y vuelve al Home, conservando el filtro previo.

---

## Constantes de tabs

Definidas en `dashboardConstants.tsx`:

```ts
export const dashboardTabs = [
  { label: "dashboard.tabTitle1", id: "dashboard.tabTitle1" }, // 7 días
  { label: "dashboard.tabTitle2", id: "dashboard.tabTitle2" }, // 15 días
  { label: "dashboard.tabTitle3", id: "dashboard.tabTitle3" }, // 1 mes
  { label: "dashboard.tabTitle4", id: "dashboard.tabTitle4" }, // 3 meses (trimestre)
  { label: "dashboard.tabTitle5", id: "dashboard.tabTitle5" }, // 6 meses (semestre)
];
```

Cada tab mapea a un `tabValue` (0–4) que `getDatesByTabvalue` convierte en fechas:

| tabValue | Período | ValuesInDatesToAdd | Unidad |
|---|---|---|---|
| 0 | 7 días | `SEVEN_DAYS` | `"day"` |
| 1 | 15 días | `FIFTEEN_DAYS` | `"day"` |
| 2 | 1 mes | `MONTH` | `"month"` |
| 3 | 3 meses | `QUARTER` | `"month"` |
| 4 | 6 meses | `SEMESTER` | `"month"` |
| `null` | Rango libre | — | — |

**Atención**: `getFinalDate` usa `moment.subtract`, no `add`. `initialDate` es HOY y `finalDate` es la fecha en el pasado. El nombre `finalDate` puede confundir — en realidad es la fecha más antigua del rango.

---

## Helpers clave

### `getDinamicTitle(translate, tabValue)` — `selectDocumentHelper.ts`
Retorna el título H2 según el tab activo. Cuando `tabValue === null` (rango libre o sin filtro) retorna `translate("dashboard.summaryDateRange")`.

### `getDatesByTabvalue(tabValue)` — `selectDocumentHelper.ts`
Convierte un índice de tab en `DashboardHomeRequest { initialDate, finalDate }`. Usa `moment().subtract`.

### `getDateByRange(dates)` — `selectDocumentHelper.ts`
Convierte el array `[Date, Date]` del DateRangePicker en `DashboardHomeRequest`. Ordena las fechas (earlierDate → finalDate, laterDate → initialDate).

---

## Puntos frágiles

### 1. `tabValue` tipado como `any`
```ts
const [tabValue, setTabValue] = useState<any>(null);
```
Ni TypeScript ni el linter detectarán si alguien pasa un valor inválido. El switch en `getDatesByTabvalue` tiene un default que retorna `""`, pero eso puede producir requests con `finalDate: ""` al backend.

### 2. `DashboardHomeInitial` con fechas vacías
Cuando `tabValue === null` y no hay rango seleccionado, se despacha `getDashboardHomeReport({ initialDate: "", finalDate: "" })`. El comportamiento del backend ante este input debe estar controlado.

### 3. `keepDashboardConfiguration` + `dashboardConfiguration` en el mismo `useEffect`
`setValuesByConfiguration` lee `dashboardConfiguration` del closure del `useEffect([], [])` de montaje. Si el slice cambia entre renders antes de que se ejecute el efecto, puede leer un valor stale.

### 4. `disabledOpenModal` invierte la orientación del DateRangePicker
- `disabledOpenModal=false` → layout `--row` (horizontal, para desktop).
- `disabledOpenModal=true` → layout columna (para modal mobile).
El parámetro controla tanto el prop del DS como la clase CSS. No cambiar su lógica sin verificar ambos efectos.

### 5. Condición de renderizado en `Home`
`DashboardTabComponent` solo se renderiza si:
1. Existe al menos un acuerdo con `state.name === AgreementStateEnum.ACTIVE`.
2. El usuario tiene permiso `"Home_dashboard.Ver/Consultar.Ver"`.

Si el componente no aparece, verificar primero estas condiciones antes de buscar bugs internos.

---

## Reglas — Lo que NUNCA debes hacer

- **No llames a `handleOnSubmit` sin que el usuario haya seleccionado un rango** — `disableSubmit()` existe para evitar requests con `dashboardDateRange` vacío. Si lo ignoras, se despacha con `DashboardHomeInitial`.
- **No modifiques `dashboardDate` directamente** — siempre va a través de `setDashboardDate`. El `useEffect([dashboardDate])` es el único lugar que dispara `getDashboardHomeReport`.
- **No cambies el orden de `earlierDate`/`laterDate` en `getDateByRange`** — `initialDate = laterDate` (más reciente), `finalDate = earlierDate` (más antigua). El backend espera este orden.
- **No uses `tabValue` como booleano** — hay condiciones en el código que usan `tabValue !== null` para detectar si hay tab seleccionado. Asegúrate de comparar siempre con `null`, no con `falsy` (porque `0` es un tab válido).
- **No elimines el `return () => dispatch(setKeepDashboardConfiguration(false))`** del `useEffect` de montaje — es la limpieza que garantiza que la próxima visita al Home no restaure un estado viejo.
- **No agregues tabs a `dashboardTabs` sin actualizar** `getDatesByTabvalue`, `getDinamicTitle` y los `ValuesInDatesToAdd` correspondientes en `constants.ts`.

---

## Cómo diagnosticar y corregir bugs

1. **Dashboard no muestra datos**: verificar que el componente se renderiza (`Home.tsx`, condición `dashboardComponent()`). Si no hay acuerdo activo o falta el permiso, el componente directamente no monta.

2. **Tab seleccionado no actualiza tarjetas**: revisar que `getDashboardHomeReport` se despachó (`useEffect([dashboardDate])`). Si `dashboardDate` no cambió (mismo objeto referencial), el efecto no se dispara. Verificar que `getDatesByTabvalue` retornó un nuevo objeto.

3. **DateRangePicker no aplica el filtro**: verificar que el usuario hizo clic en "Consultar" (`handleOnSubmit`). `handleChangeRangeDataPicker` solo guarda el rango temporalmente en `dashboardDateRange`, no en `dashboardDate`.

4. **"Restablecer filtros" no se habilita**: `disabledResetFilters()` retorna `true` cuando `disableSubmit() && tabValue === null`. Si el rango fue seleccionado pero no enviado, `disableSubmit()` = `false` → el botón estará habilitado.

5. **Estado no se restaura al volver al Home**: verificar que antes de navegar se despachó `setKeepDashboardConfiguration(true)`. Si no, el `useEffect` de montaje llama `setConciliationDashboardConfiguration(initialDashboardConfiguration)` y limpia el estado.

6. **Título dinámico incorrecto**: revisar `getDinamicTitle(translate, tabValue)` en `selectDocumentHelper.ts`. Si `tabValue` es `null`, retorna el texto de rango libre. Si es un número fuera de 0–4, retorna el mismo texto de rango libre (caso `default`).

7. **Modal mobile no cierra**: `showDateFilterModal` se controla con `setShowDateFilterModal`. `handleOnSubmit` lo cierra (`setShowDateFilterModal(false)`). El `onClose` del `Modal` MUI también llama `setShowDateFilterModal(false)`.

---

## Tests

No se encontraron archivos `__test__` directamente en el directorio del componente ni del hook. Si se agregan tests, seguir el patrón de `ReceivableTab`:

```
src/presentation/home/pages/components/dashboardTabComponent/__test__/DashboardTabComponent.test.tsx
src/presentation/home/pages/hooks/__test__/useDashboardTab.test.ts
```

Correr con:
```bash
pnpm test -- src/presentation/home/pages/components/dashboardTabComponent
```

---

## Historial de cambios

| Fecha | Bug / Motivo | Resumen |
|---|---|---|
| 2026-02-26 | Creación inicial del skill | Documentación completa del componente, hook y helpers asociados. |
