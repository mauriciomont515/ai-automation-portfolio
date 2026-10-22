# Portafolio de automatización con IA — Mauricio Montes

Construyo y mantengo sistemas de automatización con IA en producción: flujos que conectan CRMs, bases de datos y APIs de modelos de lenguaje, con manejo de errores, reintentos y monitoreo. Este repositorio documenta algunos de ellos: qué problema resolvían, cómo están armados y cómo medí que funcionaran.

No son demos. Son sistemas que corren solos y que me toca arreglar cuando se rompen.

## Sistemas

| Sistema | Qué hace | Stack |
|---|---|---|
| [Radar de empleo](job-radar/) | Revisa ofertas publicadas cada día, las puntúa con un LLM contra una rúbrica y avisa solo cuando algo pasa el filtro | n8n · Apify · OpenAI · Telegram |

## Cómo está documentado cada uno

Cada sistema tiene el mismo esqueleto, porque son las preguntas que yo quiero tener respondidas cuando vuelvo a un flujo seis meses después:

- **El problema** y la línea base antes de automatizar.
- **La arquitectura**, de disparador a sistema destino.
- **Las integraciones** y por qué cada una.
- **Las decisiones de diseño**, incluido lo que decidí *no* construir.
- **El manejo de errores**: reintentos, idempotencia, qué pasa cuando el modelo devuelve basura.
- **La calibración**: qué falló en cada versión y qué cambié.
- **El resultado medido** y qué cambiaría hoy.

## Stack

**Automatización:** n8n, Make, Zapier
**IA y LLM:** API de OpenAI y Anthropic, diseño y calibración de prompts, rúbricas de evaluación, Whisper, modelos de visión
**Integraciones:** API REST, webhooks, OAuth 2.0, Meta Graph API, WhatsApp Business API, HubSpot, Shopify, Apify
**Datos:** SQL, Python, JavaScript, Power BI, Streamlit
**Infraestructura:** Docker, Cloudflare Zero Trust

## Contacto

Cali, Colombia (UTC-5) · [LinkedIn](https://linkedin.com/in/mauricio-montes-2a23aa42b) · mauriciomontes798@gmail.com
