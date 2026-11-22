# Quantum Zone Assistants — Automatización Inteligente de Atención y Gestión por Correo

**Curso:** Automatización Inteligente de Procesos con n8n  
**Institución:** Institución Universitaria EAM — Periodo 2026-II  
**Docente:** Robert García Rey  
**Repositorio:** `n8n-final-lievano_patino_lopez-QuantumZoneAssistants`

---

## 1. Identificación del Proyecto

* **Nombre del Proyecto:** Quantum Zone Assistants
* **Integrantes del Equipo:**
  * Miguel Angel Liévano Buitrago
  * Carlos Mario Patiño Ramirez
  * Carlos Andrés López Perez
* **Problema:** La recepción manual de correos electrónicos con solicitudes, consultas y soportes genera tiempos de respuesta elevados, falta de trazabilidad sobre qué mensajes han sido atendidos y pérdida de contexto cuando un usuario escribe nuevamente o consulta información almacenada en bases de datos.
* **Objetivo:** Implementar un asistente automatizado en **n8n** impulsado por Inteligencia Artificial (Google Gemini) capaz de analizar correos entrantes, consultar información estructurada en Baserow, responder automáticamente al remitente, registrar una auditoría completa de cada interacción y gestionar fallos técnicos mediante contingencia automatizada.
* **Alcance de la Versión Final:**
  * Monitoreo automático de bandeja de entrada (Gmail Trigger).
  * Preprocesamiento y clasificación/análisis inicial mediante LLM (Google Gemini).
  * Agente conversacional (`AI Agent`) con memoria de sesión (`Simple Memory`) y herramienta de consulta a base de datos (`Baserow Tool`).
  * Enrutamiento condicional (`Switch`) según la acción determinada por el agente.
  * Respuesta automatizada vía correo electrónico (`Gmail`).
  * Sub-proceso de auditoría que registra remitente, fecha/hora, mensaje original y respuesta enviada en Baserow (`Historial_Mensajes`).
  * Rama integrada de manejo de errores y contingencia que captura excepciones en nodos críticos, guarda logs técnicos en Baserow (`Logs_Errores`) y envía alertas al administrador.

---

## 2. Vista General de la Arquitectura

### Recorrido del Proceso
1. **Disparo y Extracción (`Gmail Trigger1` $\rightarrow$ `Edit Fields`):** Detecta la llegada de un nuevo correo electrónico y extrae los metadatos relevantes (remitente, asunto, cuerpo del mensaje y fecha).
2. **Análisis Inicial (`Message a model` $\rightarrow$ `Edit Fields1`):** Un primer modelo de Google Gemini procesa el texto entrante para normalizar la intención y preparar el contexto de entrada para el agente.
3. **Razonamiento y Consulta (`AI Agent`):**
   * **Modelo (`Google Gemini Chat Model`):** Genera respuestas coherentes en formato estructurado según las instrucciones del sistema.
   * **Memoria (`Simple Memory`):** Mantiene el hilo conversacional por sesión/remitente.
   * **Herramienta (`Get many rows in Baserow`):** Consulta registros en la base de datos de conocimiento/entregas cuando el usuario solicita información específica.
4. **Enrutamiento y Respuesta (`Switch1` $\rightarrow$ `Send a message`):** Evalúa la acción definida por el agente (`enviar_correo`) y despacha la respuesta personalizada al usuario.
5. **Sub-proceso de Auditoría (`Registrar Historial Mensaje`):** Almacena en la tabla `Historial_Mensajes` de Baserow el remitente, la marca de tiempo (`America/Bogota`), el cuerpo del correo recibido y la respuesta enviada.
6. **Manejador de Errores y Contingencia (`Formatear Error` $\rightarrow$ `Guardar Log Error` $\rightarrow$ `Notificar Error Admin`):** Conectado a las salidas de error (*Continue using error output*) de los nodos críticos. Ante una falla técnica, formatea el error, lo registra en la tabla `Logs_Errores` y notifica por correo al equipo de soporte.

### Servicios Utilizados y Función de la IA
| Servicio / Componente | Rol en el Flujo |
| :--- | :--- |
| **n8n** | Orquestador principal del workflow, enrutamiento lógico y manejo de excepciones. |
| **Google Gemini (PaLM/Gemini API)** | Comprensión del lenguaje natural, análisis de intención y redacción de respuestas contextualizadas mediante un `AI Agent`. |
| **Gmail (Google Workspace)** | Disparador de correos entrantes, envío de respuestas a usuarios y envío de alertas técnicas al administrador. |
| **Baserow** | Base de datos relacional no-code utilizada como fuente de consulta para el agente, registro de auditoría (`Historial_Mensajes`) y bitácora de fallos (`Logs_Errores`). |

---

## 3. Requisitos Previos

* **Plataforma n8n:** Versión `1.70.0` o superior (compatible con nodos `AI Agent`, `Google Gemini Chat Model` y salidas de error nativas). Ejecutable en instancia local (`localhost:5678`) o nube.
* **Cuentas y Permisos:**
  * **Cuenta de Google (Gmail):** Con permisos OAuth2 habilitados para lectura de bandeja de entrada (`gmail.readonly`) y envío de correos (`gmail.send`).
  * **Cuenta de Google AI Studio:** Clave de API (API Key) activa para consumir modelos de la familia **Google Gemini**.
  * **Cuenta de Baserow (Cloud o Self-Hosted):** Token de API de base de datos con permisos de lectura (`List rows`) y escritura (`Create row`).

---

## 4. Configuración de Recursos Externos

Antes de ejecutar el workflow, cree las siguientes tres tablas en su espacio de trabajo de **Baserow**:

