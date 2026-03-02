# Skill: VoucherSumary (componente)

## Ubicación
```
src/presentation/paymentLink/components/Voucher/VoucherSumary/
├── VoucherSumary.tsx       ← componente principal
├── VoucherSumary.scss
└── __test__/
    └── VoucherSumary.test.tsx
```

## Propósito
Muestra el resumen visual del comprobante de pago aprobado. Incluye el valor pagado, la comisión (si aplica), el convenio, las referencias y los datos del pagador. También renderiza el botón de descarga PDF (desktop).

## Usado en
Solo en `Voucher.tsx` (página) — un único consumidor. Se puede modificar directamente.

---

## Props (`VoucherSumaryProps`)

| Prop | Tipo | Descripción |
|---|---|---|
| `id` | `string` | ID de referencia de la transacción |
| `amount` | `number` | Monto total en pesos (ya convertido desde centavos) |
| `currency` | `string` | Código de moneda, e.g. `"COP"` |
| `agreement` | `string?` | Nombre del convenio |
| `references` | `Reference[]` | Referencias filtradas del pago (ya filtradas en Voucher.tsx) |
| `payerDetail` | `PayerDetail` | `{ full_name, email, cellphone }` |
| `financialEntity` | `string?` | (Recibido pero no usado en el JSX actualmente) |
| `cus` | `string?` | (Recibido pero no usado en el JSX actualmente) |
| `handleDownloadClick` | `MouseEventHandler<HTMLDivElement>` | Callback de descarga PDF |

---

## Estado Redux leído internamente

Accede directamente al store vía `useTypedSelector<PaymentLinkState>`:

| Campo | Uso |
|---|---|
| `voucher.transaction_detail.commission_amount_in_cents` | Centavos de comisión (default `0`) |
| `voucher.transaction_detail.total_amount` | Monto base para mostrar en bloque de comisión |
| `voucher.transaction_detail.commission_name` | Nombre de la comisión (o fallback a `"commissionTitle"` i18n) |
| `voucher.entry_references` | Referencias adicionales combinadas con las props |
| `voucher.documents_data` | Si `length > 0`, reemplaza las referencias por el contador de documentos |

---

## Acciones Redux disparadas

| Acción | Cuándo |
|---|---|
| `setActiveStep(0)` | Al montar (`useEffect` vacío) |
| `setSelectedDocument(undefined)` | Al montar (`useEffect` vacío) |

Ambas acciones pertenecen a `AgreementReducer`. Se usan para limpiar el estado del stepper al llegar al voucher.

---

## Feature Flag

| Flag | Constante | Efecto |
|---|---|---|
| `frontPagadorCommissionField` | `FRONT_TRANSACTION_COMMISSION` | Si `true` y `commissionCents > 0`, muestra el bloque de comisión |

---

## Lógica interna

### `getVoucherReferences()`
```ts
[...references, ...(voucher.entry_references ?? [])]
```
Combina las referencias recibidas por props con las `entry_references` del voucher en store.

### `showMultiPaymentData()`
Cuando `voucher.documents_data.length > 0`, muestra el número de documentos pagados en lugar de la lista de referencias. Usa la clave i18n `"paidDocumentsText"`.

### Responsive
```ts
const { isSm } = useQuery();
// isSm === true → clase "voucher-summary"
// isSm === false → clase "voucher-summary-mobile"
```

---

## Estructura de renderizado

```
<>
  <div class="voucher-summary" | "voucher-summary-mobile">
    <VoucherHeaderMsg svgIcon=<SumarySucces/> />       ← ícono + título + mensaje
    <div.voucher-summary__value>                        ← monto formateado
    <div.invoice-details>
      <div.voucher-summary__agreement>                 ← nombre del convenio
        ├── Referencias o contador de documentos
      <div.voucher-summary__commission>  [si flag + comisión > 0]
        ├── monto base
        └── monto comisión
      <div.voucher-summary__payer>                     ← full_name, email, cellphone
  <div.voucher-download-desktop>
    <VoucherDownload id="download-button" />
</>
```

---

## CSS relevante (`VoucherSumary.scss`)

