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

## Flujo actual (manual, pre-Fase 2)

1. El usuario toma screenshots de los movimientos de su tarjeta desde el celular.
2. Abre la app, selecciona **Quincena (Q1/Q2) + Mes + Año + Fecha de pago**.
3. Sube los screenshots; Claude (vía `api/proxy.js`) extrae: Fecha, Hora (a veces), Descripción,
   Monto, Banco, Status (Normal/Anulado), y genera el **Código de Pago**.
4. La app genera un `.xlsx` por quincena+tarjeta, con nombre tipo
   `{TARJETA}_{CODIGO}.xlsx` (ej. `N_Q1ABR26N190426.xlsx`).
5. El usuario descarga el xlsx y lo guarda en `OneDrive/Registro TDC/Registro Quincenas/{año}/`.
6. **Manualmente** llena en el histórico maestro (`OneDrive/Registro TDC/Gastos Mensuales - AJMO.xlsx`,
   hoja "Control de Gastos") las columnas que la app no captura: Usuario, Clasificación, Detalle,
   Transc Type (Ida/Regreso), Estatus de pago.

### Patrón del Código de Pago

`Q[1|2][MES][YY][TARJETA][DDMMYY]` — ej. `Q2FEB26AMEX060326` = quincena 2, febrero 2026, Amex,
fecha de pago 06/03/26.

**Importante:** este código es solo un identificador para buscar la transacción en el banco (no
está diseñado para calzar con los códigos que el usuario escribía a mano en el histórico antes de
que existiera la app). No usar como llave de deduplicación entre histórico viejo y exports nuevos —
para eso comparar por **Fecha + Monto + similitud de texto + Tarjeta**.

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
  sin `try/catch`). Recomendado migrar a `tool_use` estructurado de la API de Anthropic cuando se
  agreguen más campos al output (Usuario, flags de fecha/duplicados — ver Fase 2).
- ℹ️ **[BAJO, pendiente]** `xlsx.full.min.js` cargado desde CDN sin atributo `integrity` (SRI).
- XSS descartado en el vector sospechado (texto de transacciones nunca toca el DOM); hay un
  auto-XSS trivial en el campo de nombre de tarjeta custom (el propio usuario lo teclea) — impacto
  nulo, no priorizado.

## Bug conocido: año incorrecto en fechas extraídas (Nu Bank)

Durante la Fase 1 se detectó que ~22% de las filas nuevas de archivos `N_` (Nu Bank) traían el año
de la transacción 1-2 años por debajo del real (ej. código dice quincena de abril **2026**, pero la
fecha extraída dice marzo **2024**). Día y mes siempre correctos, solo el año falla. Es sistemático
en algunos archivos (5 de 7 analizados), no en todos.

**Hipótesis:** error de Claude al leer el año en el screenshot de Nu Bank específicamente (quizás
formato de fecha en la UI de esa app bancaria es ambiguo o se corta). El prompt actual le pide a
Claude "año inferido del contexto" en vez de pasarle el Quincena+Mes+Año que el usuario ya
seleccionó en el formulario — probable causa raíz.

**Pendiente Fase 2:** agregar validación en `api/proxy.js` — comparar el año extraído contra la
ventana esperada (Quincena+Mes+Año del formulario, que el frontend ya conoce) y flaggear (no
autocorregir silenciosamente) si se aleja demasiado.

## Fase 1 — Consolidación histórica (completada 2026-07-10/12)

Objetivo: unir los exports de la app (30 archivos, 537 filas) con el histórico manual (2080 filas)
en un solo archivo, y dejar la estructura de carpetas ordenada para lo que sigue.

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
- **Limpieza de bloat**: el Excel madre tenía fórmulas (`Data checkup`, `Pago Pendiente`) y un
  AutoFilter arrastrados hasta la fila 1,048,527 (límite de Excel) — puro cruft de cuando alguna vez
  se arrastró una fórmula de más, no volumen de datos real. Esto inflaba el archivo a 20.5MB. Se
  recortó a un buffer de 10,000 filas (suficiente para ~10 años de captura al ritmo actual),
  quedando en 3.1MB.
