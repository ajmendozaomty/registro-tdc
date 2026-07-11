# CONTEXT_TDC.md — Registro TDC

> Contexto técnico del proyecto. Ruteo general en `CLAUDE.md` (repo raíz `AJMO REPO`).

## Qué es

App web + script de extracción para llevar el control de gasto/pago de tarjetas de crédito
personales: **Rappi Card (R)**, **Santander Like U (S)**, **Nu Bank (N)**, **American Express (AMEX)**.

- **Repo:** `ajmendozaomty/registro-tdc` (separado, no vive dentro de `ajmo-repo`)
- **Deploy:** `https://registro-tdc.vercel.app` (Vercel)
- **Frontend:** `index.html` (single-file, vanilla JS)
- **Backend:** `api/proxy.js` — función serverless de Vercel que hace de proxy hacia la API de
  Anthropic (`api.anthropic.com/v1/messages`, modelo `claude-sonnet-4-6`) para extraer las
  transacciones de los screenshots subidos.

## Flujo actual (manual, pre-Fase 2)

1. El usuario toma screenshots de los movimientos de su tarjeta desde el celular.
2. Abre la app, selecciona **Quincena (Q1/Q2) + Mes + Año + Fecha de pago**.
3. Sube los screenshots; Claude (vía `api/proxy.js`) extrae: Fecha, Hora (a veces), Descripción,
   Monto, Banco, Status (Normal/Anulado), y genera el **Código de Pago**.
4. La app genera un `.xlsx` por quincena+tarjeta, con nombre tipo
   `{TARJETA}_{CODIGO}.xlsx` (ej. `N_Q1ABR26N190426.xlsx`).
5. El usuario descarga el xlsx y lo guarda en `OneDrive/Registro Quincenas/{año}/`.
6. **Manualmente** llena en el histórico maestro (`OneDrive/Gastos Mensuales - AJMO.xlsx`,
   hoja "Control de Gastos") las columnas que la app no captura: Usuario, Clasificación, Detalle,
   Transc Type (Ida/Regreso), Estatus de pago.

### Patrón del Código de Pago

`Q[1|2][MES][YY][TARJETA][DDMMYY]` — ej. `Q2FEB26AMEX060326` = quincena 2, febrero 2026, Amex,
fecha de pago 06/03/26.

**Importante:** este código es solo un identificador para buscar la transacción en el banco (no
está diseñado para calzar con los códigos que el usuario escribía a mano en el histórico antes de
que existiera la app). No usar como llave de deduplicación entre histórico viejo y exports nuevos —
para eso comparar por **Fecha + Monto + similitud de texto + Tarjeta**.

## Bug conocido: año incorrecto en fechas extraídas (Nu Bank)

Durante la Fase 1 (ver abajo) se detectó que ~22% de las filas nuevas de archivos `N_` (Nu Bank)
traían el año de la transacción 1-2 años por debajo del real (ej. código dice quincena de abril
**2026**, pero la fecha extraída dice marzo **2024**). Día y mes siempre correctos, solo el año
falla. Es sistemático en algunos archivos (5 de 7 analizados), no en todos.

**Hipótesis:** error de Claude al leer el año en el screenshot de Nu Bank específicamente (quizás
formato de fecha en la UI de esa app bancaria es ambiguo o se corta).

**Pendiente Fase 2:** agregar validación en `api/proxy.js` o en el prompt de extracción — si el año
extraído se aleja más de ~2 meses de la quincena que el usuario seleccionó en el formulario
(Quincena+Mes+Año), rechazar/flaggear esa fila para que el usuario la confirme antes de generar el
xlsx, en vez de dejarla pasar silenciosamente.

## Fase 1 — Consolidación histórica (completada 2026-07-10)

Objetivo: unir los exports de la app (30 archivos en `OneDrive/Registro Quincenas/2026/`, 537 filas)
con el histórico manual (`Gastos Mensuales - AJMO.xlsx`, 2080 filas) en un solo archivo.

**Decisiones tomadas:**
- Transacciones "Anulado" (canceladas/reversadas): se buscó su cargo original (mismo monto
  absoluto, tarjeta, fecha cercana) y se excluyó el par completo. 4 pares encontrados y excluidos
  (8 filas). 9 quedaron sin par (mayormente holds de $1 de Lime/Amazon que nunca tienen cargo
  positivo) — esas se mantienen en el consolidado con `Estatus='Anulado'`.
