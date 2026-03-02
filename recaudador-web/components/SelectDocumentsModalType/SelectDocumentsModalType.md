# SelectDocumentsModalType — Agent Skill

## Propósito

Componente de formulario que renderiza el **contenido interno del modal lateral de creación de documentos**. Permite al usuario seleccionar tres cosas en secuencia:

1. **El acuerdo** sobre el que se va a crear el documento (Dropdown filtrado por estado y permisos)
2. **El tipo de documento** a crear: Receivable o Instructions (Dropdown)
3. **La modalidad de creación**: Masiva o Manual (RadioButton horizontal)

Vive en:

```
src/presentation/home/pages/components/SelectDocumentsModalType/SelectDocumentsModalType.tsx
```

Es un componente **presentacional puro**: no despacha acciones Redux, no navega, no tiene estado local. Toda la lógica de control vive en su padre `SideModalCreateDocument`.

---

## Consumidores

Usado en **un único lugar**:

| Componente | Ruta |
|---|---|
| `SideModalCreateDocument` | `src/presentation/home/pages/components/SideModalCreateDocument/SideModalCreateDocument.tsx` |

> Por ser usado en un solo lugar, se puede modificar directamente. Aun así documentar qué se cambia y por qué.

---

## Props

| Prop | Tipo | Requerida | Descripción | De dónde viene |
|---|---|---|---|---|
| `agreements` | `Agreement[]` | ✅ | Lista completa de acuerdos del usuario. El componente los filtra internamente | Redux `AgreementState.agreements` vía padre |
| `agreementSelected` | `string` | ✅ | `agreement_code` del acuerdo actualmente seleccionado | `useState` en `SideModalCreateDocument` |
| `setAgreemenstValue` | `(value: string) => void` | ✅ | Callback para actualizar el acuerdo seleccionado (nombre con typo intencional: `Agreements` → `Agremenst`) | `setagreementSelected` del padre |
| `documentType` | `string` | ✅ | Tipo de documento seleccionado (`"Receivable"` o `"Instructions"`) | `useState` en `SideModalCreateDocument` |
| `setDocumentType` | `(type: string) => void` | ✅ | Callback para actualizar el tipo de documento | `setDocumentTypeSelected` del padre |
| `createDocumentType` | `string` | ✅ | Modalidad de creación seleccionada (`"Massive"` o `"Manual"`) | `useState` en `SideModalCreateDocument` |
| `setCreateDocumentType` | `(value: string) => void` | ✅ | Callback para actualizar la modalidad de creación | `setCreateDocumentType` del padre |

---

## Estructura visual

```
Stack (vertical)
├── div.selected-document__search
│   ├── BodyText "Acuerdo"
│   └── Dropdown → agreements (filtrados)
│
├── div.selected-document__type
│   ├── BodyText "Tipo de documento"
│   └── Dropdown → [Receivable, Instructions]
│
└── div.selected-document__type-create
    ├── BodyText "Tipo de creación"
    └── div.selected-document__type-create__radio
        └── RadioButton horizontalMode → [Masiva, Manual]
```

---

## Lógica interna

### Filtrado de acuerdos (`setAgreementOptions`)

Los acuerdos se filtran en dos pasos:

**Paso 1 — por estado y `external_api_id`** (aplica a todos los roles):
```ts
agreements.filter(a =>
  (a.state.name === AgreementStateEnum.READY_TO_USE ||
    a.state.name === AgreementStateEnum.ACTIVE) &&
  a.external_api_id !== 1
)
```

> Los acuerdos con `external_api_id === 1` quedan **siempre excluidos**, independientemente del estado y del rol.

**Paso 2 — por permisos** (solo para roles no-admin):
```ts
checkAdminRole(permissionsByUser.name)
  ? filteredAgreements          // Admin/Viewer: ve todos los activos
  : filteredAgreements.filter(a => a.has_permission_create)  // Otros roles: solo donde tiene permiso
```

`checkAdminRole` retorna `true` si el rol es `ADMIN_ROLE` o `VIEWER_ROLE` (constantes del proyecto).

El resultado se mapea a `{ label: name, value: agreement_code }` para el Dropdown.

### Opciones de tipo de documento (`optionsDocumentTypeOptions`)

