# CLAUDE.md — IntelPY

> Este archivo contiene las instrucciones para que Claude (u otro agente AI) entienda el proyecto, su arquitectura, convenciones y cómo colaborar de forma efectiva en el desarrollo de IntelPY.

---

## Contexto del proyecto

**IntelPY** es una plataforma SaaS de inteligencia digital y análisis de vulnerabilidades construida para el mercado paraguayo. Tiene tres líneas de producto:

1. **IntelPY Business** — OSINT B2B: investigación de personas, empresas, cyber intelligence, case management
2. **IntelPY Score** — Crédito personal B2C: el primer servicio público de consulta de historial crediticio en Paraguay
3. **IntelPY VulnScan** — Análisis automático de vulnerabilidades externas por dominio/IP

**Fundador y desarrollador principal:** Juan — Ciudad del Este, Paraguay  
**Stack principal:** Python 3.12 (backend), Next.js 14 (frontend)  
**Moneda:** Guaraníes (Gs.) — tipo de cambio referencial: 1 USD ≈ Gs. 7,500

---

## Reglas generales de colaboración

### Idioma
- **Código:** inglés (variables, funciones, comentarios inline)
- **Comunicación con el desarrollador:** español
- **Strings del usuario final (UI):** español paraguayo (formal pero accesible)
- **Documentación técnica (docstrings):** inglés

### Estilo de respuesta
- Ser directo y práctico — nada de explicaciones innecesariamente largas
- Siempre dar código funcional, no pseudocódigo
- Si hay varias formas de hacer algo, recomendar la más simple primero
- Preferir ejemplos concretos sobre explicaciones abstractas

---

## Arquitectura del sistema

### Servicios Docker (docker-compose.yml)

```
intelpy_db       → PostgreSQL 16 + pgvector  (puerto 5432)
intelpy_redis    → Redis 7                   (puerto 6379)
intelpy_api      → FastAPI / Uvicorn         (puerto 8000, interno)
intelpy_worker   → Celery worker (4 colas)
intelpy_beat     → Celery Beat + RedBeat
intelpy_flower   → Monitor Celery            (puerto 5555, solo dev)
intelpy_frontend → Next.js                   (puerto 3000, interno)
intelpy_nginx    → Nginx reverse proxy       (puertos 80/443)
intelpy_certbot  → Let's Encrypt             (solo perfil prod)
```

### Colas Celery (4 separadas por prioridad)

| Cola | Propósito | Workers recomendados |
|------|-----------|----------------------|
| `cyber` | VulnScan — Nuclei, Nmap, Shodan, SSLyze | 2 workers dedicados |
| `osint` | Búsqueda de personas/empresas, Playwright | 2 workers |
| `credit` | Historial crediticio, score B2C | 1 worker |
| `default` | Emails, Telegram, PDF generation, alertas | 1 worker |

### Flujo de un escaneo VulnScan

```
POST /api/v1/scans
  → FastAPI valida input y permisos
  → Crea ScanJob en PostgreSQL (status: queued)
  → Dispara task group Celery (8 módulos en paralelo)
  → Responde con { task_id, status: "queued" }

Celery workers ejecutan en paralelo:
  → recon_task()      → subdominios, tecnologías
  → ports_task()      → Nmap + Shodan
  → web_task()        → Nuclei OWASP
  → email_task()      → SPF/DKIM/DMARC
  → ssl_task()        → SSLyze
  → leaks_task()      → HIBP + Hunter.io
  → cloud_task()      → TruffleHog + gitleaks
  → scorer_task()     → Risk Score 0-100

Cuando todos terminan (chord callback):
  → Consolida findings en PostgreSQL
  → Genera PDF con WeasyPrint
  → Notifica al cliente por Telegram/email
  → Actualiza ScanJob.status = "completed"
```

---

## Estructura de archivos — backend Python

