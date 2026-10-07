# SmartPM — Contexto del proyecto

Sistema web de gestión de proyectos de software con módulos de Machine Learning.
Proyecto de tesis de Ingeniería en Sistemas (UNIDA, Asunción). Defensa: 23/11/2026.

## Stack
- Backend: Python 3.11, FastAPI, SQLAlchemy 2.0, Alembic, PostgreSQL 15, Redis
- ML: scikit-learn, XGBoost, imbalanced-learn (SMOTE), sentence-transformers (all-MiniLM-L6-v2), joblib
- Frontend: React 18 + Vite, TailwindCSS, Recharts, lucide-react, react-hot-toast, axios, react-router-dom
- Infra: Docker Compose, GitHub Actions, deploy en Railway

## Estructura
```
smartpm/
├── backend/        # FastAPI: app/{models,schemas,routers,services,ml}, tests/
├── ml/             # src/{generar_datos,preprocess,train_risk,train_effort,evaluate}.py, notebooks/, models/
├── frontend/       # React + Vite: src/{pages,components,api,context}
├── AGENTS.md       # quién hace qué + reglas SOLID
├── .claude/        # agents/{orquestador,backend,ml,frontend,qa}.md, commands/fase.md
├── docs/
│   ├── PLAN.md
│   ├── TABLERO.md  # en qué trabaja cada agente (lo escribe el orquestador)
│   ├── resultados/ # JSON + PNG generados por código (métricas, matrices, gráficos)
│   └── capturas/   # screenshots para la tesis
├── docker-compose.yml
└── Makefile
```

## Reglas para trabajar en este repo
- Una fase de docs/PLAN.md por vez. No avanzar sin cumplir sus criterios de "Hecho".
- Antes de escribir código de una fase, mostrar un plan corto y esperar OK.
- Todo endpoint nuevo lleva su test en `backend/tests/` (pytest). Correr `make test` antes de cerrar una tarea.
- Las métricas (ML, rendimiento, pruebas) se calculan con código y se guardan en
  `docs/resultados/*.json`. NUNCA escribir un número de resultados a mano ni copiarlo de la tesis.
- Fijar `random_state=42` en todo lo aleatorio para que los resultados sean reproducibles.
- Roles: `admin`, `pm`, `dev`. Toda ruta protegida valida JWT y rol.
- Nombres de tablas y campos en español, snake_case.
- UI sobria: pocas cosas por pantalla, sin acumular widgets. Ante la duda, sacar.
- Commits chicos y descriptivos en español.

## Agentes
La sesión principal actúa como **orquestador**: planifica la fase, delega a los agentes
`backend`, `ml`, `frontend` y `qa`, y mantiene `docs/TABLERO.md` (ahí se ve en qué trabaja
cada uno). Se arranca con `/fase F0`. Roles, fronteras y reglas de diseño SOLID:
@AGENTS.md

## Honestidad técnica (importante para la defensa)
- El asistente PM es clasificación de intenciones + consultas a la BD. No llamarlo RAG
  salvo que se implemente la Fase 7b (embeddings + recuperación + LLM).
- Registrar en `docs/resultados/datasets.md` el origen de cada dataset (PROMISE, sintético, real) y cómo se generó.

## Comandos
- `make up`            → docker compose up -d db redis
- `make api`           → uvicorn app.main:app --reload (desde backend/)
- `make web`           → npm run dev (desde frontend/)
- `make train`         → entrena riesgo + esfuerzo y copia los .joblib a backend/app/ml/models/
- `make test`          → pytest backend + ml
- `make results`       → regenera todo docs/resultados/
