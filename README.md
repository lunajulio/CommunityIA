# Community IA · Grupo 10 · Equipo 23

## Integrantes

<table align="center">
  <tr>
    <td align="center" width="170">
      <img src="https://github.com/iamalonso.png" width="80" height="80"><br>
      <a href="https://github.com/iamalonso">@iamalonso</a><br>
      <img src="https://img.shields.io/badge/Project_Manager-6F42C1?style=flat-square">
    </td>
    <td align="center" width="170">
      <img src="https://github.com/lunajulio.png" width="80" height="80"><br>
      <a href="https://github.com/lunajulio">@lunajulio</a><br>
      <img src="https://img.shields.io/badge/Backend_Lead-2EA44F?style=flat-square">
    </td>
    <td align="center" width="170">
      <img src="https://github.com/alanbubu18-design.png" width="80" height="80"><br>
      <a href="https://github.com/alanbubu18-design">@alanbubu18-design</a><br>
      <img src="https://img.shields.io/badge/Backend_Dev-2EA44F?style=flat-square">
    </td>
    <td align="center" width="170">
      <img src="https://github.com/alejandramcr.png" width="80" height="80"><br>
      <a href="https://github.com/alejandramcr">@alejandramcr</a><br>
      <img src="https://img.shields.io/badge/Backend_Dev-2EA44F?style=flat-square">
    </td>
  </tr>
  <tr>
    <td align="center" width="170">
      <img src="https://github.com/Aldo-CL.png" width="80" height="80"><br>
      <a href="https://github.com/Aldo-CL">@Aldo-CL</a><br>
      <img src="https://img.shields.io/badge/Cloud_Engineer-0078D4?style=flat-square">
    </td>
    <td align="center" width="170">
      <img src="https://github.com/carmenNieves6478.png" width="80" height="80"><br>
      <a href="https://github.com/carmenNieves6478">@carmenNieves6478</a><br>
      <img src="https://img.shields.io/badge/Cloud_Engineer-0078D4?style=flat-square">
    </td>
    <td align="center" width="170">
      <img src="https://github.com/claudio-hcd.png" width="80" height="80"><br>
      <a href="https://github.com/claudio-hcd">@claudio-hcd</a><br>
      <img src="https://img.shields.io/badge/Frontend_Dev-E34F26?style=flat-square">
    </td>
    <td></td>
  </tr>
</table>



## Problema
Las empresas y agencias que gestionan comunidades digitales (en plataformas como Discord, LinkedIn, Slack, entre otras) se enfrentan a un cuello de botella operativo: la curaduría manual de contenidos. Los especialistas de marketing deben revisar una abrumadora cantidad de mensajes diarios para identificar manualmente qué interacciones de los usuarios tienen el potencial de convertirse en publicaciones de alto impacto. Este trabajo exige tomar decisiones complejas de forma repetitiva: filtrar lo más atractivo, definir el formato ideal (un testimonio, una noticia o una duda técnica) y elegir la plataforma de destino adecuada.

## Motivación
El alto volumen de datos e interacciones que se generan de forma orgánica en las comunidades hace que la revisión manual sea insostenible. Esto genera un alto riesgo de dejar pasar contenido valioso, como historias de éxito o testimonios de usuarios, debido a la fatiga operativa y la pesada carga de trabajo que implica evaluar cada interacción una por una.

## Solución propuesta
La solución propuesta se enmarca en Community IA, un motor inteligente impulsado por Inteligencia Artificial (Grandes Modelos de Lenguaje como Google Gemini) diseñado para automatizar la transformación de comunidades digitales. El sistema permite cargar un archivo JSON con las interacciones de la comunidad para que la IA analice, procese y extraiga métricas objetivas de cada mensaje (sentimiento, categoría, relevancia y resúmenes), convirtiéndolos automáticamente en activos de marketing listos para publicar en LinkedIn o Newsletters e incluso registros FAQ.

### Arquitectura del sistema
<img width="80%" alt="image" src="https://github.com/user-attachments/assets/33a155ed-b514-414f-9bd0-a1606d5fadc7" />

