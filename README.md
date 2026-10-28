# Ganadería · Luis Miguel

Bitácora del hato de El Quinche, lotes comerciales de Santo Domingo, sanidad y
mercado ASOGAN. Publicada como PWA instalable en GitHub Pages.

**App:** https://lsmiguel91-ui.github.io/Ganaderia/

## Instalar en Android

Abre la dirección en Chrome → menú ⋮ → **Instalar aplicación**.
Queda con ícono propio en la pantalla de inicio y abre a pantalla completa.

## Primer uso

La app pide una vez, en **🗄️ Datos**, la dirección del Apps Script de Ganadería
(y la de Finanzas, si no está ya configurada en ese teléfono). Se guardan solo en
ese dispositivo y no viajan con el código, porque este repositorio es público.

## Datos

- Los datos viven en Drive (`bitacora_ganadera_datos.json`) a través del Apps Script.
- Cada dispositivo une sus cambios con los de Drive registro por registro: un
  teléfono sin señal o dos equipos a la vez no se pisan.
- **🗄️ Datos → Descargar respaldo** baja todo en un `.json`; guárdalo en OneDrive.

## Archivos

| Archivo | Para qué |
|---|---|
| `index.html` | La app completa, en un solo archivo |
| `sw.js` | Service worker. El caché lleva la versión, así no se queda pegada en una anterior |
| `manifest.webmanifest` | Nombre, íconos y modo pantalla completa |
| `icon*.png`, `icon.svg`, `logo-96.png` | Ícono de la app |

Al publicar una versión nueva hay que subir `index.html` (constante `APP_VERSION`)
y la constante `VERSION` de `sw.js` en el mismo commit, para que el caché viejo se descarte.
