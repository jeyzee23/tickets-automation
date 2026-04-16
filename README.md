# 🎫 Pipeline de Tickets E-commerce — Guía de Setup

## Qué es este workflow

Un pipeline completo de atención automática de tickets de soporte para e-commerce, construido en n8n. Integra los conceptos clave del curso:

- **Clasificación con IA** — Un modelo (Gemini Flash via OpenRouter) clasifica el ticket y genera una respuesta sugerida
- **Evaluación automática (Evals)** — Dos capas de evaluación validan la respuesta antes de enviarla:
  - Eval por reglas heurísticas (largo, tono, datos sensibles, promesas falsas)
  - Eval por LLM jurado (relevancia, empatía, claridad, seguridad — puntaje de 0 a 40)
- **Human-in-the-Loop (HITL)** — Si la confianza es baja, el tema es sensible, o las evals rechazan la respuesta, el workflow se pausa y espera aprobación humana via formulario
- **Fallbacks** — Reintentos automáticos en la API, fallback a reglas si el LLM jurado falla, escalamiento a supervisor si el operador no responde

---

## Arquitectura del flujo

```
Webhook (POST)
    │
    ▼
✅ Validar & Formatear ─── valida campos, genera ticket_id
    │
    ▼
🤖 Clasificar con IA ──── OpenRouter API (Gemini Flash)
    │                      retry x3 si falla
    ▼
📋 Parsear Clasificación ─ extrae category, confidence, suggested_response
    │                      fallback si el JSON viene mal
    ▼
📏 Eval Rules ──────────── 5 reglas heurísticas (score X/5)
    │
    ▼
🧑‍⚖️ Eval LLM Jurado ────── segundo modelo evalúa la respuesta (score X/40)
    │                      continue on fail activado
    ▼
📊 Consolidar Evals ────── combina ambas evals, decide si forzar HITL
    │
    ▼
🔀 ¿Necesita HITL? ─────── IF: needs_hitl == true
    │
    ├── TRUE ──▶ ⏳ Wait (Formulario) ──▶ 🔀 ¿Respondió? ──┬── ✅ Operador Respondió
    │                                                       └── 🔄 Fallback Escalar
    │
    └── FALSE ─▶ ⚡ Ruta Automática
                        │
                        ▼
                  💾 Registrar Resultado
```

---

## Setup paso a paso

### 1. Crear cuenta en OpenRouter (gratis)

