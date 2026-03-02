# DashboardCardsComponent — Agent Skill

## Propósito

Grid de tarjetas de resumen financiero del dashboard de `Home`. Muestra 4 métricas del período seleccionado en `DashboardTabComponent` y permite navegar a las secciones de conciliación o documentos de cobro al hacer clic. La tarjeta de "Total Recaudado" es especial: al clicarse expande un sub-panel (`CardsExpanded`) con el desglose por canal.

Vive en:

```
src/presentation/home/pages/components/dashboardCardsComponent/DashboardCardsComponent.tsx
src/presentation/home/pages/components/dashboardCardsComponent/CardsExpanded/CardsExpanded.tsx
```

Toda la lógica de estado y navegación vive en el hook compartido:

```
src/presentation/home/pages/hooks/useDashboardCards.ts
```

---

## Comunicación con el resto de la app

### No recibe props
El componente exportado `DashboardCardsComponent` no acepta props. A pesar de que el archivo define `DashboardCardsComponentInterface { translate: TFunction }`, **el componente no usa esa interfaz** — es código muerto. Todo lo obtiene del hook.

### Redux — slice consumido (`ConciliationState`)

| Campo | Tipo | Uso |
|---|---|---|
| `dashboardConciliationData` | `DashboardResponse` | Datos crudos que se transforman en `dashboardCardData` |
| `dashboardConciliationReady` | `boolean` | Guard para que `mapDashboardResponse` solo ejecute cuando hay datos válidos |
| `processState` | `{ current, action }` | Controla cuándo actualizar las tarjetas (success o error + acción GET_DASHBOARD) |
| `dashboardMetaData` | `DashboardHomeRequest` | Se incluye en el state de navegación al hacer clic en una tarjeta |
| `dashboardConfiguration` | `{ tabValue, metaData }` | Usado en `handleCardOnClick` para persistir la config antes de navegar |

### Acciones despachadas

| Acción | Cuándo |
|---|---|
| `setConciliationIsRedirectFromDashboard(true)` | Al clicar la tarjeta `TOTAL_RAISED_IN_CENTS` o al redirigir desde `CardsExpanded` |
| `setConciliationDashboardConfiguration({...metaData: dashboardMetaData})` | En cada `handleCardOnClick` (guarda el filtro activo antes de salir) |
| `setPaymenDocumentIsRedirectFromDashboard(true)` | Al clicar cualquier tarjeta que NO sea `TOTAL_RAISED_IN_CENTS` |
| `setDashboardExpandedCard(index)` | Al clicar una tarjeta de `CardsExpanded` (0 = Conciliación, 1 = Pagos externos) |
| `setRedirectToConciliation()` | Al clicar una tarjeta de `CardsExpanded` cuando hay tab activo (`tabValue !== null`) |

### Navegaciones producidas

| Destino | Condición |
|---|---|
| `RoutesPath.PAYMENT_DOCUMENT` | Cualquier tarjeta salvo `TOTAL_RAISED_IN_CENTS`. Lleva `state: { metaData, source: "dashboard", cardSelected }` |
| `RoutesPath.CONCILIATION` | Al clicar cualquier tarjeta de `CardsExpanded` |

### Dónde vive en el árbol

```
Home
└── dashboardComponent() [acuerdo activo + permiso "Home_dashboard.Ver/Consultar.Ver"]
    ├── DashboardTabComponent
    └── DashboardCardsComponent  ← aquí
        └── CardsExpanded  [solo si isRelevantCard && isMainCardsExpanded]
```

---

## Estructura visual

```
[Grid 4 columnas en lg / 3 en md / 2 en xs]
  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
  │  Total Docs     │  │  Total Recaudado │  │  Pendiente      │  │  Vencido        │
  │  (renderRedirect│  │  (isRelevantCard)│  │  (renderRedirect│  │  (renderRedirect │
  │   si permiso)   │  │                 │  │   si permiso)   │  │   si permiso)   │
  └─────────────────┘  └────────┬────────┘  └─────────────────┘  └─────────────────┘
                                │  [isMainCardsExpanded]
                         ┌──────┴──────┐
                         │CardsExpanded│
                         │ [card-0]    │  ← Conciliación (total_raised_in_cents)
                         │ [card-1]    │  ← Pagos externos (total_external_raised_in_cents)
                         └─────────────┘
```

