# AI Control Demo: Observabilidad de LLMs con LiteLLM, OpenTelemetry y Jaeger

Entorno local de observabilidad y pasarela para modelos de lenguaje (LLMs).

Este proyecto implementa una arquitectura basada en proyectos 100% de código abierto que permite monitorizar, auditar y
trazar peticiones a LLMs mediante el estándar **OpenTelemetry (OTel)**, visualizando latencias, costes, tokens
consumidos y prompts en **Jaeger**.

---

## 🏗️ Arquitectura del Stack

```text
               +---------------------------------------+
               |        Cliente / Aplicación           |
               +---------------------------------------+
                                   | (Petición HTTP / OpenAI API)
                                   v
+---------------------------------------------------------------------+
|                      LiteLLM Proxy (:4000)                          |
|             (API Gateway compatible con OpenAI)                     |
+---------------------------------------------------------------------+
       |                                          |
       | (Llamada al LLM)                         | (Exportación OTel v2 vía gRPC)
       v                                          v
+-----------------------------+        +------------------------------+
| Proveedor LLM               |        |  OpenTelemetry Collector     |
| - Ollama (Local :11434)     |        |          (:4317)             |
| - OpenAI / Anthropic (Cloud)|        +------------------------------+
+-----------------------------+                       |
                                                      | (Pipeline OTLP)
                                                      v
                                       +------------------------------+
                                       |          Jaeger              |
                                       |      (UI en :16686)          |
                                       +------------------------------+
```

### Componentes

1. **LiteLLM Proxy (`:4000`)**: Pasarela API unificada y compatible con el formato de OpenAI. Enruta peticiones a
   diferentes proveedores y emite métricas/trazas OTel.
2. **OpenTelemetry Collector Contrib (`:4317`)**: Recibe las trazas emitidas por LiteLLM por gRPC, procesa los datos en
   lotes (`batch`) y los reenvía a Jaeger.
3. **Jaeger (`:16686`)**: Backend de almacenamiento y visualización gráfica para inspeccionar trazas de ejecución,
   árboles de llamadas y metadatos de las peticiones.

---

## 📁 Estructura del Repositorio

```text
ai-control-demo/
├── docker-compose.yml           # Orquestación de contenedores del stack
├── litellm-config.yaml          # Configuración de modelos y callbacks OTel en LiteLLM
├── otel-collector-config.yaml   # Pipeline de recepción y exportación de OTel
├── README.md                    # Documentación del proyecto
└── test/
    └── prompt.md                # Carga útil (payload JSON) de ejemplo para pruebas
```

---

## ⚙️ Requisitos Previos

- [Docker](https://docs.docker.com/get-docker/) y [Docker Compose](https://docs.docker.com/compose/) instalados y en
  ejecución.
- *(Opcional, para ejecución 100% gratuita)* [Ollama](https://ollama.com/) ejecutándose localmente con el modelo
  `llama3` descargado:
  ```bash
  ollama run llama3
  ```
- *(Opcional, si usas proveedores cloud)* Una clave de API de OpenAI (`OPENAI_API_KEY`).

---

## 🚀 Guía de Inicio Rápido

### 1. Variables de entorno (Opcional)

- **`LITELLM_MASTER_KEY`**: Clave de autorización del proxy. Por defecto viene preconfigurada como `sk-master-key-demo-1234` en `docker-compose.yml`. Si deseas personalizarla:
  - Linux / macOS: `export LITELLM_MASTER_KEY="tu-clave-secreta"`
  - PowerShell: `$env:LITELLM_MASTER_KEY="tu-clave-secreta"`

- **`OPENAI_API_KEY`**: Si vas a interactuar con modelos de pago como OpenAI (`gpt-4o-mini`), define la variable en tu entorno antes de arrancar los servicios:
  - Linux / macOS: `export OPENAI_API_KEY="sk-..."`
  - PowerShell: `$env:OPENAI_API_KEY="sk-..."`

*(Si únicamente utilizas Ollama con la clave por defecto, no es necesario configurar variables).*

### 2. Iniciar el Stack

Ejecuta el siguiente comando en la raíz del proyecto para descargar las imágenes e inicializar los tres servicios en segundo plano:

```bash
docker compose up -d
```

### 3. Verificar Servicios

Comprueba que los contenedores estén levantados y accesibles:

| Servicio | URL / Endpoint | Propósito |
| :--- | :--- | :--- |
| **LiteLLM Proxy** | `http://localhost:4000` | Pasarela de inferencia (OpenAI Compatible) |
| **Jaeger UI** | `http://localhost:16686` | Interfaz gráfica de observabilidad y trazas |
| **OTel Collector** | `http://localhost:4317` | Puerto gRPC receptor de telemetría |

---

## 🧪 Pruebas de Inferencia

Genera eventos y trazas enviando una petición al endpoint `/v1/chat/completions` de LiteLLM (autenticándote con la clave maestra):

### Opción A: Mediante cURL (Bash / Linux / macOS)

```bash
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-master-key-demo-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3",
    "messages": [
      {"role": "user", "content": "Hola, genera una prueba de traza"}
    ]
  }'
```

### Opción B: Mediante PowerShell (Windows)

```powershell
Invoke-RestMethod -Uri "http://localhost:4000/v1/chat/completions" `
  -Method Post `
  -Headers @{ 
    "Authorization" = "Bearer sk-master-key-demo-1234"
    "Content-Type"  = "application/json" 
  } `
  -Body (@{
    model = "llama3"
    messages = @(@{ role = "user"; content = "Hola, genera una prueba de traza" })
  } | ConvertTo-Json -Depth 5)
```

### Opción C: Usando el archivo de prueba incluido

Puedes enviar directamente el payload de `test/prompt.md`:

```bash
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-master-key-demo-1234" \
  -H "Content-Type: application/json" \
  --data-binary "@test/prompt.md"
```

---

## 🔍 Inspección de Trazas en Jaeger

1. Accede con tu navegador web a: **[http://localhost:16686](http://localhost:16686)**.
2. En el panel lateral izquierdo, bajo el selector **Service**, selecciona `litellm` o `litellm-proxy`.
3. Haz clic en el botón **Find Traces**.
4. Selecciona cualquiera de las trazas listadas para desplegar su detalle completo. Podrás inspeccionar:
    - **Tiempo de respuesta y latencia** desagregada.
    - **Consumo de tokens** (prompt tokens, completion tokens y coste calculado).
    - **Modelo invocado** y parámetros de inferencia (temperatura, etc.).
    - **Mensajes y prompts** procesados durante la petición.

---

## 🛑 Detener el Stack

Para detener y retirar los contenedores creados:

```bash
docker compose down
```
