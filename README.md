<div align="center">

```
██╗███╗   ██╗████████╗███████╗██╗     ██████╗ ██╗   ██╗
██║████╗  ██║╚══██╔══╝██╔════╝██║     ██╔══██╗╚██╗ ██╔╝
██║██╔██╗ ██║   ██║   █████╗  ██║     ██████╔╝ ╚████╔╝ 
██║██║╚██╗██║   ██║   ██╔══╝  ██║     ██╔═══╝   ╚██╔╝  
██║██║ ╚████║   ██║   ███████╗███████╗██║        ██║   
╚═╝╚═╝  ╚═══╝   ╚═╝   ╚══════╝╚══════╝╚═╝        ╚═╝   
```

**Plataforma de Inteligencia Digital & Análisis de Vulnerabilidades**  
*Made in Paraguay 🇵🇾 · Python-first · Production-ready*

[![Python](https://img.shields.io/badge/Python-3.12+-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?style=flat-square&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![Next.js](https://img.shields.io/badge/Next.js-14-000000?style=flat-square&logo=next.js&logoColor=white)](https://nextjs.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?style=flat-square&logo=postgresql&logoColor=white)](https://postgresql.org)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square&logo=docker&logoColor=white)](https://docker.com)
[![License](https://img.shields.io/badge/License-Proprietary-red?style=flat-square)](LICENSE)

</div>

---
Visita nuestra web (https://juanesscobar.github.io/Intelpy/).

## ¿Qué es IntelPY?

**IntelPY** es una plataforma de inteligencia digital diseñada para el mercado paraguayo, con dos productos core:

| Producto | Descripción | Mercado |
|----------|-------------|---------|
| **IntelPY Business** | OSINT corporativo: personas, empresas, investigaciones, cyber intelligence | B2B — estudios jurídicos, financieras, RRHH |
| **IntelPY Score** | Historial crediticio personal — el primero accesible al público en PY | B2C — cualquier persona con cédula |
| **IntelPY VulnScan** | Análisis automático de vulnerabilidades externas por dominio/IP | B2B medio-alto — bancos, clínicas, industria |

> **Problema que resuelve:** Inforconf y BICSA solo atienden empresas. Las personas no pueden ver su propio perfil crediticio. Las empresas medianas no tienen acceso a análisis de seguridad automatizados. IntelPY llena ese vacío.

---

## Módulos del sistema

### 🔍 OSINT Intelligence
- **Personas** — búsqueda, perfilado, historial, alertas de cambio
- **Empresas** — RUC, dominios, estructura societaria, exposición digital  
- **Investigaciones** — case management con evidencias y línea de tiempo
- **Historial de Crédito** — score, recomendaciones, monitoreo mensual

### 🛡️ VulnScan — Análisis automático de vulnerabilidades

```
[Input: dominio/IP] → [8 workers Celery en paralelo] → [Score 0-100] → [Reporte PDF]
```

| Módulo | Herramienta | Qué detecta |
|--------|-------------|-------------|
| Reconocimiento | Amass + subfinder | Subdominios, tecnologías, superficie de ataque |
| Puertos expuestos | Nmap + Shodan API | RDP, SSH, MySQL, paneles admin abiertos |
| Vulnerabilidades web | Nuclei (9000+ templates) | OWASP Top 10, CVEs, misconfigs |
| Email security | dnspython + MXToolbox | SPF/DKIM/DMARC ausente o mal configurado |
| Credenciales filtradas | HIBP + Hunter.io | Emails corporativos en data breaches |
| SSL/TLS | SSLyze | Certificados vencidos, algoritmos débiles |
| Cloud & repos | TruffleHog + gitleaks | API keys, secrets en repos públicos |
| Score consolidado | Algoritmo propio | Risk Score 0-100 por severidad ponderada |

---

## Stack tecnológico

```
┌─────────────────────────────────────────────────────────────┐
│  Cliente (Browser / API consumer)                           │
├─────────────────────────────────────────────────────────────┤
│  Nginx (Reverse Proxy + SSL + Rate Limiting)                │
├──────────────────────┬──────────────────────────────────────┤
│  Next.js 14          │  FastAPI (Python 3.12)               │
│  TypeScript          │  SQLAlchemy 2 + Pydantic v2          │
│  Tailwind + shadcn   │  JWT Auth + Multi-tenant RLS         │
├──────────────────────┴──────────────────────────────────────┤
│  Redis 7 (broker + caché OSINT + results backend)          │
├──────────────────────────────────────────────────────────────┤
│  Celery Workers × 4 colas                                   │
│  [osint] [cyber/vulnscan] [credit] [default]               │
├──────────────────────────────────────────────────────────────┤
│  PostgreSQL 16 + pgvector (multi-tenant con RLS)           │
├──────────────────────────────────────────────────────────────┤
│  OSINT Engines: Nuclei · Nmap · Shodan · Amass · SSLyze    │
│  Playwright · HIBP · VirusTotal · TruffleHog · Hunter.io   │
└─────────────────────────────────────────────────────────────┘
```

### Ambiente de desarrollo recomendado

| Componente | Tecnología | Versión |
|------------|-----------|---------|
| OS | Kali Linux / Ubuntu 22.04+ | — |
| Lenguaje principal | Python | 3.12+ |
| Framework API | FastAPI + Uvicorn | 0.115.x |
| Task queue | Celery + RedBeat | 5.4.x |
| Frontend | Next.js + TypeScript | 14.x |
| Base de datos | PostgreSQL + pgvector | 16.x |
| Caché / Broker | Redis | 7.x |
| Contenedores | Docker + Compose | 24.x+ |
| Proxy | Nginx | alpine |
| Deploy | Hetzner VPS (CPX31) | — |

---

## Estructura del proyecto

```
intelpy/
├── 📄 docker-compose.yml          # Orquestación de 9 servicios
├── 📄 .env.example                # Variables de entorno (copiar a .env)
├── 📄 Makefile                    # Comandos de desarrollo
├── 📄 CLAUDE.md                   # Instrucciones para el agente AI
│
├── 🐍 backend/                    # Python / FastAPI
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app/
│       ├── main.py                # Entrypoint FastAPI
│       ├── config.py              # Settings con Pydantic
│       ├── api/
│       │   ├── v1/
│       │   │   ├── scans.py       # Endpoints de VulnScan
│       │   │   ├── osint.py       # Endpoints OSINT
│       │   │   ├── credit.py      # Endpoints crédito
│       │   │   └── auth.py        # Login, JWT, refresh
│       │   └── deps.py            # Dependencias compartidas
│       ├── models/
│       │   ├── scan.py            # ScanJob, Finding, Report
│       │   ├── osint.py           # Person, Company, Investigation
│       │   ├── credit.py          # CreditReport, Score
│       │   └── user.py            # User, Organization
│       ├── services/
│       │   ├── vulnscan/
│       │   │   ├── recon.py       # Amass + subfinder
│       │   │   ├── ports.py       # Nmap + Shodan
│       │   │   ├── web.py         # Nuclei + OWASP ZAP
│       │   │   ├── email_sec.py   # SPF/DKIM/DMARC
│       │   │   ├── ssl.py         # SSLyze
│       │   │   ├── leaks.py       # HIBP + Hunter
│       │   │   ├── cloud.py       # TruffleHog + gitleaks
│       │   │   └── scorer.py      # Risk Score 0-100
│       │   ├── osint/
│       │   │   ├── persons.py
│       │   │   ├── companies.py
│       │   │   └── scraper.py     # Playwright → SET, DNIT, etc.
│       │   ├── credit/
│       │   │   └── score.py
│       │   └── reports/
│       │       ├── pdf.py         # WeasyPrint PDF generation
│       │       └── templates/     # Jinja2 HTML → PDF
│       ├── worker/
│       │   ├── celery_app.py      # Config Celery + 4 colas
│       │   ├── tasks_vulnscan.py  # Tasks de escaneo
│       │   ├── tasks_osint.py
│       │   └── tasks_alerts.py    # Beat tasks (alertas periódicas)
│       └── migrations/            # Alembic migrations
│
├── ▲ frontend/                    # Next.js 14 / TypeScript
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       ├── app/
│       │   ├── (landing)/         # Página pública + pricing
│       │   ├── (dashboard)/       # App B2B autenticada
│       │   │   ├── scans/         # VulnScan dashboard
│       │   │   ├── osint/         # Búsquedas OSINT
│       │   │   └── reports/       # Historial de reportes
│       │   └── (score)/           # Portal B2C crédito personal
│       ├── components/
│       └── lib/
│
└── 🔧 infra/
    ├── nginx/conf.d/intelpy.conf  # Reverse proxy config
    ├── db/init.sql                # PostgreSQL init + pgvector
    └── certbot/                   # SSL automático
```

---

## Inicio rápido — Kali Linux / Ubuntu

### Pre-requisitos

```bash
# Docker + Compose
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Herramientas OSINT (en Kali ya vienen varias)
sudo apt install -y nmap amass
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest

# Node.js 20 LTS
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo bash -
sudo apt install -y nodejs
```

### Setup

```bash
# 1. Clonar el repositorio
git clone https://github.com/tuuser/intelpy.git
cd intelpy

# 2. Configurar variables de entorno
cp .env.example .env
nano .env   # ← completar POSTGRES_PASSWORD, SECRET_KEY, API keys

# 3. Levantar servicios
make dev

# 4. Aplicar migraciones de base de datos
make migrate

# 5. (Opcional) Cargar datos de prueba
make seed
```

### Verificar que todo funciona

```bash
# Estado de servicios
docker compose ps

# API docs (Swagger)
open http://localhost:8000/docs

# Frontend
open http://localhost:3000

# Monitor de tareas Celery
open http://localhost:5555   # usuario: admin
```

---

## Comandos útiles

```bash
make up           # Levantar todos los servicios
make dev          # Levantar con Flower (monitor Celery) en :5555
make down         # Detener todo
make logs         # Ver logs en tiempo real
make logs-api     # Logs solo del backend
make logs-worker  # Logs de los workers OSINT
make shell        # Bash dentro del contenedor API
make db-shell     # psql en PostgreSQL
make migrate      # Aplicar migraciones Alembic
make migration    # Crear nueva migración (pide nombre)
make test         # Ejecutar suite de tests
make rebuild      # Rebuild de imágenes sin caché
```

---

## Ejecución en Kali Linux — Modo desarrollo local

Para desarrollar sin Docker (más rápido para iterar):

```bash
# Backend
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
playwright install chromium

# Levantar PostgreSQL y Redis solo en Docker
docker compose up -d db redis

# Correr FastAPI en modo reload
uvicorn app.main:app --reload --port 8000

# En otra terminal — Worker Celery
celery -A app.worker.celery_app worker --loglevel=info -Q cyber,osint,credit,default

# Frontend (en otra terminal)
cd ../frontend
npm install
npm run dev   # → http://localhost:3000
```

---

## API pública — VulnScan endpoints

```bash
# Iniciar un escaneo
POST /api/v1/scans
{
  "target": "empresa-objetivo.com.py",
  "modules": ["recon", "ports", "web", "email", "ssl", "leaks"],
  "plan": "pro"
}
→ { "task_id": "abc-123", "status": "queued" }

# Consultar estado (polling)
GET /api/v1/scans/abc-123
→ { "status": "running", "progress": 45, "modules_done": ["recon", "ports"] }

# Resultado completo
GET /api/v1/scans/abc-123/results
→ { "score": 72, "findings": [...], "summary": {...} }

# Descargar PDF
GET /api/v1/scans/abc-123/report.pdf
→ application/pdf
```

---

## Seguridad y compliance

- **Ley 6534/21** — Protección de Datos Personales del Paraguay: política de privacidad registrada en MITIC
- **Uso ético**: VulnScan solo escanea dominios autorizados por escrito. Todos los escaneos requieren contrato de autorización firmado
- **Reconocimiento pasivo**: el módulo de recon usa solo fuentes públicas (CT logs, DNS, WHOIS) — sin tráfico directo al objetivo sin autorización
- **Multi-tenant RLS**: Row-Level Security en PostgreSQL garantiza aislamiento total entre clientes
- **Secrets**: nunca commitear `.env`. Usar `.env.example` como referencia

---

## Clientes objetivo

| Sector | Ejemplos | Producto |
|--------|---------|---------|
| Banca | Itaú PY, Continental, Sudameris | VulnScan Enterprise |
| Cooperativas | Chortitzer, Colonias Unidas, tipo A | VulnScan Pro |
| Salud | Clínicas privadas, distribuidoras farmacéuticas | VulnScan Pro |
| Legal | Estudios de abogados en CDE y Asunción | IntelPY Business |
| Industria | Importadoras Alto Paraná | IntelPY Business + VulnScan |
| Personas | Cualquier ciudadano paraguayo | IntelPY Score |

---

## Roadmap

- [x] Dashboard OSINT — Personas, Empresas, Cyber, Crédito
- [x] Stack Docker — FastAPI + PostgreSQL + Redis + Celery + Next.js
- [ ] VulnScan MVP — módulos recon + ports + email_sec
- [ ] Sistema de billing — Bancard integración
- [ ] Generación de reportes PDF — WeasyPrint
- [ ] API pública v1 + documentación Swagger
- [ ] IntelPY Score — portal B2C crédito personal
- [ ] Alertas por Telegram Bot
- [ ] App móvil (PWA)
- [ ] White-label para partners

---

## Licencia

Propietario — © 2025 IntelPY. Todos los derechos reservados.  
No se permite la redistribución, copia o uso comercial sin autorización expresa.

---

<div align="center">
  <strong>IntelPY</strong> · Ciudad del Este, Paraguay 🇵🇾<br>
  <a href="mailto:contacto@intelpy.com.py">contacto@intelpy.com.py</a>
</div>
