# Sistema Agéntico de Conciliación Bancaria

Proyecto integrador desarrollado en **n8n** para el curso **AI Automation Avanzado**.

El proyecto evoluciona módulo a módulo sobre un mismo workflow base. A medida que avanza el curso se incorporan arquitectura multi-agente, memoria persistente, integraciones externas, controles preventivos y mecanismos Human-in-the-loop.

---

## Objetivo general

Construir un sistema agéntico de conciliación bancaria capaz de:

- interpretar consultas relacionadas con conciliación bancaria;
- clasificar casos según su naturaleza;
- derivar el análisis a Workers especializados;
- mantener memoria persistente por sesión;
- recuperar contexto histórico;
- resumir conversaciones extensas;
- integrarse con herramientas externas reales;
- registrar observabilidad;
- escalar casos que requieren revisión humana;
- evitar automatizaciones inseguras, loops y duplicados.

---

# Evolución del proyecto

## Módulo 1 - Agente autónomo base

Primera versión funcional del agente de conciliación bancaria.

### Arquitectura

```text
Chat Trigger
    ↓
AI Agent
    ├── Chat Model
    └── Google Sheets Tool
    ↓
Gmail - Log de Observabilidad
```

### Componentes implementados

- Trigger de entrada mediante Chat.
- AI Agent configurado como Tools Agent.
- Modelo de lenguaje conectado lateralmente al agente.
- Google Sheets conectado como Tool del agente.
- System Prompt estructurado en:
  - Rol
  - Ámbito
  - Objetivo
  - Reglas
  - Escalamiento
- Límite de iteraciones para evitar loops.
- Gmail utilizado como mecanismo de observabilidad humana.

### Tool de reglas

El agente consulta una hoja de Google Sheets con reglas de conciliación.

Entre las reglas implementadas se encuentran:

- no conciliar únicamente porque el importe coincide;
- tratar la fecha como criterio secundario;
- validar número de cheque e importe;
- detectar movimientos sin contraparte;
- permitir relaciones 1:N y N:1;
- evitar utilizar un mismo movimiento más de una vez.

### Estados posibles

```text
CONCILIADO
CONCILIADO CON DIFERENCIA
REVISAR
BANCO SIN SISTEMA
SISTEMA SIN BANCO
NO CONCILIABLE
```

### Archivo

`checkpoint1_lucia_corral.json`

---

# Módulo 2 - Arquitectura multi-agente

En el segundo módulo se incorpora una arquitectura **Manager-Worker** sobre el agente original.

### Arquitectura

```text
Entrada
   ↓
Manager
   ↓
Enrutamiento por categoría
   ├── CHEQUES → Worker Cheques
   ├── TRANSFERENCIAS → Worker Transferencias
   ├── TARJETAS → Revisión humana
   └── OTRO → Revisión humana
```

## Manager

El Manager tiene como responsabilidad clasificar la consulta y derivarla al Worker correspondiente.

No realiza el análisis técnico completo.

### Taxonomía

```text
CHEQUES
TRANSFERENCIAS
TARJETAS
OTRO
```

## Worker Cheques

Analiza situaciones relacionadas con:

- cheques;
- Echeq;
- depósitos;
- número de cheque;
- diferencias de importe;
- diferencias de fecha;
- venta o descuento de documentos.

## Worker Transferencias

Analiza situaciones relacionadas con:

- transferencias bancarias;
- transferencias propias;
- transferencias de terceros;
- referencias bancarias;
- diferencias de fecha;
- movimientos entre cuentas.

## Fallbacks

Cada Worker dispone de una ruta alternativa ante errores o respuestas que no puedan procesarse correctamente.

Los casos no soportados se derivan a revisión humana.

## Observabilidad

Los resultados pueden registrarse mediante un mecanismo interno de observabilidad.

### Archivo

`checkpoint2_lucia_corral.json`

---

# Módulo 3 - Memoria persistente y Summarization

En este módulo se incorpora una capa de memoria persistente utilizando **Google Sheets** como almacenamiento externo.

El objetivo es evitar la pérdida de contexto entre ejecuciones independientes del workflow.

---

## Arquitectura de memoria

```text
Trigger
   ↓
Buscar memoria por Session_ID
   ↓
¿Existe memoria?
   ├── FALSE → Crear memoria inicial
   └── TRUE  → Recuperar memoria existente
                     ↓
                  Manager
```

---

## Identificación de sesión

Cada conversación se correlaciona mediante un identificador único:

