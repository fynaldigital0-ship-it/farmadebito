[README.md](https://github.com/user-attachments/files/29677716/README.md)
# FarmaCero — Conciliador Financiero Inteligente

SaaS B2B para farmacias independientes y pequeñas cadenas regionales de
Argentina. Cruza automáticamente lo que la farmacia vendió con descuento de
Obra Social/Prepaga (export de SIFACO/SIAF) contra lo que la auditora
realmente liquidó, detecta débitos y diferencias de precio, y prioriza qué
reclamar antes de que venza el plazo.

> Este repo es el MVP completo: backend, motor de matching, frontend y
> configuración de despliegue. Ver [Alcance de este MVP](#alcance-de-este-mvp-y-próximos-pasos)
> para lo que queda como roadmap (scraping real de portales, PDFs).

## Stack

| Capa | Tecnología |
|---|---|
| Backend / API | Python 3.11 + FastAPI |
| Motor de matching | Pandas + `difflib` (cascada exacto → difuso → manual) |
| Base de datos | PostgreSQL (SQLite automático en desarrollo local sin `DATABASE_URL`) |
| Frontend | React 18 + Vite + Tailwind CSS + Recharts |
| Auth | JWT (python-jose) + bcrypt |
| Infra | Docker (backend) + Render (blueprint incluido) |

La arquitectura, el modelo de datos y las reglas de matching siguen al pie
de la letra las decisiones de producto documentadas en
`docs/especificacion-producto/references/` (data-model.md, matching-engine.md, ux-flow.md, architecture.md).

## Estructura del repo

```
farmacero/
├── backend/            # API FastAPI
│   ├── app/
│   │   ├── models.py        # Esquema SQLAlchemy (fiel a data-model.md)
│   │   ├── matching/engine.py  # Cascada exacto → difuso → manual
│   │   ├── parsers/          # Parsers de ventas ERP y liquidaciones OS
│   │   ├── routers/          # auth, dashboard, conciliacion, ingesta, portales
│   │   └── seed.py           # Datos de demo (farmacia + ventas + liquidaciones)
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/           # SPA React + Vite + Tailwind
│   └── src/
│       ├── pages/            # Login, Registro, Dashboard, Ingesta
│       └── components/       # KpiCard, Charts, DetalleModal, Layout
├── render.yaml         # Blueprint de despliegue (API + Postgres + estático)
└── README.md
```

## Desarrollo local

### 1. Backend

```bash
cd backend
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env            # opcional: sin esto, usa SQLite local automáticamente
uvicorn app.main:app --reload --port 8000
```

Al arrancar, si `SEED_DEMO_DATA=true` (default), se crea automáticamente:

- Farmacia demo con ventas y liquidaciones de ejemplo de los últimos 5 meses.
- Usuario: **`demo@farmacero.ar`** / **`demo1234`**

La API queda en `http://localhost:8000`, con Swagger interactivo en
`http://localhost:8000/docs`.

### 2. Frontend

```bash
cd frontend
npm install
cp .env.example .env            # apunta a http://localhost:8000 por defecto
npm run dev
```

Abrí `http://localhost:5173` e iniciá sesión con las credenciales de demo.

### Alternativa: todo con Docker Compose (backend + Postgres real)

```bash
docker compose up --build
```

Esto levanta Postgres y el backend en `http://localhost:8000` con datos de
demo ya cargados. El frontend seguís corriéndolo con `npm run dev` como en
el paso 2 (apuntando `VITE_API_URL` a ese mismo puerto).

## Subir a GitHub

```bash
cd farmacero
git init
git add .
git commit -m "FarmaCero MVP: backend, motor de matching y dashboard"
git branch -M main
git remote add origin https://github.com/TU-USUARIO/farmacero.git
git push -u origin main
```

(Reemplazá `TU-USUARIO` por tu usuario u organización de GitHub, y creá el
repo vacío en GitHub antes del `push` si todavía no existe.)

## Desplegar en Render

### Opción A — Blueprint (recomendada, un solo clic)

Este repo incluye `render.yaml`, que define de una vez: la base Postgres,
el servicio del backend (Docker) y el sitio estático del frontend.

1. En Render → **New** → **Blueprint**.
2. Conectá el repo de GitHub que acabás de crear.
3. Render va a detectar `render.yaml` y proponer los 3 recursos
   (`farmacero-db`, `farmacero-api`, `farmacero-web`). Confirmá el deploy.
4. Esperá a que terminen los dos builds (unos minutos la primera vez).
5. Entrá a la URL de `farmacero-web` (algo como
   `https://farmacero-web.onrender.com`) — ya vas a ver el dashboard con
   datos de demo cargados.

**Importante sobre nombres de servicio:** los subdominios `*.onrender.com`
son globales. Si `farmacero-api` o `farmacero-web` ya están tomados por
otra cuenta, Render les va a asignar un nombre distinto. Si eso pasa:

- Actualizá `VITE_API_URL` en el servicio `farmacero-web` con la URL real
  que Render le asignó a tu backend, y volvé a desplegar el frontend.
- (Opcional, más seguro) Cambiá `CORS_ORIGINS` en `farmacero-api` de `"*"`
  a la URL real de tu frontend.

### Opción B — Servicios manuales

Si preferís no usar el blueprint:

1. **Postgres**: New → PostgreSQL. Copiá el "Internal Connection String".
2. **Backend**: New → Web Service → conectá el repo → Root Directory
   `backend` → Runtime `Docker`. Variables de entorno:
   - `DATABASE_URL`: la connection string de arriba
   - `JWT_SECRET`: cualquier string largo y random
   - `CORS_ORIGINS`: `*` (o la URL de tu frontend una vez que la tengas)
   - `SEED_DEMO_DATA`: `true`
3. **Frontend**: New → Static Site → conectá el repo → Root Directory
   `frontend` → Build Command `npm install && npm run build` → Publish
   Directory `dist`. Variable de entorno `VITE_API_URL` con la URL pública
   del backend. Agregá una regla de rewrite `/* → /index.html` (necesaria
   para que las rutas de React Router no den 404 al refrescar).

### Nota sobre el plan free de Render

Los servicios free "duermen" tras ~15 min sin tráfico (la primera request
después de eso tarda ~30-50s en responder mientras arranca), y la base
Postgres free expira a los 30 días si no la actualizás a un plan pago. Para
un demo esto es aceptable; para producción real, pasar a planes pagos.

## Motor de matching

Implementa la cascada descrita en `docs/especificacion-producto/references/matching-engine.md`:

1. **Exacto**: llave compuesta `(troquel_medicamento, id_receta normalizado, monto redondeado)` dentro de la misma Obra Social.
2. **Difuso**: scoring ponderado (troquel 35%, similitud de receta 30%, tolerancia de monto 20%, afiliado 15%). Score ≥ 0.75 se autoacepta; entre 0.55 y 0.75 queda en cola de revisión manual con el candidato sugerido, visible en el modal de detalle del dashboard.
3. **Manual**: el farmacéutico confirma o descarta el candidato sugerido con un clic.

Los cuatro estados de `detalle_conciliacion` son exactamente los definidos
en el modelo de datos: `Pagado Completo`, `Debitada con Error`,
`Diferencia de Precio`, `Pendiente de Cobro`.

## Alcance de este MVP y próximos pasos

Fiel a las restricciones de contexto del producto (sin API oficial de
SIFACO/SIAF ni de las auditoras), este repo implementa:

- ✅ Ingesta de ventas y liquidaciones por **carga manual de CSV/Excel**
  (el flujo que ya funciona hoy, sin depender de scraping).
- ✅ Motor de matching completo (exacto → difuso → manual) corriendo sobre
  cualquier dato cargado, real o de demo.
- ✅ Dashboard de Salud Financiera con KPIs, serie temporal, breakdown por
  Obra Social y tabla de acción con generación de reclamos.
- ✅ Guardado cifrado de credenciales delegadas de portales (`/api/portales`),
  como base para conectar scraping real más adelante.

Quedan como roadmap explícito (ver `docs/especificacion-producto/references/architecture.md`):

- 🔜 **Workers de scraping reales** (Playwright) contra IMED/Farmalink/
  Preserfar/Colegios, usando las credenciales ya guardadas. Requiere
  automatizar el login y layout específico de cada portal real, algo que
  no se puede construir sin acceso a esos portales.
- 🔜 **Parsing de liquidaciones en PDF** con Camelot (hoy el fallback
  manual soporta CSV/Excel, que es el formato más común de exportación).
- 🔜 Migración a AWS (RDS + Lambda para scrapers) para escalar más allá de
  un puñado de farmacias, y `recordlinkage`/`rapidfuzz` en el motor de
  matching para volúmenes grandes.

## Licencia

Uso interno / propietario — ajustar según corresponda antes de hacer el
repositorio público.
