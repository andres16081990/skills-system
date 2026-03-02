# Skill: Voucher (página)

## Ubicación
```
src/presentation/paymentLink/pages/Voucher/
├── Voucher.tsx          ← componente principal (página)
├── Voucher.scss
└── const.ts             ← STATUS_TRANSACTION, STATUS_SHOW, DIMENSION enum
```

## Propósito
Página de resultado de transacción. Orquesta el contenido mostrado al usuario según el estado (`status`) de la transacción y el método de pago (`payment_method_type`). Inicia polling, obtiene info de transacción y voucher, y permite descarga del comprobante en PDF.

## No recibe props
Es una página; se monta vía React Router y obtiene params de la URL.

## URL params (`useParams`)

| Param | Uso |
|---|---|
| `transactionIdUrl` | ID de la transacción — dispara `getTransactionInfo` y `getVoucher` |
| `businessIdentifier` | Identifica el negocio — dispara `getCustomStylesByBusiness` |
| `paymentCode` | Código de pago para el botón de reintentar |

---

## Estado Redux leído

### PaymentLinkReducer (`stateSelectorPayMethodContext`)
- `transactionInfo` — `{ status, payment_method_type, payment_method, third_payment_redirect_url, amount_in_cents, currency }`
- `voucher` — datos completos del comprobante (ver estructura abajo)
- `paymentDetail` — `{ references, code }`
- `bankList` — lista de entidades financieras
- `processState` — `{ action, current }` — estado del proceso (loading/success)

### AgreementReducer (`stateSelectorAgreementContext`)
- `selectedAgreement` — `{ name }` — usado para el voucher de corresponsal

### CommonsReducer (`stateSelectorCommonContext`)
- `processState.current` — si es `"loading"` muestra `<CustomBackdrop>`

---

## Acciones Redux disparadas

| Acción / Thunk | Cuándo |
|---|---|
| `getCustomStylesByBusiness(businessIdentifier)` | Al montar, si hay `businessIdentifier` |
| `getTransactionInfo(transactionIdUrl)` | Al montar |
| `setPayerSessionId(id)` | Al montar, si existe en `secureLocalStorage` (luego lo elimina) |
| `getVoucher(transactionIdUrl)` | Cuando `status` cambia a no-PENDING / no-ERROR / no-vacío (con 1 s de delay). También aplica cuando `payment_method_type === CORREP` y `status === PENDING` |

---

## Lógica de renderizado por estado

### `getVoucherSummaryContent()` — contenido principal
```
status === APPROVED   → getApprovedComponent()
status === DECLINED   → VoucherError (si hay reference) | VoucherPending (si no)
status === VOIDED     → VoucherError
status === PENDING    → getPendingComponent()
status === ERROR      → VoucherTransactionError
default ("")          → VoucherPending
```

### `getApprovedComponent()`
```
payment_method_type === CORREP  → cbVoucher()
else, si voucher cargado (success + reference)  → VoucherSumary + ReturnButton (desktop)
else  → VoucherPending
```

### `getPendingComponent()`
```
payment_method_type === QR      → VoucherQr (hardcoded timer "14:45")
payment_method_type === CORREP  → cbVoucher()
else                            → VoucherPending
```

### `cbVoucher()` — flujo corresponsal (CORREP)
Renderiza: `CorrespondentReference` + `VoucherDownload` + `ReturnButton` (solo desktop `isLg`).

---

## Sub-componentes renderizados

| Componente | Condición |
|---|---|
| `VoucherSumary` | APPROVED + no-CORREP + voucher cargado |
| `VoucherTransaccion` | APPROVED / VOIDED (no CORREP, no PENDING, no ERROR, no DECLINED) + voucher cargado |
| `VoucherError` | DECLINED (con reference) o VOIDED |
| `VoucherPending` | PENDING (no QR/CORREP), DECLINED sin reference, APPROVED cargando, default |
| `VoucherQr` | PENDING + QR |
| `VoucherTransactionError` | ERROR |
| `CorrespondentReference` | CORREP (PENDING o APPROVED) |
| `VoucherDownload` | APPROVED + voucher cargado (mobile) + CORREP |
| `ReturnButton` | APPROVED (desktop: `id="return-button"`, `id="return-button-cb"`), mobile: `id="return-button-mobile"`, `id="return-button-cb-mobile"` |
| `RetryButton` | DECLINED o VOIDED + hay `paymentCodeToRedirect` |
| `PaidDocumentTable` | APPROVED + `voucher.documents_data.length > 0` + no loading + `isSm` |
| `LogoCustomer` | Siempre visible en mobile (`isSm`) |
| `Alert` (info) | Cuando `toastShowed === true` (popup de redirección a tercero) |
| `ModalIFrame` | Siempre (maneja el iframe 3DS fuera del contenedor principal) |
| `CustomBackdrop` | Si `processStateCommon.current === "loading"` (renderiza y retorna) |

