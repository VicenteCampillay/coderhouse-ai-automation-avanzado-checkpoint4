# Checkpoint 4 — Integraciones Avanzadas e Interconexión de Sistemas

Proyecto integrador de Automatización con IA (CoderHouse), evolución acumulativa de los checkpoints 2 y 3.

**Autor:** Vicente Campillay | **Herramienta:** n8n (self-hosted)

---

## 1. Qué es este proyecto

Sistema multi-agente (arquitectura Manager-Worker) que automatiza la recepción, validación y resolución de solicitudes de horas extra. Es la bajada práctica del ecosistema e-commerce del vivo (tienda → CRM → soporte → Slack), con roles equivalentes: **HubSpot** = CRM del solicitante, **Gmail** = casilla de soporte, **Slack** = canal del equipo.

## 2. Archivos y continuidad entre checkpoints

| Archivo | Rol |
|---|---|
| `checkpoint4_vicente_campillay.json` | Orquestador principal: entrada multicanal, guardrails, clasificación, CRM, notificaciones. |
| `Checkpoint2_WorkerUno...json` | Valida la solicitud contra la política de horas extra (aprueba/escala/rechaza/pide dato). |
| `Checkpoint2_WorkerDos...json` | Formatea el resultado y crea el borrador de correo (HITL). |
| `Checkpoint3_WorkerTres...json` | Agente de consultas (políticas, historial, proyectos) usando herramientas. |

Los 3 workers corren como sub-workflows (`Execute Workflow`) invocados por el orquestador. **El archivo de este checkpoint no arrancó de cero:** es el mismo workflow del Checkpoint 3 (que ya traía memoria de sesión y el Worker Tres integrado y funcionando) al que se le sumaron las integraciones y controles de este checkpoint 4. Worker Tres se incluye acá como sub-workflow independiente para poder probarlo aislado, pero su integración real ya vivía en el orquestador desde antes.

## 3. Qué agrega el checkpoint 4

**Tres conectores reales:**

| Conector | Nodo(s) | Rol |
|---|---|---|
| Gmail (trigger) | `Gmail Trigger` | Canal de entrada por correo |
| HubSpot (CRM) | `Buscar/Actualizar/Crear Contacto CRM` | Registro del solicitante |
| Slack | `Notificar a Slack (Operaciones)` | Aviso al equipo |

**Cuatro controles no-code:**

| # | Control | Nodo(s) | Evita |
|---|---|---|---|
| ① | IF anti-auto-reply tras el trigger de correo | `If2` → `NoOp` | Bucle infinito de auto-respuestas |
| ② | Look up antes de Create en el CRM | `¿Hay Email?` → `Look up` → `¿Existe?` → Update/Create | Error 409 (duplicados) |
| ③ | Create Draft (HITL) | `Crear Borrador de Correo (HITL)` en Worker Dos | Envío sin revisión humana |
| ④ | Set de limpieza antes de Slack | `Set - Limpiar Payload Slack` | Saturar el canal con payload pesado |

El email del solicitante se limpia en `Edit Fields`, viaja por `Detectar Canal` → contexto de sesión → `Guardrail de Confianza` (fuente única para todo lo que sigue), se usa en el look up del CRM, y se propaga a Worker Dos para que el borrador se dirija al solicitante real.

## 4. Credenciales

Gmail, Google Sheets: OAuth2. Airtable: Token API. Anthropic/Groq: API Key. **HubSpot y Slack usan un método distinto a OAuth2** — detalle abajo.

**HubSpot (App Token / Service Key):** se intentó OAuth2, pero requiere un public app, y HubSpot deshabilitó su creación por interfaz gráfica el 23/06/2026, migrándola a un CLI (`hs project create`). Se usó un Service Key con scopes mínimos (`crm.objects.contacts.read/write`). [Fuente](https://developers.hubspot.com/changelog/legacy-public-app-creation-sunset)

**Slack (Access Token):** se configuró OAuth2 correctamente, pero n8n devolvió repetidamente `Error: Insufficient parameters for OAuth2 callback` — bug reportado en el propio repo de n8n, no específico de esta instancia. [Issue #17598](https://github.com/n8n-io/n8n/issues/17598). Se usó Access Token (Bot User OAuth Token) como alternativa soportada, con estos permisos (`channels:read ;chat:write ; groups:read`).