`renderRedirect` (flecha de navegación en la tarjeta) solo se muestra si:
- `isMd === true` (pantalla ≥ md breakpoint)
- El usuario tiene permiso `"Conciliación_-_Histórico.Ver/Consultar.Ver"`

La tarjeta de "Total Recaudado" tiene `renderRedirect` **siempre en `true`** dentro de `CardsExpanded` (no respeta permisos).

---

## Tarjetas del dashboard

Definidas en `mapDashboardCardValues` (`selectDocumentHelper.ts`):

| Índice | `cardSelected` | Descripción | `isRelevantCard` |
|---|---|---|---|
| 0 | `TOTAL_PAYMENT_DOCUMENTS_IN_CENTS` | Total docs de cobro | `false` |
| 1 | `TOTAL_RAISED_IN_CENTS` | Total Recaudado (suma canal + externo) | `true` |
| 2 | `PENDING_COLLECTION_IN_CENTS` | Pendiente de cobro | `false` |
| 3 | `OVERDUE_IN_CENTS` | Vencido | `false` |

El valor de la tarjeta "Total Recaudado" es:
```ts
total_raised_in_cents + total_external_raised_in_cents
```

### Tarjetas expandidas (`CardsExpanded`)

| Índice | `ExpandedCardIndex` | Dato fuente |
|---|---|---|
| 0 | `CONCILIATION` | `total_raised_in_cents` |
| 1 | `EXTERNAL_PAYMENTS_CONCILIATION` | `total_external_raised_in_cents` |

---

## Flujo de datos

### Actualización de tarjetas
1. `DashboardTabComponent` (via `useDashboardTab`) despacha `getDashboardHomeReport(request)`.
2. El thunk resuelve y actualiza `ConciliationState`: `processState.action = GET_DASHBOARD_HOME_CONCILIATION`, `dashboardConciliationData`, `dashboardConciliationReady = true`.
3. `useDashboardCards` reacciona en `useEffect([processState, dashboardConciliationData])`: si `processState.current === "success" || "error"` **y** `processState.action === GET_DASHBOARD_HOME_CONCILIATION` **y** `dashboardConciliationReady === true` → llama `mapDashboardResponse()`.
4. `mapDashboardResponse()` transforma los datos con `mapDashboardCardValues` y actualiza `dashboardCardData` (estado local).
5. El componente re-renderiza con los nuevos valores.

### Clic en tarjeta (no-TOTAL_RAISED)
1. `handleCardOnClick(cardSelected)` → navega a `PAYMENT_DOCUMENT` con `state: { metaData: dashboardMetaData, source: "dashboard", cardSelected }`.
2. Despacha `setPaymenDocumentIsRedirectFromDashboard(true)`.
3. Despacha `setConciliationDashboardConfiguration({...dashboardConfiguration, metaData: dashboardMetaData})` para preservar el filtro activo.

### Clic en tarjeta TOTAL_RAISED_IN_CENTS
1. `handleCardOnClick(TOTAL_RAISED_IN_CENTS)` → `handlerManinCard()` → toggle de `isMainCardsExpanded`.
2. Despacha `setConciliationIsRedirectFromDashboard(true)`.
3. Despacha `setConciliationDashboardConfiguration({...metaData})`.
4. Si `isMainCardsExpanded` pasa a `true` → se renderiza `CardsExpanded` debajo de la tarjeta.

### Clic en tarjeta expandida (`CardsExpanded`)
1. `handlerOnRedirectCards(index)` → despacha `setDashboardExpandedCard(index)` + `setConciliationIsRedirectFromDashboard(true)`.
2. Si `dashboardConfiguration.tabValue === null` (filtro por rango libre) → navega directamente a `CONCILIATION`.
3. Si hay tab activo → despacha `setRedirectToConciliation()` y navega a `CONCILIATION`.