- **Reorganización de carpetas**: todo el proyecto de datos se movió a `OneDrive/Registro TDC/` (ver
  tabla de rutas abajo), separando el archivo activo de las versiones anteriores (no borradas).

**Resultado:** 527 filas nuevas netas (537 − 10 excluidas) fusionadas con el histórico → 2607 filas
totales en el archivo activo, ya limpio y organizado.

**Artefactos reutilizables para Fase 2** (en este repo, carpeta `data/`):
- `reglas_clasificacion.json` — diccionario keyword → Clasificación con nivel de confianza. Se
  puede usar como primer clasificador automático dentro de la propia app (en vez de solo en la
  consolidación histórica).

## Fase 2 — Automatización end-to-end (pendiente, no iniciada)

Objetivo declarado por el usuario: que desde el registro del screenshot se vaya actualizando el
histórico automáticamente y se clasifiquen los gastos "de alguna manera automática", sobre todo el
tipo de transacción (Clasificación). Además, agregar:

- **Validación de saldo total por tarjeta**: permitir subir un screenshot del estado de cuenta/saldo
  total del banco, y que la app compare ese total contra la suma de las transacciones extraídas de
  la quincena, avisando si no cuadra.
- **Hashkey por transacción para detectar duplicados**: a veces (screenshots repetidos por error del
  usuario) la misma transacción queda capturada dos veces. Calcular un hash determinístico (ej.
  Fecha+Monto+Descripción normalizada+Banco) al momento de extracción, para detectar duplicados
  dentro del mismo lote y avisar antes de exportar/appendear. Ya hubo un caso real de esto en Fase 1
  (hold de -$85 repetido entre dos archivos de quincenas con ventana solapada).
- **Claude como revisor dentro del app**: capa de comentarios/alertas proactivas sobre eventualidades
  (fecha sospechosa, posible duplicado, discrepancia de saldo, clasificación de baja confianza) —
  idealmente generada en el mismo call de extracción (agregando instrucciones al prompt) en vez de
  una llamada aparte.

**Ideas discutidas (sin decisiones cerradas todavía):**
- Clasificación: aplicar `data/reglas_clasificacion.json` **server-side** en `api/proxy.js` (no en
  el frontend), matching "longest-match-wins", mostrando sugerencia editable con badge de confianza
  en la UI antes de exportar.
- Usuario: selector "¿quién trae esta tarjeta hoy?" **por lote/tarjeta**, no por transacción (menor
  fricción); permitir excepción manual puntual si hace falta.
- Auto-append al histórico maestro: dado que vive en OneDrive (no en DB), evaluar Microsoft Graph
  API (Excel REST API, append a tabla) en vez de migrar a base de datos — no está justificado por
  volumen de datos (ver Fase 3). Probablemente requiere un paso de confirmación explícita del
  usuario antes de escribir, no 100% automático, dado que el bug de fecha y el ~20% de clasificación
  sin match todavía necesitan ojo humano.
- Parseo de respuesta de Claude: migrar de texto libre a `tool_use` estructurado (ver Seguridad
  arriba) — necesario de todas formas si se agregan más campos al output (Usuario, flags, hashkey).

## Fase 3 — Dashboard / análisis de gastos

**Recomendación: Power BI Desktop**, conectado directo al Excel maestro (ya limpio, 3.1MB).

**Por qué:**
- Gratis para uso personal/local — Power BI Desktop no requiere licencia; solo se paga si se
  publica a Power BI Service para compartir con otras personas, que no es el caso acá (uso de un
  solo usuario).
- El usuario ya sabe Power BI + DAX + pandas/Excel — cero curva de aprendizaje, reutiliza
  herramientas que ya domina en vez de aprender un stack nuevo.
- Conector nativo a Excel (`Get Data from Excel`), incluye Power Query para transformar/limpiar los
  datos antes de graficar (ej. filtrar filas "Revisar" a su propia vista, normalizar el espacio
  sobrante en "Rappi Card ", etc.) sin tocar el archivo fuente.
- El volumen de datos es trivial para esta herramienta: 2607 filas hoy, creciendo
  ~500-1000/año — en 10 años seguirían siendo unas 10-15K filas, muy por debajo de cualquier límite
  práctico de Power BI (o de cualquier herramienta, en realidad).
