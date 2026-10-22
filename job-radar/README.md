# Radar de empleo — evaluación automática de ofertas con LLM

Flujo en producción que revisa ofertas de empleo publicadas cada día, las puntúa con un modelo de lenguaje contra una rúbrica definida por mí, y me avisa por Telegram solo cuando algo pasa el filtro. Corre solo desde el 6 de octubre de 2026. 246 ofertas procesadas, ~USD 0,06 al día.

---

## El problema

Las vacantes de automatización bien encajadas para LATAM remoto se cierran en 24 a 48 horas. Lo comprobé a la mala: cuatro ofertas que me interesaban se cerraron antes de que alcanzara a postularme.

Revisarlas a mano significaba abrir seis búsquedas de LinkedIn todas las mañanas y leer unas 60 ofertas, de las cuales sobrevivían dos o tres. **Línea base: alrededor de una hora diaria, con resultado irregular según qué tan temprano abriera el computador.**

El objetivo no era automatizar la búsqueda. Era reducir el tiempo entre que una oferta se publica y yo decido si vale la pena, a menos de un minuto de lectura al día.

## Arquitectura

```
[Cron 6:47 a.m. America/Bogota]
            ↓
[Apify · LinkedIn Jobs Scraper]      6 búsquedas, últimas 72 h, 10 resultados c/u
            ↓
[Normalizar · deduplicar · prefiltrar]   dedup por empresa+cargo, extracción de salario
            ↓
[Data table: ¿es nueva?] ──── no ──→ descartada
            ↓ sí
    ┌───────┴────────┐
    ↓                ↓
[Guardar fila]  [Agrupar las nuevas]
                     ↓
            [Construir el prompt]        perfil + criterios de descarte + rúbrica
                     ↓
            [LLM · modo JSON]            puntúa 0–100 y explica
                     ↓
            [Formatear el aviso]         top 5 con score ≥ 60
                     ↓
            [Telegram]                   si no hay nada nuevo, no envía
```

## Integraciones

| Servicio | Para qué | Cómo se conecta |
|---|---|---|
| Apify | Lee las ofertas publicadas en LinkedIn | Nodo oficial, actor del store |
| n8n Data Table | Memoria de lo ya visto y archivo histórico | Nodo nativo |
| API de OpenAI | Puntúa y explica cada oferta | HTTP POST, `response_format: json_object` |
| API de Telegram | Entrega el resultado donde sí lo leo | Nodo nativo, HTML |

## Decisiones de diseño

**No construí un scraper propio.** Mantener un scraper contra LinkedIn es trabajo indefinido: cambian el DOM, agregan muros de sesión, bloquean IPs. El actor del store cuesta USD 0,06 al día y el mantenimiento es de otro. Si el costo subiera un orden de magnitud, la decisión cambiaría.

**Deduplico por empresa + cargo, no solo por URL.** La primera versión usaba la URL. LinkedIn publica la misma vacante bajo URLs distintas para cada país, así que la misma oferta entraba tres veces y el aviso diario se llenaba de repetidos.

**El filtro por palabras clave no sobrevivió al contacto con la realidad.** La v1 puntuaba con expresiones regulares sobre el texto. Resultado: premiaba cualquier descripción que mencionara muchas herramientas, aunque el rol no tuviera nada que ver. Un aviso de ingeniería de datos que nombraba Python, API y Docker sacaba mejor puntaje que un rol de automatización bien encajado. Hoy ese puntaje sobrevive solo como prefiltro barato para no mandarle basura al modelo; **la decisión real la toma el LLM leyendo la descripción.**

**El modelo responde en JSON estricto, con temperatura 0,1.** El flujo necesita parsear la respuesta, no leerla. Un modelo pequeño basta: la tarea es clasificar contra criterios explícitos, no razonar de forma abierta.

**Si no hay ofertas nuevas, no mando nada.** Una notificación diaria que a veces dice "nada hoy" se vuelve ruido y se deja de leer. El silencio es información.

## Manejo de errores

