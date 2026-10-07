# SmartPM — Plan de desarrollo con Claude Code

**Hoy:** 05/10/2026 · **Defensa:** 23/11/2026 · **Tiempo disponible:** ~7 semanas

## Cómo usar este plan
1. Creá la carpeta `smartpm/`, poné `CLAUDE.md` en la raíz y este archivo en `docs/PLAN.md`.
2. Abrí Claude Code en esa carpeta.
3. Para cada fase, pegá el **Prompt** tal cual. Revisá el plan que te propone antes de decir "dale".
4. No pases a la fase siguiente hasta tildar todos los criterios de **Hecho**.
5. Al cerrar cada fase: `git commit` + anotá en el cuadro de seguimiento (abajo).

> Regla de oro: los números del Capítulo 4 de la tesis hoy son **valores de referencia**, no resultados.
> Este plan los reemplaza por números reales generados por código (Fase 10).

---

## Calendario

| Semana | Fechas | Fases |
|---|---|---|
| 1 | 05–11 oct | F0 Setup · F1 Base de datos · F2 Auth |
| 2 | 12–18 oct | F3 Proyectos/Tareas/Equipo (API) · F4 Datos ML |
| 3 | 19–25 oct | F5 Modelo de riesgo · F6 Modelo de esfuerzo |
| 4 | 26 oct–01 nov | F7 Asistente PM · F8 Frontend base |
| 5 | 02–08 nov | F8 Frontend (dashboard + IA) · F9 Deploy |
| 6 | 09–15 nov | F10 Validación y resultados reales · SUS con usuarios |
| 7 | 16–22 nov | F11 Actualizar tesis + ensayo de defensa |

Margen: si algo se atrasa, se recorta **F7b** (RAG real) y **RF-08** (asignación de recursos) — en ese orden.

---

## F0 — Setup del repo (½ día)

**Prompt**
```
Leé CLAUDE.md. Creá el esqueleto del monorepo: backend/ (FastAPI con /health),
frontend/ (Vite + React + Tailwind), ml/ (estructura src/), docker-compose.yml con
postgres:15 y redis:7, Makefile con los comandos de CLAUDE.md, .env.example,
.gitignore y un workflow de GitHub Actions que corra pytest. Nada de lógica de negocio todavía.
```

**Hecho**
- [ ] `make up` levanta db y redis
- [ ] `GET /health` responde 200
- [ ] `npm run dev` muestra una página en blanco con Tailwind funcionando
- [ ] El workflow de CI pasa en GitHub

---

## F1 — Base de datos (1 día)

**Prompt**
```
Implementá los modelos SQLAlchemy y la migración inicial con Alembic para estas
tablas: usuarios, habilidades, proyectos, miembros_equipo, sprints, tareas,
comentarios_tarea, predicciones_riesgo, alertas, chat_asistente, metricas_proyecto,
logs_acceso. Usá el esquema SQL que te paso abajo como fuente de verdad (checks,
índices, FKs). Agregá un script backend/app/seed.py que cree un admin, 2 PMs,
5 devs, 3 proyectos con sprints y ~60 tareas realistas.

[pegar aquí el script SQL de la tesis]
```

**Hecho**
- [ ] `alembic upgrade head` crea todas las tablas
- [ ] `python -m app.seed` carga datos sin errores
- [ ] Captura del DER generado (DBeaver) guardada en `docs/capturas/`

---

## F2 — Autenticación y roles (1 día) — RF-10

**Prompt**
```
Implementá auth con JWT: POST /auth/login (OAuth2 form), GET /auth/me, hash bcrypt,
dependencia require_role("admin"|"pm"|"dev"), bloqueo de cuenta 15 min tras 3
intentos fallidos y registro en logs_acceso. CRUD de usuarios solo para admin.
Tests para: login ok, login malo, bloqueo, acceso denegado por rol.
```

**Hecho**
- [ ] Tests de auth pasan (mínimo 6)
- [ ] Un `dev` no puede crear proyectos (403)

---

## F3 — API de gestión (2 días) — RF-01, RF-02, RF-03, RF-09

**Prompt**
```
Implementá routers y servicios para proyectos, sprints, tareas, miembros de equipo
y habilidades. Incluí:
- Validación fecha_inicio <= fecha_fin_estimada
- Al marcar tarea completada, setear fecha_fin_real
- GET /proyectos/{id}/metricas: avance %, tareas por estado, días restantes, SPI simple
- GET /equipo/{proyecto_id}/carga: carga % por miembro (horas activas / capacidad)
Tests para cada router (casos felices + validaciones).
```

**Hecho**
- [ ] CRUD completo funcionando en `/docs` (Swagger)
- [ ] `make test` verde, cobertura del backend ≥ 70%
- [ ] Métricas del proyecto devuelven valores coherentes con el seed