Función del helper que mapea los valores del enum a labels traducidos:

| `value` | `label` (i18n key) |
|---|---|
| `DocumentTypesEnum.RECEIVABLE` = `"Receivable"` | `modalContent.radioButtonTextReceivable` |
| `DocumentTypesEnum.INSTRUCTIONS` = `"Instructions"` | `modalContent.radioButtonTextInstructions` |

### Opciones de tipo de creación (`getTypeCreateDocumentOptions`)

| `value` | `label` (i18n key) |
|---|---|
| `TypeCreateDocumentEnum.MASSIVE` = `"Massive"` | `modalContent.massive` |
| `TypeCreateDocumentEnum.MANUAL` = `"Manual"` | `modalContent.manual` |

---

## Redux — Slices que consume

| Slice | Selector | Campos que lee |
|---|---|---|
| `EntitlementState` | `stateSelectorEntitlement` | `permissionsByUser.name` (para `checkAdminRole`) |

> El componente **no** lee `AgreementState` directamente. Los acuerdos llegan como prop desde `SideModalCreateDocument`, que sí los lee de Redux.

---

## Relación con el padre `SideModalCreateDocument`

El padre orquesta toda la lógica. Este componente es solo la vista del formulario:

| Responsabilidad | Quién la tiene |
|---|---|
| Cargar acuerdos (`getAgreements`) | `SideModalCreateDocument` |
| Despachar `setSelectedAgreement` + `getAgreementDetailByCode` | `SideModalCreateDocument` |
| Navegar a formulario manual | `SideModalCreateDocument` |
| Disparar carga masiva (`setShowComponentLoader`, `setMassiveLoadType`) | `SideModalCreateDocument` |
| Controlar si el botón "Confirmar" está habilitado | `SideModalCreateDocument` (valida `!documentTypeSelected \|\| !agreementSelected \|\| !createDocumentType`) |
| Filtrar y renderizar los dropdowns | `SelectDocumentsModalType` ← este componente |

---

## Flujo completo de uso

```
Usuario abre SideModalCreateDocument
  └─ padre carga acuerdos: dispatch(getAgreements(...))
  └─ pasa agreements={agreements} como prop

Usuario selecciona acuerdo (Dropdown 1)
  └─ handlerAgrementValue → setAgreemenstValue(event.target.value)
  └─ padre recibe el value (agreement_code)
  └─ padre despacha: setSelectedAgreement + getAgreementDetailByCode

Usuario selecciona tipo de documento (Dropdown 2)
  └─ handleRadioButtonChange → setDocumentType(event.target.value)
  └─ padre actualiza documentTypeSelected

Usuario selecciona modalidad (RadioButton)
  └─ handleTypeCreateDocumentChange → setCreateDocumentType(value)
  └─ padre actualiza createDocumentType

Usuario confirma (botón del SideModal en el padre)
  └─ handlerSelectDocumentsModalConfirm() del padre
       ├─ MASSIVE + pendingLoad → muestra modal de advertencia
       ├─ INSTRUCTIONS + MANUAL → navega a INSTRUCTIONS_CREATE
       ├─ INSTRUCTIONS + MASSIVE → dispatch(setShowComponentLoader + setMassiveLoadType(INSTRUCTION))
       ├─ RECEIVABLE + MANUAL → navega a RECEIVABLE_CREATE
       └─ RECEIVABLE + MASSIVE → dispatch(setShowComponentLoader + setMassiveLoadType(RECEIVABLE))
```

---

## i18n — Claves utilizadas

Todas bajo el keyPrefix `"properties.home.components"`:

| Clave | Uso |
|---|---|
| `modalContent.agreement` | Label del dropdown de acuerdo |
| `modalContent.dropDownPlaceHolder` | Placeholder del dropdown de acuerdo |
| `modalContent.modalContentRadioButtonText` | Label del dropdown de tipo de documento |
| `modalContent.documentType` | Placeholder del dropdown de tipo de documento |
| `modalContent.typeCreatePaymentDocument` | Label del radio button de modalidad |
| `modalContent.radioButtonTextReceivable` | Opción Receivable |
| `modalContent.radioButtonTextInstructions` | Opción Instructions |
| `modalContent.massive` | Opción Masiva |
| `modalContent.manual` | Opción Manual |