```
backend/app/
├── main.py                 # FastAPI app, routers, middleware
├── config.py               # Settings via pydantic-settings (lee .env)
├── database.py             # AsyncEngine, SessionLocal, Base
│
├── api/
│   ├── deps.py             # get_db(), get_current_user(), get_org()
│   └── v1/
│       ├── auth.py         # POST /token, POST /refresh, POST /register
│       ├── scans.py        # VulnScan CRUD + WebSocket progreso
│       ├── osint.py        # Personas, Empresas, Investigaciones
│       ├── credit.py       # Score personal B2C
│       └── reports.py      # Descarga PDF, historial
│
├── models/                 # SQLAlchemy ORM models
│   ├── user.py             # User, Organization, Subscription
│   ├── scan.py             # ScanTarget, ScanJob, Finding, Report
│   ├── osint.py            # Person, Company, Investigation, Evidence
│   └── credit.py          # CreditReport, CreditScore, Alert
│
├── schemas/                # Pydantic schemas (request/response)
│   ├── scan.py
│   ├── osint.py
│   └── user.py
│
├── services/
│   ├── vulnscan/           # Módulos de escaneo
│   │   ├── recon.py        # subfinder + amass wrapper
│   │   ├── ports.py        # nmap + shodan API
│   │   ├── web.py          # nuclei subprocess
│   │   ├── email_sec.py    # DNS/SPF/DKIM/DMARC checks
│   │   ├── ssl.py          # sslyze wrapper
│   │   ├── leaks.py        # HIBP API + Hunter.io API
│   │   ├── cloud.py        # trufflehog subprocess
│   │   └── scorer.py       # Algoritmo de score ponderado
│   ├── osint/
│   │   ├── persons.py
│   │   ├── companies.py
│   │   └── scraper.py      # Playwright → SET, DNIT, Poder Judicial
│   ├── credit/
│   │   └── score.py
│   └── reports/
│       ├── pdf.py          # WeasyPrint orchestrator
│       └── templates/      # Jinja2 HTML templates para PDF
│
└── worker/
    ├── celery_app.py       # Celery config, 4 colas, RedBeat
    ├── tasks_vulnscan.py   # Tasks de escaneo (chord pattern)
    ├── tasks_osint.py      # Tasks de búsqueda OSINT
    └── tasks_alerts.py     # Beat tasks periódicas
```

---

## Convenciones de código — Python

### Estructura de un servicio de escaneo

```python
# services/vulnscan/recon.py

import asyncio
import subprocess
import json
from typing import Any
from app.models.scan import Finding, SeverityLevel


async def run_subfinder(domain: str) -> list[str]:
    """
    Run subfinder to enumerate subdomains passively.
    Returns list of discovered subdomains.
    """
    proc = await asyncio.create_subprocess_exec(
        "subfinder", "-d", domain, "-silent", "-json",
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.DEVNULL,
    )
    stdout, _ = await proc.communicate()
    
    subdomains = []
    for line in stdout.decode().splitlines():
        try:
            data = json.loads(line)
            subdomains.append(data.get("host", ""))
        except json.JSONDecodeError:
            continue
    return [s for s in subdomains if s]


def build_findings(subdomains: list[str], domain: str) -> list[dict[str, Any]]:
    """Convert raw recon data into normalized Finding dicts."""
    findings = []
    for sub in subdomains:
        if sub != domain:
            findings.append({
                "module": "recon",
                "title": f"Subdominio descubierto: {sub}",
                "severity": SeverityLevel.INFO,
                "description": f"El subdominio {sub} está activo y expuesto públicamente.",
                "recommendation": "Verificar que este subdominio sea intencional y esté correctamente configurado.",
                "evidence": {"subdomain": sub, "parent_domain": domain},
            })
    return findings
```

### Estructura de un Celery task

```python
# worker/tasks_vulnscan.py

from celery import group, chord
from app.worker.celery_app import celery_app
from app.services.vulnscan import recon, ports, web, email_sec, ssl, leaks, cloud, scorer


@celery_app.task(queue="cyber", bind=True, max_retries=2)
def recon_task(self, scan_job_id: str, target: str) -> dict:
    """Reconnaissance module — subdomain enumeration."""
    try:
        import asyncio
        subdomains = asyncio.run(recon.run_subfinder(target))
        findings = recon.build_findings(subdomains, target)
        return {"module": "recon", "findings": findings, "status": "ok"}
    except Exception as exc:
        raise self.retry(exc=exc, countdown=30)


@celery_app.task(queue="default")
def consolidate_results(results: list[dict], scan_job_id: str) -> None:
    """Chord callback — called after all modules finish."""
    from app.database import SyncSession
    from app.models.scan import ScanJob, Finding, ScanStatus
    
    with SyncSession() as db:
        all_findings = []
        for result in results:
            if result and result.get("status") == "ok":
                all_findings.extend(result.get("findings", []))
        
        score = scorer.calculate_score(all_findings)
        
        # Guardar findings y actualizar job
        job = db.get(ScanJob, scan_job_id)
        job.score = score
        job.status = ScanStatus.COMPLETED
        for f in all_findings:
            db.add(Finding(scan_job_id=scan_job_id, **f))
        db.commit()
    
    # Generar PDF y notificar
    generate_report_task.delay(scan_job_id)
```