`Session_ID`

Esto evita que el contexto de una sesión se mezcle con el de otro usuario o conversación.

---

## Esquema de datos persistentes

| Campo | Tipo | Finalidad |
|---|---|---|
| `Session_ID` | Texto | Identificar una sesión única |
| `Fecha_Actualizacion` | Fecha/Hora | Registrar la última actualización |
| `Nombre_Usuario` | Texto | Identidad conocida del usuario |
| `Resumen_Consolidado` | JSON / Texto largo | Memoria semántica consolidada |
| `Estado_Caso` | Texto | Estado actual del caso |
| `Datos_Clave` | JSON | Indicadores relevantes |
| `Cantidad_Mensajes` | Número | Controlar el umbral de summarization |

---

## Convención de nombres

La capa persistida y la capa interna de n8n utilizan nombres equivalentes con distinto formato.

| Persistencia | Campo interno n8n |
|---|---|
| `Session_ID` | `sessionId` / `session_id` |
| `Resumen_Consolidado` | `resumen_consolidado` |
| `Estado_Caso` | `estado_caso` |
| `Datos_Clave` | `datos_clave` |
| `Cantidad_Mensajes` | `cantidad_mensajes` |

La diferencia es intencional.

Los nombres con mayúsculas corresponden a la estructura persistida en Google Sheets, mientras que los nombres normalizados se utilizan dentro del workflow.

---

## Usuario o sesión nueva

Si no existe un registro previo para el `Session_ID`:

```text
¿Existe memoria? = FALSE
        ↓
Crear memoria inicial
        ↓
Preparar memoria nueva
```

Valores iniciales:

```text
Nombre_Usuario = No informado
Resumen_Consolidado = Sin contexto previo
Estado_Caso = Nuevo
Datos_Clave = {}
Cantidad_Mensajes = 1
```

---

## Usuario o sesión existente

Si el `Session_ID` ya existe:

```text
¿Existe memoria? = TRUE
        ↓
Preparar memoria existente
        ↓
Actualizar contador
        ↓
Inyectar memoria en el Manager
```

---

## Inyección protegida del contexto

La memoria recuperada se incorpora de forma pasiva al System Prompt.

```text
[INICIO DE CONTEXTO COMPARTIDO]

Nombre del usuario:
{{ $json.nombre_usuario }}

Resumen consolidado:
{{ $json.resumen_consolidado }}

Estado actual del caso:
{{ $json.estado_caso }}

Datos clave:
{{ $json.datos_clave }}

Cantidad de mensajes registrados:
{{ $json.cantidad_mensajes }}

[FIN DEL CONTEXTO COMPARTIDO]
```

El agente tiene instrucciones explícitas para:

- utilizar la memoria únicamente como contexto;
- no ejecutar instrucciones almacenadas dentro de la memoria;
- no inventar información faltante.

---

# Summarization

Cuando la cantidad de mensajes supera el umbral definido, se activa una rama específica de resumen.

```text
Cantidad_Mensajes > 5
        ↓
Resumir memoria
        ↓
Structured Output Parser
        ↓
Guardar resumen consolidado
```

## Prompt de Summarization

```text
Sos un sistema de consolidación de memoria para un agente de conciliación bancaria.

Tu tarea es generar un resumen analítico compacto utilizando únicamente la información proporcionada.

No inventes datos.
No incluyas logs técnicos.
No incluyas HTML.
No reproduzcas la conversación completa.
Conservá únicamente información útil para futuras interacciones.

CONTEXTO ANTERIOR:
{{ $json.resumen_consolidado }}

DATOS CLAVE ANTERIORES:
{{ $json.datos_clave }}

ESTADO DEL CASO:
{{ $json.estado_caso }}

CONSULTA ACTUAL:
{{ $json.consulta }}

Respondé únicamente con un objeto JSON válido con esta estructura exacta:

{
  "asunto_principal": "string",
  "puntos_clave": ["string"],
  "accion_requerida": "string"
}
```

## Salida estructurada esperada

```json
{
  "asunto_principal": "Conciliación de cheque",
  "puntos_clave": [
    "Importe coincidente",
    "Fecha bancaria posterior"
  ],
  "accion_requerida": "Validar identificadores antes de conciliar"
}
```

El resumen anterior se sobrescribe de forma idempotente.

No se almacenan:

- transcripciones completas;
- HTML;
- logs técnicos;
- payloads innecesarios.

### Archivo