### Colapso automático en mobile
`useEffect([isMd])` → si `isMd` pasa a `false` (pantalla pequeña) → `setisMainCardsExpanded(false)`. `CardsExpanded` se oculta automáticamente.

---

## Hooks consumidos

### `useDashboardCards()`

| Retorna | Tipo | Uso |
|---|---|---|
| `dashboardCardData` | `CardMapingComponent[]` | Array de 4 tarjetas listo para renderizar |
| `isMainCardsExpanded` | `boolean` | Controla si `CardsExpanded` está visible |
| `handleCardOnClick` | `(cardSelected: CardSelected) => void` | Handler para el clic en tarjetas principales |
| `getExpandCardValue` | `(index: number) => string` | Valor formateado de tarjeta expandida |
| `getExpandKey` | `(index: number) => string` | Clave única para el map de `CardsExpanded` |
| `handlerOnRedirectCards` | `(index: number) => void` | Navegación desde tarjeta expandida |
| `getTooltipText` | `(index: number) => string` | Tooltip de tarjeta expandida |
| `dashboardConciliationData` | `DashboardResponse` | Datos crudos (expuesto pero no usado en el componente padre) |

### `useQuery()`

| Retorna | Uso |
|---|---|
| `isMd` | Oculta `CardsExpanded` en mobile; controla `renderRedirect` |

### `useShowElementByPermission()`

| Método | Uso |
|---|---|
| `checkUserHasPermission(["Conciliación_-_Histórico.Ver/Consultar.Ver"])` | Controla si las tarjetas muestran la flecha de redirección |

---

## Puntos frágiles

### 1. Interfaz `DashboardCardsComponentInterface` sin uso
```ts
export interface DashboardCardsComponentInterface {
  translate: TFunction;
}
// El componente NO usa esta interfaz
export const DashboardCardsComponent = () => { ... }
```
Es código muerto. Si se refactoriza el componente y alguien intenta pasarle `translate` como prop, no tendrá efecto.

### 2. `getTooltipText` tiene lógica idéntica en ambas ramas
```ts
const getTooltipText = (index: number): string => {
  if (ExpandedCardIndex.CONCILIATION === index) {
    return translate(`dashboard.tooltipExpandedCards${index}`);
  }
  return translate(`dashboard.tooltipExpandedCards${index}`); // mismo resultado
};
```
La condicional no produce ninguna diferencia. Ambas ramas retornan el mismo string. Si se necesita lógica diferente por índice, esta función debe corregirse.

### 3. `dashboardConciliationReady` como guard adicional
El `useEffect` requiere que las tres condiciones sean true simultáneamente. Si el thunk llega en `"error"` pero `dashboardConciliationReady === false` (primer intento fallido antes de tener datos), las tarjetas no se actualizan. El usuario podría ver valores vacíos o del estado inicial.

### 4. `isMainCardsExpanded` es estado local — no persiste
Si el usuario colapsa/expande `CardsExpanded` y luego cambia el período (tab), el expand se mantiene. Solo se fuerza a `false` cuando `isMd` cambia a mobile. Podría producir comportamientos visuales inesperados al cambiar tabs con el panel expandido.

### 5. `CardsExpanded` usa `position: absolute`
```scss
.card-expand-container {
  position: absolute;
  right: 0px;
  height: 100%;
  width: 94%;
  z-index: 1;
}
```
El panel flotante se posiciona relativo al Grid item padre (que tiene `position: relative` via la clase `__relevant-card`). Si cambia el layout del Grid, el panel puede desalinearse visualmente.

### 6. `renderRedirect` en `CardsExpanded` no respeta permisos
```tsx
// CardsExpanded.tsx
<DashboardCard
  renderRedirect  // ← siempre true, sin checkUserHasPermission
  ...
/>
```
A diferencia de las tarjetas principales, las expandidas siempre muestran la flecha de navegación, independientemente del permiso `"Conciliación_-_Histórico.Ver/Consultar.Ver"`.

---

## Reglas — Lo que NUNCA debes hacer