### FastAPI endpoint pattern

```python
# api/v1/scans.py

from fastapi import APIRouter, Depends, HTTPException, BackgroundTasks
from sqlalchemy.ext.asyncio import AsyncSession
from app.api.deps import get_db, get_current_user, require_plan
from app.schemas.scan import ScanCreate, ScanResponse, ScanStatus
from app.models.user import User
from app.worker.tasks_vulnscan import launch_vulnscan_chord

router = APIRouter(prefix="/scans", tags=["vulnscan"])


@router.post("/", response_model=ScanResponse, status_code=202)
async def create_scan(
    payload: ScanCreate,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
    _: None = Depends(require_plan(["pro", "enterprise"])),
):
    """
    Start a new VulnScan for a target domain or IP.
    Returns task_id for polling.
    """
    # Crear ScanJob en DB
    job = ScanJob(
        org_id=current_user.org_id,
        created_by=current_user.id,
        target=payload.target,
        modules=payload.modules,
        status=ScanStatus.QUEUED,
    )
    db.add(job)
    await db.commit()
    await db.refresh(job)
    
    # Lanzar chord Celery (todos los módulos en paralelo)
    launch_vulnscan_chord(str(job.id), payload.target, payload.modules)
    
    return ScanResponse.from_orm(job)
```

---

## Convenciones de código — Frontend (Next.js)

```typescript
// Siempre TypeScript strict
// Componentes en PascalCase, archivos en kebab-case
// Server Components por defecto, "use client" solo cuando necesario

// Fetch de API siempre tipado
async function getScan(id: string): Promise<ScanJob> {
  const res = await fetch(`/api/v1/scans/${id}`, {
    headers: { Authorization: `Bearer ${getToken()}` },
    next: { revalidate: 0 }, // no cachear scans
  });
  if (!res.ok) throw new Error("Scan not found");
  return res.json();
}
```

---

## Variables de entorno (.env)

El archivo `.env` nunca se commitea. Usar `.env.example` como referencia.

Variables críticas que deben estar siempre definidas:

```bash
# Core
SECRET_KEY          # mínimo 32 chars aleatorios
DATABASE_URL        # postgresql+asyncpg://...
REDIS_URL           # redis://:password@redis:6379/0

# APIs externas (para VulnScan)
SHODAN_API_KEY      # plan Freelancer mínimo
HIBP_API_KEY        # Have I Been Pwned API
HUNTER_API_KEY      # Hunter.io para emails

# Pagos (Paraguay)
BANCARD_PUBLIC_KEY
BANCARD_PRIVATE_KEY

# Notificaciones
TELEGRAM_BOT_TOKEN
```

---

## Ejecución en Kali Linux

### Setup inicial en Kali (una sola vez)

```bash
# 1. Clonar repo
git clone https://github.com/tuuser/intelpy.git && cd intelpy

# 2. Instalar herramientas OSINT del sistema
# Kali ya trae: nmap, whois, dnsrecon, dirb
# Instalar las que faltan:
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/owasp-amass/amass/v4/...@latest

# Verificar Go PATH
export PATH=$PATH:$(go env GOPATH)/bin
echo 'export PATH=$PATH:$(go env GOPATH)/bin' >> ~/.bashrc

# 3. Variables de entorno
cp .env.example .env && nano .env

# 4. Levantar
make dev
make migrate
```

### Workflow de desarrollo diario en Kali

```bash
# Arrancar la plataforma
make dev

# Trabajar en el backend sin Docker (más rápido para iterar)
cd backend
source .venv/bin/activate
uvicorn app.main:app --reload --port 8000

# Worker Celery en otra terminal
celery -A app.worker.celery_app worker \
  --loglevel=info \
  --concurrency=2 \
  -Q cyber,osint,credit,default

# Ver logs de todos los servicios
make logs

# Tests
make test
# o directamente:
pytest tests/ -v -x --tb=short
```

### Test manual de un escaneo desde Kali

```bash
# Con httpie (más legible que curl)
pip install httpie

# Login
http POST localhost:8000/api/v1/auth/token \
  username=admin@intelpy.com password=admin123

# Iniciar escaneo (reemplazar TOKEN)
http POST localhost:8000/api/v1/scans \
  Authorization:"Bearer TOKEN" \
  target=ejemplo.com.py \
  modules:='["recon","ports","email","ssl"]'

# Polling de estado
http GET localhost:8000/api/v1/scans/TASK_ID \
  Authorization:"Bearer TOKEN"
```