---

## Lógica 3DS (WOMPI)

```ts
const three_ds_auth = payment_method?.extra?.three_ds_auth;
const showIframe =
  three_ds_auth?.current_step === WOMPI_3DS_STEPS.CHALLENGE &&
  three_ds_auth.current_step_status === WOMPI_3DS_STATUS.PENDING &&
  three_ds_auth.three_ds_method_data;
```
Cuando `showIframe === true`, el contenedor principal (`#sumary`) **no se renderiza**. Solo `<ModalIFrame />` queda visible.

---

## Polling

Gestión via `usePollingTransaction`:
- `isPollingEnabled` — activa/desactiva el polling
- `startPolling()` / `stopPolling()`
- Se activa en el `useEffect` que observa `isPollingEnabled`
- Se detiene en el cleanup del `useEffect` vacío (unmount)

---

## Descarga PDF

```ts
const ignoreElements = [
  "download-button", "download-button-cb", "download-button-mobile",
  "return-button", "return-button-cb", "return-button-mobile", "return-button-cb-mobile",
];

handleDownloadClick() → downloadCaptureToPdf(
  "sumary",             // id del contenedor capturado
  ignoreElements,       // IDs de botones excluidos del PDF
  "comprobante",        // nombre del archivo
  getDimensionDocumentPdf(isLg, isSm, isMini),
  voucher.documents_data?.length || 0,
);
```
El contenedor con `id="sumary"` es el `div.principal-container`.

---

## `mappingDataSumary()` — datos para `VoucherSumary`

| Campo | Fuente |
|---|---|
| `id` | `voucher.transaction_detail.reference` |
| `amount` | `total_amount + commission_amount_in_cents` (en centavos → `amountInCop`) |
| `currency` | `voucher.transaction_detail.currency` |
| `agreement` | `voucher.agreement_information?.name` |
| `references` | `paymentDetail.references` filtradas: `show_to_payer_detail && !is_entry_reference` |
| `payerDetail` | `voucher.payer_detail` |
| `handleDownloadClick` | función de descarga PDF |

---

## `mappingDataTransaction()` — datos para `VoucherTransaccion`

Construye un array `DropdownData[]` con los campos: status (mapeado via `STATUS_SHOW`), fecha, nequiId (si `external_identifier`), nombre del recaudador, entidad financiera (si `financial_institution_code`), `cust_id`, ticket, método de pago, últimos 4 dígitos de tarjeta (si TC y `last_four_credit_card`).

---

## `mappingDataCB()` — datos para `CorrespondentReference`

Toma `amount_in_cents`, `currency` de `transactionInfo` y `business_agreement_code`, `payment_intention_identifier` de `payment_method.extra`, y `commission_*` del voucher.

---

## `paymentCodeToRedirect` (useMemo)

```ts
// Prefiere el param de URL; si es nulo/vacío usa paymentDetail.code
paymentCode && paymentCode !== "null" ? paymentCode : paymentDetail.code
```

---

## Redirección a tercero

```ts
redirectPayment(third_payment_redirect_url)
// Solo si !businessIdentifier y payment_method_type no es QR/CORREP/TC
// Muestra Alert tipo info con usePopUpAlert; solo se muestra una vez (isRedirectedUrl flag)
```

---

## Constantes (`const.ts`)

