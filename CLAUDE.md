# Chatbot - Servicios Integrales de Seguridad Social

## Descripción del Proyecto

Chatbot inteligente para una empresa de **afiliaciones a seguridad social** con alto volumen de usuarios. El bot debe:

1. **Resolver dudas** sobre procesos de afiliación a EPS, ARL, AFP (pensiones) y Caja de Compensación
2. **Guiar al usuario** paso a paso en el proceso de vinculación
3. **Gestionar pagos** e informar sobre costos, fechas de pago y estados de cuenta
4. **Capturar datos** del usuario para iniciar trámites de afiliación
5. **Escalar a agente humano** cuando la consulta supere la capacidad del bot

El chatbot se construye y orquesta completamente con **n8n** como motor de flujos de trabajo, integrando canales como WhatsApp (Meta API), Telegram o webchat.

---

## Stack Tecnológico

| Componente | Tecnología |
|---|---|
| Orquestador de flujos | n8n (self-hosted o cloud) |
| IA / LLM | Claude (Anthropic API) via n8n AI nodes |
| MCP Server | n8n-mcp (acceso programático a n8n desde Claude Code) |
| Skills de n8n | n8n-skills (7 skills especializados para Claude Code) |
| Base de conocimiento | Documentos internos + RAG en n8n |
| Canal principal | WhatsApp Business API / Telegram / Webchat |
| Almacenamiento | PostgreSQL o SQLite (estados de conversación) |

---

## Herramientas MCP (n8n-MCP)

Este proyecto usa el servidor MCP **n8n-mcp** para que Claude Code pueda crear, editar y gestionar flujos en n8n directamente.

**Repositorio:** https://github.com/czlonkowski/n8n-mcp

### Configuración del MCP

Agrega en tu `~/.claude/claude_desktop_config.json` o en la configuración MCP de Claude Code:

```json
{
  "mcpServers": {
    "n8n": {
      "command": "npx",
      "args": ["n8n-mcp"],
      "env": {
        "N8N_API_URL": "http://localhost:5678",
        "N8N_API_KEY": "<tu-api-key-de-n8n>"
      }
    }
  }
}
```

### Herramientas disponibles vía MCP

**Documentación (sin API key):**
- `search_nodes` — Buscar nodos por funcionalidad
- `get_node` — Obtener configuración detallada de un nodo
- `validate_node` — Validar configuración de un nodo
- `validate_workflow` — Validar un flujo completo antes de desplegarlo
- `search_templates` — Buscar plantillas de flujos existentes (2,709+ templates)
- `get_template` — Obtener un template específico
- `tools_documentation` — Documentación de todas las herramientas

**Gestión de n8n (requiere N8N_API_KEY):**
- `list_workflows` / `get_workflow` / `create_workflow` / `update_workflow` / `delete_workflow`
- `activate_workflow` / `deactivate_workflow`
- `list_executions` / `get_execution`
- `get_n8n_info` — Verificar salud del sistema

### Reglas de Seguridad con MCP

- **SIEMPRE** exportar/respaldar un flujo antes de modificarlo con AI
- **SIEMPRE** probar en ambiente de desarrollo antes de activar en producción
- **NUNCA** eliminar un flujo sin confirmación explícita del usuario
- Los valores de parámetros por defecto son la causa #1 de fallas en runtime — revisar siempre

---

## Skills de n8n para Claude Code

**Repositorio:** https://github.com/czlonkowski/n8n-skills

Instala los 7 skills copiando los archivos `.md` al directorio de skills de Claude Code (`~/.claude/skills/` o mediante el plugin de Claude Code para VSCode).

### Skills disponibles

| Skill | Uso principal |
|---|---|
| `expression-syntax` | Sintaxis de expresiones n8n: `$json`, `$node`, `$now`, `$env` |
| `mcp-tools-expert` | Uso correcto de herramientas MCP y formatos de nodeType |
| `workflow-patterns` | 5 patrones arquitectónicos probados (webhook, scheduled, etc.) |
| `validation-expert` | Interpretación de errores y falsos positivos en validación |
| `node-configuration` | Configuración de 525+ nodos y sus dependencias |
| `javascript-code` | Patrones de datos en Code nodes, `$helpers`, formato de retorno |
| `python-code` | Código Python en n8n (sin librerías externas) |