---

## Base de datos — modelos clave

### ScanJob (tabla: scan_jobs)

```python
class ScanJob(Base):
    __tablename__ = "scan_jobs"
    
    id: UUID                    # PK
    org_id: UUID                # FK organizations — RLS key
    created_by: UUID            # FK users
    target: str                 # dominio o IP objetivo
    modules: list[str]          # módulos seleccionados
    status: ScanStatus          # queued|running|completed|failed
    score: int | None           # 0-100, None si no completó
    findings_count: int         # cache de cantidad
    started_at: datetime | None
    completed_at: datetime | None
    created_at: datetime        # auto
```

### Finding (tabla: findings)

```python
class Finding(Base):
    __tablename__ = "findings"
    
    id: UUID
    scan_job_id: UUID           # FK scan_jobs
    org_id: UUID                # RLS key
    module: str                 # recon|ports|web|email|ssl|leaks|cloud
    title: str
    severity: SeverityLevel     # critical|high|medium|low|info
    description: str
    recommendation: str
    evidence: dict              # JSONB — datos crudos del hallazgo
    cvss_score: float | None
    cve_id: str | None
    is_resolved: bool           # para tracking de remediación
    created_at: datetime
```

---

## Algoritmo de Score (0-100)

```python
# services/vulnscan/scorer.py

SEVERITY_WEIGHTS = {
    "critical": 25,
    "high":     10,
    "medium":    4,
    "low":       1,
    "info":      0,
}

MAX_SCORE = 100

def calculate_score(findings: list[dict]) -> int:
    """
    Score = 100 - penalizaciones acumuladas.
    100 = sin vulnerabilidades / 0 = comprometido.
    Se clampa entre 0 y 100.
    """
    penalty = sum(
        SEVERITY_WEIGHTS.get(f.get("severity", "info"), 0)
        for f in findings
    )
    return max(0, MAX_SCORE - penalty)
```

---

## Tareas pendientes (backlog ordenado por prioridad)

### 🔴 Prioridad 1 — MVP VulnScan
- [ ] `services/vulnscan/recon.py` — subfinder async wrapper
- [ ] `services/vulnscan/ports.py` — nmap + shodan integration
- [ ] `services/vulnscan/email_sec.py` — SPF/DKIM/DMARC checker
- [ ] `services/vulnscan/ssl.py` — sslyze async wrapper
- [ ] `worker/tasks_vulnscan.py` — chord pattern completo
- [ ] `api/v1/scans.py` — endpoints CRUD + WebSocket
- [ ] Modelos SQLAlchemy + migración inicial
- [ ] Auth JWT básico (login, refresh, register)

### 🟡 Prioridad 2 — Producto completo
- [ ] `services/vulnscan/web.py` — nuclei subprocess
- [ ] `services/vulnscan/leaks.py` — HIBP + Hunter
- [ ] `services/reports/pdf.py` — WeasyPrint + Jinja2 template
- [ ] Billing con Bancard SDK
- [ ] Telegram Bot alerts
- [ ] Frontend dashboard — scans list + detail

### 🟢 Prioridad 3 — Escalado
- [ ] IntelPY Score (B2C crédito personal)
- [ ] API pública v1 + documentación
- [ ] White-label para partners
- [ ] App móvil PWA
- [ ] Integración Playwright con fuentes PY (SET, DNIT)

---

## Notas importantes

### Sobre el escaneo legal y ético
Todos los escaneos VulnScan **requieren autorización escrita previa** del propietario del dominio. El sistema debe:
1. Verificar que existe un `AuthorizationContract` firmado antes de ejecutar el escaneo
2. Registrar en DB quién autorizó, cuándo y qué alcance tiene el escaneo
3. Nuclei debe correr en modo `passive` o con templates no intrusivos salvo autorización explícita

### Sobre los datos paraguayos
- Fuentes públicas permitidas: SET (RUC), DNIT (cédulas), Poder Judicial (expedientes), CONEC, DNCP
- Playwright debe rotar User-Agents y respetar robots.txt
- Cachear resultados de Playwright en Redis mínimo 6h para no sobrecargar sitios del Estado

### Sobre multi-tenancy
- Siempre incluir `org_id` en **todas** las queries
- El middleware de FastAPI debe inyectar `org_id` desde el JWT automáticamente
- Nunca exponer datos de un org_id desde un endpoint de otro tenant

---

*Última actualización: 2025 · IntelPY — Ciudad del Este, Paraguay*