- Refresh manual al abrir el archivo es suficiente (no hay necesidad de refresh programado ni de
  pagar Power BI Service) dado que los datos se actualizan por quincena, no en tiempo real.

**Alternativas consideradas y descartadas:**
- **Google Sheets + Looker Studio**: gratis, pero obliga a vivir en Sheets en vez de Excel
  (contradice mantener Excel como fuente de verdad), y el equivalente a DAX en Looker Studio es más
  débil — curva de reaprendizaje sin necesidad.
- **Streamlit Community Cloud** (dashboard en Python/pandas): gratis, encaja con el conocimiento de
  pandas del usuario, pero implica mantener una app/servicio vivo (repo, deploys) — más piezas
  móviles de las que se justifican para un dashboard personal. Revisar esta opción más adelante
  únicamente si Fase 2 termina necesitando visualizaciones muy custom (ej. para las alertas de
  Claude-revisor) que Power BI no cubra bien.
- **Metabase self-host**: gratis pero requiere correr y mantener un servidor (Docker) — overhead de
  infraestructura no justificado para uso personal.
- **PivotTables/gráficas nativas de Excel**: cero herramienta nueva, pero no da la experiencia de
  "dashboard" (sin refresh centralizado, menos pulido visual) — sirve de parche rápido, no como
  respuesta definitiva de Fase 3.

**Nota práctica:** cuando se arme el dashboard, conectar Power BI directo a la tabla "Control de
Gastos" del archivo activo (`OneDrive/Registro TDC/Gastos Mensuales - AJMO.xlsx`). Si más adelante
se quiere refresh automático sin tener el archivo abierto, existe el conector "OneDrive - Personal"
de Power BI — evaluarlo cuando/si se necesite, no de entrada.

## Checklist de handover (para retomar)

- [x] Fusión de las 527 filas nuevas al histórico — hecho, 2607 filas totales.
- [x] Limpieza del bloat del Excel madre (20.5MB → 3.1MB) — hecho.
- [x] Reorganización de carpetas en OneDrive (`Registro TDC/`) — hecho.
- [x] Fix crítico de seguridad del proxy (shared secret) — hecho, PR #2 mergeado y verificado en
      producción.
- [x] Recomendación de stack de dashboard (Power BI Desktop) — documentada arriba.
- [ ] **PR #1** (`feature/context-tdc-fase1`, este archivo + `data/reglas_clasificacion.json`) —
      sigue sin mergear a `main`. Revisar y aprobar.
- [ ] Llenar a mano la columna Usuario de las 527 filas nuevas (no es automatizable con lo que hay
      hoy — ver Fase 2 para plan de captura futura).
- [ ] Segundo barrido de clasificación: el diccionario actual no cubre todos los conceptos nuevos —
      cuando el histórico crezca con estas 527 filas ya revisadas, regenerar
      `reglas_clasificacion.json` con el set ampliado para mejorar cobertura.
- [ ] Definir plan de implementación concreto de Fase 2 (clasificación server-side, captura de
      Usuario, validación de fecha/saldo, hashkey de duplicados, Claude-revisor) — hoy solo hay
      ideas discutidas, falta secuenciar qué se construye primero.
- [ ] Hallazgos de seguridad medios/bajos sin resolver (ver sección Seguridad) — confirmar
      visibilidad del repo en GitHub, mover `reglas_clasificacion.json` si es público.

## Archivos y rutas clave

| Qué | Dónde |
|---|---|
| Histórico maestro (**activo**, ya limpio) | `OneDrive/Registro TDC/Gastos Mensuales - AJMO.xlsx` |
| Versiones anteriores (no borrar) | `OneDrive/Registro TDC/Versiones anteriores/` |
| Exports de la app (2026) | `OneDrive/Registro TDC/Registro Quincenas/2026/` |
| Consolidado solo-2026 (referencia, previo a fusión) | `OneDrive/Registro TDC/Registro Quincenas/Consolidado 2026.xlsx` |
| Este repo (clonado) | `C:\Aleja\Repo\AJMO REPO\Registro tarjetas\` |