### A. Tabla de Consulta (Fuente de conocimiento del Agente)
* **Nombre sugerido:** `entregas_evidencias` (o la tabla de inventario/servicios de su dominio).
* **Campos:** Columnas con la información que el agente consultará para responder preguntas de los usuarios.

### B. Tabla de Auditoría: `Historial_Mensajes`
Cree exactamente las siguientes columnas:
* `Remitente` *(Single line text)*
* `Fecha_Hora` *(Single line text)*
* `Cuerpo_Mensaje` *(Long text)*
* `Respuesta_Enviada` *(Long text)*

### C. Tabla de Contingencia: `Logs_Errores`
Cree exactamente las siguientes columnas:
* `Fecha_Hora` *(Single line text)*
* `Nodo_Fallido` *(Single line text)*
* `Detalle_Error` *(Long text)*
* `Remitente_Afectado` *(Single line text)*

---

## 5. Importación y Ejecución en n8n

1. **Descargar el workflow:** Obtenga el archivo [`workflows/Workflow_Final.json`](workflows/Workflow_Final.json) de este repositorio.
2. **Importar en n8n:**
   * Abra su editor de n8n y cree un nuevo workflow vacío.
   * Haga clic en el menú superior derecho (`...`) $\rightarrow$ **Import from File...** y seleccione `Workflow_Final.json`.
3. **Configurar Credenciales (Sustitución de valores propios):**
   * **Gmail (`Gmail Trigger1`, `Send a message`, `Notificar Error Admin`):** Seleccione o cree su credencial **Gmail OAuth2 API**. En `Notificar Error Admin`, reemplace el destinatario ficticio (`admin@quantumzone-demo.com`) por el correo del administrador de pruebas.
   * **Google Gemini (`Message a model`, `Google Gemini Chat Model`):** Asigne su credencial **Google Gemini(PaLM) Api account** con su API Key propia.
   * **Baserow (`Get many rows in Baserow`, `Registrar Historial Mensaje`, `Guardar Log Error`):** Asigne su credencial **Baserow API** y seleccione su *Database ID* y *Table ID* correspondientes a cada tabla creada en el paso 4.
4. **Ejecución y Detención:**
   * Para pruebas manuales, haga clic en **Execute workflow** y envíe un correo de prueba a la cuenta configurada.
   * Para operación continua, active el interruptor **Active / Publish** en la esquina superior derecha. **Importante:** Recuerde desactivar el workflow una vez finalizadas las pruebas para evitar consumo innecesario de cuota de API o polling continuo de Gmail.

---

## 6. Ejemplo Reproducible

### Caso 1: Flujo Exitoso (Consulta y Auditoría)
* **Condiciones iniciales:** Workflow activo, tablas de Baserow configuradas y credenciales conectadas.
* **Entrada ficticia:** Enviar un correo desde `cliente.prueba@ejemplo.com` hacia la cuenta monitoreada con el asunto *"Consulta de documentos"* y cuerpo: `"Hola, adjunto los documentos solicitados para revisión."`
* **Pasos:** El disparador captura el correo $\rightarrow$ Gemini analiza el texto $\rightarrow$ El `AI Agent` redacta una confirmación cordial en formato JSON $\rightarrow$ `Switch1` enruta hacia `Send a message` $\rightarrow$ Se envía respuesta al remitente y se guarda el registro.
* **Resultado esperado:**
  1. El remitente recibe un correo automático agradeciendo y confirmando la recepción.
  2. En la tabla `Historial_Mensajes` de Baserow se crea una nueva fila con `Remitente = cliente.prueba@ejemplo.com`, la fecha/hora exacta, el cuerpo original y la respuesta generada.

### Caso 2: Simulación de Fallo Técnico (Manejador de Errores)
* **Condiciones iniciales:** Cambiar temporalmente el `Table ID` del nodo `Registrar Historial Mensaje` por un ID inexistente (ej. `999999`) o usar una credencial inválida en un nodo principal.
* **Resultado esperado:** El nodo falla y activa su salida de error (*Continue using error output*), ejecutando la rama inferior (`Formatear Error` $\rightarrow$ `Guardar Log Error` $\rightarrow$ `Notificar Error Admin`). Se crea un registro en `Logs_Errores` y llega un correo con el asunto `[ALERTA N8N] Fallo en Quantum Zone Assistant`.

---

## 7. Navegación del Repositorio y Entregables

* **Workflow Final (JSON):** [`workflows/Workflow_Final.json`](workflows/Workflow_Final.json)
* **Documentación Técnica Completa:** [`documentacion/Documentacion_Proyecto_Final.md`](documentacion/Documentacion_Proyecto_Final.md)
* **Diagramas de Proceso:**
  * Proceso Manual: [`documentacion/diagramas/Proceso_Manual.png`](documentacion/diagramas/Proceso_Manual.png)
  * Proceso Automatizado: [`documentacion/diagramas/Proceso_Automatizado.png`](documentacion/diagramas/Proceso_Automatizado.png)
* **Bitácora de Pruebas (Clase 11):** [`pruebas/Bitacora_Clase_11_Pruebas.md`](pruebas/Bitacora_Clase_11_Pruebas.md)
* **Carpeta de Evidencias:** [`pruebas/evidencias/`](pruebas/evidencias/)
* **Presentación Ejecutiva (PDF):** [`presentacion/Presentacion_Proyecto_Final.pdf`](presentacion/Presentacion_Proyecto_Final.pdf)

### Limitaciones Conocidas
* La herramienta `Get many rows in Baserow` conectada al `AI Agent` maneja internamente errores de consulta devolviendo una observación al modelo en lugar de detener el flujo; por ello, las excepciones críticas de infraestructura se capturan directamente en los nodos principales del canvas.
* La memoria (`Simple Memory`) es volátil en ejecución y se reinicia si se reinicia el servidor local de n8n.
