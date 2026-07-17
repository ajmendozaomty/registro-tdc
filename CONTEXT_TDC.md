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
  transacciones de los screenshots subidos. Protegido con header `x-app-secret` (ver
  "Seguridad" abajo).

## Flujo actual (2026-07-12, con automatización de Fase 2 ya en producción)

1. El usuario toma screenshots de los movimientos de su tarjeta desde el celular.
2. Abre la app, selecciona **Quincena (Q1/Q2) + Mes + Año + Fecha de pago**.
3. Sube los screenshots de transacciones **y, opcionalmente, un screenshot del saldo total /
   estado de cuenta** de esa tarjeta.
4. Claude (vía `api/proxy.js`) extrae: Fecha, Hora (a veces), Descripción, Monto, Banco, Status
   (Normal/Anulado/Pendiente), y genera el **Código de Pago**. Recibe el año esperado de la
   quincena seleccionada (ya no lo "infiere"), y marca `alerta` en transacciones dudosas.
5. La app, en el navegador, enriquece cada transacción con:
   - **Clasificación automática** (`data/reglas_clasificacion.json`, matching longest-match-wins).
   - **Hash** (FNV-1a de Fecha+Hora+Descripción+Monto+Tarjeta) para detectar duplicados dentro
     del mismo lote y contra tarjetas ya procesadas en la sesión.
   - Si se subió screenshot de saldo, lo compara contra la suma de transacciones y muestra
     "✓ saldo cuadra" o "⚠ saldo difiere $X".
6. La app genera un `.xlsx` por quincena+tarjeta (`{TARJETA}_{CODIGO}.xlsx`) con las **mismas 14
   columnas y orden que "Control de Gastos"** del histórico maestro (incluyendo fórmulas de Data
   checkup/Pago Pendiente), más 2 columnas extra de referencia (`Hash`, `Alerta`).
7. El usuario descarga el xlsx y lo guarda en `OneDrive/Registro TDC/Registro Quincenas/{año}/`.
8. Para incorporarlo al histórico maestro: **copiar el bloque de columnas A:N y pegarlo al final**
   de "Control de Gastos" (las fórmulas se re-ajustan solas al pegar). Ya no hay que reconciliar
   columnas a mano — sigue siendo un paso manual (el auto-append real vía Graph API es un
   fast-follow, ver Pendientes).
9. **Todavía manual**: Usuario (quién hizo el gasto) y Transc Type (Ida/Regreso) — no son
   inferibles por texto (ver Fase 1). Clasificación y fecha ya vienen resueltas en ~80% de los
   casos; el resto queda marcado para revisar (columna `Clasificacion` vacía, o `Alerta`/`Hash`
   con nota).

### Patrón del Código de Pago

`Q[1|2][MES][YY][TARJETA][DDMMYY]` — ej. `Q2FEB26AMEX060326` = quincena 2, febrero 2026, Amex,
fecha de pago 06/03/26.

**Importante:** este código es solo un identificador para buscar la transacción en el banco (no
está diseñado para calzar con los códigos que el usuario escribía a mano en el histórico antes de
que existiera la app). No usar como llave de deduplicación entre histórico viejo y exports nuevos —
para eso comparar por **Fecha + Monto + similitud de texto + Tarjeta** (o el `Hash` ya calculado
para exports nuevos entre sí).

## Seguridad

Auditoría de seguridad completada 2026-07-12. Hallazgo crítico corregido, resto documentado:

- ✅ **[CRÍTICO, corregido]** `api/proxy.js` no tenía autenticación — reenviaba cualquier `req.body`
  sin validar hacia la API de Anthropic. Cualquiera que descubriera la URL de Vercel podía usar la
  API key del dueño sin límite. Fix: header `x-app-secret` validado contra env var
  `APP_SHARED_SECRET` en Vercel (marcada Sensitive, igual que `ANTHROPIC_API_KEY`). PR #2, mergeado
  y verificado en producción.
- ⚠️ **[MEDIO, pendiente]** `data/reglas_clasificacion.json` (en este repo) contiene nombres reales y
  categorías médicas derivadas del histórico del usuario. Si el repo es público en GitHub queda
  indexado — verificar visibilidad en Settings → Danger Zone y poner en privado si aplica.
  Ya se confirmó que el usuario es el único con acceso al proyecto de Vercel, así que no fue
  necesario rotar `ANTHROPIC_API_KEY` pese a la advertencia de "Sensitive" de Vercel.
- ⚠️ **[MEDIO, pendiente]** `proxy.js` reenvía errores crudos de Anthropic al cliente — bajo impacto,
  innecesario exponer detalles internos.
