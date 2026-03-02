# Skill: TermsConditions

## Ubicación
`src/presentation/paymentLink/components/Main/TermsConditions/TermsConditions.tsx`

## Estilos
`src/presentation/paymentLink/components/Main/TermsConditions/TermsConditions.scss`

## Propósito
Renderiza la sección de aceptación de términos y condiciones dentro del flujo de pago de PaymentLink. Muestra un checkbox junto a un texto que contiene enlaces para descargar:
- El PDF de **Términos y Condiciones** de THREXIO
- El PDF de **Política de Privacidad** de Threxio

## Props

| Prop             | Tipo                                          | Requerida | Descripción                                       |
|------------------|-----------------------------------------------|-----------|---------------------------------------------------|
| `termsChecked`   | `boolean`                                     | Sí        | Estado actual del checkbox                        |
| `setTermsChecked`| `React.Dispatch<React.SetStateAction<boolean>>`| Sí        | Setter del estado del checkbox desde el padre     |

## Dependencias internas

- **`useCustomStyles`** (`src/utils/useCustomStyles.ts`)
  - `getCheckStyle()` → aplica `accent_color` al checkbox cuando está marcado
  - `getBodyTextStyle()` → aplica `accent_color` a los textos de enlace (términos y políticas)
- **`useTranslation`** con `keyPrefix: "properties.payment_link.component.formPayment"`
  - `terms.accept` → "Acepto haber leído los"
  - `terms.termsConditions` → "términos y condiciones"
  - `terms.and` → "y"
  - `terms.politic` → "políticas de privacidad"
  - `terms.pay` → "para hacer este pago"

## Dependencias externas

- `@recaudify/recaudify-design-systems`: `Checkbox`, `BodyText`
- Assets PDF estáticos importados directamente:
  - `src/common/assets/templates/Términos y Condiciones de THREXIO S.pdf`
  - `src/common/assets/templates/Política de Privacidad Threxio.pdf`

## Comportamiento

- Al hacer clic en el checkbox, invierte el valor de `termsChecked` vía `setTermsChecked(!termsChecked)`
- Los enlaces descargan los PDFs directamente (atributo `download` en la etiqueta `<a>`)
- El texto del enlace adopta el color de la marca (`accent_color`) a través de `getBodyTextStyle()`
- El checkbox adopta el color de la marca cuando está marcado, vía `getCheckStyle()`
- Los `<a>` no tienen subrayado (`.form-payment-data a { text-decoration: none }`)

## Dónde se usa

- **Un único consumidor:** `FormPayment.tsx`
  (`src/presentation/paymentLink/components/Main/FormPayment/FormPayment.tsx`)
- Se renderiza solo cuando `!payerPaymentDocumentSelected` y el formulario de pago está visible

## Clases CSS relevantes

| Clase                                | Descripción                                              |
|--------------------------------------|----------------------------------------------------------|
| `.form-payment-data__terms`          | Contenedor flex centrado con gap 4px y margin 24px arriba/abajo |
| `.form-payment-data__terms__label`   | Texto inline con `font-size: small`                      |
| Responsive `≤600px`                  | Cambia margin del contenedor a `0px 0px 0px 6px`        |

## Tests
No existen tests unitarios para este componente. Si se modifica, se recomienda crear tests que validen:
- Render del checkbox con el estado inicial
- Toggle del checkbox al hacer clic
- Presencia de los dos enlaces de descarga
- Texto traducido correctamente

## Reglas para modificar este componente

- Es usado en **un solo lugar** (`FormPayment.tsx`) → se puede modificar directamente
- Si se necesita una variante con comportamiento diferente, crear un componente nuevo independiente
- No modificar `useCustomStyles` directamente; cualquier nuevo estilo puntual debe hacerse con `sx` local
- Los PDFs son assets estáticos importados; si cambia la ruta o el nombre del archivo, actualizar los imports