- **No uses `dashboardConciliationData` directamente para calcular valores sin pasar por `mapDashboardCardValues`** — la transformación incluye lógica de formato de moneda y suma de canales.
- **No toggle `isMainCardsExpanded` fuera de `handlerManinCard`** — es el único lugar que lo controla. Si agregas otra forma de controlarlo sin pasar por el hook, desincronizarás el estado.
- **No navegues a `CONCILIATION` directamente** sin despachar primero `setConciliationIsRedirectFromDashboard(true)` y `setDashboardExpandedCard(index)` — la página de conciliación depende de estos flags para saber de dónde viene el usuario y qué tab expandido mostrar.
- **No navegues a `PAYMENT_DOCUMENT` sin el state** `{ metaData, source, cardSelected }` — la página de documentos de cobro usa `ComponentSource.DASHBOARD` para aplicar filtros pre-seleccionados.
- **No cambies el orden del array retornado por `mapDashboardCardValues`** — el índice 1 (posición fija) es el que tiene `isRelevantCard: true` y activa `CardsExpanded`. Cambiar el orden rompe el expand.
- **No agregues tarjetas a `mapDashboardCardValues` sin verificar** que el nuevo `CardSelected` está manejado en `handleCardOnClick` y que no requiere un comportamiento especial como el de `TOTAL_RAISED_IN_CENTS`.

---

## Cómo diagnosticar y corregir bugs

1. **Tarjetas con valores vacíos o cero**: verificar que `dashboardConciliationReady === true` en el slice. Si `getDashboardHomeReport` falla, el `useEffect` no actualiza las tarjetas. Revisar `processState.current` y `processState.action` con Redux DevTools.

2. **CardsExpanded no aparece al clicar "Total Recaudado"**: confirmar que `isMainCardsExpanded` cambió a `true` (estado local del hook). Verificar que `isRelevantCard === true` en la tarjeta — si `mapDashboardCardValues` cambió el índice 1, puede haberse perdido.

3. **Flecha de redirección no aparece en tarjetas**: verificar `isMd` (breakpoint ≥ md) y que el usuario tiene permiso `"Conciliación_-_Histórico.Ver/Consultar.Ver"`.

4. **Navegación a documentos no pre-filtra**: confirmar que el `state` de navegación llega correcto a `PAYMENT_DOCUMENT`. Verificar `dashboardMetaData` en el slice — si `DashboardTabComponent` no guardó las fechas antes del clic, `dashboardMetaData` puede estar vacío (`DashboardHomeInitial`).

5. **CardsExpanded persiste en mobile**: verificar que `useEffect([isMd])` está activo. Si `isMd` no cambia (resize sin re-render), el panel puede quedar visible. Forzar un cambio de tab para limpiar el estado.

6. **Datos no se actualizan al cambiar tab**: el ciclo es `DashboardTabComponent → getDashboardHomeReport → ConciliationState → useDashboardCards → mapDashboardResponse`. Si alguno de estos pasos falla silenciosamente, las tarjetas muestran datos del período anterior. Revisar el thunk `getDashboardHomeReport` en `ConciliationThunk`.

7. **Antes de modificar `mapDashboardCardValues`**: es usada por `useDashboardCards` y se importa desde `selectDocumentHelper.ts`. Verificar cuántos otros archivos la importan antes de modificarla. Si hay más de uno, crear versión independiente.

---

## Tests

No se encontraron archivos `__test__` para este componente ni para `useDashboardCards`. Si se agregan, la ruta recomendada es:

```
src/presentation/home/pages/components/dashboardCardsComponent/__test__/DashboardCardsComponent.test.tsx
src/presentation/home/pages/hooks/__test__/useDashboardCards.test.ts
```

Correr con:
```bash
pnpm test -- src/presentation/home/pages/components/dashboardCardsComponent
pnpm test -- src/presentation/home/pages/hooks/useDashboardCards
```

---

## Historial de cambios

| Fecha | Bug / Motivo | Resumen |
|---|---|---|
| 2026-02-26 | Creación inicial del skill | Documentación completa del componente, `CardsExpanded`, `useDashboardCards` y helpers asociados. |
