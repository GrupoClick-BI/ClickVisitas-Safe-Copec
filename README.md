# Safe CRM — Wrapper iOS (Capacitor)

Empaquetado de la web app `https://safe-copec.clickbi.cl` (Safe Copec / Safe CRM, ClickBI)
como app iOS usando **Capacitor 8** con `server.url` — la app carga el sitio remoto
dentro de un WKWebView. No hay código web local salvo un `www/index.html` de respaldo.

## Qué incluye

- `capacitor.config.json` — `server.url` apuntando al sitio; `allowNavigation`
  permite dominios `*.clickbi.cl` y `*.grupo-click.cl` dentro del webview (el
  flujo de solicitud de acceso/soporte vive en `soporte.grupo-click.cl` — no
  hay auto-registro, las cuentas se piden ahí). Todo lo demás externo —Google
  Maps, etc.— abre en Safari.
- Proyecto iOS generado en `ios/` (Xcode project listo).
- `Info.plist` con permisos declarados:
  - `NSLocationWhenInUseUsageDescription` (georreferencia de gestiones)
  - `NSCameraUsageDescription` (fotos de evidencia)
  - `NSPhotoLibraryUsageDescription` / `NSPhotoLibraryAddUsageDescription`
  - `NSMicrophoneUsageDescription` (grabación de audio en gestiones/casos)
- Plugins instalados (Capacitor 8):
  - `@capacitor/app` — estado del ciclo de vida (foreground/background) y manejo
    de deep links de retorno.
  - `@capacitor/browser` — abre links externos (Mesa de Ayuda, Google Maps) en
    Safari en vez de romper la navegación dentro del WKWebView.
  - `@capacitor/camera` — fotos de evidencia en gestiones (requiere
    `NSCameraUsageDescription`).
  - `@capacitor/filesystem` — guardar a disco los Excel/CSV/PDF de DataTables,
    que en WKWebView no caen solos (ver checklist punto 3).
  - `@capacitor/geolocation` — georreferencia de gestiones (requiere
    `NSLocationWhenInUseUsageDescription`).
  - `@capacitor/keyboard` — ajustar el viewport cuando aparece el teclado nativo.
  - `@capacitor/push-notifications` — base para avisos de nuevos casos/gestiones
    (pendiente integración server-side con APNs, ver checklist punto 6).
  - `@capacitor/share` — hoja de compartir nativa para los archivos exportados
    (Excel/CSV/PDF) una vez guardados con filesystem.
  - `@capacitor/splash-screen` — pantalla de carga nativa mientras el WKWebView
    carga el sitio remoto.
  - `@capacitor/status-bar` — color/estilo de la barra de estado acorde al sitio.

## Requisitos para compilar

- **Mac** con Xcode 16+
- Node.js 22+ y npm
- Cuenta **Apple Developer Program** (US$99/año) — idealmente la organización de
  ClickBI o de Copec
- Nada más: las dependencias nativas se resuelven por **Swift Package Manager**
  automáticamente al abrir el proyecto en Xcode (Capacitor 8 ya no usa CocoaPods
  por defecto).

## Pasos para build y TestFlight

```bash
npm install
npm run sync          # equivale a: npx cap sync ios
npm run open          # abre el proyecto en Xcode (npx cap open ios)
```

En Xcode:

1. Target **App** → **Signing & Capabilities** → seleccionar el *Team* (cuenta
   developer). Bundle ID sugerido: `cl.clickbi.safecopec` (cambiar si se registra otro).
2. (Opcional, solo si se usará push) `+ Capability` → **Push Notifications**.
3. Reemplazar íconos en `ios/App/App/Assets.xcassets/AppIcon.appiconset`
   (o usar `npx @capacitor/assets generate` con un ícono fuente).
4. **Product → Archive** → **Distribute App** → **App Store Connect** → Upload.
5. En App Store Connect: crear la ficha de la app, esperar el procesamiento del
   build y habilitarlo en **TestFlight** (testers internos bastan para la demo).

## Build en Codemagic (alternativa sin Mac propio)

`codemagic.yaml` — dos workflows:

- `ios-safe-crm` — firma vía App Store Connect API, sube directo a TestFlight.
- `android-safe-crm` — build debug sin firma, genera un `.apk` para pruebas.