1. Ir a [openrouter.ai](https://openrouter.ai)
2. Registrarse (se puede con Google)
3. Ir a [openrouter.ai/settings/keys](https://openrouter.ai/settings/keys)
4. Click **Create Key** → copiar la key (empieza con `sk-or-v1-...`)

> El workflow usa `google/gemini-2.0-flash-001` que tiene free tier, no necesitás cargar crédito.

### 2. Configurar n8n

#### Si usás n8n Cloud:

1. Ir a **Settings > Variables**
2. Click **Add Variable**
3. Name: `OPENROUTER_API_KEY`
4. Value: pegar tu key de OpenRouter

#### Si usás n8n self-hosted:

Setear la variable de entorno antes de levantar n8n:

```bash
export OPENROUTER_API_KEY=sk-or-v1-tu-key-aca
```

### 3. Importar el workflow

1. En n8n, ir al menú **⋯** (tres puntos arriba a la derecha)
2. Click **Import from File**
3. Seleccionar el archivo `Pipeline_Tickets_E-commerce_HITL_Fallbacks.json`

### 4. Verificar el header de Authorization

Abrir el nodo **🤖 Clasificar con IA** y verificar que el header `Authorization` tenga el botón **fx** activado y diga:

```
Bearer {{ $vars.OPENROUTER_API_KEY }}
```

Hacer lo mismo en el nodo **🧑‍⚖️ Eval LLM Jurado**.

> Si no funciona con `$vars`, probar con `$env.OPENROUTER_API_KEY` (depende de si es Cloud o self-hosted).

---

## Cómo probarlo

### Opción A: Modo Test (una ejecución por vez)

1. Click en **Execute workflow** (botón abajo del canvas)
2. n8n queda esperando el POST
3. Desde Postman, Reqbin, o curl, mandar el request

### Opción B: Modo Producción

1. Click en **Publish** (arriba a la derecha)
2. El webhook queda activo permanentemente

### URL del webhook

- **Modo test:** `https://TU-INSTANCIA.app.n8n.cloud/webhook-test/ticket-ingreso`
- **Modo producción:** `https://TU-INSTANCIA.app.n8n.cloud/webhook/ticket-ingreso`

### Enviar un ticket

**POST** a la URL del webhook con body JSON:

```json
{
  "customer": "María González",
  "message": "No me llegó el paquete y dice entregado",
  "email": "maria@test.com"
}
```

Desde **curl**:

```bash
curl -X POST https://TU-INSTANCIA.app.n8n.cloud/webhook/ticket-ingreso \
  -H "Content-Type: application/json" \
  -d '{"customer": "María González", "message": "No me llegó el paquete y dice entregado", "email": "maria@test.com"}'
```

---

## Tickets de ejemplo para probar

### Ruta automática (confianza alta, evals pasan)

```json
{
  "customer": "Laura Méndez",
  "message": "Necesito factura A para mi empresa, ¿cómo la pido?",
  "email": "laura@test.com"
}
```

```json
{
  "customer": "Pedro Díaz",
  "message": "La app no me deja agregar productos al carrito desde ayer",
  "email": "pedro@test.com"
}
```

### Ruta HITL (devolución = tema sensible)

```json
{
  "customer": "Jorge Sánchez",
  "message": "Me mandaron el producto equivocado y quiero que me devuelvan la plata",
  "email": "jorge@test.com"
}
```

```json
{
  "customer": "Carlos Ruiz",
  "message": "Me cobraron dos veces la misma compra en la tarjeta de crédito",
  "email": "carlos@test.com"
}
```

### Mensaje ambiguo (confianza baja → HITL)

```json
{
  "customer": "Ana López",
  "message": "Hola tengo un problema no sé bien qué es pero algo anda mal",
  "email": "ana@test.com"
}
```

---

## Cómo funciona el HITL

Cuando un ticket necesita revisión humana:

1. El workflow se **pausa** en el nodo Wait
2. n8n genera una **URL de formulario**
3. Al abrir esa URL, aparece un form con:
   - **Acción**: approve / edit / reject (dropdown)
   - **Comentarios**: campo de texto libre
4. Al enviar el form, el workflow **continúa**

Para encontrar la URL del formulario:
- Ir a la pestaña **Executions** en n8n
- Buscar la ejecución con estado **"Waiting"**
- Abrirla → el nodo Wait muestra la URL

---

## Qué hace cada nodo

| Nodo | Tipo | Qué hace |
|------|------|----------|
| **Webhook** | Trigger | Recibe el POST con los datos del ticket |
| **✅ Validar & Formatear** | Code | Valida campos, genera ticket_id, formatea |
| **🤖 Clasificar con IA** | HTTP Request | Llama a OpenRouter para clasificar y generar respuesta |
| **📋 Parsear Clasificación** | Code | Extrae el JSON de la IA, determina si necesita HITL |
| **📏 Eval Rules** | Code | 5 reglas heurísticas: largo, tono, promesas, categoría, datos sensibles |
| **🧑‍⚖️ Eval LLM Jurado** | HTTP Request | Segundo modelo evalúa la respuesta (4 criterios, score /40) |
| **📊 Consolidar Evals** | Code | Combina evals, fuerza HITL si alguna rechaza |
| **🔀 ¿Necesita HITL?** | IF | Bifurca el flujo: automático o revisión humana |
| **⏳ Wait — Revisión Humana** | Wait (Form) | Pausa y muestra formulario de aprobación |
| **🔀 ¿Respondió o Timeout?** | IF | Verifica si el operador respondió o venció el plazo |
| **✅ Operador Respondió** | Code | Procesa la decisión del operador |
| **🔄 Fallback — Escalar** | Code | Escala a supervisor si nadie respondió |
| **⚡ Ruta Automática** | Code | Prepara la respuesta para envío automático |
| **💾 Registrar Resultado** | Code | Registra métricas: decisión, latencia, categoría |

---

## Evaluaciones automáticas (Evals)

### Eval por Reglas (📏 Eval Rules)

Evalúa 5 reglas heurísticas. Cada una suma 1 punto. Aprueba con 4/5 o más.

| Regla | Qué verifica |
|-------|-------------|
| `largo_minimo` | La respuesta tiene más de 30 caracteres |
| `sin_promesas_falsas` | No contiene "inmediatamente", "al instante", "ya mismo" |
| `tono_empatico` | Contiene algún saludo o expresión empática |
| `categoria_valida` | La categoría está en la lista permitida |
| `sin_datos_sensibles` | No hay patrones de DNI o tarjeta de crédito |

### Eval por LLM Jurado (🧑‍⚖️)

Un segundo modelo evalúa la respuesta en 4 dimensiones (0-10 cada una):

| Criterio | Qué mide |
|----------|----------|
| **Relevancia** | ¿Responde al problema real del cliente? |
| **Empatía** | ¿Es empática y profesional? |
| **Claridad** | ¿Es clara y accionable? |
| **Seguridad** | ¿No promete cosas imposibles? |

Aprueba con **28/40** o más. Si el LLM jurado falla (API down), el sistema usa **fallback a las reglas** heurísticas.

### Decisión consolidada

| Resultado | Condición | Acción |
|-----------|-----------|--------|
| `aprobado` | Rules ✅ + LLM ✅ | Sigue normalmente |
| `aprobado_con_advertencia` | Rules ✅ + LLM falló | Sigue pero loguea warning |
| `rechazado_rules` | Rules ❌ | Fuerza HITL |
| `rechazado_llm` | LLM ❌ | Fuerza HITL |

---

## Conceptos del curso aplicados

| Concepto | Dónde se aplica en el workflow |
|----------|-------------------------------|
| APIs REST + webhooks | Webhook de ingreso, llamadas a OpenRouter |
| Fallbacks | Retry x3 en la API, fallback a reglas si LLM falla, escalar si timeout |
| Observabilidad / Logs | Cada nodo Code registra timestamps y métricas |
| HITL | Wait node con formulario, 3 acciones posibles |
| Eval rule-based | Nodo 📏 Eval Rules (5 reglas heurísticas) |
| Eval LLM as judge | Nodo 🧑‍⚖️ Eval LLM Jurado (4 criterios semánticos) |
| Output schemas | Respuestas en JSON estructurado con response_format |
| Temperatura | 0.2 para clasificación (determinista), 0.1 para evaluación |
| Secrets | API key en variable de entorno, nunca hardcodeada |
| Principio de mínimos privilegios | La key solo tiene permisos de completions |

---

## Ideas para extender

- Agregar un nodo de **email** (Gmail, SMTP) para enviar la respuesta aprobada al cliente
- Conectar **Slack** para notificar al equipo cuando un ticket entra en HITL
- Guardar los resultados en **Google Sheets** o **Supabase** como dashboard de métricas
- Agregar más reglas al Eval Rules (ej: detectar lenguaje ofensivo, verificar que mencione número de pedido)
- Crear un **dataset de evaluación offline**: varios tickets con respuestas esperadas, correr todos y medir scores
- Implementar **A/B testing de prompts**: comparar dos system prompts distintos usando las evals como métrica