```ts
STATUS_TRANSACTION.WOMPI = { APPROVED, PENDING, DECLINED, VOIDED, ERROR }

STATUS_SHOW: Record<string, string> = {
  "Aceptado": "Aprobada",
  "Rechazada": "Rechazada",
  "Rechazado": "Rechazada",
}
// Normaliza etiquetas del backend para mostrar en VoucherTransaccion

DIMENSION enum = { mobilexs: 0, mobile: 1, tablet: 2, desktop: 3 }
// Usado por getDimensionDocumentPdf
```

---

## CSS relevante (`Voucher.scss`)

| Clase | Descripción |
|---|---|
| `.principal-container` | Contenedor raíz capturado para PDF (id="sumary") |
| `.principal-container__alert` | Contenedor del Alert de info (redirección) |
| `.logo-customer` | Logo visible solo en mobile |
| `.voucher-container` | Contenedor del estado + modificadores por status |
| `.voucher-container-declined` | DECLINED o VOIDED |
| `.voucher-container-pending` | PENDING o status vacío |
| `.voucher-container-approved` | APPROVED |
| `.voucher-correp` | Flujo corresponsal |
| `.voucher-sumary__container` | Contenedor del resumen |
| `.voucher-sumary__container__pending` | Modificador pending |
| `.voucher-sumary__container__qr` | Modificador corresponsal/QR |
| `.voucher-sumary__transaction` | Contenedor de `VoucherTransaccion` |
| `.voucher-download-mobile` | Botón de descarga mobile |
| `.voucher-download-correp` | Modificador corresponsal para el botón mobile |
| `.btn-retry-mobile` | Contenedor del botón de reintento |

---

## Traducciones (`keyPrefix: "properties.payment_link.pages.voucher"`)

| Clave | Uso |
|---|---|
| `statusLabel` | Label de estado en la tabla de transacción |
| `dateLabel` | Label de fecha |
| `nequiId` | Label del identificador externo (Nequi) |
| `nameCollectorLabel` | Label nombre del recaudador |
| `financialEntity` | Label entidad financiera |
| `cust_id` | Label del cust_id |
| `ticketLabel` | Label del ticket |
| `methodLabel` | Label del método de pago |
| `cardNumberLabel` | Label de últimos 4 dígitos de tarjeta |
| `popUpAlert` | Mensaje del Alert de redirección a tercero |
| `titleDeclined` | Título para VoucherError en DECLINED |
| `messageDeclined` | Mensaje para VoucherError en DECLINED |
| `titleVoided` | Título para VoucherError en VOIDED |
| `messageVoided` | Mensaje para VoucherError en VOIDED |
| `download` | Título del botón de descarga mobile (no-CORREP) |
| `downloadCb` | Título del botón de descarga CORREP |

---

## Historial de cambios

| Fecha | Bug | Resumen |
|---|---|---|
| 2026-02-24 | BUG-003 | Fix en `mappingDataTransaction`: el label de `external_identifier` ahora varía según `payment_method_type` — DAVIPLATA/SUMAS_PAY/COMPRA_AHORA_PAGA_DESPUÉS usan `"tiketId"` ("Ticket ID"); NEQUI mantiene `"nequiId"` ("ID Nequi") |

---

## Reglas para modificar este componente

- Es una **página única** (no compartida); cambios afectan todos los flujos de resultado de pago.
- `getVoucherSummaryContent`, `getApprovedComponent`, `getPendingComponent` son la lógica central de branching — cualquier nuevo estado o método de pago debe agregarse aquí.
- El `useEffect` de `getVoucher` tiene un delay de `1000ms` (`DELAY_GET_VOUCHER_TIMER`) — no eliminarlo para evitar condiciones de carrera con el API.
- La condición de `showIframe` oculta **todo** el contenido del voucher; al tocar la lógica 3DS verificar que el iframe no quede atrapado.
- `ignoreElements` debe incluir los IDs de **todos** los botones que no deben aparecer en el PDF; al agregar nuevos botones con IDs, considerar si deben excluirse.
- `payerSessionId` se lee y elimina del `secureLocalStorage` al montar — es de un solo uso; no persistir ni leer dos veces.
- El `useEffect` de polling observa solo `isPollingEnabled`; el cleanup de unmount siempre llama `stopPolling()`.
- La condición de `APPROVED` en `getVoucherTransaction` excluye CORREP para no duplicar el resumen del corresponsal.