Arquitectura de despliegue en OCI basada en dos máquinas virtuales independientes para los contenedores de backend y frontend, respaldadas por un bucket de OCI Object Storage para la persistencia de los paquetes de contenido.

## El ciclo de vida propuesto
Para garantizar un flujo de trabajo ágil, seguro y con supervisión humana, el sistema opera mediante un ciclo sencillo de 4 pasos:

<img width="1488" height="206" alt="mermaid-diagram-2026-10-25-120934" src="https://github.com/user-attachments/assets/64c143ba-2c7b-4c7c-9740-0d2afc3533e5" />

# Instrucciones de uso
[![Ver demo](https://img.shields.io/badge/Ver_demo-FF0000?style=for-the-badge&logo=youtube&logoColor=white)](https://youtu.be/FErg-uWqOAQ)

## Capturas de la aplicación

<p align="center">
  <img src="assets/img1.png" width="600" alt="Captura de la bandeja de revisión">
  <img src="assets/img2.png" width="600" alt="Captura del detalle por interacción">
  <img src="assets/img3.png" width="600" alt="Captura del historial">
  <img src="assets/img4.png" width="600" alt="Captura del dashboard">

</p>


# Especificaciones técnicas

Community IA convierte las conversaciones de una comunidad de aprendizaje (Discord, CSV, formularios, GitHub) en **activos de comunicación listos para publicar**: posts de LinkedIn, destacados para el newsletter semanal y sugerencias de FAQ. Agentes con LLM (Google Gemini) analizan, clasifican y redactan; una persona revisa y aprueba cada pieza, y el paquete final se guarda en **OCI Object Storage**.

**Demo en vivo (OCI Always Free)**

| Servicio | URL |
| --- | --- |
| Aplicación web (bandeja de revisión, historial y dashboard) | <http://138.2.245.128:3000/> |
| API y documentación interactiva (Swagger) | <http://164.152.246.126:8000/docs> |

---

## Pipeline completo

![Pipeline completo de Community IA](docs/diagram.svg)

> Diagrama BPMN editable: [docs/FlujoCommunityLab.bpmn](docs/FlujoCommunityLab.bpmn) (se abre en <https://demo.bpmn.io> o Camunda Modeler).
> En el diagrama, **✦ LLM** marca las decisiones que toma un modelo, ⚙ las tareas de sistema, 📋 las reglas deterministas y 👤 las decisiones humanas.

1. **Carga del lote.** El Community Manager sube un JSON (hasta 500 interacciones). El backend crea un trabajo en segundo plano (`en_cola` → `en_proceso` → `completado` / `fallido`) y la interfaz consulta su avance cada 2 s, mostrando cada interacción apenas termina.
2. **Procesamiento por interacción**, una a la vez, con una pausa de 3 s entre interacciones para respetar la cuota de Gemini. Si Gemini responde 429, se espera 4.5 s y se reintenta una vez.
   - **✦ Agente Analista:** extrae sentimiento (-1 a 1), temas, entidades, resumen e intención. Si falla, usa un análisis neutral marcado `degradado` y el flujo continúa. El campo `tipo` de la fuente nunca se usa para clasificar.
   - **✦ Agente Clasificador (LangGraph):** propone una categoría con su confianza. Si no responde tras 2 intentos, o si su confianza es menor a **0.75**, se usa la categoría del clasificador por reglas. Luego calcula el `relevance_score` (0–100) y decide la ruta: generar activos, alerta de soporte o descartar.
   - **✦ Agente Redactor:** según la categoría elige los formatos (éxito, testimonio o post social → LinkedIn + Newsletter; FAQ o contenido educativo → FAQ + LinkedIn) y los redacta usando solo hechos del análisis. Si el LLM falla, usa plantillas. Todo candidato nace `pendiente`.
3. **Revisión humana (HITL).** Cada candidato se aprueba, se edita y aprueba, o se rechaza. "Guardar cambios" queda bloqueado mientras quede alguno `pendiente`.
4. **Publicación.** Por cada formato oficial gana el candidato **aprobado** con mayor `relevance_score`. Se calcula el resumen de la comunidad y el paquete se guarda en OCI como `activos/paquete_{N}/paquete-distribucion.json`.
5. **Historial y dashboard.** La interfaz lista los paquetes guardados, muestra 4 KPIs y 4 gráficas (sentimiento, activos por canal, temas y relevancia) y permite inspeccionar cada caso en modo de solo lectura.

Un error en cualquier fase queda registrado en el campo `error` de esa interacción y **no detiene el resto del lote**.

## Arquitectura y stack

Monolito modular: las fases del pipeline son módulos de un único backend, no microservicios.

| Capa | Tecnología |
| --- | --- |
| Frontend | Astro 7 + React (bandeja HITL, historial, dashboard) con rutas BFF `/api/*` que actúan de proxy hacia la API |
| Backend | FastAPI + Pydantic v2, LangChain + LangGraph |
| LLM | Google Gemini vía `langchain-google-genai` (recomendado `gemini-3.5-flash-lite`) |
| Persistencia | SQLite local para autenticación, lotes, candidatos e historial; OCI Object Storage para paquetes publicados |
| Infraestructura | Docker Compose; 2 VMs `VM.Standard.E2.1.Micro` (RAM 1GB - SWAP 4GB) de OCI Always Free (región `sa-vinhedo-1`) |

## Inicio rápido (local)

Requisitos: Docker Desktop con Docker Compose v2 y una clave de Gemini.

```bash
cp .env.example .env          # completa GEMINI_API_KEY
docker compose up --build     # backend en :8000 y frontend en :3000
```

- Aplicación: <http://localhost:3000> · API: <http://localhost:8000/docs> · Salud: `curl http://localhost:8000/health`
- **Sin credenciales de OCI** el backend usa un almacenamiento local equivalente (`backend/data/oci_mock/`), así que guardar paquetes e historial funcionan igual en desarrollo.
- **Sin `GEMINI_API_KEY`** la API arranca, pero los lotes terminan en `fallido`.
- El `.env` se lee al crear el contenedor: después de cambiarlo, ejecuta `docker compose up -d --force-recreate backend`.
- **Cuota de Gemini:** el límite gratuito es por proyecto y por modelo, y cada interacción consume de 3 a 5 solicitudes. Usa `GEMINI_MODEL=gemini-3.5-flash-lite` y consulta tus límites en <https://ai.dev/rate-limit>.

## Prueba local de la interfaz con datos demo

Para visualizar la aplicación sin configurar Gemini, OCI, usuarios ni el backend, activa el modo de demostración del frontend. Este modo es intencional: muestra resultados simulados y no guarda cambios en SQLite ni publica paquetes en OCI.

Desde PowerShell, en la raíz del repositorio:

```powershell
Copy-Item .env.example .env
notepad .env
```

En `.env`, define:

```dotenv
DEMO_MODE=true
```

Después levanta el frontend:

```powershell
docker compose up -d --build frontend
```

Abre <http://localhost:3000/>. En este modo no necesitas iniciar sesión. En la cabecera usa **Datos de prueba** para cargar:

- **Caso oficial**, para explorar historias de éxito, preguntas técnicas, contenido de comunidad y candidatos en revisión.
- **Lote sin oportunidades**, para comprobar la vista de interacciones descartadas sin activos generados.

También puedes usar **Cargar JSON** o arrastrar un archivo de ejemplo desde [`examples/inputs/demo-mixed-batch.json`](examples/inputs/demo-mixed-batch.json). El simulador clasifica el contenido a partir del texto: mensajes con términos como “empleo” o “logro” generan una historia de éxito; preguntas generan una propuesta FAQ; errores de acceso generan una alerta de soporte; y textos muy breves se descartan.

En la interfaz puedes cambiar entre las pestañas **Todos**, **LinkedIn**, **Newsletter**, **FAQ** y **Alertas de soporte**, abrir el detalle de una interacción y aprobar, rechazar o editar candidatos. Los cambios del modo demo viven solo en la sesión del navegador; las vistas de historial y paquetes publicados requieren el backend conectado.

Para volver al modo normal:

```powershell
(Get-Content .env) -replace '^DEMO_MODE=.*$', 'DEMO_MODE=false' | Set-Content .env
docker compose up -d --build frontend
```

Si prefieres ejecutar Astro sin Docker, instala Node.js 22.23 o superior:

```powershell
cd frontend
npm install
$env:DEMO_MODE = "true"
npm run dev
```

## Prueba local con inicio de sesión

Para probar la autenticación real, desactiva el modo demo y levanta ambos servicios:

```powershell
(Get-Content .env) -replace '^DEMO_MODE=.*$', 'DEMO_MODE=false' | Set-Content .env
docker compose up -d --build
```

En una terminal adicional, crea el primer administrador:

```powershell
docker compose exec backend python -m app.auth.crear_admin --email admin@example.com --nombre "Administrador local"
```

El comando solicitará la contraseña de forma interactiva. Después:

1. Abre <http://localhost:3000/login>.
2. Introduce `admin@example.com` y la contraseña definida.
3. Comprueba que puedes acceder a la bandeja, historial, dashboard y **Administración → Equipo**.
4. Usa **Cerrar sesión** y verifica que la aplicación vuelve a solicitar autenticación.

Las cuentas adicionales no se registran libremente: créalas desde **Administración → Equipo** mediante una invitación. El enlace permite definir la contraseña en `/registro`; para una cuenta existente, administración puede generar un enlace de recuperación. Los cambios de rol y desactivación se pueden comprobar cerrando sesión y volviendo a solicitar una ruta protegida.

Para comprobar la separación de permisos, crea usuarios con los roles `lector` y `revisor`: el lector puede consultar historial y dashboard, mientras que el revisor también puede cargar lotes y revisar candidatos. Solo `admin` puede gestionar usuarios y publicar paquetes.

## Guía de despliegue en OCI

Arquitectura de dos capas en OCI Always Free: una VM para el frontend (puerto 3000), otra para el backend (puerto 8000) y un bucket de Object Storage. El paso a paso completo (compartimento, permisos, red, bucket, instancias, backend, frontend, verificación y operación) está en **[docs/Guia_Despliegue_OCI.md](docs/Guia_Despliegue_OCI.md)**.

Resumen de comandos, una vez preparada cada VM:

```bash
# VM backend (.env con GEMINI_API_KEY y credenciales de OCI en ~/.oci o instance principals)
docker compose up -d --build backend

# VM frontend (.env con BACKEND_URL=http://<IP_PRIVADA_BACKEND>:8000)
docker compose up -d --build frontend

# Actualizar una versión
cd ~/communityia && git pull && docker compose up -d --build backend   # o frontend
```

## API

| Método | Ruta | Propósito |
| --- | --- | --- |
| `GET` | `/health` | Estado del servicio. |
| `POST` | `/api/v1/auth/login` | Inicia sesión y devuelve el token de sesión. |
| `POST` | `/api/v1/auth/registro` | Crea una cuenta mediante una invitación. |
| `POST` | `/api/v1/auth/restablecer` | Define una nueva contraseña con un enlace de recuperación. |
| `GET` | `/api/v1/auth/me` | Consulta la sesión y el usuario actual. |
| `POST` | `/api/v1/auth/logout` | Cierra la sesión actual. |
| `GET` | `/api/v1/usuarios` | Lista usuarios (solo administración). |
| `POST` | `/api/v1/usuarios/invitaciones` | Crea una invitación (solo administración). |
| `PATCH` | `/api/v1/usuarios/{usuario_id}` | Actualiza rol o estado de un usuario (solo administración). |
| `POST` | `/api/v1/pipeline/jobs` | Procesa un lote en segundo plano (A → B → C). Responde `202` con un `job_id`. Es lo que usa la interfaz. |
| `GET` | `/api/v1/pipeline/jobs/{job_id}?desde=n` | Estado y avance del trabajo, con las interacciones procesadas desde el índice `n`. |
| `POST` | `/api/v1/pipeline/process` | El mismo flujo, síncrono: responde al terminar el lote. |
| `POST` | `/api/v1/ingest/process` | Solo el análisis del Equipo A, sin clasificación ni generación. |
| `GET` | `/api/v1/review/assets` | Lista los candidatos y su estado de aprobación. |
| `PATCH` | `/api/v1/review/assets/{candidato_id}` | Aprueba, rechaza o edita un candidato. |
| `POST` | `/api/v1/storage/publish` | Elige los ganadores, arma el paquete oficial y lo guarda en OCI. |
| `GET` | `/api/v1/storage/packages` | Lista los paquetes guardados con su contenido. |
| `GET` | `/api/v1/storage/packages/{ruta}` | Descarga un paquete. |
| `GET` | `/api/v1/storage/status` | Configuración de OCI (bucket y namespace). |

Ejemplo de solicitud de lote (ver también [schemas/input.json](schemas/input.json)):

```json
{
  "origen_comunidad": "Discord",
  "periodo_referencia": "2026-W40",
  "interacciones": [
    { "autor": "Mariana", "canal": "#logros", "tipo": "mensaje",
      "texto": "¡Conseguí mi primer empleo como desarrolladora backend gracias al bootcamp!" }
  ]
}
```

`POST /api/v1/storage/publish` recibe `{ "lote_id": "..." }`. El backend lee del servidor las decisiones de revisión, selecciona por cada formato el candidato aprobado con mayor `relevance_score`, genera el paquete con su auditoría y lo persiste en OCI. La respuesta sigue el contrato `RespuestaProcesamiento` ([schemas/output.json](schemas/output.json)): `status`, `resumen_comunidad`, `activos_distribucion_generados` (`post_linkedin`, `destaque_newsletter_semanal`, `sugerencia_contenido_faq`; un formato sin aprobados se omite) y `almacenamiento_oci`.

## Pruebas

```bash
cd backend
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
python -m pip install -r requirements.txt
python -m pytest -q
```

La suite simula Gemini y OCI: no requiere credenciales ni consume cuota. Con Docker: `docker compose exec backend python -m pytest -q`.

## Estructura del repositorio

```text
backend/app/
  api/v1/         Routers: auth, usuarios, ingest, pipeline, review, storage
  ai/             Equipo A: análisis estructurado con LLM
  intelligence/   Equipo B: clasificador LLM, reglas y relevance_score
  workflow/       Grafo LangGraph, orquestación A→B→C, jobs y persistencia de lotes
  content/        Equipo C: generación de activos, candidatos e historial de revisión
  auth/            Sesiones, roles, invitaciones y recuperación de contraseña
  persistence/     Conexión y migraciones de SQLite
  storage/        Equipo D: ensamblado del paquete y cliente de OCI Object Storage
  contracts/      Modelos Pydantic y enums compartidos (contrato externo e interno)
backend/tests/    Pruebas por módulo
frontend/         Astro + React: bandeja HITL, historial, dashboard y proxy BFF
docs/             Arquitectura, contratos, diagramas y documentación por equipo
schemas/          Ejemplos JSON de entrada y salida
```

## Limitaciones

- Los trabajos se ejecutan en un único hilo en segundo plano y asumen un solo worker de Uvicorn. Si el proceso se reinicia, los lotes `en_cola` o `en_proceso` se marcan como `fallido`; las interacciones ya procesadas se conservan en SQLite.
- La validación de que no quedan candidatos pendientes se hace en la interfaz antes de publicar; el backend vuelve a leer las decisiones guardadas y solo publica candidatos aprobados.
- El número de paquete se calcula contando los existentes: dos publicaciones simultáneas podrían usar el mismo número.

## Documentación

- [Arquitectura](docs/Architecture.md) · [Contratos de datos](docs/DataContract.md)
- [Equipo D: almacenamiento OCI, dashboard y despliegue](docs/Documentacion_Equipo_D_y_Despliegue_OCI.md)
- Equipos: [A · Ingesta y análisis](docs/Equipo%20A%20-%20Ingesta%20y%20analisis.md) · [B · Clasificación, score y router](docs/Equipo%20B%20-%20Clasificacion%20score%20y%20router.md) · [C · Generación y HITL](docs/Equipo%20C%20-%20Generacion%20y%20HITL.md) · [D · Storage y dashboard](docs/Equipo%20D%20-%20Storage%20dashboard%20y%20RAG.md)
- [Plan integral de equipos](docs/CommunityLab%20-%20Plan%20de%20equipos.md) · [Frontend](frontend/README.md)