- ℹ️ **[BAJO, pendiente]** Parseo de JSON de respuesta frágil (`indexOf("{")`/`lastIndexOf("}")`,
  sin `try/catch`). Recomendado migrar a `tool_use` estructurado de la API de Anthropic — cada vez
  más relevante ahora que el output tiene más campos (`alerta`, y en `extractSaldo` el `saldo_total`).
- ℹ️ **[BAJO, pendiente]** `xlsx.full.min.js` cargado desde CDN sin atributo `integrity` (SRI).
- XSS descartado en el vector sospechado (texto de transacciones nunca toca el DOM); hay un
  auto-XSS trivial en el campo de nombre de tarjeta custom (el propio usuario lo teclea) — impacto
  nulo, no priorizado.

## Bug de año incorrecto en fechas extraídas (Nu Bank) — mitigado

Durante la Fase 1 se detectó que ~22% de las filas nuevas de archivos `N_` (Nu Bank) traían el año
de la transacción 1-2 años por debajo del real. Día y mes siempre correctos, solo el año fallaba.

**Causa raíz identificada:** el prompt le pedía a Claude "año inferido del contexto" en vez de
darle el año real que el usuario ya seleccionó en el formulario.

**Fix aplicado (PR #3):** el prompt ahora recibe el año esperado de la quincena (`state.year`) y
le indica a Claude que lo use salvo que el screenshot muestre explícitamente otro año — en ese
caso debe marcar `alerta`. Esto ataca la causa raíz (ya no hay que "inferir" nada), pero **no se ha
confirmado con un lote real de Nu Bank todavía** — vigilar en el próximo uso si el badge de alertas
marca algo raro en esa tarjeta específicamente.

## Fase 1 — Consolidación histórica (completada 2026-07-10/12)

Unió los exports de la app (30 archivos, 537 filas) con el histórico manual (2080 filas) en un
archivo limpio y organizado.

**Decisiones tomadas:**
- Transacciones "Anulado": se emparejaron con su cargo original y se excluyó el par completo (4
  pares, 8 filas). 9 quedaron sin par (holds de $1, sin cargo positivo) — se mantienen con
  `Estatus='Anulado'`.
- 1 fila duplicada de alta confianza (ya en el histórico manual) — excluida. 1 hold duplicado
  entre 2 archivos con ventanas de quincena solapadas — se dejó solo 1 ocurrencia.
- Bug de fecha: se corrigieron 120 filas usando la quincena esperada según el nombre del archivo.
- Clasificación automática: diccionario de 649 keywords (`data/reglas_clasificacion.json`) desde
  las 2080 filas ya clasificadas a mano. Cobertura ~78-80% con confianza ≥70%. **Usuario y Transc
  Type NO son inferibles por texto** (sesgo de una sola persona ~88-93% de las filas; dirección de
  viaje no depende del merchant).
- Limpieza de bloat: fórmulas y AutoFilter arrastrados hasta la fila 1,048,527 (cruft, no volumen
  real) inflaban el Excel a 20.5MB. Recortado a buffer de 10,000 filas → 3.1MB.
- Reorganización de carpetas: todo a `OneDrive/Registro TDC/` (ver tabla abajo).

**Resultado:** 2607 filas totales en el archivo activo, ya limpio y organizado.

## Fase 2 — Automatización end-to-end

**Completado y en producción (PR #3, PR #4 — 2026-07-12):**
- Clasificación automática integrada directamente en la app (client-side, usa
  `data/reglas_clasificacion.json`, matching longest-match-wins).
- Hashkey (FNV-1a) por transacción para detectar duplicados dentro del mismo lote y contra
  tarjetas ya procesadas en la sesión.
- Fix a la causa raíz del bug de fecha (ver arriba) — pendiente de confirmar con uso real.
- Validación opcional de saldo total: screenshot de estado de cuenta → comparación contra la suma
  de transacciones extraídas → badge de resultado.
- Export del xlsx con las mismas 14 columnas del histórico maestro (bridge hacia auto-append real
  — pegar el bloque A:N ya no requiere reconciliar columnas).
- Resumen visual de alertas (badge verde/amarillo) antes de descargar.

**Pendiente (sin empezar):**
- **Usuario como input explícito**: no es inferible por texto. Idea validada pero no construida:
  selector "¿quién trae esta tarjeta hoy?" por lote/tarjeta (no por transacción), con opción de
  excepción manual puntual.
- **Auto-append real vía Microsoft Graph API**: el bridge actual sigue siendo copiar/pegar manual.
  El auto-append sin ningún paso manual requiere registrar una app en Azure AD y usar la Excel REST
  API de Graph para escribir directo a la tabla en OneDrive — no se hizo por el setup de Azure, que
  no es viable en una sola sesión. Probablemente conviene que requiera confirmación explícita del
  usuario antes de escribir (no 100% automático), dado que Usuario y ~20% de Clasificación siguen
  necesitando ojo humano.
- **Parseo estructurado**: migrar de texto libre (`indexOf`/`lastIndexOf` + `JSON.parse`) a
  `tool_use` de la API de Anthropic — cada vez más valioso porque el output ya tiene más campos.
- **Segundo barrido de clasificación**: regenerar `reglas_clasificacion.json` cuando el histórico
  crezca con las filas ya revisadas de esta sesión, para mejorar cobertura más allá del ~78-80%.

## Fase 3 — Dashboard / análisis de gastos (para el final, no iniciada)

**Recomendación: Power BI Desktop**, conectado directo al Excel maestro (ya limpio, 3.1MB).

**Por qué:** gratis para uso personal/local (solo se paga si se comparte vía Power BI Service, no
aplica acá); el usuario ya sabe Power BI + DAX + pandas/Excel — cero curva de aprendizaje; conector
nativo a Excel con Power Query para limpiar datos sin tocar el archivo fuente; el volumen de datos
es trivial (2607 filas, ~500-1000/año) para cualquier herramienta seria. Refresh manual al abrir es
suficiente dado que los datos se actualizan por quincena, no en tiempo real.

**Alternativas descartadas:** Google Sheets+Looker Studio (saca de Excel, DAX más débil), Streamlit
Community Cloud (mantener un servicio vivo — revisar solo si Power BI no alcanza para
visualizaciones custom de las alertas), Metabase self-host (infra propia no justificada),
PivotTables nativas (parche rápido, no dashboard real).

**Pendiente explícito del usuario:** quiere poder ver el dashboard **sin necesidad de la laptop**
(ej. desde el celular). Power BI Desktop por sí solo no lo resuelve — evaluar en su momento
publicar a **Power BI Service** (tiene capa gratuita para que el dueño del reporte lo vea desde la
app móvil sin compartir con nadie más) en vez de quedarse solo con Desktop. No investigado a fondo
todavía — queda como primera pregunta a resolver cuando se retome Fase 3.

## Checklist de handover (actualizado 2026-07-12, fin de sesión)

**Hecho hoy:**
- [x] Fusión de las 527 filas nuevas al histórico (2607 filas totales).
- [x] Limpieza del bloat del Excel madre (20.5MB → 3.1MB).
- [x] Reorganización de carpetas en OneDrive (`Registro TDC/`).
- [x] Fix crítico de seguridad del proxy — PR #2, verificado en producción.
- [x] Clasificación automática + hashkey de duplicados + fix de causa raíz del bug de fecha +
      export listo para pegar — PR #3, verificado en producción.
- [x] Validación opcional de saldo total — PR #4, verificado en producción.
- [x] Recomendación de stack de dashboard (Power BI Desktop) documentada.
- [x] PR #1 (este archivo + `data/reglas_clasificacion.json`) — mergeado a `main`.

**Pendiente para la próxima sesión:**
- [ ] **Probar con datos reales** (el usuario terminó la sesión de hoy sin haber corrido todavía
      un lote real completo con las 4 features nuevas juntas) — poner atención especial a: si la
      clasificación automática acierta, si el badge de saldo funciona con un estado de cuenta real,
      si el bloque exportado pega limpio en el maestro, y si el bug de fecha de Nu Bank ya no
      aparece (o si aparece marcado como alerta en vez de silencioso).
- [ ] Llenar a mano la columna Usuario de las 527 filas fusionadas en Fase 1 (no automatizado).
- [ ] Diseñar y construir captura de Usuario en la app (selector por tarjeta/lote).
- [ ] Auto-append real vía Microsoft Graph API (requiere registro de app en Azure AD).
- [ ] Migrar parseo de respuesta de Claude a `tool_use` estructurado.
- [ ] Segundo barrido de clasificación cuando el histórico tenga más filas revisadas.
- [ ] Hallazgos de seguridad medios/bajos sin resolver (confirmar visibilidad del repo en GitHub;
      mover `reglas_clasificacion.json` si es público; errores crudos del proxy; SRI del CDN).
- [ ] Fase 3 (dashboard): resolver si Power BI Desktop solo o Power BI Service (para verlo sin
      laptop) antes de empezar a construirlo.

## Archivos y rutas clave

| Qué | Dónde |
|---|---|
| Histórico maestro (**activo**, ya limpio) | `OneDrive/Registro TDC/Gastos Mensuales - AJMO.xlsx` |
| Versiones anteriores (no borrar) | `OneDrive/Registro TDC/Versiones anteriores/` |
| Exports de la app (2026) | `OneDrive/Registro TDC/Registro Quincenas/2026/` |
| Consolidado solo-2026 (referencia, previo a fusión) | `OneDrive/Registro TDC/Registro Quincenas/Consolidado 2026.xlsx` |
| Este repo (clonado) | `C:\Aleja\Repo\AJMO REPO\Registro tarjetas\` |
