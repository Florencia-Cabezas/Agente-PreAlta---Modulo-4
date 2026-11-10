# Agente de Pre-Alta de Comercios · Checkpoint 4 · Integraciones

Proyecto integrador del curso de AI Automation (n8n). Agente multi-agente para una procesadora de pagos **ficticia (RedCobro)** que recibe solicitudes de alta de comercios. En este módulo se conecta con tres herramientas externas reales mediante **OAuth2**:

| Rol | Herramienta | Uso en el flujo |
|---|---|---|
| CRM | **HubSpot** | Lectura (búsqueda de contacto) y escritura (crear / actualizar contacto) |
| Casilla de correo | **Gmail** | Lectura (trigger de emails entrantes) y escritura (borrador de respuesta) |
| Canal del equipo | **Slack** | Aviso al canal `#operaciones-altas` |

Archivo del workflow: `checkpoint4_florencia_cabezas.json`

## Credenciales (OAuth2)

![Credentials de n8n con las 3 conexiones activas](Modulo%204%20-%20Credenciales.png)

Las tres credenciales usan **OAuth2** (Managed OAuth2 de n8n). El `.json` exportado no incluye tokens: al importarlo, hay que reasignar las credenciales propias en los nodos de HubSpot, Gmail y Slack.

## Arquitectura

El workflow parte del Módulo 3 (chat con memoria + Router + Workers) y suma un **canal de entrada por email** dentro del mismo workflow:

```
[Gmail_Entrante]  trigger, filtro: subject:"[RedCobro]" is:unread
      │
      ▼
① [Es_AutoReply]  IF (OR) ─ true ─▶ [Stop_AutoReply]   corta el bucle infinito
      │ false
      ▼
④ [Limpiar_Email]  Set: from_email, from_name, subject, body_text (máx. 3000 caracteres), message_id, thread_id
      │
      ▼
   [Email_Valido]  Switch: from_email contiene "@" ─ Fallback ─▶ [Descartar_Sin_Email]   evita el Error 400
      │ VALIDO
      ▼
   [Agente_Email]  AI Agent (Claude Sonnet) + Structured Output Parser
      │            clasifica (taxonomía cerrada), extrae datos y redacta borrador
      ▼
② [Buscar_Contacto]  HubSpot · Search por email (Look up)
      ▼
   [Contacto_Existe]  IF
      ├─ true  ─▶ [Actualizar_Contacto]  HubSpot · Update (solo Company Name)
      └─ false ─▶ [Crear_Contacto]       HubSpot · Create (Email, First Name, Company, Phone)
      ▼
③ [Crear_Borrador]  Gmail · Create Draft en el mismo hilo   Human-in-the-loop
      ▼
④ [Resumen_Slack]  Set: un único campo de texto corto
      ▼
   [Avisar_Slack]  Slack · Send message a #operaciones-altas
```

![Lienzo del canal email ejecutado](Modulo%204%20-%20Lienzo%20Flujo%20OK.png)

## Guardrails (los 4 nodos de la rúbrica)

| # | Nodo | Qué previene | Cómo |
|---|---|---|---|
| ① | **Es_AutoReply** | Bucle infinito de auto-respuestas | IF en modo OR, sin distinguir mayúsculas: el asunto contiene *Auto-reply*, *Out of office*, *Undeliverable* o *Respuesta automática*, o el remitente contiene *no-reply*, *noreply* o *mailer-daemon*. Si se cumple, el flujo termina en Stop_AutoReply. |
| ② | **Buscar_Contacto + Contacto_Existe** | Error 409 (contactos duplicados) | Búsqueda por email antes de escribir: si existe se actualiza, si no se crea. |
| ③ | **Crear_Borrador** | Envíos sin supervisión | Gmail en modo **Create Draft**: la respuesta queda en Borradores hasta que una persona la revisa y la envía. |
| ④ | **Limpiar_Email / Email_Valido / Resumen_Slack** | Error 400 y "pasillo de la muerte" | Set que deja solo los campos necesarios en texto plano, recorte del cuerpo del mail, validación del email antes de HubSpot y un único texto corto hacia Slack. |

**Otros controles:**
- **Mínimo privilegio:** HubSpot escribe solo 4 propiedades al crear y 1 al actualizar; Slack solo publica en un canal.
- **Límite de iteraciones:** el agente corre con máximo 3.
- **Taxonomía cerrada:** ALTA_COMERCIO, CONSULTA_ESTADO, ESCALAR u OTRO.
- **Contenido del email delimitado:** el mail va entre marcadores y el prompt aclara que no son instrucciones (protección contra prompt injection).

## Pruebas de regresión

| Prueba | Email | Resultado |
|---|---|---|
| A · Alta nueva | `[RedCobro] Quiero dar de alta mi comercio` | Contacto **creado** en HubSpot, borrador en Gmail y aviso en Slack |
| B · Auto-reply | `[RedCobro] Out of office - respuesta automática` | Cortado en **Stop_AutoReply**: sin cambios en HubSpot, Gmail ni Slack |
| C · Contacto repetido | Mismo email de la prueba A | Rama **Actualizar_Contacto**: HubSpot sigue con 1 solo contacto; Slack informa "actualizado (ya existía)" |

![Contacto en HubSpot](Modulo%204%20-%20Hubspot.png)
![Borrador en Gmail](Modulo%204%20-%20Draft%20gmail.png)
![Aviso en Slack](Modulo%204%20-%20Slak.png)
![Prueba B: auto-reply cortado](Modulo%204%20-%20Respuesta%20automatica.png)
![Prueba C: contacto actualizado, sin duplicar](Modulo%204%20-%20Actualizar.png)

## Cómo importarlo

1. En n8n: **Workflows → Import from File** y elegí `checkpoint4_florencia_cabezas.json`.
2. Creá o reasigná las credenciales OAuth2 de **HubSpot**, **Gmail**, **Slack**, **Google Sheets** y **Anthropic** en los nodos correspondientes.
3. Los sub-workflows del Módulo 2 (Worker_Registro_Alta y Worker_Consulta_Estado) son necesarios solo para el canal de chat.

> Todos los datos de prueba son ficticios. Proyecto con fines educativos.
