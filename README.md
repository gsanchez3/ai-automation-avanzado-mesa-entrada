# Checkpoint 4 · Mesa de entrada inteligente: integraciones con Gmail, Slack y HubSpot

Guido Sánchez · AI Automation Avanzado · Coderhouse 2026
Todos los datos (empresas, consultas, contactos) son ficticios.

## Archivos

| Archivo | Qué es |
|---|---|
| `checkpoint4_guido_sanchez.json` | Workflow Manager. **Es el entregable.** |
| `M3_Worker_Responder.json` | Sub-workflow que responde con la base de conocimiento. Sin cambios en M4. |
| `M3_Worker_Derivar.json` | Sub-workflow que deriva a un especialista y avisa por Slack. En M4 se corrigió su contrato de error. |
| `checkpoint1_guido_sanchez.json` | Entrega del Módulo 1 (agente base). No forma parte de esta entrega. |

Al importarlo, el workflow aparece con su nombre interno, `M4_Manager`.

## Qué hace

Una empresa escribe a la casilla de la Mesa de Entrada (Gmail) o por chat. El Manager descarta las respuestas automáticas, limpia el mail, lee la memoria de la empresa (Google Sheets), clasifica la consulta y la delega en un Worker: responder con la base de conocimiento o derivar a un especialista. Después registra la ejecución en Slack, actualiza el legajo de la empresa en HubSpot y deja la respuesta como **borrador** en Gmail. Nada se envía sin aprobación humana.

**Recorrido por Gmail:**
`entrada_gmail` → `es_autorespuesta` → `limpiar_email` → `datos_consulta` → memoria, clasificación y Workers (M2–M3) → `log_observabilidad` → `es_gmail` → `preparar_salida` → `email_valido` → `buscar_contacto` → `existe` → `actualizar_contacto` o `crear_contacto` → `crear_borrador` → `actualizar_ficha` → resumen (M3)

**Recorrido por chat:** entra por el mismo `datos_consulta`. En `es_gmail` sigue directo a `actualizar_ficha` y responde en el chat, sin CRM ni borrador.

## Controles

| # | Control | Nodos | Qué evita |
|---|---|---|---|
| 1 | Anti auto-reply | `es_autorespuesta` (IF, modo OR) → `descartar_autorespuesta` (No Operation) | Bucle infinito de respuestas automáticas. Condiciones: asunto con `Auto-reply`, `Out of office`, `Undeliverable`, `Respuesta automática` o `Fuera de la oficina`; remitente con `no-reply@` o `mailer-daemon`; encabezado `Auto-Submitted: auto…`. |
| 2 | Limpieza del payload | `limpiar_email` (Set) | Pasar al agente HTML, adjuntos y metadatos. Deja solo `From`, `Subject` y `BodyText` (texto plano, sin el mail citado, máximo 3000 caracteres), más `threadId` para la conversación y el borrador. |
| 3 | Validación antes de escribir | `preparar_salida` (Set) + `email_valido` (IF) | Error 400 (payload mal formado). Si el email no es válido, no se escribe en HubSpot ni se crea el borrador; la memoria se actualiza igual y el caso ya figura en el log. |
| 4 | Búsqueda antes de crear | `buscar_contacto` (HubSpot Search) + `existe` (IF) | Error 409 y contactos duplicados. Si existe: `actualizar_contacto` escribe solo `ultimo_tema` y `ultimo_traceid`, sin pisar nombre ni empresa. Si no existe: `crear_contacto` carga nombre, apellido, empresa y los dos campos. El nodo nativo solo ofrece *Create or Update*: la búsqueda decide qué campos se escriben. |
| 5 | Aprobación humana (HITL) | `crear_borrador` (Gmail, Create Draft) | Envío sin revisión. El borrador queda en el mismo hilo, sin Markdown, y una persona lo revisa y lo envía. |

**Decisión:** no se filtran los asuntos con `RE:` ni `FW:`. Las repreguntas de las empresas llegan como respuesta en el mismo hilo y son el caso que la memoria tiene que reconocer.

