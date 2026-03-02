# Skill: AttachmenComponent (PolicyAttachmentCollapse)

## Estructura de archivos

```
src/presentation/adminSettings/TermsAndConditions/components/AttachmenComponent/
├── index.tsx                                    ← Punto de entrada público (envuelve con Provider)
├── types/AttachmenComponent.types.ts            ← Todos los tipos e interfaces
├── constants/constants.ts                       ← Enums, regex y constantes compartidas
├── context/context.tsx                          ← AttachmentProvider + useAttachmentContext
├── hooks/usePolicyAttachmentCollapse.ts         ← Toda la lógica del componente
└── components/
    ├── PolicyAttachmentCollapse.tsx             ← UI principal (PolicyAttachmentCollapseUI)
    ├── PolicyAttachmentCollapse.scss
    ├── AttachmentChild/
    │   ├── AttachmentChild.tsx                  ← Texto de ayuda debajo del input
    │   └── AttachmentChild.scss
    └── CreateModalChild/
        ├── CreateModalChild.tsx                 ← Texto de advertencia del modal de nueva versión
        └── CreateModalChild.scss
```

---

## ¿Qué hace?

Componente de selección de adjunto para políticas (Términos y Condiciones). Permite elegir entre:
- **`s3`** → carga un archivo PDF (máx. 800 KB) vía `FileLoader`
- **`url`** → ingresa un enlace con validación de formato

Visualmente: un `OutlinedInput` con el nombre del link (`linkName`) + un `ButtonCard` con ícono de link que despliega las opciones de tipo + archivo/URL.

---

## Dónde se usa cada parte

| Exportación | Consumidor externo | Nota |
|---|---|---|
| `PolicyAttachmentCollapse` (default/named de `index.tsx`) | `PolicyItem/PolicyItem.tsx` | **1 solo lugar** → modificable directamente |
| `CreateModalChild` | `CreatePolicie.tsx` | **1 solo lugar** → modificable directamente |
| `constants.ts` → `TypePolicie`, `PortalPayerType` | `useCreatePolicie.tsx`, `policiesAdapter.ts`, `PolicyItem.tsx` | **Más de 3 lugares** → NO modificar, crear nueva constante si es necesario |
| `constants.ts` → `MAX_LENGTH_TEXT` | `PolicyItem.tsx` | **1 lugar** → modificable |

---

## Props de `PolicyAttachmentCollapse` (punto de entrada público)

```ts
interface PolicyAttachmentCollapseProps {
  // Estado inicial (pasado al AttachmentProvider)
  initialType?: "url" | "s3" | "";
  initialFiles?: File[];
  initialLink?: string;       // URL externa o base64 del PDF
  initialLinkName?: string;   // Texto visible del link

  // Callbacks (disparados en tiempo real o al aplicar)
  onTypeChange(type: AttachmentType): void;
  onFilesChange(files: File[], fileSize: number): void;
  onLinkChange(link: string): void;
  onLinkNameChange(link: string): void;
  onValidation(errors: { file?: string; link?: string }): void;

  // Acciones
  onApply(): void;   // Llamado después de serializar y cerrar el popover
  onCancel(): void;  // Llamado por botón "Limpiar" (modo edición)

  disabled?: boolean;
  isCreating: boolean;                   // true → "Cancelar/Aplicar" | false → "Limpiar"
  translateText(label: string): string;  // wrapper de t() con keyPrefix ya aplicado
}
```

---

## Constantes (`constants/constants.ts`)

```ts
export enum TypePolicie {
  S3  = "s3",
  URL = "url",
  V1  = "v1",   // tipo legado, no se usa en la UI actual
}

export enum PortalPayerType {
  PORTALKEY    = "PAYER",
  MICROSITEKEY = "PAYMENT_LINK",
}

export const MAX_FILE_SIZE_8K  = 819200;  // 800 KB en bytes
export const MAX_LENGTH_TEXT   = 2000;    // límite de caracteres del RichTextEditor
export const urlPattern        = /^(?:https?:\/\/)?(?:www\.)?[a-z\d-]+(?:\.[a-z\d-]+)*\.[a-z]{2,}(?::\d+)?(?:[/?#][^\s]*)?$/i;
export const templateAttachtmentPdf = `...`;  // HTML del label del FileLoader
```

---

## Contexto (`context/context.tsx`)