- 1 fila duplicada de alta confianza (ya estaba en el histórico manual) — excluida.
- 1 hold duplicado entre 2 archivos de quincenas con ventanas solapadas — se dejó solo 1 ocurrencia.
- Bug de fecha (ver arriba): se corrigió el año de las 120 filas afectadas usando la quincena
  esperada según el nombre del archivo (día/mes se respetan, solo se ajusta año). Cada fila
  corregida quedó marcada para trazabilidad.
- **Clasificación automática**: se construyó un diccionario de reglas (`data/reglas_clasificacion.json`,
  649 keywords) a partir de las 2080 filas ya clasificadas a mano en el histórico. Cobertura ~78-80%
  con confianza ≥70%; el resto queda marcado "Revisar". **Usuario y Transc Type (Ida/Regreso) NO son
  inferibles por texto** — Usuario porque una sola persona concentra ~88-93% de las transacciones
  (sesgo, no señal real), Transc Type porque depende de la dirección del viaje, no del merchant. Este
  dato es clave para Fase 2: si se quiere automatizar Usuario, hay que capturarlo en el formulario de
  la app (ej. como un campo más al subir el screenshot), no inferirlo del texto.

**Resultado:** 527 filas nuevas netas (537 − 10 excluidas), archivo fusionado en
`OneDrive/Gastos Mensuales - AJMO (fusionado).xlsx` (2607 filas totales), pendiente de que el
usuario lo revise en Excel antes de reemplazar el archivo madre original.

**Artefactos reutilizables para Fase 2** (en este repo, carpeta `data/`):
- `reglas_clasificacion.json` — diccionario keyword → Clasificación con nivel de confianza. Se
  puede usar como primer clasificador automático dentro de la propia app (en vez de solo en la
  consolidación histórica).

## Fase 2 — Automatización end-to-end (pendiente, no iniciada)

Objetivo declarado por el usuario: que desde el registro del screenshot se vaya actualizando el
histórico automáticamente y se clasifiquen los gastos "de alguna manera automática", sobre todo el
tipo de transacción (Clasificación).

Ideas discutidas (sin decisiones cerradas todavía):
- Usar `data/reglas_clasificacion.json` como clasificador de primera pasada dentro de la app misma.
- Agregar validación de año en la extracción (ver bug conocido arriba).
- Capturar Usuario como input explícito del formulario (no inferible por texto).
- Conectar la app directo al histórico maestro (hoy es 100% manual: descargar xlsx → copiar a mano).

## Checklist de handover (para retomar)

- [ ] Usuario debe abrir `OneDrive/Gastos Mensuales - AJMO (fusionado).xlsx` en Excel y confirmar
      que las 527 filas nuevas (fila 2082 en adelante de "Control de Gastos") se ven bien:
      fechas, montos, clasificación, filas resaltadas "Revisar".
- [ ] Una vez confirmado, reemplazar el archivo madre original
      (`OneDrive/Gastos Mensuales - AJMO.xlsx`) con el fusionado — **pendiente, no hecho todavía**.
- [ ] Llenar a mano la columna Usuario de las 527 filas nuevas (no es automatizable con lo que hay
      hoy).
- [ ] Segundo barrido de clasificación: el diccionario actual no cubre todos los conceptos nuevos
      (usuario lo confirmó) — cuando el histórico crezca con estas 527 filas ya clasificadas/revisadas,
      vale la pena regenerar `reglas_clasificacion.json` con el set ampliado para mejorar cobertura.
- [ ] Decidir alcance real de Fase 2 (automatización end-to-end) — todavía no se ha definido plan de
      trabajo, solo la intención general.
- [ ] Este PR (branch `feature/context-tdc-fase1`) crea `CONTEXT_TDC.md` y `data/reglas_clasificacion.json`
      — falta que el usuario lo apruebe y mergee a `main` (no se instaló `gh` CLI en esta sesión, así
      que el PR se debe abrir manualmente desde el link que da `git push`).

## Archivos y rutas clave

| Qué | Dónde |
|---|---|
| Histórico maestro (original) | `OneDrive/Gastos Mensuales - AJMO.xlsx` |
| Histórico maestro (fusionado, pendiente revisión) | `OneDrive/Gastos Mensuales - AJMO (fusionado).xlsx` |
| Exports de la app (2026) | `OneDrive/Registro Quincenas/2026/` |
| Consolidado solo-2026 (referencia, previo a fusión) | `OneDrive/Registro Quincenas/Consolidado 2026.xlsx` |
| Este repo (clonado) | `C:\Aleja\Repo\AJMO REPO\Registro tarjetas\` |