---

## Arquitectura del Chatbot

### Flujos Principales a Crear en n8n

#### 1. `chatbot-entrada` — Recepción de mensajes
```
Webhook (WhatsApp/Telegram) → Normalizar mensaje → Router por intención
```

#### 2. `chatbot-router` — Clasificación de intenciones
```
Mensaje → Claude AI (classify intent) → Switch node →
  ├── afiliacion_nueva
  ├── estado_afiliacion
  ├── consulta_pago
  ├── documentos_requeridos
  ├── escalar_humano
  └── saludo_general
```

#### 3. `flujo-afiliacion` — Proceso de vinculación
```
Inicio afiliación → Recolectar datos (nombre, cédula, tipo contrato) →
Validar documentos → Consultar EPS disponibles →
Confirmar selección usuario → Registrar en sistema → Notificar resultado
```

#### 4. `flujo-pagos` — Consultas de pago
```
Solicitud pago → Verificar identidad → Consultar estado →
Calcular valor aportes → Generar referencia pago → Enviar instrucciones
```

#### 5. `base-conocimiento` — RAG interno
```
Pregunta → Embedding → Búsqueda vectorial (docs internos) →
Contexto relevante → Claude AI (respuesta) → Enviar al usuario
```

#### 6. `escalado-humano` — Transferencia a agente
```
Trigger escalado → Notificar agente disponible →
Transferir contexto conversación → Modo humano activo
```

### Variables de Entorno Requeridas

```env
# n8n
N8N_API_URL=http://localhost:5678
N8N_API_KEY=your_n8n_api_key

# Anthropic
ANTHROPIC_API_KEY=your_anthropic_api_key

# Canal de mensajería (elegir uno o varios)
WHATSAPP_TOKEN=your_meta_whatsapp_token
WHATSAPP_PHONE_NUMBER_ID=your_phone_number_id
TELEGRAM_BOT_TOKEN=your_telegram_bot_token

# Base de datos
DB_CONNECTION_STRING=postgresql://user:pass@localhost:5432/chatbot_db

# Empresa
COMPANY_NAME="Servicios Integrales"
SUPPORT_EMAIL=soporte@empresa.com
SUPPORT_PHONE=+57XXXXXXXXXX
```

---

## Dominio de Negocio — Seguridad Social Colombia

### Sistemas de Afiliación

| Sistema | Descripción |
|---|---|
| **EPS** | Entidad Promotora de Salud (salud) |
| **ARL** | Administradora de Riesgos Laborales (riesgos) |
| **AFP** | Administradora de Fondos de Pensiones (pensiones) |
| **CCF** | Caja de Compensación Familiar (subsidios) |

### Tipos de Afiliación

- **Dependiente**: empleado con contrato laboral (empleador hace aportes)
- **Independiente**: trabajador por cuenta propia (hace sus propios aportes)
- **Subsidiado**: personas sin capacidad de pago (Sisbén)

### Documentos Requeridos Típicos

- Cédula de ciudadanía o extranjería
- Contrato de trabajo o RUT (independientes)
- Formulario de afiliación firmado
- Carta de asignación salarial
- Datos del beneficiario (si aplica)

### Fechas Clave de Pago

- Aportes de nómina: hasta el último día hábil del mes
- Trabajadores independientes: antes del día 15 de cada mes
- Pago en mora genera intereses + posible suspensión del servicio

### Intenciones del Bot (Intent Classification)