**Fallas:** los nodos de HubSpot y el borrador tienen *On Error = Continue*. Si HubSpot o Gmail fallan, la memoria se actualiza igual y el caso ya quedó registrado en Slack.

## Herramientas, autenticación y permisos (mínimo privilegio)

| Herramienta | Autenticación | Alcance |
|---|---|---|
| Gmail | OAuth2 | Casilla de prueba exclusiva de la mesa. Lectura de la bandeja de entrada (no leídos) y creación de borradores. Descarga de adjuntos desactivada. |
| Slack | OAuth2 | Dos canales: `#mesa-log` (log de cada consulta) y `#mesa-aprobaciones` (derivaciones). |
| Google Sheets | OAuth2 | Planillas de memoria y de base de conocimiento, con acceso restringido. |
| HubSpot (CRM) | Service Key | Solo `crm.objects.contacts.read`, `crm.objects.contacts.write` y `crm.schemas.contacts.read`. Dominio permitido en n8n: `api.hubapi.com`. |
| Anthropic | API key | Claude Haiku 4.5, temperatura 0. |

**Por qué Service Key en HubSpot:** es el método que recomienda la documentación de n8n. Con OAuth2, un n8n instalado localmente necesita una app pública de HubSpot con unos 13 permisos fijos (empresas, negocios, tickets, formularios, listas), lo opuesto al mínimo privilegio. La Service Key permite habilitar solo los contactos.

**Roles de cada registro:** la planilla de memoria es el cuaderno del agente (resumen, puntos clave, acción requerida) y nadie la edita a mano. HubSpot es el legajo oficial de la empresa para las personas del área: datos de contacto, último tema consultado y `traceId` de la última consulta.

## Identificadores

| Campo | Qué es | Chat | Gmail |
|---|---|---|---|
| `email` | Llave estable de la empresa. Es el **Session_ID** de la consigna: identifica la fila de memoria y el contacto de HubSpot. | Extraído del mensaje | Remitente |
| `sessionId` | Conversación en curso | ID de la ventana de chat | ID del hilo de Gmail (`threadId`) |
| `traceId` | Una ejecución puntual | ID de ejecución de n8n | ID de ejecución de n8n |

El `traceId` viaja a los Workers, al log de Slack, a la planilla (`traceId`) y a HubSpot (`ultimo_traceid`), así que cada contacto se puede cruzar con la ejecución que lo modificó.

En la planilla, `puntos_clave` se guarda como texto multilínea, una línea por punto que empieza con `- `.

## Pruebas de regresión

| traceId | Caso | Resultado |
|---|---|---|
| 237 | Mail de empresa nueva (Rodados del Oeste SRL) | Crea la fila de memoria y el contacto en HubSpot, y deja el borrador con la respuesta (RTE-001). |
| 239 | Mismo mail procesado de nuevo | `existe` = true → actualiza el contacto. No se crea un segundo contacto ni una segunda fila. |
| sin ejecución completa | Asunto "Respuesta automática: fuera de la oficina" | Termina en `descartar_autorespuesta`: sin log, sin escrituras. |
| 242 | Chat de empresa conocida con más de 5 mensajes (Plásticos del Sur) | `es_gmail` = false: sin CRM ni borrador. Responde en el chat (REG-001) y reescribe el resumen. |
| 245 | Respuesta de Rodados en el mismo hilo de Gmail | Mismo `sessionId` que el primer mail, texto citado eliminado, empresa reconocida por memoria, contacto actualizado. Borrador con "No sé" y derivación (el plazo no figura en la ficha). |

La planilla de memoria conserva una sola fila por email.

## Importación

1. En n8n: *Import from file* para cada uno de los tres JSON.
2. Asignar las credenciales propias. Los JSON contienen solo el nombre y el ID de cada credencial, nunca claves ni tokens.
3. En `ejecutar_responder` y `ejecutar_derivar`, volver a seleccionar los Workers importados (se referencian por ID).