| Clase | Descripción |
|---|---|
| `.voucher-summary` | Desktop/tablet (isSm=true) |
| `.voucher-summary-mobile` | Mobile (isSm=false) |
| `.voucher-summary__value` | Contenedor del monto |
| `.voucher-summary__value__label` | Label "Valor pagado" |
| `.voucher-summary__value__val` | Monto formateado (font xlarge, semibold) |
| `.invoice-details` | Wrapper de convenio + comisión + pagador |
| `.voucher-summary__agreement` | Bloque del convenio y referencias |
| `.voucher-summary__commission` | Bloque de comisión (condicional) |
| `.voucher-summary__payer` | Bloque de datos del pagador |
| `.voucher-download-desktop` | Wrapper del botón de descarga (desktop) |

**Breakpoints:**
- `600px–1200px` (tablet): `.invoice-details` cambia a `flex` con `gap: 32px`; payer/commission pierden `margin-top`
- `< 600px` (mobile): `.invoice-details` pasa a `block`; `.voucher-summary-mobile` ocupa `100%`

---

## Traducciones (`keyPrefix: "properties.payment_link.component.voucherSumary"`)

| Clave | Uso |
|---|---|
| `titleSuccess` | Título del header de éxito |
| `messageSuccess` | Mensaje del header de éxito |
| `paymentValue` | Label del monto pagado |
| `agreementTitle` | Título de la sección convenio |
| `agreement` | Label "Convenio:" dentro de la sección |
| `paidDocumentsText` | Label cuando hay múltiples documentos pagados |
| `baseAmountTitle` | Label del monto base en bloque comisión |
| `commissionTitle` | Fallback del nombre de comisión |
| `payerTitle` | Título de la sección pagador |
| `cellphone` | Label del teléfono |
| `download` | Título del botón de descarga |

---

## Sub-componentes utilizados

| Componente | Descripción |
|---|---|
| `VoucherHeaderMsg` | Header con ícono SVG, título y mensaje de éxito |
| `SumarySucces` | SVG del ícono de éxito (verde) |
| `References` | Lista de referencias (del flujo agreement) |
| `VoucherDownload` | Botón de descarga PDF (desktop, `id="download-button"`) |

---

## Tests (`VoucherSumary.test.tsx`)

Todos los módulos externos están mockeados. Los casos cubiertos:

| Test | Qué verifica |
|---|---|
| `renders basic summary info and payer data` | Header, monto, convenio, pagador, botón descarga |
| `shows references and no commission block when flag is false` | Se renderiza `<References>` y no aparece bloque comisión |
| `shows commission info when flag is true and commission data exists` | Bloque comisión visible con nombre y montos formateados |
| `dispatches reset actions on mount` | `setActiveStep(0)` y `setSelectedDocument(undefined)` son despachados |

**Mocks necesarios para nuevos tests:**
- `@featbit/react-client-sdk` → `useFlags`
- `react-i18next` → `useTranslation` (devuelve la key como valor)
- `../../../../../state/Hooks` → `useTypedSelector`, `useAppDispatch`
- `../../../../state/PaymentLinkReducer` → `stateSelectorPayMethodContext`
- `../../../../../common/hooks/useMediaQuery` → `useQuery` → `{ isSm }`
- `../../../../../../utils/currencyFormatter` → `formatter.format`
- `../../../../../../utils/amountInCent` → `amountInCop`

---

## Reglas para modificar este componente

- Usado en **un solo lugar** (`Voucher.tsx`) → se puede modificar directamente.
- El bloque de comisión está condicionado por dos guardas: `commissionFlag === true` **y** `commissionCents > 0`. Ambas deben cumplirse.
- `getVoucherReferences()` siempre combina `references` (props) + `entry_references` (store). No usar solo uno.
- El `useEffect` de mount siempre despacha `setActiveStep(0)` y `setSelectedDocument(undefined)` — son limpieza del estado del stepper anterior.
- `VoucherDownload` dentro de `.voucher-download-desktop` usa `id="download-button"` — este ID está en la lista `ignoreElements` de `Voucher.tsx` para excluirlo del PDF. Si se renombra, actualizar también `Voucher.tsx`.
- `financialEntity` y `cus` están en las props pero **no se renderizan** en el JSX actual. Si se agregan al UI, verificar que el padre (`mappingDataSumary`) los esté enviando.
