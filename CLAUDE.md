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
   sembrados en `localStorage` bajo la clave `bitacora_ganadera_v3`. Servir el repo con
   `python3 -m http.server` y abrirlo con Playwright/Chromium (si falta, instalarlo con pip o npm).
4. **Al publicar, subir `APP_VERSION` (index.html) y `VERSION` (sw.js) en el MISMO commit.**
   Sin eso el service worker no entrega la versión nueva y el usuario sigue viendo la vieja.
5. **Canary antes de entregar**: `grep -c "prompt("` = 0 y
   `grep -c "script.google.com/macros/s/AKfy"` = 0 — las URL de los Apps Script NUNCA van en el
   código, el repositorio es público y se piden una vez por dispositivo a `localStorage`.
6. **Nunca presentar y corregir después.** Si falla, corregir, verificar de nuevo, y recién entregar.
7. **Nunca duplicar lógica de cálculo** entre Lotes, Análisis, Finanzas y el banner: se centraliza.
8. Respaldo de cada publicación: tag `vN` en el commit publicado, subido con `git push origin vN`
   (desde el 17-nov-2026; antes eran copias fechadas en OneDrive `_versiones\ganaderia\`, que
   quedan como archivo histórico).

## Dónde se trabaja (desde 17-nov-2026: en la nube)

Esta app se maneja desde una sesión de Claude Code en la nube (claude.ai/code o la app Claude del
celular), para poder trabajar con la PC apagada.
- **Una sola sesión a la vez.** No trabajar Ganadería en la PC y en la nube al mismo tiempo.
- **Al empezar:** `git pull` de `main`. **Al terminar:** todo cambio verificado queda subido a
  `main` con su tag; nunca dejar trabajo solo en la VM, porque se borra.
- **Datos y referencia en Drive** (conector de Google Drive; si no está disponible en la sesión,
  decirlo al inicio en vez de suponer datos): ver "Dónde están los datos" y "Referencia completa".
- **Publicar solo con aprobación del usuario.** Hallazgos y propuestas en tabla (problema, causa,
  solución, ¿aprueba?).
- **Repo público:** no escribir aquí datos personales nuevos (correo, montos privados, ids de cuentas).

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
- Las **vacunas no se dosifican por peso**; los antiparasitarios sí, y las vitaminas según su etiqueta (Vigantol ADE Fuerte va por categoría, no por peso). `sanDosis` y
  `dosisConPeso` calculan los ml y **siempre muestran de qué fecha es el peso**.

## Estado al 15 de noviembre de 2026

Versión: **v20** (17-nov-2026, Optimizador Bio vs Vigantol y guardia de 90 días del ADE). v14-v16 se hicieron desde claude.ai con la PC apagada. Historial reciente: v4 modal bajo el nav · v5 versión visible y
`preciosLocal` · v6 el simulador explica sus números · v7 % sobre el desembolso y referencia desde
el historial · v8 diagnóstico al pegar JSON · v9 fasciolicida pasa a Saguaymic Plus · v10 dosis con
fecha del peso y cálculo en pasos de protocolo · v11 formulario del boletín, sin importar/exportar · v12 el banner "Mercado ASOGAN — señal de hoy" sale de Visitas (queda solo en Lotes) · v13 guía desplegable de categorías por edad en Lotes, y el $/lb de compra (`compraLbItem`, `lbCompraLote`) en la tarjeta ASOGAN, el análisis, la venta (peso vendido opcional → `pesoVentaLb`) y la fila del lote. · v14 el protocolo respiratorio pasa a NEUMOBAC (bacterina, SC, 5 mL, refuerzo a 21 días; NO cubre IBR/DVB/PI3/BRSV) · v15 producto centralizado en `RESP_PRODUCTO`/`RESP_VIA`/`RESP_MODO` y `repararProductoRespiratoria()` · v16 auditoría de sanidad (brucelosis por edad incluyendo terneras, clostridial en serie de 2 dosis, ivermectina feb/ago intercalada con el fasciolicida may/nov) · v17 `repararRespiratoriaNov2026()` en `normalizarDb` alinea dia/intervalo de las fichas respiratorias con el catálogo (21 días), corrige la vía de la revacunación anual a SC y fija los datos de Normando 2; la fecha real de aplicación arranca en hoy y no acepta fechas futuras · v18 auditoría G1-G8: `_senal` sale del lote (mapa `SENAL_LOTE` en memoria; `_jsReg` la ignora al unir), `leerDrive` rechaza respuestas sin `animales`/`visitas` o con error y la unión no se aplica si deja la app sin animales, protocolos por ciclo (`t.ciclo`, `protCiclo`, `protocoloEnCurso`), `guardarLote` fusiona, la dosis por peso se congela al cumplir (`vac.dosis`, `t.dosisAplicada`) y avisa peso viejo (`avisoPesoViejo`: 30 días terneros, 90 adultos), `editadaManual` y `config.sanDescartes` frenan las reparaciones automáticas, `sincronizarSiHaceFalta` en `online`/`visibilitychange`, sin fechas futuras en brigada ni Peso & CC · v19 vitaminas ADE = **Vigantol ADE Fuerte** (Elanco; IM profunda; dosis FIJA por animal: terneros 2–3 ml, adultos 5–7 ml, campo `dosisCat` en la planilla) e ivermectina = **Ivermec-JB** 1% sola (1 ml/50 kg SC; su etiqueta prohíbe vacas en lactancia y los 28 días antes del parto; retiro carne 28 días). Fantel ya no aparece (las 2 fichas del Día 1 de Hunter y Normando 2 se reescriben a Vigantol en `normalizarDb`). `sanIvermRiesgo()`: aviso rojo y confirmación al asignar ivermectina a una hembra preñada, servida o lactando (Manchas). Decisión del usuario: a Manchas ya se le aplicó ivermectina porque su leche solo la toma Hunter; mientras pueda estar preñada NO se le aplica sin consultar al veterinario. · v20 (desde la nube) **Optimizador Bio ≠ Vigantol**: Optimizador Bio (Agro Insumos, cartilla Provegan) es sobre todo minerales (Se, Cu, Co, Fe, Zn, Mn, I, Ca, Mg) + fósforo orgánico + B12 + algo de D3/E, sin vitamina A, repetible a los 15–45 días; Vigantol es A-D3-E y no se repite antes de 90 días. Decisión: el ADE (Vigantol) sale SOLO de la planilla anual cada 90 días (ene/abr/jul/oct) y del Día 1 neonatal; Optimizador Bio queda donde se busca el mineral (verrugas con Cuprho Plus, Día 30 neonatal, crecimiento D30/D90, parto, pre-servicio, desparasitación). El protocolo de crecimiento ya no lleva Vigantol (D0 = pesar y registrar CC; D60 solo ivermectina). `sanAdeRiesgo()` avisa y pide confirmación al asignar o marcar cumplido un ADE a menos de 90 días del anterior (planilla o ficha con Vigantol/ADE en la dosis); `sinAvisoPeso` quita ese aviso del texto guardado. Se quitaron del catálogo el add-on Engorde Rápido (Engordan Plus ADE cada 30 días) y la Desparasitación Sierra mensual: la desparasitación es la de la planilla (ivermectina feb/ago + fasciolicida may/nov) y fuera de eso solo con signos o coprológico. Crecimiento D60 (ivermectina, ~3 meses) se omite si choca a <45 días con la de la planilla. Criterio del usuario: **mínimo de productos y compras**, lo que toque ni más ni menos; ADE siempre Vigantol. Hunter recibió el 4-oct 5 ml de ADE (dosis de adulto) a los 78 días: tenerlo presente; su próximo ADE (enero) es 2–3 ml. Pendiente: fotos de etiqueta de Cuprho Plus, «COBRE» y Selfos Plus de la cartilla Provegan para decidir un solo producto de cobre. → Resuelto en v20: el «COBRE» de la cartilla Provegan ES **Cuprho Plus** (= Cuprhormone Plus, Agroinsumos: dextronato de cobre 7 g + cobalto 0,043 g/100 ml; SOLO SC delante de la paleta; dosis por peso 2 ml ≤200 kg · 3 ml 200–250 kg · 5 ml >250 kg; repetir a 45–120 días; indicado para papilomatosis, fertilidad y retención de placenta; el exceso de cobre puede ser mortal). Toda ficha con «Cuprho Plus … según peso» calcula los ml en `dosisConPeso`. Pre-Parto quedó en 3 pasos (D-45 Cuprho + Optimizador, parto Optimizador, D+15 Cuprho + Optimizador): sin Magnacal de rutina (vaca de cría, fiebre de leche rara; si se cae al parir, veterinario) y sin el D+8 (Optimizador pide ≥15 días). Pre-Servicio: D-30 Cuprho (omitir si tuvo uno hace <45 días) + Optimizador, D-15 Optimizador. **Selfos Plus** (selenio + A-D-E + fósforo) y **Phosforplus** (fósforo + vitamina D) NO se compran: repiten lo que dan Optimizador Bio y Vigantol. La tarjeta fija de Nutrición ya no pide E-Se de Tornel.

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
- **Respiratoria**: no se consiguió CRB viral; se aplica NEUMOBAC, 5 mL SC a cada ternero.
  Normando 2: serie completa (NEUMOBAC el 25-oct y el 14-nov; el usuario confirmó que la 1ª también
  fue NEUMOBAC). Hunter: 1ª dosis el 14-nov, **refuerzo el 5-dic**. NEUMOBAC no cubre IBR/DVB/PI3/BRSV.
- **Lote toretes** (antes "Lote 1", Santo Domingo): 10 toretes, 891,6 lb prom., $11.233 + flete $60
  + guías $30 (los paga LM) = $11.323, comprados el 10-nov-2026, venta estimada 7-may-2027.
  Los $1.323 extra salen de la meta de Banco Guayaquil: retiro en la meta + egreso "Reinversión".
- Boletines ASOGAN cargados hasta el **9-nov-2026** (ya con el Formulario). Falta el del 26-oct.
  `db.preciosLocal` vacío. `config.rendRef` = 14 (lo puso el usuario).

## Pendiente

1. Hunter: refuerzo de NEUMOBAC el 5-dic.
2. Normando 2: refuerzo del Día 30 el 23-nov con **Optimizador Bio ~1,3 ml IM** (65 kg; NO Vigantol: recibió
   ADE el 25-oct, el próximo ADE desde el 23-ene-2027); seguir la diarrea (coprológico si no cede).
3. Castrar a Hunter (burdizzo). Con pastoreo libre es el único control de consanguinidad: es hijo
   de Manchas y entero la serviría desde los 8-9 meses.
4. Pesar a Manchas: su registro es de junio.
5. Taurus: amarrar la verruga del cuello y vigilar el peso (280 kg sin cambio).

## Referencia completa

El detalle largo —catálogo de sanidad, protocolos, casos resueltos y errores documentados— está en
Google Drive: `ganadera-1_referencia.txt` (id `1rVTqRdPzMg-kucFEPgPydMvDjJl4WqOm`), con un resumen en
`ganadera-1_prompt_manual.txt` (id `1g3Isp1EF6c9A4Rj6QJzeZ3ofZxf-yzXf`). Son copias del 1-nov-2026
(su sección "Estado publicado" está desactualizada: manda este archivo). Este archivo es el extracto
operativo; ante una duda de detalle, ir allá.

## Cómo responde el usuario

Respuestas cortas y técnicas: qué se cambió y por qué, sin mostrar código ni salidas de terminal.
Si algo falla, una línea. Preguntas numeradas en un solo mensaje antes de ejecutar, y solo las que
no se resuelven leyendo Drive o el historial. Con "corrige todo" o "hazlo", ejecutar sin otra ronda.
Señalarle siempre los problemas o mejoras que se detecten, aunque no sean el tema de la conversación.
