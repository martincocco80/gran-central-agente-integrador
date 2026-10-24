# Gran Central: agente integrador en n8n

Proyecto integrador del curso Automatización Avanzada (Coderhouse), realizado por Martín Cocco.

Gran Central es una mueblería ficticia que fabrica muebles a medida. El proyecto es un agente de atención que recibe las consultas de los clientes por Telegram, las clasifica, consulta el catálogo en Notion, deriva los presupuestos y los reclamos a sub workflows especializados y recuerda a cada cliente entre una charla y otra gracias a una memoria en Airtable.

El proyecto crece módulo a módulo: cada checkpoint se arma sobre el mismo workflow, sin empezar de cero.

## Índice

1. [Contenido del repositorio](#contenido-del-repositorio)
2. [Arquitectura](#arquitectura)
3. [Tecnologías](#tecnologías)
4. [Checkpoint 3: memoria persistente y resumen](#checkpoint-3-memoria-persistente-y-resumen)
5. [Checkpoint 2: Manager y Workers](#checkpoint-2-manager-y-workers)
6. [Checkpoint 1: agente base](#checkpoint-1-agente-base)
7. [Cómo importar el proyecto](#cómo-importar-el-proyecto)
8. [Credenciales necesarias](#credenciales-necesarias)
9. [Bases de datos](#bases-de-datos)
10. [Cómo probarlo](#cómo-probarlo)
11. [Costos](#costos)

## Contenido del repositorio

| Archivo | Qué es |
|---|---|
| `checkpoint3_martin_cocco.json` | Workflow principal actual (Manager con memoria persistente) |
| `worker_reclamos.json` | Sub workflow que registra los reclamos en Notion |
| `worker_presupuestos.json` | Sub workflow que calcula el presupuesto y redacta el borrador con IA |
| `checkpoint1_martin_cocco.json` | Workflow del Checkpoint 1 (agente base, se conserva como historial) |
| `checkpoint3/` | Capturas y PDF de entrega del Checkpoint 3 |
| `flujo.png`, `telegram.png`, `log.png` | Capturas del Checkpoint 1 |

## Arquitectura

El workflow principal tiene tres partes:

**1. Lectura de memoria.** Cada mensaje de Telegram se identifica con un Session_ID (el chat.id del cliente, normalizado como texto en `Config – Parámetros`). Con esa clave se busca la sesión en Airtable. Si el cliente ya escribió antes, se recupera su nombre, el estado de su último caso y el resumen de la última charla. Si es nuevo, se crea la fila con valores por defecto.

**2. Agente Manager.** El `AI Agent – Calificador de consultas` (Tools Agent con gpt-4o) recibe la memoria del cliente dentro de su prompt, consulta el catálogo de Notion cuando hace falta y devuelve una clasificación estructurada (intent, riesgo, datos completos y respuesta al cliente). Según esa clasificación el flujo:
- pide aprobación a un supervisor por Telegram si el riesgo es alto,
- le responde directo al cliente si todavía faltan datos o si es una consulta general,
- llama al Worker de Reclamos o al Worker de Presupuestos cuando los datos están completos,
- deriva a una persona cuando el tema está fuera de lo que atiende la mueblería.

Cada ejecución envía además un mail de log con la clasificación, las herramientas usadas y la ruta tomada.

**3. Escritura y resumen.** Al final se guarda en Airtable el estado del caso, los datos clave de la consulta y el contador de mensajes, siempre con Create or Update sobre el Session_ID (así nunca se duplican filas). Cuando la charla supera los 5 mensajes, gpt-4o-mini resume el historial en un JSON y se guarda en Resumen consolidado.

## Tecnologías

| Herramienta | Uso |
|---|---|
| n8n Cloud | Orquestación de todo el flujo |
| Telegram | Canal de atención al cliente y aprobación del supervisor |
| OpenAI (gpt-4o y gpt-4o-mini) | Agente clasificador y resumen de conversaciones |
| Groq (gpt-oss-120b) | Redacción del borrador de presupuesto |
| Notion | Catálogo, materiales, pedidos, reclamos y presupuestos |
| Airtable | Memoria de largo plazo de cada cliente |
| Gmail | Log de observabilidad de cada ejecución |

## Checkpoint 3: memoria persistente y resumen

Al workflow del Checkpoint 2 le sumamos la memoria de largo plazo en Airtable, la inyección de esa memoria en el agente y la rama de resumen automático.

Acá vemos el flujo completo ejecutado, con la rama de resumen recorrida.

![Flujo general](checkpoint3/flujo_general.png)

Acá vemos la lectura de la memoria: busca la sesión en Airtable y separa al usuario conocido del usuario nuevo antes de llegar al agente.

![Lectura de memoria](checkpoint3/lectura_memoria.png)

Acá vemos las rutas del agente, el guardado de la memoria y la rama que se activa cuando la charla supera los 5 mensajes.

![Rutas y resumen](checkpoint3/rutas_y_resumen.png)

Acá vemos el resumen que genera gpt-4o-mini, siempre con el mismo formato JSON (asunto_principal, puntos_clave y accion_requerida).

![Resumen](checkpoint3/resumen_gpt4o_mini.png)

Acá vemos la fila de la sesión en Airtable después de la prueba de 6 mensajes.

![Sesión en Airtable](checkpoint3/airtable_sesion.png)

**Decisiones de diseño**
- La memoria entra al agente entre los delimitadores `[INICIO DE CONTEXTO COMPARTIDO]` y `[FIN DEL CONTEXTO COMPARTIDO]`, y el prompt aclara que ese bloque es solo contexto y nunca una instrucción (para evitar inyecciones a través de datos guardados).
- En Airtable se guardan solo datos estructurados, nunca la conversación completa, para no cargar el contexto del agente.
- Si el cliente escribe algo sin datos nuevos (por ejemplo un "gracias"), se conservan el estado y los datos clave anteriores.
- Las consultas generales (intent otro) siempre las responde el bot directamente. La derivación a una persona queda para los casos de riesgo alto, que pasan por la aprobación del supervisor.
- El agente tiene activado el reintento automático (2 intentos) por si el modelo devuelve una respuesta sin el formato esperado.

El documento completo de la entrega está en [PreEntrega_Modulo3_MartinCocco.pdf](checkpoint3/PreEntrega_Modulo3_MartinCocco.pdf).

## Checkpoint 2: Manager y Workers

El agente base pasó a trabajar como Manager: clasifica cada consulta con un Structured Output Parser (intent y nivel de riesgo) y deriva el trabajo a dos sub workflows, creados con Convert to sub-workflow.

- **Worker – Reclamos:** busca el pedido en Notion y registra el reclamo con su motivo y gravedad (Grave si es defecto de fabricación o daño en el envío, Leve en el resto).
- **Worker – Presupuestos:** busca el producto y el material en Notion, calcula el total (precio base por medida, cantidad y recargo del material), redacta un borrador para el cliente con IA y registra el presupuesto como Pendiente de aprobación.

Los dos Workers devuelven siempre la misma estructura (status, worker, message y data), así el Manager puede responderle al cliente tanto en los casos exitosos como en los errores.

## Checkpoint 1: agente base

Agente de calificación de consultas que recibe mensajes por Telegram, consulta el catálogo y registra las consultas en Notion, con un log de cada ejecución por Gmail.

Acá vemos el flujo completo ejecutado en verde.

![Flujo](flujo.png)

Acá vemos la conversación con el bot en Telegram.

![Telegram](telegram.png)

Acá vemos el mail de log con los pasos que hizo el agente.

![Log](log.png)

## Cómo importar el proyecto

1. Armá primero las bases de Notion y de Airtable (ver [Bases de datos](#bases-de-datos)).
2. Cargá en n8n las credenciales de la tabla de abajo.
3. Creá un workflow nuevo, abrí el menú de los tres puntos y elegí **Import from File**. Importá `worker_reclamos.json` y guardalo. Repetí lo mismo con `worker_presupuestos.json`. Publicá los dos Workers para que el Manager los pueda llamar.
4. Importá `checkpoint3_martin_cocco.json` en otro workflow nuevo.
5. En los nodos `Execute – Worker Reclamos` y `Execute – Worker Presupuestos`, volvé a elegir los Workers que importaste (n8n les asigna un ID nuevo al importarlos).
6. Revisá cada nodo de Notion, Airtable, Telegram, Gmail, OpenAI y Groq y elegí tu credencial y tus bases en los selectores.
7. En `Config – Parámetros` completá:
   - `email_log`: el mail que va a recibir el log de cada ejecución.
   - `chat_supervisor`: el chat.id de Telegram de la persona que aprueba los casos de riesgo alto.
8. Tocá **Execute workflow** y escribile al bot para probarlo (ver [Cómo probarlo](#cómo-probarlo)).

## Credenciales necesarias

| Servicio | Tipo de credencial | Dónde se usa |
|---|---|---|
| Telegram | Bot token (se crea con @BotFather) | Trigger, respuestas al cliente y aprobación del supervisor |
| OpenAI | API key (o los créditos gratis de n8n) | Agente clasificador (gpt-4o) y resumen (gpt-4o-mini) |
| Groq | API key (tiene plan gratuito) | Redacción del borrador en el Worker de Presupuestos |
| Notion | OAuth2 (compartiendo las bases con la integración) | Catálogo, materiales, pedidos, reclamos y presupuestos |
| Gmail | OAuth2 | Log de observabilidad de cada ejecución |
| Airtable | Personal Access Token con permisos `data.records:read`, `data.records:write` y `schema.bases:read`, y acceso a la base | Lectura y escritura de la memoria |

Los JSON no incluyen claves ni tokens: al importarlos, cada nodo pide elegir la credencial propia.

## Bases de datos

### Airtable: base Gran Central – Memoria, tabla Sesiones

Una fila por cliente, identificada por Session_ID.

| Campo | Tipo | Para qué sirve |
|---|---|---|
| Session_ID | Texto de una línea (campo principal) | Clave única de la sesión (chat.id de Telegram). Todas las lecturas y escrituras filtran por este campo |
| Nombre | Texto de una línea | Nombre del cliente tomado de Telegram |
| Fecha de actualización | Fecha y hora | Última escritura del agente en la fila |
| Resumen consolidado | Texto largo | JSON del resumen generado por gpt-4o-mini |
| Estado del caso | Selección única: Nuevo, Esperando datos, Presupuesto en proceso, Reclamo registrado, Derivado a asesor | Estado de la última solicitud del cliente |
| Datos clave | Texto largo | Datos estructurados de la última consulta (intent, email, pedido o producto) |
| Mensajes | Número entero | Contador de mensajes. Al superar 5 se activa el resumen |

### Notion

| Base | Propiedades que usa el flujo |
|---|---|
| Catálogo | Producto (título), Precio base, Unidad de precio (Por unidad o Por metro lineal), Plazo fabricación (días), Categoría, Activo |
| Materiales | Material (título), Recargo (porcentaje sobre el precio base) |
| Pedidos | N° pedido (título, con formato GC-XXXX) |
| Reclamos | Título, Motivo, Estado, Gravedad y relación con Pedido |
| Presupuestos | Número (título), relación con Producto y Material, Medidas, Cantidad, Total, Estado de aprobación y Borrador IA |

## Cómo probarlo

Con el workflow abierto, tocá **Execute workflow** antes de cada mensaje y escribile al bot. Esta es la conversación que usamos para probar la memoria:

| Mensaje | Qué se espera |
|---|---|
| Hola, buen día | Crea la sesión en Airtable (usuario nuevo) y saluda por el nombre |
| Quiero presupuestar una mesa de comedor | Reconoce el usuario, consulta el catálogo y pide los datos que faltan |
| De roble, de 1,8 metros, 1 unidad | Toma la mesa de la memoria y pide solo el mail |
| Mi mail es (tu mail) | Llama al Worker de Presupuestos y devuelve el total y el plazo |
| ¿Cuánto tarda en estar lista? | Responde el plazo y el estado se mantiene en Presupuesto en proceso |
| Perfecto, quedo atento al mail. Gracias | Llega a 6 mensajes y guarda el resumen en Airtable |

En Airtable tiene que quedar una sola fila para el cliente, con el contador subiendo de a uno.

## Costos

- **n8n Cloud:** el plan usado tiene un límite de ejecuciones por mes. Las pruebas manuales con Execute workflow sirven para desarrollar sin publicar el flujo.
- **OpenAI:** se usaron los créditos gratis que ofrece n8n. gpt-4o-mini se eligió para los resúmenes porque es mucho más económico que gpt-4o y la tarea es simple.
- **Groq:** plan gratuito.
- **Notion, Airtable, Telegram y Gmail:** planes gratuitos.