- **Reintentos** en los tres nodos que dependen de una red ajena: Apify, OpenAI y Telegram, con dos intentos cada uno.
- **La llamada al LLM no tumba la ejecución.** Está en `continueRegularOutput` con timeout de 60 segundos: si OpenAI falla, el flujo sigue y las ofertas igual quedan guardadas en la tabla.
- **El parseo de la respuesta del modelo va en try/catch.** Un modelo puede devolver algo que no es JSON válido, aunque se le pida lo contrario. Si pasa, el aviso llega igual y me dice explícitamente que el filtro falló y revise la tabla a mano. Prefiero un aviso degradado a ningún aviso.
- **Idempotencia** por la verificación contra la data table antes de insertar: una ejecución repetida no duplica filas ni reenvía ofertas.
- **El mensaje se corta a 4.000 caracteres**, por el límite de la API de Telegram.

## Calibración, que es donde estuvo el trabajo real

Construir el flujo tomó una tarde. Hacer que el criterio fuera confiable tomó dos semanas y tres correcciones, cada una originada en un falso positivo o un falso negativo concreto:

| Fecha | Qué falló | Qué cambié |
|---|---|---|
| 6-oct | El puntaje por palabras clave premiaba el ruido | Scoring con LLM leyendo la descripción completa |
| 8-oct | Dejó pasar dos roles senior de ingeniería de software | Criterios de descarte explícitos por título y por años de experiencia |
| 22-oct | Una oferta pasó el filtro y resultó pagar menos de la mitad del piso | Ver abajo |

El tercero es el más instructivo. La descripción se estaba cortando a 1.500 caracteres antes de llegar al modelo, y yo asumí que el recorte venía del actor. **No: lo hacía mi propio nodo de normalización.** Como el salario, la jornada y las condiciones van casi siempre al final de un aviso, el modelo nunca los veía y no podía filtrar por ellos.

La corrección fue conservar **inicio y final** —1.500 caracteres de arriba, 900 del cierre, con un marcador de recorte en medio— en lugar de cortar a ciegas, más un extractor de salario que lo busca en el texto cuando el scraper no lo entrega como campo. Lo probé contra seis casos antes de publicarlo: monto mensual en dólares, rango anual, tarifa por hora, pesos colombianos, salario ya provisto por el actor, y ausencia de salario.

```
Salario al final del aviso  → Compensation: $1000 - $1600 per month.
Rango anual en el cuerpo    → Salary: USD 48,000 - 58,000 per year
Tarifa por hora             → Pay rate: $30 - $45 per hour
Pesos colombianos           → Salario: $8.000.000 - $12.000.000 mensuales
Sin salario publicado       → (vacío, y el aviso lo señala)
```

## Resultado

| | Antes | Ahora |
|---|---|---|
| Tiempo diario | ~1 hora revisando a mano | ~30 segundos leyendo un mensaje |
| Cobertura | Irregular, según la hora | 6 búsquedas, todos los días a las 6:47 |
| Criterio | En mi cabeza, inconsistente | Rúbrica explícita, versionada y corregible |
| Costo | — | ~USD 0,06/día de scraping + centavos de LLM |

246 ofertas procesadas desde el 6 de octubre. Un promedio de 12 nuevas por día, de las cuales pasan el filtro entre cero y tres.

## Qué cambiaría

**Verificar que la oferta siga abierta antes de avisar.** Es la mejora de mayor impacto y todavía no está: cuatro ofertas llegaron al aviso y ya estaban cerradas cuando fui a postularme. Un `HEAD` a la URL detectaría la redirección que LinkedIn hace cuando retira una vacante.

**Salir de una sola fuente.** Todo depende de LinkedIn, que además bloquea la lectura directa de las páginas. Agregar portales que sí se dejan leer daría mejores descripciones y menos dependencia.

**Medir la calidad del filtro contra mi propio criterio, de forma sistemática.** Hoy la calibración es reactiva: corrijo cuando noto un error. Guardar mi veredicto humano junto al del modelo permitiría medir la tasa de acuerdo y saber si el prompt mejora o empeora con cada cambio, en vez de suponerlo.

---

## Nota

Este repositorio documenta la arquitectura del sistema. Las credenciales, los identificadores de recursos y los criterios personales del perfil con que puntúa no se incluyen.
