# Skill: FormPersonalData

## Ubicación

`src/presentation/paymentLink/components/Main/FormPayment/FormPersonalData/FormPersonalData.tsx`

---

## Propósito

Formulario de datos personales del pagador dentro del flujo de payment link. Renderiza campos de nombre, email y teléfono adaptando su estructura según el método de pago activo.

---

## Props

| Prop | Tipo | Descripción |
|------|------|-------------|
| `setPersonalDataInvalid` | `React.Dispatch<React.SetStateAction<boolean>>` | Callback que notifica al padre si el formulario es inválido |
| `paymentMethodId` | `number?` | ID del método de pago seleccionado (usado para condicionar el campo de teléfono) |

---

## Store (Redux)

Consume del store `paymentLink` vía `stateSelectorPayMethodContext`:

- `transactionCourse.business_detail.id` — ID del negocio, determina qué campos y variantes se muestran
- `paymentMethodsNameById` — mapa de nombre de método de pago → array de IDs de negocio asociados

Claves relevantes de `paymentMethodsNameById`:
- `DAVIPLATA` — array de IDs donde aplica flujo Daviplata
- `SUMAS_PAY` — array de IDs donde aplica flujo Sumas Pay
- `NEQUI` — array de IDs donde aplica flujo Nequi (oculta teléfono si es Nequi)
- `COMPRA_AHORA_PAGA_DESPUÉS` — array de IDs donde se separa nombre/apellido

---

## Condicionales de render

### Disclaimer (InfoIcon + texto)

Aparece cuando `business_detail.id` **está incluido** en `DAVIPLATA` **o** `SUMAS_PAY`.

```tsx
{(isPaymentMethod(
  transactionCourse.business_detail.id,
  paymentMethodsNameById.DAVIPLATA ?? [],
) ||
  isPaymentMethod(
    transactionCourse.business_detail.id,
    paymentMethodsNameById.SUMAS_PAY ?? [],
  )) && (
  <div className="form-personal-data__disclaimer">...</div>
)}
```

> **Regla clave:** usar `isPaymentMethod` (afirmación) para mostrar cuando el id pertenece al método.
> Usar `isNotAPaymentMethod` (negación) para mostrar cuando el id NO pertenece al método.
> Nunca mezclarlos sin agrupar correctamente con paréntesis.

### Separación nombre / apellido

Activa cuando `business_detail.id` está en `COMPRA_AHORA_PAGA_DESPUÉS`. Usa `isFullNameSeparate()` internamente (llama a `array.includes(id)`).

Cuando está activo:
- El primer input pasa a ser "Nombre" (`firstName`)
- Aparece un segundo input "Apellido" (`lastName`)
- El email se mueve al segundo row (fuera del row principal)
- `full_name` se construye como `"firstName separator lastName"` (separador `" "` si es COMPRA_AHORA_PAGA_DESPUÉS, `"|"` en otro caso)

### Teléfono (InputCellPhone)

Se muestra cuando `paymentMethodId` **NO** está en `NEQUI` (`isNotAPaymentMethod`).

---

## Funciones internas

| Función | Propósito |
|---------|-----------|
| `isFullNameSeparate(id, methods)` | `true` si `id` está en `methods` (activa modo nombre/apellido separado) |
| `getLabelByPaymentMethod(id, methods, inputName)` | Retorna label localizado según método |
| `getNameValueByPaymentMethod(id, methods, inputName)` | Retorna valor del input según método |
| `getInputNameByPaymentMethod(id, methods, inputName)` | Retorna nombre del campo según método |
| `getErrorField(id, methods, inputName)` | Retorna estado de error del campo según método |

---

## Utilidades externas clave

`src/presentation/paymentLink/utils/validatePaymentMethods.ts`

```ts
isPaymentMethod(id, array)    // true si id está en array
isNotAPaymentMethod(id, array) // true si id NO está en array
```

---

## Edge cases y advertencias

- `paymentMethodsNameById.*` puede ser `undefined` — siempre usar `?? []` al pasarlo a las utilidades.
- `isNotAPaymentMethod` retorna `false` si `paymentMethodId` es `undefined`, lo que oculta el teléfono. Verificar que `paymentMethodId` llegue correctamente desde el padre.
- El `useEffect` que resetea `firstName`/`lastName`/`full_name` se dispara al cambiar `transactionCourse.business_detail.id` — asegurarse de no causar loops con validaciones.

---

## Historial de cambios

| Fecha | Bug ID | Resumen |
|-------|--------|---------|
| 2026-02-24 | BUG-004 | Fix disclaimer: reemplazado `isNotAPaymentMethod` por `isPaymentMethod` para DAVIPLATA/SUMAS_PAY y corregida precedencia de operadores `(A \|\| B) && JSX` |
