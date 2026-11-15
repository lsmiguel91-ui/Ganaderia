# Bitácora Ganadera — contexto para Claude Code

Actualizado: 15 de noviembre de 2026. Mantén este archivo al día cuando cambies algo estructural.

## Qué es esto

App single-file (`index.html`) + PWA (`sw.js`, `manifest.webmanifest`) que Luis Miguel usa para
llevar su ganado. Publicada en GitHub Pages: https://lsmiguel91-ui.github.io/Ganaderia/

Dos operaciones distintas conviven en la app:

- **El Quinche** (sierra, Normando): su hato personal. Es un **hobby**, no un negocio — el potrero
  es de sus padres y no invierte en mejorarlo. Ahí se vende **al ojo**, casi nunca con balanza.
- **Santo Domingo** (trópico): lotes comerciales al 50% con un socio. Ese sí es el negocio, y es
  el mercado al que corresponde el boletín ASOGAN.

Hay una app hermana de Finanzas (repo `lsmiguel91-ui/finanzas`) con la que ésta sincroniza.

## Reglas duras de trabajo (no negociables)

1. **Un reemplazo a la vez, con assert de ocurrencia única.** Nunca reescribir el archivo con
   slicing de strings ni re-tipear contenido salido de una salida de herramienta (puede venir
   truncada). Para editar con emojis UTF-8 multibyte usar Python, nunca `sed`.
2. **Validar el JS después de CADA cambio**: extraer los `<script>` y pasarlos por `vm.Script`
   (no `node --check`: da falsos positivos con `const` repetido en funciones distintas).
3. **Probar en Chromium a 1280 px y a 375 px** antes de publicar, con los datos reales de Drive
   sembrados en `localStorage` bajo la clave `bitacora_ganadera_v3`. Playwright y Chromium ya
   están instalados; no reinstalarlos.
4. **Al publicar, subir `APP_VERSION` (index.html) y `VERSION` (sw.js) en el MISMO commit.**
   Sin eso el service worker no entrega la versión nueva y el usuario sigue viendo la vieja.
5. **Canary antes de entregar**: `grep -c "prompt("` = 0 y
   `grep -c "script.google.com/macros/s/AKfy"` = 0 — las URL de los Apps Script NUNCA van en el
   código, el repositorio es público y se piden una vez por dispositivo a `localStorage`.