```
AFILIACION_NUEVA       → Usuario quiere afiliarse por primera vez
CAMBIO_EPS             → Quiere cambiar de EPS
ESTADO_TRAMITE         → Consultar en qué estado está su solicitud
CONSULTA_PAGO          → Cuánto debe pagar, cómo pagarlo
MORA_DEUDA             → Tiene deuda pendiente
DOCUMENTOS             → Qué documentos necesita
BENEFICIARIOS          → Agregar/quitar beneficiarios (hijos, cónyuge)
NOVEDAD_LABORAL        → Cambio de empleador, retiro, licencia
ESCALAR_HUMANO         → No entiende o prefiere hablar con persona
SALUDO_DESPEDIDA       → Inicio o cierre de conversación
FUERA_DE_TEMA         → Consulta que no corresponde al negocio
```

---

## Guía de Trabajo con Claude Code

### Flujo de Trabajo Recomendado

1. **Describe el flujo** en lenguaje natural (qué debe hacer, qué nodos necesitas)
2. **Claude Code** usa los skills y el MCP para diseñar y crear el flujo en n8n
3. **Valida** el flujo con `validate_workflow` antes de activarlo
4. **Prueba** en ambiente de desarrollo
5. **Activa** en producción con `activate_workflow`

### Comandos Frecuentes para Claude Code

```
"Crea un flujo en n8n que reciba mensajes de WhatsApp y los clasifique por intención usando Claude AI"

"Agrega un nodo de validación de cédula colombiana al flujo de afiliación"

"Busca templates de n8n para chatbot con IA y muéstrame los más relevantes"

"Valida el flujo 'chatbot-entrada' y corrije los errores encontrados"

"Lista todos los flujos activos en n8n y su estado"
```

### Patrones de Expresiones n8n Comunes

```javascript
// Obtener dato del mensaje entrante
{{ $json.body.messages[0].text.body }}

// Referencia a nodo previo
{{ $node["Clasificar Intención"].json.intent }}

// Fecha y hora actual
{{ $now.toFormat('dd/MM/yyyy HH:mm') }}

// Variable de entorno
{{ $env.COMPANY_NAME }}

// Condicional en expresión
{{ $json.tipo_contrato === 'dependiente' ? 'Empleador hace aportes' : 'Debes hacer aportes tú mismo' }}
```

---

## Convenciones de Nombres en n8n

- **Flujos**: `kebab-case` descriptivo → `chatbot-afiliacion-nueva`, `flujo-consulta-pago`
- **Nodos**: Títulos en español descriptivo → `Recibir Mensaje WhatsApp`, `Clasificar Intención`
- **Variables**: `snake_case` → `cedula_usuario`, `eps_seleccionada`
- **Webhooks**: `/webhook/chatbot/<canal>` → `/webhook/chatbot/whatsapp`

---

## Estructura del Repositorio

```
chatbot-servicios-integrales/
├── CLAUDE.md                    # Este archivo
├── workflows/                   # Exportaciones JSON de flujos n8n
│   ├── chatbot-entrada.json
│   ├── chatbot-router.json
│   ├── flujo-afiliacion.json
│   ├── flujo-pagos.json
│   └── base-conocimiento.json
├── prompts/                     # Prompts del sistema para el LLM
│   ├── system-prompt.md         # Personalidad y contexto del bot
│   ├── intent-classification.md # Prompt para clasificar intenciones
│   └── afiliacion-guide.md      # Guía de proceso de afiliación
├── docs/                        # Documentación de dominio
│   ├── eps-disponibles.md
│   ├── proceso-afiliacion.md
│   └── tarifas-aportes.md
└── scripts/                     # Scripts de utilidad
    └── export-workflows.sh      # Exportar flujos de n8n
```

---

## Recordatorios Importantes

- Usar el MCP de n8n para **toda** creación/modificación de flujos — no editar JSON manualmente
- Los 7 skills de n8n-skills se deben activar antes de pedir a Claude que construya flujos complejos
- **Nunca** hardcodear tokens o API keys en los flujos — usar variables de entorno de n8n
- Datos sensibles del usuario (cédula, info de salud) deben manejarse con cifrado y cumplir la Ley 1581 de 2012 (Habeas Data Colombia)
- Siempre incluir opción de **escalado a humano** en flujos de atención