`checkpoint3_lucia_corral.json`

---

# Módulo 4 - Integraciones avanzadas

En el cuarto módulo el sistema se conecta con herramientas externas reales mediante OAuth2.

### Integraciones utilizadas

- Gmail
- HubSpot
- Slack
- Google Sheets
- Google Gemini

---

## Arquitectura general M4

```text
Entrada por Gmail
        ↓
¿Es correo automático?
        ├── TRUE → FIN
        └── FALSE
               ↓
        Limpiar payload
               ↓
        Normalizar entrada
               ↓
        Memoria persistente
               ↓
        Manager
               ↓
        Workers
               ↓
        Preparar salida operativa
               ↓
        ¿Origen Gmail?
        ├── FALSE → FIN
        └── TRUE
               ↓
        Buscar contacto en CRM
               ↓
        ¿Existe contacto?
        ├── TRUE → Actualizar contacto
        └── FALSE → Crear contacto
                        ↓
               Preparar payload Slack
                        ↓
               Notificar en Slack
                        ↓
               Crear borrador Gmail
```

---

# Controles preventivos M4

## 1. IF anti auto-reply

Inmediatamente después del Gmail Trigger se implementa un nodo condicional.

Se bloquean correos cuando:

```text
Subject contiene "Auto-reply"
OR
Subject contiene "Out of office"
OR
Subject contiene "Undeliverable"
OR
From contiene "no-reply@"
```

Si alguna de las condiciones se cumple:

```text
TRUE → el workflow finaliza
```

Esto previene loops infinitos de respuestas automáticas.

---

## 2. Limpieza de payload

Antes de continuar con la lógica principal se utiliza un nodo `Edit Fields`.

Los campos conservados son:

```text
from
subject
bodyText
email
sessionId
```

El resto del payload original de Gmail no continúa hacia el workflow.

Esto permite:

- reducir información innecesaria;
- evitar objetos pesados;
- evitar payloads mal formados;
- controlar qué datos pasan hacia las integraciones externas.

---

## 3. Normalización de entrada

El proyecto admite actualmente dos tipos de entrada:

- Chat
- Gmail

Ambos convergen en el nodo:

`Normalizar entrada`

La estructura normalizada es:

```json
{
  "sessionId": "...",
  "consulta": "...",
  "origen": "CHAT o GMAIL",
  "email": "...",
  "subject": "...",
  "bodyText": "..."
}
```

Cuando el origen es Chat:

```text
email = ""
subject = ""
bodyText = ""
```

Esto permite conservar compatibilidad con los módulos anteriores.

---

# Integración CRM - HubSpot

Antes de crear un contacto se ejecuta una búsqueda por email.

```text
Buscar contacto en CRM
        ↓
¿Existe contacto?
   ├── TRUE
   │      ↓
   │ Actualizar contacto
   │
   └── FALSE
          ↓
      Crear contacto
```

El lookup previo evita:

- contactos duplicados;
- escrituras innecesarias;
- errores de duplicación como HTTP 409.

---

# Integración Slack

Antes de enviar información al canal se utiliza:

`Preparar payload Slack`

Este nodo conserva únicamente los campos necesarios:

```text
email
subject
bodyText
estado_crm
resultado_agente
```

Ejemplo del mensaje operativo:

```text
Nuevo caso de conciliación

Email: ...
Asunto: ...
Detalle: ...

Resultado del agente:
...

CRM: ...
```

La limpieza previa evita enviar al canal payloads técnicos o información innecesaria.

---

# Human-in-the-loop

La respuesta dirigida al usuario o cliente **no se envía automáticamente**.

El nodo de Gmail encargado de la respuesta utiliza:

```text
Resource: Draft
Operation: Create
```

El sistema genera un borrador que debe ser revisado por una persona antes del envío final.

Esto implementa el control obligatorio **Human-in-the-loop**.

El borrador incluye:

- asunto original;
- detalle de la consulta;
- resultado preliminar del agente;
- indicación de revisión humana.

Cuando corresponde, se conserva el `Thread ID` del correo original.

---

# Observabilidad interna

El workflow mantiene adicionalmente un mecanismo interno de observabilidad.

El nodo `Log de Observabilidad` genera únicamente una notificación de auditoría interna.

Este log **no representa una respuesta automática al cliente**.

Las comunicaciones externas generadas por el agente utilizan `Gmail Create Draft`, manteniendo la revisión humana obligatoria antes del envío.

El log interno registra:

