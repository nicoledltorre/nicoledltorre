# SmartPM — Agentes

Cómo se reparte el trabajo en este repo. Las reglas del proyecto están en `CLAUDE.md`
y las fases en `docs/PLAN.md`; este archivo solo define **quién hace qué**.

## Equipo

| Agente | Responsabilidad única | Puede escribir en | Fases |
|---|---|---|---|
| `orquestador` | Planificar la fase, delegar, mantener el tablero, cerrar la fase | `docs/TABLERO.md`, `docs/PLAN.md` (seguimiento) | todas |
| `backend` | API FastAPI, base de datos, infraestructura | `backend/`, `docker-compose.yml`, `Makefile`, `.github/`, `.env.example`, `docs/deploy.md` | F0–F3, F9 y la parte de API de F5–F7, F10 |
| `ml` | Datos, entrenamiento y evaluación de modelos | `ml/`, `docs/resultados/` | F4–F7, F10 |
| `frontend` | Interfaz React | `frontend/`, `docs/capturas/` | F0, F8 |
| `qa` | Verificar los criterios de "Hecho" | nada (solo lectura y ejecución) | cierre de cada fase |

Las definiciones están en `.claude/agents/*.md`.

## Flujo de una fase

1. Se arranca con `/fase F3` (o "Estamos en la fase F3").
2. El **orquestador** lee la fase en `docs/PLAN.md`, la parte en tareas, propone un plan corto y **espera el OK**.
3. Anota las tareas en `docs/TABLERO.md` y delega cada una al agente dueño de esa carpeta.
   Tareas en carpetas distintas van en paralelo; tareas en la misma carpeta, en serie.
4. Cada agente trabaja solo en su carpeta, corre sus tests y devuelve un reporte:
   qué hizo, archivos tocados, resultado de los tests, pendientes.
5. El orquestador actualiza el tablero con cada reporte.
6. `qa` verifica uno por uno los criterios de "Hecho" de la fase y responde ✅/❌ con evidencia.
7. Con todo en ✅: commit, se marca la fase en el seguimiento de `docs/PLAN.md` y se limpia el tablero.

## Cómo ver en qué trabaja cada agente

- **`docs/TABLERO.md`** — una fila por agente: estado, tarea actual y archivos. Es la vista rápida.
- El panel de tareas de Claude Code muestra los agentes que están corriendo en vivo.

El tablero tiene **un solo escritor: el orquestador**. Los demás agentes informan en su
reporte y no lo editan. Así nunca hay dos agentes pisándose el mismo archivo.

## Reglas entre agentes

- Un agente no edita fuera de su columna "Puede escribir en". Si necesita un cambio ajeno
  (ej.: `frontend` necesita un campo nuevo en un endpoint), lo pide en su reporte y el
  orquestador se lo delega al dueño.
- El contrato entre `backend` y `frontend` es el OpenAPI (`/docs`). El contrato entre `ml` y
  `backend` son los `.joblib` en `backend/app/ml/models/` y las interfaces de `backend/app/ml/`.
- Ningún agente escribe un número de resultados a mano. `qa` rechaza la fase si encuentra uno.
- Si algo falla dos veces seguidas, el agente se detiene y reporta la causa en vez de reintentar.

## Diseño SOLID (obligatorio en el código)

Lo justo: estas cinco reglas y nada más. Sin capas genéricas, sin patrones "por las dudas".

| Principio | Cómo se aplica acá |
|---|---|
| **S** — responsabilidad única | `routers/` solo HTTP (parsear, llamar al servicio, devolver). `services/` lógica de negocio. `models/` persistencia. `schemas/` validación. Un script de entrenamiento por modelo. Un componente React hace una cosa; las llamadas HTTP viven en `src/api/`. |
| **O** — abierto/cerrado | Los modelos a comparar en `train_*.py` salen de un diccionario `{nombre: (estimador, grilla)}`: agregar uno no toca el bucle. Las intenciones del asistente son un registro `{intención: handler}`: agregar una no toca el clasificador. |
| **L** — sustitución | Toda implementación de una interfaz se puede cambiar por otra sin romper al que la usa. Los tests usan un predictor falso en lugar del modelo real y el servicio no se entera. |
| **I** — interfaces chicas | `Protocol` separados en `backend/app/ml/`: `PredictorRiesgo`, `EstimadorEsfuerzo`, `Embedder`. No un "servicio de IA" gigante. |
| **D** — inversión de dependencias | Los servicios reciben sesión de BD y predictores por `Depends` de FastAPI; dependen del `Protocol`, no de joblib ni de sentence-transformers. Los modelos se cargan una sola vez al arrancar. |
