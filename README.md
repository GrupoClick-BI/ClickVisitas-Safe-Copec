# Safe CRM — Wrapper iOS (Capacitor)

Empaquetado de la web app `https://safe-copec.clickbi.cl` (Safe Copec / Safe CRM, ClickBI)
como app iOS usando **Capacitor 8** con `server.url` — la app carga el sitio remoto
dentro de un WKWebView. No hay código web local salvo un `www/index.html` de respaldo.

## Qué incluye

- `capacitor.config.json` — `server.url` apuntando al sitio; `allowNavigation`
  solo permite dominios `*.clickbi.cl` dentro del webview (todo lo externo —Mesa de
  Ayuda, Google Maps— abre en Safari).
- Proyecto iOS generado en `ios/` (Xcode project listo).
- `Info.plist` con permisos declarados:
  - `NSLocationWhenInUseUsageDescription` (georreferencia de gestiones)
  - `NSCameraUsageDescription` (fotos de evidencia)
  - `NSPhotoLibraryUsageDescription` / `NSPhotoLibraryAddUsageDescription`
- Plugins instalados: app, browser, camera, filesystem, geolocation, keyboard,
  push-notifications, share, splash-screen, status-bar.

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

El proyecto incluye `codemagic.yaml` listo para Codemagic:

1. Subir el proyecto a un repo git y conectarlo en Codemagic.
2. Crear una API key en App Store Connect (Users and Access → Keys) y registrar
   la integración **"app_store_connect"** en Codemagic — así la firma es
   automática (certificados + provisioning).
3. Cada build produce el `.ipa` y lo sube directo a **TestFlight**.

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