---

## F4 — Datos para ML (1–2 días)

Esta fase define cuán defendible es la tesis. Hacela con calma.

**Prompt**
```
En ml/src/generar_datos.py:
1) Descargá/usá un dataset público de esfuerzo de PROMISE (ej. China, Desharnais o
   ISBSG-like disponibles en PROMISE/Zenodo) y documentá cuál y de dónde.
2) Generá un dataset sintético de proyectos para riesgo (2000 filas) con features:
   tamanio_equipo, complejidad, dias_totales, dias_transcurridos, avance_real,
   avance_gap, ratio_esfuerzo, sprints_retrasados, tareas_criticas,
   requisitos_cambiantes, experiencia_promedio. La etiqueta de riesgo debe salir de
   reglas documentadas + ruido (no de una fórmula trivial que el modelo copie al 100%).
3) Generá un dataset de tareas para esfuerzo con descripciones de texto en español.
4) Escribí docs/resultados/datasets.md explicando origen, reglas y limitaciones.
5) Un notebook de EDA que guarde gráficos en docs/resultados/.
random_state=42 en todo.
```

**Hecho**
- [ ] `datasets.md` explica qué es real y qué es sintético
- [ ] EDA con distribuciones y balance de clases guardado como PNG
- [ ] Revisaste que la etiqueta de riesgo no sea 100% predecible (si el modelo da F1=1.0, algo está mal)

---

## F5 — Modelo de predicción de riesgos (2 días) — RF-04

**Prompt**
```
En ml/src/train_risk.py: split 80/20 estratificado, SMOTE solo en train, comparar
Regresión Logística (baseline), Random Forest, Gradient Boosting y XGBoost con
GridSearchCV (cv=5). Guardar en docs/resultados/riesgo.json: precision, recall, F1,
AUC-ROC, CV media±std, tiempo de entrenamiento por modelo. Guardar matriz de
confusión, curva ROC e importancia de features como PNG. Serializar el mejor
modelo + preprocesador con joblib.
Después: backend/app/services/riesgo_service.py que cargue el modelo, calcule las
features desde la BD para un proyecto, devuelva score, nivel (bajo <0.4, medio
0.4–0.7, alto >0.7) y top 3 factores; POST /proyectos/{id}/analizar-riesgo guarda
en predicciones_riesgo y crea alerta si es alto. Tests.
```

**Hecho**
- [ ] `riesgo.json` generado por código
- [ ] El endpoint devuelve score + factores para los 3 proyectos del seed
- [ ] Si no llegás a F1 ≥ 0.80, **no se tocan los datos para forzarlo**: se reporta y se discute en la tesis

---

## F6 — Modelo de estimación de esfuerzo (2 días) — RF-05

**Prompt**
```
En ml/src/train_effort.py: embeddings de la descripción con all-MiniLM-L6-v2 +
features (tipo, complejidad, prioridad). Comparar Regresión Lineal, Ridge, Random
Forest y XGBoost. Guardar en docs/resultados/esfuerzo.json: MAE en horas, MAE %,
RMSE, R², y MAE por tipo de tarea. Comparar también contra una heurística simple
(promedio por tipo×complejidad) para medir cuánto aporta el NLP.
Backend: POST /tareas/estimar-esfuerzo {descripcion, tipo, complejidad} → horas,
intervalo aproximado y 3 tareas similares del historial (similitud coseno).
El modelo de embeddings se carga una sola vez al arrancar. Tests.
```

**Hecho**
- [ ] `esfuerzo.json` con la comparación vs. heurística
- [ ] Estimación responde en < 1 s en local
- [ ] Devuelve tareas similares reales del seed

---

## F7 — Asistente PM (1–2 días) — RF-06

**Prompt**
```
Implementá el asistente como clasificador de intenciones (deadline, riesgo, avance,
recursos, tareas, desconocida) usando similitud de embeddings contra frases ejemplo
por intención (no solo palabras clave). Cada intención arma la respuesta con datos
reales de la BD (métricas, último riesgo, carga del equipo). Si la confianza es
baja, responder "no entendí" y sugerir preguntas. Guardar cada intercambio en
chat_asistente. POST /asistente/consultar.
Crear ml/eval/consultas_asistente.csv con 50 consultas etiquetadas a mano
(variadas, incluyendo ambiguas) y un script que calcule precisión por intención →
docs/resultados/asistente.json.
```

**Hecho**
- [ ] Precisión medida sobre las 50 consultas (no estimada)
- [ ] Las respuestas citan números reales del proyecto

### F7b — Opcional: RAG real (solo si vas adelantada)
Agregar un LLM chico (Ollama + Llama 3.2 3B o una API) que redacte la respuesta
usando como contexto los datos recuperados. Solo con esto se puede llamar "RAG" en la tesis.