`AttachmentProvider` gestiona el estado local del componente. Solo vive dentro de `index.tsx`.

| Estado | Tipo | Valor inicial |
|---|---|---|
| `open` | `boolean` | `false` |
| `type` | `"url" \| "s3" \| ""` | `initialType \|\| ""` |
| `files` | `File[]` | `initialFiles \|\| []` |
| `link` | `string` | `initialLink \|\| ""` |
| `linkName` | `string` | `initialLinkName \|\| ""` |

**Importante:** `link` tiene un `useEffect` que sincroniza con `initialLink` cada vez que cambia. El resto de campos (`type`, `files`, `linkName`) no se sincronizan — solo se inicializan una vez.

`useAttachmentContext()` lanza error si se usa fuera del Provider.

---

## Hook `usePolicyAttachmentCollapse`

Consume el contexto y orquesta toda la lógica. Retorna:

| Campo | Tipo | Descripción |
|---|---|---|
| `open` | `boolean` | Estado de apertura del `ButtonCard` |
| `toggleOpen` | `() => void` | Alterna `open` |
| `type` | `AttachmentType` | Tipo seleccionado actualmente |
| `files` | `File[]` | Archivos cargados |
| `link` | `string` | URL o base64 |
| `linkName` | `string` | Nombre visible del enlace |
| `urlError` | `string` | Mensaje de error si la URL no pasa `urlPattern` |
| `fileLoadError` | `boolean` | `true` si el `FileLoader` reportó error |
| `buttonCardRef` | `RefObject<any>` | Ref del `ButtonCard` para llamar `closeCard()` |
| `handleTypeChange` | `(type) => void` | Actualiza `type` en contexto + llama `onTypeChange` |
| `handleFilesUpdate` | `(pondFiles) => void` | Extrae `File[]` de FilePond, actualiza contexto + `onFilesChange` |
| `handleLinkChange` | `(e) => void` | Valida URL con regex, actualiza `urlError` + `onLinkChange` |
| `handleLinkNameChange` | `(e) => void` | Actualiza `linkName` en contexto + `onLinkNameChange` |
| `handleApplyClick` | `async () => void` | Serializa y dispara callbacks (ver flujo abajo) |
| `validateButton` | `() => boolean` | `true` (disabled) si hay error de carga, s3 sin archivos, o url vacía/inválida |
| `errors` | `(error?: string) => void` | Setter de `fileLoadError` (pasado al `FileLoader` como `setErrors`) |
| `closeCard` | `() => void` | Cierra `ButtonCard` y llama `handleApplyClick` |
| `onCancelButton` | `() => void` | Solo cierra el `ButtonCard` sin disparar nada |
| `onCleanButton` | `() => void` | No-op (sin implementación) |

---

## Flujo de `handleApplyClick`

```
Usuario presiona "Aplicar" → closeCard()
  → buttonCardRef.current.closeCard()   # cierra el popover visualmente
  → handleApplyClick()
      ├── type === "url"
      │     → onLinkChange(link)        # propaga la URL tal cual
      │     → onFilesChange([], 0)      # limpia archivos
      └── type === "s3" && files.length > 0
            → fileToBase64(files[0])    # convierte File a data URL
            → extrae solo el segmento base64 (split("base64,")[1])
            → onLinkChange(base64)      # propaga el base64 puro
            → onFilesChange(files, files[0].size)
  → onApply()                           # callback genérico del padre
```

**Nota importante:** `onLinkChange` recibe base64 puro (sin prefijo `data:application/pdf;base64,`). El padre (Formik) lo guarda en `linkUrl`. Al reconstruir desde Redux, si `linkUrl` empieza con `data:`, el hook lo convierte de vuelta a `File` para el `FileLoader`.

---

## Lógica de reconstitución de archivo desde Redux

En `usePolicyAttachmentCollapse`, hay un `useEffect` que observa `link`:

```ts
useEffect(() => {
  if (type === "s3" && link?.startsWith("data:") && files.length === 0) {
    const file = dataURLtoFile(link, "policy-attachment.pdf");
    if (file) setFiles([file]);
  }
}, [link]);
```

Esto ocurre cuando Formik reinicializa valores desde Redux y `linkUrl` contiene una data URL (e.g. `data:application/pdf;base64,...`). El componente la convierte a `File` para que el `FileLoader` muestre el archivo como ya cargado.