6. **Nunca presentar y corregir después.** Si falla, corregir, verificar de nuevo, y recién entregar.
7. **Nunca duplicar lógica de cálculo** entre Lotes, Análisis, Finanzas y el banner: se centraliza.
8. Respaldo fechado a OneDrive `LUIS MIGUEL IA APPS\_versiones\ganaderia\` en cada publicación.

## Antes de analizar o responder cualquier cosa sobre el hato

Este orden es obligatorio y se incumplió varias veces con costo real:

1. Leer el JSON de Drive (abajo).
2. Leer las decisiones ya tomadas (este archivo y la memoria del proyecto).
3. Leer las notas de visita y las fotos que haya mandado.
4. Recién entonces responder o preguntar.

**Preguntar algo que ya está registrado es un error, no una precaución.**

## Dónde están los datos

- **Ganadería**: Drive `bitacora_ganadera_datos.json`, id `1bT4HusRVnZJNsCHJWoxyMwu23WQM9mms`.
  Baja en base64; decodificar con Python, nunca copiar el base64 a mano.
- **Finanzas**: hoja de cálculo "Finanzas personales", id
  `1kCmVig1u4LlgcWsATOHMOuwDhSeXOmkSBdUDZoLiDMU` (exportar a CSV; trae JSON embebido).
- El MCP de Drive **lee y crea, pero no reemplaza el contenido** de un archivo existente. Las
  correcciones de datos se hacen con auto-reparación dentro de la app.
- `script.google.com` está **bloqueado** por el proxy de salida: no se puede escribir a Drive
  desde aquí por el Apps Script. Desde v11 el boletín semanal lo carga el usuario con el botón
  **📝 Formulario** de Precios (viene lleno con el boletín anterior y él corrige lo que cambió).
  Solo cuando se atrasa varias semanas manda las capturas y se le entrega un JSON para "Pegar JSON".
  Se quitaron "Importar JSON" (archivo) y "Exportar datos".

## Arquitectura

- Pestañas: Visitas · Ganado LM · Lotes · Análisis · Precios · Hoja de Ruta · Peso & CC · Nutrición.
- **Sincronización = unión por registro** (`sincUnirDatos(local, base, remoto)`), con base acordada
  por dispositivo en `localStorage` (`bitacora_ganadera_v3_base`). Sin base gana Drive; con base
  gana el lado que cambió; un borrado solo se aplica si el otro lado no tocó el registro.
  **Nunca volver a "subir el db local completo" ni a "bajar y reemplazar".**
  Colecciones que se unen por id en `SYNC_COLECCIONES`; `precios` se une por fecha; `config` y
  `hojaRuta` por clave.
- Antes de normalizar o reparar datos hay que **copiar** el remoto (`db = JSON.parse(_js(remoto))`):
  si no, queda compartido por referencia con `db` y la reparación no se sube.
- **Fechas**: todo pasa por `hoy()`, `parseFechaLocal()`, `fechaLocalStr()` y `sumarMeses()`.
  Prohibido `new Date('AAAA-MM-DD')` (corre un día en Ecuador) y `toISOString()` para fechas.
  La fecha de venta estimada de un lote usa **meses de calendario**, igual que Finanzas, y
  Ganadería envía la fecha exacta (`ff`) a Finanzas, que no la recalcula.
- `<main>` **no debe tener `z-index`** y la etiqueta `</main>` no existe en el archivo: los 17
  modales quedan dentro de `main`, así que un `z-index` ahí crea contexto de apilamiento y el nav
  sticky del celular los tapa. Si alguien cierra `</main>` correctamente, eso deja de importar.
- En bloques `if`/`switch` usar `const f = () => …`, nunca `function f(){}`: Samsung Browser en
  modo estricto los rechaza.
- Guardado = `guardarLocal()` / `save()`, nunca `saveDB()` (no existe).
- Toda función de guardar en edición **fusiona** con el registro existente (`{...existente, ...nuevo}`).

## Motor MKT (fuente única de todo cálculo de mercado)

Objeto global `MKT`, declarado justo después de `getCatIndex`. Lotes, Análisis, Tabla General, el
banner y `calcSenalAsogan` lo consultan; **nunca reimplementar una fórmula fuera de MKT**.

- Tendencia = promedio móvil de 12 meses calendario por fecha real, no por índice de fila.
- Estacionalidad con umbral de confianza (≥3 años, efecto ≥ max(1,5%, 0,6×ruido), consistencia ≥75%).
  Si no es confiable, no se aplica.
- Señal por precio vs tendencia, umbral 3%.
- Offsets por categoría: `b = 1 + catIdx*4` → exc, max, prom, min. catIdx: 0 torete · 1 toro ·
  2 novilloGordo · 3 vacona · 4 vaca · 5 cerdoEngorde.
- "Mes actual" exige **mes Y año**.
- `MKT.backtest` existe pero **no tiene UI**: la compra se decide hacia adelante, no con histórico.

**Veredicto único** (una sola función para banner, Análisis, Lotes y Finanzas): compara el precio
de hoy contra lo normal del mes y el rendimiento anual de la parte del usuario contra su referencia
y contra el plazo fijo CPN (6,5%). Verde = rinde la referencia y el precio está ≤2% sobre lo normal.
Amarillo = rinde pero caro, o rinde menos que la referencia. **Rojo = solo si no supera el CPN.**

## Reglas de negocio

- **Aparcería**: Luis Miguel pone el 100% del capital, lo recupera entero y recibe el 50% de la
  utilidad neta. El trabajo del socio ES su 50%. `capitalLM` excluye los gastos del socio; el
  Análisis de Mercado sí los suma (son métricas distintas, nunca unificarlas).
- Desde v7 **todo porcentaje se calcula sobre compra + gastos**, no solo sobre la compra.
- El **rendimiento de referencia** sale del historial real de lotes liquidados (`rendRealLotes`,
  hoy 14,3%/año con 1 lote); el 15% fijo solo aplica mientras no haya ninguno. El usuario puede
  escribir el suyo y manda sobre ambos.
- La **GDP** del simulador sale de los lotes cerrados (`gdpLotesCerrados`, hoy 0,53 kg/día).
- **ASOGAN no aplica a Ganado LM.** Es el mercado de Santo Domingo. En El Quinche pagan alrededor
  del 54%: referencia real, un toro similar a Taurus se vendió en $400, o sea ~$0,64/lb contra
  $1,18/lb de torete. Para El Quinche el referencial se construye con `db.preciosLocal`, que se
  alimenta de las ventas y ofertas que el propio usuario registre con el peso que ÉL mide.
- **Consanguinidad**: el cruce Taurus × Manchas (madre-hijo) está aceptado **una sola generación**
  y su cría va **exclusivamente a venta comercial**, sea macho o hembra. No repetir en la
  generación siguiente: Normando 2 es el reemplazo.
- Las **vacunas no se dosifican por peso**; los antiparasitarios y vitaminas sí. `sanDosis` y
  `dosisConPeso` calculan los ml y **siempre muestran de qué fecha es el peso**.

## Estado al 15 de noviembre de 2026

Versión: **v16** (14-nov-2026; v14-v16 se hicieron desde claude.ai con la PC apagada). Historial reciente: v4 modal bajo el nav · v5 versión visible y
`preciosLocal` · v6 el simulador explica sus números · v7 % sobre el desembolso y referencia desde
el historial · v8 diagnóstico al pegar JSON · v9 fasciolicida pasa a Saguaymic Plus · v10 dosis con
fecha del peso y cálculo en pasos de protocolo · v11 formulario del boletín, sin importar/exportar · v12 el banner "Mercado ASOGAN — señal de hoy" sale de Visitas (queda solo en Lotes) · v13 guía desplegable de categorías por edad en Lotes, y el $/lb de compra (`compraLbItem`, `lbCompraLote`) en la tarjeta ASOGAN, el análisis, la venta (peso vendido opcional → `pesoVentaLb`) y la fila del lote. · v14 el protocolo respiratorio pasa a NEUMOBAC (bacterina, SC, 5 mL, refuerzo a 21 días; NO cubre IBR/DVB/PI3/BRSV) · v15 producto centralizado en `RESP_PRODUCTO`/`RESP_VIA`/`RESP_MODO` y `repararProductoRespiratoria()` · v16 auditoría de sanidad (brucelosis por edad incluyendo terneras, clostridial en serie de 2 dosis, ivermectina feb/ago intercalada con el fasciolicida may/nov).

**Hato El Quinche (4 animales):**

| Animal | Categoría | Último peso |
|---|---|---|
| Manchas | vaca | 440 kg (26-jun-2026) — desactualizado, el usuario dice que bajó |
| Taurus | torete | 280 kg CC 3,3 (14-nov); el registro del 1-nov quedó corregido a 280 → sin ganancia en 2 semanas |
| Hunter | ternero | 101 kg CC 2,8 (14-nov) |
| Normando 2 | ternero | 65 kg CC 3,5 (14-nov): 55 → 65 kg en 21 días ≈ 0,48 kg/día, mejorando |

- **Taurus**: se queda solo para preñar a Manchas y ganar peso; después se vende. No califica como
  reproductor (escrotal bajo el estándar y es hijo de Manchas). Papilomatosis: D96 cumplido el
  14-nov (protocolo terminado); queda una verruga pequeña y casi seca en el cuello.
- **Normando 2**: biberón con Sprayfo Azul, taza al ras (~130-160 g) por litro, 5 L/día. Diarrea
  desde que llegó; el 8-nov recibió 1,5 mL de Valbentad 25 (albendazol). La visita del 14-nov lo ve
  "más alentado". Si la diarrea sigue: coprológico antes de comprar coccidicida (Baycox se descartó
  por precio, $40).
- **Respiratoria**: no se consiguió CRB viral; se aplica NEUMOBAC. Hunter recibió su 1ª dosis el
  14-nov. Normando 2 tenía una 1ª dosis "CRB" el 25-oct y el 14-nov recibió NEUMOBAC registrado
  como "refuerzo" (con `fechaCumplida` 18-nov, fecha futura: error de registro). Como la bacterina
  necesita 2 dosis del mismo producto, Normando 2 necesita una 2ª dosis de NEUMOBAC ~5-dic.
- **Lote toretes** (antes "Lote 1", Santo Domingo): 10 toretes, 891,6 lb prom., $11.233 + flete $60
  + guías $30 (los paga LM) = $11.323, comprados el 10-nov-2026, venta estimada 7-may-2027.
  Los $1.323 extra salen de la meta de Banco Guayaquil: retiro en la meta + egreso "Reinversión".
- Boletines ASOGAN cargados hasta el **9-nov-2026** (ya con el Formulario). Falta el del 26-oct.
  `db.preciosLocal` vacío. `config.rendRef` = 14 (lo puso el usuario).

## Pendiente

1. Corregir en la app las fichas respiratorias: Normando 2 cumplida el 14-nov (no el 18), su 2ª
   dosis de NEUMOBAC ~5-dic; refuerzo de Hunter el 5-dic (21 días), no el 8-dic.
2. Confirmar si cada ternero recibió 5 mL de NEUMOBAC (el gasto dice "1 dosis entera para Hunter y
   Normando"): si se repartieron 5 mL entre los dos, cada uno quedó con media dosis.
3. Normando 2: refuerzo vitamínico el 23-nov; seguir la diarrea (coprológico si no cede).
4. Castrar a Hunter (burdizzo). Con pastoreo libre es el único control de consanguinidad: es hijo
   de Manchas y entero la serviría desde los 8-9 meses.
5. Pesar a Manchas: su registro es de junio.
6. Taurus: amarrar la verruga del cuello y vigilar el peso (280 kg sin cambio).

## Referencia completa

El detalle largo —catálogo de sanidad, protocolos, casos resueltos y errores documentados— está en
`LUIS MIGUEL IA APPS\Skills\ganadera-1.txt`, con un resumen en
`ganadera-1_prompt_manual.txt`. Este archivo es el extracto operativo; ante una duda de detalle,
ir allá.

## Cómo responde el usuario

Respuestas cortas y técnicas: qué se cambió y por qué, sin mostrar código ni salidas de terminal.
Si algo falla, una línea. Preguntas numeradas en un solo mensaje antes de ejecutar, y solo las que
no se resuelven leyendo Drive o el historial. Con "corrige todo" o "hazlo", ejecutar sin otra ronda.
Señalarle siempre los problemas o mejoras que se detecten, aunque no sean el tema de la conversación.