### Setup único de firma iOS (`ios_signing`)

El certificado de distribución se reutiliza entre builds (no se crea uno
nuevo cada vez — eso agotaba los 3 certificados de Apple). Setup de una sola
vez, ya hecho para este proyecto:

- Grupo de variables en Codemagic: `ios_signing`
- Variable: `IOS_CERTIFICATE_PRIVATE_KEY` (segura, contenido del artifact
  `cert-key.pem.b64` del primer build "bootstrap")

Detalle completo del proceso: `PROYECTOS/_templates/capacitor-wrapper/NEW_PROJECT_CHECKLIST.md`.

## Checklist de fallos esperados (evidencia para pedir el código)

Estas son las limitaciones de un wrapper puro — se pueden demostrar en TestFlight
y sirven para justificar el acceso al código fuente:

| # | Función | Qué pasa en la app |
|---|---------|--------------------|
| 1 | **Login "Ingresar con Go"** (Google OAuth) | **Falla.** Google bloquea OAuth dentro de WKWebView (`disallowed_useragent` o redirect roto). Solo funciona el login usuario/clave. |
| 2 | **Botón Print** en las tablas DataTables | **No hace nada.** `window.print()` no existe en WKWebView. |
| 3 | **Botones Excel / CSV / PDF** de DataTables | **El archivo no se guarda.** Las descargas blob no caen a disco en WKWebView (se necesita `@capacitor/filesystem` + código en el sitio). |
| 4 | **Geolocalización en gestiones** | Funciona **solo porque** se declaró `NSLocationWhenInUseUsageDescription` en el Info.plist. Sin el entitlement falla en silencio. |
| 5 | **Links externos** (Mesa de Ayuda, links a Google Maps) | Abren en **Safari fuera de la app**; al volver se pierde contexto de navegación. |
| 6 | **Push notifications** | No hay avisos de nuevos casos/gestiones — requiere integración server-side con APNs. |
| 7 | **Offline / sin señal** | La app queda en blanco — no hay service worker ni caché (la PWA del sitio está incompleta). |
| 8 | **Notch / safe area** | El viewport actual no usa `viewport-fit=cover`; puede verse con bandas en iPhones con notch. |
| 9 | **Layout de login en pantallas angostas** (confirmado en TestFlight build 1) | Los campos Usuario/Password se ven cortados por el lado izquierdo (muestran "...iano" / "...word" en vez del texto completo). El sitio no tiene un diseño responsive real para móvil — pasaría igual en Safari, no es un bug del wrapper. |
| 10 | **Sesión se cierra sola** (confirmado en TestFlight build 1) | Tras iniciar sesión, la app vuelve a pedir login poco después. Probable causa: cookie de sesión sin `Max-Age`/`Expires`, o `SameSite` restrictivo que WKWebView trata distinto que Safari. |

## Qué corregir una vez tengan el código

1. **PWA real**: arreglar `manifest.json` (íconos rotos 404), agregar service worker,
   `meta mobile-web-app-capable`, `viewport-fit=cover`.
2. **Descargas**: exponer los Excel/CSV por endpoint y usar `Capacitor.Filesystem` +
   `Capacitor.Share` cuando `Capacitor.isNativePlatform()`.
3. **Print**: ocultar el botón en la app (`Capacitor.isNativePlatform()`) o
   implementar puente a `UIPrintInteractionController`.
4. **Login Google**: sacarlo del webview o migrar a flujo nativo
   (`@capacitor/browser` + deep link de retorno).
5. **Push**: endpoint para registrar tokens APNs y webhook al crear caso/gestión.
6. **Offline mínimo**: service worker con caché de login/shell.

## Distribución recomendada (post-TestFlight)

Como app vendida a un cliente específico (Copec), la vía correcta **no** es el
App Store público sino **Custom App (B2B)**: en App Store Connect →
*Pricing & Availability → Distribution for Business/Education* → se asigna la
organización de Copec (vía su Organization ID en Apple Business Manager) y la app
se instala por MDM o códigos. Evita la revisión 4.2 por "solo wrapper web" y no
queda visible públicamente.