---

## `validateButton()` — cuándo deshabilita "Aplicar"

```
fileLoadError === true        → disabled
type === "s3" && !files.length → disabled
type === "url" && (link === "" || !urlPattern.test(link)) → disabled
type === ""                   → no aplica (botones no se muestran)
```

---

## Claves i18n usadas

Todas usan el keyPrefix `properties.termsAndConditions.pages.main`.

| Clave | Uso |
|---|---|
| `textLink` | Label del `OutlinedInput` principal (nombre del link) |
| `loadFile` | Label del radio "Cargar archivo" |
| `addLink` | Label del radio "Agregar enlace" |
| `linkName` | Placeholder del `OutlinedInput` de URL |
| `fileLoaderText` | Texto de ayuda bajo el `FileLoader` |
| `linkLoaderText` | Texto de ayuda bajo el input de URL |
| `cancel` | Botón cancelar (`isCreating = true`) |
| `apply` | Botón aplicar (`isCreating = true`) |
| `clear` | Botón limpiar (`isCreating = false`) |
| `notValidUrl` | Error de validación de URL |

---

## `CreateModalChild`

Componente independiente (no usa el contexto de `AttachmentProvider`). Se usa en el `GenericModal` de confirmación de nueva versión en `CreatePolicie.tsx`.

- Tiene su propio `useTranslation` con el mismo keyPrefix.
- Muestra tres textos: `policiesWarning` (con `<Trans>`), `confirmNewVersionNote`, `confirmNewVersionQuestion`.
- No recibe props.

---

## `AttachmentChild`

Componente de texto de ayuda simple. Recibe `translate` y `childText` (clave i18n). Solo renderiza un `<BodyText>`. No tiene lógica.

---

## Estructura visual

```
<OutlinedInput label="Nombre del link" value={linkName}>
  endAdornment:
    <ButtonCard icon={<LinkIcon />} ref={buttonCardRef}>
      <RadioButton options={["Cargar archivo", "Agregar enlace"]}>
        auxiliarElement (s3):
          <FileLoader accept="pdf" maxSize=800KB />
          <AttachmentChild text="fileLoaderText" />
        auxiliarElement (url):
          <Divider />
          <OutlinedInput placeholder="URL" error={urlError} />
          <AttachmentChild text="linkLoaderText" />
      </RadioButton>

      {type && (
        isCreating
          ? <SecondaryButton "Cancelar"> + <PrimaryButton "Aplicar" disabled={validateButton()}>
          : <SecondaryButton "Limpiar">
      )}
    </ButtonCard>
</OutlinedInput>
```

---

## Edge cases y advertencias

- **`onCancel` vs `onCancelButton`:** `onCancel` es prop del padre (llamado por "Limpiar" en modo edición). `onCancelButton` es interno (solo cierra el `ButtonCard` en modo creación). Son funciones distintas con distinto alcance.
- **`onCleanButton` es un no-op.** Declarado en el hook pero sin implementación; no borrar por si acaso se usa en el futuro, pero no tiene efecto.
- **`isCreating` hardcodeado en `PolicyItem`.** El padre (`PolicyItem.tsx`) siempre pasa `isCreating={true}` y `onApply={() => {}}`. El callback `onApply` del padre es inerte; toda la lógica real de persistencia ocurre vía `onTypeChange`, `onFilesChange`, etc. en tiempo real.
- **`onValidation` no está conectado en el hook.** La prop `onValidation` llega a `PolicyAttachmentCollapseUI` pero no se pasa a `usePolicyAttachmentCollapse`. Los errores de archivo se manejan internamente con `fileLoadError`; los de link con `urlError`. La prop `onValidation` del padre (`usePolicyItem.handleValidation`) solo actúa si el padre la recibe, pero actualmente el hook interno no la llama.
- **`MAX_LENGTH_TEXT` se usa fuera.** `PolicyItem.tsx` importa `MAX_LENGTH_TEXT` directamente de `constants/constants.ts` para pasarlo al `RichTextEditor`. No es exclusivo de `AttachmenComponent`.
- **`PortalPayerType` y `TypePolicie` son compartidos.** `policiesAdapter.ts` y `useCreatePolicie.tsx` los importan también. No modificar estos enums — crear nuevas constantes si se necesita un valor distinto.