---

## Dependencias críticas

| Dependencia | Propósito |
|---|---|
| `optionsDocumentTypeOptions(translate)` | Genera opciones traducidas del Dropdown de tipo de documento |
| `getTypeCreateDocumentOptions(translate)` | Genera opciones traducidas del RadioButton de modalidad |
| `checkAdminRole(permissionsByUser.name)` | Decide si mostrar solo acuerdos con permiso `has_permission_create` |
| `AgreementStateEnum.READY_TO_USE` / `ACTIVE` | Estados válidos para que un acuerdo aparezca en el dropdown |
| `Agreement.has_permission_create` | Flag por acuerdo que controla visibilidad para roles no-admin |

---

## Puntos frágiles

### 1. Typo en el nombre de la prop `setAgreemenstValue`

```ts
setAgreemenstValue: (value: string) => void;  // ← "Agremenst" en lugar de "Agreement"
```

El typo viene del padre y está en la interfaz. No corregirlo sin actualizar también el padre y la interfaz — están acoplados.

### 2. `setAgreementOptions` se recalcula en cada render

No está envuelta en `useMemo`. Si `agreements` o `permissionsByUser` cambian frecuentemente, produce un nuevo array en cada render y puede causar re-renders del Dropdown.

### 3. No hay estado de "selección vacía" explícita en el Dropdown de acuerdos

Si `filterAgreementOptions` resulta vacío (no hay acuerdos READY_TO_USE ni ACTIVE), el Dropdown muestra solo el placeholder pero no hay mensaje de "sin acuerdos disponibles". El usuario puede quedar atascado sin feedback.

### 4. El filtro por `has_permission_create` no aplica para Admin/Viewer

`checkAdminRole` retorna `true` tanto para `ADMIN_ROLE` como para `VIEWER_ROLE`. Esto significa que un `VIEWER_ROLE` puede ver **todos** los acuerdos activos aunque no tenga permiso real de creación. La validación de permiso real ocurre en el backend, pero visualmente puede ser confuso.

---

## Reglas — Lo que NUNCA debes hacer

- **No muevas lógica de navegación o dispatch a este componente** — es presentacional por diseño. Si necesitas lógica nueva, ponla en `SideModalCreateDocument` o en un hook nuevo.
- **No cambies los valores de `DocumentTypesEnum` o `TypeCreateDocumentEnum`** sin actualizar también el `switch` en `handlerSelectDocumentsModalConfirm` del padre — son los mismos strings que se evalúan para decidir la navegación.
- **No corrijas el typo `setAgreemenstValue` a secas** — está en la interfaz, en el componente y en el padre. Los tres deben cambiar juntos o se rompe la compilación.
- **No agregues lógica de filtrado de acuerdos en el padre** para evitar duplicar con `filterAgreementOptions` interno — el filtro por estado vive aquí, el filtro por rol también. Cualquier nuevo criterio de filtrado va aquí.
- **No uses `tabIndex` o valor numérico** para identificar qué tipo de documento está seleccionado — usa siempre los valores del enum (`DocumentTypesEnum.RECEIVABLE`, `DocumentTypesEnum.INSTRUCTIONS`).

---

## Tests

No hay archivo de test específico para este componente.

```
src/presentation/home/pages/components/SelectDocumentsModalType/  ← sin __test__/
```

Al crear tests, considerar:
- Renderizar con rol admin vs rol no-admin y verificar cuántos acuerdos aparecen en el Dropdown
- Verificar que acuerdos con estado distinto a `READY_TO_USE` / `ACTIVE` no aparecen
- Verificar que `has_permission_create = false` oculta el acuerdo para roles no-admin
- Verificar que los tres callbacks se llaman con los valores correctos al interactuar con cada control

---

## Historial de cambios

| Fecha | Motivo | Resumen |
|---|---|---|
| 2026-02-27 | Creación del skill | Documentación inicial del componente a partir de lectura directa del código. |
| 2026-02-27 | BUG-007 | Se agregó `&& agreement.external_api_id !== 1` en `filterAgreementOptions` para excluir acuerdos de integración externa del Dropdown. |