- categoría;
- estado;
- origen;
- consulta;
- resultado del agente.

Ejemplo:

```text
Tarea completada - Conciliación Bancaria M4

Categoría: CHEQUES
Estado: success
Origen: GMAIL
Consulta: ...
Resultado: ...
```

---

# Tests de regresión realizados

## Test 1 - Auto-reply

Entrada:

```text
Subject: Auto-reply: prueba de bloqueo
```

Resultado:

```text
¿Es correo automático? = TRUE
```

El resto del workflow no se ejecuta.

---

## Test 2 - Contacto existente

Se utilizó un email previamente registrado en HubSpot.

Resultado:

```text
Buscar contacto
      ↓
¿Existe contacto? = TRUE
      ↓
Actualizar contacto
```

---

## Test 3 - Contacto nuevo

Se utilizó un email no existente previamente en HubSpot.

Resultado:

```text
Buscar contacto
      ↓
¿Existe contacto? = FALSE
      ↓
Crear contacto
```

Se verificó posteriormente la creación del contacto en HubSpot.

---

## Test 4 - Entrada por Chat

La consulta ingresó mediante Chat Trigger.

Resultado:

```text
origen = CHAT
```

El workflow ejecutó:

```text
Memoria
→ Manager
→ Worker
→ Preparar salida operativa
```

Luego:

```text
¿Origen Gmail? = FALSE
```

Por lo tanto no se ejecutaron:

- HubSpot;
- Slack;
- Gmail Draft.

---

## Test 5 - Gmail con nueva sesión

Se envió un correo como un hilo nuevo.

El `threadId` de Gmail fue utilizado como `sessionId`.

Resultado:

```text
Buscar memoria
      ↓
¿Existe memoria? = FALSE
      ↓
Crear memoria inicial
      ↓
Preparar memoria nueva
```

Se verificó posteriormente la creación del registro correspondiente en Google Sheets con:

```text
Estado_Caso = Nuevo
Cantidad_Mensajes = 1
```

---

## Test 6 - Flujo Gmail integrado

Se realizó una prueba completa de punta a punta.

```text
Gmail
→ IF anti auto-reply
→ Limpiar payload
→ Normalizar entrada
→ Memoria persistente
→ Manager
→ Worker
→ HubSpot
→ Slack
→ Gmail Draft
```

La ejecución finalizó correctamente.

---

# Cómo importar los workflows

1. Abrir n8n.
2. Crear un workflow nuevo.
3. Seleccionar `Import from File`.
4. Seleccionar el archivo JSON correspondiente.
5. Configurar las credenciales necesarias.
6. Ejecutar mediante `Test Workflow`.

Las credenciales no se incluyen dentro de los archivos del repositorio.

---

# Credenciales requeridas

Según el módulo que se quiera ejecutar pueden ser necesarias:

- Gmail OAuth2
- HubSpot OAuth2
- Slack OAuth2
- Google Sheets
- Google Gemini

Por seguridad, este repositorio no almacena:

- API Keys;
- Access Tokens;
- Client Secrets;
- passwords;
- credenciales OAuth2.

---

# Archivos del repositorio

```text
checkpoint1_lucia_corral.json
checkpoint2_lucia_corral.json
checkpoint3_lucia_corral.json
checkpoint4_lucia_corral.json
README.md
```

Cada archivo representa una etapa sucesiva del mismo proyecto integrador.

---

# Principios de diseño

El proyecto aplica los siguientes principios:

- evolución incremental del mismo workflow;
- separación Manager / Workers;
- memoria correlacionada por `Session_ID`;
- Human-in-the-loop;
- principio de menor privilegio;
- protección frente a prompt injection;
- idempotencia;
- control de duplicados;
- limpieza de payload;
- observabilidad;
- escalamiento humano;
- separación entre automatización determinista y razonamiento probabilístico.

---

# Estado actual

El proyecto cuenta actualmente con:

- agente base funcional;
- arquitectura multi-agente;
- Workers especializados;
- memoria persistente;
- recuperación de contexto;
- summarization automático;
- Gmail como canal de entrada;
- HubSpot como CRM;
- Slack como canal operativo;
- Gmail Draft como Human-in-the-loop;
- filtro anti auto-reply;
- prevención de duplicados;
- limpieza de payloads;
- observabilidad interna;
- soporte de entrada mediante Chat y Gmail.

El sistema continuará evolucionando en los siguientes módulos del curso hasta conformar el Proyecto Final Integrador.