---

## F8 — Frontend (4 días) — RF-07 y UI de todo lo anterior

Pantallas, en este orden. Una por sesión de Claude Code.

| # | Pantalla | Contenido mínimo |
|---|---|---|
| 1 | Login | email, contraseña, errores claros |
| 2 | Layout | sidebar con 5 ítems, usuario abajo, cerrar sesión |
| 3 | Dashboard | 4 KPIs, barras de score de riesgo, tabla de proyectos |
| 4 | Proyectos + detalle | listado con filtro, detalle con panel "Análisis IA" y burndown |
| 5 | Tareas | Kanban 5 columnas, modal con botón "Estimar con IA" |
| 6 | Equipo | tarjetas con barra de carga |
| 7 | Asistente | chat + 5 preguntas sugeridas |

**Prompt (repetir por pantalla)**
```
Implementá la pantalla [N] según docs/PLAN.md F8, consumiendo los endpoints ya
existentes. Mantenela sobria: si algo no aporta a la decisión del PM, no lo pongas.
Estados de carga, vacío y error en cada vista. Respetá el rol (dev no ve botones de PM).
```

**Hecho**
- [ ] Flujo completo demostrable: login → dashboard → analizar riesgo → crear tarea con estimación → preguntar al asistente
- [ ] Sin datos hardcodeados (ej. el `avance = 45` del borrador viejo)
- [ ] Capturas de cada pantalla en `docs/capturas/`

---

## F9 — Deploy (1 día)

**Prompt**
```
Prepará Dockerfiles de producción para backend y frontend, variables de entorno
para Railway, script de seed para producción con datos demo y un healthcheck.
Documentá el deploy en docs/deploy.md.
```

**Hecho**
- [ ] URL pública funcionando
- [ ] Monitor de uptime configurado (UptimeRobot gratis) desde el día del deploy — de ahí sale la disponibilidad real

---

## F10 — Validación y resultados reales (5 días)

**Prompt**
```
1) Convertí la tabla de casos de prueba funcionales en tests automatizados donde se
   pueda; el resto, un checklist manual en docs/resultados/pruebas_manuales.md.
2) Prueba de carga con Locust: 1 y 10 usuarios concurrentes sobre los 6 endpoints
   principales → docs/resultados/rendimiento.json (p50 y p95).
3) Script make results que regenere todos los JSON y PNG de docs/resultados/.
4) Un docs/resultados/RESUMEN.md con la tabla final de criterios vs. obtenidos,
   leyendo los JSON (no a mano).
```

**Fuera de Claude Code (lo hacés vos)**
- [ ] Sesiones SUS con 5+ PMs reales (30 min c/u). Cargá las respuestas en `docs/resultados/sus.csv` y que el script calcule el puntaje
- [ ] Si podés, conseguí 1 empresa que te pase datos anonimizados de un proyecto real para probar el modelo de riesgo. Aunque sea 1, suma muchísimo en la defensa

**Hecho**
- [ ] `RESUMEN.md` generado automáticamente
- [ ] Cada número de la tesis tiene un archivo en `docs/resultados/` que lo respalda

---

## F11 — Cierre de tesis (semana final)

- [ ] Reemplazar en Cap. 4 y 5, Resumen, Abstract y diapositivas los números de referencia por los de `RESUMEN.md`
- [ ] Ajustar la descripción del asistente (clasificación de intenciones, o RAG si hiciste F7b)
- [ ] Insertar capturas reales y diagramas exportados
- [ ] Ensayo de defensa con las 3 preguntas difíciles: datos sintéticos, n del SUS, cómo funciona el asistente

---

## Seguimiento

| Fase | Inicio | Fin | Estado | Nota |
|---|---|---|---|---|
| F0 Setup | | | ⬜ | |
| F1 Base de datos | | | ⬜ | |
| F2 Auth | | | ⬜ | |
| F3 API gestión | | | ⬜ | |
| F4 Datos ML | | | ⬜ | |
| F5 Riesgo | | | ⬜ | |
| F6 Esfuerzo | | | ⬜ | |
| F7 Asistente | | | ⬜ | |
| F8 Frontend | | | ⬜ | |
| F9 Deploy | | | ⬜ | |
| F10 Validación | | | ⬜ | |
| F11 Tesis | | | ⬜ | |

## Tips para Claude Code
- Empezá cada sesión con: *"Leé CLAUDE.md y docs/PLAN.md. Estamos en la fase X."*
- Usá `/clear` entre fases para que no arrastre contexto viejo.
- Pedí siempre el plan antes del código en fases grandes (F3, F5, F8).
- Si algo falla dos veces seguidas, pedile que explique la causa antes de intentar otra vez.
