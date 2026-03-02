# Skill: getFilteredDocuments

## Ubicación
`src/presentation/agreement/utils/getFilteredDocuments.ts`

## Exports
El archivo exporta dos funciones independientes:
1. `getFilteredDocuments` — flujo de convenios (step 1, selección de documentos)
2. `getFilteredDocumentsStep` — flujo de PaymentLink (step 2, tabla de pago)

---

## `getFilteredDocuments`

### Propósito
Transforma `DocumentByReferenceResponse` en una lista de `ReceivableLite[]` lista para ser renderizada en la tabla de documentos del flujo de convenios. También aplica filtros de estado y fecha.

### Firma
```ts
export const getFilteredDocuments = (
  value: string,                     // value_type_code del valor a mostrar (e.g. "VALTYPE_1")
  documents: DocumentByReferenceResponse,
  selectedReferenceCode?: string,    // reference_type_code de la referencia secundaria a mostrar
): ReceivableLite[]
```

### Dónde se usa

| Consumidor | Contexto | Args pasados |
|---|---|---|
| `AgreementReducer.ts` → `setFilteredDocuments` (reducer) | Cuando el usuario cambia el filtro de valor | `(valueCode, state.documents, state.selectedReference?.reference_type_code)` |
| `AgreementReducer.ts` → `getDocumentsByReference.fulfilled` | Docs no externos cargados | `(filterValues[0].value, state.documents, state.selectedReference?.reference_type_code)` |
| `AgreementReducer.ts` → `getExternalDocumentsByReferences.fulfilled` | Docs externos cargados | ⚠️ `(filterValues[0].value, state.documents, state.entryReferenceFlag, state.selectedReference?.reference_type_code)` — 3er arg es `boolean`, 4o ignorado (ver advertencia abajo) |

### Lógica de filtrado

**Receivables (`receivable_data_list`):**
- Excluye documentos con `state_name === PAID`
- No filtra por fecha (incluye vencidos)

**Instructions (`receivable_instruction_data_list`):**
- Excluye documentos con `state_name === PAID`
- **Además filtra por fecha:** solo incluye instrucciones cuyo `end_date` (al final del día) sea posterior al momento actual

### Construcción del ID — Regla crítica

```ts
const isExternalReceivableId = p.uuid ?? p.id;
id: isExternalReceivableId + "/" + DocumentReceivableTypeEnum.RECEIVABLE
```

| Tipo de convenio | `uuid` presente | ID generado |
|---|---|---|
| Externo | ✓ | `"{uuid}/RECEIVABLE"` o `"{uuid}/INSTRUCTION"` |
| No externo | ✗ | `"{numericId}/RECEIVABLE"` o `"{numericId}/INSTRUCTION"` |

**⚠️ Regla de oro:** Cualquier componente que construya IDs para hacer match con `filteredDocuments` **debe usar la misma lógica**: `(p.uuid ?? p.id)/${type}`. Si usa solo `p.id`, romperá la selección en convenios externos.

### Campos del `ReceivableLite` generado

| Campo | Fuente | Notas |
|---|---|---|
| `id` | `(p.uuid ?? p.id) + "/" + type` | Clave de selección en el DataGrid |
| `startDate` | `emition_date` (receivable) / `start_date` (instruction) | Formato `DATE_FORMAT_SHOW` |
| `endDate` | `due_date` (receivable) / `end_date` (instruction) | Formato `DATE_FORMAT_SHOW` |
| `reference` | `getSecondaryReference(p, selectedReferenceCode)?.value` | Valor de la referencia secundaria |
| `referenceName` | `getSecondaryReference(p, selectedReferenceCode)?.name` | Nombre del tipo de referencia |
| `state` | `p.state_name` | Como `DocumentStateKeyEnum` |
| `stateName` | `DocumentStateEnum[state_name]` | Texto legible del estado |
| `value` | `amountInCop(valueToPay?.value)` | Valor entero en centavos |
| `formattedValue` | `formatter.format(amountInCop(...))` | Valor formateado con separadores |
| `type` | `DocumentReceivableTypeEnum.RECEIVABLE / INSTRUCTION` | Tipo del documento |
| `isSaved` | `p.is_saved` | Si fue guardado por el usuario |
| `agreementId` | `p.agreement_id` | ID del convenio |

### Dependencias
| Util | Aporte |
|---|---|
| `getSecondaryReference` | Busca la referencia secundaria del documento según `selectedReferenceCode` |
| `amountInCop` | Convierte string/number a entero de centavos |
| `formatter` | Formatea valor numérico a moneda |
| `moment` | Formateo de fechas |

---

## `getFilteredDocumentsStep`

### Propósito
Transforma `DocumentsToPay[]` (respuesta del PaymentDetail) en `DocumentLite[]` para la tabla del segundo paso del flujo PaymentLink.

### Firma
```ts
export const getFilteredDocumentsStep = (
  documents: DocumentsToPay[],
  selectedReferenceCode?: string,
  agreement?: AgreementTransacction,
): DocumentLite[]
```

### Dónde se usa

| Consumidor | Contexto |
|---|---|
| `PaymentLinkReducer.ts` → `getPaymentDetail.fulfilled` | Cuando se carga el detalle de pago |

### Lógica especial
- Si `pending_value_ref_code` existe: filtra `orderedValues` al solo ese tipo de valor y copia `value` a `original_value`
- `id`: usa `p.pay_ref_id` (string) — **no usa** `uuid` porque viene de un modelo diferente (`PaymentDetailResponse`)
- `valuesOptions`: construye dropdown con todos los tipos de valor disponibles del documento

---

## ⚠️ Advertencia — Bug conocido en llamada desde `getExternalDocumentsByReferences.fulfilled`

En `AgreementReducer.ts` (línea ~349), el call para convenios externos pasa argumentos incorrectos:

```ts
// INCORRECTO (bug existente):
state.filteredDocuments = getFilteredDocuments(
  state.filterValues[0].value,
  state.documents,
  state.entryReferenceFlag,          // ← boolean en lugar de string | undefined
  state.selectedReference?.reference_type_code,  // ← 4o arg, ignorado por la función
);

// CORRECTO (como en los otros dos calls):
state.filteredDocuments = getFilteredDocuments(
  state.filterValues[0].value,
  state.documents,
  state.selectedReference?.reference_type_code,  // string | undefined
);
```

Consecuencia: para convenios externos, `selectedReferenceCode` recibe `true` o `false` (boolean) en lugar del código de referencia. Esto afecta los campos `reference` y `referenceName` del `ReceivableLite` (pueden mostrar `-` en lugar del valor real). El `id` del documento **no** se ve afectado por este bug.

---

## Reglas para modificar

- Si se cambia la construcción del `id` (la línea `isExternalReceivableId + "/" + type`), **hay que actualizar también** `useHeadersRowsTable.tsx` y `buildShadowDocumentsToLink` en `useStepperReferences.ts` para mantener consistencia
- `getFilteredDocuments` y `getFilteredDocumentsStep` son funciones independientes — no comparten lógica interna
- No hay tests unitarios para estas funciones

## Historial de cambios

| Fecha | Bug | Resumen |
|---|---|---|
| 2026-02-23 | BUG-002 | Identificado que `getFilteredDocuments` usa `p.uuid ?? p.id` para construir IDs. `useHeadersRowsTable` fue corregido para usar la misma lógica (`rec.uuid ?? rec.id`) y así evitar mismatch de IDs en convenios externos |
